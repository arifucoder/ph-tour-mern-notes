# 30 — Booking, Payment, Transaction আর SSLCommerz Integration

এই note-এ যা বানাব:

- **Booking** আর **Payment** module (interface, model)
- **Transaction rollback** — একসাথে কয়েকটা collection-এ লেখার সময় সব হবে, নয়তো কিছুই হবে না
- **SSLCommerz** দিয়ে payment: initialize, success, fail, cancel, আবার pay করা
- Payment **validate** করা (নিরাপত্তার জন্য জরুরি)

## পুরো Flow এক নজরে

```
Frontend (localhost:5173)
   │  User tour বেছে "Book Now" চাপল
   ▼
POST /api/v1/booking  ──► Booking (PENDING) + Payment (UNPAID) তৈরি
   │                       SSLCommerz-এ payment initialize → paymentUrl
   ▼
Frontend user-কে paymentUrl-এ পাঠায় → SSLCommerz-এর payment page
   │
   ├─ ✅ Payment সফল
   │     SSLCommerz → POST /api/v1/payment/success
   │     Backend: SSLCommerz-এর কাছে validate → Payment PAID + Booking COMPLETE
   │     redirect → localhost:5173/payment/success
   │
   ├─ ❌ Payment fail
   │     SSLCommerz → POST /api/v1/payment/fail
   │     Backend: Payment FAILED + Booking FAILED
   │     redirect → localhost:5173/payment/fail
   │
   └─ 🚫 User cancel করল
         SSLCommerz → POST /api/v1/payment/cancel
         Backend: Payment CANCELLED + Booking CANCEL
         redirect → localhost:5173/payment/cancel
```

> **কেন সরাসরি frontend-এ না, আগে backend-এ?** SSLCommerz-এর `success_url` সরাসরি frontend দিলে user success page দেখবে ঠিকই, কিন্তু DB-তে payment PAID বা booking COMPLETE **হবেই না**। তাই আগে backend-এর route-এ আসে, backend DB update করে, তারপর frontend-এ redirect করে।

## Folder Structure

```
src/app/modules/
├── booking/
│   ├── booking.interface.ts
│   ├── booking.model.ts
│   ├── booking.validation.ts
│   ├── booking.service.ts
│   ├── booking.controller.ts
│   └── booking.route.ts
├── payment/
│   ├── payment.interface.ts
│   ├── payment.model.ts
│   ├── payment.service.ts
│   ├── payment.controller.ts
│   └── payment.route.ts
└── sslCommerz/
    ├── sslCommerz.interface.ts
    └── sslCommerz.service.ts
```

আলাদা module মানে DB-তে আলাদা collection: `bookings`, `payments`। (`sslCommerz`-এর কোনো collection নেই, এটা শুধু বাইরের payment gateway-র সাথে কথা বলে।)

---

# Part 1: Booking আর Payment-এর Model

## Step 1: Booking-এ কী তথ্য থাকবে?

User login করে একটা tour বেছে booking করবে। Booking-এ থাকবে:

- **কে** booking করছে → `user`
- **কোন tour** → `tour`
- **কতজন** যাবে (একা নাকি সাথে আরও কেউ) → `guestCount`
- **Payment-এর তথ্য** → `payment`
- **অবস্থা** → `status`

### Status-গুলো

| Booking Status | কখন |
|---|---|
| `PENDING` | Booking হয়েছে, এখনো pay করেনি (শুরুর অবস্থা) |
| `COMPLETE` | Payment সফল |
| `FAILED` | Payment করতে গিয়ে fail |
| `CANCEL` | User cancel করেছে |

| Payment Status | কখন |
|---|---|
| `UNPAID` | Payment record তৈরি, টাকা আসেনি (শুরুর অবস্থা) |
| `PAID` | টাকা এসেছে |
| `FAILED` | Payment fail |
| `CANCELLED` | User cancel করেছে |
| `REFUNDED` | টাকা ফেরত দেওয়া হয়েছে (booking বাতিল হলে পরে লাগবে) |

> Teacher-এর flow comment-এ "Booking CONFIRM" লেখা, কিন্তু enum-এ নাম `COMPLETE`। Code-এ `COMPLETE`-ই ব্যবহার হবে।

---

## Step 2: Booking Interface

```ts
// src/app/modules/booking/booking.interface.ts
import type { Types } from "mongoose";

export enum BOOKING_STATUS {
	PENDING = "PENDING",
	CANCEL = "CANCEL",
	COMPLETE = "COMPLETE",
	FAILED = "FAILED",
}

export interface IBooking {
	user: Types.ObjectId;
	tour: Types.ObjectId;
	payment?: Types.ObjectId; // booking তৈরির মুহূর্তে payment থাকে না, পরে বসে
	guestCount: number;
	status: BOOKING_STATUS;
}
```

> `Types` শুধু type হিসেবে ব্যবহার হচ্ছে, তাই `import type` (`verbatimModuleSyntax`)।

---

## Step 3: Payment Interface

```ts
// src/app/modules/payment/payment.interface.ts
import type { Types } from "mongoose";

export enum PAYMENT_STATUS {
	PAID = "PAID",
	UNPAID = "UNPAID",
	CANCELLED = "CANCELLED",
	FAILED = "FAILED",
	REFUNDED = "REFUNDED",
}

export interface IPayment {
	booking: Types.ObjectId;
	transactionId: string;
	amount: number;
	paymentGatewayData?: unknown; // SSLCommerz যা পাঠায়, যেকোনো shape
	invoiceUrl?: string;
	status: PAYMENT_STATUS;
}
```

### দুই দিকের Reference (Two-way referencing)

```
Booking { payment: <paymentId> }  ◄──►  Payment { booking: <bookingId> }
```

- Booking দেখার সময় এক click-এ তার payment দেখা যাবে।
- Payment দেখার সময় এক click-এ তার booking দেখা যাবে।
- Payment সবসময় একটা booking-এর জন্যই হয়, তাই payment-এ `booking` **required**।

### অন্য field

- **`transactionId`**: প্রতিটা payment-এর আলাদা পরিচয়, **unique**। SSLCommerz-এ এটাই পাঠাই, আর ফিরে এলে এটা দিয়েই বুঝি কোন payment।
- **`paymentGatewayData`, `invoiceUrl` optional**: শুরুতে এই তথ্য থাকে না, payment শেষ হওয়ার পরে আসে।

> ✏️ **`any` → `unknown`:** Teacher `any` দিয়েছিলেন (তাই eslint-disable লাগত)। `unknown`-ও "যেকোনো type" বোঝায়, কিন্তু নিরাপদ: ব্যবহারের আগে check করতে বাধ্য করে। ESLint-ও আর আপত্তি করে না।

---

## Step 4: Booking Model

```ts
// src/app/modules/booking/booking.model.ts
import { model, Schema } from "mongoose";
import { BOOKING_STATUS, type IBooking } from "./booking.interface";

const bookingSchema = new Schema<IBooking>(
	{
		user: {
			type: Schema.Types.ObjectId,
			ref: "User",
			required: true,
		},
		tour: {
			type: Schema.Types.ObjectId,
			ref: "Tour",
			required: true,
		},
		payment: {
			type: Schema.Types.ObjectId,
			ref: "Payment",
		},
		status: {
			type: String,
			enum: Object.values(BOOKING_STATUS),
			default: BOOKING_STATUS.PENDING,
		},
		guestCount: {
			type: Number,
			required: true,
			min: 1,
		},
	},
	{
		timestamps: true,
	},
);

export const Booking = model<IBooking>("Booking", bookingSchema);
```

