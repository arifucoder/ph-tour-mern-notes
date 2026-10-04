# 26 — Division ও Tour Module (CRUD, Auto Slug, QueryBuilder)

এই class-এ দুটো নতুন module বানাব:

- **Division:** tour কোন division-এ হবে (Dhaka, Chattogram, Sylhet...)
- **Tour:** আসল tour-এর তথ্য। এর ভেতরেই থাকবে **TourType** (short tour নাকি long tour)

সাথে শিখব:
- `name`/`title` থেকে **auto slug** বানানো (Mongoose pre hook দিয়ে)
- **QueryBuilder** class দিয়ে search, filter, sort, field select আর pagination

```
src/app/
├── constants.ts                  ← excludeField
├── utils/
│   ├── QueryBuilder.ts           ← নতুন
│   └── slug.ts                   ← নতুন (slugify + generateUniqueSlug)
├── routes/index.ts               ← division আর tour route যোগ
└── modules/
    ├── division/
    │   ├── division.interface.ts
    │   ├── division.model.ts
    │   ├── division.validation.ts
    │   ├── division.service.ts
    │   ├── division.controller.ts
    │   └── division.route.ts
    └── tour/
        ├── tour.interface.ts     ← ITour + ITourType
        ├── tour.model.ts         ← Tour + TourType
        ├── tour.constant.ts      ← searchable fields
        ├── tour.validation.ts
        ├── tour.service.ts
        ├── tour.controller.ts
        └── tour.route.ts
```

---

# Part 1: Division Module

## Step 1: Interface আর Slug-এর ধারণা

```ts
// src/app/modules/division/division.interface.ts
export interface IDivision {
	name: string;
	slug: string;
	thumbnail?: string;
	description?: string;
}
```

### Slug কী?

Slug হলো নাম থেকে বানানো URL-friendly text: সব lowercase, space-এর জায়গায় `-`, আর `'`, `&`, `!`-এর মতো special character বাদ।

```
name = "Chattogram"
slug = "chattogram-division"
```

সাধারণত আমরা data আনি `id` দিয়ে। Division-এর ক্ষেত্রে `slug` দিয়ে আনব:

| দিয়ে খোঁজা | URL | কেমন |
|---|---|---|
| `id` | `/division/6650f1a2b3c4d5e6f7a8b9c0` | পড়া যায় না, মনে রাখা যায় না |
| `slug` | `/division/chattogram-division` | পড়া যায়, SEO-friendly, share করতে সুবিধা |

Slug client পাঠাবে না, **name থেকে auto-generate** হবে (Step 3)।

> 💡 Slug-এর শেষে আমরা নিজেরাই `-division` যোগ করি। কেউ name-এ `"Chattogram Division"` দিলেও helper আগে শেষের "division" সরিয়ে নেয়, তাই `chattogram-division-division` হবে না, হবে `chattogram-division` (Step 3)।

---

## Step 2: Model

```ts
// src/app/modules/division/division.model.ts
import { model, Schema } from "mongoose";
import type { IDivision } from "./division.interface";

const divisionSchema = new Schema<IDivision>(
	{
		name: { type: String, required: true, unique: true },
		slug: { type: String, unique: true }, // required না, কারণ hook নিজেই বসাবে
		thumbnail: { type: String },
		description: { type: String },
	},
	{
		timestamps: true,
	},
);

export const Division = model<IDivision>("Division", divisionSchema);
```

- `Schema` অবশ্যই **`mongoose`** থেকে import করতে হবে। Auto-import অনেক সময় অন্য package থেকে ভুল জিনিস এনে দেয়, তাই খেয়াল রাখতে হবে।

> ✏️ **সংশোধন:** "`const Division` আর `model("Division")` একই নাম হতে হবে" — এটা বাধ্যতামূলক না। `const`-এর নাম যা খুশি হতে পারে। যেটা গুরুত্বপূর্ণ তা হলো **`model()`-এর ভেতরের string (`"Division"`)**। কারণ:
> - অন্য model-এ `ref: "Division"` দিয়ে এই string-কেই reference করা হয়।
> - এই string থেকেই collection-এর নাম হয় (`"Division"` → `divisions`)।
>
> তবে দুটো একই রাখাই ভালো অভ্যাস, এতে confusion হয় না।

---

## Step 3: Auto Slug — Model-এর Pre Hook

Slug-এর কাজ service-এও করা যেত (teacher প্রথমে practice হিসেবে service-এ দেখিয়েছেন, সেই code এখন comment করা)। কিন্তু **model-এ করাই ভালো**, কারণ:
- যেখান থেকেই division create/update হোক, slug সবসময় তৈরি হবে
- Service-এ বারবার একই code লিখতে হয় না

### Hook-এ arrow function না, normal function কেন?

Hook-এর ভেতরে `this` দিয়ে document (বা query) access করতে হয়। Arrow function-এর নিজের `this` থাকে না, তাই `this` কাজ করবে না। তাই `async function () {}` লিখি।

### দুটো hook কেন?

| Hook | কখন চলে | `this` কী |
|---|---|---|
| `pre("save")` | `Division.create()`, `doc.save()` | document |
| `pre("findOneAndUpdate")` | `findByIdAndUpdate()`, `findOneAndUpdate()` | query |

`findByIdAndUpdate` ভেতরে ভেতরে `findOneAndUpdate`-ই call করে, তাই এই hook দিয়েই update ধরা যায়। (`updateOne`, `insertMany` এই দুই hook-এর কোনোটাই চালায় না।)

### Duplicate slug হলে?

একই slug আগে থাকলে শেষে number যোগ করব: `dhaka-division`, `dhaka-division-1`, `dhaka-division-2`...

> ⚠️ **Bug fix — counter:** Teacher-এর code-এ ছিল `slug = \`${slug}-${counter++}\``। এখানে আগের `slug`-এর সাথেই আবার যোগ হয়, আর counter `0` থেকে শুরু:
> ```
> dhaka-division → dhaka-division-0 → dhaka-division-0-1 → dhaka-division-0-1-2 ❌
> ```
> তাই সবসময় `baseSlug`-এর সাথে যোগ করেছি আর counter `1` থেকে শুরু করেছি:
> ```
> dhaka-division → dhaka-division-1 → dhaka-division-2 ✅
> ```

> ⚠️ **Bug fix — নিজের slug:** Update-এর সময় একই name আবার পাঠালে `Division.exists({ slug })` **নিজের document-কেই** খুঁজে পায়, ফলে slug অকারণে বদলে `dhaka-division-1` হয়ে যায়। তাই check-এর সময় নিজের `_id` বাদ দিয়েছি: `_id: { $ne: currentId }`।

> 💡 **`next` সরিয়েছি:** `async` function হলে Mongoose নিজেই Promise শেষ হওয়ার জন্য অপেক্ষা করে, তাই `next()` লাগে না। Async আর `next` একসাথে দেওয়া অপ্রয়োজনীয়, আর Mongoose-এর নতুন version-এ async hook `next` ছাড়া লেখাই নিয়ম।

