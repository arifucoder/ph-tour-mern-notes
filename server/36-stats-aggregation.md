# 36 — Stats API আর MongoDB Aggregation

Admin dashboard-এ সংখ্যা দেখাতে হবে: কতজন user, শেষ ৭ দিনে কতজন নতুন, কোন tour সবচেয়ে বেশি book হয়েছে, মোট কত টাকা আয় হলো ইত্যাদি। এর জন্য ৪টা API:

| Route | কী দেখাবে |
|---|---|
| `GET /api/v1/stats/user` | মোট user, active/inactive/blocked, শেষ ৭/৩০ দিনে নতুন, role অনুযায়ী সংখ্যা |
| `GET /api/v1/stats/tour` | মোট tour, type আর division অনুযায়ী সংখ্যা, গড় খরচ, সবচেয়ে বেশি booked tour |
| `GET /api/v1/stats/booking` | মোট booking, status অনুযায়ী, tour অনুযায়ী, গড় guest, শেষ ৭/৩০ দিনে, কতজন আলাদা user |
| `GET /api/v1/stats/payment` | মোট payment, status অনুযায়ী, মোট আয়, গড় payment |

এই সংখ্যাগুলো বের করতে লাগবে **MongoDB Aggregation**। তাই আগে aggregation ভালো করে বুঝি, তারপর code।

> ✏️ Note-এ লেখা ছিল "project-এ ৪টা module: tour, booking, payment, division"। Stats-এর ৪টা route আসলে **user, tour, booking, payment**-এর জন্য — division-এর আলাদা stats নেই, division আসছে tour stats-এর ভেতরে ("division অনুযায়ী tour")।

---

# Part 1: Aggregation কী? — সহজ করে

## Step 1: `find()` আর `aggregate()`-এর পার্থক্য

| | `find()` | `aggregate()` |
|---|---|---|
| কাজ | Document **খুঁজে এনে দেয়** যেমন আছে তেমন | Document নিয়ে **হিসাব করে নতুন ফলাফল বানায়** |
| প্রশ্নের ধরন | "Dhaka-র tour-গুলো দাও" | "প্রতিটা division-এ কয়টা করে tour আছে?" |
| ফলাফল | Document-এর list | গুনতি, যোগফল, গড়, group — যা চাই |

`countDocuments()` শুধু একটা সংখ্যা দেয় ("মোট ১২টা tour")। কিন্তু "**প্রতিটা** type-এ কয়টা" — এমন ভাগ ভাগ করে গোনা, যোগ, গড় করতে লাগে aggregation।

## Step 2: Pipeline — পরীক্ষার খাতা দিয়ে বুঝি

ধরো স্কুলের অফিসে পুরো স্কুলের সব পরীক্ষার খাতা একটা স্তূপে রাখা। হেডস্যার জানতে চাইলেন: **"ক্লাস ১০-এর কোন section-এর গড় নম্বর সবচেয়ে বেশি?"**

একজন শিক্ষক এটা বের করবেন ধাপে ধাপে, আর প্রতিটা ধাপ আগের ধাপের ফলাফল নিয়ে কাজ করবে:

```
সব খাতা (পুরো স্কুল)
   │
   ▼ ধাপ ১: শুধু ক্লাস ১০-এর খাতা আলাদা করো, বাকি সরিয়ে রাখো      → $match
   │
   ▼ ধাপ ২: section অনুযায়ী আলাদা আলাদা স্তূপ বানাও (A, B, C),
   │        প্রতিটা স্তূপের গড় নম্বর বের করো                      → $group
   │
   ▼ ধাপ ৩: গড় নম্বর অনুযায়ী বেশি থেকে কম সাজাও                   → $sort
   │
   ▼ ধাপ ৪: সবার উপরেরটা নাও                                     → $limit
   │
ফলাফল: "Section B, গড় ৭৮"
```

MongoDB-র aggregation হুবহু এভাবেই কাজ করে:
- **খাতা** = document
- **প্রতিটা ধাপ** = একটা **stage**
- **সব ধাপ মিলিয়ে পুরো প্রক্রিয়া** = **pipeline**

"Pipeline" নামটা এসেছে পানির পাইপ থেকে — এক মাথা দিয়ে data ঢোকে, পাইপের ভেতর দিয়ে একটার পর একটা ধাপ পার হয়, অন্য মাথা দিয়ে ফলাফল বের হয়।

Code-এ pipeline হলো একটা **array**, আর array-র প্রতিটা object একটা stage (ধাপ):

```ts
Model.aggregate([
	{ $match: { ... } },  // stage ১
	{ $group: { ... } },  // stage ২
	{ $sort: { ... } },   // stage ৩
]);
```

**খুব জরুরি:** প্রতিটা stage শুধু **আগের stage-এর output** দেখে, মূল collection না। খাতার উদাহরণে — ধাপ ২-এর শিক্ষক পুরো স্কুলের খাতা দেখেন না, শুধু ধাপ ১-এ আলাদা করা ক্লাস ১০-এর খাতা পান। তাই ধাপের **order বদলালে ফলাফল বদলে যায়**: আগে section অনুযায়ী ভাগ করে পরে ক্লাস ১০ বাছাই করলে, অন্য ক্লাসের খাতাও গড়ে ঢুকে যেত।

## Step 3: আমাদের project-এর আসল code দিয়ে ধাপে ধাপে

নিচের তিনটা উদাহরণই এই note-এর **আসল stats code** থেকে নেওয়া। প্রতিটাতে আগে দেখব DB-তে data কেমন আছে, তারপর প্রতিটা stage-এর পর data কেমন হয়।

---

### উদাহরণ ১: Role অনুযায়ী কতজন user — শুধু `$group`

Admin dashboard-এ দেখাতে চাই: "কতজন সাধারণ user, কতজন guide, কতজন admin"।

**আসল code** (User Stats থেকে):

```ts
User.aggregate([
	{ $group: { _id: "$role", count: { $sum: 1 } } },
]);
```

> ❓ **`_id: "$role"` মানে কি "প্রতিটা user-এর id ধরে তার role গোনা"?**
>
> না — এটাই সবচেয়ে বেশি গুলিয়ে যায়। `$group`-এর ভেতরের `_id`-এর সাথে user-এর নিজের `_id` (`ObjectId("6650...")`)-এর **কোনো সম্পর্ক নেই**। নামটা একই, কাজ পুরো আলাদা।
>
> `$group`-এ `_id` মানে: **"কোন জিনিস দেখে দল বানাব?"** — অর্থাৎ দলের পরিচয়।
>
> তাই `_id: "$role"` পড়তে হবে এভাবে: **"role দেখে দল বানাও"** (SQL-এর ভাষায় *group by role*)।
>
> | লেখা | বাংলায় পড়ো | কয়টা দল হবে |
> |---|---|---|
> | `_id: "$role"` | role দেখে দল বানাও | যত রকম role, তত দল (USER, GUIDE, ADMIN, SUPER_ADMIN → ৪টা) |
> | `_id: "$status"` | status দেখে দল বানাও | যত রকম status, তত দল |
> | `_id: "$tour"` | কোন tour, সেটা দেখে দল বানাও | যতগুলো আলাদা tour, তত দল |
> | `_id: null` | কিছু দেখে ভাগ করো না | সবাই মিলে ১টা দল |
>
> আর `"$role"`-এর সামনের `$` মানে "role **field-এর value**" — প্রতিটা user-এর `role`-এ যা লেখা আছে (`"USER"`, `"ADMIN"`...), সেটাই তার দলের নাম হবে।
>
> MongoDB নামটা `_id` রেখেছে কারণ ফলাফলে প্রতিটা দল একটা নতুন document হয়, আর প্রতিটা document-এর একটা `_id` লাগে — তাই দলের নামটাই সেই নতুন document-এর `_id` হয়ে যায়: `{ _id: "USER", count: 3 }`।

