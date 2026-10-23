# Frontend Notes — ২২: Tour Details Page — Slug দিয়ে একটা Tour, Gallery আর Booking Card

২১ নম্বর নোটের card-এ click করলে `/tours/sajek-valley-cloud-escape`-এর মতো URL-এ যায়। এই নোটে সেই page: একটা tour-এর সব তথ্য, ছবির gallery, দিনভিত্তিক plan, আর ডান পাশে booking card (অতিথির সংখ্যা, মোট দাম, "Book now")। Book now চাপলে কী হয়, সেটা ২৩ নম্বর নোটে।

প্রতিটা ধাপে 🎯 লক্ষ্য, 🛠️ কাজ, 💡 কেন, ✅ যাচাই। শেষে সমস্যা, উন্নতি আর চূড়ান্ত কোড।

> নোট: আপনার project-এর বর্তমান কোড পড়ে লেখা (React 19, React Router 8, RTK 2)। পুরনো ধরনের কোড দিয়ে কিছু বদলানো হয়নি।

---

## ধাপের মানচিত্র

| ধাপ | বিষয় |
|---|---|
| ১ | Dynamic route `tours/:slug` আর `useParams` |
| ২ | Backend: slug দিয়ে খোঁজা + `populate` |
| ৩ | `getSingleTour` endpoint আর `skip` |
| ৪ | Loading skeleton, আর hook-এর নিয়ম (early return-এর আগে সব hook) |
| ৫ | "পাওয়া যায়নি" বনাম "load হয়নি" |
| ৬ | যে তথ্য আছে শুধু সেটাই দেখানো (`facts`) |
| ৭ | ছবির gallery (১, ২, ৩–৫টা ছবি) |
| ৮ | Tour plan-এর timeline আর তালিকার card |
| ৯ | Booking card: অতিথি +/−, মোট দাম |
| ১০ | Share button |

---

## ধাপ ১ — Dynamic route আর `useParams`

🎯 **লক্ষ্য:** প্রতিটা tour-এর জন্য আলাদা route না লিখে একটা route সব tour সামলাবে।

🛠️ **কাজ:**

```tsx
// src/routes/index.tsx  (App-এর children-এ)
{ Component: TourDetails, path: "tours/:slug" },
```

```tsx
// src/pages/TourDetails.tsx
import { useParams } from "react-router";

const { slug } = useParams();   // /tours/sajek-valley → slug = "sajek-valley"
```

💡 **`:slug` কী:** `:` দিয়ে শুরু হওয়া অংশ হলো **ফাঁকা জায়গা** (URL parameter)। `/tours/` এর পরে যা-ই থাকুক, এই route মিলবে, আর সেই অংশটা `slug` নামে পাওয়া যাবে। নামটা আমাদের দেওয়া: `:slug` লিখলে `useParams().slug`, `:id` লিখলে `useParams().id`।

```
URL                                   useParams()
/tours/sajek-valley-cloud-escape  →   { slug: "sajek-valley-cloud-escape" }
/tours/srimangal-tea-trails       →   { slug: "srimangal-tea-trails" }
/tours                            →   মেলে না (এটা "tours" route-এর, ২১ নম্বর নোট)
```

💡 **id না, slug কেন:** `/tours/6650f1a2b3c4…`-এর চেয়ে `/tours/sajek-valley-cloud-escape` পড়তে আর share করতে সুন্দর, Google-এর মতো search engine-ও URL-এর শব্দ দেখে বোঝে page কী নিয়ে। ১৯–২০ নম্বর নোটে title থেকে slug বানানোর কারণ এটাই।

---

## ধাপ ২ — Backend: slug দিয়ে খোঁজা + `populate`

🎯 **লক্ষ্য:** একটা tour-এর সাথে division আর tour type-এর **নাম**ও পাওয়া।

> এই ধাপ **backend project**-এর কোড।

```ts
// backend: tour.service.ts
const getSingleTour = async (slug: string) => {
  const tour = await Tour.findOne({ slug }).populate("division", "name slug").populate("tourType", "name");
  if (!tour) {
    throw new AppError(404, "Tour not found.");
  }
  return tour;
};
```

💡 **কী ঠিক করা হলো:** আগে slug দিয়ে খুঁজত, কিন্তু `populate` ছিল না। ফলে `division` আসত শুধু id হিসেবে (`"6650f1…"`), আর page-এ division-এর নাম দেখানো যেত না। ২০ নম্বর নোটের তালিকার API-তে যে `populate` দেওয়া হয়েছিল, একক tour-এর API-তেও সেটা লাগে। একই আকারের data দুই API থেকে আসা উচিত, নাহলে frontend-এর একই `ITour` type দুই জায়গায় দুই রকম হয়ে যায়।

💡 **`findOne({ slug })`:** id-র বদলে অন্য field দিয়ে একটা document খোঁজা। Slug unique হতে হবে (model-এ `unique: true`), নাহলে দুটো tour-এর একই slug হলে প্রথমটাই আসবে।

💡 **না পেলে 404:** ভুল slug দিলে `null` আসে; সেটাকে স্পষ্ট "not found" (404) বানিয়ে পাঠানো, যাতে frontend বুঝতে পারে "নেই" আর "server-এ সমস্যা"-র পার্থক্য (ধাপ ৫)।

---

## ধাপ ৩ — `getSingleTour` endpoint আর `skip`

🎯 **লক্ষ্য:** Slug দিয়ে একটা tour আনার hook।

🛠️ **কাজ:**

```ts
// src/redux/features/Tour/tour.api.ts
getSingleTour: builder.query<ITour, string>({
  query: (slug) => ({
    url: `/tour/${slug}`,
    method: "GET",
  }),
  providesTags: ["TOUR"],
  transformResponse: (response: IResponse<ITour>) => response.data,
}),
```

💡 **`<ITour, string>`:** ফলাফল একটা `ITour` (array না), আর argument একটা `string` (slug)। ২১ নম্বর নোটের ধাপ ৪-এর নিয়ম: param নিলে নির্দিষ্ট type।

💡 **`providesTags: ["TOUR"]`:** admin কোনো tour বদলালে বা মুছলে (`invalidatesTags: ["TOUR"]`) এই page-এর data-ও আবার আসে।