### সমস্যা: শুধু lowercase + dash যথেষ্ট না

Teacher-এর `toLowerCase().split(" ").join("-")` শুধু space বদলায়। বাকি সব character URL-এ থেকে যায়:

| Name / Title | শুধু lowercase + dash | সমস্যা |
|---|---|---|
| `Cox's Bazar` | `cox's-bazar` | `'` URL-এ `%27` হয়ে যায় |
| `Sylhet & Srimangal` | `sylhet-&-srimangal` | `&` URL-এ query ভেঙে দেয় |
| `Bandarban!!  Trip` | `bandarban!!--trip` | `!` আর পাশাপাশি দুটো `-` |
| `Café Tour` | `café-tour` | accent-যুক্ত অক্ষর |

তাই slug বানানোর জন্য একটা পরিষ্কার **`slugify`** function, আর duplicate check-সহ পুরো slug বানানোর একটা **`generateUniqueSlug`** helper বানাব। Division, Tour, পরে আরও যেকোনো module সবাই এটাই ব্যবহার করবে, তাই `utils`-এ রাখি।

### `slugify` — ধাপে ধাপে কী করে

উদাহরণ: `"  Cox's Bazar & Café!! "`

| ধাপ | Code | ফলাফল |
|---|---|---|
| 1 | `.normalize("NFKD")` | `é` ভেঙে `e` + accent চিহ্ন (দুটো আলাদা character) |
| 2 | `.replace(/[\u0300-\u036f]/g, "")` | accent চিহ্ন মুছে দেয় → `Cafe` |
| 3 | `.toLowerCase()` | `"  cox's bazar & cafe!! "` |
| 4 | `.trim()` | শুরু/শেষের space বাদ → `"cox's bazar & cafe!!"` |
| 5 | `` .replace(/['"`‘’“”]/g, "") `` | quote/apostrophe বাদ → `"coxs bazar & cafe!!"` |
| 6 | `.replace(/&/g, " and ")` | `&` → `and` → `"coxs bazar  and  cafe!!"` |
| 7 | `.replace(/[^a-z0-9\s-]/g, " ")` | অক্ষর, সংখ্যা, space, `-` ছাড়া সব space → `"coxs bazar  and  cafe  "` |
| 8 | `.replace(/[\s_-]+/g, "-")` | এক বা একাধিক space/`_`/`-` মিলে একটা `-` → `"coxs-bazar-and-cafe-"` |
| 9 | `.replace(/^-+\|-+$/g, "")` | শুরু/শেষের `-` বাদ → **`"coxs-bazar-and-cafe"`** ✅ |

> ⚠️ **Bangla বা অন্য ভাষার নাম:** ধাপ 7-এ `a-z0-9`-এর বাইরের সব character মুছে যায়। তাই name শুধু বাংলায় (`"ঢাকা"`) দিলে slug হয়ে যায় খালি string (`""`)। এজন্য `generateUniqueSlug`-এ খালি slug হলে error দিই — name-এ English অক্ষর বা সংখ্যা থাকতেই হবে।

### `generateUniqueSlug` — পুরো slug বানানো

এই helper তিনটা কাজ করে:
1. `slugify` দিয়ে পরিষ্কার করা
2. দরকার হলে `suffix` যোগ করা (division-এর জন্য `-division`)
3. DB-তে আগে থেকে থাকলে `-1`, `-2` যোগ করা (নিজের `_id` বাদ দিয়ে)

**Suffix দুবার হওয়া আটকানো:** কেউ `"Chattogram Division"` দিলে slugify-এর পর হয় `chattogram-division`, তারপর আবার `-division` যোগ করলে `chattogram-division-division`। তাই suffix যোগ করার আগে শেষে থাকা suffix সরিয়ে নিই:

```
"Chattogram"          → chattogram          → chattogram-division ✅
"Chattogram Division" → chattogram-division → chattogram → chattogram-division ✅
```

> ✏️ **তোমার দেওয়া code থেকে যা বদলেছি:**
> - তোমার version-এ `Tour.exists` hard-coded ছিল, অথচ ভেতরে division-এর logic (`-division` সরানো, "Invalid division name" message)। একই function দুই model-এ ব্যবহার করা যেত না। তাই **model আর suffix parameter** হিসেবে নিয়েছি, একটাই helper সবার জন্য।
> - `throw new Error` → `AppError(400)`। নাহলে globalErrorHandler 500 পাঠাত, অথচ এটা client-এর দেওয়া ভুল name।
> - `/-?division$/` → `(^|-)division$`। পুরোনোটা `"subdivision"`-এর শেষের `division`-ও কেটে দিত (`sub` বাকি থাকত)। নতুনটা শুধু আলাদা শব্দ হিসেবে থাকা `division` কাটে।
> - `excludeId`-এর type `unknown`, কারণ update hook-এ `getQuery()._id` string বা ObjectId দুটোই হতে পারে।
> - Comment `cox-bazar-tour-1` ঠিক ছিল না। Apostrophe মুছে যায় বলে আসলে হয় `coxs-bazar-tour-1`।

### Code: `src/app/utils/slug.ts`

```ts
// src/app/utils/slug.ts
import httpStatus from "http-status-codes";
import AppError from "../errorHelpers/AppError";

export const slugify = (text: string): string =>
	text
		.normalize("NFKD") // é → e + accent
		.replace(/[\u0300-\u036f]/g, "") // accent চিহ্ন বাদ
		.toLowerCase()
		.trim()
		.replace(/['"`‘’“”]/g, "") // quote/apostrophe বাদ: cox's → coxs
		.replace(/&/g, " and ") // & → and
		.replace(/[^a-z0-9\s-]/g, " ") // অক্ষর, সংখ্যা, space, - ছাড়া সব বাদ
		.replace(/[\s_-]+/g, "-") // space/underscore/dash → একটা -
		.replace(/^-+|-+$/g, ""); // শুরু/শেষের - বাদ

// যেকোনো Mongoose model যার exists() আছে (Division, Tour...)
interface ISlugModel {
	exists(filter: Record<string, unknown>): PromiseLike<unknown>;
}

interface IGenerateSlugOptions {
	suffix?: string; // "division" → "dhaka-division"
	excludeId?: unknown; // update-এর সময় নিজের _id, যাতে নিজের slug-কে duplicate না ভাবে
}

