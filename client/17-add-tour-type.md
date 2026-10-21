# Frontend Notes — ১৭: Tour Type যোগ করা — RTK Query দিয়ে Data আনা, পাঠানো আর তালিকা নিজে থেকে আপডেট

এই নোটে admin dashboard-এর প্রথম আসল কাজের page বানাব: **Tour types**। বাম পাশে নতুন tour type যোগ করার form, ডান পাশে সব tour type-এর তালিকা। নতুন একটা যোগ করলে তালিকা reload ছাড়াই আপডেট হবে।

নোট দুই ভাগে:
1. **ধাপে ধাপে (ধাপ ১–১০):** teacher যে ক্রমে করেছেন, আপনার কোড দিয়ে।
2. **উন্নতি আর চূড়ান্ত কোড:** teacher-এর কোডে error দেখানো, loading, খালি তালিকা এসব ছিল না। শেষে সেগুলো যোগ করে পুরো চূড়ান্ত কোড।

প্রতিটা ধাপে আগের মতো 🎯 লক্ষ্য, 🛠️ কাজ, 💡 কেন, ✅ যাচাই।

> নোট: আপনার project-এর version মেনে লেখা (RTK 2, React Hook Form 7, Zod 4, shadcn `radix-nova`)। আপনার বর্তমান কোড পড়ে নিয়েই লেখা, পুরনো ধরনের কোড দিয়ে কিছু বদলানো হয়নি।

---

## ভূমিকা: Tour-এর আগে Tour Type কেন

Postman-এ backend-এর **create tour** API দেখলে বোঝা যায়, একটা tour তৈরি করতে দুটো জিনিস পাঠাতে হয়:

1. **ছবি (image)**
2. **data**, যার ভেতরে আরও আছে **`tourType`** আর **`division`**

`tourType` আর `division`-এ নাম না, তাদের **`_id`** যায় (MongoDB-তে এক collection অন্যটার সাথে id দিয়ে জোড়া থাকে)। অর্থাৎ tour বানানোর আগে database-এ tour type আর division থাকতেই হবে, নাহলে tour-এর form-এ বেছে নেওয়ার মতো কিছু থাকবে না।

```
Tour type  ─┐
            ├──→  Tour (tourType: id, division: id, image, …)
Division   ─┘
```

তাই ক্রম: **আগে tour type (এই নোট) → division → তারপর tour।**

### ধাপের মানচিত্র

| ভাগ | ধাপ | শেষে কী হবে |
|---|---|---|
| **ক. Backend দেখা ও Page** | ১–২ | `/admin/tour-types`-এ design দেখা যায় |
| **খ. Data আনা** | ৩–৬ | Backend-এর আসল tour type-গুলো তালিকায় দেখায় |
| **গ. Data পাঠানো** | ৭–৯ | Form দিয়ে নতুন tour type যোগ হয়, তালিকা নিজে আপডেট হয় |
| **ঘ. কী বাকি** | ১০ | Error, loading, খালি তালিকা ইত্যাদি সমস্যা চেনা |
| **উন্নতি ও চূড়ান্ত কোড** | — | সব ঠিক করা পুরো কোড |

---

# ভাগ ক: Backend দেখা ও Page

## ধাপ ১ — Backend থেকে field দেখে নেওয়া

🎯 **লক্ষ্য:** কোড লেখার আগে জানা, backend কী নেয় আর কী দেয়।

🛠️ **কাজ:** Postman-এ দুটো API দেখুন:

| কাজ | Method + URL | পাঠাই | ফেরত পাই |
|---|---|---|---|
| নতুন tour type | `POST /tour/create-tour-type` | `{ "name": "Short tour" }` | নতুন tour type |
| সব tour type | `GET /tour/tour-types` | কিছু না | tour type-এর তালিকা |

`GET`-এর response মোটামুটি এমন (আমাদের backend-এর সব response-এর একই আকার, ১৩ নম্বর নোটের `IResponse`):

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Tour types retrieved successfully",
  "data": [
    { "_id": "6650f1…", "name": "Short tour", "createdAt": "…", "updatedAt": "…" },
    { "_id": "6650f2…", "name": "Long tour", "createdAt": "…", "updatedAt": "…" }
  ]
}
```

💡 **কেন আগে দেখা:** URL-এর বানান, body-তে কোন key (`name`, নাকি `title`?), response-এ data কোথায়, এগুলো না জেনে frontend লিখলে অন্ধকারে ঢিল ছোড়া। Postman-এ একবার দেখে নিলে পরে ভুল কম। আপনার backend-এর আসল response-এ field একটু আলাদা হলে (যেমন `createdAt` না থাকা) সেটাই মানবেন।

✅ **যাচাই:** Postman-এ (admin token দিয়ে) দুটো API-ই কাজ করে।

---

## ধাপ ২ — Page-এর design আর route

🎯 **লক্ষ্য:** `/admin/tour-types`-এ page-এর কাঠামো দেখানো (এখনো hardcode data)।

🛠️ **কাজ ১ — design:** আপনি Claude-এর সাহায্যে একটা design বানিয়েছেন: বাম পাশে form, ডান পাশে তালিকা (৩টা নমুনা row hardcode)। পুরো কোড শেষের "চূড়ান্ত কোড"-এ আছে, এখানে শুধু কাঠামো:

```tsx
// src/pages/Admin/AddTourType.tsx  (কাঠামো)
<div className="flex w-full max-w-6xl flex-col gap-5">
  <div>{/* শিরোনাম: Tour types */}</div>

  <div className="grid grid-cols-1 items-start gap-5 lg:grid-cols-[360px_1fr]">
    <section>{/* বাম: Add form */}</section>
    <section>{/* ডান: All tour types-এর তালিকা */}</section>
  </div>