```tsx
// page-এ
const { data: tour, isLoading, isError, error, refetch } = useGetSingleTourQuery(slug as string, {
  skip: !slug,
});
```

💡 **`skip: !slug`:** `useParams()`-এর `slug`-এর type `string | undefined` (TypeScript জানে না route-এ সবসময় slug থাকবে)। `skip: true` হলে hook request **পাঠায়ই না**। তাই slug না থাকলে `/tour/undefined`-এ অকারণ request যায় না।

💡 **`slug as string`:** hook-এর argument `string` চায়, কিন্তু `slug` হলো `string | undefined`। `skip` থাকায় `undefined` থাকলে আসলে ব্যবহারই হবে না, কিন্তু TypeScript সেটা জানে না, তাই `as string` দিয়ে জোর। (এর চেয়ে পরিষ্কার উপায় উন্নতি ১-এ।)

---

## ধাপ ৪ — Loading skeleton, আর hook-এর নিয়ম

🎯 **লক্ষ্য:** Data আসার সময় page-এর আকারের skeleton, আর React-এর একটা গুরুত্বপূর্ণ নিয়ম মানা।

🛠️ **কাজ:**

```tsx
export default function TourDetails() {
  const { slug } = useParams();
  const { data: tour, isLoading, isError, error, refetch } = useGetSingleTourQuery(slug as string, { skip: !slug });
  const navigate = useNavigate();
  const location = useLocation();
  const { data: userData } = useUserInfoQuery(undefined);
  const [createBooking, { isLoading: isBooking }] = useCreateBookingMutation();
  const [guests, setGuests] = useState(1);

  /* ── Loading ── */
  if (isLoading) return <TourDetailsSkeleton />;

  /* ── Not found / error ── */
  if (isError || !tour) { return ... }

  // ...এর নিচে tour নিশ্চিতভাবে আছে
}
```

### 💡 Hook-এর নিয়ম: সব hook early return-এর **আগে**

React একটা কঠিন নিয়ম মানে: **প্রতিটা render-এ একই hook, একই ক্রমে** ডাকতে হবে। React hook-গুলোকে নাম দিয়ে না, **ক্রম** দিয়ে চেনে (১ম `useState`, ২য় `useQuery` …)।

```tsx
// ❌ ভুল
if (isLoading) return <TourDetailsSkeleton />;
const [guests, setGuests] = useState(1);   // loading-এর সময় এই লাইনে পৌঁছায়ই না
```

প্রথম render-এ (loading) `useState` চলল না, দ্বিতীয় render-এ (data এসেছে) চলল। React-এর হিসাবে hook-এর সংখ্যা বদলে গেল, আর error দেয়: *"Rendered more hooks than during the previous render."*

আপনার কোডে সব hook (`useParams` থেকে `useState(1)` পর্যন্ত) প্রথম `return`-এর আগে, তাই ঠিক আছে ✅। নিয়মটা মনে রাখার উপায়: **component-এর শুরুতে সব hook, তারপর `if … return`, তারপর বাকি হিসাব।** ১৩ নম্বর নোটের Verify page-এও একই কারণে `if (!email) return null` সব hook-এর পরে ছিল।

### 💡 Skeleton-কে আসল page-এর আকারে

`TourDetailsSkeleton` একই layout-এ: breadcrumb-এর দাগ, শিরোনামের দাগ, ৫ ঘরের gallery, বাম পাশে লেখার অংশ, ডান পাশে booking card। ১৫ নম্বর নোটের নিয়ম: skeleton আসল আকারে হলে data আসার সময় কিছু লাফায় না, আর মনে হয় page প্রায় এসে গেছে।

💡 **একই ফাইলে ছোট component:** `TourDetailsSkeleton`, `Fact`, `ListCard` শুধু এই page-এ লাগে, তাই এই ফাইলেই (২০ নম্বর নোটে teacher-এর reusable-এর কথা)।

---

## ধাপ ৫ — "পাওয়া যায়নি" বনাম "load হয়নি"

🎯 **লক্ষ্য:** দুই রকম ব্যর্থতায় দুই রকম বার্তা।

```tsx
if (isError || !tour) {
  const notFound = (error as TApiError | undefined)?.status === 404 || (!isError && !tour);
  return (
    // icon + notFound ? "Tour not found" : "Couldn't load this tour"
    // notFound না হলে "Try again" button; সবসময় "Browse all tours"
  );
}
```

| অবস্থা | কেন | বার্তা | Button |
|---|---|---|---|
| **404** | ভুল link, বা tour মুছে ফেলা হয়েছে | "Tour not found" | শুধু "Browse all tours" (আবার চেষ্টা করে লাভ নেই) |
| **অন্য error** | internet নেই, server বন্ধ | "Couldn't load this tour" | "Try again" + "Browse all tours" |

💡 **`error.status`:** `axiosBaseQuery` (১৩ নম্বর নোট) error-কে `{ status, data }` আকারে দেয়, তাই status দেখে বোঝা যায় কী হয়েছে।

💡 **`|| !tour`:** error নেই কিন্তু data-ও নেই (যেমন `skip` হয়েছে), সেটাও "not found" ধরা। নিচের কোডে `tour` আছে ধরে নিয়ে লেখা যায় (TypeScript-ও এই `if`-এর পরে জানে `tour` আছে)।

---

## ধাপ ৬ — যে তথ্য আছে শুধু সেটাই: `facts`

🎯 **লক্ষ্য:** খালি তথ্যের জন্য "—" ভরা ঘর না দেখিয়ে, শুধু যেটা আছে সেটা।

🛠️ **কাজ:**

```tsx
const hasDates = Boolean(tour.startDate && tour.endDate);
const days = hasDates ? getDays(tour.startDate!, tour.endDate!) : 0;

const facts: { icon: ReactNode; iconCls: string; label: string; value: string }[] = [];

if (tour.departureLocation || tour.arrivalLocation)
  facts.push({ icon: <Route />, iconCls: "bg-accent text-primary", label: "Departure → Arrival", value: "..." });

if (hasDates)
  facts.push(
    { icon: <CalendarDays />, iconCls: "bg-pink-soft text-pink", label: "Start → End date", value: "..." },
    { icon: <Clock />, iconCls: "bg-yellow-soft text-warning", label: "Duration",
      value: days === 1 ? "1 day" : `${days} days · ${days - 1} ${days - 1 === 1 ? "night" : "nights"}` },
  );

if (tour.maxGuest) facts.push({ ... });
if (tour.minAge !== undefined) facts.push({ ... });
if (tour.costFrom !== undefined) facts.push({ ... });
```