export const generateUniqueSlug = async (
	model: ISlugModel,
	text: string,
	options: IGenerateSlugOptions = {},
): Promise<string> => {
	const { suffix, excludeId } = options;

	let cleaned = slugify(text);

	// "chattogram-division" + suffix "division" → আগে "chattogram" বানাই, যাতে দুবার না হয়
	if (suffix) {
		cleaned = cleaned.replace(new RegExp(`(^|-)${suffix}$`), "");
	}

	if (!cleaned) {
		throw new AppError(
			httpStatus.BAD_REQUEST,
			"Cannot generate slug from this name. Please include English letters or numbers.",
		);
	}

	const baseSlug = suffix ? `${cleaned}-${suffix}` : cleaned;
	let slug = baseSlug;
	let counter = 1;

	while (
		await model.exists({
			slug,
			...(excludeId ? { _id: { $ne: excludeId } } : {}),
		})
	) {
		slug = `${baseSlug}-${counter++}`; // dhaka-division-1, coxs-bazar-tour-1
	}

	return slug;
};
```

- **`ISlugModel`**: helper-কে কোনো নির্দিষ্ট model (Tour/Division) জানতে হয় না, শুধু এমন কিছু লাগে যার `exists()` method আছে। সব Mongoose model-এই এটা আছে, তাই `Division`, `Tour` যেকোনোটা পাঠানো যায়।
- **`...(excludeId ? { _id: { $ne: excludeId } } : {})`**: `excludeId` থাকলে `_id` condition যোগ হয়, না থাকলে খালি object spread হয় (কিছুই যোগ হয় না)।
- Hook-এর ভেতরে `AppError` throw করলেও কোনো সমস্যা নেই: `Division.create()` সেই error নিয়ে reject হয় → `catchAsync` → globalErrorHandler → 400।

### Code: Division-এর Hook

Helper থাকায় hook এখন অনেক ছোট:

```ts
// src/app/modules/division/division.model.ts
import { model, Schema } from "mongoose";
import { generateUniqueSlug } from "../../utils/slug";
import type { IDivision } from "./division.interface";

// ... divisionSchema (Step 2-এর মতোই)

// Create / save-এর সময়
divisionSchema.pre("save", async function () {
	if (!this.isModified("name")) return; // name না বদলালে slug-ও বদলাবে না

	this.slug = await generateUniqueSlug(Division, this.name, {
		suffix: "division",
		excludeId: this._id,
	});
});

// findByIdAndUpdate / findOneAndUpdate-এর সময়
divisionSchema.pre("findOneAndUpdate", async function () {
	const division = this.getUpdate() as Partial<IDivision> | null; // update-এ যা পাঠানো হয়েছে

	if (!division?.name) return; // name update না হলে কিছু করার নেই

	division.slug = await generateUniqueSlug(Division, division.name, {
		suffix: "division",
		excludeId: this.getQuery()._id, // যে division update হচ্ছে
	});

	this.setUpdate(division); // নতুন slug সহ update object বসিয়ে দেওয়া
});

export const Division = model<IDivision>("Division", divisionSchema);
```

| Name | Slug |
|---|---|
| `Dhaka` | `dhaka-division` |
| `Dhaka` (আবার, অন্য division) | `dhaka-division-1` |
| `Chattogram Division` | `chattogram-division` |
| `Cox's Bazar` | `coxs-bazar-division` |
| `ঢাকা` | ❌ 400 — "Cannot generate slug..." |

- `this.isModified("name")`: name নতুন বা বদলেছে কিনা। Create-এর সময় সবসময় `true`।
- `this.getUpdate()`: update-এ পাঠানো data (যেমন `{ name: "Sylhet" }`)।
- `this.setUpdate()`: বদলানো update object আবার query-তে বসানো।
- Hook-এর ভেতরে `Division` ব্যবহার করা যায়, কারণ hook চলে অনেক পরে (তখন `Division` তৈরি হয়ে গেছে)।

---

## Step 4: Validation (Zod)

Slug client পাঠাবে না, তাই validation-এ slug নেই।

```ts
// src/app/modules/division/division.validation.ts
import { z } from "zod";

export const createDivisionSchema = z.object({
	name: z.string({ error: "Name is required" }).min(1, { error: "Name cannot be empty" }),
	thumbnail: z.string().optional(),
	description: z.string().optional(),
});

export const updateDivisionSchema = createDivisionSchema.partial(); // সব field optional
```

> 💡 `updateDivisionSchema` আলাদা করে আবার না লিখে `.partial()` দিয়েছি। এটা create schema-র সব field-কে optional বানিয়ে দেয়, ফলে দুই জায়গায় একই জিনিস লিখতে হয় না।

---

## Step 5: Service

> ⚠️ **Bug fix:** Teacher-এর service-এ `throw new Error(...)` ছিল। সাধারণ `Error`-এ status code থাকে না, তাই globalErrorHandler সব সময় **500** পাঠাত (যেমন "already exists"-ও 500, যেটা আসলে client-এর ভুল)। তাই সব জায়গায় `AppError` দিয়ে সঠিক status দিয়েছি।

```ts
// src/app/modules/division/division.service.ts
import httpStatus from "http-status-codes";
import AppError from "../../errorHelpers/AppError";
import type { IDivision } from "./division.interface";
import { Division } from "./division.model";

const createDivision = async (payload: IDivision) => {
	const existingDivision = await Division.findOne({ name: payload.name });
	if (existingDivision) {
		throw new AppError(httpStatus.BAD_REQUEST, "A division with this name already exists.");
	}

	const division = await Division.create(payload); // pre("save") hook slug বানাবে

	return division;
};

const getAllDivisions = async () => {
	const divisions = await Division.find({});
	const totalDivisions = await Division.countDocuments();

	return {
		data: divisions,
		meta: {
			total: totalDivisions,
		},
	};
};

const getSingleDivision = async (slug: string) => {
	const division = await Division.findOne({ slug });
	if (!division) {
		throw new AppError(httpStatus.NOT_FOUND, "Division not found.");
	}

	return {
		data: division,
	};
};

const updateDivision = async (id: string, payload: Partial<IDivision>) => {
	const existingDivision = await Division.findById(id);
	if (!existingDivision) {
		throw new AppError(httpStatus.NOT_FOUND, "Division not found.");
	}

	// name বদলালে: এই id ছাড়া অন্য কোনো division-এ একই name আছে কিনা
	if (payload.name) {
		const duplicateDivision = await Division.findOne({
			name: payload.name,
			_id: { $ne: id },
		});

		if (duplicateDivision) {
			throw new AppError(httpStatus.BAD_REQUEST, "A division with this name already exists.");
		}
	}

	// pre("findOneAndUpdate") hook slug update করবে
	const updatedDivision = await Division.findByIdAndUpdate(id, payload, {
		new: true, // update-এর পরের document return করবে
		runValidators: true, // schema-র validation update-এও চলবে
	});

	return updatedDivision;
};

const deleteDivision = async (id: string) => {
	const existingDivision = await Division.findById(id);
	if (!existingDivision) {
		throw new AppError(httpStatus.NOT_FOUND, "Division not found.");
	}

	await Division.findByIdAndDelete(id);
	return null;
};

export const DivisionService = {
	createDivision,
	getAllDivisions,
	getSingleDivision,
	updateDivision,
	deleteDivision,
};
```

- **`$ne`** = not equal। `_id: { $ne: id }` মানে "এই id বাদে বাকিগুলোর মধ্যে খোঁজো"।
- **`runValidators: true`**: Mongoose default-ভাবে update-এর সময় schema validation চালায় না। এটা দিলে model-এর rule (required, enum, type) update-এও check হয়।
- Teacher-এর code-এ `payload.name` না থাকলেও duplicate check চলত। এখন `if (payload.name)` দিয়ে শুধু name বদলালেই check হয়।
- `getSingleDivision`-এ না পেলে আগে `data: null` সহ 200 যেত, এখন 404।