</div>
```

💡 **`lg:grid-cols-[360px_1fr]`:** বড় screen-এ দুই column: বামটা ঠিক ৩৬০px (form), ডানটা বাকি সব জায়গা (`1fr`)। ছোট screen-এ `grid-cols-1`, তাই form উপরে, তালিকা নিচে।

🛠️ **কাজ ২ — route:** ১৪ নম্বর নোটের ব্যবস্থায় শুধু sidebar item-এ লিখলেই route তৈরি হয়, আর ১৫ নম্বর নোট অনুযায়ী lazy:

```ts
// src/routes/adminSidebarItems.ts
const AddTourType = lazy(() => import("@/pages/Admin/AddTourType"));

// "Tour Management" section-এ
{ title: "Tour types", url: "/admin/tour-types", component: AddTourType },
```

> নোট: আপনার কোডে menu-র লেখা এখন `"Tour type"` (একবচন), আর page-এর শিরোনাম "Tour types"। এই page-এ সব tour type-এর তালিকা থাকে, তাই menu-তেও `"Tour types"` রাখলে দুই জায়গায় মিল থাকে।

✅ **যাচাই:** Sidebar-এর "Tour types" চাপলে design দেখায় (এখনো নমুনা data)।

---

# ভাগ খ: Data আনা

## ধাপ ৩ — `tour.api.ts`: নতুন feature-এর API ফাইল

🎯 **লক্ষ্য:** Tour-সংক্রান্ত সব API-র জন্য আলাদা ফাইল।

🛠️ **কাজ:**

```ts
// src/redux/features/Tour/tour.api.ts  (teacher-এর কোড)
import { baseApi } from "@/redux/baseApi";

export const tourApi = baseApi.injectEndpoints({
  endpoints: (builder) => ({
    addTourType: builder.mutation({
      query: (tourTypeName) => ({
        url: "/tour/create-tour-type",
        method: "POST",
        data: tourTypeName,
      }),
    }),
    getTourTypes: builder.query({
      query: () => ({
        url: "/tour/tour-types",
        method: "GET",
      }),
      transformResponse: (response) => response.data,
    }),
  }),
});

export const { useGetTourTypesQuery, useAddTourTypeMutation } = tourApi;
```

💡 **কেন আলাদা ফাইল:** ১৩ নম্বর নোটের module pattern। Auth-এর endpoint `features/auth/auth.api.ts`-এ, tour-এর endpoint `features/Tour/tour.api.ts`-এ। দুটোই একই `baseApi`-তে `injectEndpoints` দিয়ে যোগ হয়, তাই store-এ নতুন কিছু connect করতে হয় না।

| Endpoint | ধরন | কেন |
|---|---|---|
| `addTourType` | `mutation` | Server-এ নতুন কিছু **তৈরি** করে (POST) |
| `getTourTypes` | `query` | Server থেকে **পড়ে** (GET), page খুললেই নিজে চলে |

(Query আর mutation-এর পার্থক্য ১৩ নম্বর নোটের ধাপ ১২-এ।)

> নোট: Feature folder-এর নাম `auth` ছোট হাতের, কিন্তু `Tour` বড় হাতের। দুটোতে একই নিয়ম রাখুন (ছোট হাতের `tour` প্রচলিত)। Mac-এর file system বড়-ছোট হাতের পার্থক্য ধরে না, কিন্তু Linux server (deploy) ধরে, তখন import-এর বানান না মিললে build ভাঙে। নাম বদলাতে চাইলে editor-এর rename দিয়ে করুন, যাতে import-গুলোও বদলায়।

---

## ধাপ ৪ — `transformResponse`: শুধু দরকারি অংশ রাখা

🎯 **লক্ষ্য:** Component-এ সরাসরি tour type-এর array পাওয়া।

### 💡 সমস্যা

Backend পুরো response পাঠায়: `{ statusCode, success, message, data: [...] }`। Component-এ আমাদের লাগে শুধু `data`-র ভেতরের array। `transformResponse` না দিলে প্রতিবার লিখতে হতো:

```tsx
const { data } = useGetTourTypesQuery(undefined);
data?.data?.map(...)   // দুইবার data (১৩ নম্বর নোটের Navbar-এর মতো)
```

### 💡 সমাধান

```ts
transformResponse: (response) => response.data,
```

Response cache-এ রাখার **আগে** RTK Query এই function চালায়, আর যা ফেরত দেয় সেটাই `data` হিসেবে component পায়:

```
Backend:   { statusCode, success, message, data: [A, B, C] }
              ↓ transformResponse: (response) => response.data
