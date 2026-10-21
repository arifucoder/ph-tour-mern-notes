# Frontend Notes — ১৫: Performance — Lazy Loading দিয়ে Site দ্রুত করা

এই নোটে একটাই মূল নীতি:

> **যা এখন লাগবে না, তা এখন download করব না।**

Home page-এ থাকা user-কে admin-এর Analytics page-এর কোড পাঠানোর কোনো মানে নেই। Lazy loading দিয়ে প্রতিটা page-এর কোড শুধু তখনই আনব, যখন user সত্যিই সেই page-এ যাবে। শেষ ভাগে lazy loading ছাড়াও site দ্রুত করার আরও কয়েকটা কাজের উপায় আছে (ছবি, font, library), কারণ আপনার আগের site-গুলো slow ছিল, আর সেটার কারণ প্রায়ই শুধু JavaScript না।

আগের নোটগুলোর মতো প্রতিটা ধাপে:

- 🎯 **লক্ষ্য**
- 🛠️ **কাজ**
- 💡 **কেন**
- ✅ **যাচাই**

> নোট: আপনার project-এর version মেনে লেখা: Vite 8, React 19, React Router 8। এখানকার কোনো কোড আপনার বর্তমান কোডকে পুরনো ধরনে বদলাতে বলে না।

---

## ভূমিকা: Bundle কী, আর কেন slow হয়

আমরা অনেকগুলো ফাইলে কোড লিখি (`Home.tsx`, `Analytics.tsx`, `Navbar.tsx` …)। Browser এগুলো সরাসরি চালাতে পারে না (TypeScript, JSX)। তাই `bun run build` দিলে Vite সব ফাইল জোড়া লাগিয়ে, browser-এর বোঝার মতো করে কয়েকটা JavaScript ফাইল বানায়। এই ফাইলগুলোকে বলে **bundle**, আর প্রতিটা আলাদা টুকরোকে বলে **chunk**।

```
আমাদের ফাইল                         build-এর পরে
─────────────                         ─────────────
main.tsx ─┐
Home.tsx  │
Navbar.tsx├──→  Vite (bun run build) ──→  dist/assets/index-a1b2c3.js   ← সব এক ফাইলে
Analytics │                                 (একটা বড় chunk)
recharts ─┘
```

**সমস্যা:** user যে page-এই আসুক, এই পুরো বড় ফাইল download হয়, parse হয়, তারপর page দেখায়। ফাইল যত বড়, প্রথম page দেখাতে তত দেরি। বিশেষ করে মোবাইল internet-এ।

**সমাধান (lazy loading):** page-গুলোকে আলাদা chunk-এ ভাগ করা, আর প্রতিটা chunk শুধু দরকারের সময় আনা:

```
dist/assets/index-a1b2c3.js       ← শুধু Home, Navbar, Router … (ছোট)
dist/assets/Analytics-d4e5f6.js   ← শুধু /admin/analytics খুললে
dist/assets/Login-g7h8i9.js       ← শুধু /login খুললে
```

### ধাপের মানচিত্র

| ভাগ | ধাপ | শেষে কী হবে |
|---|---|---|
| **ক. সমস্যা মাপা** | ১–২ | ঠিক কোথায় কত বড়, সংখ্যা দিয়ে জানব |
| **খ. Dashboard page lazy করা** | ৩–৬ | Home-এ থাকলে admin-এর কোড আসে না, আর লোড হওয়ার সময় সুন্দর skeleton দেখায় |
| **গ. কোন page lazy করব (সেরা পরামর্শ)** | ৭–৯ | Login, Register, Dashboard layout-ও আলাদা chunk-এ |
| **ঘ. Lazy ছাড়াও দ্রুত করা** | ১০–১৩ | ছবি, font, library, আর মাপার নিয়ম |

---

# ভাগ ক: সমস্যা মাপা

## ধাপ ১ — Network tab-এ সমস্যা দেখা

🎯 **লক্ষ্য:** Home-এ থেকেও কেন Analytics-এর কোড আসছে, সেটা নিজের চোখে দেখা।

🛠️ **কাজ:**
1. `bun run dev` চালিয়ে Home page খুলুন।
2. DevTools → **Network** tab → উপরের filter-এ **JS** বেছে নিন।
3. Page reload দিন।
4. Filter box-এ `analytics` লিখে search করুন।

✅ **যাচাই (এবং সমস্যা):** List-এ `Analytics.tsx` পাওয়া যায়, অথচ আমরা Home-এ আছি, admin-এর ধারেকাছেও যাইনি।

### 💡 কেন এমন হয়: import-এর শিকল

`import` হলো একটা শিকল। একটা ফাইল আরেকটাকে import করলে, সেটা না এনে প্রথমটা চলতে পারে না:

```
main.tsx
  └── import { router } from "./routes/index.tsx"
        └── import { adminSidebarItems } from "./adminSidebarItems"
              └── import Analytics from "@/pages/Admin/Analytics"   ← এখানে
                    └── (পরে) import { LineChart } from "recharts"  ← আরও ভারী
```

`main.tsx` চলতেই হবে (app-এর শুরু), তাই শিকল ধরে ধরে **শেষ পর্যন্ত সবকিছু** আসে। Analytics কোনো page-এ দেখানো হচ্ছে কিনা, সেটা `import` দেখে না। উপরে `import` লেখা থাকলেই আনে। একে বলে **static import**।