> ❓ **`count: { $sum: 1 }` মানে কী?**
>
> দুটো অংশ:
> - **`count`** — ফলাফলে নতুন field-এর নাম। এটা আমরা নিজেরা দিই, যা খুশি হতে পারে (`total`, `userCount`...)।
> - **`{ $sum: 1 }`** — দলের প্রতিটা member-এর জন্য **১ যোগ করো**।
>
> USER দলে Rahim, Karim, Nadia — তিনজন। প্রত্যেকের জন্য ১ যোগ: 1 + 1 + 1 = **3**। মানে ১ যোগ করতে থাকলে আসলে **গোনা** হয়ে যায়। হাতে গোনার সময় আমরা যেমন "এক, দুই, তিন" বলি — প্রতিটা মানুষের জন্য ১ বাড়াই — ঠিক তাই।

### `$sum`-এর মতো আর কী কী আছে? — Accumulator

`$group`-এর ভেতরে প্রতিটা দলে যে হিসাব করা হয়, সেগুলোকে বলে **accumulator** (জমা করে হিসাব করে যে)। সবচেয়ে বেশি ব্যবহার হয় এগুলো।

উদাহরণের জন্য ধরো Sajek tour-এর দলে ৩টা booking আছে:

```js
{ user: "Rahim", guestCount: 2, createdAt: "1 Oct" }
{ user: "Sumon", guestCount: 5, createdAt: "5 Oct" }
{ user: "Karim", guestCount: 2, createdAt: "9 Oct" }
```

| Accumulator | কাজ | এই দলে ফলাফল | কোথায় কাজে লাগে |
|---|---|---|---|
| `{ $sum: 1 }` | গোনা — প্রত্যেকের জন্য ১ যোগ | 1 + 1 + 1 = **3** | কয়টা booking, কতজন user |
| `{ $sum: "$guestCount" }` | Field-এর value যোগ | 2 + 5 + 2 = **9** | মোট কতজন যাবে, মোট আয় (`"$amount"`) |
| `{ $avg: "$guestCount" }` | গড় | 9 ÷ 3 = **3** | গড় guest, গড় খরচ |
| `{ $min: "$guestCount" }` | সবচেয়ে ছোট | **2** | সবচেয়ে সস্তা tour |
| `{ $max: "$guestCount" }` | সবচেয়ে বড় | **5** | সবচেয়ে বড় booking |
| `{ $first: "$user" }` | দলের প্রথম জনের value | **"Rahim"** | প্রথম booking কে করেছে (আগে `$sort` দিয়ে সাজিয়ে নিতে হয়) |
| `{ $last: "$user" }` | দলের শেষ জনের value | **"Karim"** | সর্বশেষ booking |
| `{ $push: "$user" }` | সবার value একটা array-তে | **["Rahim", "Sumon", "Karim"]** | কে কে book করেছে তার list |
| `{ $addToSet: "$guestCount" }` | array-তে, কিন্তু **duplicate বাদ** | **[2, 5]** | কত রকম guest সংখ্যা এসেছে |

**খেয়াল করো:**
- `$sum: 1` → সবসময় **১** যোগ (গোনা)
- `$sum: "$guestCount"` → **`$` দিয়ে field-এর নাম** দিলে সেই field-এর value যোগ (যোগফল)

একটা `$group`-এ একসাথে কয়েকটা হিসাব করা যায়:

```ts
Booking.aggregate([
	{
		$group: {
			_id: "$tour",
			totalBookings: { $sum: 1 }, // 3
			totalGuests: { $sum: "$guestCount" }, // 9
			avgGuests: { $avg: "$guestCount" }, // 3
			biggestBooking: { $max: "$guestCount" }, // 5
		},
	},
]);
// Sajek-এর ফলাফল: { _id: <Sajek-এর id>, totalBookings: 3, totalGuests: 9, avgGuests: 3, biggestBooking: 5 }
```

> আমাদের stats code-এ ব্যবহার হয়েছে `$sum: 1` (গোনা), `$sum: "$amount"` (মোট আয়) আর `$avg` (গড় খরচ, গড় guest, গড় payment)।

**DB-তে `users` collection** (দরকারি field ছাড়া বাকিগুলো বাদ রাখলাম):

```js
{ name: "Rahim",  role: "USER" }
{ name: "Karim",  role: "USER" }
{ name: "Arif",   role: "SUPER_ADMIN" }
{ name: "Sumon",  role: "GUIDE" }
{ name: "Nadia",  role: "USER" }
{ name: "Tanvir", role: "ADMIN" }
```

**`$group` কী করে — ধাপে ধাপে:**

**ক) `_id: "$role"` — প্রতিটা user-এর `role` দেখে একই role-ওয়ালাদের এক দলে রাখো:**

| User | তার role | কোন দলে যাবে |
|---|---|---|
| Rahim | USER | USER দল |
| Karim | USER | USER দল |
| Arif | SUPER_ADMIN | SUPER_ADMIN দল |
| Sumon | GUIDE | GUIDE দল |
| Nadia | USER | USER দল |
| Tanvir | ADMIN | ADMIN দল |

দল ভাগ শেষে:

```
USER দল:        Rahim, Karim, Nadia
GUIDE দল:       Sumon
ADMIN দল:       Tanvir
SUPER_ADMIN দল: Arif
```

**খ) `count: { $sum: 1 }` — প্রতিটা দলে প্রত্যেক user-এর জন্য ১ যোগ করো (মানে গোনো):**

```
USER দল:        Rahim(1) + Karim(1) + Nadia(1) = 3
GUIDE দল:       Sumon(1)                       = 1
ADMIN দল:       Tanvir(1)                      = 1
SUPER_ADMIN দল: Arif(1)                        = 1
```

**গ) ফলাফল — প্রতিটা দলের জন্য একটা করে document:**

```js
[
	{ _id: "USER", count: 3 },
	{ _id: "GUIDE", count: 1 },
	{ _id: "ADMIN", count: 1 },
	{ _id: "SUPER_ADMIN", count: 1 },
]
```

- `_id` = দলের নাম (কোন role)
- `count` = ওই দলে কতজন

৬ জন user → ৪টা দল → ৪টা ফলাফল। আগের user document-গুলো আর থাকে না, শুধু দলের হিসাব থাকে।

> Booking status অনুযায়ী (`_id: "$status"`) আর payment status অনুযায়ী গোনাও হুবহু এভাবেই কাজ করে — শুধু field-এর নাম আলাদা।

---

### উদাহরণ ২: মোট আয় — `$match` + `$group` (`_id: null`)

Admin জানতে চায়: "এখন পর্যন্ত মোট কত টাকা আয় হয়েছে?"

**আসল code** (Payment Stats থেকে, ছোট করে):

```ts
Payment.aggregate([
	{ $match: { status: "PAID" } },
	{ $group: { _id: null, totalRevenue: { $sum: "$amount" } } },
]);
```

**DB-তে `payments` collection:**

```js
{ transactionId: "tran_a1", amount: 5000, status: "PAID" }      // Rahim, Sajek
{ transactionId: "tran_b2", amount: 8000, status: "PAID" }      // Karim, Cox's Bazar
{ transactionId: "tran_c3", amount: 5000, status: "UNPAID" }    // Nadia — এখনো pay করেনি
{ transactionId: "tran_d4", amount: 3000, status: "FAILED" }    // Sumon — card-এ সমস্যা
{ transactionId: "tran_e5", amount: 4000, status: "PAID" }      // Rahim, Bandarban
```

**Stage ১ — `$match: { status: "PAID" }`** — যারা সত্যিই টাকা দিয়েছে, শুধু তাদের রাখো:

```js
{ transactionId: "tran_a1", amount: 5000, status: "PAID" }
{ transactionId: "tran_b2", amount: 8000, status: "PAID" }
{ transactionId: "tran_e5", amount: 4000, status: "PAID" }
```

Nadia-র UNPAID আর Sumon-এর FAILED বাদ — ওদের টাকা আসেনি, আয়ে ধরা যাবে না।