```tsx
{facts.length > 0 && (
  <dl className="grid ...">
    {facts.map((fact) => <Fact key={fact.label} {...fact} />)}
  </dl>
)}
```

💡 **কেন এভাবে:** admin অনেক field ঐচ্ছিক রেখেছেন (২০ নম্বর নোট)। প্রতিটা ঘর JSX-এ সরাসরি লিখলে, যে tour-এ বয়স দেওয়া নেই সেখানে "Minimum age: — years" দেখাত। Array-তে শুধু থাকা তথ্য `push` করে তারপর `map`: খালি ঘর কখনো আসে না, আর সব খালি হলে পুরো বাক্সটাই না (`facts.length > 0`)।

💡 **`tour.startDate!` (non-null assertion):** `startDate`-এর type `string | undefined`। `hasDates` সত্য মানে দুটোই আছে, কিন্তু TypeScript `hasDates` variable দেখে সেটা বুঝতে পারে না। `!` দিয়ে বলা "আমি নিশ্চিত, এটা `undefined` না"। শুধু তখনই ব্যবহার করবেন, যখন সত্যিই আগে যাচাই করা আছে; নাহলে TypeScript-এর পাহারা অকারণে বন্ধ হয়।

💡 **রাত গোনা:** ৩ দিনের tour = ২ রাত (`days - 1`); ১ রাত হলে "night", বেশি হলে "nights"।

💡 **`minAge !== undefined`, `if (tour.minAge)` না:** বয়সের নিয়ম `0` হতে পারে ("যেকোনো বয়স")। `if (0)` মিথ্যা ধরে, তাই `0` দেখাত না। `!== undefined` দিয়ে "দেওয়া আছে কিনা" দেখা, মান যা-ই হোক।

💡 **`<dl>`, `<dt>`, `<dd>`:** HTML-এর "description list": নাম (`dt`, যেমন "Duration") আর মান (`dd`, "3 days")। Screen reader জোড়া হিসেবে পড়ে।

💡 **রঙিন icon-এর ঘর:** `bg-pink-soft text-pink`, `bg-yellow-soft text-warning`, `bg-mint-soft text-success`: সব আপনার `index.css`-এর semantic token, dark mode-এও মানানসই।

---

## ধাপ ৭ — ছবির gallery

🎯 **লক্ষ্য:** ছবির সংখ্যা অনুযায়ী সুন্দর বিন্যাস।

```tsx
{images.length <= 1 ? (
  // ০ বা ১টা ছবি: একটা বড় ছবি (না থাকলে icon)
  <div className="flex h-64 ... sm:h-[454px]">
    {images[0] ? <img src={images[0]} alt={tour.title} className="size-full object-cover" /> : <ImageIcon />}
  </div>
) : (
  // ২–৫টা: grid, প্রথমটা বড়
  <div className="grid grid-cols-1 gap-3.5 sm:grid-cols-[2fr_1fr_1fr] sm:grid-rows-[220px_220px]">
    {images.slice(0, 5).map((src, i) => (
      <div
        key={src}
        className={[
          "overflow-hidden rounded-2xl bg-accent",
          i === 0 ? "h-64 sm:row-span-2 sm:h-auto" : "hidden sm:block",
          images.length === 2 && i === 1 ? "sm:col-span-2 sm:row-span-2" : "",
        ].join(" ")}
      >
        <img src={src} alt={`${tour.title} photo ${i + 1}`} loading={i === 0 ? "eager" : "lazy"} className="size-full object-cover" />
      </div>
    ))}
  </div>
)}
```

**বিন্যাস (desktop):**

```
৫টা ছবি                          ২টা ছবি
┌──────────┬─────┬─────┐         ┌──────────┬───────────┐
│          │  2  │  3  │         │          │           │
│    1     ├─────┼─────┤         │    1     │     2     │
│          │  4  │  5  │         │          │           │
└──────────┴─────┴─────┘         └──────────┴───────────┘
```

💡 **লাইনগুলোর মানে:**
- **`grid-cols-[2fr_1fr_1fr]`:** তিন column: প্রথমটা বাকি দুটোর দ্বিগুণ চওড়া।
- **`grid-rows-[220px_220px]`:** দুই সারি, প্রতিটা ২২০px।
- **প্রথম ছবি `row-span-2`:** দুই সারি জুড়ে, তাই বড়।
- **২টা ছবি হলে দ্বিতীয়টা `col-span-2 row-span-2`:** নাহলে ডান পাশে একটা ছোট ঘর আর বাকি তিনটা ফাঁকা থাকত।
- **Mobile-এ শুধু প্রথম ছবি** (`hidden sm:block` বাকিগুলোয়): ছোট screen-এ ৫টা ছবি স্তূপ করলে অনেক নিচে নামতে হতো।
- **`slice(0, 5)`:** সর্বোচ্চ ৫টা দেখায় (১০টা পর্যন্ত upload করা যায়, ২০ নম্বর নোট)।
- **প্রথমটা `eager`, বাকিগুলো `lazy`:** প্রথম ছবি page খুললেই চোখে পড়ে (১৫ নম্বর নোট: প্রথমে দেখা ছবিতে lazy না)।
- **`alt`-এ নম্বর:** প্রতিটা ছবির আলাদা বর্ণনা (`photo 2`, `photo 3`), screen reader-এর জন্য।

---

## ধাপ ৮ — Tour plan-এর timeline আর তালিকার card

🎯 **লক্ষ্য:** দিনভিত্তিক plan পরপর, সংযোগ রেখাসহ।

```tsx
<ol className="flex flex-col">
  {tour.tourPlan.map((step, i) => (
    <li key={`${i}-${step}`} className="flex gap-4.5">
      <div className="flex flex-col items-center">
        <span className="flex size-11 ... rounded-xl bg-primary ... text-primary-foreground">{i + 1}</span>
        {i < tour.tourPlan!.length - 1 && <span className="my-1.5 min-h-6 w-0.5 flex-1 bg-border" />}
      </div>
      <div className="mb-4.5 flex-1 rounded-xl bg-card ...">
        <span className="text-[12.5px] font-semibold text-primary">Day {i + 1}</span>
        <p className="text-sm text-muted-foreground">{step}</p>
      </div>
    </li>
  ))}
</ol>
```