- **Interface-এ** `Types.ObjectId`, **Schema-তে** `Schema.Types.ObjectId`।
- **`ref: "User"`**: বড় হাতের `User`, কারণ `model("User", ...)`-এ যেভাবে নাম দেওয়া আছে, হুবহু সেটাই দিতে হবে।
- **`enum: Object.values(BOOKING_STATUS)`**: enum-এর সব value-র array (`["PENDING", "CANCEL", ...]`)। এর বাইরে কিছু save হবে না।
- `guestCount`-এ `min: 1` যোগ করেছি, যাতে DB-তেও ০ বা negative না ঢোকে।

---

## Step 5: Payment Model

```ts
// src/app/modules/payment/payment.model.ts
import { model, Schema } from "mongoose";
import { type IPayment, PAYMENT_STATUS } from "./payment.interface";

const paymentSchema = new Schema<IPayment>(
	{
		booking: {
			type: Schema.Types.ObjectId,
			ref: "Booking",
			required: true,
			unique: true, // একটা booking-এর একটাই payment
		},
		transactionId: {
			type: String,
			required: true,
			unique: true,
		},
		status: {
			type: String,
			enum: Object.values(PAYMENT_STATUS),
			default: PAYMENT_STATUS.UNPAID,
		},
		amount: {
			type: Number,
			required: true,
		},
		paymentGatewayData: {
			type: Schema.Types.Mixed,
		},
		invoiceUrl: {
			type: String,
		},
	},
	{
		timestamps: true,
	},
);

export const Payment = model<IPayment>("Payment", paymentSchema);
```

- **`booking: { unique: true }`**: একটা booking-এর জন্য payment record একবারই তৈরি হবে। একই booking-এ দুটো payment বানাতে গেলে duplicate error (`11000`)।
- **`Schema.Types.Mixed`**: interface-এ যেহেতু "যেকোনো type", schema-তেও যেকোনো কিছু রাখা যায় এমন type লাগবে। Payment gateway JSON পাঠাক বা array — যা-ই আসুক এখানে রাখা যাবে।

---

# Part 2: Transaction Rollback

## Step 6: সমস্যাটা কী?

Booking করার সময় **৩টা write** হয়:

```
① Booking তৈরি → ② Payment তৈরি → ③ Booking-এ payment id বসানো
```

ধরো ① হয়ে গেল, তারপর কোনো error-এ ② হলো না:

```ts
const booking = await Booking.create({ ... }); // ✅ DB-তে save হয়ে গেছে

throw new Error("Some fake error");            // 💥

const payment = await Payment.create({ ... }); // ❌ কখনো চলবে না
```

এখন DB-তে একটা **PENDING booking আছে, কিন্তু তার কোনো payment নেই**। User কোনোদিন pay করতে পারবে না, আর এই অর্ধেক data DB-তে ময়লা হয়ে পড়ে থাকবে।

**আমরা চাই: সব write সফল হবে, নয়তো কোনোটাই হবে না।** এর সমাধানই **transaction**।

---

## Step 7: Transaction কী?

কয়েকটা DB operation-কে একটা **বাক্সে** রাখা হয়:

```
session শুরু
┌─────────────────────────────────────────────────┐
│ Create Booking → Create Payment → Update Booking │   ← এখনো "খসড়া", কেউ দেখতে পায় না
└─────────────────────────────────────────────────┘
        │
        ├─ সব ঠিক → commitTransaction() → সব একসাথে আসল DB-তে ✅
        └─ কোনো error → abortTransaction() → সব মুছে যায়, যেন কিছুই হয়নি ↩️ (rollback)
```

এই বাক্সকে Mongoose-এ বলে **session**।

> ✏️ **সংশোধন:** "Transaction DB-র collection-গুলোর duplicate copy বা replica বানায়" — আসলে MongoDB কোনো copy বানায় না। Transaction-এর ভেতরের write-গুলো **commit হওয়ার আগ পর্যন্ত লুকানো থাকে** (অন্য কেউ দেখতে পায় না), commit হলে একসাথে সবার জন্য দৃশ্যমান হয়, abort হলে বাতিল হয়। "আলাদা sandbox"-এর মতো ভাবা বোঝার জন্য ঠিক আছে, শুধু জেনে রাখো আসলে copy হয় না।
>
> "Replica" শব্দটা আসে অন্য জায়গা থেকে: MongoDB-তে transaction শুধু **replica set**-এ চলে (একাধিক server-এ একই data রাখার ব্যবস্থা)।

> ⚠️ **জরুরি:** MongoDB **Atlas** সবসময় replica set, তাই সেখানে transaction কাজ করে। কিন্তু নিজের computer-এ সাধারণভাবে install করা MongoDB (standalone) হলে error আসবে: `Transaction numbers are only allowed on a replica set member or mongos`। তখন Atlas ব্যবহার করো, বা local MongoDB-কে replica set হিসেবে চালাও।

**কখন transaction লাগবে?** যখন একসাথে **একাধিক collection-এ (বা একাধিক document-এ) write** হচ্ছে, আর সবগুলো একসাথে সফল হওয়া জরুরি। একটা মাত্র document create/update হলে লাগে না।

---

## Step 8: Transaction কীভাবে লিখব (Template)

```ts
const session = await Booking.startSession(); // ১. session শুরু
session.startTransaction();                   // ২. transaction শুরু

try {
	// ৩. প্রতিটা DB operation-এ { session } দিতে হবে
	const booking = await new Booking({ ... }).save({ session });
	const payment = await new Payment({ ... }).save({ session });
	await Booking.findByIdAndUpdate(booking._id, { payment: payment._id }, { session });

	await session.commitTransaction(); // ৪. সব ঠিক → আসল DB-তে
	return result;
} catch (error) {
	await session.abortTransaction();  // ৫. error → সব বাতিল (rollback)
	throw error;                       // ৬. error আবার ছুঁড়ে দাও
} finally {
	await session.endSession();        // ৭. সফল হোক বা না হোক, session বন্ধ
}
```

### প্রতিটা অংশ

- **`Booking.startSession()`**
  > ✏️ "Booking-এর উপর session শুরু করছি, তাই booking-এর উপর sandbox" — ঠিক না। Session collection-ভিত্তিক না, পুরো DB connection-এর। `Booking.startSession()`, `Payment.startSession()` বা `mongoose.startSession()` — সব একই জিনিস। একটা session দিয়েই Booking, Payment সব collection-এ কাজ করা যায়।

- **`try/catch` বাধ্যতামূলক**
  > ✏️ "try/catch না দিলেও হতো" — transaction-এ না দিলে হবে না। Error হলে `abortTransaction()` call করার জায়গাই `catch`। এটা না থাকলে rollback হবে না।

- **প্রতিটা operation-এ `{ session }`**
  যে operation-এ session দেওয়া নেই, সেটা transaction-এর **বাইরে** চলে — সাথে সাথে আসল DB-তে লেখা হয়ে যায়, rollback হয় না।
  Transaction-এর ভেতরে লেখার **পরে** পড়তে হলে (find/populate), সেখানেও session দিতে হয়। নাহলে সে commit-না-হওয়া (লুকানো) data দেখতে পায় না।

- **`Model.create()` আর array**
  > ✏️ "Session দিলে result array হয়ে যায়" — পুরোপুরি ঠিক না। `Model.create()`-এ options (যেমন session) দিতে হলে Mongoose-এর নিয়ম হলো data **array**-তে দিতে হবে: `Booking.create([{ ... }], { session })`। তাই result-ও array, আর `booking[0]._id` লিখতে হয়।
  >
  > কিন্তু আমাদের project-এ `noUncheckedIndexedAccess: true`, তাই `booking[0]`-এর type হয় `... | undefined` — TypeScript error দেয়। তাই সহজ উপায় ব্যবহার করেছি: **`new Booking({...}).save({ session })`** — এটা সরাসরি একটা document দেয়, array না।