---

## Step 6: Controller

> ⚠️ **TypeScript fix:** `noUncheckedIndexedAccess: true` থাকায় `req.params.id`-এর type হয় `string | undefined`। Service-এ `string` লাগে, তাই `as string` দিয়েছি (route-এ `:id` থাকায় এটা সবসময় থাকবে)।

```ts
// src/app/modules/division/division.controller.ts
import type { Request, Response } from "express";
import httpStatus from "http-status-codes";
import { catchAsync } from "../../utils/catchAsync";
import { sendResponse } from "../../utils/sendResponse";
import { DivisionService } from "./division.service";

const createDivision = catchAsync(async (req: Request, res: Response) => {
	const result = await DivisionService.createDivision(req.body);
	sendResponse(res, {
		statusCode: httpStatus.CREATED,
		success: true,
		message: "Division created successfully",
		data: result,
	});
});

const getAllDivisions = catchAsync(async (req: Request, res: Response) => {
	const result = await DivisionService.getAllDivisions();
	sendResponse(res, {
		statusCode: httpStatus.OK,
		success: true,
		message: "Divisions retrieved successfully",
		data: result.data,
		meta: result.meta,
	});
});

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

const updateDivision = catchAsync(async (req: Request, res: Response) => {
	const id = req.params.id as string;
	const result = await DivisionService.updateDivision(id, req.body);
	sendResponse(res, {
		statusCode: httpStatus.OK,
		success: true,
		message: "Division updated successfully",
		data: result,
	});
});

const deleteDivision = catchAsync(async (req: Request, res: Response) => {
	const id = req.params.id as string;
	const result = await DivisionService.deleteDivision(id);
	sendResponse(res, {
		statusCode: httpStatus.OK,
		success: true,
		message: "Division deleted successfully",
		data: result,
	});
});

export const DivisionController = {
	createDivision,
	getAllDivisions,
	getSingleDivision,
	updateDivision,
	deleteDivision,
};
```

---

## Step 7: Route

```ts
// src/app/modules/division/division.route.ts
import { Router } from "express";
import { checkAuth } from "../../middlewares/checkAuth";
import { validateRequest } from "../../middlewares/validateRequest";
import { Role } from "../user/user.interface";
import { DivisionController } from "./division.controller";
import { createDivisionSchema, updateDivisionSchema } from "./division.validation";

const router = Router();

router.post(
	"/create",
	checkAuth(Role.ADMIN, Role.SUPER_ADMIN),
	validateRequest(createDivisionSchema),
	DivisionController.createDivision,
);

router.get("/", DivisionController.getAllDivisions);

router.get("/:slug", DivisionController.getSingleDivision); // slug দিয়ে

router.patch(
	"/:id",
	checkAuth(Role.ADMIN, Role.SUPER_ADMIN),
	validateRequest(updateDivisionSchema),
	DivisionController.updateDivision,
);

router.delete("/:id", checkAuth(Role.ADMIN, Role.SUPER_ADMIN), DivisionController.deleteDivision);

export const DivisionRoutes = router;
```

| Method | Route | কে পারবে |
|---|---|---|
| POST | `/api/v1/division/create` | ADMIN, SUPER_ADMIN |
| GET | `/api/v1/division` | সবাই |
| GET | `/api/v1/division/:slug` | সবাই |
| PATCH | `/api/v1/division/:id` | ADMIN, SUPER_ADMIN |
| DELETE | `/api/v1/division/:id` | ADMIN, SUPER_ADMIN |

> সবাইকে দেখতে দিই (GET), কিন্তু বানানো/বদলানো/মোছা শুধু admin-রা পারবে।

`routes/index.ts`-এর `moduleRoutes` array-তে যোগ:

```ts
{
	path: "/division",
	route: DivisionRoutes,
},
```

---

# Part 2: Tour Module

## Step 8: Interface

```ts
// src/app/modules/tour/tour.interface.ts
import type { Types } from "mongoose";

export interface ITourType {
	name: string;
}

export interface ITour {
	title: string;
	slug: string;
	description?: string;
	images?: string[];
	location?: string;
	costFrom?: number;
	startDate?: Date;
	endDate?: Date;
	departureLocation?: string;
	arrivalLocation?: string;
	included?: string[];
	excluded?: string[];
	amenities?: string[];
	tourPlan?: string[];
	maxGuest?: number;
	minAge?: number;
	division: Types.ObjectId;
	tourType: Types.ObjectId;
}
```

| Field | মানে | উদাহরণ |
|---|---|---|
| `costFrom` | সর্বনিম্ন খরচ ("৫০০০ টাকা থেকে শুরু") | `5000` |
| `startDate`, `endDate` | Tour শুরু ও শেষ (JS `Date` object) | `2026-12-01` |
| `departureLocation` | কোথা থেকে যাত্রা শুরু | `"Dhaka"` |
| `arrivalLocation` | কোথায় গিয়ে পৌঁছাবে | `"Cox's Bazar"` |
| `included` | Tour-এ যা পাবেন | `["খাবার", "Hotel", "Gift"]` |
| `excluded` | Tour-এ যা পাবেন **না** | `["Personal খরচ"]` |
| `amenities` | সাথে দেওয়া জিনিস | `["কাঁথা", "Bag", "Jersey"]` |
| `tourPlan` | কোথায় যাব, কী দেখব (দিন অনুযায়ী) | `["Day 1: ...", "Day 2: ..."]` |
| `maxGuest` | সর্বোচ্চ কতজন যেতে পারবে (যেমন bus-এর seat) | `40` |
| `minAge` | সর্বনিম্ন বয়স | `12` |
| `division` | কোন division-এ tour (**required**) | Division-এর `_id` |
| `tourType` | Short নাকি long tour (**required**) | TourType-এর `_id` |

- **`tourPlan` array কেন?** Tour কয়েক দিনের হলে প্রতিদিনের plan আলাদা হতে পারে।
- **`Types` কোথা থেকে?**
  - **Interface-এ** (TypeScript type): `import type { Types } from "mongoose"` → `Types.ObjectId`
  - **Schema-তে** (runtime): `Schema.Types.ObjectId`
- Interface-এ `Types` শুধু type হিসেবে ব্যবহার হচ্ছে, তাই `import type` দিয়েছি (`verbatimModuleSyntax`)।

### TourType আলাদা collection কেন?

Frontend-এ user filter করবে: "আমি short tour দেখতে চাই" বা "long tour"। TourType আলাদা collection-এ থাকলে admin নতুন type যোগ করতে পারবে (যেমন "Family Tour"), আর frontend সেই list দিয়ে filter বানাতে পারবে।

---

## Step 9: Model (TourType + Tour)

`tourType` field-এ reference দিতে হলে TourType-এর model লাগবে। TourType-এ একটাই field, তাই teacher এটা tour-এর file-এর ভেতরেই রেখেছেন।