Component: data = [A, B, C]      → data?.map(...)  ✅ একবারই data
```

Teacher-এর কথা: আমাদের এখানে `statusCode` বা `message` লাগছে না, শুধু data। কোনো endpoint-এ শুধু message লাগলে `(response) => response.message` দিলে শুধু message-ই পাবেন। যেটা লাগে, সেটাই রাখুন।

> নোট — একটা সতর্কতা: পরে tour-এর তালিকায় backend সাধারণত pagination-এর তথ্য পাঠায় (`meta: { page, total … }`), সেটা `data`-র **পাশে** থাকে। তখন `response.data` দিলে `meta` হারিয়ে যাবে। সেখানে পুরো response রাখবেন, বা `(response) => ({ data: response.data, meta: response.meta })`।

---

## ধাপ ৫ — Data আসছে কিনা পরীক্ষা

🎯 **লক্ষ্য:** UI-তে বসানোর আগে console-এ data দেখে নেওয়া।

🛠️ **কাজ (শুধু পরীক্ষা):**

```tsx
// src/pages/Admin/AddTourType.tsx-এর ভেতরে
const { data, isLoading } = useGetTourTypesQuery(undefined);
console.log(data);
```

> ⚠️ সংশোধন: নোটে লেখা ছিল `const { data, loading } = useGetTourTypesQuery(undefined)`। RTK Query-র hook-এ `loading` নামে কিছু নেই, নাম **`isLoading`**। `loading` লিখলে TypeScript error দেবে (বা type না থাকলে চুপচাপ সবসময় `undefined` হবে)। Hook আর কী কী দেয়:
>
> | নাম | মানে |
> |---|---|
> | `data` | ফলাফল (`transformResponse`-এর পরে) |
> | `isLoading` | **প্রথমবার** আনছে, এখনো কোনো data নেই |
> | `isFetching` | যেকোনো সময় আনছে (refetch-সহ) |
> | `isError`, `error` | ব্যর্থ হয়েছে, আর error-এর তথ্য |
> | `isSuccess` | সফলভাবে এসেছে |
> | `refetch` | হাতে আবার আনার function |

✅ **যাচাই:** Console-এ tour type-এর একটা array (`statusCode`/`message` ছাড়া, ধাপ ৪-এর কারণে)। দেখা হলে `console.log` মুছে দিন।

---

## ধাপ ৬ — তালিকায় আসল data দেখানো

🎯 **লক্ষ্য:** নমুনা row-এর বদলে backend-এর tour type।

🛠️ **কাজ:**

```tsx
<ul className="list-none">
  {data?.map((item) => (
    <li key={item._id} className="flex items-center gap-3.5 border-t border-border px-4.5 py-3.5">
      {/* ... icon */}
      <strong className="block truncate">{item?.name}</strong>
      {/* ... buttons */}
    </li>
  ))}
</ul>
```

💡 **লাইনগুলোর মানে:**
- **`data?.map`:** প্রথমবার data আসার আগে `data` হলো `undefined`, আর `undefined.map(...)` page ভেঙে দেয়। `?.` মানে "থাকলে তবেই map করো"।
- **`key={item._id}`:** তালিকার প্রতিটা item-কে React-এর একটা স্থায়ী পরিচয় দিতে হয়, যাতে নতুন item যোগ হলে React বুঝতে পারে কোনটা নতুন আর কোনটা আগের। Database-এর `_id` সবসময় আলাদা আর কখনো বদলায় না, তাই এটাই সেরা key। (Teacher-এর কোডে `key` ছিল না, console-এ warning দিত; আপনার কোডে আছে, ঠিক আছে।)

✅ **যাচাই:** তালিকায় Postman-এর মতো একই tour type-গুলো দেখায়।

---

# ভাগ গ: Data পাঠানো

## ধাপ ৭ — Page আর Form আলাদা ফাইলে

🎯 **লক্ষ্য:** বড় page-কে দুই ভাগে ভাগ করা।

🛠️ **কাজ:**

```
src/
├── pages/Admin/
│   └── AddTourType.tsx                          ← page: শিরোনাম + layout + তালিকা
└── components/modules/Admin/TourType/
    └── AddTourTypeForm.tsx                      ← শুধু form
```

```tsx
// page-এ
import AddTourTypeForm from "@/components/modules/Admin/TourType/AddTourTypeForm";

<AddTourTypeForm />
```

💡 **কেন (teacher-এর কথা):** একই page-এ পরে আরও জিনিস আসবে (edit-এর modal, delete-এর confirm dialog …)। সব এক ফাইলে রাখলে ফাইল বিশাল হয়ে যায়। ৬ নম্বর নোটের folder নিয়ম অনুযায়ী page-এর এক-একটা অংশ থাকে `components/modules/<Page>/`-এ। Form-এর নিজের state, validation, submit সব ওর ফাইলেই থাকে; page শুধু জোড়া লাগায়।

✅ **যাচাই:** Page আগের মতোই দেখায়।

---

## ধাপ ৮ — Form-এ React Hook Form + Zod + mutation

🎯 **লক্ষ্য:** Form submit করলে backend-এ নতুন tour type তৈরি হবে।

🛠️ **কাজ (আপনার কোড, মূল অংশ):**

```tsx
// src/components/modules/Admin/TourType/AddTourTypeForm.tsx
const formSchema = z.object({
  name: z
    .string()
    .trim()
    .min(2, { error: "Name must be at least 2 characters long." })
    .max(50, { error: "Name cannot exceed 50 characters." }),
});