- **`finally`-তে `endSession()`**
  Teacher `try` আর `catch` দুই জায়গায় আলাদা করে `session.endSession()` লিখেছিলেন। `finally` সবসময় চলে (সফল হোক বা error), তাই একবার লিখলেই হয়। আর `endSession()` একটা Promise দেয়, তাই `await` দিয়েছি।

- **`catch`-এ `throw error`, `AppError` না**
  Error-কে যেমন আছে তেমনই আবার ছুঁড়ে দিই। কারণ:
  - Error-টা আগেই `AppError` হতে পারে (যেমন "Tour not found" — 404), নতুন করে মোড়ালে সঠিক status হারিয়ে যেত।
  - Mongoose/Zod error হলে globalErrorHandler (note 25) নিজেই সুন্দর করে handle করবে।
  - `new AppError(400, error)` এমনিতেও ভুল: `error`-এর type `unknown`, অথচ AppError-এর ২য় parameter string।

---

# Part 3: SSLCommerz Setup

## Step 9: Sandbox Account খোলা

Real account client খুলবে, তারপর আমাদের access দিলে আমরা integrate করব। Development-এর জন্য নিজের **sandbox (test) account** খুলি:

1. যাও: https://developer.sslcommerz.com/ → Get Started → documentation
2. Sandbox account-এর জন্য register করো:
   - **Domain:** `localhost` দিলেই হবে
   - **Email:** আসল email লাগবে (credential এখানে আসবে)। Real account-এ domain-এর email লাগে, sandbox-এ personal Gmail দিলেই চলে।
   - **Phone:** demo number দিলেও চলে
   - **Username:** মনে রাখো, পরে login-এ লাগবে
3. Email-এ **Store ID** আর **Store Password** আসবে। এগুলো `.env`-এ রাখব।

Documentation: https://developer.sslcommerz.com/doc/v4/

---

## Step 10: Integration-এর উপায় আর ৩টা API

### দুই ধরনের integration

| উপায় | কীভাবে কাজ করে |
|---|---|
| **Easy Checkout** | আমাদের frontend-এর page-এই একটা popup/modal খুলে payment হয় |
| **Hosted Page** ✅ | User-কে SSLCommerz-এর নিজস্ব page-এ redirect করা হয়, সেখানে pay করে, তারপর backend → frontend-এ ফিরে আসে |

আমরা **Hosted Page** ব্যবহার করব (উপরের flow-টাই এটা)।

### SSLCommerz-এর ৩টা API

| API | কাজ | কখন |
|---|---|---|
| **Create Session** (Init) | Payment শুরু করা, payment page-এর URL পাওয়া | Booking-এর সময় |
| **IPN** (Instant Payment Notification) | Payment হলে SSLCommerz নিজে আমাদের server-এর একটা URL-এ POST করে জানায় | Payment-এর পরে |
| **Order Validation** | "এই payment সত্যিই সফল তো?" — SSLCommerz-এর কাছে যাচাই | Success-এর পরে, **বাধ্যতামূলক** |

> ✏️ **সংশোধন:** "Validation-এর জন্য deploy করা live link লাগবে" — শুধু **IPN**-এর জন্য লাগে, কারণ IPN-এ SSLCommerz-এর server **আমাদের server-কে** call করে, আর SSLCommerz `localhost`-এ পৌঁছাতে পারে না। কিন্তু **Order Validation API**-তে আমাদের server **SSLCommerz-কে** call করে — এটা localhost থেকেও চলে। তাই validation এখনই যোগ করেছি (Step 19), কারণ এটা ছাড়া payment system নিরাপদ না। IPN পরে deploy-এর সময়।

---

## Step 11: Environment Variable

### `.env`

```bash
# SSLCommerz credentials
SSL_STORE_ID=your_store_id
SSL_STORE_PASS=your_store_password
SSL_PAYMENT_API="https://sandbox.sslcommerz.com/gwprocess/v4/api.php"
SSL_VALIDATION_API="https://sandbox.sslcommerz.com/validator/api/validationserverAPI.php"

# SSLCommerz এখানে পাঠাবে (backend)
SSL_SUCCESS_BACKEND_URL="http://localhost:5000/api/v1/payment/success"
SSL_FAIL_BACKEND_URL="http://localhost:5000/api/v1/payment/fail"
SSL_CANCEL_BACKEND_URL="http://localhost:5000/api/v1/payment/cancel"

# Backend user-কে এখানে পাঠাবে (frontend)
SSL_SUCCESS_FRONTEND_URL="http://localhost:5173/payment/success"
SSL_FAIL_FRONTEND_URL="http://localhost:5173/payment/fail"
SSL_CANCEL_FRONTEND_URL="http://localhost:5173/payment/cancel"
```

মোট ৬টা URL: ৩টা backend (SSLCommerz যেখানে আসবে), ৩টা frontend (user শেষে যেখানে যাবে)।

> ⚠️ `SSL_STORE_PASS` গোপন। `.env` যেন `.gitignore`-এ থাকে।

### `env.ts` — ৩ জায়গায় যোগ

সব SSL variable একটা nested `SSL` object-এ রাখা হয়েছে, যাতে `envVars.SSL.STORE_ID` এভাবে গুছিয়ে ব্যবহার করা যায়।

```ts
// src/app/config/env.ts

// ① interface-এ
interface EnvConfig {
	// ... আগের সব
	SSL: {
		STORE_ID: string;
		STORE_PASS: string;
		SSL_PAYMENT_API: string;
		SSL_VALIDATION_API: string;
		SSL_SUCCESS_BACKEND_URL: string;
		SSL_FAIL_BACKEND_URL: string;
		SSL_CANCEL_BACKEND_URL: string;
		SSL_SUCCESS_FRONTEND_URL: string;
		SSL_FAIL_FRONTEND_URL: string;
		SSL_CANCEL_FRONTEND_URL: string;
	};
}

// ② requiredEnvVariables array-তে
const requiredEnvVariables: string[] = [
	// ... আগের সব
	"SSL_STORE_ID",
	"SSL_STORE_PASS",
	"SSL_PAYMENT_API",
	"SSL_VALIDATION_API",
	"SSL_SUCCESS_BACKEND_URL",
	"SSL_FAIL_BACKEND_URL",
	"SSL_CANCEL_BACKEND_URL",
	"SSL_SUCCESS_FRONTEND_URL",
	"SSL_FAIL_FRONTEND_URL",
	"SSL_CANCEL_FRONTEND_URL",
];

// ③ return object-এ
return {
	// ... আগের সব
	SSL: {
		STORE_ID: process.env.SSL_STORE_ID as string,
		STORE_PASS: process.env.SSL_STORE_PASS as string,
		SSL_PAYMENT_API: process.env.SSL_PAYMENT_API as string,
		SSL_VALIDATION_API: process.env.SSL_VALIDATION_API as string,
		SSL_SUCCESS_BACKEND_URL: process.env.SSL_SUCCESS_BACKEND_URL as string,
		SSL_FAIL_BACKEND_URL: process.env.SSL_FAIL_BACKEND_URL as string,
		SSL_CANCEL_BACKEND_URL: process.env.SSL_CANCEL_BACKEND_URL as string,
		SSL_SUCCESS_FRONTEND_URL: process.env.SSL_SUCCESS_FRONTEND_URL as string,
		SSL_FAIL_FRONTEND_URL: process.env.SSL_FAIL_FRONTEND_URL as string,
		SSL_CANCEL_FRONTEND_URL: process.env.SSL_CANCEL_FRONTEND_URL as string,
	},
};
```

