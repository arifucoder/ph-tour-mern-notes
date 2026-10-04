# 27 — Search, Filter, Sort, Field Limiting, Pagination → QueryBuilder

## কেন দরকার?

ধরো তোমার কাছে ১০ লাখ বা ২০ লাখ data আছে। সব data কি একসাথে দেখাবে? অবশ্যই না।

User আরও অনেক কিছু চাইবে:
- নির্দিষ্ট data খুঁজতে চাইবে (**filter / search**)
- Alphabet, date বা number অনুযায়ী সাজাতে চাইবে (**sort**)
- Tour-এ অনেক field, কিন্তু হয়তো শুধু title আর কিছু তথ্য দেখাতে চাই (**field limiting**)
- একবারে অল্প অল্প করে দেখাতে চাই (**pagination**)

এখনকার `getAllTours` API সব data একবারে দিয়ে দেয়, কোনো limit নেই, কোনো page নেই। এই customization-গুলো এভাবে সম্ভব না।

এই note-এ প্রথমে একটা practice function `getAllToursOld` বানিয়ে **ধাপে ধাপে** সব feature যোগ করব, তারপর সব logic একটা reusable **QueryBuilder** class-এ নিয়ে যাব।

> Practice-এর জন্য route-এ সাময়িকভাবে `router.get("/getAllToursOld", TourController.getAllToursOld)` যোগ করতে হবে। শেষে এটা মুছে আসল `getAllTours` ব্যবহার করব।

---

# Part 1: `getAllToursOld` দিয়ে ধাপে ধাপে শেখা

## Step 1: Hard-coded Filter

ধরো শুধু Dhaka location-এর tour চাই:

```ts
const getAllToursOld = async () => {
	const tours = await Tour.find({ location: "Dhaka" });

	const totalTours = await Tour.countDocuments({ location: "Dhaka" });

	return {
		data: tours,
		meta: {
			total: totalTours,
		},
	};
};
```

Dhaka location-এর সব tour আসবে। কিন্তু এটা **hard-coded**: আমরা নিজেরা "Dhaka" লিখে দিয়েছি। আমরা চাই user নিজে location দেবে, সেই অনুযায়ী data আসবে:

```
localhost:5000/api/v1/tour/getAllToursOld?location=Dhaka
```

---

## Step 2: URL থেকে Query নেওয়া

URL-এ `?`-এর পরে যা থাকে তাকে বলে **query** (query parameter)। Express এটা `req.query`-তে object হিসেবে দেয়।

```ts
// tour.controller.ts
const getAllToursOld = catchAsync(async (req: Request, res: Response) => {
	const query = req.query;
	const result = await TourService.getAllToursOld(query as Record<string, string>);
	sendResponse(res, {
		statusCode: httpStatus.OK,
		success: true,
		message: "Tours retrieved successfully",
		data: result.data,
		meta: result.meta,
	});
});
```

### `Record<string, string>` কেন?

`req.query` একটা object, কিন্তু এর ভেতরে কোন কোন property আসবে (location? searchTerm? page?) আগে থেকে জানি না। যখন object-এর key আর value কেমন হবে নির্দিষ্টভাবে জানি না, তখন `Record<string, string>` দিই। মানে: "key যেকোনো string, value-ও string"।

Service-এ `console.log(query)` করলে terminal-এ দেখাবে:

```
[Object: null prototype] { location: 'Dhaka' }
```

> ❓ **`[Object: null prototype]` কী?** Express query-র জন্য এমন object বানায় যার কোনো prototype নেই (সাধারণ object-এর মতো `toString` ইত্যাদি method নেই)। এটা নিরাপত্তার জন্য করা হয়। আমাদের কাজে এতে কোনো পার্থক্য নেই, এটা সাধারণ object-এর মতোই ব্যবহার করা যায়।

> ⚠️ `console.log` শুধু বোঝার জন্য। কাজ শেষে মুছে দেবে (ESLint `no-console` warning দেয়)।

---

## Step 3: Query দিয়ে Dynamic Filter

এখন query-টাই সরাসরি `find()`-এ বসিয়ে দিই। পড়তে সুবিধার জন্য একটা `filter` variable-এ রাখি:

```ts
const getAllToursOld = async (query: Record<string, string>) => {
	const filter = query;
	const tours = await Tour.find(filter); // Tour.find({ location: "Dhaka" })

	// ...
};
```

এখন `?location=Dhaka` দিলে Dhaka-র tour, `?location=Barishal` দিলে Barishal-এর tour।

> ⚠️ **Case-sensitive:** DB-তে location যেভাবে save করা আছে (`"Dhaka"`), ঠিক সেভাবেই দিতে হবে। `?location=dhaka` দিলে কিছুই পাবে না, কারণ আমরা lowercase বা case handle করিনি।

---

## Filter vs Search — পার্থক্য

| | Filter | Search |
|---|---|---|
| Match | **Exact match** — হুবহু মিলতে হবে | **Partial match** — অংশ মিললেই হবে |
| উদাহরণ | `location=Dhaka` → শুধু location ঠিক "Dhaka" | `searchTerm=fish` → "Fishing in Barishal", "Fish Market Tour" |
| কাজ | "Dhaka location-এ tour আছে কিনা দেখো" | "Dhaka-র মধ্যে golf নামে কোনো tour আছে কিনা খোঁজো" |

