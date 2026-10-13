# 37 — SSLCommerz IPN দিয়ে Payment Validate করা

[Note 28](./28-booking-payment-sslcommerz.md)-এর শেষে IPN-এর জায়গা রেখে দিয়েছিলাম ("deploy-এর সময় করব")। এবার সেটা।

SSLCommerz documentation: https://developer.sslcommerz.com/doc/v4/#validate-payment-with-ipn

---

## Step 1: IPN কী, আর কেন দরকার?

**IPN** = **I**nstant **P**ayment **N**otification। Payment শেষ হলে SSLCommerz-এর server **নিজে**, user-এর browser ছাড়াই, আমাদের server-এর একটা URL-এ POST request পাঠিয়ে জানায়: "এই transaction-এর payment হয়েছে।"

Documentation-এ প্রথমেই বলছে "grab the notification" — মানে আমাদের একটা route বানাতে হবে, যেটা SSLCommerz-এর পাঠানো এই POST request ধরবে (grab) আর তার body পড়বে।

### Success route থাকতে IPN কেন?

এখন আমাদের success route আছে (`/payment/success`)। কিন্তু সেটা চলে **user-এর browser-এর মাধ্যমে**:

```
User pay করল → SSLCommerz user-এর browser-কে বলে "/payment/success-এ যাও"
             → browser আমাদের backend-এ আসে → PAID
```

ধরো user pay করল, টাকা কেটে গেল, কিন্তু redirect হওয়ার ঠিক আগে:
- Internet চলে গেল
- ভুল করে browser tab বন্ধ করে দিল
- Phone-এর battery শেষ

তাহলে browser কখনো `/payment/success`-এ পৌঁছায় না। ফলে **টাকা কাটা গেছে, কিন্তু DB-তে payment এখনো UNPAID, booking PENDING।**

IPN এই সমস্যা মেটায়, কারণ এটা browser-এর উপর নির্ভর করে না — SSLCommerz-এর server সরাসরি আমাদের server-কে জানায়:

```
                    ┌──→ ① Browser redirect → /payment/success   (user-এর browser দিয়ে, মাঝপথে হারাতে পারে)
User pay করল ──────┤
                    └──→ ② IPN → /payment/validate-payment       (SSLCommerz server থেকে সরাসরি, browser লাগে না)
```

দুটো পথই একই কাজ করে: যাচাই করে PAID করা। **যেটা আগে পৌঁছায়, সেটা কাজ করে; পরেরটা দেখে "আগেই PAID" — কিছু করে না।** তাই দুটো একসাথে এলেও booking দুবার confirm হয় না, invoice-ও দুবার যায় না।

### ⚠️ IPN-এর body বিশ্বাস করা যাবে না

IPN route-এ `checkAuth` দেওয়া যায় না, কারণ SSLCommerz-এর কাছে আমাদের token নেই। মানে **যে কেউ** এই URL-এ POST পাঠাতে পারে:

```json
{ "tran_id": "tran_xxx", "status": "VALID", "amount": "5000" }
```

এই body দেখে PAID করে দিলে কেউ টাকা না দিয়েই booking confirm করে ফেলবে। তাই body থেকে শুধু `val_id` নিই, তারপর SSLCommerz-এর **Validation API**-কে জিজ্ঞেস করি: "এই val_id কি সত্যিই সফল payment?" SSLCommerz যা বলবে, শুধু সেটাই বিশ্বাস করব। এটা note 28-এর success route-এর মতোই।

---

## Step 2: Environment Variable

```bash
# .env — SSLCommerz-এর অংশে
SSL_IPN_URL="http://localhost:5000/api/v1/payment/validate-payment"
```

### `env.ts` — ৩ জায়গায় (SSL object-এর ভেতরে)

```ts
// ① interface
SSL: {
	// ... আগের সব
	SSL_IPN_URL: string;
};

// ② requiredEnvVariables
"SSL_IPN_URL",

// ③ return object
SSL: {
	// ... আগের সব
	SSL_IPN_URL: process.env.SSL_IPN_URL as string,
},
```

### ⚠️ `localhost` দিয়ে IPN কাজ করবে না