> 💡 `DB_URL: process.env.DB_URL!` — `!` (non-null assertion) দিয়েও করা যায়, কিন্তু আমাদের ESLint config-এ `!` error দেয় (তাই eslint-disable লাগছিল)। উপরে `requiredEnvVariables` আগেই check করে, তাই সব জায়গায় একইভাবে `as string` রাখাই পরিষ্কার।

---

## Step 12: SSLCommerz Service (Payment Initialize + Validate)

### Axios install

SSLCommerz-এর API-তে request পাঠাতে হবে। `fetch` দিয়েও হয়, তবে আমরা **axios** ব্যবহার করব:

```bash
npm i axios
```

### Interface

```ts
// src/app/modules/sslCommerz/sslCommerz.interface.ts

// SSLCommerz-এ পাঠানোর জন্য আমাদের দরকারি তথ্য
export interface ISSLCommerz {
	amount: number;
	transactionId: string;
	name: string;
	email: string;
	phoneNumber: string;
	address: string;
}

// Initialize করার পর SSLCommerz যা ফেরত দেয় (দরকারি অংশ)
export interface ISSLInitResponse {
	status: "SUCCESS" | "FAILED";
	GatewayPageURL?: string; // এই URL-এ user-কে পাঠাব
	failedreason?: string;
}

// Validation API যা ফেরত দেয় (দরকারি অংশ)
export interface ISSLValidationResponse {
	status: string; // "VALID" | "VALIDATED" | "INVALID_TRANSACTION"
	tran_id: string;
	amount: string;
}
```

### Service

```ts
// src/app/modules/sslCommerz/sslCommerz.service.ts
import axios from "axios";
import httpStatus from "http-status-codes";
import { envVars } from "../../config/env";
import AppError from "../../errorHelpers/AppError";
import type { ISSLCommerz, ISSLInitResponse, ISSLValidationResponse } from "./sslCommerz.interface";

// Payment শুরু করা → SSLCommerz-এর payment page-এর URL return করে
const sslPaymentInit = async (payload: ISSLCommerz): Promise<string> => {
	const data = {
		store_id: envVars.SSL.STORE_ID,
		store_passwd: envVars.SSL.STORE_PASS,
		total_amount: payload.amount,
		currency: "BDT",
		tran_id: payload.transactionId,
		success_url: `${envVars.SSL.SSL_SUCCESS_BACKEND_URL}?transactionId=${payload.transactionId}`,
		fail_url: `${envVars.SSL.SSL_FAIL_BACKEND_URL}?transactionId=${payload.transactionId}`,
		cancel_url: `${envVars.SSL.SSL_CANCEL_BACKEND_URL}?transactionId=${payload.transactionId}`,
		// ipn_url: পরে deploy-এর সময়
		shipping_method: "N/A",
		product_name: "Tour",
		product_category: "Service",
		product_profile: "general",
		cus_name: payload.name,
		cus_email: payload.email,
		cus_add1: payload.address,
		cus_add2: "N/A",
		cus_city: "Dhaka",
		cus_state: "Dhaka",
		cus_postcode: "1000",
		cus_country: "Bangladesh",
		cus_phone: payload.phoneNumber,
		cus_fax: "01711111111",
		ship_name: "N/A",
		ship_add1: "N/A",
		ship_add2: "N/A",
		ship_city: "N/A",
		ship_state: "N/A",
		ship_postcode: "1000",
		ship_country: "N/A",
	};

	const response = await axios
		.post<ISSLInitResponse>(envVars.SSL.SSL_PAYMENT_API, data, {
			headers: { "Content-Type": "application/x-www-form-urlencoded" },
		})
		.catch(() => {
			throw new AppError(httpStatus.BAD_GATEWAY, "Could not connect to SSLCommerz. Please try again.");
		});

	// SSLCommerz উত্তর দিয়েছে, কিন্তু payment শুরু করতে পারেনি (যেমন ভুল store id)
	if (response.data.status !== "SUCCESS" || !response.data.GatewayPageURL) {
		throw new AppError(
			httpStatus.BAD_GATEWAY,
			`Payment initialization failed: ${response.data.failedreason ?? "Unknown reason"}`,
		);
	}

	return response.data.GatewayPageURL;
};

// Payment সত্যিই সফল কিনা SSLCommerz-এর কাছে যাচাই
const validatePayment = async (valId: string): Promise<ISSLValidationResponse> => {
	const response = await axios
		.get<ISSLValidationResponse>(envVars.SSL.SSL_VALIDATION_API, {
			params: {
				val_id: valId,
				store_id: envVars.SSL.STORE_ID,
				store_passwd: envVars.SSL.STORE_PASS,
				format: "json",
			},
		})
		.catch(() => {
			throw new AppError(httpStatus.BAD_GATEWAY, "Could not validate payment with SSLCommerz.");
		});

	return response.data;
};

export const SSLService = {
	sslPaymentInit,
	validatePayment,
};
```

### ❓ `Content-Type: "application/x-www-form-urlencoded"` কেন?

Data পাঠানোর দুটো common format:

| Format | দেখতে কেমন | কে ব্যবহার করে |
|---|---|---|
| `application/json` | `{"store_id":"abc","total_amount":1500}` | আমাদের নিজের API |
| `application/x-www-form-urlencoded` | `store_id=abc&total_amount=1500` | সাধারণ HTML `<form>` submit |

SSLCommerz-এর API **form data** আশা করে, JSON না। তাই header-এ বলে দিই "আমি form format-এ পাঠাচ্ছি"। Axios এই header দেখে নিজেই object-কে `key=value&key=value` format-এ রূপান্তর করে পাঠায়।

### Teacher-এর code থেকে যা বদলেছি

| বিষয় | আগে | এখন | কেন |
|---|---|---|---|
| URL-এ query | `transactionId`, `amount`, `status` | শুধু `transactionId` | `amount`/`status` URL থেকে নিলে যে কেউ বদলে দিতে পারে; আসল amount DB-তে আছে, status route থেকেই বোঝা যায় |
| Return | পুরো `response.data` | শুধু `GatewayPageURL` | Caller-এর শুধু URL-টাই দরকার |
| SSL "FAILED" উত্তর | Check ছিল না → `paymentUrl: undefined` | Error throw | ভুল store id-তে চুপচাপ undefined যেত |
| Error | `catch (error: any)` + `console.log` | `.catch()` → `AppError(502)` | `any` লাগে না; 502 = বাইরের service-এ সমস্যা |
| `ship_postcode` | `1000` (number) | `"1000"` | বাকি সব field-এর মতো string |
| Validation | ছিল না | `validatePayment` | নিরাপত্তা (Step 19) |

---

# Part 4: Booking API

## Step 13: Validation

```ts
// src/app/modules/booking/booking.validation.ts
import { z } from "zod";
import { BOOKING_STATUS } from "./booking.interface";

export const createBookingZodSchema = z.object({
	tour: z.string({ error: "Tour id is required" }),
	guestCount: z
		.number({ error: "Guest count is required" })
		.int({ error: "Guest count must be a whole number" })
		.positive({ error: "Guest count must be at least 1" }),
});

export const updateBookingStatusZodSchema = z.object({
	status: z.enum(BOOKING_STATUS),
});

// Zod schema থেকে সরাসরি TypeScript type
export type TCreateBookingPayload = z.infer<typeof createBookingZodSchema>;
```

- **`user` validation-এ নেই কেন?** User কে, সেটা client পাঠাবে না, **token থেকে** নেব (Step 15)। Client-এর পাঠানো user id বিশ্বাস করলে যে কেউ অন্যের নামে booking করতে পারত।
- **Update-এ শুধু `status`** লাগবে।
- ✏️ **`z.enum(Object.values(BOOKING_STATUS) as [string])` → `z.enum(BOOKING_STATUS)`**: Zod v4-এ TypeScript enum সরাসরি দেওয়া যায় (Role-এর মতো)। `as [string]` লাগে না, আর type-ও ঠিক থাকে।
- **`z.infer`**: Schema থেকে type বানায়: `{ tour: string; guestCount: number }`। Service-এ এই type ব্যবহার করলে `guestCount` নিশ্চিতভাবে number — teacher-এর `payload.guestCount!` (non-null assertion) আর লাগে না।

