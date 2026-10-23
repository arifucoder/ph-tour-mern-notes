# Frontend Notes — ২১: Public Tours Page — Search, Filter, Sort আর Pagination

এতদিন সব কাজ ছিল admin dashboard-এ। এবার প্রথম **public** page: সাধারণ visitor যেখানে tour খুঁজবে। Admin-এর Tours table (২০ নম্বর নোট) আর এই page একই API (`getTours`) ব্যবহার করে, কিন্তু উদ্দেশ্য আলাদা: admin সাজায় আর মোছে, visitor **খোঁজে আর বাছে**। তাই এখানে নতুন জিনিস: search, filter, sort, আর সুন্দর card।

এই নোটে teacher-এর "Type-Safe RTK Query GET Requests" অংশের ব্যাখ্যাও আছে (ধাপ ৪), কারণ সেটা ঠিক এই page-এর API নিয়েই।

প্রতিটা ধাপে 🎯 লক্ষ্য, 🛠️ কাজ, 💡 কেন, ✅ যাচাই। শেষে সমস্যা, উন্নতি আর চূড়ান্ত কোড।

> নোট: আপনার project-এর বর্তমান কোড পড়ে লেখা (React Router 8, RTK 2, shadcn `radix-nova`)। পুরনো ধরনের কোড দিয়ে কিছু বদলানো হয়নি।

---

## ধাপের মানচিত্র

| ধাপ | বিষয় |
|---|---|
| ১ | Route: Home (index) আর Tours |
| ২ | Static design-এর কাঠামো |
| ৩ | Backend কোন query param বোঝে |
| ৪ | TypeScript: `builder.query<Result, Arg>`, teacher-এর `unknown` বনাম আমাদের type |
| ৫–৭ | State: search, filter, sort, page; শুধু ভরা param পাঠানো; page 1-এ ফেরা |
| ৮–৯ | Filter-এর option আর গণনা; sidebar + mobile sheet-এ একই `Filters` |
| ১০ | Tour card |
| ১১ | Pagination-এর নম্বর (`1 … 4 5 6 … 10`) |
| ১২ | চারটা অবস্থা |

---

## ধাপ ১ — Route: Home (index) আর Tours

🎯 **লক্ষ্য:** `/` আর `/tours` দুটোই Navbar/Footer সহ খোলা।

🛠️ **কাজ:**

```tsx
// src/routes/index.tsx
{
  Component: App,
  path: "/",
  children: [
    { index: true, Component: Home },
    { Component: About, path: "about" },
    { Component: Tours, path: "tours" },
    { Component: TourDetails, path: "tours/:slug" },   // ২২ নম্বর নোট
  ],
},
```

💡 **`index: true` কেন:** এতদিন `/` খুললে `App` (Navbar + Footer) দেখাত, কিন্তু মাঝখান ফাঁকা, কারণ `<Outlet />`-এ বসানোর মতো কোনো child ছিল না। ১৪ নম্বর নোটের `/user`-এর মতো একই সমস্যা। **Index route** মানে "parent-এর ঠিক নিজের URL-এ (`/`) এলে এটা দেখাও"। এখন `/` = Home।

💡 **`"tours"`, `"/tours"` না:** `App`-এর child, তাই relative path (১৪ নম্বর নোটের ধাপ ৪)। আসল URL `/` + `tours` = `/tours`।

> নোট: Admin-এর `pages/Admin/Tours.tsx` আর এই `pages/Tours.tsx` দুটোর component-এর নামই `Tours`। আলাদা folder, আলাদা import path, তাই কোনো সংঘর্ষ নেই। শুধু import-এর সময় path খেয়াল রাখবেন।

> নোট (১৫ নম্বর নোট): Home প্রথম page, তাই সাধারণ import ঠিক আছে; কিন্তু Tours, TourDetails, PaymentResult public হলেও প্রথম page না, এগুলো `lazy` করলে Home আরও দ্রুত খোলে।

✅ **যাচাই:** `/`-এ "This is Home component", Navbar-এর Tours চাপলে `/tours`।

---

## ধাপ ২ — Static design-এর কাঠামো

🎯 **লক্ষ্য:** Page-এর বড় অংশগুলো চেনা।

```
┌─────────────────────────────────────────────┐
│ Home / Tours                                │  ← breadcrumb
│ Find your tour                              │
│ [🔍 Search by tour name or location…][Search]│  ← hero + search (bg-muted)
├──────────────┬──────────────────────────────┤
│ Filters      │ 12 tours found     [Sort ▾]  │  ← toolbar
│ ○ Division   │ ┌────┐ ┌────┐ ┌────┐          │
│ ○ Tour type  │ │card│ │card│ │card│          │  ← card grid
│              │ └────┘ └────┘ └────┘          │
│ (desktop-এ)  │   ‹  1  2  3  …  5  ›          │  ← pagination
└──────────────┴──────────────────────────────┘
```

💡 **কিছু অংশ:**
- **`<form role="search">`:** screen reader-কে বলে এটা খোঁজার জায়গা।
- **`<label className="sr-only">`:** চোখে দেখা যায় না (placeholder-ই যথেষ্ট দেখতে), কিন্তু screen reader পড়ে। Placeholder label-এর বদলি না, কারণ লেখা শুরু করলেই মুছে যায়।
- **Filters:** desktop-এ বাম পাশে (`hidden lg:block`), মোবাইলে "Filters" button চাপলে পাশ থেকে আসা Sheet-এ (ধাপ ৯)।
- **`<select>` (সাধারণ HTML) sort-এর জন্য:** মাত্র ৩টা option, খোঁজার দরকার নেই, তাই browser-এর নিজেরটা যথেষ্ট আর মোবাইলে phone-এর নিজের বাছাই-চাকা খোলে।

---

## ধাপ ৩ — Backend কোন query param বোঝে

🎯 **লক্ষ্য:** Search, filter, sort, page কীভাবে backend-এ যায়।

Backend-এর `getAllTours`-এ `QueryBuilder` (২০ নম্বর নোটের ধাপ ১৪) URL-এর query string পড়ে। আমরা `TToursParams` type-এ লিখে রাখি কী কী পাঠানো যায়:

```ts
// src/redux/features/Tour/tour.api.ts
type TToursParams = {
  page?: number;
  limit?: number;
  searchTerm?: string; // title, description, location-এ খোঁজে
  sort?: string;       // "-createdAt" (default) | "costFrom" | "-costFrom"
  division?: string;   // division-এর _id
  tourType?: string;   // tourType-এর _id
  fields?: string;     // শুধু এই field-গুলো আনো, যেমন "division,tourType"
};
```

| Param | উদাহরণ URL | QueryBuilder-এর কাজ |
|---|---|---|
| `searchTerm` | `?searchTerm=sajek` | `.search(...)`: কয়েকটা field-এ লেখা খোঁজে |
| `division`, `tourType` | `?division=6650f1…` | `.filter()`: হুবহু মেলায় |
| `sort` | `?sort=-costFrom` | `.sort()`: `-` মানে উল্টো (বড় থেকে ছোট) |
| `fields` | `?fields=division,tourType` | `.fields()`: শুধু এই field আনে, response ছোট |
| `page`, `limit` | `?page=2&limit=12` | `.paginate()` |