IPN-এ **SSLCommerz-এর server** আমাদের server-কে call করে। SSLCommerz-এর server internet-এ আছে। সে `localhost:5000` খুঁজলে নিজের computer-এর localhost-এ খুঁজবে, তোমার computer-এ না। তাই `localhost` দেওয়া URL-এ IPN **কখনো পৌঁছাবে না**।

| অবস্থা | IPN URL |
|---|---|
| Deploy করার পর | `https://ph-tour-api.vercel.app/api/v1/payment/validate-payment` ✅ |
| Local-এ test করতে | **ngrok**-এর মতো tunnel দিয়ে localhost-কে সাময়িক public URL বানানো (`ngrok http 5000` → `https://abc123.ngrok-free.app`) |
| `http://localhost:5000/...` | ❌ SSLCommerz পৌঁছাতে পারবে না |

Local-এ `localhost` রেখে দিলেও কোনো সমস্যা নেই — শুধু IPN আসবে না, success route আগের মতোই কাজ করবে।

> 💡 SSLCommerz merchant panel-এর store settings-এও (`IPN URL (HTTP/HTTPS)`) একটা IPN URL দেওয়া যায়। আমরা প্রতিটা payment-এর সাথে `ipn_url` পাঠাচ্ছি, তাই panel-এ না দিলেও চলবে।

---

## Step 3: Payment Initialize-এ `ipn_url` যোগ

Note 28-এর `sslPaymentInit`-এ শুধু **একটা লাইন** যোগ হবে:

```ts
// src/app/modules/sslCommerz/sslCommerz.service.ts
const data = {
	store_id: envVars.SSL.STORE_ID,
	store_passwd: envVars.SSL.STORE_PASS,
	total_amount: payload.amount,
	currency: "BDT",
	tran_id: payload.transactionId,
	success_url: `${envVars.SSL.SSL_SUCCESS_BACKEND_URL}?transactionId=${payload.transactionId}`,
	fail_url: `${envVars.SSL.SSL_FAIL_BACKEND_URL}?transactionId=${payload.transactionId}`,
	cancel_url: `${envVars.SSL.SSL_CANCEL_BACKEND_URL}?transactionId=${payload.transactionId}`,
	ipn_url: envVars.SSL.SSL_IPN_URL, // ← নতুন: payment শেষে SSLCommerz এখানে নিজে জানাবে
	// ... বাকি সব আগের মতো
};
```

এতে SSLCommerz জানে, এই payment শেষ হলে কোন URL-এ notification পাঠাতে হবে।

> ✏️ **খেয়াল করো:** তুমি যে `sslPaymentInit` দিয়েছ সেটা teacher-এর **পুরোনো** version। Note 28-এ আমরা এটা আগেই ঠিক করেছি: URL-এ শুধু `transactionId` (`amount`/`status` না), শুধু `GatewayPageURL` return, SSLCommerz "FAILED" দিলে error, `ship_postcode` string, ইত্যাদি। তাই পুরো function আবার লিখছি না — note 28-এর version-এ শুধু উপরের `ipn_url` লাইনটা যোগ করো।

### ❓ `Content-Type: "application/x-www-form-urlencoded"` কেন লাগল?

এটা [note 28](./28-booking-payment-sslcommerz.md)-এ বিস্তারিত আছে। সংক্ষেপে: SSLCommerz-এর API **form format**-এ data চায় (`store_id=abc&total_amount=5000`), JSON-এ না। Header দিয়ে বলে দিই "form format-এ পাঠাচ্ছি", আর axios সেটা দেখে নিজেই object-কে ওই format-এ রূপান্তর করে পাঠায়।

IPN-এও SSLCommerz আমাদের কাছে **form format**-এই data পাঠায়। তাই সেটা পড়তে `app.ts`-এ `express.urlencoded({ extended: true })` থাকতে হবে, যেটা note 28-এ আগেই যোগ করেছি।

---

## Step 4: SSLCommerz IPN-এ কী পাঠায়

SSLCommerz IPN route-এ form data হিসেবে অনেকগুলো field পাঠায়। আমাদের দরকারি কয়েকটা:

| Field | মানে | উদাহরণ |
|---|---|---|
| `status` | Payment-এর অবস্থা | `VALID`, `FAILED`, `CANCELLED`, `UNATTEMPTED`, `EXPIRED` |
| `tran_id` | আমাদের দেওয়া transaction id | `tran_3f9a1c2e-...` |
| `val_id` | SSLCommerz-এর validation id — **এটা দিয়ে যাচাই করব** | `2410131234567abcDEF` |
| `amount` | কত টাকা | `"5000.00"` |
| `card_type`, `bank_tran_id` ... | Card আর bank-এর তথ্য | |