**Stage ২ — `$group: { _id: null, totalRevenue: { $sum: "$amount" } }`**

**ক) `_id: null`** — কোনো ভাগ করব না, **তিনটাকেই এক দলে** রাখো। (আমরা "role অনুযায়ী" বা "tour অনুযায়ী" আয় চাই না, চাই সবার মিলিয়ে একটা সংখ্যা।)

**খ) `$sum: "$amount"`** — এবার `1` না, প্রতিটা payment-এর **`amount`** যোগ করো:

```
5000 + 8000 + 4000 = 17000
```

**গ) ফলাফল:**

```js
[{ _id: null, totalRevenue: 17000 }]
```

একটাই দল, তাই একটাই document — তবুও array-র ভেতরে (এজন্যই service-এ `[0]?.totalRevenue` দিয়ে শুধু সংখ্যাটা বের করি, Step 9-এ)।

**`$sum: 1` আর `$sum: "$amount"`-এর পার্থক্য এই উদাহরণেই:**

| লিখলে | প্রতিটা payment-এর জন্য যোগ হয় | ফলাফল | মানে |
|---|---|---|---|
| `$sum: 1` | 1, 1, 1 | **3** | কয়টা payment সফল |
| `$sum: "$amount"` | 5000, 8000, 4000 | **17000** | মোট কত টাকা |

---

### উদাহরণ ৩: সবচেয়ে বেশি book হওয়া tour — পুরো pipeline

Admin দেখতে চায়: "কোন tour সবচেয়ে জনপ্রিয়?" এটা আমাদের আসল `getTopBookedTours` function — সবচেয়ে বড় pipeline, তাই প্রতিটা stage আলাদা করে দেখি।

**আসল code:**

```ts
Booking.aggregate([
	{ $match: { status: "COMPLETE" } },                                                  // ১
	{ $group: { _id: "$tour", bookingCount: { $sum: 1 } } },                             // ২
	{ $sort: { bookingCount: -1 } },                                                     // ৩
	{ $limit: 5 },                                                                       // ৪
	{ $lookup: { from: "tours", localField: "_id", foreignField: "_id", as: "tour" } },  // ৫
	{ $unwind: "$tour" },                                                                // ৬
	{ $project: { bookingCount: 1, "tour.title": 1, "tour.slug": 1 } },                  // ৭
]);
```

**DB-তে `bookings` collection:**

Booking-এ tour-এর নাম থাকে না, থাকে tour-এর **id** (একটা লম্বা ObjectId)। পড়ার সুবিধার জন্য id-র জায়গায় লিখছি `<Sajek-এর id>` — মানে "Sajek Valley Trip tour-এর ObjectId"।

```js
{ user: <Rahim-এর id>,  tour: <Sajek-এর id>,       status: "COMPLETE" }
{ user: <Karim-এর id>,  tour: <Cox's Bazar-এর id>, status: "COMPLETE" }
{ user: <Nadia-র id>,   tour: <Sajek-এর id>,       status: "PENDING" }   // pay করেনি
{ user: <Sumon-এর id>,  tour: <Sajek-এর id>,       status: "COMPLETE" }
{ user: <Tanvir-এর id>, tour: <Bandarban-এর id>,   status: "CANCEL" }    // cancel করেছে
{ user: <Rahim-এর id>,  tour: <Cox's Bazar-এর id>, status: "COMPLETE" }
{ user: <Karim-এর id>,  tour: <Sajek-এর id>,       status: "COMPLETE" }
```

**Stage ১ — `$match: { status: "COMPLETE" }`** — শুধু সফল booking:

```js
{ user: <Rahim-এর id>, tour: <Sajek-এর id>,       status: "COMPLETE" }
{ user: <Karim-এর id>, tour: <Cox's Bazar-এর id>, status: "COMPLETE" }
{ user: <Sumon-এর id>, tour: <Sajek-এর id>,       status: "COMPLETE" }
{ user: <Rahim-এর id>, tour: <Cox's Bazar-এর id>, status: "COMPLETE" }
{ user: <Karim-এর id>, tour: <Sajek-এর id>,       status: "COMPLETE" }
```

৭টা থেকে ৫টা। Nadia-র PENDING আর Tanvir-এর CANCEL বাদ।

**Stage ২ — `$group: { _id: "$tour", bookingCount: { $sum: 1 } }`** — একই tour-এর booking এক দলে, তারপর গোনা:

```
Sajek-এর দল:       Rahim, Sumon, Karim   → 1 + 1 + 1 = 3
Cox's Bazar-এর দল: Karim, Rahim          → 1 + 1     = 2
```

```js
{ _id: <Sajek-এর id>,       bookingCount: 3 }
{ _id: <Cox's Bazar-এর id>, bookingCount: 2 }
```

খেয়াল করো, `_id`-তে এখন **tour-এর id** (কারণ `_id: "$tour"` দিয়েছি)। Bandarban নেই — তার একমাত্র booking cancel হয়েছিল।

**Stage ৩ — `$sort: { bookingCount: -1 }`** — বেশি থেকে কম (`-1`):

```js
{ _id: <Sajek-এর id>,       bookingCount: 3 }   // ১ নম্বর
{ _id: <Cox's Bazar-এর id>, bookingCount: 2 }   // ২ নম্বর
```

**Stage ৪ — `$limit: 5`** — প্রথম ৫টা রাখো। আমাদের মাত্র ২টা, তাই দুটোই থাকে। ১০০টা tour থাকলে এখানে ৯৫টা বাদ পড়ত।

**Stage ৫ — `$lookup`** — এখন সমস্যা: শুধু id দেখে frontend বুঝবে না কোনটা কোন tour। তাই `tours` collection থেকে আসল tour আনি:

```ts
{ $lookup: { from: "tours", localField: "_id", foreignField: "_id", as: "tour" } }
```

"আমার document-এর `_id` (tour-এর id) দিয়ে `tours` collection-এ সেই `_id`-ওয়ালা tour খোঁজো, পেলে `tour` নামে বসিয়ে দাও":

```js
{
	_id: <Sajek-এর id>,
	bookingCount: 3,
	tour: [ { _id: <Sajek-এর id>, title: "Sajek Valley Trip", slug: "sajek-valley-trip", costFrom: 5000, description: "...", images: [...], ... } ]
}
{
	_id: <Cox's Bazar-এর id>,
	bookingCount: 2,
	tour: [ { _id: <Cox's Bazar-এর id>, title: "Cox's Bazar Sea Beach", slug: "coxs-bazar-sea-beach", costFrom: 8000, ... } ]
}
```

`tour` এসেছে **array** হিসেবে (`[ ]`-এর ভেতরে) — `$lookup` সবসময় array দেয়।

**Stage ৬ — `$unwind: "$tour"`** — array খুলে object বানাও:

```js
{ _id: <Sajek-এর id>,       bookingCount: 3, tour: { title: "Sajek Valley Trip", slug: "sajek-valley-trip", costFrom: 5000, ... } }
{ _id: <Cox's Bazar-এর id>, bookingCount: 2, tour: { title: "Cox's Bazar Sea Beach", slug: "coxs-bazar-sea-beach", costFrom: 8000, ... } }
```

`[ { ... } ]` → `{ ... }`। এখন `tour.title` সরাসরি লেখা যায়।

**Stage ৭ — `$project`** — tour-এর সব field (description, images...) লাগবে না, শুধু title আর slug রাখো:

```js
{ _id: <Sajek-এর id>,       bookingCount: 3, tour: { title: "Sajek Valley Trip",     slug: "sajek-valley-trip" } }
{ _id: <Cox's Bazar-এর id>, bookingCount: 2, tour: { title: "Cox's Bazar Sea Beach", slug: "coxs-bazar-sea-beach" } }
```

**এটাই API-র response।** Frontend এখন দেখাতে পারে:

```
🏆 সবচেয়ে জনপ্রিয় tour
1. Sajek Valley Trip       — 3 booking
2. Cox's Bazar Sea Beach   — 2 booking
```

