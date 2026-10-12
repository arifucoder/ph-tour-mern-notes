# 36 — Payment-এর পর PDF Invoice (PDFKit + Cloudinary + Email)

## কী বানাব?

User সফলভাবে payment করার পর দুটো কাজ:

1. **PDF invoice বানিয়ে email-এ attachment হিসেবে পাঠানো।**
2. **PDF-টা Cloudinary-তে রেখে URL DB-তে save করা**, যাতে user পরে যেকোনো সময় download করতে পারে।

`payment.interface.ts`-এ আগেই `invoiceUrl?: string` রেখেছিলাম ([note 30](./30-booking-payment-sslcommerz.md)) — এবার সেটার কাজ।

## পুরো Flow

```
SSLCommerz → POST /payment/success
   │
   ▼
successPayment
   ├─ ① SSLCommerz-এর কাছে validate (note 30)
   ├─ ② Transaction: Payment PAID + Booking COMPLETE → commit ✅
   │        ── টাকা নেওয়া শেষ, এখান থেকে আর কিছুই উল্টাবে না ──
   └─ ③ Invoice (transaction-এর বাইরে, fail করলেও payment PAID-ই থাকবে)
          ├─ booking + tour + user-এর তথ্য আনো
          ├─ PDFKit দিয়ে PDF বানাও (memory-তে, Buffer হিসেবে)
          ├─ Buffer → Cloudinary (upload_stream) → invoiceUrl
          ├─ Payment-এ invoiceUrl save
          └─ Email: invoice.pdf attachment + download link
   ▼
redirect → frontend /payment/success

পরে: GET /payment/invoice/:paymentId → শুধু মালিক বা admin → invoiceUrl
```

## যে file-গুলো ছুঁব

```
src/app/
├── config/cloudinary.config.ts        ← uploadBufferToCloudinary যোগ
├── utils/
│   ├── invoice.ts                     ← নতুন (generatePdf)
│   ├── sendEmail.ts                   ← template নাম "invoice" যোগ
│   └── templates/invoice.ejs          ← নতুন
└── modules/
    ├── booking/booking.interface.ts   ← createdAt type যোগ
    └── payment/
        ├── payment.service.ts         ← invoice + download
        ├── payment.controller.ts
        └── payment.route.ts
```

---

# Part 1: PDF বানানো (PDFKit)

## Step 1: Install

```bash
npm i pdfkit
npm i -D @types/pdfkit
```

**PDFKit** দিয়ে code লিখে নিজের মতো PDF বানানো যায় — text, font size, alignment, line, image, table সবই। Docs: https://pdfkit.org/

> `.npmrc`-এ `legacy-peer-deps=true` দেওয়া থাকলে ([note 34](./34-forgot-reset-password-nodemailer.md)) আলাদা flag লাগবে না।

## Step 2: Buffer, Stream আর Chunk — আগে ধারণা

PDF বানিয়ে আমরা disk-এ কোনো file save করব না। পুরোটা **memory-তে** থাকবে, তারপর সরাসরি Cloudinary আর email-এ যাবে। এর জন্য তিনটা শব্দ বুঝতে হবে:

| শব্দ | মানে | উদাহরণ |
|---|---|---|
| **Binary data** | File-এর আসল ভেতরের রূপ — byte-এর লম্বা ধারা (প্রতিটা byte 0–255) | PDF, image, video |
| **Stream** | Data একবারে না দিয়ে **টুকরো টুকরো** করে পাঠানোর ব্যবস্থা | YouTube video যেভাবে একটু একটু করে load হয় |
| **Chunk** | Stream-এর একেকটা টুকরো | |
| **Buffer** | Memory-তে রাখা binary data — Node.js-এর built-in class | পুরো PDF এক টুকরোয় |

PDFKit-এর document আসলে একটা **stream**: আমরা content লিখতে থাকি, আর সে PDF-এর byte গুলো chunk আকারে বের করতে থাকে। আমাদের কাজ: সব chunk একটা array-তে জমানো, শেষে জোড়া লাগিয়ে একটা Buffer বানানো।