💡 **লাইনগুলোর মানে:**
- **`<ol>` (ordered list):** ক্রম জরুরি, তাই নম্বরযুক্ত তালিকা; screen reader "list, 3 items" পড়ে।
- **সংযোগ রেখা:** নম্বরের নিচে একটা সরু (`w-0.5`) ধূসর দাগ, `flex-1` দিয়ে card-এর উচ্চতা পর্যন্ত লম্বা। **শেষ দিনে না** (`i < length - 1`), কারণ তার পরে কিছু নেই।
- **`key={`${i}-${step}`}`:** দুই দিনের লেখা হুবহু এক হতে পারে ("Free day"), তাই শুধু `step` দিলে key মিলে যেত; index জুড়ে অনন্য।

> নোট: Plan-এর প্রতিটা item-কে "Day 1, Day 2" ধরা হচ্ছে। Admin যদি এক দিনে দুটো step লেখেন, তখন "Day 3" আসলে দিন ২-এর দ্বিতীয় অংশ হবে। এজন্যই ২০ নম্বর নোটে বলা হয়েছিল, plan-কে পরে `{ day, title, description }` বানালে `useFieldArray` কাজে লাগবে।

**Included / Not included / Amenities:** একই `ListCard`, শুধু icon আলাদা: ✓ (সবুজ, `text-success`), ✕ (লাল, `text-destructive`), ✨ (`text-primary`)। তিনটার কোনোটা না থাকলে পুরো section-ই না।

---

## ধাপ ৯ — Booking card: অতিথি আর মোট দাম

🎯 **লক্ষ্য:** কতজন যাবে বাছাই, সাথে সাথে মোট দাম।

🛠️ **কাজ:**

```tsx
const DEFAULT_MAX_GUESTS = 10; // tour-এ maxGuest না থাকলে

const [guests, setGuests] = useState(1);
const maxGuests = tour.maxGuest ?? DEFAULT_MAX_GUESTS;
const price = tour.costFrom ?? 0;
const total = price * guests;
```

```tsx
<div role="group" aria-labelledby="guests-label" className="flex items-center gap-1.5">
  <Button onClick={() => setGuests((g) => Math.max(1, g - 1))} disabled={guests <= 1} aria-label="Remove a guest">
    <Minus />
  </Button>
  <output aria-live="polite">{guests}</output>
  <Button onClick={() => setGuests((g) => Math.min(maxGuests, g + 1))} disabled={guests >= maxGuests} aria-label="Add a guest">
    <Plus />
  </Button>
</div>
```

💡 **লাইনগুলোর মানে:**
- **`Math.max(1, g - 1)` / `Math.min(maxGuests, g + 1)`:** সীমার বাইরে যেতে দেয় না (১-এর কম, maxGuest-এর বেশি না)। Button `disabled` থাকলেও এই হিসাব রাখা নিরাপদ।
- **`role="group"` + `aria-labelledby`:** তিনটা জিনিস (−, সংখ্যা, +) মিলে একটা নিয়ন্ত্রণ, নাম "Guests"।
- **`<output aria-live="polite">`:** সংখ্যা বদলালে screen reader নিজে থেকে নতুন সংখ্যা পড়ে, কাজ শেষ হওয়ার অপেক্ষা করে (polite)।
- **`aria-label="Add a guest"`:** শুধু + icon-এর button, তাই নাম লাগে।
- **মোট দাম:** `৳7,500 × 3 guests = ৳22,500`; দাম না থাকলে (`costFrom === undefined`) এই অংশ দেখায় না, আর "Price on request"।

💡 **`lg:sticky lg:top-6`:** desktop-এ লম্বা বর্ণনা scroll করলেও booking card পাশে আটকে থাকে, যেকোনো জায়গা থেকে Book করা যায় (২০ নম্বর নোটের sticky bar-এর একই ধারণা)।

> ⚠️ জরুরি নীতি: এখানের মোট দাম **শুধু দেখানোর জন্য**। আসল টাকার হিসাব backend নিজে করবে (tour-এর দাম × guestCount), frontend-এর পাঠানো দাম বিশ্বাস করবে না, কারণ browser-এ যে কেউ কোড বদলে দাম কমাতে পারে। এজন্যই booking-এ আমরা শুধু `{ tour, guestCount }` পাঠাই, দাম না (২৩ নম্বর নোট)।

---

## ধাপ ১০ — Share button

🎯 **লক্ষ্য:** মোবাইলে phone-এর নিজের share menu, desktop-এ link কপি।

```tsx
const handleShare = async () => {
  const url = window.location.href;
  try {
    if (navigator.share) {
      await navigator.share({ title: tour.title, url });   // মোবাইল: WhatsApp, Messenger…
    } else {
      await navigator.clipboard.writeText(url);           // desktop: কপি
      toast.success("Link copied to clipboard");
    }
  } catch {
    // user share-এর জানালা বন্ধ করেছে — কিছু করার নেই
  }
};
```

💡 **`navigator.share`:** browser-এর Web Share API: মোবাইলে (আর কিছু desktop-এ) phone-এর নিজের share menu খোলে। সব browser-এ নেই, তাই `if (navigator.share)` দিয়ে আগে দেখা, না থাকলে clipboard-এ কপি।

💡 **খালি `catch`:** user share menu খুলে কিছু না বেছে বন্ধ করলে `navigator.share` error দেয় (`AbortError`)। এটা আসলে ভুল না, তাই চুপচাপ উপেক্ষা।

✅ **যাচাই (পুরো page):**
- Tours-এর card থেকে details খোলে; URL-এ slug
- ভুল slug (`/tours/abc`) → "Tour not found", "Try again" নেই
- Backend বন্ধ → "Couldn't load this tour" + "Try again"
- ২টা ছবির tour-এ দুই ভাগ, ৫টার বেশিতে প্রথম ৫টা
- যে tour-এ বয়স নেই, সেখানে "Minimum age" ঘর নেই
- Guests ১-এর কম বা maxGuest-এর বেশি হয় না; মোট দাম সাথে সাথে বদলায়
- মোবাইলে Share চাপলে phone-এর share menu; desktop-এ "Link copied"

---