```ts
// src/app/modules/sslCommerz/sslCommerz.interface.ts
export interface ISSLIpnBody {
	status?: string;
	tran_id?: string;
	val_id?: string;
	amount?: string;
}
```

সব field `?` (optional), কারণ body বাইরে থেকে আসছে — কিছু না থাকতেও পারে, কেউ ভুয়া request-ও পাঠাতে পারে।

---

## Step 5: Teacher-এর `validatePayment` — কী সমস্যা

```ts
const validatePayment = async (payload: any) => {
	try {
		const response = await axios({
			method: "GET",
			url: `${envVars.SSL.SSL_VALIDATION_API}?val_id=${payload.val_id}&store_id=...&store_passwd=...`,
		});

		await Payment.updateOne(
			{ transactionId: payload.tran_id },
			{ paymentGatewayData: response.data },
			{ runValidators: true },
		);
	} catch (error: any) {
		throw new AppError(401, `Payment Validation Error, ${error.message}`);
	}
};
```

এটা SSLCommerz-কে জিজ্ঞেস করে ঠিকই, কিন্তু উত্তর **দেখে না**:

| সমস্যা | কী হয় |
|---|---|
| ⚠️ উত্তরের `status` দেখে না | SSLCommerz "INVALID_TRANSACTION" বললেও সেটা save করে চুপচাপ চলে যায় |
| ⚠️ Payment PAID করে না | IPN-এর আসল উদ্দেশ্যই এটা (browser বন্ধ হলেও PAID) — অথচ শুধু `paymentGatewayData` save হচ্ছে, status যেমন ছিল তেমনই |
| ⚠️ `tran_id` মেলায় না | Body-র `tran_id` বিশ্বাস করে save করে। কেউ নিজের ছোট একটা সফল payment-এর `val_id` আর **অন্যের** `tran_id` পাঠালে, অন্যের payment-এ ভুল gateway data বসে যায় |
| ⚠️ `amount` মেলায় না | কম টাকার payment দিয়ে বেশি টাকার booking-এর data লেখা যায় |
| URL হাতে বানানো | `val_id`-এ `&` বা special character থাকলে URL ভেঙে যায়; axios-এর `params` নিজেই encode করে |
| `401` | 401 মানে login সমস্যা — এখানে সেটা না |
| SSLCommerz module DB ছোঁয় | `sslCommerz` module-এর কাজ শুধু SSLCommerz-এর সাথে কথা বলা। DB update করা `payment` module-এর কাজ |

**ভালো খবর:** এগুলোর প্রায় সব সমাধান আমরা note 28-এ আগেই বানিয়ে রেখেছি:

- `SSLService.validatePayment(valId)` — শুধু SSLCommerz-কে জিজ্ঞেস করে উত্তর return করে (params সহ, DB ছোঁয় না)
- `PaymentService.successPayment(transactionId, valId)` — validate করে, **status + tran_id + amount** তিনটা মিলিয়ে PAID করে, আগেই PAID থাকলে কিছু করে না, আর commit-এর পরে invoice পাঠায় ([note 34](./34-pdf-invoice-pdfkit.md))

তাই IPN-এর জন্য নতুন করে validation logic লেখার দরকার নেই — **একই function আবার ব্যবহার করব।**

---

## Step 6: IPN Handler (Payment Service)