Search হলো Google-এর মতো: পুরো লেখা লাগে না। "yt" লিখলেও Google বোঝে তুমি YouTube খুঁজছ। অল্প কিছু লেখার পরও related data এনে দেওয়াই হলো searching।

---

## Step 4: Hard-coded Search (`$regex`)

ধরো DB-তে বিভিন্ন জায়গায় অনেক Fishing tour আছে। "fishing" লিখে search করলে সবগুলো দেখাতে চাই।

```ts
const tours = await Tour.find({ title: "fishing" }); // ❌ কিছুই আসবে না
```

এটা কাজ করবে না, কারণ এটা exact match খোঁজে: title হুবহু `"fishing"` হতে হবে। পুরো title দিলে আসবে, কিন্তু আমরা চাই একটা শব্দ লিখলেই মিলিয়ে আনুক।

এর জন্য MongoDB-র **`$regex`**:

```ts
const tours = await Tour.find({
	title: { $regex: "Fish" },
});
```

এখন title-এর **যেকোনো জায়গায়** "Fish" থাকলেই আসবে।

---

## Step 5: Dynamic Search + Case-insensitive

Hard-coded না রেখে user-কে পাঠাতে দিই:

```
localhost:5000/api/v1/tour/getAllToursOld?searchTerm=fish
```

```ts
const searchTerm = query.searchTerm;

const tours = await Tour.find({
	title: { $regex: searchTerm },
});
```

কিন্তু কোনো data আসবে না! কারণ `$regex` **case-sensitive**: আমরা পাঠিয়েছি ছোট হাতের `fish`, কিন্তু DB-তে আছে `Fish`।

সমাধান: **`$options: "i"`** (i = insensitive)

```ts
const tours = await Tour.find({
	title: { $regex: searchTerm, $options: "i" },
});
```

এখন fish, Fish, FISH সব একই ধরা হবে।

---

## Step 6: `searchTerm` না দিলে Error

`searchTerm` ছাড়া request পাঠালে error আসে:

```json
{ "message": "$regex has to be a string" }
```

কারণ `query.searchTerm` তখন `undefined`, আর `$regex`-এ string লাগে। তাই না থাকলে empty string দিই:

```ts
const searchTerm = query.searchTerm || "";
```

Empty string (`""`) regex সব কিছুর সাথে মেলে, তাই search না দিলে সব data আসবে।

> 💡 **আরও ভালো উপায়:** searchTerm না থাকলে search query যোগই না করা (Final code-এ এভাবেই করেছি)। কারণ field না থাকলে regex মেলে না — যেমন কোনো tour-এর `description` না থাকলে `description: { $regex: "" }` সেই tour-কে বাদ দিতে পারে।

---

## Step 7: একাধিক Field-এ Search (`$or`)

User শুধু title-এ না, description বা অন্য field-এও খুঁজতে চাইতে পারে। Title-এ না পেলে description-এ, সেখানেও না পেলে অন্য field-এ।

```ts
const tours = await Tour.find({
	$or: [
		{ title: { $regex: searchTerm, $options: "i" } },
		{ description: { $regex: searchTerm, $options: "i" } },
		{ included: { $regex: searchTerm, $options: "i" } },
	],
});
```

**`$or`** মানে: এই field-গুলোর **যেকোনো একটায়** partial match হলেই হবে।

> 💡 `included` একটা array (`["খাবার", "Hotel"]`)। Array field-এ `$regex` দিলে MongoDB array-র প্রতিটা item-এ খোঁজে, যেকোনো একটা মিললেই হবে।

---

## Step 8: `map` দিয়ে Dynamic বানানো

প্রতিটা field-এর জন্য একই লাইন বারবার লিখছি। Field-এর নামগুলো একটা array-তে রেখে `map` দিয়ে বানাই (`map` নতুন array return করে):

```ts
const searchArray = tourSearchableFields.map((field) => ({ [field]: { $regex: searchTerm, $options: "i" } }));

const tours = await Tour.find({
	$or: searchArray,
});
```

### ❓ `[field]` কেন?

এটাকে বলে **computed property name**। Object-এর key হিসেবে variable-এর **value** বসাতে চাইলে `[]` দিতে হয়।

```ts
const field = "title";

{ field: "x" }   // → { field: "x" }   ❌ key-টা আক্ষরিকভাবে "field"
{ [field]: "x" } // → { title: "x" }   ✅ variable-এর value key হলো
```

### `map`-এ `{}`-এর বদলে `()` কেন?

Arrow function-এ `{}` দিলে JavaScript সেটাকে function-এর **body** ভাবে, object না। তাই সরাসরি object return করতে চাইলে `()` দিয়ে মুড়িয়ে দিতে হয়:

```ts
(field) => { title: "x" }    // ❌ body ভাবে, কিছু return হয় না
(field) => ({ title: "x" })  // ✅ object return হয়
```

আরও clean করে:

```ts
const getAllToursOld = async (query: Record<string, string>) => {
	const filter = query;
	const searchTerm = query.searchTerm || "";

	const tourSearchableFields = ["title", "description", "location"];

	const searchQuery = {
		$or: tourSearchableFields.map((field) => ({ [field]: { $regex: searchTerm, $options: "i" } })),
	};

	const tours = await Tour.find(searchQuery);

	// ...
};
```

---