```
doc.text(...) doc.text(...) ... doc.end()
        │
        ▼  "data" event (বারবার)
   [chunk1] [chunk2] [chunk3] ...  → chunks array-তে push
        │
        ▼  "end" event (একবার)
   Buffer.concat(chunks) → পুরো PDF (একটা Buffer) → resolve
```

## Step 3: `generatePdf` Function

```ts
// src/app/utils/invoice.ts
import PDFDocument from "pdfkit";

export interface IInvoiceData {
	transactionId: string;
	bookingDate: Date;
	userName: string;
	tourTitle: string;
	guestCount: number;
	totalAmount: number;
}

// 12 October 2026
const formatDate = (date: Date) =>
	date.toLocaleDateString("en-GB", { day: "numeric", month: "long", year: "numeric" });

export const generatePdf = (invoiceData: IInvoiceData): Promise<Buffer> => {
	return new Promise((resolve, reject) => {
		const doc = new PDFDocument({ size: "A4", margin: 50 });
		const chunks: Buffer[] = [];

		// PDFKit একটা stream — তার event শুনে data জমাই
		doc.on("data", (chunk: Buffer) => chunks.push(chunk)); // প্রতিটা টুকরো জমাও
		doc.on("end", () => resolve(Buffer.concat(chunks))); // শেষ হলে জোড়া লাগিয়ে দাও
		doc.on("error", reject); // কোনো সমস্যা হলে Promise reject

		// ── PDF-এর content ──
		doc.fontSize(20).text("Invoice", { align: "center" });
		doc.moveDown();

		doc.fontSize(14).text(`Transaction ID: ${invoiceData.transactionId}`);
		doc.text(`Booking Date: ${formatDate(invoiceData.bookingDate)}`);
		doc.text(`Customer: ${invoiceData.userName}`);
		doc.moveDown();

		doc.text(`Tour: ${invoiceData.tourTitle}`);
		doc.text(`Guests: ${invoiceData.guestCount}`);
		doc.text(`Total Amount: BDT ${invoiceData.totalAmount.toFixed(2)}`);
		doc.moveDown();

		doc.text("Thank you for booking with us!", { align: "center" });

		doc.end(); // লেখা শেষ → "end" event চালু হবে
	});
};
```

### প্রতিটা অংশ

- **`new Promise((resolve, reject) => ...)`**: PDF তৈরি হয় event দিয়ে (async), কিন্তু আমরা চাই বাইরে থেকে শুধু `await generatePdf(data)` লিখতে। তাই event-ভিত্তিক কাজকে Promise-এ মুড়ে দিই — `end` হলে `resolve`, `error` হলে `reject`।
- **`doc.on(...)`**: এটা "guideline-এ আছে বলে" না — PDFKit document একটা Node.js **stream**, আর stream থেকে data পাওয়ার নিয়মই হলো তার event শোনা (`data`, `end`, `error`)।
- **`Buffer.concat(chunks)`**: অনেকগুলো ছোট Buffer-কে একটা বড় Buffer বানায়।
- **`doc.moveDown()`**: এক লাইন নিচে নামা — Enter চাপার মতো, অনেকটা HTML-এর `<br>`।
- **`doc.end()`**: "লেখা শেষ" — এটা না দিলে `end` event কখনো আসবে না, Promise চিরকাল অপেক্ষা করবে।

> ✏️ **সংশোধন:** Note-এ লেখা ছিল "`Uint8Array[]` হলো JS-এর type"। `Uint8Array` হলো JavaScript-এর একটা **typed array** — byte (0–255)-এর array, binary data রাখার জন্য। Node.js-এর `Buffer` আসলে `Uint8Array`-এরই একটা উন্নত রূপ (subclass)। PDFKit-এর chunk-গুলো Buffer, তাই `Buffer[]` লেখাই বেশি সঠিক।

### Teacher-এর code থেকে যা বদলেছি