### Reference (`ref`)

```ts
division: { type: Schema.Types.ObjectId, ref: "Division", required: true },
```

- Tour-এ পুরো division save না করে শুধু division-এর `_id` রাখি।
- `ref: "Division"` = `model<IDivision>("Division", ...)`-এর **double quote-এর ভেতরের নাম**। এটা হুবহু মিলতে হবে, নাহলে পরে `populate()` কাজ করবে না।

### Code

```ts
// src/app/modules/tour/tour.model.ts
import { model, Schema } from "mongoose";
import { generateUniqueSlug } from "../../utils/slug";
import type { ITour, ITourType } from "./tour.interface";

/* ---------------------- TOUR TYPE ---------------------- */
const tourTypeSchema = new Schema<ITourType>(
	{
		name: { type: String, required: true, unique: true },
	},
	{
		timestamps: true,
	},
);

export const TourType = model<ITourType>("TourType", tourTypeSchema);

/* ------------------------- TOUR ------------------------ */
const tourSchema = new Schema<ITour>(
	{
		title: { type: String, required: true },
		slug: { type: String, unique: true }, // hook বসাবে
		description: { type: String },
		images: { type: [String], default: [] },
		location: { type: String },
		costFrom: { type: Number },
		startDate: { type: Date },
		endDate: { type: Date },
		departureLocation: { type: String },
		arrivalLocation: { type: String },
		included: { type: [String], default: [] },
		excluded: { type: [String], default: [] },
		amenities: { type: [String], default: [] },
		tourPlan: { type: [String], default: [] },
		maxGuest: { type: Number },
		minAge: { type: Number },
		division: {
			type: Schema.Types.ObjectId,
			ref: "Division",
			required: true,
		},
		tourType: {
			type: Schema.Types.ObjectId,
			ref: "TourType",
			required: true,
		},
	},
	{
		timestamps: true,
	},
);

// Create / save-এর সময় title থেকে slug
tourSchema.pre("save", async function () {
	if (!this.isModified("title")) return;

	this.slug = await generateUniqueSlug(Tour, this.title, { excludeId: this._id });
});

// Update-এর সময় title বদলালে slug-ও বদলাবে
tourSchema.pre("findOneAndUpdate", async function () {
	const tour = this.getUpdate() as Partial<ITour> | null;

	if (!tour?.title) return;

	tour.slug = await generateUniqueSlug(Tour, tour.title, { excludeId: this.getQuery()._id });

	this.setUpdate(tour);
});

export const Tour = model<ITour>("Tour", tourSchema);
```

- Tour-এর slug-এ `-division`-এর মতো কোনো suffix নেই, শুধু title, তাই `suffix` দিইনি।

| Title | Slug |
|---|---|
| `Cox's Bazar Sea Beach` | `coxs-bazar-sea-beach` |
| `Cox's Bazar Sea Beach` (আবার) | `coxs-bazar-sea-beach-1` |
| `Sylhet & Srimangal Tour` | `sylhet-and-srimangal-tour` |
| `Bandarban!!  Trip` | `bandarban-trip` |
- আগে `slug`-এ `required: true` ছিল, এখন সরানো হয়েছে, কারণ client slug পাঠায় না, hook বসায়।
- `images`-এ এখনো validation নেই, পরে file upload-এর সময় যোগ হবে।

### ❓ TourType-এর জন্য আলাদা module বানানো উচিত?

এখন TourType-এ একটাই field আর এর কাজ শুধু tour-কে সাহায্য করা, তাই tour module-এর ভেতরে রাখা ঠিক আছে। তবে পরে TourType-এ field বা logic বাড়লে (যেমন image, description) `modules/tourType/` নামে আলাদা module বানানো ভালো। তখন interface, model, service আলাদা হবে, আর tour.model.ts শুধু `ref: "TourType"` দিয়ে reference করবে, কিছুই বদলাতে হবে না।

---

## Step 10: Validation

```ts
// src/app/modules/tour/tour.validation.ts
import { z } from "zod";

export const createTourZodSchema = z.object({
	title: z.string({ error: "Title is required" }).min(1, { error: "Title cannot be empty" }),
	description: z.string().optional(),
	location: z.string().optional(),
	costFrom: z.number().nonnegative({ error: "Cost cannot be negative" }).optional(),
	startDate: z.coerce.date({ error: "Invalid start date" }).optional(),
	endDate: z.coerce.date({ error: "Invalid end date" }).optional(),
	departureLocation: z.string().optional(),
	arrivalLocation: z.string().optional(),
	included: z.array(z.string()).optional(),
	excluded: z.array(z.string()).optional(),
	amenities: z.array(z.string()).optional(),
	tourPlan: z.array(z.string()).optional(),
	maxGuest: z.number().int().positive({ error: "Max guest must be at least 1" }).optional(),
	minAge: z.number().int().nonnegative({ error: "Min age cannot be negative" }).optional(),
	division: z.string({ error: "Division is required" }),
	tourType: z.string({ error: "Tour type is required" }),
});

export const updateTourZodSchema = createTourZodSchema.partial();

export const createTourTypeZodSchema = z.object({
	name: z.string({ error: "Tour type name is required" }).min(1),
});
```

> ✏️ **যা ঠিক করেছি:**
> - `.optional().optional()` — দুবার লেখা অপ্রয়োজনীয়, একবারই যথেষ্ট।
> - `startDate`/`endDate`: `z.string()` যেকোনো string নিত (যেমন `"hello"`)। `z.coerce.date()` string-কে Date বানায় আর ভুল date হলে error দেয়।
> - Teacher-এর `updateTourZodSchema`-এ `division` ছিল না, ফলে update-এ division বদলানো যেত না। `.partial()` দিয়ে সব field optional হয়ে গেছে, `division`-ও এসেছে, আর একই field দুবার লিখতে হচ্ছে না।
> - Number field-এ ছোট rule (negative cost, ০ জন guest আটকানো)।

---

## Step 11: Service (Tour + TourType)

> ⚠️ **Bug fix — TourType:**
> - Controller পাঠাচ্ছিল শুধু `name` (string), কিন্তু service নিচ্ছিল `payload: ITourType` (object)। ফলে `payload.name` = `undefined`।
> - Service-এ `TourType.create({ name })` — এখানে `name` নামে কোনো variable-ই নেই! (`payload.name` হওয়ার কথা।)
> - Update-এও একই সমস্যা: string পাঠিয়ে `findByIdAndUpdate(id, "Short Tour")` হচ্ছিল।
>
> তাই controller থেকে পুরো `req.body` পাঠাচ্ছি, আর service `payload` দিয়ে কাজ করছে।
>
> ⚠️ **Bug fix — `throw new Error`:** Division-এর মতোই সব `AppError` দিয়ে বদলেছি (নাহলে 500 যেত)।