---

## Step 14: Transaction ID

```ts
// booking.service.ts-এর উপরে
import { randomUUID } from "node:crypto";

const getTransactionId = () => `tran_${randomUUID()}`;
// tran_3f9a1c2e-8b4d-4e7a-9c1f-2d5e6a7b8c9d
```

> ✏️ **সংশোধন:** Teacher-এর `tran_${Date.now()}_${Math.floor(Math.random() * 1000)}` "secure" না। `Date.now()` থেকে সময় অনুমান করা যায়, আর random অংশ মাত্র ১০০০ রকম — একই millisecond-এ দুটো booking হলে মিলে যাওয়ার সম্ভাবনা আছে। Node-এর built-in **`randomUUID()`** প্রায় অসম্ভব রকম unique আর অনুমান করা যায় না। কোনো package install লাগে না।

---

## Step 15: Create Booking Service

### ❓ Booking-এর সাথে সাথেই Payment তৈরি — ঠিক process?

হ্যাঁ, এটাই standard process। কারণ এখানে Payment record মানে **"টাকা এসেছে" না**, মানে **"এই booking-এর জন্য এত টাকা দিতে হবে" — একটা bill/invoice**, যার status `UNPAID`।

User-কে SSLCommerz page-এ পাঠানোর **আগেই** এটা লাগবে, কারণ:

1. **SSLCommerz-কে `transactionId` আর `amount` দিতে হয়।** এগুলো কোথাও save না থাকলে payment শেষে SSLCommerz যখন `transactionId` নিয়ে ফিরে আসবে, আমরা বুঝবই না এটা কোন booking-এর।
2. **Amount booking-এর সময়েই ঠিক হয়ে যায়।** পরে tour-এর দাম বদলালেও এই booking-এর amount বদলাবে না।

আসল টাকা লেনদেন হয় SSLCommerz-এর page-এ। সফল হলে আমরা শুধু এই record-এর status `UNPAID` → `PAID` বদলাই।

### Code

```ts
const createBooking = async (payload: TCreateBookingPayload, userId: string) => {
	// ── Transaction-এর আগে: শুধু পড়া আর check ──
	const user = await User.findById(userId);

	// Booking-এর পর user-এর সাথে যোগাযোগ করতে হবে, তাই phone আর address বাধ্যতামূলক
	if (!user?.phone || !user.address) {
		throw new AppError(httpStatus.BAD_REQUEST, "Please update your profile (phone and address) to book a tour.");
	}

	// শুধু costFrom দরকার, তাই select করে শুধু ওটাই আনি
	const tour = await Tour.findById(payload.tour).select("costFrom");

	if (!tour) {
		throw new AppError(httpStatus.NOT_FOUND, "Tour not found.");
	}
	if (!tour.costFrom) {
		throw new AppError(httpStatus.BAD_REQUEST, "This tour has no price set yet.");
	}

	const amount = tour.costFrom * payload.guestCount;
	const transactionId = getTransactionId();

	// ── Transaction: ৩টা write একসাথে ──
	const session = await Booking.startSession();
	session.startTransaction();

	try {
		// ① Booking তৈরি
		const booking = await new Booking({
			user: userId,
			tour: payload.tour,
			guestCount: payload.guestCount,
			status: BOOKING_STATUS.PENDING,
		}).save({ session });

		// ② Payment (bill) তৈরি
		const payment = await new Payment({
			booking: booking._id,
			transactionId,
			amount,
			status: PAYMENT_STATUS.UNPAID,
		}).save({ session });

		// ③ Booking-এ payment id বসানো + response-এর জন্য populate
		const updatedBooking = await Booking.findByIdAndUpdate(
			booking._id,
			{ payment: payment._id },
			{ new: true, runValidators: true, session },
		)
			.populate("tour", "title costFrom")
			.populate("payment");

		// ④ SSLCommerz-এ payment শুরু
		const paymentUrl = await SSLService.sslPaymentInit({
			name: user.name,
			email: user.email,
			phoneNumber: user.phone,
			address: user.address,
			amount,
			transactionId,
		});

		await session.commitTransaction(); // সব ঠিক → আসল DB-তে
		return { paymentUrl, booking: updatedBooking };
	} catch (error) {
		await session.abortTransaction(); // কোথাও error → সব বাতিল
		throw error;
	} finally {
		await session.endSession();
	}
};
```

### বোঝার মতো কিছু point

- **SSLCommerz init transaction-এর ভেতরে কেন?** SSLCommerz payment শুরু করতে না পারলে (network সমস্যা, ভুল store id) user pay-ই করতে পারবে না। তখন booking রেখে কী লাভ? Error হলে পুরো booking rollback হয়ে যায়, user আবার চেষ্টা করতে পারে।
- **`.populate("tour", "title costFrom")`**: ১ম argument = **schema-র field-এর নাম** (`tour`), ২য় = কোন কোন field আনব। `ref: "Tour"` দেখে Mongoose বোঝে কোন collection থেকে আনতে হবে।
- **Populate-এ session:** Query-তে session দেওয়া থাকলে populate-ও সেই session ব্যবহার করে, তাই এখনো commit-না-হওয়া payment-ও দেখতে পায়।

### Teacher-এর code থেকে যা বদলেছি

| বিষয় | আগে | এখন | কেন |
|---|---|---|---|
| ⚠️ `...payload` | `{ user, status, ...payload }` | শুধু `tour`, `guestCount` আলাদা করে | `...payload` **শেষে** থাকায় client body-তে `status: "COMPLETE"` বা অন্য `user` পাঠিয়ে সেগুলো **override** করতে পারত — টাকা না দিয়েই booking COMPLETE! |
| User-এর তথ্য | Populate করে `(updatedBooking?.user as any).address` | আগেই আনা `user` থেকে সরাসরি | একবার আনা data আবার আনার দরকার নেই, `any`-ও লাগে না |
| Tour না পেলে | "No Tour Cost Found!" | আলাদা 404 "Tour not found." | আসল সমস্যা বোঝা যায় |
| Check-গুলো | Transaction-এর ভেতরে | Transaction-এর আগে | শুধু পড়া, transaction লাগে না; transaction ছোট ও দ্রুত থাকে |
| Create | `Booking.create([...])` + `booking[0]` | `new Booking().save({ session })` | `noUncheckedIndexedAccess`-এ `[0]` TypeScript error দেয় |
| `endSession` | `try` আর `catch` দুই জায়গায় | `finally`-তে একবার, `await` সহ | |
| `console.log(sslPayment)` | ছিল | সরানো | ESLint `no-console` |

---

## Step 16: Booking Controller

```ts
// src/app/modules/booking/booking.controller.ts
import type { Request, Response } from "express";
import httpStatus from "http-status-codes";
import type { JwtPayload } from "jsonwebtoken";
import { catchAsync } from "../../utils/catchAsync";
import { sendResponse } from "../../utils/sendResponse";
import { BookingService } from "./booking.service";

const createBooking = catchAsync(async (req: Request, res: Response) => {
	const decodedToken = req.user as JwtPayload;
	const result = await BookingService.createBooking(req.body, decodedToken.userId as string);

	sendResponse(res, {
		statusCode: httpStatus.CREATED,
		success: true,
		message: "Booking created successfully",
		data: result,
	});
});
```

### ❓ `req.user` কীভাবে decoded token হলো?