💡 **`sort`-এর `-`:** `costFrom` = দাম কম থেকে বেশি, `-costFrom` = বেশি থেকে কম, `-createdAt` = নতুনটা আগে। এটা Mongoose-এর নিয়ম, backend সরাসরি ব্যবহার করে।

💡 **Param-এর নাম backend-এর সাথে হুবহু:** frontend `search` পাঠালে আর backend `searchTerm` খুঁজলে খোঁজা কাজই করবে না, আর কোনো error-ও দেবে না (backend অজানা param উপেক্ষা করে)। তাই type-এ comment সহ লিখে রাখা।

---

## ধাপ ৪ — TypeScript: `builder.query<Result, Arg>` (teacher-এর অংশ)

🎯 **লক্ষ্য:** Teacher-এর `<ITourPackage[], unknown>` আর আমাদের `<{ data; meta }, TToursParams | void>`-এর পার্থক্য পরিষ্কার বোঝা।

### 💡 দুটো ফাঁকা জায়গা

```ts
builder.query< ResultType , QueryArg >(...)
//              ↑             ↑
//   hook-এর data কেমন   hook-এ কী পাঠানো যাবে
```

প্রথমটা: component-এ `const { data } = useXQuery(...)` করলে `data`-র type। দ্বিতীয়টা: `useXQuery(এখানে কী দেওয়া যাবে)`।

### 💡 দুটো কোড পাশাপাশি

```ts
// Teacher
getAllTours: builder.query<ITourPackage[], unknown>({
  query: () => ({ url: "/tour", method: "GET" }),
  providesTags: ["TOUR"],
  transformResponse: (response: IResponse<ITourPackage[]>) => response.data,
}),

// আমরা
getTours: builder.query<{ data: ITour[]; meta: IMeta }, TToursParams | void>({
  query: (params) => ({ url: "/tour", method: "GET", params: params ?? undefined }),
  providesTags: ["TOUR"],
  transformResponse: (response: IResponse<ITour[]> & { meta: IMeta }) => ({
    data: response.data,
    meta: response.meta,
  }),
}),
```

| | Teacher | আমরা |
|---|---|---|
| Tour-এর interface-এর নাম | `ITourPackage` | `ITour` |
| ResultType | `ITourPackage[]` (শুধু tour-এর array) | `{ data: ITour[]; meta: IMeta }` (tour + pagination-এর তথ্য) |
| QueryArg | `unknown` | `TToursParams \| void` |
| Hook-এ param পাঠানো | `query: () =>` কিছুই নেয় না | `query: (params) =>` নিয়ে URL-এ বসায় |
| `transformResponse` | শুধু `response.data` (meta হারায়) | `data` + `meta` দুটোই |

**`ITourPackage` কী:** শুধু নামের পার্থক্য। Teacher tour-এর interface-কে "tour package" বলেছেন, আমরা বলেছি `ITour`। ভেতরে একই জিনিস (title, images, costFrom …)। নিজের project-এ একটা নাম ঠিক করে সব জায়গায় সেটাই।

### 💡 `unknown`, `any`, `void` — তিনটার পার্থক্য

এটাই আসল বিভ্রান্তির জায়গা:

| Type | মানে | Hook-এ যা দেওয়া যায় | TypeScript কতটা পাহারা দেয় |
|---|---|---|---|
| `void` | "কিছুই পাঠাবে না" | `useX()` (বা `undefined`) | ভুল করে কিছু পাঠালে error ✅ |
| `unknown` | "যেকোনো কিছু হতে পারে, কিন্তু আমি জানি না কী" | **যেকোনো কিছু**: `useX()`, `useX(5)`, `useX({ abc: 1 })` | পাঠানোর সময় কিছুই ধরে না; আর `query`-র ভেতরে ব্যবহার করতে গেলে আগে type যাচাই করতে বাধ্য করে |
| `any` | "type দেখা বন্ধ" | যেকোনো কিছু | কিছুই না, কোথাও না ❌ |
| `TToursParams \| void` | "এই আকারের object, অথবা কিছুই না" | `useX()` বা `useX({ page: 2 })` | `useX({ pagee: 2 })` লিখলে error ✅ |

Teacher-এর `query: () => ...` কোনো argument ব্যবহারই করে না, তাই সেখানে `unknown` দিলে "যা খুশি দাও, আমি দেখব না"। কাজ চলে, কিন্তু কেউ `useGetAllToursQuery({ page: 2 })` লিখলে TypeScript কিছু বলে না, আর page-ও যায় না (query তো param নেয়ই না)। সেখানে **`void`** বেশি সৎ: "এই endpoint কিছু নেয় না"।

আমাদের endpoint param **নেয়**, তাই নির্দিষ্ট type: `TToursParams`। আর `| void` যোগ করায় কিছু না দিয়েও ডাকা যায় (admin-এর কোথাও সব tour লাগলে `useGetToursQuery()`)।

**নিয়ম:** QueryArg-এ
- কিছু না নিলে → **`void`**
- নিলে → **নির্দিষ্ট type** (`string`, `TToursParams` …)
- `unknown`/`any` → এড়িয়ে চলুন; শুধু তখন, যখন সত্যিই জানেন না কী আসবে আর ভেতরে যাচাই করবেন।

### 💡 `IResponse<ITour[]> & { meta: IMeta }`

Backend-এর response:

```json
{ "statusCode": 200, "success": true, "message": "...", "data": [ ... ], "meta": { "page": 1, "limit": 12, "total": 30, "totalPage": 3 } }
```

আমাদের `IResponse<T>`-এ (১৩ নম্বর নোট) `meta` নেই। `&` (intersection) মানে "দুটো type জুড়ে দাও": `IResponse`-এর সব field **আর** `meta`। এটা শুধু `transformResponse`-এর **ভেতরে** আসা আকার; বাইরে component পায় ResultType-এর আকার (`{ data, meta }`)।

```
Backend-এর response   ──transformResponse──→   component-এর data
IResponse<ITour[]> & { meta }                  { data: ITour[]; meta: IMeta }
```

### 💡 `params: params ?? undefined`

`params` হলো `TToursParams | void`। `void` মানে আসলে `undefined`, কিন্তু TypeScript `void`-কে axios-এর `params`-এ সরাসরি দিতে দেয় না। `?? undefined` দিয়ে পরিষ্কার `undefined` বানানো হয়। Axios `params` object পেলে `?page=2&limit=12` বানায়, `undefined` পেলে কিছুই যোগ করে না।

---

## ধাপ ৫ — State: খোঁজার লেখা দুটো state কেন

🎯 **লক্ষ্য:** User যা টাইপ করছে আর যা দিয়ে আসলে খোঁজা হচ্ছে, আলাদা রাখা।

🛠️ **কাজ:**

```tsx
const [searchInput, setSearchInput] = useState("");   // box-এ এখন যা লেখা
const [searchTerm, setSearchTerm] = useState("");     // Search চাপার পরে যা দিয়ে খোঁজা হচ্ছে
const [division, setDivision] = useState("");
const [tourType, setTourType] = useState("");
const [sort, setSort] = useState("-createdAt");
const [page, setPage] = useState(1);
```

💡 **`searchInput` আর `searchTerm` আলাদা কেন:** একটাই রাখলে প্রতিটা অক্ষর টাইপে request যেত: `s`, `sa`, `saj`, `saje`, `sajek`, ৫টা request, প্রথম ৪টা অকারণ। দুটো রাখলে:
- টাইপের সময় শুধু `searchInput` বদলায় (box-এ লেখা দেখায়), কোনো request না।
- Search চাপলে (বা Enter) `searchTerm = searchInput`, তখন একবার request।