```ts
// src/app/modules/tour/tour.service.ts
import httpStatus from "http-status-codes";
import AppError from "../../errorHelpers/AppError";
import { QueryBuilder } from "../../utils/QueryBuilder";
import { tourSearchableFields } from "./tour.constant";
import type { ITour, ITourType } from "./tour.interface";
import { Tour, TourType } from "./tour.model";

/* ------------------------- TOUR ------------------------ */
const createTour = async (payload: ITour) => {
	const existingTour = await Tour.findOne({ title: payload.title });
	if (existingTour) {
		throw new AppError(httpStatus.BAD_REQUEST, "A tour with this title already exists.");
	}

	const tour = await Tour.create(payload); // pre("save") hook slug বানাবে

	return tour;
};

const getAllTours = async (query: Record<string, string>) => {
	const queryBuilder = new QueryBuilder(Tour.find(), query);

	const tours = queryBuilder.search(tourSearchableFields).filter().sort().fields().paginate();

	// data আর meta একসাথে (parallel) আনা
	const [data, meta] = await Promise.all([tours.build(), queryBuilder.getMeta()]);

	return {
		data,
		meta,
	};
};

const updateTour = async (id: string, payload: Partial<ITour>) => {
	const existingTour = await Tour.findById(id);
	if (!existingTour) {
		throw new AppError(httpStatus.NOT_FOUND, "Tour not found.");
	}

	const updatedTour = await Tour.findByIdAndUpdate(id, payload, {
		new: true,
		runValidators: true,
	});

	return updatedTour;
};

const deleteTour = async (id: string) => {
	const existingTour = await Tour.findById(id);
	if (!existingTour) {
		throw new AppError(httpStatus.NOT_FOUND, "Tour not found.");
	}

	return await Tour.findByIdAndDelete(id);
};

/* ---------------------- TOUR TYPE ---------------------- */
const createTourType = async (payload: ITourType) => {
	const existingTourType = await TourType.findOne({ name: payload.name });
	if (existingTourType) {
		throw new AppError(httpStatus.BAD_REQUEST, "Tour type already exists.");
	}

	return await TourType.create(payload);
};

const getAllTourTypes = async () => {
	return await TourType.find();
};

const updateTourType = async (id: string, payload: ITourType) => {
	const existingTourType = await TourType.findById(id);
	if (!existingTourType) {
		throw new AppError(httpStatus.NOT_FOUND, "Tour type not found.");
	}

	const duplicateTourType = await TourType.findOne({ name: payload.name, _id: { $ne: id } });
	if (duplicateTourType) {
		throw new AppError(httpStatus.BAD_REQUEST, "Tour type already exists.");
	}

	const updatedTourType = await TourType.findByIdAndUpdate(id, payload, {
		new: true,
		runValidators: true,
	});

	return updatedTourType;
};

const deleteTourType = async (id: string) => {
	const existingTourType = await TourType.findById(id);
	if (!existingTourType) {
		throw new AppError(httpStatus.NOT_FOUND, "Tour type not found.");
	}

	return await TourType.findByIdAndDelete(id);
};

export const TourService = {
	createTour,
	getAllTours,
	updateTour,
	deleteTour,
	createTourType,
	getAllTourTypes,
	updateTourType,
	deleteTourType,
};
```

- `getAllTours`-এ teacher-এর code-এ ছিল `const tours = await queryBuilder...paginate()`। কিন্তু `paginate()` Promise না, `QueryBuilder` object return করে, তাই `await` কোনো কাজ করে না। সরিয়ে দিয়েছি। আসল DB call হয় `tours.build()`-কে `await` করার সময়।
- `updateTour`-এ `runValidators: true` যোগ করেছি, Division-এর মতো।

---

## Step 12: Controller

```ts
// src/app/modules/tour/tour.controller.ts
import type { Request, Response } from "express";
import httpStatus from "http-status-codes";
import { catchAsync } from "../../utils/catchAsync";
import { sendResponse } from "../../utils/sendResponse";
import { TourService } from "./tour.service";

/* ------------------------- TOUR ------------------------ */
const createTour = catchAsync(async (req: Request, res: Response) => {
	const result = await TourService.createTour(req.body);
	sendResponse(res, {
		statusCode: httpStatus.CREATED,
		success: true,
		message: "Tour created successfully",
		data: result,
	});
});

const getAllTours = catchAsync(async (req: Request, res: Response) => {
	const query = req.query as Record<string, string>;
	const result = await TourService.getAllTours(query);
	sendResponse(res, {
		statusCode: httpStatus.OK,
		success: true,
		message: "Tours retrieved successfully",
		data: result.data,
		meta: result.meta,
	});
});

const updateTour = catchAsync(async (req: Request, res: Response) => {
	const id = req.params.id as string;
	const result = await TourService.updateTour(id, req.body);
	sendResponse(res, {
		statusCode: httpStatus.OK,
		success: true,
		message: "Tour updated successfully",
		data: result,
	});
});

const deleteTour = catchAsync(async (req: Request, res: Response) => {
	const id = req.params.id as string;
	const result = await TourService.deleteTour(id);
	sendResponse(res, {
		statusCode: httpStatus.OK,
		success: true,
		message: "Tour deleted successfully",
		data: result,
	});
});

/* ---------------------- TOUR TYPE ---------------------- */
const createTourType = catchAsync(async (req: Request, res: Response) => {
	const result = await TourService.createTourType(req.body); // পুরো body, শুধু name না
	sendResponse(res, {
		statusCode: httpStatus.CREATED,
		success: true,
		message: "Tour type created successfully",
		data: result,
	});
});

const getAllTourTypes = catchAsync(async (req: Request, res: Response) => {
	const result = await TourService.getAllTourTypes();
	sendResponse(res, {
		statusCode: httpStatus.OK,
		success: true,
		message: "Tour types retrieved successfully",
		data: result,
	});
});

const updateTourType = catchAsync(async (req: Request, res: Response) => {
	const id = req.params.id as string;
	const result = await TourService.updateTourType(id, req.body);
	sendResponse(res, {
		statusCode: httpStatus.OK,
		success: true,
		message: "Tour type updated successfully",
		data: result,
	});
});

const deleteTourType = catchAsync(async (req: Request, res: Response) => {
	const id = req.params.id as string;
	const result = await TourService.deleteTourType(id);
	sendResponse(res, {
		statusCode: httpStatus.OK,
		success: true,
		message: "Tour type deleted successfully",
		data: result,
	});
});

export const TourController = {
	createTour,
	getAllTours,
	updateTour,
	deleteTour,
	createTourType,
	getAllTourTypes,
	updateTourType,
	deleteTourType,
};
```

---

## Step 13: Route

Tour আর TourType দুটোই এক file থেকে handle হচ্ছে, আর `routes/index.ts`-এ শুধু `/tour` একবার যোগ হয়।