> ⚠️ সংশোধন (dev বনাম production): Dev mode-এ (`bun run dev`) Vite প্রতিটা ফাইল **আলাদা আলাদা** পাঠায়, তাই Network tab-এ `Analytics.tsx` নামে আলাদা request দেখা যায়। কিন্তু production build-এ (`bun run build`) Analytics আলাদা ফাইল থাকে না, সবকিছু মিলে একটা বড় `index-xxxx.js`-এ ঢুকে যায় (আপনি যেমন লিখেছেন: "build করলে এক ফাইলে build হয়ে যাবে")। তখন search করলে `Analytics` নামে কিছু পাবেন না, কিন্তু তার কোড ঐ বড় ফাইলের ভেতরেই আছে। সমস্যা দুই জায়গাতেই একই, শুধু দেখতে আলাদা। আর **আসল speed মাপতে হয় production build দিয়েই** (পরের ধাপ), কারণ dev mode ইচ্ছা করেই ধীর (দ্রুত কোড বদলানোর সুবিধার জন্য)।

---

## ধাপ ২ — Production build দিয়ে সঠিকভাবে মাপা

🎯 **লক্ষ্য:** আসল user যা download করবে, ঠিক সেটার মাপ জানা। প্রতিটা উন্নতির আগে-পরে এভাবেই মাপব।

🛠️ **কাজ:**

```sh
bun run build
```

Terminal-এ Vite প্রতিটা ফাইলের মাপ দেখায়, অনেকটা এরকম (সংখ্যাগুলো উদাহরণ):

```
dist/index.html                      0.46 kB │ gzip:   0.30 kB
dist/assets/inani-beach-x1y2.jpeg  849.26 kB
dist/assets/index-k3l4.css          48.10 kB │ gzip:   9.20 kB
dist/assets/index-a1b2.js          612.40 kB │ gzip: 190.15 kB
```

| Column | মানে |
|---|---|
| বাম পাশের সংখ্যা | ফাইলের আসল মাপ |
| `gzip` | server চাপ দিয়ে (compress করে) পাঠালে আসলে যতটুকু internet দিয়ে যায়। User-এর download সময়ের জন্য এটাই বেশি গুরুত্বপূর্ণ |

কোনো একটা JS chunk 500 kB-এর বেশি হলে Vite হলুদ রঙে সতর্কও করে ("Some chunks are larger than 500 kB")। এটা দেখলে বুঝবেন ভাগ করার সময় এসেছে।

এরপর build-করা site চালিয়ে দেখুন:

```sh
bun run preview
```

DevTools → Network-এ:
- **Disable cache**-এ টিক দিন (নাহলে আগের download থেকে নিয়ে নেয়, আসল সময় বোঝা যায় না)।
- Throttling-এ **Fast 4G** বা **Slow 4G** বেছে নিন (বাংলাদেশের অনেক user-এর মোবাইল internet এমনই)।
- Reload দিয়ে দেখুন কোন ফাইল কত বড়, কতক্ষণ লাগছে।

💡 **কেন আগে মাপব:** না মেপে উন্নতি করলে বোঝা যায় না কোনটা আসলে কাজে লাগল। মেপে নিলে "আগে ১৯০ kB, এখন ১২০ kB" এমন পরিষ্কার ফল দেখা যায়।

✅ **যাচাই:** `bun run build`-এর output-এর একটা screenshot বা নোট রেখে দিন। ভাগ খ আর গ-এর পরে আবার build দিয়ে তুলনা করবেন।

---

# ভাগ খ: Dashboard page lazy করা

## ধাপ ৩ — `lazy` আর dynamic `import()`

🎯 **লক্ষ্য:** Analytics-কে আলাদা chunk-এ পাঠানো, যেটা শুধু ঐ page খুললে আসবে।

### 💡 দুই রকম import

| | Static import | Dynamic import |
|---|---|---|
| লেখা | `import Analytics from "@/pages/Admin/Analytics";` | `import("@/pages/Admin/Analytics")` |
| কোথায় লেখা হয় | ফাইলের একদম উপরে | যেকোনো জায়গায়, function-এর মতো ডাকা যায় |
| কখন আনে | app চালুর সাথে সাথে, সবসময় | যখন ঐ লাইনটা **চলে**, তখন |
| ফেরত দেয় | সরাসরি জিনিসটা | একটা **Promise** (পরে আসবে, এমন প্রতিশ্রুতি) |
| Build-এ | মূল bundle-এর ভেতরে | **আলাদা chunk** |

Vite build-এর সময় `import("...")` দেখলেই বোঝে: "এটা পরে লাগবে", তাই ঐ ফাইলটা (আর শুধু ওটারই যা যা লাগে, যেমন recharts) আলাদা chunk-এ রাখে।

কিন্তু dynamic import একটা Promise দেয়, আর route-এর `Component`-এ Promise দেওয়া যায় না, component দিতে হয়। এখানে আসে React-এর **`lazy`**:

```ts
const Analytics = lazy(() => import("@/pages/Admin/Analytics"));
```

`lazy` একটা "নকল" component বানায়। এটা প্রথমবার render হওয়ার সময়:
1. ভেতরের function চালায় → `import()` → chunk download শুরু।
2. Download চলাকালীন React-কে বলে "আমি এখনো তৈরি না, অপেক্ষা করো" (এটাকে বলে **suspend** করা; কী দেখাবে সেটা ধাপ ৫-এ)।
3. Download শেষ হলে আসল `Analytics` component দেখায়।
4. পরের বার আর download লাগে না, browser-এর memory-তে থেকে যায়।

**`() =>` কেন (function-এর ভেতরে):** `lazy(import("..."))` লিখলে import **এখনই** চলে যেত, lazy হতো না। Function-এ মুড়ে দিলে সেটা চলে শুধু যখন `lazy` নিজে ডাকে, অর্থাৎ page প্রথমবার দেখানোর সময়।

---

## ধাপ ৪ — Admin-এর সব page lazy করা