## Step 9: Searchable Field-গুলো Constant File-এ

`tourSearchableFields` বদলাবে না, আর পরে অন্য জায়গাতেও লাগতে পারে। তাই tour module-এ আলাদা file-এ রাখি:

```ts
// src/app/modules/tour/tour.constant.ts
export const tourSearchableFields = ["title", "description", "location"];
```

---

## Step 10: Search + Filter একসাথে — প্রথম সমস্যা

```ts
const tours = await Tour.find(searchQuery).find(filter);
```

> Mongoose-এ একাধিক `.find()` chain করলে সব condition **merge** হয়ে যায় (AND)। মানে search-ও মিলতে হবে, filter-ও মিলতে হবে।

এখন request পাঠাই:

```
localhost:5000/api/v1/tour/getAllToursOld?location=Barishal&searchTerm=fishing
```

**কোনো data আসে না! কেন?**

`filter`-এ আমরা পুরো `query` রেখে দিয়েছি:

```
[Object: null prototype] { location: 'Barishal', searchTerm: 'fishing' }
```

তাই Mongoose খুঁজছে: location `"Barishal"` **এবং** `searchTerm` নামের field-এর value `"fishing"`। কিন্তু DB-তে `searchTerm` নামে কোনো field নেই! এটা আমরা শুধু search-এর জন্য বানিয়েছি। তাই কিছুই মেলে না।

**সমাধান:** filter থেকে `searchTerm` সরিয়ে দাও:

```ts
const filter = query;
const searchTerm = query.searchTerm || ""; // delete-এর আগেই value নিয়ে রাখতে হবে

delete filter["searchTerm"];
```

> ⚠️ **খেয়াল রাখো:** `const filter = query` কোনো copy বানায় না, দুটো **একই object**-কে দেখায়। তাই `filter` থেকে delete করলে `query` থেকেও মুছে যায়। এজন্যই `searchTerm`-এর value delete-এর **আগে** নিয়ে রাখতে হয়। (QueryBuilder-এ আমরা সঠিকভাবে copy বানাব, Step 21-এ।)

---

## Step 11: Sorting

Sort মানে কোনো field অনুযায়ী **ascending** (ছোট → বড়, A → Z) বা **descending** (বড় → ছোট, Z → A) সাজানো।

```ts
const tours = await Tour.find(searchQuery).find(filter).sort("location"); // A → Z
```

```ts
const tours = await Tour.find(searchQuery).find(filter).sort("-location"); // Z → A
```

নামের আগে **minus (`-`)** দিলে descending।

Dynamic বানাই:

```ts
const sort = query.sort || "-createdAt";

const tours = await Tour.find(searchQuery).find(filter).sort(sort);
```

কিছু না দিলে default `-createdAt`, মানে **সবশেষে যোগ করা data সবার আগে**।

---

## Step 12: Sort-এও একই সমস্যা → `excludeField`

```
localhost:5000/api/v1/tour/getAllToursOld?sort=location
```

আবার কোনো data আসে না!

```
[Object: null prototype] { sort: 'location' }
```

> ❓ **Filter-এর মধ্যে `sort` কীভাবে চলে গেল?**
> আমরা কিছু "পাঠাইনি", এটা আগে থেকেই ছিল। `filter = query` মানে **পুরো query** filter। URL-এ `?sort=location` দিয়েছি, তাই query-তে `sort` আছে, ফলে filter-এও আছে। Mongoose তখন `sort` নামে field খুঁজছে যার value `"location"` — DB-তে এমন field নেই, তাই কিছুই আসে না।
>
> নিয়ম: **query-তে যা কিছু DB-র field না** (searchTerm, sort, page...), সেগুলো filter-এ যাওয়ার আগে বাদ দিতে হবে।

আবার delete করতে হবে:

```ts
delete filter["searchTerm"];
delete filter["sort"];
```

কিন্তু প্রতিবার আলাদা delete লেখা ভালো না, আর এমন field আরও আসবে (fields, page, limit)। তাই একটা array আর loop:

```ts
const excludeField = ["searchTerm", "sort"];

for (const field of excludeField) {
	delete filter[field];
}
```

এই list সব module-এর API-তে একই থাকবে, তাই app-level-এ রাখি:

```ts
// src/app/constants.ts
export const excludeField = ["searchTerm", "sort", "fields", "page", "limit"];
```

---

## Step 13: Field Limiting (কোন field দেখাবে)

Data থেকে কিছু field নিতে বা বাদ দিতে `.select()`:

```ts
const tours = await Tour.find(searchQuery).find(filter).sort(sort).select("title");
```

→ শুধু `_id` আর `title` আসবে (`_id` default-ভাবে সবসময় আসে)।

```ts
.select("-title")
```

→ `title` বাদে **বাকি সব** আসবে।

Dynamic বানাই:

```ts
const fields = query.fields || "";

const tours = await Tour.find(searchQuery).find(filter).sort(sort).select(fields);
```

আবার একই সমস্যা (`fields` filter-এ চলে যায়), তাই `excludeField`-এ `"fields"` যোগ করলেই ঠিক।

### একাধিক field

Mongoose `select()` চায় **space দিয়ে** আলাদা করা string:

```ts
.select("title location") // ✅ title আর location দুটোই
```

কিন্তু URL-এ comma দেওয়া সহজ (`fields=title,description`)। তাই comma-কে space বানাই:

```ts
const fields = query.fields?.split(",").join(" ") || "";
// "title,description" → ["title", "description"] → "title description"
```

```
localhost:5000/api/v1/tour/getAllToursOld?fields=title,description
```

এটাকেই বলে **field filtering / field limiting**।

> ⚠️ **সংশোধন:** `?fields=title&fields=description` (একই key দুবার) দিলে Express value-টাকে **array** বানায়: `["title", "description"]`। `split()` এখন যোগ হওয়ার পর array-তে `split` নেই, তাই server crash করবে (500)। তাই সবসময় comma দিয়ে পাঠাতে হবে: `?fields=title,description`।

---

## Step 14: Pagination-এর ধারণা — `skip` আর `limit`

ধরো প্রতিটা box একটা data:

```
[1][2][3][4][5][6][7][8][9][10]
```

- **skip:** শুরু থেকে কয়টা **বাদ** দেব
- **limit:** তারপর কয়টা **নেব**

```
skip(3):  [বাদ][বাদ][বাদ][4][5][6][7][8][9][10]
limit(5): [1][2][3][4][5] ← প্রথম ৫টা
```

Mongoose-এ দুটোই function:

```ts
.limit(2) // প্রথম ২টা
.skip(2)  // প্রথম ২টা বাদে বাকি সব
```

---

## Step 15: Skip-এর সূত্র

ধরো প্রতি page-এ ১০টা দেখাব, মোট data ২১টা:

| Page | Skip | Limit | যা দেখাবে |
|---|---|---|---|
| 1 | 0 | 10 | 1–10 |
| 2 | 10 | 10 | 11–20 |
| 3 | 20 | 10 | 21 (শুধু ১টা বাকি) |

Page 3-এর আগে ২টা page আছে → `2 × 10 = 20` skip। Page 4-এর আগে ৩টা page → `(4 − 1) × 10 = 30`।

```
skip = (page − 1) × limit
```

> ✏️ **সংশোধন:** সূত্রে `10` না, `limit` বসবে। কারণ user প্রতি page-এ কয়টা চায় সেটা বদলাতে পারে।

শুধু `skip` দিলে হবে না, `limit`-ও দিতে হবে। নাহলে skip-এর পরের **সব** data চলে আসবে।

---

## Step 16: Query থেকে `page` আর `limit`

```
localhost:5000/api/v1/tour/getAllToursOld?page=2&limit=10
```

```ts
const page = Number(query.page) || 1;
const limit = Number(query.limit) || 10;
const skip = (page - 1) * limit;

const tours = await Tour.find(searchQuery).find(filter).sort(sort).select(fields).skip(skip).limit(limit);
```

> **`Number()` কেন?** Query-র সব value **string** আসে (`"2"`, `"10"`)। হিসাব করার জন্য number লাগে। আর `Number("abc")` হলে `NaN` হয়, তখন `||` দিয়ে default (1 বা 10) বসে।

`excludeField`-এ `"page"` আর `"limit"` আগেই যোগ করা আছে (Step 12)।

---

## Step 17: Meta Data

Frontend-এর pagination UI (page number, next/prev button) বানাতে কিছু তথ্য লাগে:

```ts
const meta = {
	page: 1,       // এখন কোন page
	limit: 10,     // প্রতি page-এ কয়টা
	total: 27,     // মোট কয়টা data
	totalPage: 3,  // মোট কয়টা page
};
```

`totalPage = Math.ceil(total / limit)` → `Math.ceil(21 / 10) = Math.ceil(2.1) = 3`। (`ceil` সবসময় উপরের পূর্ণসংখ্যায় নেয়, কারণ বাকি ১টা data-র জন্যও একটা page লাগবে।)

```ts
const getAllToursOld = async (query: Record<string, string>) => {
	const filter = query;
	const searchTerm = query.searchTerm || "";
	const sort = query.sort || "-createdAt";
	const fields = query.fields?.split(",").join(" ") || "";
	const page = Number(query.page) || 1;
	const limit = Number(query.limit) || 10;
	const skip = (page - 1) * limit;

	for (const field of excludeField) {
		// eslint-disable-next-line @typescript-eslint/no-dynamic-delete
		delete filter[field];
	}

	const searchQuery = {
		$or: tourSearchableFields.map((field) => ({ [field]: { $regex: searchTerm, $options: "i" } })),
	};

	const tours = await Tour.find(searchQuery).find(filter).sort(sort).select(fields).skip(skip).limit(limit);

	// ✅ search আর filter-এর একই condition দিয়ে গুনতে হবে
	const totalTours = await Tour.countDocuments({ ...searchQuery, ...filter });
	const totalPage = Math.ceil(totalTours / limit);

	return {
		data: tours,
		meta: { page, limit, total: totalTours, totalPage },
	};
};
```

> ⚠️ **Bug fix:** Teacher-এর code-এ ছিল `Tour.countDocuments()` — কোনো condition ছাড়া, তাই পুরো collection গোনে। ধরো DB-তে ১০০টা tour, search করে পাওয়া গেল ৩টা, কিন্তু meta-তে `total: 100, totalPage: 10` দেখাবে। Frontend তখন ১০টা page দেখাবে, অথচ ২য় page থেকে সব খালি। তাই search + filter-এর condition দিয়েই গুনতে হবে।