এজন্যই `<form onSubmit={handleSearch}>`: Enter চাপলেও form submit হয়, আলাদা কোড লাগে না।

```tsx
const handleSearch = (e: FormEvent<HTMLFormElement>) => {
  e.preventDefault();               // page reload বন্ধ
  setSearchTerm(searchInput.trim());
  setPage(1);
};
```

> নোট: টাইপ করতে করতেই ফল দেখাতে চাইলে (live search), প্রতি অক্ষরে না, টাইপ থামার ৪০০ms পরে খোঁজা হয়, একে বলে **debounce**। এখনকার "Search চাপলে খোঁজা" সহজ আর পরিষ্কার; পরে চাইলে debounce যোগ করা যায়।

---

## ধাপ ৬ — শুধু ভরা param পাঠানো

🎯 **লক্ষ্য:** খালি filter URL-এ না পাঠানো।

🛠️ **কাজ:**

```tsx
const params = {
  page,
  limit: LIMIT,
  sort,
  ...(searchTerm && { searchTerm }),
  ...(division && { division }),
  ...(tourType && { tourType }),
};

const { data, isLoading, isError, isFetching, refetch } = useGetToursQuery(params);
```

💡 **`...(searchTerm && { searchTerm })` কীভাবে কাজ করে:**

| `searchTerm` | `searchTerm && { searchTerm }` | spread-এর পরে |
|---|---|---|
| `"sajek"` | `{ searchTerm: "sajek" }` | object-এ `searchTerm: "sajek"` যোগ হয় |
| `""` | `""` (খালি string, falsy) | object-এ spread করলে **কিছুই যোগ হয় না** |

না করলে URL হতো `?searchTerm=&division=&tourType=`, আর backend হয়তো খালি `division` দিয়ে ছেঁকে **কিছুই** পেত না।

(১৩ নম্বর নোটের register-এ `...(data.phone ? { phone } : {})` লেখা হয়েছিল; `&&` দিয়ে ছোট লেখা, একই কাজ। TypeScript object-এ `""` spread করা মেনে নেয়।)

### 💡 প্রতি render-এ নতুন `params` object, তবু বারবার request কেন হয় না

`params` প্রতিটা render-এ নতুন object। সাধারণ `useEffect`-এ dependency-তে দিলে প্রতি render-এ আবার চলত। কিন্তু **RTK Query argument-কে object হিসেবে না, তার ভেতরের মান দেখে** একটা key বানায় (serialize), মোটামুটি `getTours({"limit":12,"page":1,"sort":"-createdAt"})`। মান না বদলালে key একই, তাই নতুন request যায় না। মান বদলালে (page ২) নতুন key, নতুন request, আর আগের পাতা cache-এ থেকে যায়।

---

## ধাপ ৭ — যেকোনো filter বদলালে page 1-এ ফেরা

🎯 **লক্ষ্য:** খালি পাতায় আটকে না যাওয়া।

🛠️ **কাজ:**

```tsx
const handleDivision = (value: string) => {
  setDivision(value);
  setPage(1);
};
// handleTourType, handleSort, handleReset, handleSearch সবগুলোতেই setPage(1)
```

💡 **কেন:** User পাতা ৩-এ আছে (মোট ৩০টা tour)। এখন "Sylhet" বাছল, Sylhet-এ মোট ৫টা tour, অর্থাৎ ১টাই পাতা। Page ৩-ই থাকলে backend খালি পাতা দেবে, আর দেখাবে "No tours found", অথচ ৫টা tour আছে! তাই নতুন filter মানে নতুন ফলাফল, শুরু থেকে।

```tsx
const goToPage = (p: number) => {
  setPage(p);
  window.scrollTo({ top: 0, behavior: "smooth" });
};
```

পাতা বদলালে উপরে scroll: নিচের pagination চেপে নতুন পাতায় গেলে user নিচেই থেকে যেত, নতুন card-গুলো উপরে।

---

## ধাপ ৮ — Filter-এর option আর গণনা

🎯 **লক্ষ্য:** Sidebar-এ শুধু যেসব division/type-এ tour আছে, পাশে কয়টা tour।

🛠️ **কাজ:**

```tsx
// সব tour, শুধু division + tourType → sidebar-এর option আর গণনার জন্য
const { data: optionData } = useGetToursQuery({ limit: 1000, fields: "division,tourType" });

const { divisions, tourTypes, totalTours } = useMemo(() => {
  const all = optionData?.data ?? [];
  return {
    divisions: buildOptions(all, "division"),
    tourTypes: buildOptions(all, "tourType"),
    totalTours: all.length,
  };
}, [optionData]);
```

```ts
type TFilterOption = { _id: string; name: string; count: number };

// tour-গুলো থেকে অনন্য { _id, name, count }, নাম অনুযায়ী সাজানো
const buildOptions = (tours: ITour[], key: "division" | "tourType"): TFilterOption[] => {
  const map = new Map<string, TFilterOption>();
  tours.forEach((tour) => {
    const item = tour[key];
    if (!item?._id) return;
    const existing = map.get(item._id);
    if (existing) existing.count += 1;
    else map.set(item._id, { _id: item._id, name: item.name, count: 1 });
  });
  return [...map.values()].sort((a, b) => a.name.localeCompare(b.name));
};
```

### 💡 `Map` দিয়ে গোনা, ধাপে ধাপে

ধরুন ৪টা tour: Sylhet, Chattogram, Sylhet, Dhaka।

```
tour ১ → Sylhet      map-এ নেই → { Sylhet: 1 }
tour ২ → Chattogram  নেই       → { Sylhet: 1, Chattogram: 1 }
tour ৩ → Sylhet      আছে!      → { Sylhet: 2, Chattogram: 1 }
tour ৪ → Dhaka       নেই       → { Sylhet: 2, Chattogram: 1, Dhaka: 1 }

[...map.values()] → array, তারপর নাম অনুযায়ী: Chattogram 1, Dhaka 1, Sylhet 2
```

- **`Map`:** key (এখানে `_id`) দিয়ে জিনিস রাখার বাক্স। "এই id আগে দেখেছি কিনা" খুব দ্রুত বলে দেয়, array-তে বারবার `find` করার চেয়ে।
- **`key: "division" | "tourType"`:** একই function দুই কাজে; `tour[key]` দিয়ে কোন field দেখবে।
- **`if (!item?._id) return;`:** division মুছে ফেলা tour-এ `division: null` (২০ নম্বর নোট), সেটা বাদ।
- **`localeCompare`:** লেখা বর্ণানুক্রমে সাজানোর সঠিক উপায় (বড়-ছোট হাত, ভাষার নিয়ম মেনে)।
- **`useMemo`:** `optionData` না বদলালে প্রতি render-এ আবার গোনে না (২০ নম্বর নোটের ব্যাখ্যা)। এখানে সত্যিই কাজের, কারণ ১০০০টা পর্যন্ত tour ঘুরে গোনা হতে পারে।

💡 **`fields: "division,tourType"`:** সব tour আনা হচ্ছে, কিন্তু শুধু দুটো field; ছবি, description, plan কিছুই না। Response অনেক ছোট।