# কী বাকি: সমস্যা চেনা

| # | সমস্যা |
|---|---|
| ১ | `slug as string` + `skip`: কাজ করে, কিন্তু `as` দিয়ে জোর করা; RTK-র `skipToken` দিয়ে পরিষ্কার উপায় আছে। |
| ২ | Browser-এর tab-এ page-এর নাম সব tour-এ একই (Vite-এর `index.html`-এর `<title>`)। Share বা bookmark করলে tour-এর নাম দেখায় না। |
| ৩ | Tour plan-এর নম্বরের ছায়ায় hard-coded রং: `shadow-[0_8px_18px_rgba(91,76,240,.25)]` (আমাদের নিয়ম: রং token থেকে)। Primary রং কখনো বদলালে ছায়া পুরনো রঙে থেকে যাবে। |
| ৪ | Gallery-র class `[...].join(" ")` দিয়ে; `cn()` না। |
| ৫ | `<h1>`, `<h2>`, `<h3>`-এ `text-foreground` নেই। |
| ৬ | Book now button-এর উপরে পুরনো comment `{/* TODO: connect to booking … */}` রয়ে গেছে, অথচ ইতিমধ্যে যুক্ত। |
| ৭ | ৫টার বেশি ছবি থাকলে বাকিগুলো দেখার কোনো উপায় নেই। |

---

# উন্নতি

## উন্নতি ১ — `skipToken`: `as` ছাড়াই skip

```tsx
import { skipToken } from "@reduxjs/toolkit/query/react";

const { data: tour, isLoading, isError, error, refetch } = useGetSingleTourQuery(slug ?? skipToken);
```

💡 **`skipToken`:** RTK Query-র একটা বিশেষ মান। Argument-এর জায়গায় এটা দিলে hook skip করে। `slug ?? skipToken` মানে: slug থাকলে সেটা দাও, না থাকলে skip। TypeScript খুশি (argument হয় `string`, নয় `skipToken`), `as string` আর আলাদা `skip` option লাগে না।

## উন্নতি ২ — Tab-এ tour-এর নাম (React 19)

React 19-এ component-এর ভেতরে সরাসরি `<title>` লিখলে React নিজে সেটা `<head>`-এ বসিয়ে দেয়, আলাদা library লাগে না:

```tsx
return (
  <div className="bg-background">
    <title>{`${tour.title} · Phovia Tour`}</title>
    <meta name="description" content={tour.description?.slice(0, 150) ?? tour.title} />
    {/* ...বাকি page */}
  </div>
);
```

Browser-এর tab-এ "Sajek Valley Cloud Escape · Phovia Tour", আর bookmark-এও একই নাম। (Template literal দিয়ে একটা string বানিয়ে দেওয়া, কারণ `<title>`-এর ভেতরে একাধিক টুকরো দিলে React সতর্ক করে।)

> নোট: Facebook/WhatsApp-এ link share করলে যে preview card দেখায়, সেটা এভাবে কাজ করে না। ওদের bot JavaScript চালায় না, শুধু server-এর পাঠানো প্রথম HTML পড়ে। সেজন্য server-side rendering লাগে, এই course-এর বিষয় না।

## উন্নতি ৩ — ছায়ায় token

```tsx
// আগে
className="... shadow-[0_8px_18px_rgba(91,76,240,.25)]"
// পরে (Navbar-এর Sign up button-এর মতো, ৮ নম্বর নোট)
className="... shadow-lg shadow-primary/25"
```

`shadow-primary/25` = primary রঙে ২৫% স্বচ্ছ ছায়া; primary বদলালে ছায়াও বদলায়, dark mode-এও।

## উন্নতি ৪ — ছোটখাটো

- Gallery-র class: `cn("overflow-hidden rounded-2xl bg-accent", i === 0 ? "..." : "hidden sm:block", images.length === 2 && i === 1 && "sm:col-span-2 sm:row-span-2")`।
- শিরোনামে `text-foreground`।
- পুরনো `TODO` comment মুছুন।
- (ঐচ্ছিক) ৫টার বেশি ছবি থাকলে শেষ ঘরের ওপর "+৩ photos" লিখে, click করলে সব ছবির একটা `Dialog`। পরে Lightbox হিসেবে বানানো যায়।
- (ঐচ্ছিক) route-এ `lazy` (১৫ নম্বর নোট)।

---

# চূড়ান্ত কোড

### `src/pages/TourDetails.tsx` — আপনার বর্তমান কোড

> এর ওপর উন্নতি বসাতে: (১) `useGetSingleTourQuery(slug as string, { skip: !slug })` → `useGetSingleTourQuery(slug ?? skipToken)` আর import-এ `skipToken`; (২) data দেখানোর `return`-এর ঠিক ভেতরে `<title>` আর `<meta>`; (৩) tour plan-এর ছায়া `shadow-lg shadow-primary/25`; (৪) `TODO` comment মোছা। বাকি সব হুবহু। Booking-এর অংশ (`handleBook`) ২৩ নম্বর নোটে।