🎯 **লক্ষ্য:** Dashboard-এর কোনো page-ই মূল bundle-এ না রাখা।

🛠️ **কাজ:**

```ts
// src/routes/adminSidebarItems.ts
import type { ISidebarItem } from "@/types";
import { lazy } from "react";

const Overview = lazy(() => import("@/pages/Admin/Overview"));
const Analytics = lazy(() => import("@/pages/Admin/Analytics"));
const AddTour = lazy(() => import("@/pages/Admin/AddTour"));
const AddTourTypes = lazy(() => import("@/pages/Admin/AddTourTypes"));

export const adminSidebarItems: ISidebarItem[] = [
  {
    title: "Dashboard",
    items: [
      { title: "Overview", url: "/admin", component: Overview },
      { title: "Analytics", url: "/admin/analytics", component: Analytics },
    ],
  },
  {
    title: "Tour Management",
    items: [
      { title: "Tours", url: "/admin/tours", component: AddTour },
      { title: "Tour types", url: "/admin/tour-types", component: AddTourTypes },
    ],
  },
];
```

User-এর ফাইলেও একই:

```ts
// src/routes/userSidebarItems.ts
import type { ISidebarItem } from "@/types";
import { lazy } from "react";

const Bookings = lazy(() => import("@/pages/User/Bookings"));

export const userSidebarItems: ISidebarItem[] = [
  {
    title: "History",
    items: [{ title: "Bookings", url: "/user/bookings", component: Bookings }],
  },
];
```

💡 **কেন সবগুলো, শুধু Analytics না:** Teacher-এর কথাই ঠিক: public route ছাড়া বাকি সব lazy করা উচিত। Dashboard-এর page সাধারণ visitor কখনো দেখে না (তারা tour দেখতে আসে), শুধু login করা কিছু মানুষ দেখে। তাদের কোড সবাইকে পাঠানো অপচয়। আর এখন ছোট দেখালেও এই page-গুলোতে পরে form, table, chart আসবে, তখন বড় হবে।

💡 **`generateRoutes` কিছু বদলাতে হলো না:** lazy component দেখতে-আচরণে সাধারণ component-এর মতোই, তাই `component: Analytics` আগের মতোই চলে, route-ও আগের মতো তৈরি হয় (১৪ নম্বর নোট)। একটা ফাইলে item রাখার সুবিধা এখানে আবার দেখা গেল: lazy করতে শুধু এই ফাইলের import বদলাতে হলো।

> নোট — `lazy`-র শর্ত: `import()` করা ফাইলে component **`export default`** হতে হবে। আমাদের সব page `export default function ...`, তাই সমস্যা নেই। কোনো page শুধু named export দিলে (`export function Analytics`) এভাবে লিখতে হয়:
> ```ts
> const Analytics = lazy(() =>
>   import("@/pages/Admin/Analytics").then((m) => ({ default: m.Analytics })),
> );
> ```

> নোট: Comment করা পুরনো import-গুলো (`// import Analytics from ...`) কাজ শেষে মুছে দিন। পরে কেউ (বা আপনি নিজে) ভুল করে uncomment করলে আবার static import হয়ে যাবে, আর lazy loading চুপচাপ নষ্ট হবে।

✅ **যাচাই:**
- Dev mode-এ Home reload করে Network → JS → `analytics` search → **কিছু পাওয়া যায় না**।
- এবার `/admin/analytics`-এ যান → ঠিক তখন `Analytics.tsx` request আসে।
- `bun run build` → output-এ এখন আলাদা লাইন: `dist/assets/Analytics-xxxx.js`, `Overview-xxxx.js` … আর মূল `index-xxxx.js` আগের চেয়ে ছোট।

---

## ধাপ ৫ — `Suspense`: লোড হওয়ার সময় কী দেখাবে

🎯 **লক্ষ্য:** Chunk download হওয়ার মুহূর্তে ফাঁকা বা আটকে থাকা page-এর বদলে একটা loading skeleton দেখানো, আর sidebar যেন জায়গায় থাকে।

### 💡 কেন লাগে

ধাপ ৩-এ দেখেছি, lazy component download চলাকালীন "suspend" করে, অর্থাৎ বলে "আমি এখনো তৈরি না"। তখন React উপরের দিকে খোঁজে: সবচেয়ে কাছের **`<Suspense>`** কোনটা? সেটার `fallback` (বিকল্প UI) দেখায়, আর download শেষ হলে আসল page বসিয়ে দেয়।

```
<Suspense fallback={<Skeleton />}>     ← এখানে থামে, Skeleton দেখায়
  <Analytics />                        ← "এখনো তৈরি না"
</Suspense>
```

**কোথায় বসাব:** `DashboardLayout`-এর ভেতরে, শুধু `<Outlet />`-কে ঘিরে। তাহলে page লোড হওয়ার সময় **শুধু মাঝের অংশে** skeleton দেখায়, sidebar আর header যেমন ছিল তেমন থাকে। পুরো layout-কে ঘিরলে প্রতিবার page বদলাতেই sidebar-ও এক মুহূর্তের জন্য হারিয়ে যেত।

### 🛠️ কাজ ১ — loading skeleton

shadcn-এর `skeleton` আগেই আছে (`sidebar` install-এর সাথে এসেছিল, ১৪ নম্বর নোট)। এটা দিয়ে একটা ছোট reusable component:

```tsx
// src/components/ui/page-loader.tsx
import { Skeleton } from "@/components/ui/skeleton";

export function PageLoader() {
  return (
    <div className="space-y-4" aria-busy="true" aria-label="Loading page">
      <Skeleton className="h-8 w-48" />
      <Skeleton className="h-4 w-72" />
      <div className="grid gap-4 sm:grid-cols-2 lg:grid-cols-3">
        <Skeleton className="h-28" />
        <Skeleton className="h-28" />
        <Skeleton className="h-28" />
      </div>
      <Skeleton className="h-64" />
    </div>
  );
}
```

