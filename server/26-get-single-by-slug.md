# 28 — Slug দিয়ে Single Data Get করা (+ Route Order-এর নিয়ম)

আগের note-এ ([26](./26-division-tour-module-and-query-builder.md)) Division আর Tour-এর slug auto-generate করেছি। এখন দেখব কীভাবে **slug দিয়ে একটা single division** আনা যায়। একই pattern পরে Tour বা অন্য যেকোনো module-এর single API-তে ব্যবহার করা যাবে।

```
GET /api/v1/division/chattogram-division
        │
        ▼
route: router.get("/:slug")  → req.params.slug = "chattogram-division"
        │
        ▼
controller → service: Division.findOne({ slug })
        │
        ├─ পাওয়া গেছে → 200 + data
        └─ পাওয়া যায়নি → AppError 404
```

---

## কেন id-র বদলে slug?

| | id দিয়ে | slug দিয়ে |
|---|---|---|
| URL | `/division/6650f1a2b3c4d5e6f7a8b9c0` | `/division/chattogram-division` |
| পড়া / মনে রাখা | কঠিন | সহজ |
| SEO (Google search) | কাজে আসে না | URL-এ শব্দ থাকায় ভালো |
| Share করা | অসুন্দর link | সুন্দর, বোঝা যায় এমন link |

> ✏️ **সংশোধন:** "id sensitive information, তাই frontend-এ expose করা যায় না" — এটা পুরোপুরি ঠিক না।
> - MongoDB-র `_id` কোনো গোপন তথ্য না। কেউ id জানলেও কিছু করতে পারে না, কারণ নিরাপত্তা আসে **`checkAuth`** (login + role check) থেকে, id লুকিয়ে রাখা থেকে না।
> - আমরা নিজেরাই response-এ `_id` পাঠাচ্ছি, আর update/delete route (`PATCH /:id`, `DELETE /:id`) id দিয়েই চলে। Frontend-এর এগুলোর জন্য id জানতেই হবে।
>
> Slug ব্যবহারের আসল কারণ: **URL সুন্দর, পড়া যায়, SEO-friendly, share করতে সুবিধা।** সাধারণত public "দেখার" page-এ slug আর admin-এর "বদলানো/মোছার" কাজে id ব্যবহার হয়।

---

## Step 1: Route

```ts
// src/app/modules/division/division.route.ts
router.get("/:slug", DivisionController.getSingleDivision);
```

- `:slug` হলো **dynamic** অংশ (route parameter)। URL-এর ওই জায়গায় যা থাকবে, সেটা `req.params.slug`-এ আসবে।
- `GET /division/chattogram-division` → `req.params.slug = "chattogram-division"`
- এখানে `checkAuth` নেই, কারণ division দেখা সবার জন্য খোলা (public)।

---

## Step 2: Route Order-এর নিয়ম — Static আগে, Dynamic পরে

**নিয়ম:** যে route-এ `:id`, `:slug`-এর মতো dynamic অংশ আছে, সেগুলো **একই method-এর** static route-গুলোর **পরে** রাখতে হবে।

### কেন?

Express route **উপর থেকে নিচে** একটা একটা করে মিলিয়ে দেখে। **প্রথম যেটা মিলে যায়**, request সেখানেই চলে যায়, নিচেরগুলো আর দেখে না।

আর dynamic `:slug` মানে "এই জায়গায় **যেকোনো** লেখা"। তাই `/tour-types`, `/stats`, `/anything` — সবই `/:slug`-এর সাথে মিলে যায়।

### উদাহরণ: ভুল order ❌

```ts
router.get("/:slug", TourController.getSingleTour);        // ১ম
router.get("/tour-types", TourController.getAllTourTypes); // ২য়
```

```
GET /tour/tour-types
  │
  ├─ "/:slug" মেলে? ✅ হ্যাঁ ("tour-types" কে slug ভাবল) → এখানেই ঢুকে গেল
  │     → Tour.findOne({ slug: "tour-types" }) → 404 "Tour not found" ❌
  │
  └─ "/tour-types" → কখনো পৌঁছাতেই পারে না
```

### সঠিক order ✅

```ts
router.get("/tour-types", TourController.getAllTourTypes); // static আগে
router.get("/:slug", TourController.getSingleTour);        // dynamic পরে
```

এখন `/tour-types` প্রথমেই নিজের route পেয়ে যায়, আর বাকি সব লেখা (`/coxs-bazar-tour`) `/:slug`-এ যায়।

### ❓ "create route-এর উপরে PATCH `/:id` দিলে create-ও ওই route-এ ঢুকে যাবে" — ঠিক কিনা?

না, ঠিক না। Express দুটো জিনিস **একসাথে** মেলায়: **HTTP method** আর **path**।

```ts
router.patch("/:id", ...);   // PATCH
router.post("/create", ...); // POST
```

`POST /create` এলে Express প্রথমে `PATCH /:id` দেখে — path মিলতে পারত, কিন্তু **method মেলে না** (POST ≠ PATCH), তাই এড়িয়ে যায় আর ঠিকই `POST /create`-এ পৌঁছায়। তাই এখানে কোনো সমস্যা নেই।

সমস্যা হয় শুধু তখন, যখন **method-ও এক, path-ও মিলে যায়**:

| উপরের route | নিচের route | Conflict? | কারণ |
|---|---|---|---|
| `PATCH /:id` | `POST /create` | ❌ না | Method আলাদা |
| `GET /:slug` | `GET /tour-types` | ✅ হ্যাঁ | Method এক, `/tour-types` `/:slug`-এ মেলে |
| `POST /:id` | `POST /create` | ✅ হ্যাঁ | Method এক, `/create` `/:id`-এ মেলে |
| `PATCH /:id` | `PATCH /tour-types/:id` | ❌ না | Segment সংখ্যা আলাদা (১টা বনাম ২টা) |