type FormValues = z.infer<typeof formSchema>;

export default function AddTourTypeForm() {
  const [addTourType] = useAddTourTypeMutation();

  const form = useForm<FormValues>({
    resolver: zodResolver(formSchema),
    defaultValues: { name: "" },
  });

  const onSubmit = async (data: FormValues) => {
    const res = await addTourType({ name: data.name }).unwrap();
    if (res.success) {
      toast.success("Tour Type Added");
    }
  };

  // JSX: Controller + Field + Input + FieldError (১২ নম্বর নোটের মতো)
}
```

💡 **কিছু অংশ:**
- **`.trim()` তারপর `.min(2)`:** `"  a  "` দিলে আগে দুই পাশের space কেটে `"a"`, তারপর গোনে। শুধু space দিয়ে নাম দেওয়া যায় না (১২ নম্বর নোট)।
- **`addTourType({ name: data.name })`:** ধাপ ১-এ দেখা body-র আকার ঠিক এটাই।
- **`FieldError`-এর ভেতরে icon + message (children):** আপনার design-এ error-এর আগে ⚠️ icon আছে। shadcn-এর `FieldError`-এ `errors` না দিয়ে ভেতরে (children) নিজের লেখা দিলে সেটাই দেখায়, তাই icon বসানো গেছে। ✅

✅ **যাচাই:** নতুন নাম দিয়ে submit → success toast, Network-এ `create-tour-type` 201। কিন্তু **ডান পাশের তালিকায় নতুনটা আসে না**, reload দিলে তবেই আসে। পরের ধাপে।

---

## ধাপ ৯ — যোগ করার পরে তালিকা আপডেট: Tags

🎯 **লক্ষ্য:** নতুন tour type যোগ হলেই তালিকা নিজে থেকে নতুন করে আসবে।

### 💡 কেন তালিকা আপডেট হয় না

`getTourTypes` একবার data এনে cache-এ রেখেছে। নতুন tour type server-এ তৈরি হলো, কিন্তু RTK Query জানে না যে তালিকাটা এখন পুরনো। তাই cache থেকে পুরনো তালিকাই দেখায়। ১৩ নম্বর নোটের logout-এর ঠিক একই সমস্যা (ধাপ ২৫–২৬)।

### 🛠️ কাজ

```ts
// src/redux/baseApi.ts
tagTypes: ["USER", "TOUR"],
```

```ts
// src/redux/features/Tour/tour.api.ts
addTourType: builder.mutation({
  // ...
  invalidatesTags: ["TOUR"],   // সফল হলে "TOUR" লেবেলের data বাতিল
}),
getTourTypes: builder.query({
  // ...
  providesTags: ["TOUR"],      // আমার data-র গায়ে "TOUR" লেবেল
}),
```

এখন: নতুন tour type যোগ → `TOUR` invalidate → যে component `getTourTypes` ব্যবহার করছে (আমাদের page), তার জন্য RTK Query নিজে আবার `/tour/tour-types` আনে → তালিকায় নতুনটা চলে আসে।

Logout-এর সময় এই কৌশল কাজ করেনি (refetch-এ error এসেছিল, ১৩ নম্বর নোট); এখানে করে, কারণ refetch সফল হয় আর নতুন data এসে পুরনোটা বদলে দেয়।

Teacher প্রথমে tag ছাড়া লিখেছিলেন, তারপর reload লাগছে দেখে যোগ করেছেন। আপনি আগেই রেখেছিলেন। ✅

### 💡 আপনার প্রশ্ন: tag-এর নাম `"TOUR"` রাখা কি ঠিক?

**কাজ করে, তবে ভালো নাম `"TOUR_TYPE"`।**

Tag-এর নাম যেকোনো string হতে পারে, শুধু `tagTypes`, `providesTags`, `invalidatesTags` তিন জায়গায় একই হতে হয়। তাই `"TOUR"` দিয়ে এখন সব ঠিকঠাক চলছে।

কিন্তু tag-এর আসল কাজ: **"কোন data বদলালে কোন তালিকা আবার আনতে হবে"** সেটা বলা। Tour type আর tour আলাদা জিনিস (আলাদা collection, আলাদা API)। পরে tour-এর তালিকার জন্য আরেকটা tag লাগবে, আর তখন:

| যদি দুটোতেই `"TOUR"` দিই | ফল |
|---|---|
| নতুন **tour type** যোগ | tour type-এর তালিকা **আর** tour-এর তালিকা, দুটোই আবার আসে |
| নতুন **tour** যোগ | tour-এর তালিকা **আর** tour type-এর তালিকা, দুটোই আবার আসে |

একটা বদলালে অন্যটাও অকারণে আবার আসবে, বাড়তি request। আলাদা নাম দিলে প্রতিটা শুধু নিজের তালিকা আপডেট করে:

```ts
tagTypes: ["USER", "TOUR_TYPE", "TOUR", "DIVISION"],  // পরে যেগুলো আসবে সহ
```

**নিয়ম:** প্রতিটা আলাদা ধরনের data (প্রতিটা backend collection) = একটা tag। নাম দেখেই যেন বোঝা যায় কোন data।

✅ **যাচাই:** নতুন tour type যোগ করুন → reload ছাড়াই তালিকায় চলে আসে, Network-এ `create-tour-type`-এর ঠিক পরে নিজে থেকে `tour-types` request।

---

# ভাগ ঘ: কী বাকি

## ধাপ ১০ — Teacher-এর কোডে যা নেই (সমস্যা চেনা)

🎯 **লক্ষ্য:** Page "চলছে", কিন্তু আসল user ব্যবহার করলে কোথায় ঝামেলা হবে, সেটা খুঁজে বের করা।

✅ **যাচাই (এবং সমস্যা)** — নিজে করে দেখুন:

| # | পরীক্ষা | কী হয় | সমস্যা |
|---|---|---|---|
| ১ | আগে থেকে আছে এমন নাম দিয়ে আবার যোগ করুন ("Short tour") | কিছুই দেখায় না, console-এ লাল *Uncaught (in promise)* | `onSubmit`-এ `try/catch` নেই। `.unwrap()` error throw করে, কেউ ধরে না, user কিছুই জানে না। অথচ আপনার design-এ ঠিক এই error-এর জায়গা আছে ("Short tour" already exists) |
| ২ | একটা নাম সফলভাবে যোগ করুন | Toast আসে, কিন্তু input-এ নামটা থেকে যায় | Form reset হয় না, আরেকটা যোগ করতে আগে হাতে মুছতে হয় |
| ৩ | Network-এ Slow 4G দিয়ে page খুলুন | ডান পাশে কিছুক্ষণ শুধু শিরোনাম, তালিকা ফাঁকা | Loading অবস্থা নেই, মনে হয় কোনো tour type নেই |
| ৪ | Backend বন্ধ রেখে page খুলুন | তালিকা ফাঁকা, কোনো বার্তা নেই | Error অবস্থা নেই |
| ৫ | সব tour type মুছে ফেলা database (নতুন project) | ফাঁকা সাদা জায়গা | খালি তালিকার (empty state) বার্তা নেই |
| ৬ | তালিকার উপরে দেখুন | সবসময় "3 types" | সংখ্যা hardcode |
| ৭ | প্রতিটা row দেখুন | সবার নিচে "46 tours · slug: short-tour" | Design-এর নমুনা লেখা রয়ে গেছে |
| ৮ | Screen reader / button-এর `aria-label` | সব row-এ "Rename Short tour" | নাম hardcode |
| ৯ | Edit (✏️) আর Delete (🗑️) | কিছু করে না; Delete সবসময় disabled, title-এ "In use by 46 tours" | এখনো কাজ নেই, অথচ দেখতে কাজের মতো |
| ১০ | Code-এ `res.success`, `item.name` | Editor কোনো suggestion দেয় না | API-তে type নেই (সব `any`) |

এগুলো সবই পরের অংশে ঠিক করা হলো।

---

# উন্নতি

## উন্নতি ১ — Type দেওয়া

Tour type-এর type:

```ts
// src/types/tour.type.ts
export interface ITourType {
  _id: string;
  name: string;
  createdAt?: string;
  updatedAt?: string;
}