```ts
// src/app/modules/payment/payment.service.ts
import type { ISSLIpnBody } from "../sslCommerz/sslCommerz.interface";

/* ───────────── IPN: SSLCommerz server থেকে সরাসরি notification ───────────── */
const handleIpnNotification = async (body: ISSLIpnBody) => {
	const { tran_id: transactionId, val_id: valId, status } = body;

	if (!transactionId) {
		throw new AppError(httpStatus.BAD_REQUEST, "tran_id is missing in IPN.");
	}

	const payment = await Payment.findOne({ transactionId }).select("status");

	if (!payment) {
		throw new AppError(httpStatus.NOT_FOUND, "Payment not found.");
	}

	// success route আগেই PAID করে ফেলেছে → আর কিছু করার নেই
	if (payment.status === PAYMENT_STATUS.PAID) {
		return { message: "Payment already completed" };
	}

	// body-র status শুধু কোন পথে যাব সেটা ঠিক করে;
	// আসল যাচাই successPayment-এর ভেতরে SSLCommerz Validation API দিয়ে হয়
	if (status === "VALID" || status === "VALIDATED") {
		return successPayment(transactionId, valId);
	}
	if (status === "FAILED") {
		return failPayment(transactionId);
	}
	if (status === "CANCELLED") {
		return cancelPayment(transactionId);
	}

	// UNATTEMPTED, EXPIRED বা অচেনা কিছু → কিছু বদলাব না
	return { message: `IPN ignored for status: ${status ?? "unknown"}` };
};

export const PaymentService = {
	initPayment,
	successPayment,
	failPayment,
	cancelPayment,
	handleIpnNotification, // ← নতুন
	getInvoiceDownloadUrl,
};
```

### ধাপে ধাপে

```
IPN এলো { tran_id, val_id, status }
   │
   ├─ tran_id নেই?                  → 400
   ├─ এই tran_id-এর payment নেই?     → 404
   ├─ আগেই PAID?                    → "already completed" (success route আগে পৌঁছেছে)
   │
   ├─ status VALID / VALIDATED  → successPayment(tran_id, val_id)
   │                                 ├─ SSLCommerz-এর কাছে val_id যাচাই
   │                                 ├─ status + tran_id + amount মিলল? → PAID + COMPLETE + invoice ✅
   │                                 └─ না মিললে → FAILED
   ├─ status FAILED             → failPayment
   ├─ status CANCELLED          → cancelPayment
   └─ অন্য কিছু                 → কিছু বদলাই না
```

### দুটো পথ একসাথে এলে কী হয়?

| কোনটা আগে | প্রথমটা | দ্বিতীয়টা |
|---|---|---|
| Success route আগে | validate → PAID → invoice | IPN দেখে "already PAID" → কিছু করে না |
| IPN আগে | validate → PAID → invoice | Success route দেখে "already completed" → শুধু success page-এ redirect |
| শুধু IPN (browser বন্ধ) | validate → PAID → invoice ✅ | — |

তিন ক্ষেত্রেই booking একবারই confirm হয়, invoice একবারই যায়। একই কাজ বারবার হলেও ফল একই থাকা — একে বলে **idempotent**। Payment-এর মতো জায়গায় এটা খুব জরুরি, কারণ SSLCommerz কখনো কখনো একই IPN একাধিকবার পাঠায়।

> 💡 খুব কাছাকাছি সময়ে দুটো এলে দুজনেই "এখনো PAID না" দেখে এগোতে পারে। তবু ভয় নেই: note 28-এর `updatePaymentAndBooking`-এ শর্ত আছে `status: { $ne: PAID }` — যে আগে update করবে সে-ই PAID করবে, দ্বিতীয়জন "No pending payment" error পাবে, কিছুই দুবার হবে না।

---

## Step 7: Controller

```ts
// src/app/modules/payment/payment.controller.ts
import type { ISSLIpnBody } from "../sslCommerz/sslCommerz.interface";

const validatePayment = catchAsync(async (req: Request, res: Response) => {
	const result = await PaymentService.handleIpnNotification(req.body as ISSLIpnBody);

	sendResponse(res, {
		statusCode: httpStatus.OK,
		success: true,
		message: result.message,
		data: null,
	});
});

export const PaymentController = {
	// ... আগের সব
	validatePayment,
};
```

- এই response user কখনো দেখে না — পায় SSLCommerz-এর server। তার শুধু জানা দরকার "পেয়েছি" (`200`)।
- Success route-এর মতো এখানে **redirect নেই**, কারণ ওপাশে কোনো browser নেই।
- ✏️ `console.log("sslcommerz ipn url body", req.body)` সরিয়েছি (ESLint `no-console`)। Test করার সময় দেখতে চাইলে সাময়িক রাখতে পারো, কাজ শেষে মুছে দিও।
- ✏️ `200` → `httpStatus.OK`।

---

## Step 8: Route

```ts
// src/app/modules/payment/payment.route.ts

// SSLCommerz-এর server call করে — checkAuth দেওয়া যাবে না
// নিরাপত্তা আসে service-এর ভেতরে SSLCommerz Validation API থেকে
router.post("/validate-payment", PaymentController.validatePayment);
```