**৭টা stage এক নজরে:**

| Stage | কাজ | কয়টা document থাকল |
|---|---|---|
| শুরু | সব booking | 7 |
| ১. `$match` | শুধু COMPLETE | 5 |
| ২. `$group` | tour অনুযায়ী দল, গোনা | 2 (দল) |
| ৩. `$sort` | বেশি থেকে কম | 2 |
| ৪. `$limit` | প্রথম ৫টা | 2 |
| ৫. `$lookup` | tour-এর তথ্য জোড়া | 2 |
| ৬. `$unwind` | array → object | 2 |
| ৭. `$project` | শুধু দরকারি field | 2 |

## Step 4: `$group` — সবচেয়ে জরুরি Stage

```ts
{
	$group: {
		_id: "$tour",          // কীসের ভিত্তিতে দল বানাব
		count: { $sum: 1 },    // প্রতিটা দলে কী হিসাব করব
	},
}
```

### `_id` — দলের নাম

`$group`-এ `_id` মানে document-এর id **না**। এটা হলো "**কোন field দেখে ভাগ করব**":

| `_id` | মানে | ফলাফল |
|---|---|---|
| `"$role"` | একই role-এর user এক দলে | `{ _id: "USER", count: 40 }`, `{ _id: "ADMIN", count: 2 }` |
| `"$status"` | একই status এক দলে | `{ _id: "PAID", count: 30 }`, ... |
| `null` | **কোনো ভাগ না**, সব document এক দলে | `{ _id: null, avgCostFrom: 4500 }` |

তাই গড় খরচ বা মোট আয়ের মতো "সবার মিলিয়ে একটা সংখ্যা" চাইলে `_id: null`।

### `"$role"` — সামনে `$` কেন?

Aggregation-এ string-এর শুরুতে `$` থাকলে মানে "**এই নামের field-এর value**":

```ts
_id: "$role"   // প্রতিটা document-এর role field-এর value (যেমন "USER")
_id: "role"    // ❌ সবার জন্য একই লেখা "role" → সব এক দলে
```

### ❓ `$sum: 1` কেন 1?

`$sum` মানে যোগ করো। **প্রতিটা document-এর জন্য কী যোগ করব** — সেটা পাশে লিখি:

| লেখা | মানে | ফলাফল |
|---|---|---|
| `{ $sum: 1 }` | প্রতিটা document-এর জন্য **১** যোগ করো | দলে **কয়টা** document → গোনা (count) |
| `{ $sum: "$amount" }` | প্রতিটা document-এর **amount** যোগ করো | দলের **মোট টাকা** |

দলে ৫টা document থাকলে `$sum: 1` → 1+1+1+1+1 = **5**। এটাই গোনার কৌশল।

### Accumulator — দলে কী হিসাব করা যায়

| Accumulator | কাজ | উদাহরণ |
|---|---|---|
| `$sum: 1` | গোনা | কয়টা booking |
| `$sum: "$field"` | যোগফল | মোট আয় |
| `$avg: "$field"` | গড় | গড় guest সংখ্যা |
| `$min` / `$max` | সবচেয়ে কম / বেশি | সবচেয়ে সস্তা tour |

একটা `$group`-এ একসাথে কয়েকটা হিসাব করা যায়:

```ts
{ $group: { _id: null, total: { $sum: "$amount" }, average: { $avg: "$amount" }, count: { $sum: 1 } } }
```

## Step 5: বাকি Stage-গুলো

| Stage | কাজ | তুলনা |
|---|---|---|
| `$match` | শর্ত মেলানো document রাখো, বাকি বাদ | `find()`-এর filter |
| `$group` | দলে ভাগ করে হিসাব | উপরে |
| `$sort` | সাজাও (`1` = ছোট→বড়, `-1` = বড়→ছোট) | `.sort()` |
| `$limit` | প্রথম N-টা রাখো | `.limit()` |
| `$lookup` | অন্য collection থেকে data জোড়া লাগাও | `populate()`-এর মতো |
| `$unwind` | Array ভেঙে প্রতিটা item-এর জন্য আলাদা document | নিচে |
| `$project` | কোন field রাখব/বাদ দেব | `.select()` |

### `$lookup` — অন্য Collection জোড়া লাগানো

Tour document-এ tour type-এর শুধু id আছে (`tourType: ObjectId("...")`), নাম নেই। "Type অনুযায়ী tour" দেখাতে type-এর **নাম** লাগবে, তাই `tourtypes` collection থেকে আনতে হবে:

```ts
{
	$lookup: {
		from: "tourtypes",       // কোন collection থেকে আনব (DB-তে collection-এর আসল নাম)
		localField: "tourType",  // আমার (Tour) document-এর কোন field
		foreignField: "_id",     // ওই collection-এর কোন field-এর সাথে মেলাব
		as: "type",              // মিলে যাওয়া data কোন নামে বসবে
	},
}
```

সহজ ভাষায়: "**Tour-এর `tourType` আর TourType-এর `_id` মিললে, সেই TourType document-টা `type` নামে tour-এর ভেতরে বসিয়ে দাও।**"

```js
// আগে
{ title: "Sajek Trip", tourType: ObjectId("abc") }
// $lookup-এর পরে
{ title: "Sajek Trip", tourType: ObjectId("abc"), type: [ { _id: ObjectId("abc"), name: "Short Tour" } ] }
//                                                      ↑ array!
```

**`from`-এ collection-এর নাম কেন `"tourtypes"`?** Mongoose model-এর নাম (`"TourType"`) থেকে collection-এর নাম বানায় — **ছোট হাতের অক্ষর + শেষে `s`**: `TourType` → `tourtypes`, `Division` → `divisions`, `Tour` → `tours`। MongoDB Compass/Atlas-এ গিয়েও দেখা যায়। `$lookup` সরাসরি MongoDB-র কাজ, Mongoose model চেনে না, তাই collection-এর আসল নামই দিতে হয়।

> 💡 হাতে নাম লিখে বানান ভুলের ঝুঁকি না নিয়ে model থেকেই নেওয়া যায়: `from: TourType.collection.name` → `"tourtypes"`। Final code-এ এভাবেই করেছি।

### `$unwind` — Array খুলে ফেলা

`$lookup` সবসময় **array** দেয় (কারণ একাধিক মিলতে পারে)। কিন্তু আমাদের প্রতিটা tour-এর type একটাই। Array থাকলে `$group`-এ `"$type.name"` ঠিকমতো কাজ করে না। তাই `$unwind` দিয়ে array খুলে object বানাই:

```js
// আগে
{ title: "Sajek Trip", type: [ { name: "Short Tour" } ] }
// $unwind: "$type"-এর পরে
{ title: "Sajek Trip", type: { name: "Short Tour" } }   ← এখন object
```

**নামের মানে:** তোমার উদাহরণটা সুন্দর — ঘুড়ি ওড়ানোর সময় নাটাইয়ে সুতা প্যাঁচানো হলো *wind*, আর নাটাই থেকে সুতা ছাড়া হলো *unwind*। এখানে array-র ভেতরে গুটিয়ে থাকা item-গুলো খুলে বের করা হচ্ছে।

**Array-তে একাধিক item থাকলে** `$unwind` প্রতিটা item-এর জন্য **আলাদা document** বানায়:

```js
{ title: "A", tags: ["x", "y"] }   →   { title: "A", tags: "x" }
                                        { title: "A", tags: "y" }
```

> ⚠️ **সাবধান:** Array **খালি** হলে (`type: []` — মানে lookup-এ কিছুই মেলেনি) `$unwind` সেই document **বাদ দিয়ে দেয়**। কোনো tour-এর tourType ভুল হলে বা সেই type মুছে দেওয়া হলে, ওই tour stats থেকে চুপচাপ হারিয়ে যাবে। রাখতে চাইলে: `{ $unwind: { path: "$type", preserveNullAndEmptyArrays: true } }`।

এই কারণেই note-এর শেষে teacher-এর comment করা `updateMany` code-টা আছে — নিচে Step 11-এ।