export interface IAddTourType {
  name: string;
}
```

```ts
// src/types/index.ts  (উপরের দিকে)
export type { IAddTourType, ITourType } from "./tour.type";
```

Endpoint-এ:

```ts
addTourType: builder.mutation<IResponse<ITourType>, IAddTourType>({ ... }),
getTourTypes: builder.query<ITourType[], void>({
  // ...
  transformResponse: (response: IResponse<ITourType[]>) => response.data,
}),
```

- **`builder.query<ITourType[], void>`:** `transformResponse`-এর **পরের** আকার (array) আর argument নেই (`void`)। তাই component-এ `useGetTourTypesQuery()` লিখলেই হয়, `undefined` লাগে না।
- **`response: IResponse<ITourType[]>`:** backend যা পাঠায় (transform-এর **আগের** আকার)।
- এখন `item.name`, `item._id` লিখলে editor suggestion দেয়, ভুল বানানে লাল দাগ।

> ⚠️ সংশোধন (নাম): mutation-এর parameter-এর নাম ছিল `tourTypeName`, কিন্তু আসলে আসে একটা object (`{ name }`), শুধু নাম না। নাম দেখে ভুল বোঝা যায়, তাই `tourTypeInfo`।

## উন্নতি ২ — Tag-এর নাম `TOUR_TYPE` (ধাপ ৯-এর উত্তর অনুযায়ী)

## উন্নতি ৩ — Form: error দেখানো, reset, loading

```tsx
const onSubmit = async (data: FormValues) => {
  try {
    await addTourType({ name: data.name }).unwrap();
    toast.success(`"${data.name}" added`);
    form.reset();
  } catch (err) {
    form.setError("name", {
      type: "server",
      message: getErrorMessage(err, "Couldn't add the tour type. Please try again."),
    });
  }
};
```

- **`try/catch`:** `.unwrap()` ব্যর্থ হলে error `catch`-এ আসে (১৩ নম্বর নোট)।
- **`form.setError("name", …)`:** Backend-এর error (যেমন "already exists") **input-এর ঠিক নিচে** দেখায়, ঠিক যেমন আপনার design-এ ছিল। Zod-এর error যেখানে দেখায়, একই জায়গায়, একই চেহারায়। User নাম ঠিক করে আবার টাইপ শুরু করলেই message চলে যায় (প্রথম submit-এর পরে React Hook Form প্রতি অক্ষরে আবার যাচাই করে, তখন পুরনো error মুছে যায়)।
- **`form.reset()`:** সফল হলে input খালি, পরেরটা সাথে সাথে লেখা যায়।
- **Toast-এ নামসহ:** `"Short tour" added` — কোনটা যোগ হলো পরিষ্কার।
- **`isLoading` (mutation থেকে):** request চলার সময় button বন্ধ আর লেখা "Adding…", দুইবার চাপলে দুইবার যায় না।
- **`noValidate`:** আমাদের নিয়ম (১২ নম্বর নোট)।

`getErrorMessage` হলো ১৩ নম্বর নোটের ধাপ ১৪-এর helper। আপনার project-এ এটা এখনো আলাদা ফাইলে নেই (Verify page-এর ভেতরে লেখা), তাই এবার `utils`-এ বানিয়ে নিন, পরে Verify থেকেও এখান থেকে import করবেন:

```ts
// src/utils/getErrorMessage.ts
import type { TApiError } from "@/types";