> ⚠️ সতর্কতা (উন্নতি ২-এ বিস্তারিত): এই উপায় এখন চলবে, কিন্তু tour হাজার ছাড়ালে `limit: 1000` সব আনবে না, আর গণনা ভুল হবে। বড় হলে backend থেকে গণনাসহ আলাদা API ভালো।

---

## ধাপ ৯ — একই `Filters` দুই জায়গায়: sidebar আর mobile sheet

🎯 **লক্ষ্য:** Filter-এর কোড একবার লিখে desktop আর mobile দুই জায়গায় ব্যবহার।

🛠️ **কাজ:**

```tsx
const filterProps = {
  divisions, tourTypes, totalTours, division, tourType,
  onDivisionChange: handleDivision,
  onTourTypeChange: handleTourType,
  onReset: handleReset,
};

{/* Desktop */}
<aside aria-label="Filters" className="hidden w-70 shrink-0 rounded-xl bg-card p-6 shadow-card lg:block">
  <Filters {...filterProps} idPrefix="desk" />
</aside>

{/* Mobile */}
<Sheet>
  <SheetTrigger asChild>
    <Button variant="outline" className="... lg:hidden">
      <SlidersHorizontal /> Filters{activeFilterCount > 0 && ` · ${activeFilterCount}`}
    </Button>
  </SheetTrigger>
  <SheetContent side="left" className="w-80 overflow-y-auto p-6">
    <SheetHeader className="sr-only"><SheetTitle>Filters</SheetTitle></SheetHeader>
    <Filters {...filterProps} idPrefix="mob" />
  </SheetContent>
</Sheet>
```

💡 **`{...filterProps}`:** একই ৮টা prop দুইবার না লিখে একটা object-এ রেখে ছড়িয়ে দেওয়া।

💡 **`idPrefix` কেন:** `Filters`-এর ভেতরে প্রতিটা radio-র `id` আছে (label-এর `htmlFor`-এর জন্য)। একই page-এ দুটো `Filters` থাকলে দুটো একই `id` হতো (`div-sylhet`)। HTML-এ একই `id` দুবার থাকা নিষেধ; তখন mobile-এর label চাপলে desktop-এর radio বাছাই হতে পারে। `desk-div-…` আর `mob-div-…` আলাদা রাখে। একইভাবে radio-র `name`-ও আলাদা, নাহলে দুই জায়গার radio এক দল ধরা হতো।

💡 **`SheetHeader className="sr-only"`:** Sheet (dialog)-এর একটা title থাকা বাধ্যতামূলক (৮ নম্বর নোট), কিন্তু design-এ উপরে আলাদা title লাগছে না (`Filters`-এর ভেতরেই "Filters" লেখা)। তাই title আছে, শুধু চোখে লুকানো।

💡 **Division-এ radio, tour type-এ pill button:**
- Raw design-এ division-এ checkbox ছিল (একাধিক বাছা)। কিন্তু backend-এর `division` param একটাই id নেয়, তাই **radio** (একটা), সাথে "All divisions" (মান `""`)।
- Tour type-এ `aria-pressed` সহ button: চাপা অবস্থা screen reader জানতে পারে। এটাও একটা বাছাই, সাথে "All"।

💡 **`activeFilterCount`:** `[division, tourType].filter(Boolean).length`, কয়টা filter চালু। Mobile button-এ "Filters · 2"; বন্ধ sheet-এর বাইরে থেকেই বোঝা যায় কিছু ছাঁকা আছে।

---

## ধাপ ১০ — Tour card

🎯 **লক্ষ্য:** প্রতিটা tour-এর সুন্দর card, পুরোটাই click করা যায়।

```tsx
<Link
  key={tour._id}
  to={`/tours/${tour.slug}`}
  className="group flex flex-col overflow-hidden rounded-xl bg-card ... hover:shadow-lg focus-visible:ring-3 focus-visible:ring-ring/50"
>
  <div className="relative h-50 ... bg-accent">
    {tour.images?.[0] ? (
      <img src={tour.images[0]} alt={tour.title} loading="lazy"
        className="size-full object-cover transition-transform duration-300 group-hover:scale-105" />
    ) : (
      <ImageIcon className="size-7" />
    )}
    {tour.tourType && <span className="absolute top-3 left-3 ...">{tour.tourType.name}</span>}
  </div>
  {/* location, title, meta, price */}
</Link>
```

💡 **কিছু অংশ:**
- **পুরো card একটা `<Link>`:** card-এর যেকোনো জায়গায় click করলে details-এ যায়; ভেতরে আলাদা "View" button লাগে না। আর এটা `<Link>`, `<a href>` না (আমাদের নিয়ম)।
- **`to={`/tours/${tour.slug}`}`:** URL-এ id না, slug (`/tours/sajek-valley-cloud-escape`): পড়তে সুন্দর, search engine-এর জন্যও ভালো।
- **`group` + `group-hover:scale-105`:** card-এর (parent) ওপর mouse এলে ভেতরের ছবি একটু বড় হয়, আর শিরোনাম `group-hover:text-primary`। Tailwind-এর `group`: parent-এর অবস্থা দেখে child-কে সাজানো।
- **`focus-visible:ring-…`:** keyboard (Tab) দিয়ে card-এ এলে চারপাশে দাগ, যাতে বোঝা যায় কোথায় আছি।
- **`loading="lazy"`:** নিচের card-এর ছবি scroll করে কাছে এলে আসে (১৫ নম্বর নোট)।

### 💡 দিন আর তারিখের লেখা

```ts
const getDays = (start: string, end: string) =>
  Math.round((new Date(end).getTime() - new Date(start).getTime()) / (1000 * 60 * 60 * 24)) + 1;
```

- `getTime()`: তারিখকে millisecond-এ (১৯৭০ থেকে)। দুটোর পার্থক্য ÷ একদিনের millisecond (১০০০ × ৬০ × ৬০ × ২৪) = কত দিনের ফারাক।
- **`+ 1`:** ১২ থেকে ১৪ তারিখ: ফারাক ২ দিন, কিন্তু tour **৩ দিনের** (১২, ১৩, ১৪), দুই দিকই গোনা।
- **`Math.round`:** daylight saving-এর মতো কারণে ফারাক ঠিক পূর্ণ সংখ্যা না হলে গোল করা।

`getTourMeta` এগুলো জোড়া লাগায়: `"3 days · 12 Nov – 14 Nov · Max 20 guests"`; যেটা নেই সেটা বাদ (তারিখ না থাকলে দিনও না)।

---

## ধাপ ১১ — Pagination-এর নম্বর: `1 … 4 5 6 … 10`

🎯 **লক্ষ্য:** ১০০ পাতা হলেও ১০০টা button না দেখিয়ে কাজের কয়েকটা।

```ts
const getPageNumbers = (current: number, total: number): (number | "…")[] => {
  if (total <= 5) return Array.from({ length: total }, (_, i) => i + 1);  // অল্প হলে সব
  const pages: (number | "…")[] = [1];                                    // প্রথম পাতা সবসময়
  if (current > 3) pages.push("…");                                       // মাঝে ফাঁক থাকলে …
  for (let p = Math.max(2, current - 1); p <= Math.min(total - 1, current + 1); p++) pages.push(p); // বর্তমান ±১
  if (current < total - 2) pages.push("…");
  pages.push(total);                                                      // শেষ পাতা সবসময়
  return pages;
};
```

**উদাহরণ (মোট ১০ পাতা):**