### `$project` — কী দেখাব

```ts
{ $project: { bookingCount: 1, "tour.title": 1, "tour.slug": 1 } }
```

`1` = রাখো। এখানে শুধু `bookingCount`, tour-এর `title` আর `slug` থাকবে — tour-এর বাকি সব field (description, images...) বাদ। `_id` না বললেও default-ভাবে থাকে (`_id: 0` দিলে বাদ যায়)।

---

# Part 2: Setup

## Step 6: Route

```ts
// src/app/modules/stats/stats.route.ts
import express from "express";
import { checkAuth } from "../../middlewares/checkAuth";
import { Role } from "../user/user.interface";
import { StatsController } from "./stats.controller";

const router = express.Router();

router.get("/booking", checkAuth(Role.ADMIN, Role.SUPER_ADMIN), StatsController.getBookingStats);
router.get("/payment", checkAuth(Role.ADMIN, Role.SUPER_ADMIN), StatsController.getPaymentStats);
router.get("/user", checkAuth(Role.ADMIN, Role.SUPER_ADMIN), StatsController.getUserStats);
router.get("/tour", checkAuth(Role.ADMIN, Role.SUPER_ADMIN), StatsController.getTourStats);

export const StatsRoutes = router;
```

```ts
// src/app/routes/index.ts — moduleRoutes-এ
{
	path: "/stats",
	route: StatsRoutes,
},
```

শুধু admin আর super admin — business-এর আয়, user সংখ্যা সাধারণ user-এর দেখার জিনিস না।

## Step 7: Controller

```ts
// src/app/modules/stats/stats.controller.ts
import type { Request, Response } from "express";
import httpStatus from "http-status-codes";
import { catchAsync } from "../../utils/catchAsync";
import { sendResponse } from "../../utils/sendResponse";
import { StatsService } from "./stats.service";

const getBookingStats = catchAsync(async (req: Request, res: Response) => {
	const stats = await StatsService.getBookingStats();
	sendResponse(res, {
		statusCode: httpStatus.OK,
		success: true,
		message: "Booking stats fetched successfully",
		data: stats,
	});
});

const getPaymentStats = catchAsync(async (req: Request, res: Response) => {
	const stats = await StatsService.getPaymentStats();
	sendResponse(res, {
		statusCode: httpStatus.OK,
		success: true,
		message: "Payment stats fetched successfully",
		data: stats,
	});
});

const getUserStats = catchAsync(async (req: Request, res: Response) => {
	const stats = await StatsService.getUserStats();
	sendResponse(res, {
		statusCode: httpStatus.OK,
		success: true,
		message: "User stats fetched successfully",
		data: stats,
	});
});

const getTourStats = catchAsync(async (req: Request, res: Response) => {
	const stats = await StatsService.getTourStats();
	sendResponse(res, {
		statusCode: httpStatus.OK,
		success: true,
		message: "Tour stats fetched successfully",
		data: stats,
	});
});

export const StatsController = {
	getBookingStats,
	getPaymentStats,
	getUserStats,
	getTourStats,
};
```

## Step 8: Interface-এ `createdAt`

```ts
// src/app/modules/user/user.interface.ts
export interface IUser {
	// ... আগের সব
	createdAt?: Date; // timestamps: true থেকে আসে
}
```

Schema-তে `timestamps: true` থাকায় DB-তে `createdAt` এমনিতেই আছে। Interface-এ যোগ করলে TypeScript-ও জানে — `user.createdAt` লিখলে আর error দেবে না। (Booking-এ [note 34](./34-pdf-invoice-pdfkit.md)-এ আগেই যোগ করেছি।)

---

# Part 3: Service — ৪টা Stats

## Step 9: আগে দুটো সাধারণ কৌশল

### ① Promise তৈরি করো, পরে একসাথে `await`

```ts
const totalUsersPromise = User.countDocuments();          // await নেই → query শুরু, অপেক্ষা না
const activeUsersPromise = User.countDocuments({ ... });  // এটাও শুরু

const [totalUsers, activeUsers] = await Promise.all([totalUsersPromise, activeUsersPromise]);
```

প্রতিটা query আলাদা, একটা আরেকটার উপর নির্ভর করে না। আলাদা `await` দিলে একটা শেষ হলে আরেকটা শুরু হতো (৭টা query × সময়)। `Promise.all` সবগুলো **একসাথে** চালায় — মোট সময় প্রায় সবচেয়ে ধীর query-টার সমান ([note 25](./25-query-builder.md)-এ data + meta-র মতো)।

> Result array-র order = input array-র order। Destructure-এ নাম **ঠিক একই ক্রমে** লিখতে হবে।

### ② তারিখ — "৭ দিন আগে"

```ts
const daysAgo = (days: number) => {
	const date = new Date(); // এখন
	date.setDate(date.getDate() - days); // n দিন পিছিয়ে দাও
	return date;
};
```

```ts
createdAt: { $gte: daysAgo(7) }
```

**`$gte`** = **g**reater **t**han or **e**qual (≥)। মানে "যাদের `createdAt` ৭ দিন আগের মুহূর্ত বা তার পরে"। আজ ১৩ তারিখ সকাল ১০টা হলে → ৬ তারিখ সকাল ১০টা থেকে এখন পর্যন্ত।

> ⚠️ **Bug fix — তারিখ ছিল file-এর একদম উপরে:**
> ```ts
> const now = new Date();
> const sevenDaysAgo = new Date(now).setDate(now.getDate() - 7);
> ```
> এই code function-এর **বাইরে**, file-এর top-level-এ। Top-level code চলে **একবারই — server চালুর সময়**। Server ১ তারিখে চালু হয়ে ২০ দিন চললে, ২০ তারিখেও "শেষ ৭ দিন" মানে হবে ১ তারিখের ৭ দিন আগে থেকে — ভুল সংখ্যা! তাই প্রতিটা request-এ নতুন করে হিসাব করতে function-এর **ভেতরে** নিয়েছি।
>
> **আরেকটা ছোট সমস্যা:** `setDate()` **Date return করে না, একটা number** (millisecond timestamp) return করে। `countDocuments`-এ Mongoose number-কে Date বানিয়ে নেয় বলে কাজ করছিল, কিন্তু `aggregate`-এর `$match`-এ Mongoose কোনো রূপান্তর করে না — সেখানে number দিলে কিছুই মিলত না। `daysAgo` helper সবসময় আসল Date দেয়।

### aggregate-এর result সবসময় array

```ts
const result = await Tour.aggregate([{ $group: { _id: null, avgCostFrom: { $avg: "$costFrom" } } }]);
// result = [ { _id: null, avgCostFrom: 4523.5 } ]   ← array, ভেতরে একটা object
```

`_id: null` দিয়ে একটাই ফলাফল এলেও সেটা array-র ভেতরে। Frontend-কে `avgTourCost[0].avgCostFrom` লিখতে বাধ্য না করে, service-এই শুধু সংখ্যাটা বের করে দিই:

```ts
avgTourCost: result[0]?.avgCostFrom ?? 0
```

- `?.` — collection খালি হলে result `[]`, `[0]` = `undefined`; `?.` না দিলে crash।
- `?? 0` — না থাকলে 0।

---

## Step 10: User Stats

```ts
const getUserStats = async () => {
	const sevenDaysAgo = daysAgo(7);
	const thirtyDaysAgo = daysAgo(30);

	const [
		totalUsers,
		totalActiveUsers,
		totalInactiveUsers,
		totalBlockedUsers,
		newUsersInLast7Days,
		newUsersInLast30Days,
		usersByRole,
	] = await Promise.all([
		User.countDocuments(),
		User.countDocuments({ isActive: IsActive.ACTIVE }),
		User.countDocuments({ isActive: IsActive.INACTIVE }),
		User.countDocuments({ isActive: IsActive.BLOCKED }),
		User.countDocuments({ createdAt: { $gte: sevenDaysAgo } }),
		User.countDocuments({ createdAt: { $gte: thirtyDaysAgo } }),
		// role অনুযায়ী ভাগ করে গোনা
		User.aggregate([{ $group: { _id: "$role", count: { $sum: 1 } } }]),
	]);

	return {
		totalUsers,
		totalActiveUsers,
		totalInactiveUsers,
		totalBlockedUsers,
		newUsersInLast7Days,
		newUsersInLast30Days,
		usersByRole,
	};
};
```