| বিষয় | আগে | এখন | কেন |
|---|---|---|---|
| ⚠️ `try/catch` | `async` + `try { return new Promise(...) } catch` | `try/catch` সরানো, শুধু `reject` | Promise-এর ভেতরে পরে (event-এ) যে error হয়, বাইরের `try/catch` সেটা **কখনো ধরতে পারে না** — ততক্ষণে `try` block শেষ। Error ঠিকমতো যায় `reject` দিয়েই। তাই ওই catch অকেজো code ছিল। |
| AppError | `401` + `console.log` | (সরানো) | 401 = login সমস্যা, PDF error-এর সাথে সম্পর্ক নেই |
| Currency | `$` | `BDT` | আমরা SSLCommerz-এ `currency: "BDT"` পাঠাই |
| Date | `${bookingDate}` → `Mon Oct 12 2026 12:07:33 GMT+0600 (Bangladesh Standard Time)` | `12 October 2026` | Invoice-এ পড়ার মতো তারিখ |
| Return type | `Promise<Buffer<ArrayBufferLike>>` | `Promise<Buffer>` | একই জিনিস, সহজ করে লেখা |

> ⚠️ **বাংলা লেখা:** PDFKit-এর default font (Helvetica) বাংলা অক্ষর চেনে না। User-এর নাম বা tour title বাংলায় হলে PDF-এ উল্টাপাল্টা চিহ্ন আসবে। দরকার হলে একটা বাংলা font (`.ttf`) project-এ রেখে `doc.font("path/to/Font.ttf")` দিয়ে ব্যবহার করতে হবে। ঠিক এই কারণেই `৳` না লিখে `BDT` লিখেছি।

---

# Part 2: PDF Cloudinary-তে Upload

## Step 4: Multer কেন কাজে আসবে না?

[Note 31](./31-file-upload-multer-cloudinary.md)-এ image upload করেছিলাম `multer-storage-cloudinary` দিয়ে। সেটা কাজ করে যখন file **user-এর request-এ আসে** (form-data)। Multer একটা middleware — request থেকে file পড়ে।

এখানে PDF কোনো request থেকে আসছে না — **আমাদের server নিজেই বানাচ্ছে**, memory-তে Buffer হিসেবে আছে। তাই Multer-এর কোনো ভূমিকা নেই, Cloudinary-তে **নিজেরা** upload করতে হবে।

Cloudinary-র সাধারণ `upload()` চায় file-এর path বা URL। আমাদের file disk-এ নেই, তাই ব্যবহার করব **`upload_stream()`** — যেটা Buffer-কে stream হিসেবে নিয়ে upload করে, disk-এ কিছু না লিখেই।

## Step 5: `uploadBufferToCloudinary`

```ts
// src/app/config/cloudinary.config.ts (আগের code-এর নিচে)
import type { UploadApiResponse } from "cloudinary";

export const uploadBufferToCloudinary = (buffer: Buffer, fileName: string): Promise<UploadApiResponse> => {
	return new Promise((resolve, reject) => {
		const uploadStream = cloudinary.uploader.upload_stream(
			{
				resource_type: "auto", // PDF নিজে চিনে নেবে
				folder: "invoices", // Cloudinary-তে এই folder-এ রাখবে
				public_id: fileName, // যেমন "invoice-tran_3f9a1c2e-..."
			},
			(error, result) => {
				if (error) return reject(error);
				if (!result) return reject(new Error("Cloudinary returned no result"));
				resolve(result);
			},
		);

		uploadStream.end(buffer); // পুরো Buffer stream-এ ঢেলে দিয়ে বন্ধ
	});
};
```

- **`upload_stream(options, callback)`**: একটা "upload-এর পাইপ" খোলে। এতে যা ঢালব, সেটা Cloudinary-তে যাবে। শেষ হলে callback-এ `result` (যার ভেতরে `secure_url`) বা `error`।
- **`uploadStream.end(buffer)`**: Buffer-টা পাইপে ঢেলে বলে "শেষ" — তখনই upload সম্পূর্ণ হয়।
- আবারও callback-ভিত্তিক কাজকে Promise-এ মোড়া, যাতে `await` করা যায়।

### Teacher-এর code থেকে যা বদলেছি