এটা আমাদের **`checkAuth`** middleware-এর কাজ, যেটা controller-এর **আগে** চলে:

```
Request (Authorization: <accessToken>)
   │
   ▼
checkAuth
   ├─ jwt.verify(token, secret)  → token সঠিক হলে ভেতরের data বের করে (decode)
   │                                { userId, email, role, iat, exp }
   ├─ DB-তে user আছে / blocked না — check
   └─ req.user = verifiedToken   ← এখানে বসায়
   │
   ▼
controller → req.user.userId পাওয়া যায়
```

Login-এর সময় `createUserTokens` token-এর ভেতরে `userId`, `email`, `role` রেখেছিল। `jwt.verify` সেগুলোই বের করে দেয়, আর checkAuth সেটা `req.user`-এ রেখে দেয়। তাই একে বলি **decoded token**।

`req.user`-এর type আমরা `interfaces/index.d.ts`-এ `JwtPayload` বলে দিয়েছি। `JwtPayload`-এর ভেতরের custom field (`userId`) TypeScript চেনে না, তাই `as string`।

### বাকি Controller (এখন stub)

`getUserBookings`, `getSingleBooking`, `getAllBookings`, `updateBookingStatus` এখন শুধু খালি `{}` return করে। পরের class-এ বানানো হবে। Service-এও এদের জায়গা রাখা আছে:

```ts
// booking.service.ts — পরে implement হবে
const getUserBookings = async () => ({});
const getBookingById = async () => ({});
const updateBookingStatus = async () => ({});
const getAllBookings = async () => ({});

export const BookingService = {
	createBooking,
	getUserBookings,
	getBookingById,
	updateBookingStatus,
	getAllBookings,
};
```

---

## Step 17: Booking Route

```ts
// src/app/modules/booking/booking.route.ts
import express from "express";
import { checkAuth } from "../../middlewares/checkAuth";
import { validateRequest } from "../../middlewares/validateRequest";
import { Role } from "../user/user.interface";
import { BookingController } from "./booking.controller";
import { createBookingZodSchema, updateBookingStatusZodSchema } from "./booking.validation";

const router = express.Router();

// POST /api/v1/booking — সবাই booking করতে পারবে
router.post(
	"/",
	checkAuth(...Object.values(Role)),
	validateRequest(createBookingZodSchema),
	BookingController.createBooking,
);

// GET /api/v1/booking — সব booking, শুধু admin
router.get("/", checkAuth(Role.ADMIN, Role.SUPER_ADMIN), BookingController.getAllBookings);

// GET /api/v1/booking/my-bookings — নিজের booking (static, তাই /:bookingId-এর আগে)
router.get("/my-bookings", checkAuth(...Object.values(Role)), BookingController.getUserBookings);

// GET /api/v1/booking/:bookingId
router.get("/:bookingId", checkAuth(...Object.values(Role)), BookingController.getSingleBooking);

// PATCH /api/v1/booking/:bookingId/status
router.patch(
	"/:bookingId/status",
	checkAuth(...Object.values(Role)),
	validateRequest(updateBookingStatusZodSchema),
	BookingController.updateBookingStatus,
);

export const BookingRoutes = router;
```

- **`checkAuth(...Object.values(Role))`**: Role enum-এর সব value spread করে দেওয়া, মানে login করা **যেকোনো** role ঢুকতে পারবে।
- **`/my-bookings` আগে, `/:bookingId` পরে** — নাহলে "my-bookings"-কে bookingId ভেবে নিত ([note 28](./28-get-single-by-slug.md))।

`routes/index.ts`-এ:

```ts
{
	path: "/booking",
	route: BookingRoutes,
},
```

### পরে implement করার সময় যে নিয়মগুলো মানতে হবে

| Route | কে পারবে | Extra check |
|---|---|---|
| `GET /` | শুধু admin | — |
| `GET /my-bookings` | Login করা সবাই | শুধু **নিজের** booking (token-এর userId দিয়ে খোঁজা) |
| `GET /:bookingId` | Login করা সবাই | Booking-টা **নিজের** কিনা (admin হলে সব) |
| `PATCH /:bookingId/status` | Admin আর booking-এর মালিক | নিচে দেখো ⬇️ |

> ⚠️ **Status update-এ সাবধান:** এখন validation যেকোনো status নেয়, আর route-এ সব role ঢুকতে পারে। Implement করার সময় service-এ অবশ্যই check দিতে হবে:
> - সাধারণ user শুধু **`CANCEL`** করতে পারবে, তাও শুধু নিজের booking। নাহলে user নিজেই `COMPLETE` পাঠিয়ে টাকা ছাড়া booking confirm করে ফেলবে।
> - `COMPLETE` হবে **শুধু payment success** থেকে, কারো হাতে না।
> - Booking আগেই `COMPLETE` হয়ে গেলে cancel করতে দেওয়া হবে না (টাকা ফেরতের আলাদা refund process লাগবে)।

---

# Part 5: Payment API

## Step 18: Form Data পড়ার Setup (`app.ts`)

SSLCommerz payment শেষে user-এর browser দিয়ে আমাদের backend-এ **POST** request পাঠায়, আর data পাঠায় **form format**-এ (`val_id=...&tran_id=...`), JSON-এ না। `express.json()` শুধু JSON পড়ে, তাই form data পড়তে এটা লাগবে:

```ts
// src/app.ts
app.use(express.json());
app.use(express.urlencoded({ extended: true })); // ← form data পড়ার জন্য
```

এটা না থাকলে `req.body` খালি থাকবে, `val_id` পাওয়া যাবে না।

---

## Step 19: Payment Service

### ⚠️ সবচেয়ে বড় নিরাপত্তা সমস্যা: Validation ছাড়া Success

Teacher-এর `successPayment` শুধু URL-এর `transactionId` দেখে payment PAID করে দিত। মানে যে কেউ Postman থেকে:

```
POST /api/v1/payment/success?transactionId=tran_xxx
```

পাঠালেই **টাকা না দিয়েই** booking COMPLETE! `transactionId` booking response-এই user পেয়ে যায়, তাই এটা জানা খুব সহজ।

**সমাধান:** SSLCommerz success-এর সময় body-তে একটা **`val_id`** পাঠায়। আমরা এই `val_id` দিয়ে SSLCommerz-এর **Validation API**-কে জিজ্ঞেস করি: "এই payment সত্যিই সফল তো?" SSLCommerz-এর উত্তর থেকে ৩টা জিনিস মিলিয়ে দেখি:

| Check | কেন |
|---|---|
| `status` = `VALID` বা `VALIDATED` | সত্যিই টাকা এসেছে |
| `tran_id` = আমাদের `transactionId` | অন্য কোনো payment-এর `val_id` দিয়ে ঠকানো যাবে না |
| `amount` = DB-র amount | কম টাকা দিয়ে বেশি টাকার booking confirm করা যাবে না |

তিনটা মিললেই PAID, নাহলে FAILED।

### আরেকটা সমস্যা: PAID payment-কে আবার বদলে দেওয়া

Teacher-এর `failPayment`/`cancelPayment` শুধু `transactionId` দিয়ে খুঁজে status বদলাত। কেউ সফল payment-এর `transactionId` দিয়ে `/payment/fail` hit করলে **PAID → FAILED** হয়ে যেত।

**সমাধান:** update-এর সময় শর্ত দিই `status: { $ne: PAID }` — **PAID payment কখনো বদলানো যাবে না।**

### একই code ৩ বার → একটা helper

`successPayment`, `failPayment`, `cancelPayment` — তিনটাতেই একই transaction code, শুধু status আলাদা। তাই একটা helper function `updatePaymentAndBooking` বানিয়েছি, তিনজনই সেটা ব্যবহার করে।