export const getErrorMessage = (err: unknown, fallback: string) => {
  const error = err as TApiError;
  return error?.data?.errorSources?.[0]?.message || error?.data?.message || fallback;
};
```

## উন্নতি ৪ — তালিকা: loading, error, খালি অবস্থা, আসল সংখ্যা

একটা তালিকার সবসময় **চারটা অবস্থা** ভাবতে হয়:

| অবস্থা | কী দেখাব |
|---|---|
| লোড হচ্ছে | ৩টা skeleton row (shadcn `Skeleton`) |
| Error | লাল লেখায় "Couldn't load tour types" |
| খালি | "No tour types yet — add your first one." |
| Data আছে | আসল তালিকা |

আর:
- উপরে **আসল সংখ্যা**: `tourTypes.length` (১টা হলে "1 type", বেশি হলে "types")।
- Row-এর নিচের নমুনা লেখা ("46 tours · slug: short-tour") সরানো হয়েছে। Backend এই তথ্য পাঠায় না; পাঠালে পরে বসাবেন।
- `aria-label` এখন প্রতিটা row-এর নিজের নাম দিয়ে: `Rename Short tour`, `Rename Long tour`।
- Edit আর Delete এখনো কাজ করে না, তাই দুটোই `disabled`, title "Coming soon"। কাজ না করা button দেখতে চালু থাকলে user চেপে ভাবে app ভাঙা। Edit/Delete-এর endpoint এলে চালু করবেন। Comment করা edit-mode row-ও তখনকার জন্য রেখে দেওয়া যায়।

> নোট — teacher-এর পাঠের শিরোনাম "ShadCN Table": Teacher তালিকা দেখাতে shadcn-এর `Table` ব্যবহার করেছেন। আপনার design-এ এক column (শুধু নাম), তাই `<ul>` তালিকাই বেশি মানানসই। অনেক column হলে (যেমন পরে tour-এর তালিকা: নাম, দাম, division, তারিখ) `bunx --bun shadcn@latest add table` দিয়ে Table ব্যবহার করবেন।

## উন্নতি ৫ — ছোট নাম-সংশোধন

- Page ফাইল `AddTourType.tsx`, কিন্তু ভেতরের component `AddTourTypes` (শেষে `s`)। নিয়ম: ফাইলের নাম = component-এর নাম → `AddTourType`।
- Form-এর `id="add-tour-type"` কোথাও ব্যবহার হয় না (বাইরের কোনো button `form="..."` দিয়ে এটাকে ডাকে না), তাই সরানো হয়েছে।
- শিরোনামে `text-foreground` যোগ (আমাদের নিয়ম: লেখায় semantic token)।

---

# চূড়ান্ত কোড

### `src/types/tour.type.ts`

```ts
export interface ITourType {
  _id: string;
  name: string;
  createdAt?: string;
  updatedAt?: string;
}

export interface IAddTourType {
  name: string;
}
```

### `src/types/index.ts` (শুধু যোগ করা লাইন)

```ts
export type { IAddTourType, ITourType } from "./tour.type";
```

### `src/redux/baseApi.ts`

```ts
import { createApi } from "@reduxjs/toolkit/query/react";
import axiosBaseQuery from "./axiosBaseQuery";

export const baseApi = createApi({
  reducerPath: "baseApi",
  baseQuery: axiosBaseQuery(),
  tagTypes: ["USER", "TOUR_TYPE"],
  endpoints: () => ({}),
});
```

### `src/redux/features/Tour/tour.api.ts`

```ts
import { baseApi } from "@/redux/baseApi";
import type { IAddTourType, IResponse, ITourType } from "@/types";