| বিষয় | আগে | এখন | কেন |
|---|---|---|---|
| অব্যবহৃত code | `new Stream.PassThrough()` বানিয়ে `bufferStream.end(buffer)` | সরানো | এই stream কোথাও ব্যবহারই হচ্ছিল না — upload হচ্ছিল `upload_stream(...).end(buffer)` দিয়ে। Buffer দুই জায়গায় পাঠানো হচ্ছিল, একটা অকেজো। |
| Path দুবার | `public_id: "pdf/..."` **আর** `folder: "pdf"` | শুধু `folder` | দুটো একসাথে দিলে Cloudinary path হয়ে যেতে পারে `pdf/pdf/...` |
| ⚠️ File-এর নাম | `invoice-${Date.now()}` | `invoice-${transactionId}` (caller দেয়) | নিচে দেখো ⬇️ |
| `result` undefined | Type-এ `\| undefined`, caller-কে check করতে হতো | এখানেই reject | Caller নিশ্চিন্তে `result.secure_url` লিখতে পারে |
| `try/catch` + `AppError(401)` | ছিল | সরানো | `generatePdf`-এর মতোই অকেজো catch আর ভুল status |

> ⚠️ **কেন timestamp না, transactionId?** Cloudinary-র URL **public** — URL জানলে যে কেউ PDF খুলতে পারে, login লাগে না। `invoice-1791525831404`-এর মতো timestamp অনুমান করা সহজ (কাছাকাছি সংখ্যা চেষ্টা করলেই অন্যের invoice — নাম, tour, টাকার পরিমাণ সহ)। Note 30-এ `transactionId` বানিয়েছি `randomUUID()` দিয়ে, যেটা অনুমান করা প্রায় অসম্ভব। তাই নামে সেটা ব্যবহার করলে URL-ও অনুমান করা যায় না, আর প্রতিটা payment-এর invoice-এর নাম আলাদা হয়।

## Step 6: Cloudinary-তে PDF দেখার অনুমতি

Image upload হওয়ার পর সাথে সাথে URL দিয়ে দেখা যায়, কিন্তু **PDF আর ZIP** নিরাপত্তার কারণে Cloudinary default-ভাবে **deliver করে না** (URL খুললে 401/error)।

চালু করতে: Cloudinary Console → **Settings → Security** → **"Allow delivery of PDF and ZIP files"** ✅ → Save।

---

# Part 3: Payment Success-এ Invoice

## Step 7: Email Template

```html
<!-- src/app/utils/templates/invoice.ejs -->
<h2>Hello <%= userName %>,</h2>
<p>Thank you for your booking! Your payment was successful.</p>

<table cellpadding="6">
  <tr><td><strong>Tour</strong></td><td><%= tourTitle %></td></tr>
  <tr><td><strong>Guests</strong></td><td><%= guestCount %></td></tr>
  <tr><td><strong>Transaction ID</strong></td><td><%= transactionId %></td></tr>
  <tr><td><strong>Amount Paid</strong></td><td>BDT <%= totalAmount.toFixed(2) %></td></tr>
</table>

<p>Your invoice is attached to this email.</p>
<% if (invoiceUrl) { %>
  <p><a href="<%= invoiceUrl %>">Download Invoice PDF</a></p>
<% } %>
```

`sendEmail.ts`-এ template-এর নাম যোগ:

```ts
type TEmailTemplate = "forgetPassword" | "otp" | "invoice";
```

> ✏️ **Teacher-এর template-এ সমস্যা ছিল:**
> - `<h1>Invoice for Booking ></h1>` — শেষে ভুল করে একটা `>`।
> - `Payment ID:`, `Amount Paid:` লেখা, কিন্তু পাশে কোনো data নেই — email-এ খালি দেখাত।
> - `<a>Download Invoice PDF</a>` — `href` নেই, click করলে কিছুই হতো না।
>
> এখন `<%= %>` দিয়ে আসল data বসে। `<% if %>` (`=` ছাড়া) মানে শুধু JavaScript চালানো, কিছু print করা না — link থাকলে তবেই দেখাবে।

## Step 8: Booking Interface-এ `createdAt`