```tsx
import { Button } from "@/components/ui/button";
import { Skeleton } from "@/components/ui/skeleton";
import { useUserInfoQuery } from "@/redux/features/auth/auth.api";
import { useCreateBookingMutation } from "@/redux/features/Booking/booking.api";
import { useGetSingleTourQuery } from "@/redux/features/Tour/tour.api";
import type { TApiError } from "@/types";
import {
	CalendarDays,
	Check,
	Clock,
	ImageIcon,
	Loader2,
	MapPin,
	MapPinOff,
	Minus,
	Plus,
	RefreshCw,
	Route,
	Share2,
	Sparkles,
	Tag,
	UserRound,
	Users,
	X,
} from "lucide-react";
import { useState, type ReactNode } from "react";
import { Link, useLocation, useNavigate, useParams } from "react-router";
import { toast } from "sonner";

/* ───────── helpers ───────── */

const formatDate = (date: string) =>
	new Date(date).toLocaleDateString("en-GB", { day: "numeric", month: "short", year: "numeric" });

const formatPrice = (amount: number) => `৳${amount.toLocaleString("en-IN")}`;

const getDays = (start: string, end: string) =>
	Math.round((new Date(end).getTime() - new Date(start).getTime()) / (1000 * 60 * 60 * 24)) + 1;

const DEFAULT_MAX_GUESTS = 10; // used when tour has no maxGuest

function Fact({ icon, iconCls, label, value }: { icon: ReactNode; iconCls: string; label: string; value: string }) {
	return (
		<div className="flex items-center gap-3.5">
			<span className={`flex size-12 shrink-0 items-center justify-center rounded-xl ${iconCls}`}>{icon}</span>
			<div>
				<dt className="text-[12.5px] text-muted-foreground">{label}</dt>
				<dd className="font-semibold text-foreground">{value}</dd>
			</div>
		</div>
	);
}

function ListCard({ title, items, icon }: { title: string; items: string[]; icon: ReactNode }) {
	return (
		<div className="flex flex-col gap-3.5 rounded-xl bg-card p-5.5 shadow-card">
			<h3 className="text-[17px] font-semibold">{title}</h3>
			<ul className="flex flex-col gap-2.5">
				{items.map((item) => (
					<li key={item} className="flex items-start gap-2.5 text-sm">
						<span className="mt-0.5 shrink-0">{icon}</span>
						{item}
					</li>
				))}
			</ul>
		</div>
	);
}

function TourDetailsSkeleton() {
	return (
		<div className="mx-auto flex max-w-[1200px] flex-col gap-5.5 px-4 pt-4 pb-24 sm:px-10">
			<Skeleton className="h-4 w-64" />
			<div className="flex flex-col gap-3">
				<Skeleton className="h-6 w-40 rounded-full" />
				<Skeleton className="h-10 w-full max-w-xl" />
				<Skeleton className="h-4 w-72" />
			</div>
			<div className="grid grid-cols-1 gap-3.5 sm:grid-cols-[2fr_1fr_1fr] sm:grid-rows-[220px_220px]">
				<Skeleton className="h-64 rounded-2xl sm:row-span-2 sm:h-auto" />
				{[1, 2, 3, 4].map((i) => (
					<Skeleton key={i} className="hidden rounded-2xl sm:block" />
				))}
			</div>
			<div className="flex flex-col gap-12 pt-6 lg:flex-row">
				<div className="flex flex-1 flex-col gap-6">
					<Skeleton className="h-44 rounded-xl" />
					<Skeleton className="h-6 w-48" />
					<Skeleton className="h-24 rounded-xl" />
				</div>
				<Skeleton className="h-96 rounded-xl lg:w-[370px]" />
			</div>
		</div>
	);
}

/* ───────── page ───────── */

export default function TourDetails() {
	const { slug } = useParams();
	const {
		data: tour,
		isLoading,
		isError,
		error,
		refetch,
	} = useGetSingleTourQuery(slug as string, {
		skip: !slug,
	});

	const navigate = useNavigate();
	const location = useLocation();
	const { data: userData } = useUserInfoQuery(undefined);
	const [createBooking, { isLoading: isBooking }] = useCreateBookingMutation();

	const isLoggedIn = Boolean(userData?.data?.email);

	const [guests, setGuests] = useState(1);

	/* ── Loading ── */
	if (isLoading) return <TourDetailsSkeleton />;

	/* ── Not found / error ── */
	if (isError || !tour) {
		const notFound = (error as TApiError | undefined)?.status === 404 || (!isError && !tour);
		return (
			<div className="mx-auto flex max-w-[1200px] flex-col items-center gap-4 px-4 py-24 text-center sm:px-10">
				<span className="flex size-14 items-center justify-center rounded-full bg-accent text-accent-foreground">
					<MapPinOff className="size-6" />
				</span>
				<div>
					<h1 className="text-2xl font-bold">{notFound ? "Tour not found" : "Couldn't load this tour"}</h1>
					<p className="text-muted-foreground">
						{notFound
							? "This tour may have been removed or the link is wrong."
							: "Check your connection and try again."}
					</p>
				</div>
				<div className="flex gap-2.5">
					{!notFound && (
						<Button variant="outline" onClick={() => refetch()} className="h-11 gap-2 rounded-lg">
							<RefreshCw className="size-4" />
							Try again
						</Button>
					)}
					<Button asChild className="h-11 rounded-lg">
						<Link to="/tours">Browse all tours</Link>
					</Button>
				</div>
			</div>
		);
	}

	/* ── Derived values ── */
	const images = tour.images ?? [];
	const hasDates = Boolean(tour.startDate && tour.endDate);
	const days = hasDates ? getDays(tour.startDate!, tour.endDate!) : 0;
	const maxGuests = tour.maxGuest ?? DEFAULT_MAX_GUESTS;
	const price = tour.costFrom ?? 0;
	const total = price * guests;

	const facts: { icon: ReactNode; iconCls: string; label: string; value: string }[] = [];
	if (tour.departureLocation || tour.arrivalLocation)
		facts.push({
			icon: <Route className="size-5.5" />,
			iconCls: "bg-accent text-primary",
			label: "Departure → Arrival",
			value: `${tour.departureLocation ?? "—"} → ${tour.arrivalLocation ?? "—"}`,
		});
	if (hasDates)
		facts.push(
			{
				icon: <CalendarDays className="size-5.5" />,
				iconCls: "bg-pink-soft text-pink",
				label: "Start → End date",
				value: `${formatDate(tour.startDate!)} → ${formatDate(tour.endDate!)}`,
			},
			{
				icon: <Clock className="size-5.5" />,
				iconCls: "bg-yellow-soft text-warning",
				label: "Duration",
				value: days === 1 ? "1 day" : `${days} days · ${days - 1} ${days - 1 === 1 ? "night" : "nights"}`,
			},
		);
	if (tour.maxGuest)
		facts.push({
			icon: <Users className="size-5.5" />,
			iconCls: "bg-mint-soft text-success",
			label: "Max guests",
			value: `${tour.maxGuest} per departure`,
		});
	if (tour.minAge !== undefined)
		facts.push({
			icon: <UserRound className="size-5.5" />,
			iconCls: "bg-accent text-primary",
			label: "Minimum age",
			value: `${tour.minAge} years`,
		});
	if (tour.costFrom !== undefined)
		facts.push({
			icon: <Tag className="size-5.5" />,
			iconCls: "bg-pink-soft text-pink",
			label: "Price",
			value: `${formatPrice(tour.costFrom)} per person`,
		});

	const handleShare = async () => {
		const url = window.location.href;
		try {
			if (navigator.share) {
				await navigator.share({ title: tour.title, url });
			} else {
				await navigator.clipboard.writeText(url);
				toast.success("Link copied to clipboard");
			}
		} catch {
			// user closed the share sheet — nothing to do
		}
	};

	const handleBook = async () => {
		// not logged in → login, then come back here
		if (!isLoggedIn) {
			toast.info("Please log in to book this tour");
			navigate("/login", { state: { from: location.pathname } });
			return;
		}

		const toastId = toast.loading("Creating your booking…");
		try {
			const res = await createBooking({ tour: tour._id, guestCount: guests }).unwrap();

			const paymentUrl =
				typeof res.paymentUrl === "string"
					? res.paymentUrl
					: (res.paymentUrl as { GatewayPageURL?: string } | undefined)?.GatewayPageURL;

			if (paymentUrl) {
				toast.success("Redirecting to payment…", { id: toastId });
				window.location.href = paymentUrl;
			} else {
				toast.error("Payment link not received. Please try again.", { id: toastId });
			}
		} catch (err) {
			const error = err as TApiError;
			const message = error.data?.message || "Booking failed. Please try again.";
			toast.error(message, { id: toastId });
		}
	};

	return (
		<div className="bg-background">
			{/* ───── Header + gallery ───── */}
			<div className="mx-auto flex max-w-[1200px] flex-col gap-5.5 px-4 pt-4 sm:px-10">
				<nav aria-label="Breadcrumb" className="flex flex-wrap gap-2 text-[13.5px] text-muted-foreground">
					<Link to="/" className="hover:text-foreground">
						Home
					</Link>
					<span aria-hidden="true">/</span>
					<Link to="/tours" className="hover:text-foreground">
						Tours
					</Link>
					{tour.division && (
						<>
							<span aria-hidden="true">/</span>
							<span>{tour.division.name}</span>
						</>
					)}
					<span aria-hidden="true">/</span>
					<span className="font-medium text-primary">{tour.title}</span>
				</nav>

				<div className="flex flex-col gap-4 sm:flex-row sm:items-end sm:justify-between">
					<div className="flex flex-col gap-2.5">
						{(tour.tourType || tour.division) && (
							<div className="flex flex-wrap gap-2">
								{tour.tourType && (
									<span className="rounded-full bg-accent px-3 py-1 text-xs font-semibold text-primary">
										{tour.tourType.name}
									</span>
								)}
								{tour.division && (
									<span className="rounded-full bg-pink-soft px-3 py-1 text-xs font-semibold text-pink">
										{tour.division.name}
									</span>
								)}
							</div>
						)}
						<h1 className="text-3xl leading-tight font-bold tracking-tight sm:text-[42px]">{tour.title}</h1>
						{tour.location && (
							<p className="flex items-center gap-1.5 text-muted-foreground">
								<MapPin className="size-4 shrink-0 text-primary" />
								{tour.location}
							</p>
						)}
					</div>
					<Button
						variant="outline"
						type="button"
						onClick={handleShare}
						className="h-11 gap-2 self-start rounded-lg border-[1.5px] border-primary px-5 text-primary sm:self-auto"
					>
						<Share2 className="size-4" />
						Share
					</Button>
				</div>

				{/* Gallery */}
				{images.length <= 1 ? (
					<div className="flex h-64 items-center justify-center overflow-hidden rounded-2xl bg-accent text-muted-foreground sm:h-[454px]">
						{images[0] ? (
							<img src={images[0]} alt={tour.title} className="size-full object-cover" />
						) : (
							<ImageIcon className="size-9" />
						)}
					</div>
				) : (
					<div className="grid grid-cols-1 gap-3.5 sm:grid-cols-[2fr_1fr_1fr] sm:grid-rows-[220px_220px]">
						{images.slice(0, 5).map((src, i) => (
							<div
								key={src}
								className={[
									"overflow-hidden rounded-2xl bg-accent",
									i === 0 ? "h-64 sm:row-span-2 sm:h-auto" : "hidden sm:block",
									// 2 images → second one spans the rest
									images.length === 2 && i === 1 ? "sm:col-span-2 sm:row-span-2" : "",
								].join(" ")}
							>
								<img
									src={src}
									alt={`${tour.title} photo ${i + 1}`}
									loading={i === 0 ? "eager" : "lazy"}
									className="size-full object-cover"
								/>
							</div>
						))}
					</div>
				)}
			</div>

			{/* ───── Content + booking card ───── */}
			<div className="mx-auto flex max-w-[1200px] flex-col items-start gap-12 px-4 pt-11 pb-24 sm:px-10 lg:flex-row">
				<main className="flex w-full min-w-0 flex-1 flex-col gap-11">
					{/* Facts */}
					{facts.length > 0 && (
						<dl className="grid grid-cols-1 gap-x-7 gap-y-6 rounded-xl bg-card p-6.5 shadow-card sm:grid-cols-2 xl:grid-cols-3">
							{facts.map((fact) => (
								<Fact key={fact.label} {...fact} />
							))}
						</dl>
					)}

					{/* About */}
					{tour.description && (
						<section className="flex flex-col gap-3.5">
							<h2 className="text-2xl font-bold sm:text-[26px]">About this tour</h2>
							<p className="max-w-[70ch] text-[15.5px] leading-relaxed whitespace-pre-line text-muted-foreground">
								{tour.description}
							</p>
						</section>
					)}

					{/* Tour plan */}
					{tour.tourPlan && tour.tourPlan.length > 0 && (
						<section className="flex flex-col gap-5.5">
							<h2 className="text-2xl font-bold sm:text-[26px]">Tour plan</h2>
							<ol className="flex flex-col">
								{tour.tourPlan.map((step, i) => (
									<li key={`${i}-${step}`} className="flex gap-4.5">
										<div className="flex flex-col items-center">
											<span className="flex size-11 shrink-0 items-center justify-center rounded-xl bg-primary font-bold text-primary-foreground shadow-[0_8px_18px_rgba(91,76,240,.25)]">
												{i + 1}
											</span>
											{i < tour.tourPlan!.length - 1 && <span className="my-1.5 min-h-6 w-0.5 flex-1 bg-border" />}
										</div>
										<div className="mb-4.5 flex flex-1 flex-col gap-1.5 rounded-xl bg-card px-5 py-4.5 shadow-card">
											<span className="text-[12.5px] font-semibold text-primary">Day {i + 1}</span>
											<p className="text-sm leading-relaxed text-muted-foreground">{step}</p>
										</div>
									</li>
								))}
							</ol>
						</section>
					)}

					{/* Included / Excluded / Amenities */}
					{Boolean(tour.included?.length || tour.excluded?.length || tour.amenities?.length) && (
						<section className="grid grid-cols-1 gap-4.5 sm:grid-cols-2 xl:grid-cols-3">
							{tour.included && tour.included.length > 0 && (
								<ListCard
									title="Included"
									items={tour.included}
									icon={<Check className="size-4.5 text-success" strokeWidth={2.6} />}
								/>
							)}
							{tour.excluded && tour.excluded.length > 0 && (
								<ListCard
									title="Not included"
									items={tour.excluded}
									icon={<X className="size-4.5 text-destructive" strokeWidth={2.6} />}
								/>
							)}
							{tour.amenities && tour.amenities.length > 0 && (
								<ListCard
									title="Amenities"
									items={tour.amenities}
									icon={<Sparkles className="size-4.5 text-primary" />}
								/>
							)}
						</section>
					)}
				</main>

				{/* Booking card */}
				<aside
					aria-label="Book this tour"
					className="flex w-full flex-col gap-4.5 rounded-xl bg-card p-6.5 shadow-card lg:sticky lg:top-6 lg:w-[370px] lg:shrink-0"
				>
					<div>
						<span className="text-[13px] text-muted-foreground">from</span>
						<div className="flex items-baseline gap-1.5">
							<strong className="text-[32px] font-bold text-primary">
								{tour.costFrom !== undefined ? formatPrice(tour.costFrom) : "Price on request"}
							</strong>
							{tour.costFrom !== undefined && <span className="text-sm text-muted-foreground">/ person</span>}
						</div>
					</div>

					{hasDates && (
						<div className="grid grid-cols-2 gap-2.5">
							<div className="rounded-lg bg-muted px-3.5 py-3">
								<span className="text-xs text-muted-foreground">Start</span>
								<div className="text-sm font-semibold">{formatDate(tour.startDate!)}</div>
							</div>
							<div className="rounded-lg bg-muted px-3.5 py-3">
								<span className="text-xs text-muted-foreground">End</span>
								<div className="text-sm font-semibold">{formatDate(tour.endDate!)}</div>
							</div>
						</div>
					)}

					<div className="flex items-center justify-between gap-3">
						<div>
							<div id="guests-label" className="font-semibold">
								Guests
							</div>
							<div className="text-[12.5px] text-muted-foreground">
								Max {maxGuests}
								{tour.minAge !== undefined && ` · age ${tour.minAge}+`}
							</div>
						</div>
						<div role="group" aria-labelledby="guests-label" className="flex items-center gap-1.5">
							<Button
								variant="outline"
								size="icon"
								type="button"
								onClick={() => setGuests((g) => Math.max(1, g - 1))}
								disabled={guests <= 1}
								aria-label="Remove a guest"
								className="size-11 rounded-lg border-[1.5px]"
							>
								<Minus className="size-4.5" />
							</Button>
							<output aria-live="polite" className="min-w-8.5 text-center text-lg font-bold">
								{guests}
							</output>
							<Button
								variant="outline"
								size="icon"
								type="button"
								onClick={() => setGuests((g) => Math.min(maxGuests, g + 1))}
								disabled={guests >= maxGuests}
								aria-label="Add a guest"
								className="size-11 rounded-lg border-[1.5px] border-primary text-primary"
							>
								<Plus className="size-4.5" />
							</Button>
						</div>
					</div>

					{tour.costFrom !== undefined && (
						<div className="flex flex-col gap-2 border-t border-dashed border-border pt-4">
							<div className="flex justify-between text-sm">
								<span className="text-muted-foreground">
									{formatPrice(price)} × {guests} {guests === 1 ? "guest" : "guests"}
								</span>
								<span>{formatPrice(total)}</span>
							</div>
							<div className="flex items-baseline justify-between pt-1">
								<strong>Total</strong>
								<strong className="text-2xl font-bold">{formatPrice(total)}</strong>
							</div>
						</div>
					)}

					{/* TODO: connect to booking (pass tour._id + guests) */}
					<Button
						type="button"
						onClick={handleBook}
						disabled={isBooking || tour.costFrom === undefined}
						className="h-13 gap-2 rounded-lg text-[15.5px] font-semibold"
					>
						{isBooking ? (
							<>
								<Loader2 className="size-4.5 animate-spin" />
								Processing…
							</>
						) : (
							"Book now"
						)}
					</Button>
					<p className="text-center text-[12.5px] text-muted-foreground">
						You review everything before paying · secure checkout in taka
					</p>
				</aside>
			</div>
		</div>
	);
}
```