💡 **Spinner না, skeleton কেন:** skeleton আসল page-এর আকারের ধূসর বাক্স দেখায় (শিরোনাম, কয়েকটা card, একটা বড় অংশ)। User-এর মনে হয় page প্রায় এসে গেছে, আর আসল content আসার সময় কিছু লাফিয়ে সরে না। Spinner-এ মনে হয় "অপেক্ষা করো", skeleton-এ মনে হয় "আসছে"। একই সময় লাগলেও skeleton-এ দ্রুত মনে হয়।

- **`aria-busy`, `aria-label`:** screen reader user-কে জানায় page লোড হচ্ছে।
- রং `Skeleton` নিজে semantic token (`bg-accent`/`bg-muted`) দিয়ে নেয়, তাই dark mode-এও ঠিক।

### 🛠️ কাজ ২ — `DashboardLayout`-এ বসানো

```tsx
// src/components/layout/DashboardLayout.tsx
import { AppSidebar } from "@/components/layout/AppSidebar";
import { PageLoader } from "@/components/ui/page-loader";
import { Separator } from "@/components/ui/separator";
import { SidebarInset, SidebarProvider, SidebarTrigger } from "@/components/ui/sidebar";
import { Suspense } from "react";
import { Outlet } from "react-router";

export default function DashboardLayout() {
  return (
    <SidebarProvider>
      <AppSidebar />
      <SidebarInset>
        <header className="flex h-16 shrink-0 items-center gap-2 border-b px-4">
          <SidebarTrigger className="-ml-1" />
          <Separator orientation="vertical" className="mr-2 data-[orientation=vertical]:h-4" />
        </header>
        <div className="flex flex-1 flex-col gap-4 p-4">
          <Suspense fallback={<PageLoader />}>
            <Outlet />
          </Suspense>
        </div>
      </SidebarInset>
    </SidebarProvider>
  );
}
```

✅ **যাচাই:**
- `bun run preview` চালান, Network-এ **Slow 4G** দিন।
- Address bar-এ সরাসরি `/admin/analytics` লিখে খুলুন → sidebar দেখায়, মাঝে এক মুহূর্ত skeleton, তারপর Analytics।

> নোট: Page-এর ভেতরে sidebar-এর link চাপলে অনেক সময় skeleton না দেখিয়ে **আগের page-টাই এক মুহূর্ত থেকে যায়**, তারপর নতুনটা আসে। এটা ভুল না: React Router page বদলানোর কাজ React-এর "transition" হিসেবে করে, আর transition-এর সময় React ইতিমধ্যে দেখানো জিনিস সরিয়ে skeleton বসায় না (ঝাঁকুনি এড়াতে)। প্রথমবার সরাসরি URL খুললে skeleton দেখাবে। দুই অবস্থাতেই page ঠিকমতো আসে।

---

## ধাপ ৬ — (ঐচ্ছিক) Hover করলেই আগে থেকে আনা

🎯 **লক্ষ্য:** User link-এ click করার আগেই, mouse রাখলেই chunk download শুরু করা, যাতে click-এর পরে প্রায় সাথে সাথে page আসে।

💡 **ধারণা:** মানুষ link-এ mouse রাখার পর click করতে সাধারণত কয়েকশো millisecond নেয়। ঐ সময়েই download শুরু করে দিলে click-এর পরে অপেক্ষা প্রায় থাকে না। আর browser একই `import()` দুইবার download করে না, তাই আগে আনলে পরে `lazy` সরাসরি তৈরি জিনিস পায়।

🛠️ **কাজ (একটা উদাহরণ, Analytics-এর জন্য):**

```ts
// src/routes/adminSidebarItems.ts
// import আলাদা function-এ রাখা, যাতে দুই জায়গা থেকে ডাকা যায়
export const loadAnalytics = () => import("@/pages/Admin/Analytics");
const Analytics = lazy(loadAnalytics);
```

```tsx
// AppSidebar.tsx — Analytics-এর link-এ
<Link to={item.url} onMouseEnter={loadAnalytics} onFocus={loadAnalytics}>
  {item.title}
</Link>
```

- **`onFocus`:** keyboard দিয়ে (Tab চেপে) link-এ এলেও আগে থেকে আনে।

> নোট: সব item-এর জন্য করতে হলে `ISidebarItem`-এ একটা ঐচ্ছিক `preload?: () => Promise<unknown>` field যোগ করে `AppSidebar`-এ `onMouseEnter={item.preload}` দিলেই হয়। এখন page-গুলো ছোট, তাই এটা না করলেও চলে। Analytics-এ chart আসার পর (ভারী page) এটা সত্যিই কাজে লাগবে।

✅ **যাচাই:** Network tab খোলা রেখে sidebar-এর Analytics link-এ শুধু mouse রাখুন (click না) → `Analytics` chunk-এর request চলে আসে। তারপর click → page প্রায় সাথে সাথে।

---

# ভাগ গ: কোন page lazy করব — সেরা পরামর্শ

## ধাপ ৭ — নিয়ম: কোনটা lazy, কোনটা না

🎯 **লক্ষ্য:** প্রতিটা নতুন page-এর ক্ষেত্রে এক নজরে সিদ্ধান্ত নেওয়ার নিয়ম।

আপনি জানতে চেয়েছিলেন: Home, About-এর মতো public route lazy না করলেও চলে, বাকি সব lazy করা উচিত কিনা। নিয়মটা এভাবে মনে রাখুন:

> **প্রথমেই যেটা দেখা যায়, সেটা সরাসরি (eager)। বাকি সব lazy।**

| Page / অংশ | Lazy? | কারণ |
|---|---|---|
| **Home** | ❌ না | বেশিরভাগ visitor এখানেই আসে। Lazy করলে Home দেখাতে **দুইবার** অপেক্ষা: আগে মূল bundle, তারপর Home-এর chunk। প্রথম page যত দ্রুত, তত ভালো |
| **Navbar, Footer, `CommonLayout`** | ❌ না | প্রতিটা public page-এ লাগে, Home-এর সাথেই দরকার |
| **About** | ✅ হ্যাঁ (ছোট লাভ) | কম মানুষ যায়। এখন ছোট, তাই লাভও ছোট; তবে lazy করতে খরচ নেই |
| **Login, Register, Verify** | ✅ **অবশ্যই** | এদের সাথে আসে `react-hook-form`, `zod`, `input-otp`, form-এর সব কোড। Home দেখতে আসা visitor-এর এগুলো লাগে না |
| **`DashboardLayout` (+ `AppSidebar`, sidebar component)** | ✅ **অবশ্যই** | পুরো sidebar-এর কোড শুধু login করা কিছু মানুষের জন্য |
| **Dashboard-এর সব page** | ✅ **অবশ্যই** | ধাপ ৪-এ করা হয়েছে |
| **ভারী library যে page-এ** (chart, map, rich text editor, PDF) | ✅ **অবশ্যই** | এক page-এর জন্য পুরো library সবাইকে পাঠানো সবচেয়ে বড় অপচয় |
| **404 (NotFound)** | ✅ হ্যাঁ | খুব কম দেখা যায় |
| **ছোট shared component** (`Button`, `Input`, `Logo`) | ❌ না | অনেক জায়গায় লাগে, আর এত ছোট যে আলাদা request-এর খরচই বেশি |

💡 **Route-স্তরে lazy করাই সবচেয়ে কাজের:** প্রতিটা ছোট component আলাদা করে lazy করার দরকার নেই। প্রতিটা chunk একটা আলাদা network request, অনেক ছোট ছোট chunk হলে request-এর খরচ বাড়ে, লাভ কমে। Page (route) ধরে ভাগ করলে প্রতিটা chunk-এ যথেষ্ট কোড থাকে, আর user-এর এক page-এ যাওয়া মানে একটা chunk।

---

## ধাপ ৮ — Public route-এ lazy (Login, Register, Verify, About)

🎯 **লক্ষ্য:** Home ছাড়া বাকি public page-গুলোও আলাদা chunk-এ।

🛠️ **কাজ:**

```tsx
// src/routes/index.tsx
import App from "@/App";
import { generateRoutes } from "@/utils/generateRoutes";
import { lazy } from "react";
import { createBrowserRouter, Navigate } from "react-router";
import { adminSidebarItems } from "./adminSidebarItems";
import { userSidebarItems } from "./userSidebarItems";

// Public — প্রথম page না, তাই lazy
const About = lazy(() => import("@/pages/About"));
const LoginPage = lazy(() => import("@/pages/Login"));
const RegisterPage = lazy(() => import("@/pages/Register"));
const Verify = lazy(() => import("@/pages/verify"));

// Dashboard-এর কাঠামো — শুধু login করা user-এর জন্য
const DashboardLayout = lazy(() => import("@/components/layout/DashboardLayout"));

export const router = createBrowserRouter([
  {
    Component: App,
    path: "/",
    children: [{ Component: About, path: "about" }],
  },
  {
    Component: DashboardLayout,
    path: "/admin",
    children: [...generateRoutes(adminSidebarItems)],
  },
  {
    Component: DashboardLayout,
    path: "/user",
    children: [
      { index: true, element: <Navigate to="/user/bookings" replace /> },
      ...generateRoutes(userSidebarItems),
    ],
  },
  { Component: LoginPage, path: "/login" },
  { Component: RegisterPage, path: "/signup" },
  { Component: Verify, path: "/verify" },
]);
```

(Home page বানানো হলে সেটা `App`-এর `children`-এ `{ index: true, Component: Home }` হিসেবে বসবে, **সাধারণ static import** দিয়ে।)

💡 **`DashboardLayout` দুই route-এ, chunk কিন্তু একটা:** `/admin` আর `/user` দুজনেই একই lazy `DashboardLayout` ব্যবহার করছে। একবার download হলে দুই জায়গাতেই কাজে লাগে।

---

## ধাপ ৯ — উপরের স্তরে `Suspense`

🎯 **লক্ষ্য:** Login, Register, About, DashboardLayout লোড হওয়ার সময়ও যেন কিছু একটা দেখায়।

### 💡 সমস্যা

ধাপ ৫-এর `Suspense` আছে `DashboardLayout`-এর **ভেতরে**। কিন্তু এখন `DashboardLayout` নিজেই lazy, আর Login-এর মতো page কোনো layout-এর ভেতরেই নেই। এদের লোড হওয়ার সময় React উপরে খুঁজে কোনো `Suspense` পাবে না। তাই এদের জন্য একদম উপরে একটা `Suspense` লাগবে।

### 🛠️ কাজ

**১. পুরো app-এর জন্য (main.tsx):** সবচেয়ে বাইরের নিরাপত্তা জাল।

```tsx
// src/main.tsx
import { Suspense } from "react";
import { PageLoader } from "@/components/ui/page-loader";

<ThemeProvider defaultTheme="system" storageKey="phovia-ui-theme">
  <Suspense
    fallback={
      <div className="min-h-dvh bg-background p-6">
        <PageLoader />
      </div>
    }
  >
    <RouterProvider router={router} />
  </Suspense>
  <Toaster richColors />
</ThemeProvider>
```