Invoice-এ booking-এর তারিখ লাগবে। Schema-তে `timestamps: true` থাকায় MongoDB-তে `createdAt` আছে, কিন্তু TypeScript জানে না (আমাদের `IBooking`-এ নেই)। তাই teacher লিখেছিলেন `updatedBooking.createdAt as Date`। পরিষ্কার উপায়:

```ts
// src/app/modules/booking/booking.interface.ts
export interface IBooking {
	user: Types.ObjectId;
	tour: Types.ObjectId;
	payment?: Types.ObjectId;
	guestCount: number;
	status: BOOKING_STATUS;
	createdAt?: Date; // timestamps: true থেকে আসে
	updatedAt?: Date;
}
```

## Step 9: সবচেয়ে বড় সমস্যা — Invoice কি Transaction-এর ভেতরে থাকবে?

Teacher-এর code-এ PDF বানানো, Cloudinary upload, email পাঠানো — **সব transaction-এর ভেতরে**, `commitTransaction()`-এর আগে:

```
session শুরু
  Payment PAID
  Booking COMPLETE
  PDF বানাও
  Cloudinary-তে upload
  invoiceUrl save
  Email পাঠাও        ← 💥 Gmail-এ সমস্যা
  commit
catch → abort → সব rollback
```

> 🚨 **ভাবো কী হয়:** User-এর **টাকা কেটে গেছে**, SSLCommerz বলছে payment সফল। কিন্তু Gmail-এর limit শেষ বা internet-এর ছোট্ট সমস্যায় email পাঠাতে fail করল → `catch` → `abortTransaction()` → **Payment আবার UNPAID, Booking আবার PENDING!**
>
> টাকা দিয়েও user-এর booking confirm হলো না, শুধু একটা email না যাওয়ার কারণে। আর user-কে fail page-এ পাঠানো হবে, সে হয়তো আবার pay করবে — দুবার টাকা কাটা।

**আরও দুটো সমস্যা:**
- **Rollback বাইরের জিনিস ফেরায় না:** Cloudinary-তে upload হওয়া PDF বা পাঠানো email transaction ফেরত আনতে পারে না ([note 31](./31-file-upload-multer-cloudinary.md))। ফলে rollback হলেও PDF Cloudinary-তে পড়ে থাকে।
- **Transaction অনেকক্ষণ খোলা থাকে:** PDF + upload + email মিলে কয়েক second। MongoDB transaction এতক্ষণ খোলা রাখা ভালো না (default সর্বোচ্চ ৬০ second, আর খোলা থাকা অবস্থায় ওই document-গুলো অন্য কাজে আটকে যেতে পারে)।

### নিয়ম: আসল কাজ আর "ভালো লাগার" কাজ আলাদা

| ধরন | কাজ | Fail করলে |
|---|---|---|
| **জরুরি** | Payment PAID, Booking COMPLETE | পুরোটা rollback — transaction-এর ভেতরে |
| **পরবর্তী (side effect)** | PDF, upload, email | শুধু log, payment যেমন আছে থাকবে — transaction-এর **বাইরে, commit-এর পরে** |

Invoice না পেলে user support-এ যোগাযোগ করতে পারে, admin পরে আবার পাঠাতে পারে। কিন্তু টাকা দিয়েও booking না পাওয়া অনেক বড় সমস্যা।

## Step 10: Payment Service

Note 30-এ `successPayment` আগে **validate** করে, তারপর `updatePaymentAndBooking` helper দিয়ে transaction-এ PAID + COMPLETE করে। সেই code-এ শুধু **commit-এর পরে** invoice-এর কাজ যোগ করব।

