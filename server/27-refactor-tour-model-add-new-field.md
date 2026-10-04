# 29 — Refactor Tour Model: চলমান Project-এ নতুন Field যোগ করা

## Scenario

ধরো একটা project-এ শুরু থেকে কাজ করছ, অনেক দূর এগিয়ে গেছ, DB-তে অনেক data-ও জমে গেছে। হঠাৎ client বলল: "Tour-এ আরও দুটো জিনিস লাগবে।"

তখন তাড়াহুড়ো করে যোগ না করে ভেবেচিন্তে, **optimize ভাবে** যোগ করতে হয়, যাতে আগের data বা চলমান feature ভেঙে না যায়।

Client চাইছে tour-এ দুটো নতুন property:

| Field | মানে | উদাহরণ |
|---|---|---|
| `departureLocation` | যাত্রা কোথা থেকে শুরু | `"Dhaka"` |
| `arrivalLocation` | কোথায় গিয়ে পৌঁছাবে | `"Chattogram"` |

---

## নতুন field-কে ৩ জায়গায় যোগ করতে হয়

একটা request আমাদের server-এ এই পথ দিয়ে DB-তে যায়:

```
Client (body: { title, departureLocation, ... })
   │
   ▼
① Zod schema (validateRequest)   → এখানে না থাকলে field বাদ পড়ে যায়
   │
   ▼
Service → Tour.create / findByIdAndUpdate
   │
   ▼
② Mongoose schema (model)        → এখানে না থাকলে field চুপচাপ বাদ পড়ে যায়
   │
   ▼
MongoDB
```

আর TypeScript-কে জানাতে **③ Interface**।

তাই order হবে: **Interface → Model → Zod schema → পুরোনো data update**।

> 💡 **কোনো একটা জায়গায় ভুলে গেলে কী হয়?**
> - **Model-এ না দিলে:** Mongoose-এর `strict` mode default-ভাবে চালু। Schema-তে নেই এমন field Mongoose **কোনো error ছাড়াই** ফেলে দেয়, DB-তে save হয় না। সবচেয়ে বিপজ্জনক, কারণ কিছু বোঝা যায় না।
> - **Zod-এ না দিলে:** `z.object()` অচেনা field বাদ দিয়ে দেয় (validateRequest যদি parse করা body আবার `req.body`-তে বসায়)। ফলে model-এ থাকলেও data পৌঁছায় না।
> - **Interface-এ না দিলে:** Runtime-এ কাজ করে, কিন্তু TypeScript `tour.departureLocation` চিনবে না, error দেবে।

---

## Step 1: Interface

```ts
// src/app/modules/tour/tour.interface.ts
export interface ITour {
	title: string;
	slug: string;
	// ... আগের field
	startDate?: Date;
	endDate?: Date;
	departureLocation?: string; // ← নতুন
	arrivalLocation?: string;   // ← নতুন
	// ... বাকি field
	division: Types.ObjectId;
	tourType: Types.ObjectId;
}
```

দুটোই **optional** (`?`)। কেন, সেটা নিচে Step 4-এর আগে বিস্তারিত।

---

## Step 2: Model

```ts
// src/app/modules/tour/tour.model.ts
const tourSchema = new Schema<ITour>(
	{
		// ... আগের field
		startDate: { type: Date },
		endDate: { type: Date },
		departureLocation: { type: String }, // ← নতুন
		arrivalLocation: { type: String },   // ← নতুন
		// ... বাকি field
	},
	{ timestamps: true },
);
```

- `required: true` দিইনি, কারণ field optional।
- Interface-এ `?` আর model-এ `required` না থাকা — দুটো মিলতে হবে।

---

## Step 3: Zod Schema

```ts
// src/app/modules/tour/tour.validation.ts
export const createTourZodSchema = z.object({
	// ... আগের field
	departureLocation: z.string().optional(), // ← নতুন
	arrivalLocation: z.string().optional(),   // ← নতুন
	// ...
});

export const updateTourZodSchema = createTourZodSchema.partial();
```

`updateTourZodSchema` যেহেতু `.partial()` দিয়ে create থেকেই বানানো ([note 26](./26-division-tour-module-and-query-builder.md)), তাই update-এ আলাদা করে যোগ করতে হয়নি, নিজে থেকেই চলে এসেছে। দুবার লিখতে না হওয়ার এটাই লাভ।

> Teacher-এর পুরোনো code-এ create আর update schema আলাদা লেখা ছিল, তখন দুটোতেই আলাদা করে যোগ করতে হতো।

---

## নতুন field কেন অবশ্যই optional?

MongoDB **schemaless**: model-এ নতুন field যোগ করলে DB-র **পুরোনো document-গুলো বদলায় না**। তাদের মধ্যে এই field থাকেই না।