> 💡 শেষ সারি: `/:id` শুধু **একটা** segment মেলায়। `/tour-types/abc`-এ দুটো segment (`tour-types` আর `abc`), তাই `/:id`-এর সাথে মেলে না।

**নিরাপদ অভ্যাস:** method আলাদা হলেও সব static route উপরে আর সব dynamic route নিচে রাখো। এতে ভবিষ্যতে নতুন route যোগ করলেও ভুল হবে না।

```ts
// ✅ ভালো সাজানো
router.post("/create", ...);   // static
router.get("/", ...);          // static
router.get("/:slug", ...);     // dynamic
router.patch("/:id", ...);     // dynamic
router.delete("/:id", ...);    // dynamic
```

---

## Step 3: Controller

```ts
// src/app/modules/division/division.controller.ts
const getSingleDivision = catchAsync(async (req: Request, res: Response) => {
	const slug = req.params.slug as string;
	const result = await DivisionService.getSingleDivision(slug);
	sendResponse(res, {
		statusCode: httpStatus.OK,
		success: true,
		message: "Division retrieved successfully",
		data: result.data,
	});
});
```

- **`req.params.slug`**: route-এর `:slug`-এর জায়গায় URL-এ যা এসেছে। (`?` দিয়ে আসা query না, তাই `req.query` না।)
- **`as string`**: `noUncheckedIndexedAccess`-এর কারণে type `string | undefined` আসে। Route-এ `:slug` থাকায় এটা সবসময় থাকবে, তাই `as string`।
- ✏️ Message `"Divisions retrieved"` → `"Division retrieved successfully"` (একটা division, তাই singular), আর `200` → `httpStatus.OK`।

---

## Step 4: Service

```ts
// src/app/modules/division/division.service.ts
const getSingleDivision = async (slug: string) => {
	const division = await Division.findOne({ slug });

	if (!division) {
		throw new AppError(httpStatus.NOT_FOUND, "Division not found.");
	}

	return {
		data: division,
	};
};
```

- **`findOne` কেন, `findById` না?** `findById` শুধু `_id` দিয়ে খোঁজে। Slug অন্য field, তাই `findOne({ slug })`। (`{ slug }` হলো `{ slug: slug }`-এর shorthand।)
- **একটাই আসবে তো?** হ্যাঁ। Model-এ `slug: { unique: true }` দেওয়া, আর `generateUniqueSlug` duplicate হলে `-1`, `-2` যোগ করে। তাই এক slug-এ একটাই division।
- **দ্রুত হবে?** হ্যাঁ। `unique: true` দিলে MongoDB slug-এর উপর **index** বানায়, তাই পুরো collection না ঘেঁটে সরাসরি খুঁজে পায়।

> ⚠️ **Bug fix:** Teacher-এর code-এ না পেলে কোনো check ছিল না, তাই response যেত `200 OK` সহ `data: null`। Frontend ভাবত সফল হয়েছে, কিন্তু data নেই। এখন না পেলে সঠিকভাবে **404** যায়।

| Request | আগে | এখন |
|---|---|---|
| `/division/chattogram-division` | 200 + data | 200 + data |
| `/division/abcxyz` | 200 + `data: null` ❌ | 404 "Division not found." ✅ |

> 💡 **Uppercase URL:** Slug সবসময় lowercase-এ save হয়। কেউ `/division/Chattogram-Division` লিখলে পাবে না। চাইলে service-এ `Division.findOne({ slug: slug.toLowerCase() })` দিয়ে এটাও সামলানো যায়।

---

## অন্য Module-এ ব্যবহার: Template

যেকোনো module-এর single API একই ৩ ধাপে:

```ts
// 1. route — static route-গুলোর পরে
router.get("/:slug", XController.getSingleX);

// 2. controller
const getSingleX = catchAsync(async (req: Request, res: Response) => {
	const result = await XService.getSingleX(req.params.slug as string);
	sendResponse(res, {
		statusCode: httpStatus.OK,
		success: true,
		message: "X retrieved successfully",
		data: result,
	});
});

// 3. service
const getSingleX = async (slug: string) => {
	const x = await X.findOne({ slug });
	if (!x) {
		throw new AppError(httpStatus.NOT_FOUND, "X not found.");
	}
	return x;
};
```

### উদাহরণ: Single Tour

```ts
// tour.route.ts — "/tour-types" route-গুলোর পরে, "/:id"-এর সাথে
router.get("/:slug", TourController.getSingleTour);
```

```ts
// tour.service.ts
const getSingleTour = async (slug: string) => {
	const tour = await Tour.findOne({ slug });
	if (!tour) {
		throw new AppError(httpStatus.NOT_FOUND, "Tour not found.");
	}
	return tour;
};
```

```
GET /api/v1/tour/coxs-bazar-sea-beach → Cox's Bazar Sea Beach tour
GET /api/v1/tour/tour-types           → সব tour type (static route আগে থাকায়)
```

---

## সারাংশ

| বিষয় | মনে রাখার কথা |
|---|---|
| Slug কেন | URL সুন্দর, পড়া যায়, SEO, share — id গোপন রাখার জন্য না |
| Route | `router.get("/:slug", ...)` → `req.params.slug` |
| Route order | Express উপর থেকে নিচে, প্রথম মিলটাই নেয়; static আগে, dynamic পরে |
| Conflict কখন | শুধু method এক **আর** path মিলে গেলে |
| Service | `findOne({ slug })`, না পেলে `AppError` 404 |
| Unique | `unique: true` → একটাই result + index-এর কারণে দ্রুত |