### `src/redux/features/Tour/tour.api.ts` — `getSingleTour` অংশ

```ts
getSingleTour: builder.query<ITour, string>({
  query: (slug) => ({
    url: `/tour/${slug}`,
    method: "GET",
  }),
  providesTags: ["TOUR"],
  transformResponse: (response: IResponse<ITour>) => response.data,
}),

// export-এ: useGetSingleTourQuery
```

### Backend — `tour.service.ts`

ধাপ ২-এর `getSingleTour`।

---

# Check-list

- [ ] `/tours/<slug>` খোলে, division আর tour type-এর **নাম** দেখায় (id না)
- [ ] Loading-এর সময় page-এর আকারের skeleton, console-এ hook-এর error নেই
- [ ] ভুল slug → "Tour not found"; backend বন্ধ → "Couldn't load" + Try again
- [ ] খালি তথ্যের কোনো ঘর নেই; `minAge: 0` হলে "0 years" দেখায়
- [ ] Gallery ১, ২, ৩–৫টা ছবিতে ঠিক বিন্যাসে; mobile-এ শুধু প্রথমটা
- [ ] Plan-এর শেষ দিনের নিচে সংযোগ রেখা নেই
- [ ] Guests-এর +/− সীমার ভেতরে; মোট দাম ঠিক
- [ ] Share: মোবাইলে share menu, desktop-এ কপি
- [ ] (উন্নতির পরে) browser tab-এ tour-এর নাম; code-এ `as string` নেই