```ts
// src/app/modules/tour/tour.route.ts
import express from "express";
import { checkAuth } from "../../middlewares/checkAuth";
import { validateRequest } from "../../middlewares/validateRequest";
import { Role } from "../user/user.interface";
import { TourController } from "./tour.controller";
import { createTourTypeZodSchema, createTourZodSchema, updateTourZodSchema } from "./tour.validation";

const router = express.Router();

/* ------------------ TOUR TYPE ROUTES -------------------- */
router.get("/tour-types", TourController.getAllTourTypes);

router.post(
	"/create-tour-type",
	checkAuth(Role.ADMIN, Role.SUPER_ADMIN),
	validateRequest(createTourTypeZodSchema),
	TourController.createTourType,
);

router.patch(
	"/tour-types/:id",
	checkAuth(Role.ADMIN, Role.SUPER_ADMIN),
	validateRequest(createTourTypeZodSchema),
	TourController.updateTourType,
);

router.delete("/tour-types/:id", checkAuth(Role.ADMIN, Role.SUPER_ADMIN), TourController.deleteTourType);

/* --------------------- TOUR ROUTES ---------------------- */
router.get("/", TourController.getAllTours);

router.post(
	"/create",
	checkAuth(Role.ADMIN, Role.SUPER_ADMIN),
	validateRequest(createTourZodSchema),
	TourController.createTour,
);

router.patch(
	"/:id",
	checkAuth(Role.ADMIN, Role.SUPER_ADMIN),
	validateRequest(updateTourZodSchema),
	TourController.updateTour,
);

router.delete("/:id", checkAuth(Role.ADMIN, Role.SUPER_ADMIN), TourController.deleteTour);

export const TourRoutes = router;
```

> 💡 **Order খেয়াল রাখো:** নির্দিষ্ট route (`/tour-types`) সবসময় dynamic route (`/:id`)-এর **আগে** রাখা ভালো। নাহলে ভবিষ্যতে `GET /:id` বানালে `/tour-types`-কে Express একটা id ভেবে বসবে।

`routes/index.ts`-এ:

```ts
{
	path: "/tour",
	route: TourRoutes,
},
```

| Method | Route | কাজ |
|---|---|---|
| GET | `/api/v1/tour/tour-types` | সব tour type |
| POST | `/api/v1/tour/create-tour-type` | নতুন tour type |
| PATCH | `/api/v1/tour/tour-types/:id` | tour type update |
| DELETE | `/api/v1/tour/tour-types/:id` | tour type delete |
| GET | `/api/v1/tour` | সব tour (search, filter, sort, pagination সহ) |
| POST | `/api/v1/tour/create` | নতুন tour |
| PATCH | `/api/v1/tour/:id` | tour update |
| DELETE | `/api/v1/tour/:id` | tour delete |

---

# Part 3: Search, Filter, Sort, Fields, Pagination (QueryBuilder)

`GET /api/v1/tour`-এ user query দিয়ে tour খুঁজবে। যেমন:

```
/api/v1/tour?searchTerm=sea&location=Cox's Bazar&sort=costFrom&fields=title,costFrom&page=2&limit=5
```

| Query | কাজ |
|---|---|
| `searchTerm=sea` | title/description/location-এ "sea" আছে এমন tour (**search**) |
| `location=Cox's Bazar` | location ঠিক এটাই এমন tour (**filter**) |
| `sort=costFrom` | খরচ কম থেকে বেশি (**sort**) |
| `fields=title,costFrom` | শুধু এই field-গুলো দেখাও (**field select**) |
| `page=2&limit=5` | ২য় page, প্রতি page-এ ৫টা (**pagination**) |

প্রথমে teacher সব কিছু `getAllTours`-এর ভেতরেই লিখেছিলেন (comment করা `getAllToursOld`)। কিন্তু পরে Division, Booking ইত্যাদিতেও একই কাজ লাগবে। তাই এই সব logic একটা **reusable class** `QueryBuilder`-এ নেওয়া হয়েছে।

## Step 14: Constants

```ts
// src/app/modules/tour/tour.constant.ts
export const tourSearchableFields = ["title", "description", "location"];
```

কোন কোন field-এ search চলবে। প্রতিটা module-এর searchable field আলাদা, তাই module-এর ভেতরে রাখা।

```ts
// src/app/constants.ts
export const excludeField = ["searchTerm", "sort", "fields", "page", "limit"];
```

এগুলো DB-র কোনো field না, এগুলো **instruction** (কীভাবে search/sort/paginate করতে হবে)। Filter করার আগে এগুলো query থেকে বাদ দিতে হবে, নাহলে Mongoose `{ page: "2" }` নামে field খুঁজবে আর কিছুই পাবে না।

### ❓ `excludeField` এখানে (`src/app/constants.ts`) কেন?

এটা কোনো একটা module-এর না, **সব module-এর QueryBuilder**-এ লাগে। তাই teacher এটাকে app-level-এ রেখেছেন (module-এর বাইরে, সবার জন্য common)। তোমার পছন্দ না হলে এটা দুইভাবে সাজানো যায়:
- `src/app/constants/index.ts` — একটা folder, পরে আরও global constant এলে এখানেই থাকবে
- `src/app/utils/QueryBuilder.ts`-এর ভেতরেই — কারণ এটা শুধু QueryBuilder ব্যবহার করে

যেকোনোটাই ঠিক, শুধু import path বদলাতে হবে।

## Step 15: QueryBuilder Class

### মূল ধারণা: Method Chaining

```ts
queryBuilder.search(...).filter().sort().fields().paginate()
```

প্রতিটা method `this.modelQuery`-তে কিছু যোগ করে আর **`return this`** করে। তাই একটার পর একটা `.` দিয়ে chain করা যায়।

Mongoose query **lazy**: `Tour.find().find(a).sort(b)` লিখলে এখনই DB-তে যায় না, শুধু query বানায়। `await` করলে তখন একবারে DB-তে যায়। আর `.find()` একাধিকবার দিলে সব condition merge হয়ে যায়।

```
Tour.find()
   │ .search()   → .find({ $or: [...regex] })
   │ .filter()   → .find({ location: "Cox's Bazar" })
   │ .sort()     → .sort("costFrom")
   │ .fields()   → .select("title costFrom")
   │ .paginate() → .skip(5).limit(5)
   ▼
.build() → await → DB-তে একবার query
```

### প্রতিটা method

**1. `search()`** — একাধিক field-এ আংশিক মিল খোঁজা

```js
{ $or: [
  { title:       { $regex: "sea", $options: "i" } },
  { description: { $regex: "sea", $options: "i" } },
  { location:    { $regex: "sea", $options: "i" } }
] }
```
- `$or`: যেকোনো একটা field-এ মিললেই হবে
- `$regex`: আংশিক মিল ("sea" → "Seaside", "Sea Beach")
- `$options: "i"`: case-insensitive (Sea, sea, SEA সব এক)

> ⚠️ **Security fix:** User-এর দেওয়া text সরাসরি `$regex`-এ দিলে `(` বা `*`-এর মতো special character দিলে regex error হয় (500), আর জটিল pattern দিয়ে server ধীর করে দেওয়াও সম্ভব (ReDoS)। তাই special character escape করে দিয়েছি। এছাড়া `searchTerm` না থাকলে search query যোগই করি না।

**2. `filter()`** — exact match