| বর্তমান | ফল |
|---|---|
| ১ | `1 2 … 10` |
| ৫ | `1 … 4 5 6 … 10` |
| ১০ | `1 … 9 10` |
| (মোট ৪) | `1 2 3 4` |

- **`(number | "…")[]`:** array-তে সংখ্যা **অথবা** `"…"` থাকতে পারে। Render-এর সময় `p === "…"` হলে শুধু লেখা, নাহলে button।
- **`Array.from({ length: total }, (_, i) => i + 1)`:** `[1, 2, …, total]` বানানোর ছোট উপায়।
- **`aria-current="page"`:** screen reader-কে বলে কোন পাতায় আছি।
- **`…`-এর key `dots-${i}`:** দুটো `…` থাকতে পারে, তাই index দিয়ে আলাদা।

---

## ধাপ ১২ — চারটা অবস্থা

১৭ নম্বর নোট থেকে একই নিয়ম, এখানে card-এর আকারে:

| অবস্থা | কী দেখায় |
|---|---|
| `isLoading` | ৬টা card-আকারের skeleton |
| `isError` | "Couldn't load tours" + **Try again** (`refetch`) |
| খালি | "No tours found" + **Clear search & filters** (`handleReset`) |
| Data | card grid; নতুন filter/পাতা আসার সময় (`isFetching`) ফিকে |

💡 **খালি অবস্থায় "Clear" button:** খোঁজার page-এ খালি ফল প্রায়ই মানে "filter বেশি কড়া"। এক click-এ সব মুছে দেওয়ার উপায় রাখলে user আটকে থাকে না।

✅ **যাচাই (পুরো page):**
- "sajek" লিখে Search → শুধু মেলা tour, উপরে `found for "sajek"`
- Division বাছলে ফল ছাঁকা হয় আর পাতা ১-এ ফেরে
- Sort বদলালে দামের ক্রম বদলায়
- Mobile width-এ Filters button, চাপলে বাম থেকে sheet, filter চালু থাকলে "Filters · 1"
- Network-এ খালি param (`searchTerm=`) যায় না

---

# কী বাকি: সমস্যা চেনা

| # | সমস্যা |
|---|---|
| ১ | **Filter, search, page সব `useState`-এ, URL-এ না।** ফলে: (ক) page reload দিলে সব filter মুছে যায়; (খ) **Sylhet-এর filter দিয়ে একটা tour খুলে Back চাপলে filter-হীন পাতা ১-এ ফিরে আসে**, আবার শুরু থেকে খুঁজতে হয়; (গ) কাউকে "Sylhet-এর long tour"-এর link পাঠানো যায় না। |
| ২ | Filter-এর option আর গণনার জন্য `limit: 1000` দিয়ে সব tour আনা: tour বাড়লে ভারী, আর ১০০০-এর বেশি হলে গণনা ভুল। |
| ৩ | Class জোড়া লাগাতে `[...].join(" ")`, project-এর `cn()` না (১৯ নম্বর নোট)। |
| ৪ | Error-এর ঘরে `bg-destructive/10` → `bg-destructive-soft`; `<h1>`, `<h2>`, card-এর শিরোনামে `text-foreground` নেই। |
| ৫ | Card-এর ছবিতে `width`/`height` নেই (layout shift, ১৫ নম্বর নোট)। |

---

# উন্নতি

## উন্নতি ১ — Filter-গুলো URL-এ: `useSearchParams` (সবচেয়ে জরুরি)

**ধারণা:** filter-এর মান `useState`-এ না রেখে **URL-এর query string**-এ রাখা:

```
/tours?division=6650f1…&tourType=6650e9…&sort=costFrom&page=2
```

তাহলে reload, Back, link share সব জায়গায় filter থেকে যায়, কারণ URL-ই হলো state। React Router-এর `useSearchParams` এটা সহজ করে (২৩ নম্বর নোটের PaymentResult-এও এটা ব্যবহার হয়েছে, পড়ার জন্য)।

🛠️ **কাজ:** ধাপ ৫-এর ছয়টা `useState`-এর জায়গায়:

```tsx
import { useSearchParams } from "react-router";

const DEFAULT_SORT = "-createdAt";

const [searchParams, setSearchParams] = useSearchParams();

// URL থেকে পড়া (না থাকলে default)
const searchTerm = searchParams.get("searchTerm") ?? "";
const division = searchParams.get("division") ?? "";
const tourType = searchParams.get("tourType") ?? "";
const sort = searchParams.get("sort") ?? DEFAULT_SORT;
const page = Math.max(1, Number(searchParams.get("page")) || 1);

// শুধু box-এর লেখা এখনো সাধারণ state (টাইপের সময় URL বদলাবে না)
const [searchInput, setSearchInput] = useState(searchTerm);

// এক বা একাধিক মান বদলানো; খালি বা default মান URL থেকে বাদ
// page নিজে না বদলালে, যেকোনো filter বদলালে page মুছে যায় (= 1)
const updateParams = (changes: Record<string, string | number>) => {
  setSearchParams((prev) => {
    const next = new URLSearchParams(prev);
    if (!("page" in changes)) next.delete("page");
    Object.entries(changes).forEach(([key, value]) => {
      const isDefault = value === "" || (key === "page" && value === 1) || (key === "sort" && value === DEFAULT_SORT);
      if (isDefault) next.delete(key);
      else next.set(key, String(value));
    });
    return next;
  });
};

const handleSearch = (e: FormEvent<HTMLFormElement>) => {
  e.preventDefault();
  updateParams({ searchTerm: searchInput.trim() });
};
const handleDivision = (value: string) => updateParams({ division: value });
const handleTourType = (value: string) => updateParams({ tourType: value });
const handleSort = (value: string) => updateParams({ sort: value });
const handleReset = () => {
  setSearchParams({});
  setSearchInput("");
};
const goToPage = (p: number) => {
  updateParams({ page: p });
  window.scrollTo({ top: 0, behavior: "smooth" });
};
```

বাকি page (`params` object, `useGetToursQuery(params)`, JSX) **একদম আগের মতো**, কারণ নামগুলো (`searchTerm`, `division`, `page` …) একই রাখা হয়েছে।

💡 **লাইনগুলোর মানে:**
- **`searchParams.get("division")`:** URL-এ না থাকলে `null`, তাই `?? ""`।
- **`Number(...) || 1`:** URL-এ সবকিছু লেখা; `"2"` → `2`। কেউ হাতে `?page=abc` লিখলে `NaN`, তখন `|| 1`। `Math.max(1, …)` দিয়ে `?page=-5`-ও আটকানো।
- **`setSearchParams((prev) => …)`:** আগের URL-এর param থেকে শুরু করে শুধু যা বদলাচ্ছে সেটা বদলানো, বাকিগুলো (অন্য filter) থেকে যায়।
- **`new URLSearchParams(prev)`:** আগেরটার একটা কপি; সরাসরি বদলানো যায় না (React-এর নিয়ম, ২০ নম্বর নোটের `[...value, text]`-এর মতো)।
- **Default মান URL থেকে বাদ:** `/tours?page=1&sort=-createdAt` না, শুধু `/tours`। URL পরিষ্কার থাকে।
- **`setPage(1)` আর আলাদা লিখতে হয় না:** `updateParams` নিজেই page মুছে দেয় (ধাপ ৭-এর নিয়ম এক জায়গায়)।