export const tourApi = baseApi.injectEndpoints({
  endpoints: (builder) => ({
    addTourType: builder.mutation<IResponse<ITourType>, IAddTourType>({
      query: (tourTypeInfo) => ({
        url: "/tour/create-tour-type",
        method: "POST",
        data: tourTypeInfo,
      }),
      invalidatesTags: ["TOUR_TYPE"],
    }),

    getTourTypes: builder.query<ITourType[], void>({
      query: () => ({
        url: "/tour/tour-types",
        method: "GET",
      }),
      providesTags: ["TOUR_TYPE"],
      transformResponse: (response: IResponse<ITourType[]>) => response.data,
    }),
  }),
});

export const { useGetTourTypesQuery, useAddTourTypeMutation } = tourApi;
```

### `src/utils/getErrorMessage.ts`

```ts
import type { TApiError } from "@/types";

export const getErrorMessage = (err: unknown, fallback: string) => {
  const error = err as TApiError;
  return error?.data?.errorSources?.[0]?.message || error?.data?.message || fallback;
};
```

### `src/components/modules/Admin/TourType/AddTourTypeForm.tsx`

```tsx
import { Button } from "@/components/ui/button";
import { Field, FieldError, FieldLabel } from "@/components/ui/field";
import { Input } from "@/components/ui/input";
import { useAddTourTypeMutation } from "@/redux/features/Tour/tour.api";
import { getErrorMessage } from "@/utils/getErrorMessage";
import { zodResolver } from "@hookform/resolvers/zod";
import { AlertCircle, Plus } from "lucide-react";
import { Controller, useForm } from "react-hook-form";
import { toast } from "sonner";
import * as z from "zod";

const formSchema = z.object({
  name: z
    .string()
    .trim()
    .min(2, { error: "Name must be at least 2 characters long." })
    .max(50, { error: "Name cannot exceed 50 characters." }),
});

type FormValues = z.infer<typeof formSchema>;

export default function AddTourTypeForm() {
  const [addTourType, { isLoading }] = useAddTourTypeMutation();

  const form = useForm<FormValues>({
    resolver: zodResolver(formSchema),
    defaultValues: { name: "" },
  });

  const onSubmit = async (data: FormValues) => {
    try {
      await addTourType({ name: data.name }).unwrap();
      toast.success(`"${data.name}" added`);
      form.reset();
    } catch (err) {
      // Backend-এর error (যেমন "already exists") input-এর নিচেই দেখাবে
      form.setError("name", {
        type: "server",
        message: getErrorMessage(err, "Couldn't add the tour type. Please try again."),
      });
    }
  };

  return (
    <section className="rounded-xl border border-border bg-card shadow-card lg:sticky lg:top-4">
      <div className="border-b border-border px-4.5 py-4">
        <h2 className="text-base font-semibold text-foreground">Add a tour type</h2>
        <p className="text-[13.5px] text-muted-foreground">Give it a short, clear name.</p>
      </div>

      <form className="flex flex-col gap-4 p-4.5" onSubmit={form.handleSubmit(onSubmit)} noValidate>
        <Controller
          name="name"
          control={form.control}
          render={({ field, fieldState }) => (
            <Field data-invalid={fieldState.invalid} className="gap-1.5">
              <FieldLabel htmlFor="tt-new" className="text-sm font-semibold">
                Name
              </FieldLabel>
              <Input
                {...field}
                id="tt-new"
                placeholder="e.g. Short tour"
                autoComplete="off"
                aria-invalid={fieldState.invalid}
                className="h-11 rounded-lg px-3.5 text-[15px]"
              />
              {fieldState.invalid && (
                <FieldError className="flex items-center gap-1.5 text-[13px]">
                  <AlertCircle className="size-3.75 shrink-0" />
                  {fieldState.error?.message}
                </FieldError>
              )}
            </Field>
          )}
        />

        <Button
          type="submit"
          disabled={isLoading}
          className="h-11 w-full gap-2 rounded-lg text-[14.5px] font-semibold"
        >
          <Plus className="size-4.25" strokeWidth={2.4} />
          {isLoading ? "Adding…" : "Add tour type"}
        </Button>
      </form>
    </section>
  );
}
```

### `src/pages/Admin/AddTourType.tsx`

```tsx
import AddTourTypeForm from "@/components/modules/Admin/TourType/AddTourTypeForm";
import { Button } from "@/components/ui/button";
import { Skeleton } from "@/components/ui/skeleton";
import { useGetTourTypesQuery } from "@/redux/features/Tour/tour.api";
import { Pencil, Tag, Trash2 } from "lucide-react";