---

## Step 9: SSLCommerz Service-এ কিছু বদলাতে হবে?

না। Teacher-এর `SSLService.validatePayment(payload)`-এর বদলে note 28-এর version-টাই থাকবে:

```ts
// src/app/modules/sslCommerz/sslCommerz.service.ts (note 28 থেকে, অপরিবর্তিত)
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
```

| Teacher | এখানে |
|---|---|
| URL string জোড়া লাগিয়ে বানানো | `params` — axios নিজে `?val_id=...&store_id=...` বানায় আর encode করে |
| `Payment` model import করে DB update | শুধু উত্তর return করে; কী করব সেটা payment service ঠিক করে |
| উত্তর দেখে না | Caller (`successPayment`) status, tran_id, amount মিলিয়ে দেখে |
| `any`, `401` | Type সহ, `502 BAD_GATEWAY` (বাইরের service-এ সমস্যা) |

> 📝 `sslCommerz` module এখন DB সম্পর্কে কিছুই জানে না, আর `payment` module জানে না SSLCommerz-এর URL বা password কী। প্রতিটা module শুধু নিজের কাজ করে। ভবিষ্যতে অন্য payment gateway (bKash, Stripe) যোগ করলে শুধু নতুন একটা gateway module লাগবে, payment-এর logic একই থাকবে।

---

## Step 10: Test

### Local-এ (ngrok দিয়ে)

1. `ngrok http 5000` চালাও → `https://abc123.ngrok-free.app`-এর মতো একটা URL পাবে।
2. `.env`-এ: `SSL_IPN_URL="https://abc123.ngrok-free.app/api/v1/payment/validate-payment"` → server restart।
3. Booking করে SSLCommerz sandbox-এ pay করো।
4. **IPN-কে আলাদা করে test করতে:** pay করার পর success page-এ redirect হওয়ার **আগেই tab বন্ধ করে দাও**।
5. কিছুক্ষণ পর DB দেখো → payment **PAID**, booking **COMPLETE**, invoice email এসেছে। ✅ (Success route চলেনি, তবুও IPN দিয়ে কাজ হয়েছে।)

### Postman দিয়ে নিরাপত্তা test

```
POST localhost:5000/api/v1/payment/validate-payment
Body (x-www-form-urlencoded):
  tran_id = <কোনো UNPAID payment-এর transactionId>
  status  = VALID
  val_id  = fake123
```

→ SSLCommerz `fake123`-কে valid বলবে না → payment **PAID হবে না** ✅। (Teacher-এর code-এ এখানে ভুয়া data save হয়ে যেত।)

| Request | Expected |
|---|---|
| সঠিক IPN (আসল val_id) | PAID + COMPLETE + invoice |
| একই IPN দ্বিতীয়বার | "Payment already completed", কিছু বদলায় না |
| ভুয়া `val_id` | PAID হয় না |
| `tran_id` ছাড়া | 400 |
| অচেনা `tran_id` | 404 |
| `status: "EXPIRED"` | কিছু বদলায় না |

---

## সারাংশ

| বিষয় | মনে রাখার কথা |
|---|---|
| IPN | SSLCommerz-এর server থেকে সরাসরি notification — browser বন্ধ হলেও payment ধরা পড়ে |
| Success route vs IPN | দুটোই রাখো; যেটা আগে আসে কাজ করে, পরেরটা "already PAID" |
| `ipn_url` | `sslPaymentInit`-এ এক লাইন; env-এ `SSL_IPN_URL` |
| localhost | IPN পৌঁছাবে না — deploy বা ngrok লাগবে |
| ⚠️ Body বিশ্বাস না | শুধু `val_id` নিয়ে Validation API-তে যাচাই; status + tran_id + amount মেলাতে হবে |
| Reuse | IPN নতুন logic লেখে না — note 28-এর `successPayment` ব্যবহার করে |
| Idempotent | একই IPN দশবার এলেও একবারই PAID, একবারই invoice |
| Module ভাগ | `sslCommerz` = gateway-র সাথে কথা; `payment` = DB আর ব্যবসার নিয়ম |
| Route | Public (`checkAuth` না), নিরাপত্তা আসে validation থেকে |