**এখন Back বোতাম:** প্রতিটা filter বদলানো browser-এর history-তে একটা ধাপ। Tour খুলে Back চাপলে ঠিক আগের filter আর পাতায় ফেরে। (Filter বদলের ধাপগুলো history-তে না চাইলে `setSearchParams(fn, { replace: true })`।)

✅ **যাচাই:** Sylhet বেছে পাতা ২-এ যান → URL-এ `?division=…&page=2` → একটা tour খুলুন → Back → একই filter, একই পাতা; reload দিলেও তাই; URL কপি করে নতুন tab-এ খুললেও তাই।

## উন্নতি ২ — Filter-এর option আলাদা API থেকে (পরে, বড় হলে)

এখনকার উপায়ের বদলে:
- **নামের তালিকা:** ইতিমধ্যে আছে, `useGetDivisionsQuery()` আর `useGetTourTypesQuery()` (১৭, ১৯ নম্বর নোট)। RTK Query-র cache-এ admin-এ আনা থাকলে আবার request-ও লাগে না।
- **গণনা:** backend-এ একটা ছোট endpoint (MongoDB `aggregate` দিয়ে `$group` by division), যেটা শুধু `[{ _id, name, count }]` দেয়, হাজার tour পাঠায় না।

এখন tour কম, তাই বর্তমান উপায় ঠিক আছে; শুধু মনে রাখুন এটা বড় হলে বদলাতে হবে।

## উন্নতি ৩ — ছোটখাটো

- `[...].join(" ")` → `cn(...)` (`Filters`-এর pill button আর card grid)।
- Error-এর ঘরে `bg-destructive-soft`; শিরোনামে `text-foreground`।
- Card-এর ছবিতে `width={400} height={200}`।
- `Tours`, `TourDetails`, `PaymentResult` route-এ `lazy` (১৫ নম্বর নোট)।

---

# চূড়ান্ত কোড

### `src/pages/Tours.tsx` — আপনার বর্তমান কোড

> এর ওপর উন্নতি ১ বসাতে: শুধু `/* ───────── page ───────── */`-এর নিচে ছয়টা `useState` আর `handle…`/`goToPage` function-গুলোর জায়গায় উন্নতি ১-এর কোড বসান, আর import-এ `useSearchParams` যোগ করুন। বাকি সব হুবহু।