### Code

```ts
// src/app/modules/payment/payment.service.ts
import httpStatus from "http-status-codes";
import AppError from "../../errorHelpers/AppError";
import { BOOKING_STATUS } from "../booking/booking.interface";
import { Booking } from "../booking/booking.model";
import { SSLService } from "../sslCommerz/sslCommerz.service";
import type { IUser } from "../user/user.interface";
import { PAYMENT_STATUS } from "./payment.interface";
import { Payment } from "./payment.model";

/* ───────────── Helper: Payment আর Booking-এর status একসাথে বদলানো ───────────── */
const updatePaymentAndBooking = async (
	transactionId: string,
	paymentStatus: PAYMENT_STATUS,
	bookingStatus: BOOKING_STATUS,
	paymentGatewayData?: unknown,
) => {
	const session = await Booking.startSession();
	session.startTransaction();

	try {
		const updatedPayment = await Payment.findOneAndUpdate(
			// PAID payment আর কখনো বদলানো যাবে না
			{ transactionId, status: { $ne: PAYMENT_STATUS.PAID } },
			{ status: paymentStatus, ...(paymentGatewayData ? { paymentGatewayData } : {}) },
			{ new: true, runValidators: true, session },
		);

		if (!updatedPayment) {
			throw new AppError(httpStatus.NOT_FOUND, "No pending payment found for this transaction.");
		}

		await Booking.findByIdAndUpdate(
			updatedPayment.booking,
			{ status: bookingStatus },
			{ runValidators: true, session },
		);

		await session.commitTransaction();
		return updatedPayment;
	} catch (error) {
		await session.abortTransaction();
		throw error;
	} finally {
		await session.endSession();
	}
};

/* ───────────── আবার Payment link বানানো ───────────── */
const initPayment = async (bookingId: string, userId: string) => {
	const booking = await Booking.findById(bookingId).populate<{ user: IUser }>(
		"user",
		"name email phone address",
	);

	if (!booking) {
		throw new AppError(httpStatus.NOT_FOUND, "Booking not found.");
	}

	// শুধু নিজের booking-এর জন্যই pay করা যাবে
	if (booking.user._id?.toString() !== userId) {
		throw new AppError(httpStatus.FORBIDDEN, "You can only pay for your own booking.");
	}

	const payment = await Payment.findOne({ booking: bookingId });

	if (!payment) {
		throw new AppError(httpStatus.NOT_FOUND, "Payment not found. You have not booked this tour.");
	}
	if (payment.status === PAYMENT_STATUS.PAID) {
		throw new AppError(httpStatus.BAD_REQUEST, "This booking is already paid.");
	}

	const { name, email, phone, address } = booking.user;

	if (!phone || !address) {
		throw new AppError(httpStatus.BAD_REQUEST, "Please update your profile (phone and address) to pay.");
	}

	const paymentUrl = await SSLService.sslPaymentInit({
		name,
		email,
		phoneNumber: phone,
		address,
		amount: payment.amount,
		transactionId: payment.transactionId,
	});

	return { paymentUrl };
};

/* ───────────── Success: আগে validate, তারপর PAID ───────────── */
const successPayment = async (transactionId: string, valId?: string) => {
	const payment = await Payment.findOne({ transactionId });

	if (!payment) {
		throw new AppError(httpStatus.NOT_FOUND, "Payment not found.");
	}

	// আগেই PAID হয়ে থাকলে আবার কিছু করার নেই (একই request দুবার এলেও সমস্যা নেই)
	if (payment.status === PAYMENT_STATUS.PAID) {
		return { success: true, message: "Payment already completed" };
	}

	// SSLCommerz-এর কাছে যাচাই
	const validation = valId ? await SSLService.validatePayment(valId) : null;

	const isValid =
		validation !== null &&
		["VALID", "VALIDATED"].includes(validation.status) &&
		validation.tran_id === transactionId &&
		Number(validation.amount) === payment.amount;

	if (!isValid) {
		await updatePaymentAndBooking(transactionId, PAYMENT_STATUS.FAILED, BOOKING_STATUS.FAILED, validation);
		return { success: false, message: "Payment could not be verified" };
	}

	await updatePaymentAndBooking(transactionId, PAYMENT_STATUS.PAID, BOOKING_STATUS.COMPLETE, validation);
	return { success: true, message: "Payment completed successfully" };
};

/* ───────────── Fail ───────────── */
const failPayment = async (transactionId: string) => {
	await updatePaymentAndBooking(transactionId, PAYMENT_STATUS.FAILED, BOOKING_STATUS.FAILED);
	return { success: false, message: "Payment failed" };
};

/* ───────────── Cancel ───────────── */
const cancelPayment = async (transactionId: string) => {
	await updatePaymentAndBooking(transactionId, PAYMENT_STATUS.CANCELLED, BOOKING_STATUS.CANCEL);
	return { success: false, message: "Payment cancelled" };
};

export const PaymentService = {
	initPayment,
	successPayment,
	failPayment,
	cancelPayment,
};
```

### বোঝার মতো কিছু point

- **`populate<{ user: IUser }>(...)`**: Populate-এর পর `booking.user` আর ObjectId থাকে না, পুরো user object হয়। `<{ user: IUser }>` দিয়ে TypeScript-কে সেটা জানিয়ে দিই। তাই teacher-এর `(booking?.user as any).address` লাগে না।
- **`paymentGatewayData`**: Validation-এর পুরো উত্তর এখানে save হয়। পরে কোনো সমস্যা হলে (refund, অভিযোগ) প্রমাণ হিসেবে কাজে লাগবে।
- **`...(paymentGatewayData ? { paymentGatewayData } : {})`**: data থাকলে update-এ যোগ হয়, না থাকলে (fail/cancel) কিছুই যোগ হয় না।

### Teacher-এর code থেকে যা বদলেছি

| বিষয় | আগে | এখন |
|---|---|---|
| ⚠️ Success-এ validation | ছিল না — যে কেউ free-তে booking confirm করতে পারত | `val_id` দিয়ে SSLCommerz-এ যাচাই |
| ⚠️ PAID বদলানো | `/fail` hit করে PAID → FAILED করা যেত | `status: { $ne: PAID }` |
| ⚠️ `initPayment`-এ populate | ছিল না → `booking.user` শুধু ObjectId, তাই নাম/email/phone সব `undefined` যেত SSLCommerz-এ | `.populate("user", ...)` |
| ⚠️ `initPayment`-এ auth | Route-এ checkAuth ছিল না, যে কেউ অন্যের booking-এর link বানাতে পারত | checkAuth + নিজের booking কিনা check |
| PAID booking-এ আবার pay | আটকানো ছিল না | "already paid" error |
| Payment না পেলে | `updatedPayment?.booking` → চুপচাপ কিছুই হতো না | 404 error |
| একই code ৩ বার | success / fail / cancel আলাদা | একটা helper |
| `cancelPayment`-এ `new: true` | ছিল না (বাকি দুটোয় ছিল) | helper-এ একই রকম |

---

## Step 20: Payment Controller