> ❓ **`Tour.countDocuments()` দিলেই তো total আসে, তাহলে `{ ...searchQuery, ...filter }` পাঠাতে হবে কেন?**
>
> `countDocuments()` ফাঁকা দিলে **DB-র সব tour** গোনে। কিন্তু meta-র `total` বোঝায় **এই request-এর result-এ কয়টা data আছে**, পুরো DB-তে কয়টা না। কারণ `totalPage` এই total থেকেই হিসাব হয়, আর frontend সেই অনুযায়ী page button দেখায়।
>
> উদাহরণ: DB-তে মোট ১০০টা tour, তার মধ্যে Barishal-এ ৩টা। Request: `?location=Barishal&limit=10`
>
> | | `countDocuments()` | `countDocuments({ ...searchQuery, ...filter })` |
> |---|---|---|
> | `total` | 100 ❌ | 3 ✅ |
> | `totalPage` | 10 ❌ | 1 ✅ |
> | Frontend দেখাবে | Page 1–10 button, কিন্তু ২ থেকে ১০ সব খালি | শুধু Page 1, যেখানে ৩টা data |
>
> তাই নিয়ম: **`find()`-এ যে condition দিয়ে data আনছি, `countDocuments()`-এও ঠিক সেই condition দিয়ে গুনতে হবে।** শুধু `skip`/`limit` দিই না, কারণ সেগুলো page ভাগ করে, মোট সংখ্যা বদলায় না।
>
> `{ ...searchQuery, ...filter }` মানে দুটো object জুড়ে একটা condition বানানো: `{ $or: [...], location: "Barishal" }` — ঠিক যেমন `.find(searchQuery).find(filter)` দুটো merge করে।

> `no-dynamic-delete` comment: variable দিয়ে `delete obj[field]` করলে ESLint error দেয়। এখানে এটা ইচ্ছাকৃত, তাই comment দিয়ে বন্ধ করেছি।

---

## Step 18: `await` পরে দেওয়া — Query ধাপে ধাপে বানানো

এখন সব এক লাইনে লম্বা chain। চাইলে ভেঙে লেখা যায়:

```ts
const filterQuery = Tour.find(filter);              // await নেই
const tours = filterQuery.find(searchQuery);        // await নেই
const allTours = await tours.sort(sort).select(fields).skip(skip).limit(limit); // এখানে await
```

**`await` না দিলে কী হয়?** Mongoose-এর `Tour.find()` সাথে সাথে DB-তে যায় না, শুধু একটা **query object** বানায় (কী খুঁজতে হবে তার plan)। `await` দিলে তখন query চলে আর data আসে।

> ✏️ **সংশোধন:** "`await` দিলে একটা document হয়ে যায়" — আসলে `find()`-এ `await` দিলে **document-এর array** আসে (`findOne`/`findById` দিলে একটা document)। আর একবার `await` করার পর সেটা আর query থাকে না, তাই তার পরে `.sort()` বা `.find()` যোগ করা যায় না।

**লাভ:** Query ধাপে ধাপে বানিয়ে শুধু **শেষে, যখন data দরকার**, তখন `await`। এই ধারণাটাই QueryBuilder-এর ভিত্তি।

---

# Part 2: QueryBuilder Class বানানো

## Step 19: Class কেন?

Search, filter, sort... সব কাজ সব module-এ লাগবে (all users, all tours, all tour types, bookings)। প্রতিবার এত code কপি করা ঠিক না। তাই সব logic একটা **class**-এ রাখব।

Class-এ সহজে **method chaining** করা যায়:

```ts
queryBuilder.search().filter().sort().fields().paginate()
```

কারণ class নিজের ভেতরে state রাখে (`this.modelQuery`), আর প্রতিটা method সেই state বদলে **`return this`** করে, তাই পরের method একই object-এ call করা যায়।

> ✏️ **একটু সংশোধন:** Function দিয়েও chaining সম্ভব (যদি function একটা object return করে)। কিন্তু class-এ `this` দিয়ে state রাখা আর chain করা অনেক সহজ ও পরিষ্কার, তাই class বেছে নেওয়া।

Class-এ দুটো জিনিস পাঠাব:

```ts
new QueryBuilderProvia(Tour.find(), query)
//                      ▲ model query   ▲ URL-এর query
```

1. **Model query** (`Tour.find()`) — `await` ছাড়া, যাতে ভেতরে এর উপর search, filter, sort ইত্যাদি যোগ করা যায়। `await` দিলে এখানেই resolve হয়ে যেত।
2. **Query object** — যেটা `getAllToursOld(query)`-এর parameter-এ পেতাম।

বিনিময়ে class আমাদের সব tour দেবে, service-এ আর কোনো কাজ করতে হবে না।

> নাম `QueryBuilderProvia` কারণ project-এ আগে থেকেই `QueryBuilder` ছিল। শেষে final version `QueryBuilder` নামেই থাকবে।

---

## Step 20: Property আর Constructor

```ts
class QueryBuilderProvia<T> {
	public modelQuery: Query<T[], T>;
	public readonly query: Record<string, string>;

	constructor(modelQuery: Query<T[], T>, query: Record<string, string>) {
		this.modelQuery = modelQuery;
		this.query = query;
	}
}
```

### Generic `<T>` কেন?