```tsx
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Sheet, SheetContent, SheetHeader, SheetTitle, SheetTrigger } from "@/components/ui/sheet";
import { Skeleton } from "@/components/ui/skeleton";
import { useGetToursQuery } from "@/redux/features/Tour/tour.api";
import type { ITour } from "@/types";
import {
	AlertCircle,
	ChevronLeft,
	ChevronRight,
	ImageIcon,
	MapPin,
	RefreshCw,
	Search,
	SearchX,
	SlidersHorizontal,
} from "lucide-react";
import { useMemo, useState, type FormEvent } from "react";
import { Link } from "react-router";

const LIMIT = 12;

const SORT_OPTIONS = [
	{ value: "-createdAt", label: "Newest first" },
	{ value: "costFrom", label: "Price: low to high" },
	{ value: "-costFrom", label: "Price: high to low" },
];

/* ───────── helpers ───────── */

const formatPrice = (amount?: number) => (amount === undefined ? "—" : amount.toLocaleString("en-IN"));

const shortDate = (date: string) => new Date(date).toLocaleDateString("en-GB", { day: "numeric", month: "short" });

const getDays = (start: string, end: string) =>
	Math.round((new Date(end).getTime() - new Date(start).getTime()) / (1000 * 60 * 60 * 24)) + 1;

const getTourMeta = (tour: ITour) => {
	const parts: string[] = [];
	if (tour.startDate && tour.endDate) {
		const days = getDays(tour.startDate, tour.endDate);
		parts.push(`${days} ${days === 1 ? "day" : "days"}`);
		parts.push(
			tour.startDate.slice(0, 10) === tour.endDate.slice(0, 10)
				? shortDate(tour.startDate)
				: `${shortDate(tour.startDate)} – ${shortDate(tour.endDate)}`,
		);
	}
	if (tour.maxGuest) parts.push(`Max ${tour.maxGuest} guests`);
	return parts.join(" · ");
};

// 1 … 4 5 6 … 10
const getPageNumbers = (current: number, total: number): (number | "…")[] => {
	if (total <= 5) return Array.from({ length: total }, (_, i) => i + 1);
	const pages: (number | "…")[] = [1];
	if (current > 3) pages.push("…");
	for (let p = Math.max(2, current - 1); p <= Math.min(total - 1, current + 1); p++) pages.push(p);
	if (current < total - 2) pages.push("…");
	pages.push(total);
	return pages;
};

type TFilterOption = { _id: string; name: string; count: number };

// unique { _id, name, count } list from tours, sorted by name
const buildOptions = (tours: ITour[], key: "division" | "tourType"): TFilterOption[] => {
	const map = new Map<string, TFilterOption>();
	tours.forEach((tour) => {
		const item = tour[key];
		if (!item?._id) return;
		const existing = map.get(item._id);
		if (existing) existing.count += 1;
		else map.set(item._id, { _id: item._id, name: item.name, count: 1 });
	});
	return [...map.values()].sort((a, b) => a.name.localeCompare(b.name));
};

const selectCls =
	"h-11.5 w-full rounded-lg border-[1.5px] border-border bg-background px-3.5 text-sm text-foreground outline-none focus-visible:border-ring focus-visible:ring-3 focus-visible:ring-ring/50";

/* ───────── filters panel (used in sidebar + mobile sheet) ───────── */

type FiltersProps = {
	divisions: TFilterOption[];
	tourTypes: TFilterOption[];
	totalTours: number;
	division: string;
	tourType: string;
	onDivisionChange: (value: string) => void;
	onTourTypeChange: (value: string) => void;
	onReset: () => void;
	idPrefix: string;
};

function Filters({
	divisions,
	tourTypes,
	totalTours,
	division,
	tourType,
	onDivisionChange,
	onTourTypeChange,
	onReset,
	idPrefix,
}: FiltersProps) {
	const hasFilter = Boolean(division || tourType);

	return (
		<div className="flex flex-col gap-6.5">
			<div className="flex items-center justify-between">
				<h2 className="text-lg font-semibold">Filters</h2>
				<Button
					variant="link"
					type="button"
					onClick={onReset}
					disabled={!hasFilter}
					className="h-auto p-0 text-[13.5px]"
				>
					Reset all
				</Button>
			</div>

			{/* Division (single choice) */}
			{divisions.length > 0 && (
				<fieldset className="flex flex-col gap-0.5">
					<legend className="mb-2.5 text-sm font-semibold">Division</legend>
					{[{ _id: "", name: "All divisions", count: totalTours }, ...divisions].map((d) => (
						<label
							key={d._id || "all"}
							htmlFor={`${idPrefix}-div-${d._id || "all"}`}
							className="flex min-h-8.5 cursor-pointer items-center gap-2.5 text-sm"
						>
							<input
								id={`${idPrefix}-div-${d._id || "all"}`}
								type="radio"
								name={`${idPrefix}-division`}
								checked={division === d._id}
								onChange={() => onDivisionChange(d._id)}
								className="size-4.5 cursor-pointer accent-primary"
							/>
							<span className="flex-1">{d.name}</span>
							<span className="text-[12.5px] text-muted-foreground">{d.count}</span>
						</label>
					))}
				</fieldset>
			)}

			{/* Tour type (single choice) */}
			{tourTypes.length > 0 && (
				<fieldset>
					<legend className="mb-2.5 text-sm font-semibold">Tour type</legend>
					<div className="flex flex-wrap gap-2">
						{[{ _id: "", name: "All", count: totalTours }, ...tourTypes].map((t) => {
							const active = tourType === t._id;
							return (
								<button
									key={t._id || "all"}
									type="button"
									aria-pressed={active}
									onClick={() => onTourTypeChange(t._id)}
									className={[
										"inline-flex h-9.5 items-center gap-1.5 rounded-full border-[1.5px] px-3.5 text-[13.5px] font-medium transition-colors",
										active
											? "border-primary bg-accent text-primary"
											: "border-border bg-background text-foreground hover:border-primary/50",
									].join(" ")}
								>
									{t.name}
									<span className={active ? "text-primary/70" : "text-muted-foreground"}>{t.count}</span>
								</button>
							);
						})}
					</div>
				</fieldset>
			)}
		</div>
	);
}

/* ───────── page ───────── */

export default function Tours() {
	const [searchInput, setSearchInput] = useState("");
	const [searchTerm, setSearchTerm] = useState("");
	const [division, setDivision] = useState("");
	const [tourType, setTourType] = useState("");
	const [sort, setSort] = useState("-createdAt");
	const [page, setPage] = useState(1);

	// only send filled params
	const params = {
		page,
		limit: LIMIT,
		sort,
		...(searchTerm && { searchTerm }),
		...(division && { division }),
		...(tourType && { tourType }),
	};

	const { data, isLoading, isError, isFetching, refetch } = useGetToursQuery(params);

	// all tours, only division + tourType → for sidebar options & counts
	const { data: optionData } = useGetToursQuery({ limit: 1000, fields: "division,tourType" });

	const { divisions, tourTypes, totalTours } = useMemo(() => {
		const all = optionData?.data ?? [];
		return {
			divisions: buildOptions(all, "division"),
			tourTypes: buildOptions(all, "tourType"),
			totalTours: all.length,
		};
	}, [optionData]);

	const tours = data?.data ?? [];
	const total = data?.meta?.total ?? 0;
	const totalPage = data?.meta?.totalPage ?? 1;

	const activeFilterCount = [division, tourType].filter(Boolean).length;

	// any filter change → back to page 1
	const handleSearch = (e: FormEvent<HTMLFormElement>) => {
		e.preventDefault();
		setSearchTerm(searchInput.trim());
		setPage(1);
	};
	const handleDivision = (value: string) => {
		setDivision(value);
		setPage(1);
	};
	const handleTourType = (value: string) => {
		setTourType(value);
		setPage(1);
	};
	const handleSort = (value: string) => {
		setSort(value);
		setPage(1);
	};
	const handleReset = () => {
		setDivision("");
		setTourType("");
		setSearchInput("");
		setSearchTerm("");
		setPage(1);
	};
	const goToPage = (p: number) => {
		setPage(p);
		window.scrollTo({ top: 0, behavior: "smooth" });
	};

	const filterProps = {
		divisions,
		tourTypes,
		totalTours,
		division,
		tourType,
		onDivisionChange: handleDivision,
		onTourTypeChange: handleTourType,
		onReset: handleReset,
	};

	return (
		<div className="bg-background">
			{/* ───── Hero + search ───── */}
			<section className="bg-muted">
				<div className="mx-auto flex max-w-[1200px] flex-col gap-4.5 px-4 py-12 sm:px-10">
					<nav aria-label="Breadcrumb" className="flex gap-2 text-[13.5px] text-muted-foreground">
						<Link to="/" className="hover:text-foreground">
							Home
						</Link>
						<span aria-hidden="true">/</span>
						<span className="font-medium text-primary">Tours</span>
					</nav>

					<h1 className="text-3xl font-bold tracking-tight sm:text-[40px]">Find your tour</h1>

					<form
						role="search"
						onSubmit={handleSearch}
						className="flex max-w-[720px] flex-col gap-2.5 rounded-xl bg-card p-2.5 shadow-card sm:flex-row"
					>
						<label htmlFor="tour-search" className="sr-only">
							Search tours
						</label>
						<div className="relative flex-1">
							<Search className="pointer-events-none absolute top-1/2 left-3.5 size-4.5 -translate-y-1/2 text-muted-foreground" />
							<Input
								id="tour-search"
								type="search"
								value={searchInput}
								onChange={(e) => setSearchInput(e.target.value)}
								placeholder="Search by tour name or location…"
								className="h-11.5 rounded-lg border-transparent pl-11 text-[15px] shadow-none"
							/>
						</div>
						<Button type="submit" className="h-11.5 rounded-lg px-6 text-[14.5px] font-semibold">
							Search
						</Button>
					</form>
				</div>
			</section>

			{/* ───── Filters + results ───── */}
			<div className="mx-auto flex max-w-[1200px] items-start gap-8 px-4 pt-10 pb-22 sm:px-10">
				{/* Filters (desktop) */}
				<aside aria-label="Filters" className="hidden w-70 shrink-0 rounded-xl bg-card p-6 shadow-card lg:block">
					<Filters {...filterProps} idPrefix="desk" />
				</aside>

				{/* Results */}
				<main className="flex min-w-0 flex-1 flex-col gap-6">
					{/* Toolbar */}
					<div className="flex flex-col gap-3 sm:flex-row sm:items-center sm:justify-between">
						<p>
							{isLoading ? (
								<span className="text-muted-foreground">Loading tours…</span>
							) : (
								<>
									<strong>
										{total} {total === 1 ? "tour" : "tours"}
									</strong>{" "}
									<span className="text-muted-foreground">found{searchTerm && <> for “{searchTerm}”</>}</span>
								</>
							)}
						</p>

						<div className="flex items-center gap-2.5">
							{/* Filters (mobile) */}
							<Sheet>
								<SheetTrigger asChild>
									<Button
										variant="outline"
										type="button"
										className="h-11.5 flex-1 gap-2 rounded-lg border-[1.5px] border-primary text-primary lg:hidden"
									>
										<SlidersHorizontal className="size-4.5" />
										Filters{activeFilterCount > 0 && ` · ${activeFilterCount}`}
									</Button>
								</SheetTrigger>
								<SheetContent side="left" className="w-80 overflow-y-auto p-6">
									<SheetHeader className="sr-only">
										<SheetTitle>Filters</SheetTitle>
									</SheetHeader>
									<Filters {...filterProps} idPrefix="mob" />
								</SheetContent>
							</Sheet>

							<label htmlFor="tour-sort" className="hidden text-sm text-muted-foreground sm:block">
								Sort by
							</label>
							<select
								id="tour-sort"
								value={sort}
								onChange={(e) => handleSort(e.target.value)}
								className={`${selectCls} flex-1 sm:w-48 sm:flex-none`}
							>
								{SORT_OPTIONS.map((o) => (
									<option key={o.value} value={o.value}>
										{o.label}
									</option>
								))}
							</select>
						</div>
					</div>

					{/* ── Loading ── */}
					{isLoading && (
						<div className="grid grid-cols-1 gap-6 sm:grid-cols-2 xl:grid-cols-3">
							{Array.from({ length: 6 }).map((_, i) => (
								<div key={i} className="overflow-hidden rounded-xl bg-card shadow-card">
									<Skeleton className="h-50 rounded-none" />
									<div className="flex flex-col gap-2.5 p-4.5">
										<Skeleton className="h-3.5 w-28" />
										<Skeleton className="h-5 w-full" />
										<Skeleton className="h-3.5 w-40" />
										<Skeleton className="mt-2 h-6 w-24" />
									</div>
								</div>
							))}
						</div>
					)}

					{/* ── Error ── */}
					{isError && (
						<div className="flex flex-col items-center gap-3 rounded-xl bg-card px-6 py-16 text-center shadow-card">
							<span className="flex size-12 items-center justify-center rounded-full bg-destructive/10 text-destructive">
								<AlertCircle className="size-5" />
							</span>
							<div>
								<p className="font-semibold">Couldn't load tours</p>
								<p className="text-[13.5px] text-muted-foreground">Check your connection and try again.</p>
							</div>
							<Button variant="outline" onClick={() => refetch()} className="h-10 gap-2 rounded-lg">
								<RefreshCw className="size-4" />
								Try again
							</Button>
						</div>
					)}

					{/* ── Empty ── */}
					{!isLoading && !isError && tours.length === 0 && (
						<div className="flex flex-col items-center gap-3 rounded-xl bg-card px-6 py-16 text-center shadow-card">
							<span className="flex size-12 items-center justify-center rounded-full bg-accent text-accent-foreground">
								<SearchX className="size-5" />
							</span>
							<div>
								<p className="font-semibold">No tours found</p>
								<p className="text-[13.5px] text-muted-foreground">Try a different search or clear the filters.</p>
							</div>
							<Button variant="outline" onClick={handleReset} className="h-10 rounded-lg">
								Clear search & filters
							</Button>
						</div>
					)}

					{/* ── Cards ── */}
					{!isLoading && !isError && tours.length > 0 && (
						<div
							className={[
								"grid grid-cols-1 gap-6 transition-opacity sm:grid-cols-2 xl:grid-cols-3",
								isFetching ? "opacity-60" : "",
							].join(" ")}
						>
							{tours.map((tour) => (
								<Link
									key={tour._id}
									to={`/tours/${tour.slug}`}
									className="group flex flex-col overflow-hidden rounded-xl bg-card text-foreground shadow-card transition-shadow hover:shadow-lg focus-visible:ring-3 focus-visible:ring-ring/50 focus-visible:outline-none"
								>
									{/* Image */}
									<div className="relative flex h-50 items-center justify-center overflow-hidden bg-accent text-muted-foreground">
										{tour.images?.[0] ? (
											<img
												src={tour.images[0]}
												alt={tour.title}
												loading="lazy"
												className="size-full object-cover transition-transform duration-300 group-hover:scale-105"
											/>
										) : (
											<ImageIcon className="size-7" />
										)}
										{tour.tourType && (
											<span className="absolute top-3 left-3 rounded-full bg-card px-2.5 py-1 text-xs font-semibold text-foreground">
												{tour.tourType.name}
											</span>
										)}
									</div>

									{/* Body */}
									<div className="flex flex-1 flex-col gap-1.5 px-4.5 pt-4 pb-4.5">
										{(tour.location || tour.division) && (
											<span className="flex items-center gap-1 text-[12.5px] font-medium text-primary">
												<MapPin className="size-3.5 shrink-0" />
												<span className="truncate">
													{[tour.location, tour.division?.name].filter(Boolean).join(", ")}
												</span>
											</span>
										)}
										<h3 className="text-base leading-snug font-semibold group-hover:text-primary">{tour.title}</h3>
										{getTourMeta(tour) && <p className="text-[12.5px] text-muted-foreground">{getTourMeta(tour)}</p>}
										<div className="mt-auto pt-1.5">
											<span className="text-xs text-muted-foreground">from </span>
											<strong className="text-lg font-bold text-primary">৳{formatPrice(tour.costFrom)}</strong>
										</div>
									</div>
								</Link>
							))}
						</div>
					)}

					{/* ── Pagination ── */}
					{!isLoading && !isError && totalPage > 1 && (
						<nav aria-label="Pagination" className="flex flex-wrap justify-center gap-2 pt-4">
							<Button
								variant="outline"
								size="icon"
								onClick={() => goToPage(page - 1)}
								disabled={page === 1 || isFetching}
								aria-label="Previous page"
								className="size-11 rounded-lg border-[1.5px]"
							>
								<ChevronLeft className="size-4.5" />
							</Button>

							{getPageNumbers(page, totalPage).map((p, i) =>
								p === "…" ? (
									<span key={`dots-${i}`} className="flex size-11 items-center justify-center text-muted-foreground">
										…
									</span>
								) : (
									<Button
										key={p}
										variant={p === page ? "default" : "outline"}
										onClick={() => goToPage(p)}
										disabled={isFetching}
										aria-current={p === page ? "page" : undefined}
										className="size-11 rounded-lg border-[1.5px] font-medium"
									>
										{p}
									</Button>
								),
							)}

							<Button
								variant="outline"
								size="icon"
								onClick={() => goToPage(page + 1)}
								disabled={page >= totalPage || isFetching}
								aria-label="Next page"
								className="size-11 rounded-lg border-[1.5px]"
							>
								<ChevronRight className="size-4.5" />
							</Button>
						</nav>
					)}
				</main>
			</div>
		</div>
	);
}
```