> 💡 আলাদা `...Promise` variable না বানিয়ে সরাসরি `Promise.all`-এর array-তে query লিখেছি — একই কাজ, অর্ধেক লাইন। আর destructure-এর নামের ঠিক পাশেই query থাকায় order ভুল হওয়ার সম্ভাবনা কমে (Booking stats-এ ঠিক এই ভুলটাই হয়েছিল ⬇️)।

**Response:**

```json
{
  "totalUsers": 52,
  "totalActiveUsers": 48,
  "totalInactiveUsers": 1,
  "totalBlockedUsers": 3,
  "newUsersInLast7Days": 6,
  "newUsersInLast30Days": 19,
  "usersByRole": [
    { "_id": "USER", "count": 45 },
    { "_id": "GUIDE", "count": 4 },
    { "_id": "ADMIN", "count": 2 },
    { "_id": "SUPER_ADMIN", "count": 1 }
  ]
}
```

---

## Step 11: Tour Stats

### Type অনুযায়ী tour — `$lookup` → `$unwind` → `$group`

```ts
Tour.aggregate([
	// ১. tourType-এর id দিয়ে tourtypes collection থেকে type আনো
	{ $lookup: { from: TourType.collection.name, localField: "tourType", foreignField: "_id", as: "type" } },
	// ২. [ {...} ] → {...}
	{ $unwind: "$type" },
	// ৩. type-এর নাম অনুযায়ী দল, প্রতিটায় কয়টা
	{ $group: { _id: "$type.name", count: { $sum: 1 } } },
]);
```

```
{ title: "Sajek", tourType: id1 } ─┐
{ title: "Coxs",  tourType: id2 } ─┼─ $lookup ─ $unwind ─→ { title, type: { name: "Short Tour" } } ...
{ title: "Bandarban", tourType: id1 } ┘                                      │
                                                                          $group
                                                                            ▼
                                     [ { _id: "Short Tour", count: 2 }, { _id: "Long Tour", count: 1 } ]
```

Division অনুযায়ী tour হুবহু একই, শুধু `from: Division.collection.name`, `localField: "division"`।

### Teacher-এর comment করা `updateMany` — কেন?

```ts
await Tour.updateMany(
	{ $or: [{ tourType: { $type: "string" } }, { division: { $type: "string" } }] },
	[{ $set: { tourType: { $toObjectId: "$tourType" }, division: { $toObjectId: "$division" } } }],
);
```

`$lookup` হুবহু মেলায়: **string `"abc..."` আর ObjectId `ObjectId("abc...")` আলাদা জিনিস**, মিলবে না। কোনো tour-এ `tourType` যদি string হিসেবে save হয়ে থাকে, lookup-এ কিছুই মিলবে না → `$unwind` সেই tour বাদ দেবে → stats-এ গরমিল।

Mongoose model দিয়ে create করলে Mongoose নিজেই string-কে ObjectId বানিয়ে save করে। কিন্তু Compass/Atlas থেকে হাতে data ঢোকালে বা পুরোনো কোনো bug থাকলে string রয়ে যেতে পারে। এই code সেগুলো একবারে খুঁজে (`$type: "string"`) ObjectId-তে রূপান্তর করে (`$toObjectId`) — একবার চালানোর **data migration** ([note 27](./27-refactor-tour-model-add-new-field.md)-এর `updateMany`-র মতো)। সব ঠিক থাকলে এটা লাগবে না।

### সবচেয়ে বেশি booked tour

এটা আর booking stats-এর "tour অনুযায়ী booking" — দুটো **একই pipeline**, শুধু সংখ্যা আলাদা (top 5 vs top 10)। তাই একটা helper:

```ts
// সফল booking অনুযায়ী top tour
const getTopBookedTours = (limit: number) =>
	Booking.aggregate([
		{ $match: { status: BOOKING_STATUS.COMPLETE } }, // শুধু সফল booking
		{ $group: { _id: "$tour", bookingCount: { $sum: 1 } } }, // tour অনুযায়ী গোনা
		{ $sort: { bookingCount: -1 } }, // বেশি থেকে কম
		{ $limit: limit }, // প্রথম N-টা
		{ $lookup: { from: Tour.collection.name, localField: "_id", foreignField: "_id", as: "tour" } }, // tour-এর তথ্য
		{ $unwind: "$tour" },
		{ $project: { bookingCount: 1, "tour.title": 1, "tour.slug": 1 } }, // শুধু দরকারি field
	]);
```

**Order কেন এমন?** আগে `$group` → `$sort` → `$limit`, **তারপর** `$lookup`। ১০০০টা tour থাকলেও lookup হবে শুধু top ৫-টার জন্য — অনেক দ্রুত। আগে lookup করলে ১০০০টার জন্যই tour data আনতে হতো।

**`$group`-এর পর `_id` কী?** `_id: "$tour"` দিয়েছিলাম, তাই group-এর পর প্রতিটা document-এর `_id` = সেই tour-এর id। তাই lookup-এ `localField: "_id"`।

> ✏️ **`$match: COMPLETE` যোগ করেছি:** Teacher-এর code সব booking গুনত — PENDING (pay করেনি), CANCEL, FAILED সহ। কেউ ১০ বার book করে pay না করলে সেই tour "সবচেয়ে জনপ্রিয়" হয়ে যেত। "সবচেয়ে বেশি booked" বলতে সাধারণত সফল booking বোঝায়। (সব booking দেখতে চাইলে শুধু `$match` লাইনটা মুছে দিও।)

### ❓ Teacher-এর `let` + `pipeline` lookup — আর `$` বনাম `$$`

Teacher top 5-এর lookup অন্যভাবে লিখেছিলেন:

```ts
{
	$lookup: {
		from: "tours",
		let: { tourId: "$_id" },                                  // আমার document-এর _id → variable "tourId"
		pipeline: [{ $match: { $expr: { $eq: ["$_id", "$$tourId"] } } }],
		as: "tour",
	},
}
```

| লেখা | মানে |
|---|---|
| `"$_id"` (একটা `$`) | **যে collection থেকে আনছি** (tours) তার document-এর `_id` |
| `"$$tourId"` (দুটো `$$`) | `let`-এ বানানো **variable** — এখানে আমাদের (group করা booking-এর) `_id` |

তোমার note-এর ধারণা ঠিক: `$` = ওদিকের (foreign) field, `$$` = আমাদের দিক থেকে আনা value (variable)।

এই রূপ লাগে **জটিল শর্তে** মেলাতে (যেমন "id মিলবে **আর** status ACTIVE হবে")। শুধু id মেলানোর জন্য `localField` + `foreignField` একই কাজ করে, পড়তেও সহজ — booking stats-এ teacher নিজেও তাই করেছেন। তাই helper-এ সহজটাই রেখেছি।

### Tour Stats Code

```ts
const getTourStats = async () => {
	const [totalTour, totalTourByTourType, avgTourCostResult, totalTourByDivision, topBookedTours] =
		await Promise.all([
			Tour.countDocuments(),

			// type অনুযায়ী tour
			Tour.aggregate([
				{ $lookup: { from: TourType.collection.name, localField: "tourType", foreignField: "_id", as: "type" } },
				{ $unwind: "$type" },
				{ $group: { _id: "$type.name", count: { $sum: 1 } } },
			]),

			// সব tour-এর গড় costFrom
			Tour.aggregate([{ $group: { _id: null, avgCostFrom: { $avg: "$costFrom" } } }]),

			// division অনুযায়ী tour
			Tour.aggregate([
				{ $lookup: { from: Division.collection.name, localField: "division", foreignField: "_id", as: "division" } },
				{ $unwind: "$division" },
				{ $group: { _id: "$division.name", count: { $sum: 1 } } },
			]),

			getTopBookedTours(5),
		]);

	return {
		totalTour,
		totalTourByTourType,
		avgTourCost: avgTourCostResult[0]?.avgCostFrom ?? 0,
		totalTourByDivision,
		topBookedTours,
	};
};
```