এই class শুধু Tour-এর জন্য না, User, TourType, Booking সবার জন্য। তাই type হিসেবে `Tour` লিখে দিলে অন্য কোথাও ব্যবহার করা যেত না। `T` হলো "যে model দেবে, সেই type"।

### `Query<T[], T>` মানে

| অংশ | মানে |
|---|---|
| `T[]` | `await` করলে যা আসবে → T-র array (অনেক document) |
| `T` | প্রতিটা document দেখতে কেমন → T-র মতো |

> ✏️ নোটে `Query<T[]>` ছিল, teacher-এর final code-এ `Query<T[], T>`। দ্বিতীয় parameter-টা দিলে Mongoose document-এর type ঠিকভাবে বোঝে, তাই এটাই রাখব।

### `readonly` কেন?

`query`-তে আমরা কোনো কাটাছেঁড়া, edit বা update করব না, শুধু পড়ব। `readonly` দিলে ভুল করেও বদলাতে গেলে TypeScript error দেবে।

---

## Step 21: `filter()` Method

```ts
filter(): this {
	const filter = { ...this.query }; // copy

	for (const field of excludeField) {
		// eslint-disable-next-line @typescript-eslint/no-dynamic-delete
		delete filter[field];
	}

	this.modelQuery = this.modelQuery.find(filter); // Tour.find().find(filter)

	return this;
}
```

### `{ ...this.query }` — copy কেন জরুরি?

Step 10-এ দেখেছি, `const filter = query` copy বানায় না। এখানে যদি `this.query` থেকেই delete করতাম, তাহলে `sort`, `page`, `limit` সব মুছে যেত — পরে `sort()` বা `paginate()` method আর এগুলো পেত না! Spread (`...`) দিয়ে নতুন object বানালে মূল `this.query` অক্ষত থাকে। (`readonly`-ও এখানে সাহায্য করে।)

### `return this` কেন?

Method শেষে নিজেকেই (পুরো QueryBuilder object) return করছে, তাই পরের method chain করা যায়: `.filter().sort()`।

> ⚠️ **সংশোধন:** নোটে `filter(): any` ছিল। `any` দিলে ESLint error দেয়, আর TypeScript বুঝতে পারে না return-টা কী — তখন পরের `.sort()`-এ autocomplete বা type check কাজ করে না। সঠিক return type **`this`**।

---

## Step 22: Service-এ ব্যবহার

```ts
const getAllToursOld = async (query: Record<string, string>) => {
	const modelQuery = new QueryBuilderProvia(Tour.find(), query);

	const tours = await modelQuery.modelQuery;

	return { data: tours, meta: {} };
};
```

এখানে variable-এর নাম `modelQuery` দেওয়া হয়েছে শুধু বোঝানোর জন্য যে ভেতরে একটা model query আছে। আসলে এটা একটা QueryBuilder, তাই নাম `queryBuilder` দেওয়াই ঠিক।

`filter()` call করতে:

```ts
const getAllToursOld = async (query: Record<string, string>) => {
	const queryBuilder = new QueryBuilderProvia(Tour.find(), query);

	const tours = await queryBuilder.filter().modelQuery;

	return { data: tours, meta: {} };
};
```

```
localhost:5000/api/v1/tour/getAllToursOld?location=Dhaka
```

→ Dhaka location-এর সব data আসবে।

> ❓ **শেষে `.modelQuery` কী?**
>
> এটা QueryBuilder class-এর ভেতরের **property** — Step 20-এ যেটা বানিয়েছিলাম: `public modelQuery: Query<T[], T>`। এর ভেতরে থাকে আসল **Mongoose query** (`Tour.find()...`)।
>
> ```
> queryBuilder.filter()             → QueryBuilder object (return this)
> queryBuilder.filter().modelQuery  → ভেতরের Mongoose query
> await ...modelQuery               → DB-তে query চলে, data আসে
> ```
>
> `filter()` নিজে data দেয় না, শুধু `this.modelQuery`-তে condition যোগ করে পুরো QueryBuilder object return করে। QueryBuilder-কে `await` করলে কিছু হবে না, কারণ এটা Mongoose query না, আমাদের বানানো class। তাই ভেতর থেকে `.modelQuery` বের করে `await` করি, তখন DB-তে যায়।
>
> একটা বাক্সের মতো ভাবো: **QueryBuilder হলো বাক্স, `modelQuery` হলো বাক্সের ভেতরের আসল জিনিস।** Method-গুলো বাক্সের ভেতরের জিনিসে কাজ করে, আর শেষে `.modelQuery` দিয়ে জিনিসটা বের করে নিই। পরে Step 26-এ এই কাজটাই সুন্দর নামে `build()` method দিয়ে করব।

---

## Step 23: `search()` Method

```ts
search(searchableField: string[]): this {
	const searchTerm = this.query.searchTerm || "";

	const searchQuery = {
		$or: searchableField.map((field) => ({
			[field]: { $regex: searchTerm, $options: "i" },
		})),
	};

	this.modelQuery = this.modelQuery.find(searchQuery);

	return this;
}
```

Searchable field parameter হিসেবে নিচ্ছি, কারণ প্রতিটা module-এর searchable field আলাদা (tour-এর title/location, user-এর name/email)।

```ts
const tours = await queryBuilder.search(tourSearchableFields).filter().modelQuery;
```

```
localhost:5000/api/v1/tour/getAllToursOld?location=Dhaka&searchTerm=heritage
```