**২. Public site-এর ভেতরে (`CommonLayout`):** About-এর মতো page লোড হওয়ার সময় Navbar/Footer যেন না হারায়, ধাপ ৫-এর একই কৌশল।

```tsx
// src/components/layout/CommonLayout.tsx
import { PageLoader } from "@/components/ui/page-loader";
import { Suspense, type ReactNode } from "react";

<div className="grow">
  <Suspense fallback={<div className="mx-auto max-w-300 px-4 py-8"><PageLoader /></div>}>
    {children}
  </Suspense>
</div>
```

**কোন `Suspense` কখন কাজ করে:**

| কী লোড হচ্ছে | কোন `Suspense` ধরে | কী দেখায় |
|---|---|---|
| About | `CommonLayout`-এর | Navbar + skeleton + Footer |
| Login, Register, Verify | `main.tsx`-এর | পুরো page জুড়ে skeleton |
| `DashboardLayout` (প্রথমবার) | `main.tsx`-এর | পুরো page জুড়ে skeleton |
| Dashboard-এর page | `DashboardLayout`-এর | Sidebar + skeleton |

React সবসময় **সবচেয়ে কাছের** `Suspense` বেছে নেয়। তাই ভেতরের দিকে `Suspense` বসালে বেশি অংশ জায়গায় থাকে; `main.tsx`-এরটা শুধু শেষ ভরসা।

> ➕ সংযোজন — React Router-এর নিজের `lazy`: React Router-এ route-এর একটা `lazy` option-ও আছে, যেটা page-এর কোড আগে আনে, তারপর page দেখায় (তখন `Suspense` লাগে না)। মোটামুটি এভাবে:
> ```ts
> { path: "/login", lazy: async () => ({ Component: (await import("@/pages/Login")).default }) }
> ```
> এটা বিশেষ করে কাজে লাগে যখন route-এ `loader` (page দেখানোর আগে data আনা) থাকে, কারণ কোড আর data একসাথে আনা যায়। আমাদের `generateRoutes` `Component`-এ component দেয়, তাই React-এর `lazy` আমাদের কাঠামোর সাথে সহজে মেলে, এটাই রাখছি। ভবিষ্যতে loader ব্যবহার করলে React Router-এর docs-এ বর্তমান version-এর `lazy`-র লেখার ধরন মিলিয়ে নেবেন।

> ➕ সংযোজন — নতুন deploy-এর পরে chunk না পাওয়া: Site নতুন করে deploy করলে chunk-এর নাম বদলে যায় (`Analytics-d4e5.js` → `Analytics-z9y8.js`)। যার browser-এ পুরনো version খোলা ছিল, সে তখন Analytics-এ গেলে পুরনো নামের ফাইল খুঁজবে, পাবে না, page error দেবে। সমাধান: route-এ `errorElement` দিয়ে একটা "নতুন version এসেছে, page reload দিন" জাতীয় বার্তা দেখানো। Deploy-এর পর্যায়ে গেলে এটা মনে রাখবেন।

✅ **যাচাই (ভাগ গ):**
- `bun run build` → output-এ এখন `Login-…js`, `Register-…js`, `verify-…js`, `DashboardLayout-…js`, `About-…js` আলাদা।
- **মূল `index-…js` ধাপ ২-এর চেয়ে অনেক ছোট** (gzip সংখ্যা মিলিয়ে দেখুন)। `zod`, `react-hook-form`, sidebar-এর কোড আর এতে নেই।
- `bun run preview` → Home-এ Network → JS: শুধু মূল chunk; `/login`-এ গেলে তখন Login-এর chunk।
- Slow 4G-তে সরাসরি `/login` খুললে এক মুহূর্ত skeleton, তারপর Login; কোনো ফাঁকা সাদা page না।

---

# ভাগ ঘ: Lazy ছাড়াও দ্রুত করা

আপনার আগের site-গুলো slow ছিল বলে এই অংশ। বাস্তবে site slow হওয়ার সবচেয়ে বড় কারণ প্রায়ই **ছবি**, তারপর ভারী library, তারপর font। JavaScript ভাগ করা (ভাগ খ, গ) একটা অংশ মাত্র।

## ধাপ ১০ — ছবি (সবচেয়ে বড় লাভ)

🎯 **লক্ষ্য:** ছবির মাপ কমানো, যেটা প্রায়ই সব JavaScript-এর চেয়েও বড়।

### 💡 আপনার project-এ এখনই

`src/assets/images/auth/inani-beach.jpeg` = **৮৪৯ kB**। তুলনা করুন: পুরো app-এর JavaScript (gzip) সম্ভবত এর চেয়ে কম। মোবাইল internet-এ শুধু এই একটা ছবি আসতেই কয়েক সেকেন্ড লাগতে পারে, আর এটা login/register page-এর মূল ছবি, তাই page "অসম্পূর্ণ" দেখায়।

> নোট: ছবি `import` করলেও (`import heroImage from "...jpeg"`) সেটা JavaScript-এর ভেতরে ঢোকে না। Vite শুধু ফাইলের URL দেয়, আর browser ছবিটা আনে **যখন `<img>` screen-এ বসে**। তাই Home-এ থাকলে login-এর ছবি আসে না। সমস্যা হলো login page খুললে এই ভারী ছবি আসতে দেরি।

### 🛠️ কাজ