### `src/redux/features/Tour/tour.api.ts` — `getTours` অংশ

```ts
type TToursParams = {
  page?: number;
  limit?: number;
  searchTerm?: string; // title, description, location-এ খোঁজে
  sort?: string;       // "-createdAt" (default) | "costFrom" | "-costFrom"
  division?: string;   // division-এর _id
  tourType?: string;   // tourType-এর _id
  fields?: string;     // যেমন "division,tourType"
};

getTours: builder.query<{ data: ITour[]; meta: IMeta }, TToursParams | void>({
  query: (params) => ({
    url: "/tour",
    method: "GET",
    params: params ?? undefined,
  }),
  providesTags: ["TOUR"],
  transformResponse: (response: IResponse<ITour[]> & { meta: IMeta }) => ({
    data: response.data,
    meta: response.meta,
  }),
}),
```

---

# Check-list

- [ ] `/`-এ Home, `/tours`-এ tour-এর তালিকা, দুটোতেই Navbar/Footer
- [ ] Search চাপলে (বা Enter) তবেই খোঁজে, টাইপের সময় request যায় না
- [ ] Network-এ খালি param যায় না (`?searchTerm=` নেই)
- [ ] Division/type/sort বদলালে পাতা ১-এ ফেরে
- [ ] Sidebar-এ শুধু যে division-এ tour আছে, পাশে সঠিক সংখ্যা
- [ ] Mobile-এ Filters sheet, desktop-এর filter-এর সাথে একই অবস্থা; console-এ duplicate id-র warning নেই
- [ ] Card-এর যেকোনো জায়গায় click করলে details; Tab দিয়ে card-এ এলে দাগ দেখা যায়
- [ ] ১০ পাতা হলে `1 … 4 5 6 … 10`; পাতা বদলালে উপরে scroll
- [ ] (উন্নতি ১-এর পরে) filter দিয়ে tour খুলে Back চাপলে একই filter আর পাতা; reload আর share-করা link-এও