→ Dhaka location-এ যেসব tour-এ "heritage" আছে। ✅

---

## Step 24: Class আলাদা File-এ

এই class সব module-এ (all users, all tours, all tour types) লাগবে, তাই service file-এ রাখা ঠিক না। `utils`-এ নিয়ে যাই:

```
src/app/utils/QueryBuilder.ts
```

---

## Step 25: `sort()`, `fields()`, `paginate()` Method

Part 1-এর logic-গুলোই method-এ নিয়ে আসা, প্রতিটা `this.query` থেকে value নেয়, `this.modelQuery`-তে যোগ করে, `return this`:

```ts
sort(): this {
	const sort = this.query.sort || "-createdAt";
	this.modelQuery = this.modelQuery.sort(sort);
	return this;
}

fields(): this {
	const fields = this.query.fields?.split(",").join(" ") || "";
	this.modelQuery = this.modelQuery.select(fields);
	return this;
}

paginate(): this {
	const page = Number(this.query.page) || 1;
	const limit = Number(this.query.limit) || 10;
	const skip = (page - 1) * limit;

	this.modelQuery = this.modelQuery.skip(skip).limit(limit);
	return this;
}
```

এখন call:

```ts
const tours = await queryBuilder.search(tourSearchableFields).filter().sort().fields().paginate().modelQuery;
```

---

## Step 26: `build()` Method

শেষে `.modelQuery` লিখতে হচ্ছে, দেখতে অসুন্দর আর ভেতরের property বাইরে থেকে ধরা হচ্ছে। তাই একটা method:

```ts
build() {
	return this.modelQuery;
}
```

```ts
const tours = await queryBuilder.search(tourSearchableFields).filter().sort().fields().paginate().build();
```

`build()` মানে "query বানানো শেষ, এবার দাও" — `await` করলে DB-তে যায়।

---

## Step 27: `getMeta()` Method

Meta-র কাজও QueryBuilder-এ নিয়ে আসি:

```ts
async getMeta() {
	const totalDocuments = await this.modelQuery.model.countDocuments(this.modelQuery.getFilter());

	const page = Number(this.query.page) || 1;
	const limit = Number(this.query.limit) || 10;

	const totalPage = Math.ceil(totalDocuments / limit);

	return { page, limit, total: totalDocuments, totalPage };
}
```

- **`this.modelQuery.model`**: query-টা যে model-এর (এখানে `Tour`)। তাই QueryBuilder না জেনেও ঠিক model-এ গুনতে পারে।
- **`this.modelQuery.getFilter()`**: এখন পর্যন্ত query-তে যোগ হওয়া সব condition (search + filter)।

> ⚠️ **Bug fix:** Teacher-এর code-এ `countDocuments()` ফাঁকা ছিল — Step 17-এর মতোই filter ছাড়া পুরো collection গুনত। `getFilter()` দিয়ে ঠিক করেছি।

---

## Step 28: Data আর Meta একসাথে — `Promise.all`

প্রথমে এভাবে call:

```ts
const tours = await queryBuilder.search(tourSearchableFields).filter().sort().fields().paginate().build();
const meta = await queryBuilder.getMeta();
```

> ❓ **"QueryBuilder দুবার call করা যাবে না" error কেন এসেছিল?**
> Mongoose-এর একটা query object **একবারই** execute করা যায়। একই query আবার `await` করলে error দেয়: `Query was already executed`। যদি meta-তে `this.modelQuery.countDocuments()` (মানে **একই** query object-এ) লেখা থাকত, তাহলে data আনার সময় query একবার চলেছে, meta আনতে গিয়ে আবার চালাতে চাইছে → error।
>
> সমাধান হলো meta-তে **নতুন** query বানানো: `this.modelQuery.model.countDocuments(...)` — এটা model থেকে একদম নতুন query, তাই আর error হয় না। এখন উপরের দুই লাইনের code-ও কাজ করবে।

Final-এ দুটো একসাথে চালাই:

```ts
const getAllTours = async (query: Record<string, string>) => {
	const queryBuilder = new QueryBuilder(Tour.find(), query);

	const tours = queryBuilder.search(tourSearchableFields).filter().sort().fields().paginate();

	const [data, meta] = await Promise.all([tours.build(), queryBuilder.getMeta()]);

	return { data, meta };
};
```

**`Promise.all` কেন?** Data আনা আর গোনা দুটো আলাদা DB call, একটা আরেকটার উপর নির্ভর করে না। আলাদা `await` দিলে একটা শেষ হলে আরেকটা শুরু হয়। `Promise.all` দুটোকে **একসাথে (parallel)** চালায়, তাই দ্রুত।

> ✏️ **সংশোধন:** "Array-তে যেটা আগে দেওয়া, সেটা আগে resolve হবে" — এটা ঠিক না। `Promise.all`-এ সব একসাথে চলে, যেটা আগে শেষ হয় হোক। কিন্তু **result array-র order সবসময় input-এর order-এর মতো থাকে**। তাই `[tours.build(), getMeta()]` দিলে result-এ প্রথমটা data, দ্বিতীয়টা meta। Destructure-এ `[data, meta]` সেই একই order-এ লিখতে হবে — উল্টো লিখলে `data`-তে meta চলে আসবে।