```ts
// src/app/modules/payment/payment.service.ts
import type { JwtPayload } from "jsonwebtoken";
import { uploadBufferToCloudinary } from "../../config/cloudinary.config";
import { generatePdf, type IInvoiceData } from "../../utils/invoice";
import { sendEmail } from "../../utils/sendEmail";
import type { ITour } from "../tour/tour.interface";
import { type IUser, Role } from "../user/user.interface";
// ... note 30-এর বাকি import

/* ───────────── Invoice: PDF → Cloudinary → DB → Email ───────────── */
const generateAndSendInvoice = async (paymentId: string) => {
	const payment = await Payment.findById(paymentId);
	if (!payment) {
		throw new Error(`Invoice: payment ${paymentId} not found`);
	}

	const booking = await Booking.findById(payment.booking).populate<{ tour: ITour; user: IUser }>([
		{ path: "tour", select: "title" },
		{ path: "user", select: "name email" },
	]);
	if (!booking) {
		throw new Error(`Invoice: booking for payment ${paymentId} not found`);
	}

	const invoiceData: IInvoiceData = {
		transactionId: payment.transactionId,
		bookingDate: booking.createdAt ?? new Date(),
		userName: booking.user.name,
		tourTitle: booking.tour.title,
		guestCount: booking.guestCount,
		totalAmount: payment.amount,
	};

	// ① PDF বানাও (memory-তে)
	const pdfBuffer = await generatePdf(invoiceData);

	// ② Cloudinary-তে রাখো — নাম অনুমান করা যায় না এমন
	const uploadResult = await uploadBufferToCloudinary(pdfBuffer, `invoice-${payment.transactionId}`);

	// ③ URL DB-তে
	await Payment.findByIdAndUpdate(payment._id, { invoiceUrl: uploadResult.secure_url });

	// ④ Email — PDF attachment + download link
	await sendEmail({
		to: booking.user.email,
		subject: "Your PH Tour booking invoice",
		templateName: "invoice",
		templateData: { ...invoiceData, invoiceUrl: uploadResult.secure_url },
		attachments: [
			{
				filename: `invoice-${payment.transactionId}.pdf`,
				content: pdfBuffer,
				contentType: "application/pdf",
			},
		],
	});
};

/* ───────────── Success (note 30 + invoice) ───────────── */
const successPayment = async (transactionId: string, valId?: string) => {
	// ... note 30-এর মতো: payment খোঁজা, already PAID check, SSLCommerz validate
	//     invalid হলে FAILED করে return

	// জরুরি কাজ — transaction-এর ভেতরে, commit হয়ে যায়
	const updatedPayment = await updatePaymentAndBooking(
		transactionId,
		PAYMENT_STATUS.PAID,
		BOOKING_STATUS.COMPLETE,
		validation,
	);

	// ── টাকা নেওয়া আর booking confirm শেষ। এর পরে কিছুই সেটা উল্টাবে না ──

	try {
		await generateAndSendInvoice(updatedPayment._id.toString());
	} catch (error) {
		// invoice-এ সমস্যা হলেও payment সফলই থাকবে; শুধু log
		// eslint-disable-next-line no-console
		console.error(`Invoice failed for transaction ${transactionId}:`, error);
	}

	return { success: true, message: "Payment completed successfully" };
};
```

### বোঝার মতো কিছু point

- **`populate<{ tour: ITour; user: IUser }>([...])`**: Array দিয়ে একবারে দুটো field populate, আর generic দিয়ে TypeScript-কে বলে দেওয়া — populate-এর পর `tour` আর `user` আর ObjectId না, পুরো object। তাই teacher-এর `(updatedBooking.tour as unknown as ITour).title` এর মতো দুবার cast লাগে না, সরাসরি `booking.tour.title`।
- **`new: true`**: Teacher booking update-এ `new: true` দিয়েছিলেন, কারণ update-এর পরের booking (populate সহ) দরকার ছিল। আমাদের version-এ booking আলাদা করে commit-এর **পরে** আনি, তাই এটা আপনিই সর্বশেষ অবস্থা।
- **একই PDF Buffer দুই জায়গায়:** একবার বানিয়ে Cloudinary আর email attachment দুটোতেই ব্যবহার — আবার বানাতে হয় না।
- **`generateAndSendInvoice`-এ `AppError` না, সাধারণ `Error` কেন?** এই error user-এর কাছে কখনো যায় না, শুধু server log-এ — তাই HTTP status code-এর দরকার নেই।
- **পরে আবার পাঠানো:** Invoice fail করলে log-এ transaction ID থাকে। চাইলে admin-এর জন্য একটা route বানিয়ে `generateAndSendInvoice(paymentId)` আবার call করা যায় — function আলাদা রাখার এটাও একটা লাভ।

### Teacher-এর code থেকে যা বদলেছি