**১. Format বদলানো আর মাপ কমানো:** https://squoosh.app-এ ছবি খুলে:
- **Resize:** screen-এ যত বড় দেখায় তার কাছাকাছি। Auth page-এর ছবি অর্ধেক screen জুড়ে, তাই প্রস্থ ১২০০–১৬০০px যথেষ্ট (৪০০০px-এর camera ছবি দরকার নেই)।
- **Format:** **WebP** (সব আধুনিক browser চেনে) বা **AVIF** (আরও ছোট)।
- **Quality:** ৭৫–৮০; চোখে পার্থক্য প্রায় বোঝা যায় না।
- লক্ষ্য: **১০০–২০০ kB**। প্রায় ৮৫০ kB থেকে ৫ ভাগের ১ ভাগ।

তারপর import-এ নতুন ফাইলের নাম দিন: `import heroImage from "@/assets/images/auth/inani-beach.webp";`

**২. `<img>`-এ সঠিক attribute:**

```tsx
// Login/Register-এর বড় ছবি: page খুললেই চোখে পড়ে → আগে আনো
<img
  src={heroImage}
  alt="Inani Beach, Cox's Bazar"
  width={1200}
  height={1600}
  fetchPriority="high"
  className="absolute inset-0 size-full object-cover"
/>

// Page-এর নিচের দিকের ছবি (tour card ইত্যাদি): scroll করে কাছে এলে তবেই আনো
<img src={tour.image} alt={tour.title} width={400} height={300} loading="lazy" decoding="async" />
```

| Attribute | কাজ |
|---|---|
| `width`, `height` | browser ছবি আসার আগেই জায়গা রেখে দেয়, তাই ছবি এলে লেখা নিচে লাফিয়ে সরে না (layout shift)। CSS দিয়ে আকার বদলালেও এটা দিন, browser এখান থেকে অনুপাত (ratio) বোঝে |
| `fetchPriority="high"` | "এটা সবার আগে আনো"। শুধু প্রথমেই দেখা যায় এমন **একটা** বড় ছবিতে |
| `loading="lazy"` | screen-এর কাছে না আসা পর্যন্ত আনে না। ছবির জন্য lazy loading, browser নিজেই করে |
| `decoding="async"` | ছবি খোলার (decode) কাজ page আটকে না করে আলাদাভাবে করে |

> ⚠️ সতর্কতা: প্রথমেই দেখা যায় এমন ছবিতে (hero, auth-এর বড় ছবি) **`loading="lazy"` দেবেন না**। তাতে browser ঐ ছবি আনতে ইচ্ছা করে দেরি করে, আর page-এর মূল অংশ দেরিতে আসে। Lazy শুধু নিচের দিকের ছবির জন্য।

> নোট: পরে tour-এর ছবি backend থেকে (Cloudinary-র মতো service) এলে সেখানে URL-এ মাপ আর format বলে দেওয়া যায় (যেমন প্রস্থ ৪০০, WebP)। তখন একই নিয়ম: card-এ ছোট ছবি, details page-এ বড়।

✅ **যাচাই:** `bun run build`-এর output-এ ছবির লাইন ৮৪৯ kB থেকে কয়েকশো kB-এর নিচে নেমেছে।

---

## ধাপ ১১ — Font

🎯 **লক্ষ্য:** অপ্রয়োজনীয় font weight বাদ দেওয়া।

### 💡 আপনার project-এ এখনই

`index.css`-এ Poppins-এর ৫টা weight (400, 500, 600, 700, 800) আর Hind Siliguri-র ২টা (500, 600) import করা।

**ভালো খবর:** Fontsource-এর প্রতিটা weight-এর CSS-এ `unicode-range` দেওয়া থাকে, আর browser font ফাইল আনে শুধু তখন, যখন page-এ ঐ weight-এ আর ঐ ভাষার অক্ষরে সত্যিই কিছু লেখা থাকে। তাই ৫টা import করলেই ৫টা font ফাইল আসে না। আর `font-display: swap` থাকায় font আসার আগেও লেখা system font-এ দেখা যায়।

**তবু যা করবেন:** কোনো weight project-এ কোথাও ব্যবহার না হলে (যেমন `font-extrabold` = 800 কোথাও আছে কিনা) সেই import মুছে দিন। CSS ছোট হয়, আর ভুল করে ঐ weight-এ কিছু লিখলে বাড়তি ফাইল আসে না।

```sh
# project-এ 800 weight কোথাও আছে কিনা খোঁজা
grep -rn "font-extrabold" src
```

> নোট: Bangla লেখা থাকলে Hind Siliguri-র ফাইল আসবে, সেটা স্বাভাবিক। ইংরেজি-only page-এ এটা আসে না (unicode-range-এর কারণে)।

✅ **যাচাই:** `bun run preview` → Network → **Font** filter → reload: শুধু যে weight-গুলো page-এ দেখা যাচ্ছে সেগুলোই আসে।

---

## ধাপ ১২ — Library

🎯 **লক্ষ্য:** ভারী library-কে মূল bundle-এ ঢুকতে না দেওয়া।

💡 **নিয়ম:**

| নিয়ম | উদাহরণ |
|---|---|
| ভারী library শুধু lazy page-এর ভেতরে import করুন | `recharts` শুধু `Analytics.tsx`-এ। Navbar বা `App`-এর মতো সবসময়-লোড-হওয়া ফাইলে import করলে আবার মূল bundle-এ চলে আসবে |
| Library-র শুধু দরকারি অংশ import করুন | `import { LogOut } from "lucide-react"` ✅ (শুধু ঐ icon আসে); পুরো library একসাথে import করবেন না |
| নতুন library যোগের আগে মাপ দেখুন | https://bundlephobia.com-এ নাম লিখলে কত kB, বোঝা যায়। ছোট কাজের জন্য বড় library এড়ান (যেমন শুধু তারিখ সাজাতে বিশাল date library না, browser-এর নিজের `Intl.DateTimeFormat`) |
| Shared component-এ ভারী import নয় | `components/ui/`-এর কোনো ফাইল chart import করলে, যেখানে সেটা ব্যবহার হয় সব page-এ chart চলে আসে |