export default function AddTourType() {
  const { data: tourTypes = [], isLoading, isError } = useGetTourTypesQuery();

  const countText = isLoading
    ? "Loading…"
    : `${tourTypes.length} ${tourTypes.length === 1 ? "type" : "types"}`;

  return (
    <div className="flex w-full max-w-6xl flex-col gap-5">
      {/* Page heading */}
      <div>
        <h1 className="text-3xl font-bold tracking-tight text-foreground">Tour types</h1>
        <p className="text-muted-foreground">Used to filter tours and shown as a badge on each card.</p>
      </div>

      <div className="grid grid-cols-1 items-start gap-5 lg:grid-cols-[360px_1fr]">
        {/* ───── Left: Add form ───── */}
        <AddTourTypeForm />

        {/* ───── Right: Tour types list ───── */}
        <section className="overflow-hidden rounded-xl border border-border bg-card shadow-card">
          <div className="flex items-center justify-between gap-3 px-4.5 py-4">
            <div>
              <h2 className="text-base font-semibold text-foreground">All tour types</h2>
              <p className="text-[13.5px] text-muted-foreground">{countText}</p>
            </div>
          </div>

          <ul className="list-none">
            {/* ১. লোড হচ্ছে */}
            {isLoading &&
              Array.from({ length: 3 }).map((_, i) => (
                <li key={i} className="flex items-center gap-3.5 border-t border-border px-4.5 py-3.5">
                  <Skeleton className="size-10 rounded-lg" />
                  <Skeleton className="h-4 flex-1" />
                </li>
              ))}

            {/* ২. Error */}
            {isError && (
              <li className="border-t border-border px-4.5 py-8 text-center text-sm text-destructive">
                Couldn't load tour types. Please refresh the page.
              </li>
            )}

            {/* ৩. খালি */}
            {!isLoading && !isError && tourTypes.length === 0 && (
              <li className="border-t border-border px-4.5 py-8 text-center text-sm text-muted-foreground">
                No tour types yet — add your first one.
              </li>
            )}

            {/* ৪. আসল তালিকা */}
            {tourTypes.map((item) => (
              <li key={item._id} className="flex items-center gap-3.5 border-t border-border px-4.5 py-3.5">
                <span className="flex size-10 shrink-0 items-center justify-center rounded-lg bg-accent text-accent-foreground">
                  <Tag className="size-4.5" />
                </span>

                <div className="min-w-0 flex-1">
                  <strong className="block truncate text-foreground">{item.name}</strong>
                </div>

                {/* Edit/Delete-এর endpoint এলে চালু হবে */}
                <Button
                  variant="outline"
                  size="icon"
                  type="button"
                  disabled
                  title="Coming soon"
                  aria-label={`Rename ${item.name}`}
                  className="size-10 rounded-lg disabled:opacity-40"
                >
                  <Pencil className="size-4.25" />
                </Button>
                <Button
                  variant="outline"
                  size="icon"
                  type="button"
                  disabled
                  title="Coming soon"
                  aria-label={`Delete ${item.name}`}
                  className="size-10 rounded-lg disabled:opacity-40"
                >
                  <Trash2 className="size-4.25" />
                </Button>
              </li>
            ))}
          </ul>

          <p className="border-t border-border bg-muted px-4.5 py-3 text-[13.5px] text-muted-foreground">
            A type that's still used by tours can't be deleted. Reassign those tours first.
          </p>
        </section>
      </div>
    </div>
  );
}
```

💡 **`const { data: tourTypes = [] } = …`:** দুটো কাজ একসাথে: `data`-কে `tourTypes` নামে নেওয়া (পড়তে পরিষ্কার), আর data না আসা পর্যন্ত (`undefined`) default হিসেবে খালি array `[]`। তাই নিচে `tourTypes.length`, `tourTypes.map` লিখতে `?.` লাগে না, page-ও ভাঙে না।

💡 **`Array.from({ length: 3 })`:** ৩ ঘরের একটা ফাঁকা array বানায়, যাতে `map` দিয়ে ৩টা skeleton row আঁকা যায়। Skeleton-এর জন্য `key={i}` (index) ঠিক আছে, কারণ এগুলো কখনো সরে না বা ক্রম বদলায় না।

### `src/routes/adminSidebarItems.ts` (শুধু item)

```ts
{ title: "Tour types", url: "/admin/tour-types", component: AddTourType },
```

---

# Check-list

**ধাপে ধাপে**
- [ ] Postman-এ `POST /tour/create-tour-type` আর `GET /tour/tour-types` কাজ করে
- [ ] Sidebar-এর "Tour types" চাপলে page খোলে
- [ ] তালিকায় database-এর আসল tour type-গুলো দেখায়
- [ ] নতুন নাম যোগ করলে reload ছাড়াই তালিকায় আসে

**উন্নতির পরে**
- [ ] আগে থেকে থাকা নাম দিলে input-এর নিচে লাল লেখায় backend-এর message; console-এ *Uncaught* error নেই
- [ ] সফল হলে toast-এ নাম, input খালি হয়
- [ ] Submit চলার সময় button-এ "Adding…", দুইবার চাপা যায় না
- [ ] Slow 4G-তে তালিকায় ৩টা skeleton row, তারপর আসল তালিকা
- [ ] Backend বন্ধ থাকলে "Couldn't load tour types"
- [ ] উপরের সংখ্যা আসল ("1 type" / "5 types")
- [ ] Edit/Delete disabled, hover-এ "Coming soon"
- [ ] `item.name` লিখলে editor suggestion দেয় (type কাজ করছে)
- [ ] Tag-এর নাম `TOUR_TYPE` (`baseApi` আর `tour.api.ts` দুই জায়গাতেই)
- [ ] Dark mode-এ form, তালিকা, error, skeleton সব পড়া যায়