```
DB-তে আগে থেকে থাকা tour:
{ title: "Sajek Valley", location: "Rangamati", ... }   ← departureLocation নেই

নতুন বানানো tour:
{ title: "Cox's Bazar", departureLocation: "Dhaka", arrivalLocation: "Chattogram", ... }
```

এখন যদি নতুন field-কে `required: true` করতাম:

| কাজ | কী হতো |
|---|---|
| পুরোনো tour **পড়া** (`find`) | সমস্যা নেই, field `undefined` আসে |
| পুরোনো tour `doc.save()` | ❌ ValidationError — "departureLocation is required" |
| Frontend-এর পুরোনো create form | ❌ নতুন field না পাঠালে tour বানানোই যাবে না |
| Zod-এ required | ❌ পুরোনো client/frontend-এর সব request fail |

মানে একটা field যোগ করতে গিয়ে **চলমান app ভেঙে যেত**। তাই নতুন field সবসময় optional (বা একটা `default` value সহ) দিয়ে শুরু করতে হয়।

> 💡 পরে যদি সত্যিই required করতে হয়: আগে সব পুরোনো document-এ value বসাও (Step 4), frontend-এ field যোগ করো, **তারপর** `required: true` করো।

---

## Step 4: পুরোনো Data Update করা

নতুন tour বানানোর সময় তো field দেওয়া যাবে। কিন্তু **আগে থেকে থাকা tour-গুলোতে** value বসাতে হবে। কয়েকটা উপায়:

### উপায় ১: API দিয়ে (frontend বা Postman থেকে) — যেটা আমরা করছি

আমাদের update API আগে থেকেই আছে, আর `updateTourZodSchema`-তেও নতুন field এসে গেছে:

```
PATCH /api/v1/tour/:id
Authorization: <admin token>
```

```json
{
	"departureLocation": "Dhaka",
	"arrivalLocation": "Chattogram"
}
```

- শুধু পাঠানো field-গুলো update হয়, বাকি সব আগের মতো থাকে।
- Admin panel থাকলে admin নিজেই প্রতিটা tour edit করে বসাতে পারবে।
- Tour কম হলে এটাই সহজ।

### উপায় ২: MongoDB Compass / Atlas থেকে manually

DB-তে সরাসরি document খুলে field যোগ করা। অল্প data-র জন্য চলে, তবে ভুল হওয়ার সম্ভাবনা বেশি।

### উপায় ৩: অনেক data হলে একসাথে (`updateMany`)

শত শত tour থাকলে একটা একটা করে update করা অসম্ভব। তখন একবারে:

```ts
// যেসব tour-এ departureLocation নেই, সবগুলোতে default বসাও
await Tour.updateMany(
	{ departureLocation: { $exists: false } },
	{ $set: { departureLocation: "Dhaka" } },
);
```

- `$exists: false`: field-টা নেই এমন document।
- এটা একবার চালানোর জন্য (যেমন একটা ছোট script বা seed-এর মতো)। একে বলে **data migration**।

| উপায় | কখন |
|---|---|
| PATCH API | অল্প data, বা প্রতিটার value আলাদা |
| Compass / Atlas | খুব অল্প data, দ্রুত test |
| `updateMany` | অনেক data, সবার একই default value |

---

## Optimize ভাবে field যোগ করার Checklist

- [ ] **নাম ভেবে দাও:** পরিষ্কার, camelCase, আগের field-এর style-এর সাথে মিল (`departureLocation`, `depLoc` না)।
- [ ] **Type ঠিক করো:** string, number, Date, নাকি অন্য collection-এর reference (`ObjectId`)?
- [ ] **Optional রাখো** (বা `default` দাও), যাতে পুরোনো data আর পুরোনো frontend না ভাঙে।
- [ ] **৩ জায়গায় যোগ:** Interface → Model → Zod।
- [ ] **পুরোনো data:** দরকার হলে PATCH বা `updateMany` দিয়ে value বসাও।
- [ ] **Test:** নতুন tour create, পুরোনো tour update, আর get — তিনটাই চালিয়ে দেখো।

---

## সারাংশ

| বিষয় | মনে রাখার কথা |
|---|---|
| কোথায় যোগ | Interface → Model → Zod (তিনটাতেই) |
| Model-এ ভুলে গেলে | Mongoose `strict` mode চুপচাপ field ফেলে দেয় |
| Optional কেন | পুরোনো document-এ field নেই; required করলে চলমান app ভাঙে |
| Update schema | `.partial()` থাকায় নিজে থেকেই নতুন field পায় |
| পুরোনো data | PATCH API (অল্প), `updateMany` (অনেক) |
| পরে required করতে | আগে সব data-তে value বসাও, তারপর `required: true` |