> নোট: `package.json`-এ থাকা কিন্তু কোথাও import না করা package (আগের নোটে বলা `add`, `button`, `field`, `input`) bundle-এর মাপ বাড়ায় না, কারণ Vite শুধু import-করা কোড নেয়। এগুলো সরানো দরকার project পরিষ্কার রাখতে আর install দ্রুত করতে, speed-এর জন্য না।

✅ **যাচাই:** `bun run build` output-এ সবচেয়ে বড় chunk কোনটা দেখুন। মূল `index-…js` বড় হতে থাকলে সাম্প্রতিক কোন ফাইলে নতুন import যোগ হয়েছে খুঁজুন।

---

## ধাপ ১৩ — Lighthouse দিয়ে পুরো মাপ

🎯 **লক্ষ্য:** একটা score দিয়ে বোঝা site কতটা দ্রুত, আর কোথায় সমস্যা।

🛠️ **কাজ:**
1. `bun run build` তারপর `bun run preview`।
2. Chrome-এর **Incognito** window-এ খুলুন (extension-গুলো মাপ নষ্ট করে)।
3. DevTools → **Lighthouse** → Mode: Navigation, Device: **Mobile** → Analyze।

| মাপ | মানে | ভালো |
|---|---|---|
| **Performance score** | সব মিলিয়ে | ৯০+ |
| **LCP** (Largest Contentful Paint) | সবচেয়ে বড় জিনিস (সাধারণত hero ছবি বা বড় শিরোনাম) কখন দেখা গেল | ২.৫ সেকেন্ডের কম |
| **CLS** (Cumulative Layout Shift) | লোড হওয়ার সময় জিনিস কতটা লাফিয়ে সরেছে | ০.১-এর কম (ছবিতে `width`/`height` দিলে কমে) |
| **TBT** (Total Blocking Time) | বড় JavaScript-এর কারণে page কতক্ষণ সাড়া দেয়নি | ২০০ms-এর কম (lazy loading এখানে সাহায্য করে) |

Report-এর নিচে "Opportunities"-এ Lighthouse নিজেই বলে দেয় কোন ছবি বড়, কোন JS অব্যবহৃত।

> নোট: Dev mode-এ (`bun run dev`) Lighthouse চালাবেন না, score সবসময় খারাপ আসবে। শুধু `preview` বা deploy-করা site-এ।

✅ **যাচাই:** Home আর `/login` দুটোর score লিখে রাখুন। ধাপ ১০-এর ছবি ঠিক করার আগে-পরে `/login`-এর LCP তুলনা করুন।

---

# সারসংক্ষেপ

### এই নোটে যা বদলালাম

| ফাইল | পরিবর্তন |
|---|---|
| `routes/adminSidebarItems.ts`, `routes/userSidebarItems.ts` | সব page `lazy(() => import(...))` |
| `routes/index.tsx` | About, Login, Register, Verify, `DashboardLayout` lazy; Home (পরে) static |
| `components/ui/page-loader.tsx` | নতুন: skeleton loader |
| `components/layout/DashboardLayout.tsx` | `<Outlet />`-এর চারপাশে `Suspense` |
| `components/layout/CommonLayout.tsx` | `{children}`-এর চারপাশে `Suspense` |
| `main.tsx` | `RouterProvider`-এর চারপাশে শেষ ভরসার `Suspense` |
| ছবি | ৮৪৯ kB JPEG → WebP/AVIF, `width`/`height`, `fetchPriority`/`loading` |

### নতুন page যোগ করার সময় মনে রাখার নিয়ম

1. **Dashboard page?** → `adminSidebarItems.ts`/`userSidebarItems.ts`-এ `lazy(() => import(...))` দিয়ে।
2. **Public page?** → Home ছাড়া সব `routes/index.tsx`-এ `lazy`।
3. **ভারী library?** → শুধু ঐ page-এর ভেতরে import।
4. **ছবি?** → WebP, ঠিক মাপ, `width`/`height`; নিচের দিকে হলে `loading="lazy"`।
5. **কিছু যোগ করার পর** → `bun run build` দিয়ে মূল chunk-এর মাপ দেখে নিন।

---

# Check-list

**মাপা**
- [ ] `bun run build`-এর output (বিশেষ করে মূল `index-…js`-এর gzip) আগে-পরে লিখে রাখা আছে
- [ ] মাপ সবসময় `bun run preview`-এ, Disable cache + Slow/Fast 4G দিয়ে

**Lazy loading**
- [ ] Home-এ Network → JS-এ Analytics, Login, Dashboard-এর chunk আসে না
- [ ] Build output-এ প্রতিটা lazy page-এর আলাদা `…-hash.js` আছে
- [ ] Comment করা পুরনো static import মুছে দেওয়া হয়েছে
- [ ] সরাসরি `/admin/analytics` খুললে sidebar থাকে, মাঝে skeleton, তারপর page
- [ ] সরাসরি `/login` খুললে ফাঁকা সাদা page না, skeleton দেখায়
- [ ] Dark mode-এ skeleton দেখা যায়

**Lazy ছাড়াও**
- [ ] Auth-এর ছবি WebP/AVIF, কয়েকশো kB-এর নিচে
- [ ] বড় hero ছবিতে `fetchPriority="high"`, `loading="lazy"` না
- [ ] সব `<img>`-এ `width`, `height`
- [ ] অব্যবহৃত font weight-এর import নেই
- [ ] Lighthouse (Mobile, Incognito, preview)-এ Home আর `/login`-এর Performance score লিখে রাখা আছে