| বিষয় | আগে | এখন |
|---|---|---|
| 🚨 Invoice-এর জায়গা | Transaction-এর ভেতরে — email fail হলে PAID payment rollback | Commit-এর পরে, fail হলে শুধু log |
| Validation | `query.transactionId` দেখেই PAID (note 30-এর নিরাপত্তা সমস্যা) | Note 30-এর validate করা version-এর উপর বানানো |
| Status code | "Payment not found", "Booking not found", "Error uploading pdf" — সব `401` | Invoice-এ HTTP status লাগে না; download API-তে 404/403 |
| Type cast | `as unknown as ITour`, `as unknown as IUser` (৩ বার) | `populate<{...}>` generic |
| `createdAt` | `as Date` | Interface-এ যোগ |
| PDF-এর নাম | `invoice-<timestamp>` | `invoice-<transactionId>` |
| `endSession` | `try` আর `catch` দুই জায়গায় | Note 30-এর helper-এ `finally`-তে |

---

# Part 4: Invoice Download API

## Step 11: Route

```ts
// src/app/modules/payment/payment.route.ts
router.get("/invoice/:paymentId", checkAuth(...Object.values(Role)), PaymentController.getInvoiceDownloadUrl);
```

## Step 12: Controller

```ts
// src/app/modules/payment/payment.controller.ts
const getInvoiceDownloadUrl = catchAsync(async (req: Request, res: Response) => {
	const paymentId = req.params.paymentId as string;
	const decodedToken = req.user as JwtPayload;

	const result = await PaymentService.getInvoiceDownloadUrl(paymentId, decodedToken);

	sendResponse(res, {
		statusCode: httpStatus.OK,
		success: true,
		message: "Invoice download URL retrieved successfully",
		data: result,
	});
});
```

`decodedToken` service-এ পাঠাচ্ছি, কারণ কে চাইছে সেটা দেখে অনুমতি দিতে হবে।

## Step 13: Service — শুধু নিজের Invoice (তুমি যা চেয়েছিলে)

> ⚠️ **Teacher-এর code-এ সমস্যা:** Route-এ `checkAuth` আছে, কিন্তু service শুধু `paymentId` দিয়ে খুঁজে invoice দিয়ে দিত। মানে login করা **যেকোনো** user অন্য কারো `paymentId` জানলে তার invoice — নাম, tour, টাকা — দেখতে পারত।

**নিয়ম:**

| কে চাইছে | অনুমতি |
|---|---|
| ADMIN / SUPER_ADMIN | সবার invoice ✅ |
| Booking যার নিজের | নিজের invoice ✅ |
| অন্য কেউ | ❌ 403 |

Payment-এ user-এর id সরাসরি নেই — আছে `booking`। আর booking-এ আছে `user`। তাই: **payment → booking → user** মিলিয়ে দেখি।

```ts
// src/app/modules/payment/payment.service.ts
const getInvoiceDownloadUrl = async (paymentId: string, decodedToken: JwtPayload) => {
	const payment = await Payment.findById(paymentId).select("invoiceUrl booking");

	if (!payment) {
		throw new AppError(httpStatus.NOT_FOUND, "Payment not found.");
	}

	// Admin সব দেখতে পারে; বাকিরা শুধু নিজের booking-এর
	const isAdmin = [Role.ADMIN, Role.SUPER_ADMIN].includes(decodedToken.role as Role);

	if (!isAdmin) {
		const booking = await Booking.findById(payment.booking).select("user");

		if (booking?.user.toString() !== decodedToken.userId) {
			throw new AppError(httpStatus.FORBIDDEN, "You can only view invoices for your own bookings.");
		}
	}

	if (!payment.invoiceUrl) {
		throw new AppError(httpStatus.NOT_FOUND, "Invoice is not available yet. Please try again later.");
	}

	return { invoiceUrl: payment.invoiceUrl };
};
```