> 💡 `$avg` field না থাকা (বা `null`) document বাদ দিয়ে গড় করে — যে tour-এ `costFrom` দেওয়া নেই সে গড় কমিয়ে দেবে না।

---

## Step 12: Booking Stats

> 🐛 **বড় Bug — Promise.all-এ order গোলমাল:**
>
> ```ts
> const [ totalBooking, totalBookingByStatus, bookingsPerTour, avgGuestCountPerBooking,
>         bookingsLast7Days, bookingsLast30Days, totalBookingByUniqueUsers ]   // ← ৭টা নাম
>   = await Promise.all([
>         totalBookingPromise, totalBookingByStatusPromise, bookingsPerTourPromise,
>         avgGuestCountPerBookingPromise, bookingsLast7DaysPromise, bookingsLast30DaysPromise,
>         totalBookingByStatusPromise,        // ← ভুল করে আবার! (৭ নম্বর)
>         totalBookingByUniqueUsersPromise,   // ← ৮ নম্বর, কোনো নামে যায় না
>   ]);
> ```
>
> Array-তে **৮টা** promise, কিন্তু নাম **৭টা**। ৭ নম্বর জায়গায় ভুল করে `totalBookingByStatusPromise` আবার বসেছে। ফলে `totalBookingByUniqueUsers`-এ আসত **status-এর array**, আসল unique user সংখ্যা কোথাও যেত না। কোনো error দেখায় না — শুধু ভুল data। এজন্যই এখন নামের পাশেই query লিখি (Step 10-এর 💡)।

> 🐛 **খালি collection-এ crash:** `avgGuestCountPerBooking[0].avgGuestCount` — একটাও booking না থাকলে `[0]` = `undefined` → `Cannot read properties of undefined` → 500। নতুন project-এ admin dashboard প্রথম দিনেই ভাঙত। আর `noUncheckedIndexedAccess` থাকায় TypeScript-ও এখানে error দেয়।

```ts
const getBookingStats = async () => {
	const [
		totalBooking,
		totalBookingByStatus,
		bookingsPerTour,
		avgGuestCountResult,
		bookingsLast7Days,
		bookingsLast30Days,
		totalUniqueUsers,
	] = await Promise.all([
		Booking.countDocuments(),

		// status অনুযায়ী
		Booking.aggregate([{ $group: { _id: "$status", count: { $sum: 1 } } }]),

		// tour অনুযায়ী top 10
		getTopBookedTours(10),

		// প্রতি booking-এ গড় guest
		Booking.aggregate([{ $group: { _id: null, avgGuestCount: { $avg: "$guestCount" } } }]),

		Booking.countDocuments({ createdAt: { $gte: daysAgo(7) } }),
		Booking.countDocuments({ createdAt: { $gte: daysAgo(30) } }),

		// কতজন আলাদা user booking করেছে
		Booking.distinct("user").then((users) => users.length),
	]);

	return {
		totalBooking,
		totalBookingByStatus,
		bookingsPerTour,
		avgGuestCountPerBooking: avgGuestCountResult[0]?.avgGuestCount ?? 0,
		bookingsLast7Days,
		bookingsLast30Days,
		totalUniqueUsers,
	};
};
```

### `distinct` কী?

```ts
Booking.distinct("user"); // → [userId1, userId2, userId3]  (প্রতিটা id একবারই)
```

একজন user ৫টা booking করলেও তার id list-এ **একবার**। তারপর `.length` = কতজন **আলাদা** user booking করেছে।

Teacher `.then((user: any) => ...)` লিখেছিলেন — `distinct` এমনিতেই array দেয়, তাই type বলে দেওয়ার দরকার নেই, `any`-ও লাগে না (ফাইলের উপরের `eslint-disable no-explicit-any`-ও আর লাগছে না)।

---

## Step 13: Payment Stats

```ts
const getPaymentStats = async () => {
	const [totalPayment, totalPaymentByStatus, revenueResult, paymentGatewayData] = await Promise.all([
		Payment.countDocuments(),

		// status অনুযায়ী
		Payment.aggregate([{ $group: { _id: "$status", count: { $sum: 1 } } }]),

		// শুধু PAID — মোট আয়, গড়, সংখ্যা একসাথে
		Payment.aggregate([
			{ $match: { status: PAYMENT_STATUS.PAID } },
			{
				$group: {
					_id: null,
					totalRevenue: { $sum: "$amount" },
					avgPaymentAmount: { $avg: "$amount" },
					paidCount: { $sum: 1 },
				},
			},
		]),

		// SSLCommerz validation-এর status অনুযায়ী
		Payment.aggregate([
			{ $group: { _id: { $ifNull: ["$paymentGatewayData.status", "UNKNOWN"] }, count: { $sum: 1 } } },
		]),
	]);

	const revenue = revenueResult[0];

	return {
		totalPayment,
		totalPaymentByStatus,
		totalRevenue: revenue?.totalRevenue ?? 0,
		avgPaymentAmount: revenue?.avgPaymentAmount ?? 0,
		paidCount: revenue?.paidCount ?? 0,
		paymentGatewayData,
	};
};
```

### কী বদলালাম

| বিষয় | আগে | এখন | কেন |
|---|---|---|---|
| ⚠️ গড় payment | সব payment-এর গড় (UNPAID, FAILED সহ) | শুধু **PAID**-এর গড় | UNPAID মানে টাকা আসেনি। ১০টা unpaid booking থাকলে "গড় payment" ভুলভাবে বাড়ত/কমত |
| আয় + গড় | দুটো আলাদা aggregate | একটা `$group`-এ তিনটা হিসাব | একবার DB-তে গিয়েই সব |
| Response | `totalRevenue: [ { _id: null, totalRevenue: 50000 } ]` | `totalRevenue: 50000` | Frontend সরাসরি সংখ্যা পায় |
| বানান | `avgPaymentAMount` | `avgPaymentAmount` | |

### `$ifNull` কী?

```ts
_id: { $ifNull: ["$paymentGatewayData.status", "UNKNOWN"] }
```

"`paymentGatewayData.status` থাকলে সেটা, **না থাকলে** `"UNKNOWN"`।" Fail/cancel হওয়া payment-এ gateway data থাকে না ([note 28](./28-booking-payment-sslcommerz.md)) — তখন `_id: null` না দেখিয়ে পরিষ্কার `"UNKNOWN"` group।

---

## Final: পুরো `stats.service.ts`