Query-র copy নিয়ে (`{ ...this.query }`) `excludeField`-এর সব key delete করি। বাকি যা থাকে (`{ location: "Cox's Bazar" }`) সেটা দিয়ে `find()`।
- Copy কেন? মূল `this.query` বদলালে পরে `sort`, `page` ইত্যাদি হারিয়ে যেত।
- Filter **exact** ও case-sensitive: `location=dhaka` দিলে `"Dhaka"` পাবে না।

**3. `sort()`** — সাজানো

| Query | মানে |
|---|---|
| না দিলে | `-createdAt` (নতুনগুলো আগে) |
| `sort=costFrom` | কম থেকে বেশি (ascending) |
| `sort=-costFrom` | বেশি থেকে কম (descending, `-` দিয়ে) |

**4. `fields()`** — কোন field দেখাবে

User দেয় comma দিয়ে (`title,location`), কিন্তু Mongoose-এর `select()` চায় space দিয়ে (`"title location"`)। তাই `split(",").join(" ")`। `-description` দিলে ওই field বাদ যাবে।

**5. `paginate()`** — page ভাগ করা

```
skip = (page - 1) * limit

limit = 10
page 1 → skip 0  → [1-10]
page 2 → skip 10 → [11-20]
page 3 → skip 20 → [21-30]

[বাদ][বাদ]...(skip) → [নাও][নাও]...(limit) → [বাদ][বাদ]...
```

> 💡 **Fix:** `page=-1` দিলে skip negative হয়ে MongoDB error দিত। তাই `Math.max(..., 1)` দিয়ে সর্বনিম্ন 1 রেখেছি। আর page/limit বের করার code `paginate()` আর `getMeta()` দুই জায়গায় একই ছিল, তাই একটা private method `getPagination()`-এ নিয়েছি।

**6. `build()`** — বানানো query return করে, যেটা `await` করলে data আসবে।

**7. `getMeta()`** — frontend-এর pagination UI-র জন্য তথ্য

```
total = 21, limit = 10
totalPage = Math.ceil(21 / 10) = Math.ceil(2.1) = 3
```

> ⚠️ **Bug fix:** Teacher-এর code-এ ছিল `this.modelQuery.model.countDocuments()` — এটা filter/search **ছাড়াই** পুরো collection গোনে। ধরো DB-তে ১০০টা tour, search করে পাওয়া গেল ৩টা, কিন্তু meta-তে `total: 100, totalPage: 10` দেখাবে — frontend ভুল page দেখাবে। তাই `this.modelQuery.getFilter()` দিয়ে একই search/filter condition দিয়ে গুনেছি।

### Code

```ts
// src/app/utils/QueryBuilder.ts
import type { Query } from "mongoose";
import { excludeField } from "../constants";

export class QueryBuilder<T> {
	public modelQuery: Query<T[], T>;
	public readonly query: Record<string, string>;

	constructor(modelQuery: Query<T[], T>, query: Record<string, string>) {
		this.modelQuery = modelQuery;
		this.query = query;
	}

	search(searchableField: string[]): this {
		const searchTerm = this.query.searchTerm?.trim();
		if (!searchTerm) return this; // search না থাকলে কিছু যোগ করব না

		// regex-এর special character escape: "c++" → "c\+\+"
		const safeSearchTerm = searchTerm.replace(/[.*+?^${}()|[\]\\]/g, "\\$&");

		const searchQuery = {
			$or: searchableField.map((field) => ({ [field]: { $regex: safeSearchTerm, $options: "i" } })),
		};

		this.modelQuery = this.modelQuery.find(searchQuery);
		return this;
	}

	filter(): this {
		const filter = { ...this.query };

		for (const field of excludeField) {
			// eslint-disable-next-line @typescript-eslint/no-dynamic-delete
			delete filter[field];
		}

		this.modelQuery = this.modelQuery.find(filter); // Tour.find().find(filter)
		return this;
	}

	sort(): this {
		const sort = this.query.sort || "-createdAt";

		this.modelQuery = this.modelQuery.sort(sort);
		return this;
	}

	fields(): this {
		const fields = this.query.fields?.split(",").join(" ") || ""; // "title,location" → "title location"

		this.modelQuery = this.modelQuery.select(fields);
		return this;
	}

	private getPagination() {
		const page = Math.max(Number(this.query.page) || 1, 1);
		const limit = Math.max(Number(this.query.limit) || 10, 1);
		const skip = (page - 1) * limit;

		return { page, limit, skip };
	}

	paginate(): this {
		const { skip, limit } = this.getPagination();

		this.modelQuery = this.modelQuery.skip(skip).limit(limit);
		return this;
	}

	build() {
		return this.modelQuery;
	}

	async getMeta() {
		const { page, limit } = this.getPagination();

		// search + filter-এর একই condition দিয়ে গোনা
		const totalDocuments = await this.modelQuery.model.countDocuments(this.modelQuery.getFilter());
		const totalPage = Math.ceil(totalDocuments / limit);

		return { page, limit, total: totalDocuments, totalPage };
	}
}
```

- `import type { Query }`: `Query` শুধু type হিসেবে ব্যবহার হচ্ছে।
- `no-dynamic-delete` comment: variable দিয়ে `delete obj[field]` করলে ESLint warning দেয়, এখানে সেটা ইচ্ছাকৃত।
- `Promise.all` (Step 11): data আর meta দুটো আলাদা DB call, একটার জন্য আরেকটা অপেক্ষা না করে একসাথে চালানো, তাই দ্রুত।

### Response উদাহরণ

`GET /api/v1/tour?searchTerm=sea&page=1&limit=2`

```json
{
  "success": true,
  "message": "Tours retrieved successfully",
  "meta": { "page": 1, "limit": 2, "total": 3, "totalPage": 2 },
  "data": [
    { "title": "Cox's Bazar Sea Beach", "slug": "coxs-bazar-sea-beach", "...": "..." },
    { "title": "Saint Martin Sea Trip", "slug": "saint-martin-sea-trip", "...": "..." }
  ]
}
```

---

## সারাংশ

| বিষয় | মনে রাখার কথা |
|---|---|
| Slug | `utils/slug.ts`-এর `slugify` দিয়ে পরিষ্কার, `generateUniqueSlug` দিয়ে model-এর pre hook-এ auto তৈরি; duplicate হলে `-1`, `-2` |
| Hook | `this` লাগে বলে normal function; create → `save`, update → `findOneAndUpdate` |
| `ref` | `model("Division")`-এর string-এর সাথে হুবহু মিলতে হবে |
| `Types` | Interface → `import type { Types } from "mongoose"`, Schema → `Schema.Types.ObjectId` |
| Error | Service-এ `throw new Error` না, সবসময় `AppError` (নাহলে 500) |
| Update | `runValidators: true` দিলে schema validation update-এও চলে |
| QueryBuilder | search → filter → sort → fields → paginate → build, প্রতিটা `return this` |
| Meta | `countDocuments(getFilter())` — filter সহ গুনতে হবে |