> ✏️ **সংশোধন:** Teacher-এর code-এ `const tours = await queryBuilder...paginate()` ছিল। `paginate()` Promise না, QueryBuilder object return করে, তাই এখানে `await`-এর কোনো কাজ নেই। সরিয়ে দিয়েছি। আসল DB call হয় `Promise.all`-এর ভেতরে।

---

# Part 3: Final Code

Teacher-এর final code, উপরে বলা bug fix আর ছোট improvement সহ। এগুলো note 26-এর final code-এর সাথে হুবহু এক।

### Improvement-গুলো এক নজরে

| পরিবর্তন | কেন |
|---|---|
| `countDocuments(this.modelQuery.getFilter())` | Filter/search সহ সঠিক total |
| searchTerm না থাকলে search বাদ | অপ্রয়োজনীয় regex নয়, missing field-এর document বাদ পড়বে না |
| Regex special character escape | `(`, `*` দিলে 500 error, আর ReDoS (জটিল pattern দিয়ে server ধীর করা) আটকানো |
| `Math.max(..., 1)` | `page=-1` দিলে negative skip-এ MongoDB error |
| `getPagination()` private method | page/limit-এর হিসাব দুই জায়গায় লেখা লাগবে না |
| `import type { Query }` | শুধু type হিসেবে ব্যবহার (`verbatimModuleSyntax`) |
| Service-এ অপ্রয়োজনীয় `await` সরানো | `paginate()` Promise না |

### `src/app/constants.ts`

```ts
export const excludeField = ["searchTerm", "sort", "fields", "page", "limit"];
```

### `src/app/modules/tour/tour.constant.ts`

```ts
export const tourSearchableFields = ["title", "description", "location"];
```

### `src/app/utils/QueryBuilder.ts`

```ts
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
		const filter = { ...this.query }; // copy, মূল query অক্ষত থাকবে

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

		// search + filter-এর একই condition দিয়ে নতুন query-তে গোনা
		const totalDocuments = await this.modelQuery.model.countDocuments(this.modelQuery.getFilter());
		const totalPage = Math.ceil(totalDocuments / limit);

		return { page, limit, total: totalDocuments, totalPage };
	}
}
```

### `src/app/modules/tour/tour.service.ts`

```ts
import { QueryBuilder } from "../../utils/QueryBuilder";
import { tourSearchableFields } from "./tour.constant";
import { Tour } from "./tour.model";

const getAllTours = async (query: Record<string, string>) => {
	const queryBuilder = new QueryBuilder(Tour.find(), query);

	const tours = queryBuilder.search(tourSearchableFields).filter().sort().fields().paginate();

	const [data, meta] = await Promise.all([tours.build(), queryBuilder.getMeta()]);

	return {
		data,
		meta,
	};
};
```

### `src/app/modules/tour/tour.controller.ts`

```ts
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
```

> 💡 `sendResponse`-এর `meta` type-এ যদি শুধু `total` থাকে, তাহলে সেখানে `page`, `limit`, `totalPage`-ও optional হিসেবে যোগ করে নিও, যাতে type ঠিক থাকে।

> Practice-এর `getAllToursOld` function, controller আর route এখন মুছে দিতে পারো।

---

## সব কিছু একসাথে — URL উদাহরণ

```
GET /api/v1/tour?searchTerm=sea&location=Cox's Bazar&sort=-costFrom&fields=title,costFrom&page=2&limit=5
```

| Query | Method | Mongoose-এ যা হয় |
|---|---|---|
| `searchTerm=sea` | `search()` | `.find({ $or: [{ title: /sea/i }, ...] })` |
| `location=Cox's Bazar` | `filter()` | `.find({ location: "Cox's Bazar" })` |
| `sort=-costFrom` | `sort()` | `.sort("-costFrom")` → দাম বেশি থেকে কম |
| `fields=title,costFrom` | `fields()` | `.select("title costFrom")` |
| `page=2&limit=5` | `paginate()` | `.skip(5).limit(5)` |

```
Tour.find()
   │ .search()   → $or + $regex
   │ .filter()   → exact match (excludeField বাদ দিয়ে)
   │ .sort()     → default "-createdAt"
   │ .fields()   → select
   │ .paginate() → skip + limit
   ▼
.build() ──┐
           ├─ Promise.all → [data, meta]
.getMeta() ┘
```

---

## সারাংশ

| বিষয় | মনে রাখার কথা |
|---|---|
| Filter | Exact match, case-sensitive |
| Search | Partial match, `$regex` + `$options: "i"`, একাধিক field-এ `$or` |
| `[field]` | Computed property — variable-এর value key হয় |
| `excludeField` | DB-র field না এমন query (searchTerm, sort, fields, page, limit) filter থেকে বাদ |
| Sort | `field` = ascending, `-field` = descending, default `-createdAt` |
| Fields | Comma → space (`split(",").join(" ")`), `-field` দিলে বাদ |
| Pagination | `skip = (page - 1) * limit`, skip-এর সাথে limit-ও লাগে |
| Meta | `totalPage = Math.ceil(total / limit)`, total অবশ্যই filter সহ |
| Lazy query | `await` ছাড়া query শুধু plan; শেষে একবার `await` |
| QueryBuilder | প্রতিটা method `return this` → chaining; `{ ...this.query }` copy |
| `Promise.all` | Parallel চলে, result-এর order = input-এর order |