```ts
// src/app/modules/stats/stats.service.ts
import { BOOKING_STATUS } from "../booking/booking.interface";
import { Booking } from "../booking/booking.model";
import { Division } from "../division/division.model";
import { PAYMENT_STATUS } from "../payment/payment.interface";
import { Payment } from "../payment/payment.model";
import { Tour, TourType } from "../tour/tour.model";
import { IsActive } from "../user/user.interface";
import { User } from "../user/user.model";

// প্রতি request-এ নতুন করে "n দিন আগে" হিসাব
const daysAgo = (days: number) => {
	const date = new Date();
	date.setDate(date.getDate() - days);
	return date;
};

// সফল booking অনুযায়ী top tour (tour stats আর booking stats দুই জায়গায় লাগে)
const getTopBookedTours = (limit: number) =>
	Booking.aggregate([
		{ $match: { status: BOOKING_STATUS.COMPLETE } },
		{ $group: { _id: "$tour", bookingCount: { $sum: 1 } } },
		{ $sort: { bookingCount: -1 } },
		{ $limit: limit },
		{ $lookup: { from: Tour.collection.name, localField: "_id", foreignField: "_id", as: "tour" } },
		{ $unwind: "$tour" },
		{ $project: { bookingCount: 1, "tour.title": 1, "tour.slug": 1 } },
	]);

/* ─────────────── User ─────────────── */
const getUserStats = async () => {
	const [
		totalUsers,
		totalActiveUsers,
		totalInactiveUsers,
		totalBlockedUsers,
		newUsersInLast7Days,
		newUsersInLast30Days,
		usersByRole,
	] = await Promise.all([
		User.countDocuments(),
		User.countDocuments({ isActive: IsActive.ACTIVE }),
		User.countDocuments({ isActive: IsActive.INACTIVE }),
		User.countDocuments({ isActive: IsActive.BLOCKED }),
		User.countDocuments({ createdAt: { $gte: daysAgo(7) } }),
		User.countDocuments({ createdAt: { $gte: daysAgo(30) } }),
		User.aggregate([{ $group: { _id: "$role", count: { $sum: 1 } } }]),
	]);

	return {
		totalUsers,
		totalActiveUsers,
		totalInactiveUsers,
		totalBlockedUsers,
		newUsersInLast7Days,
		newUsersInLast30Days,
		usersByRole,
	};
};

/* ─────────────── Tour ─────────────── */
const getTourStats = async () => {
	const [totalTour, totalTourByTourType, avgTourCostResult, totalTourByDivision, topBookedTours] =
		await Promise.all([
			Tour.countDocuments(),
			Tour.aggregate([
				{ $lookup: { from: TourType.collection.name, localField: "tourType", foreignField: "_id", as: "type" } },
				{ $unwind: "$type" },
				{ $group: { _id: "$type.name", count: { $sum: 1 } } },
			]),
			Tour.aggregate([{ $group: { _id: null, avgCostFrom: { $avg: "$costFrom" } } }]),
			Tour.aggregate([
				{ $lookup: { from: Division.collection.name, localField: "division", foreignField: "_id", as: "division" } },
				{ $unwind: "$division" },
				{ $group: { _id: "$division.name", count: { $sum: 1 } } },
			]),
			getTopBookedTours(5),
		]);

	return {
		totalTour,
		totalTourByTourType,
		avgTourCost: avgTourCostResult[0]?.avgCostFrom ?? 0,
		totalTourByDivision,
		topBookedTours,
	};
};

/* ─────────────── Booking ─────────────── */
const getBookingStats = async () => {
	const [
		totalBooking,
		totalBookingByStatus,
		bookingsPerTour,
		avgGuestCountResult,
		bookingsLast7Days,
		bookingsLast30Days,
		totalUniqueUsers,
	] = await Promise.all([
		Booking.countDocuments(),
		Booking.aggregate([{ $group: { _id: "$status", count: { $sum: 1 } } }]),
		getTopBookedTours(10),
		Booking.aggregate([{ $group: { _id: null, avgGuestCount: { $avg: "$guestCount" } } }]),
		Booking.countDocuments({ createdAt: { $gte: daysAgo(7) } }),
		Booking.countDocuments({ createdAt: { $gte: daysAgo(30) } }),
		Booking.distinct("user").then((users) => users.length),
	]);

	return {
		totalBooking,
		totalBookingByStatus,
		bookingsPerTour,
		avgGuestCountPerBooking: avgGuestCountResult[0]?.avgGuestCount ?? 0,
		bookingsLast7Days,
		bookingsLast30Days,
		totalUniqueUsers,
	};
};

/* ─────────────── Payment ─────────────── */
const getPaymentStats = async () => {
	const [totalPayment, totalPaymentByStatus, revenueResult, paymentGatewayData] = await Promise.all([
		Payment.countDocuments(),
		Payment.aggregate([{ $group: { _id: "$status", count: { $sum: 1 } } }]),
		Payment.aggregate([
			{ $match: { status: PAYMENT_STATUS.PAID } },
			{
				$group: {
					_id: null,
					totalRevenue: { $sum: "$amount" },
					avgPaymentAmount: { $avg: "$amount" },
					paidCount: { $sum: 1 },
				},
			},
		]),
		Payment.aggregate([
			{ $group: { _id: { $ifNull: ["$paymentGatewayData.status", "UNKNOWN"] }, count: { $sum: 1 } } },
		]),
	]);

	const revenue = revenueResult[0];

	return {
		totalPayment,
		totalPaymentByStatus,
		totalRevenue: revenue?.totalRevenue ?? 0,
		avgPaymentAmount: revenue?.avgPaymentAmount ?? 0,
		paidCount: revenue?.paidCount ?? 0,
		paymentGatewayData,
	};
};

export const StatsService = {
	getBookingStats,
	getPaymentStats,
	getTourStats,
	getUserStats,
};
```

---

## সব পরিবর্তন এক নজরে

| # | সমস্যা | কী হতো | সমাধান |
|---|---|---|---|
| 1 | 🐛 Booking stats-এ Promise.all ৮টা, নাম ৭টা | `totalBookingByUniqueUsers`-এ status-এর array | Query নামের পাশেই |
| 2 | 🐛 তারিখ top-level-এ | Server যত দিন চলে, "শেষ ৭ দিন" তত ভুল | `daysAgo()` function-এর ভেতরে |
| 3 | `setDate()` number দেয় | `aggregate`-এ মিলত না | `daysAgo()` আসল Date দেয় |
| 4 | 🐛 `[0].avgGuestCount` | খালি DB-তে crash (500) | `[0]?.x ?? 0` |
| 5 | গড় payment-এ UNPAID ধরা | ভুল গড় | শুধু PAID |
| 6 | Top booked-এ PENDING/CANCEL ধরা | Pay না করেও "জনপ্রিয়" | `$match: COMPLETE` |
| 7 | Revenue/avg array হিসেবে | Frontend-এ `[0].x` লিখতে হতো | সরাসরি সংখ্যা |
| 8 | একই pipeline দুবার (top 5, top 10) | | `getTopBookedTours(limit)` |
| 9 | Collection নাম হাতে লেখা | বানান ভুলে চুপচাপ খালি ফলাফল | `Model.collection.name` |
| 10 | `any`, `200`, `avgPaymentAMount` | | type ছাড়াই, `httpStatus.OK`, বানান ঠিক |

> 💡 **পরে:** Stats-এর query ভারী, আর dashboard বারবার খোলা হয়। Data এক মিনিট পুরোনো হলেও সমস্যা নেই — তাই ফলাফল Redis-এ ৬০ second cache করে রাখা যায় ([note 33](./33-redis.md)-এ Redis-এর "Caching" ব্যবহার)। এখন দরকার নেই, data অনেক বাড়লে কাজে লাগবে।

---

## সারাংশ

| বিষয় | মনে রাখার কথা |
|---|---|
| Aggregation | Pipeline — stage-এর array; প্রতিটা stage আগেরটার output নিয়ে কাজ করে |
| `$match` | Filter (find-এর মতো) — যত আগে, তত দ্রুত |
| `$group` | `_id` = কীসের ভিত্তিতে দল; `_id: null` = সব এক দলে |
| `"$field"` | `$` দিয়ে শুরু = field-এর value |
| `$sum: 1` | প্রতিটা document-এ ১ যোগ = গোনা; `$sum: "$amount"` = যোগফল |
| `$lookup` | অন্য collection জোড়া (populate-এর মতো); ফলাফল সবসময় array |
| `from` | Collection-এর আসল নাম (ছোট হাতের + s) → `Model.collection.name` |
| `$unwind` | Array → object; খালি array-র document বাদ যায় |
| `$` vs `$$` | `$` = field, `$$` = `let`-এ বানানো variable |
| `$project` | কোন field রাখব (`1`) |
| Result | aggregate সবসময় array → `[0]?.x ?? 0` |
| তারিখ | function-এর ভেতরে হিসাব, top-level-এ না |
| `Promise.all` | স্বাধীন query একসাথে; নামের order = query-র order |