```ts
// src/app/modules/payment/payment.controller.ts
import type { Request, Response } from "express";
import httpStatus from "http-status-codes";
import type { JwtPayload } from "jsonwebtoken";
import { envVars } from "../../config/env";
import { catchAsync } from "../../utils/catchAsync";
import { sendResponse } from "../../utils/sendResponse";
import { PaymentService } from "./payment.service";

// frontend-এ redirect — URLSearchParams space বা special character ঠিকভাবে encode করে
const redirectToFrontend = (res: Response, baseUrl: string, transactionId: string, message: string) => {
	const params = new URLSearchParams({ transactionId, message });
	res.redirect(`${baseUrl}?${params.toString()}`);
};

const initPayment = catchAsync(async (req: Request, res: Response) => {
	const bookingId = req.params.bookingId as string;
	const decodedToken = req.user as JwtPayload;

	const result = await PaymentService.initPayment(bookingId, decodedToken.userId as string);

	sendResponse(res, {
		statusCode: httpStatus.OK,
		success: true,
		message: "Payment link created successfully",
		data: result,
	});
});

const successPayment = catchAsync(async (req: Request, res: Response) => {
	const transactionId = req.query.transactionId as string;
	const valId = (req.body as { val_id?: string }).val_id; // SSLCommerz form data-তে পাঠায়

	const result = await PaymentService.successPayment(transactionId, valId);

	// যাচাই সফল হলে success page, নাহলে fail page
	redirectToFrontend(
		res,
		result.success ? envVars.SSL.SSL_SUCCESS_FRONTEND_URL : envVars.SSL.SSL_FAIL_FRONTEND_URL,
		transactionId,
		result.message,
	);
});

const failPayment = catchAsync(async (req: Request, res: Response) => {
	const transactionId = req.query.transactionId as string;
	const result = await PaymentService.failPayment(transactionId);

	redirectToFrontend(res, envVars.SSL.SSL_FAIL_FRONTEND_URL, transactionId, result.message);
});

const cancelPayment = catchAsync(async (req: Request, res: Response) => {
	const transactionId = req.query.transactionId as string;
	const result = await PaymentService.cancelPayment(transactionId);

	redirectToFrontend(res, envVars.SSL.SSL_CANCEL_FRONTEND_URL, transactionId, result.message);
});

export const PaymentController = {
	initPayment,
	successPayment,
	failPayment,
	cancelPayment,
};
```

### Teacher-এর code থেকে যা বদলেছি

- **`initPayment`-এর message আর status:** `201 "Payment done successfully"` ছিল, অথচ এখানে কোনো payment হয়নি, শুধু link তৈরি হয়েছে। তাই `200 "Payment link created successfully"`।
- **`if (result.success) res.redirect(...)`:** শর্ত না মিললে কোনো response-ই যেত না, browser loading হতেই থাকত। এখন সবসময় redirect হয়।
- **URL encode:** `message=Payment Completed Successfully` — URL-এ space থাকলে ভেঙে যেতে পারে। `URLSearchParams` নিজেই encode করে (`Payment+Completed+Successfully`)।
- **একই redirect code ৩ বার** → `redirectToFrontend` helper।

---

## Step 21: Payment Route

> ✏️ Note-এ `payment.route.ts`-এর code ছিল না (ভুল করে কেটে গেছে)। এখানে পুরোটা দিলাম।

```ts
// src/app/modules/payment/payment.route.ts
import express from "express";
import { checkAuth } from "../../middlewares/checkAuth";
import { Role } from "../user/user.interface";
import { PaymentController } from "./payment.controller";

const router = express.Router();

// User নিজে call করবে — login লাগবে
router.post("/init-payment/:bookingId", checkAuth(...Object.values(Role)), PaymentController.initPayment);

// SSLCommerz call করবে — এখানে checkAuth দেওয়া যাবে না (SSLCommerz-এর কাছে আমাদের token নেই)
router.post("/success", PaymentController.successPayment);
router.post("/fail", PaymentController.failPayment);
router.post("/cancel", PaymentController.cancelPayment);

export const PaymentRoutes = router;
```

- **`POST` কেন, `GET` না?** SSLCommerz success/fail/cancel URL-এ **POST** request পাঠায় (form data সহ)।
- **Success route-এ checkAuth নেই**, তাই নিরাপত্তা আসে **validation** থেকে (Step 19)। এজন্যই validation এত জরুরি।

`routes/index.ts`-এ:

```ts
{
	path: "/payment",
	route: PaymentRoutes,
},
```

---

## Step 22: আবার Pay করার সুযোগ (Retry Payment)

### Scenario

User SSLCommerz page-এ গিয়ে ভাবল, "এখন না, একটু পরে pay করি" — cancel করে বের হয়ে গেল। পরে pay করতে চাইল, কিন্তু আগের payment link আর কাজ করে না (PIN দেওয়ার পর error)।

### সমাধান

Payment link শুধু booking-এর সময় না দিয়ে, **booking id দিয়ে আলাদা একটা API**-তে যেকোনো সময় নতুন link বানানো যায়:

```
POST /api/v1/payment/init-payment/687accd3aca0f50b75cb889
Authorization: <user token>
```

```json
{
	"success": true,
	"message": "Payment link created successfully",
	"data": { "paymentUrl": "https://sandbox.sslcommerz.com/EasyCheckOut/..." }
}
```

Frontend এই `paymentUrl`-এ user-কে পাঠাবে। এবার pay করলে আবার সেই success flow চলবে।

```
Booking (CANCEL) + Payment (CANCELLED)
   │  user "Pay Now" চাপল
   ▼
POST /payment/init-payment/:bookingId → নতুন paymentUrl
   │
   ▼
SSLCommerz page → সফল → /payment/success → validate
   │
   ▼
Booking (COMPLETE) + Payment (PAID) ✅
```

`status: { $ne: PAID }` শর্তের কারণে CANCELLED বা FAILED payment আবার PAID হতে পারে, কিন্তু PAID কখনো উল্টো দিকে যায় না।

---

# Part 6: IPN — পরে (Deploy-এর সময়)

SSLCommerz-এর store settings-এ (`Merchant Panel → Store → Edit`) একটা **IPN URL** দেওয়ার জায়গা আছে।

**IPN কী?** Payment সফল হলে SSLCommerz **নিজে** (user-এর browser ছাড়াই) আমাদের server-এর ওই URL-এ একটা POST request পাঠায়।

**কেন দরকার?** ধরো user pay করল, কিন্তু success page-এ redirect হওয়ার আগেই internet চলে গেল বা browser বন্ধ করে দিল। তখন `/payment/success` কখনো hit হবে না — টাকা কাটা গেছে কিন্তু booking PENDING! IPN এই ক্ষেত্রেও server-কে জানিয়ে দেয়।

**এখন কেন না?** IPN-এ SSLCommerz-এর server আমাদের server-কে call করে, আর সে `localhost` খুঁজে পায় না। তাই deploy করা live URL লাগবে।

> 📌 Teacher যখন IPN করাবেন, এই section-এ যোগ করব। IPN handler-ও একইভাবে `val_id` দিয়ে validate করে `updatePaymentAndBooking` call করবে — helper আগেই তৈরি আছে।

---

## সারাংশ

| বিষয় | মনে রাখার কথা |
|---|---|
| Booking | PENDING দিয়ে শুরু; user token থেকে, client body থেকে না |
| Payment | Booking-এর সাথেই তৈরি (UNPAID bill), `transactionId` unique |
| Two-way ref | Booking ↔ Payment, দুই দিক থেকেই এক click-এ |
| Transaction | একাধিক collection-এ write → সব হবে নয়তো কিছুই না; replica set লাগে |
| Session | প্রতিটা operation-এ `{ session }`; catch-এ abort; finally-তে endSession |
| SSLCommerz | Hosted page; init → success/fail/cancel (POST) → frontend redirect |
| Form data | `x-www-form-urlencoded` পাঠানো; পড়তে `express.urlencoded()` |
| ⚠️ Validation | `val_id` দিয়ে যাচাই ছাড়া কখনো PAID না |
| ⚠️ PAID | `$ne: PAID` — সফল payment কেউ বদলাতে পারবে না |
| Retry | `POST /payment/init-payment/:bookingId` |
| IPN | Deploy-এর পরে; browser বন্ধ হলেও payment ধরা পড়ে |