- **`.select("invoiceUrl booking")`**: শুধু দরকারি দুটো field আনি। `booking` লাগবে মালিক যাচাইয়ের জন্য — teacher শুধু `invoiceUrl` আনতেন।
- **`booking?.user.toString()`**: `booking.user` একটা ObjectId; token-এর `userId` string। তুলনা করতে আগে string বানাতে হয় (ObjectId আর string `!==` দিয়ে কখনো মেলে না)।
- **Check-এর order**: আগে অনুমতি, তারপর invoice আছে কিনা। উল্টো করলে অন্যের payment-এর ক্ষেত্রেও "Invoice not available" বলে দিত — মানে অনুমতি না থাকলেও তথ্য ফাঁস হতো।
- **Return `{ invoiceUrl }`**: শুধু string না দিয়ে object — পরে আরও field যোগ করা সহজ, আর frontend-এ `data.invoiceUrl` পরিষ্কার।

### Teacher-এর code থেকে যা বদলেছি

| বিষয় | আগে | এখন |
|---|---|---|
| ⚠️ মালিকানা check | ছিল না — যেকোনো user যেকোনো invoice | মালিক বা admin |
| "Payment not found" | `401` | `404` |
| "No invoice found" | `401` | `404`, পরিষ্কার message |
| অন্যের invoice | — | `403` |

> 💡 **একটা সীমাবদ্ধতা মনে রাখো:** এই API নিশ্চিত করে কে **URL পাবে**। কিন্তু Cloudinary-র URL নিজে public — কেউ একবার URL পেলে (যেমন email forward করলে) login ছাড়াই খুলতে পারবে। নাম অনুমান-অযোগ্য করায় (transactionId) অপরিচিত কেউ খুঁজে পাবে না, শেখার project-এর জন্য এটা যথেষ্ট। পুরোপুরি private করতে চাইলে Cloudinary-র **authenticated delivery + signed URL** (কিছুক্ষণের জন্য বৈধ link) ব্যবহার করতে হয় — সেটা advanced বিষয়।

---

## Step 14: Test

1. Tour book করো → SSLCommerz sandbox-এ pay করো।
2. **Email**: "Your PH Tour booking invoice" — `invoice-tran_....pdf` attachment আর download link।
3. **Cloudinary** → Media Library → `invoices` folder-এ PDF।
4. **MongoDB** → payment document-এ `invoiceUrl`, `status: "PAID"`।
5. **Download API:**

| Request | Expected |
|---|---|
| নিজের token + নিজের `paymentId` | 200 + `invoiceUrl` |
| অন্য user-এর token + তোমার `paymentId` | 403 |
| Admin token + যেকোনো `paymentId` | 200 |
| ভুল `paymentId` (valid format) | 404 |
| Invoice বানানোর আগেই (খুব দ্রুত) | 404 "not available yet" |

6. **Invoice fail হলেও payment ঠিক থাকে কিনা** (জরুরি test): `.env`-এ সাময়িকভাবে `SMTP_PASS` ভুল দাও → pay করো → terminal-এ "Invoice failed..." log আসবে, কিন্তু DB-তে payment **PAID**, booking **COMPLETE**, আর user success page-এ যাবে। ✅ Test শেষে password ঠিক করে দাও।

---

## সারাংশ

| বিষয় | মনে রাখার কথা |
|---|---|
| PDFKit | Code দিয়ে PDF; document একটা stream |
| Buffer | Memory-তে binary data; chunk জমিয়ে `Buffer.concat` |
| Promise-এ মোড়ানো | Event/callback → `resolve`/`reject`; বাইরের `try/catch` এগুলো ধরে না |
| Multer না কেন | File request থেকে না, server নিজে বানাচ্ছে |
| `upload_stream` | Disk ছাড়া Buffer সরাসরি Cloudinary-তে |
| PDF delivery | Cloudinary Settings → Security → PDF/ZIP delivery চালু |
| 🚨 Transaction | শুধু PAID + COMPLETE ভেতরে; invoice **commit-এর পরে**, fail হলে শুধু log |
| File নাম | `invoice-<transactionId>` — অনুমান করা যায় না |
| ⚠️ Download API | Payment → Booking → User মিলিয়ে শুধু মালিক বা admin |
| Status code | না পেলে 404, অনুমতি নেই 403 — 401 শুধু login সমস্যায় |