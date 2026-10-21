# Frontend Notes — ১৪: Dashboard আর Role-based Sidebar — এক জায়গা থেকে Menu আর Route

এই নোটে user আর admin দুজনের জন্য dashboard বানাব। শেষে এমন একটা ব্যবস্থা দাঁড়াবে যেখানে **একটা ফাইলে** item লিখলেই sidebar-এ menu আসবে, route-ও তৈরি হয়ে যাবে, আর কোন role কোন menu দেখবে সেটাও এক জায়গা থেকে ঠিক হবে।

আগের নোটের মতো প্রতিটা ধাপে চারটা জিনিস থাকবে:

- 🎯 **লক্ষ্য:** এই ধাপে কী অর্জন করব
- 🛠️ **কাজ:** শুধু এই ধাপে যতটুকু কোড লাগে
- 💡 **কেন:** এটা না করলে কী হতো
- ✅ **যাচাই:** browser-এ কী দেখলে বুঝব ধাপটা ঠিক হয়েছে

Teacher যেভাবে ভেঙে ভেঙে দেখিয়েছেন, এখানেও সেই ক্রম: আগে সাধারণভাবে বানানো, তারপর সমস্যা দেখা, তারপর ভালো ব্যবস্থা।

> নোট: এই নোটের কোড আপনার project-এর বর্তমান version মেনে লেখা: React Router 8, shadcn-এর নতুন `radix-nova` style (`cn` আসে `cn` package থেকে, Slot আসে `radix-ui` থেকে)। shadcn-এর docs বা teacher-এর ভিডিওতে পুরনো কোড দেখলে (যেমন `clsx` + `tailwind-merge` দিয়ে `cn`, বা `@radix-ui/react-slot`), সেটা দিয়ে আপনার কোড বদলাবেন না।

---

## ভূমিকা: শেষে কী দাঁড়াবে

```
routes/adminSidebarItems.ts   ← এখানে একটা item লিখলাম: { title, url, component }
            │
            ├──→ generateRoutes()  → routes/index.tsx-এ route তৈরি
            │
            └──→ getSidebarItems(role) → AppSidebar-এ menu দেখায় (শুধু ঐ role-কে)
```

**আগে:** নতুন page যোগ করতে দুই জায়গায় লিখতে হতো (sidebar-এর menu + route)। একটা ভুলে গেলে link আছে কিন্তু page নেই, বা page আছে কিন্তু link নেই।
**পরে:** এক জায়গায় লিখলেই দুটোই হয়।

### ধাপের মানচিত্র

| ভাগ | ধাপ | শেষে কী কাজ করবে |
|---|---|---|
| **ক. সাধারণ Dashboard** | ১–৪ | `/admin/analytics`, `/user/bookings` খোলে |
| **খ. shadcn Sidebar** | ৫–৮ | Sidebar সহ সুন্দর dashboard, কিন্তু সবাই admin-এর menu দেখে |
| **গ. এক জায়গা থেকে Menu + Route** | ৯–১২ | Item লিখলেই route তৈরি হয় |
| **ঘ. Role অনুযায়ী** | ১৩–১৯ | Admin admin-এর menu, user user-এর menu দেখে |

### শেষে যে ফাইলগুলো থাকবে

```
src/
├── components/layout/
│   ├── DashboardLayout.tsx     ← ধাপ ২, ৬
│   ├── AppSidebar.tsx          ← ধাপ ৭, ৯, ১৭
│   └── Navbar.tsx              ← ধাপ ১৮
├── constants/
│   └── role.ts                 ← ধাপ ১৩
├── pages/
│   ├── Admin/
│   │   ├── Overview.tsx        ← ধাপ ১, ১০
│   │   ├── Analytics.tsx
│   │   ├── AddTour.tsx
│   │   └── AddTourTypes.tsx
│   └── User/
│       └── Bookings.tsx
├── routes/
│   ├── index.tsx               ← ধাপ ৩, ১২, ১৪, ১৯
│   ├── adminSidebarItems.ts    ← ধাপ ৯, ১০
│   └── userSidebarItems.ts     ← ধাপ ১৪
├── types/
│   └── index.ts                ← ধাপ ১০, ১৫
└── utils/
    ├── generateRoutes.ts       ← ধাপ ১১
    └── getSidebarItems.ts      ← ধাপ ১৫
```

---

# ভাগ ক: সাধারণ Dashboard

## ধাপ ১ — Dashboard-এর page বানানো

🎯 **লক্ষ্য:** Admin আর user-এর জন্য কয়েকটা খালি page তৈরি করা, যাতে route-এ বসানোর মতো কিছু থাকে।

🛠️ **কাজ:** `pages`-এর ভেতরে দুটো folder, আর প্রতিটায় সাধারণ component (snippet দিয়ে বানানো, এখনো শুধু একটা লেখা):

```
pages/
├── Admin/
│   ├── Analytics.tsx
│   ├── AddTour.tsx
│   └── AddTourTypes.tsx
└── User/
    └── Bookings.tsx
```

```tsx
// src/pages/Admin/Analytics.tsx  (বাকিগুলোও একই রকম, শুধু নাম আলাদা)
export default function Analytics() {
  return (
    <div>
      <h1>This is Analytics component</h1>
    </div>
  );
}
```

💡 **Folder ভাগ কেন:** Dashboard দুই রকম user-এর জন্য: **Admin** (tour যোগ করা, analytics দেখা) আর **User** (নিজের booking দেখা)। Role অনুযায়ী page আলাদা folder-এ রাখলে পরে খুঁজে পাওয়া সহজ।

> ⚠️ সংশোধন (ছোট): আপনার `pages/User/Bookings.tsx` ফাইলের ভেতরে component-এর নাম `Booking` (শেষে `s` নেই)। আর নোটের প্রথম route-এ import ছিল `@/pages/User/Booking` (ফাইলের নামেও `s` নেই)। আমাদের নিয়ম: ফাইলের নাম = component-এর নাম। তাই component-এর নাম `Bookings` রাখুন। (Default export হওয়ায় import-এ যে নামই দিন কাজ করে, তাই code ভাঙেনি; কিন্তু নাম না মিললে পরে খুঁজতে গিয়ে গুলিয়ে যায়।)

✅ **যাচাই:** ফাইলগুলোতে কোনো error নেই।

---

## ধাপ ২ — `DashboardLayout` (প্রথম রূপ)

🎯 **লক্ষ্য:** Dashboard-এর page-গুলো যেখানে বসবে, সেই layout।

🛠️ **কাজ:**

```tsx
// src/components/layout/DashboardLayout.tsx  (প্রথম রূপ — ধাপ ৬-এ sidebar যোগ হবে)
import { Outlet } from "react-router";

export default function DashboardLayout() {
  return (
    <div>
      <Outlet />
    </div>
  );
}
```

💡 **কেন:** `<Outlet />` হলো ফাঁকা জায়গা, যেখানে child route-এর page বসবে (৫ নম্বর নোট)। এখন শুধু Outlet রাখছি, যাতে আগে route ঠিকমতো কাজ করে কিনা দেখা যায়। Sidebar পরে আসবে। Public site-এর `App`-এ যেমন Navbar + Footer থাকে, dashboard-এ তেমন sidebar থাকবে।

---

## ধাপ ৩ — Dashboard-এর route

🎯 **লক্ষ্য:** `/admin/...` আর `/user/...` URL-এ page দেখানো।

🛠️ **কাজ:** Admin আর User দুজনেই `DashboardLayout` ব্যবহার করবে, কিন্তু আলাদা route group-এ:

```tsx
// src/routes/index.tsx  (প্রথম রূপ — হাতে লেখা route)
import App from "@/App";
import DashboardLayout from "@/components/layout/DashboardLayout";
import About from "@/pages/About";
import AddTour from "@/pages/Admin/AddTour";
import Analytics from "@/pages/Admin/Analytics";
import LoginPage from "@/pages/Login";
import RegisterPage from "@/pages/Register";
import Bookings from "@/pages/User/Bookings";
import Verify from "@/pages/verify";
import { createBrowserRouter } from "react-router";

export const router = createBrowserRouter([
  {
    Component: App,
    path: "/",
    children: [{ Component: About, path: "about" }],
  },
  {
    Component: DashboardLayout,
    path: "/admin",
    children: [
      { Component: Analytics, path: "analytics" },
      { Component: AddTour, path: "add-tour" },
    ],
  },
  {
    Component: DashboardLayout,
    path: "/user",
    children: [{ Component: Bookings, path: "bookings" }],
  },
  { Component: LoginPage, path: "/login" },
  { Component: RegisterPage, path: "/signup" },
  { Component: Verify, path: "/verify" },
]);
```

💡 **কেন এভাবে:**
- `/admin` আর `/user` দুটোই **top level**-এ, `App`-এর বাইরে। কারণ dashboard-এ public site-এর Navbar/Footer থাকবে না, নিজের sidebar থাকবে (৫ নম্বর নোটের Admin Layout অংশ)।
- একই `DashboardLayout` দুই জায়গায়: কাঠামো একই (sidebar + content), শুধু ভেতরের page আলাদা।

> ⚠️ সংশোধন: নোটে ফাইলের নাম `routes/index.ts` লেখা ছিল। আপনার project-এ ফাইলটা `index.tsx`, আর সেটাই ঠিক। কারণ ধাপ ১৯-এ এখানে `<Navigate />` (JSX) বসবে, আর JSX থাকলে ফাইল `.tsx` হতেই হবে (৫ নম্বর নোট)।

> ⚠️ সংশোধন: Route-এ register page `/signup`-এ আছে, কিন্তু আপনার `Navbar.tsx`-এ desktop-এর Sign up `/signup`-এ আর mobile menu-র Sign up `/register`-এ যাচ্ছে। ফলে mobile থেকে Sign up চাপলে কোনো page পাওয়া যাবে না। যেকোনো একটা নাম বেছে তিন জায়গায় (route, desktop link, mobile link) একই রাখুন। (আগের নোটগুলোতে `/register` ছিল।)

✅ **যাচাই:**
- `/admin/analytics` খুললে "This is Analytics component"
- `/user/bookings` খুললে "This is Booking component"

---

## ধাপ ৪ — Relative আর Absolute path

🎯 **লক্ষ্য:** Child route-এর path-এ শুরুতে `/` কখন দেব আর কখন দেব না, সেটা বোঝা।

উপরের route-এ খেয়াল করুন: parent-এ `/` আছে (`"/admin"`), কিন্তু child-এ নেই (`"analytics"`)।

### Relative path (শুরুতে `/` নেই)

```tsx
{
  path: "/admin",
  children: [{ path: "analytics", Component: Analytics }],
}
// আসল URL = parent + child = /admin/analytics
```

শুরুতে `/` না থাকা মানে "আমার parent-এর **সাপেক্ষে**"। অর্থাৎ analytics-এর আগে নিজে থেকেই admin বসে যায়।

### Absolute path (শুরুতে `/` আছে)

শুরুতে `/` দিলে path আর parent-এর সাপেক্ষে থাকে না, website-এর একদম শুরু (root) থেকে ধরা হয়। তখন পুরো path নিজেকেই লিখতে হয়:

```tsx
{
  path: "/admin",
  children: [{ path: "/admin/analytics", Component: Analytics }], // ✅ পুরোটা লেখা
}
```

```tsx
{
  path: "/admin",
  children: [{ path: "/analytics", Component: Analytics }], // ❌ error
}
```

> ⚠️ সংশোধন: নোটে লেখা ছিল child-এ `/analytics` দিলে "`/admin/analytics`-এ গেলেও error দেবে, `/analytics`-এ গেলেও error দেবে"। আসলে এটা কোনো page-এ যাওয়ার সময়ের error না: **app চালু হওয়ার সময়েই** React Router error দেয়, কোনো page-ই খোলে না। কারণ নিয়ম হলো: **parent-এর ভেতরে absolute path দিলে সেটা parent-এর পুরো path দিয়েই শুরু হতে হবে।** `/analytics` তো `/admin` দিয়ে শুরু হয়নি, তাই React Router বুঝতে পারে না এটা কীভাবে `/admin`-এর child হয়। Error-এর লেখা এমন: *"Absolute route path "/analytics" nested under path "/admin" is not valid. An absolute child route path must start with the combined path of all its parent routes."*

### কোনটা কখন

| | Relative (`"analytics"`) | Absolute (`"/admin/analytics"`) |
|---|---|---|
| Route-এ লেখা | ছোট | পুরোটা |
| Parent-এর নাম বদলালে | child নিজে থেকেই ঠিক থাকে | প্রতিটা child-এ বদলাতে হয় |
| অন্য জায়গায় ব্যবহার (Link, navigate) | ওখানে পুরো `/admin/analytics` আলাদা করে লিখতে হয় | একই string সরাসরি ব্যবহার করা যায় |

Teacher-এর শেষ কথাটা জরুরি: route-এ relative লিখলেও, কোডের **অন্য কোথাও** (যেমন `<Link to>`, `navigate()`) শুধু `analytics` বা `/analytics` লিখলে হবে না, পুরো `/admin/analytics` লিখতে হবে।

💡 **এটা পরে কেন কাজে লাগবে:** ভাগ গ-তে আমরা sidebar-এর item থেকে route বানাব। Sidebar-এর link-এ পুরো URL (`/admin/tours`) লাগে, আবার route-এও একই string ব্যবহার করব। তাই তখন **absolute path** ব্যবহার করব, যাতে একই `url` দুই কাজেই চলে।

✅ **যাচাই (পরীক্ষা):** সাময়িকভাবে `"analytics"`-কে `"/admin/analytics"` বানান → আগের মতোই কাজ করে। `"/analytics"` বানান → app খোলেই না, console-এ উপরের error। দেখা হলে আগের মতো `"analytics"` করে দিন।

### সমস্যা যেটা সামনে আসছে

`routes/index.tsx` এখনই বড় হচ্ছে, আর admin-এর page আরও আসবে (tours, divisions, bookings, users …)। ফাইলটা বড় হয়ে maintain করা কঠিন হবে। Teacher বলছেন, এটাকে **user-এর role অনুযায়ী** ভাগ করব, আর এমন একটা ফাইল রাখব যেখান থেকে **dashboard-এর menu আর route দুটোই** তৈরি হবে। এতে maintain করা সহজ হবে, কোডও কমবে। সেটা ভাগ গ-তে। আগে dashboard দেখতে ভালো করি।

---

# ভাগ খ: shadcn Sidebar

## ধাপ ৫ — shadcn component install

🎯 **লক্ষ্য:** Sidebar-সহ dashboard বানানোর component আনা।

🛠️ **কাজ:**

```sh
bunx --bun shadcn@latest add sidebar breadcrumb avatar select badge chart
```

| Component | কাজ | কখন লাগবে |
|---|---|---|
| `sidebar` | বাম দিকের menu, mobile-এ খোলা/বন্ধ হয় | এখনই |
| `breadcrumb` | উপরের "Admin › Overview" পথ | এখনই |
| `avatar`, `select`, `badge`, `chart` | ছবি, dropdown, ছোট label, graph | পরে, dashboard-এর ভেতরের page-এ |

`sidebar` install করলে shadcn নিজে থেকেই এর লাগা অন্য component-গুলোও আনে (`sheet`, `tooltip`, `skeleton`, `separator`, `input`) আর `hooks/use-mobile.ts` বানায়। `chart` install করলে `recharts` package আসে।

💡 **Teacher যা করলেন:** https://ui.shadcn.com/blocks/sidebar থেকে একটা তৈরি dashboard (block) বেছে, **Code** tab থেকে কোড কপি করে নিজের ফাইলে বসিয়েছেন। আপনি Claude-এর সাহায্যে আমাদের ডিজাইনের (রং, logo) সাথে মিলিয়ে নিয়েছেন। দুটোই ঠিক।

> নোট: Block-এর কোড কপির সময় উপরে style-টা দেখে নেবেন। আপনার project-এর style `radix-nova` (`components.json`)। ভিন্ন style-এর কোড কপি করলে import বা class মিলবে না।

> নোট: নতুন shadcn-এ `cn` আসছে `cn` নামের package থেকে (shadcn-এর নিজের, `lib/utils.ts`-এ `export { cn } from "cn"`)। তাই shadcn-এর ফাইলগুলোতে `import { cn } from "cn"` দেখলে অবাক হবেন না, এটাই নতুন নিয়ম। পুরনো docs-এর `clsx` + `tailwind-merge` দিয়ে বদলাবেন না।

✅ **যাচাই:** `src/components/ui/`-এ `sidebar.tsx`, `breadcrumb.tsx` ইত্যাদি এসেছে।

---

## ধাপ ৬ — `DashboardLayout`-এ sidebar

🎯 **লক্ষ্য:** বাম দিকে sidebar, উপরে header, ডান দিকে page।

🛠️ **কাজ:**

```tsx
// src/components/layout/DashboardLayout.tsx  (চূড়ান্ত)
import { AppSidebar } from "@/components/layout/AppSidebar";
import { Separator } from "@/components/ui/separator";
import { SidebarInset, SidebarProvider, SidebarTrigger } from "@/components/ui/sidebar";
import { Outlet } from "react-router";

export default function DashboardLayout() {
  return (
    <SidebarProvider>
      <AppSidebar />
      <SidebarInset>
        <header className="flex h-16 shrink-0 items-center gap-2 border-b px-4">
          <SidebarTrigger className="-ml-1" />
          <Separator orientation="vertical" className="mr-2 data-[orientation=vertical]:h-4" />
          {/* Breadcrumb এখানে বসবে (নিচে নোট) */}
        </header>
        <div className="flex flex-1 flex-col gap-4 p-4">
          <Outlet />
        </div>
      </SidebarInset>
    </SidebarProvider>
  );
}
```

💡 **অংশগুলো:**

| Component | কাজ |
|---|---|
| `SidebarProvider` | sidebar খোলা না বন্ধ, সেই অবস্থা মনে রাখে (context)। ভেতরের সবাই এটা জানতে পারে |
| `AppSidebar` | আমাদের নিজের sidebar (পরের ধাপে) |
| `SidebarInset` | sidebar-এর পাশের মূল অংশ। Sidebar খুললে/বন্ধ হলে নিজে থেকে জায়গা ঠিক করে |
| `SidebarTrigger` | ☰ button: চাপলে sidebar খোলে/বন্ধ হয় (desktop-এ ছোট-বড়, mobile-এ পাশ থেকে আসে) |
| `<Outlet />` | এখানেই Analytics, Bookings ইত্যাদি page বসে |

> নোট — Breadcrumb: Teacher-এর কোডে header-এ `Admin › Overview` লেখা breadcrumb ছিল। আপনি সেটা comment করে রেখেছেন, কারণ লেখাটা hardcode: যে page-এই যান, সবসময় "Overview" দেখাত। এটা ঠিক সিদ্ধান্ত। পরে চাইলে বর্তমান URL (`useLocation`) আর sidebar-এর item মিলিয়ে আসল page-এর নাম দেখানো যাবে।

✅ **যাচাই:** `/admin/analytics`-এ বাম দিকে sidebar, উপরে ☰ button; ☰ চাপলে sidebar খোলে-বন্ধ হয়।

---

## ধাপ ৭ — `AppSidebar` (hardcode করা menu)

🎯 **লক্ষ্য:** Sidebar-এ section ভাগ করা menu দেখানো।

🛠️ **কাজ:** shadcn-এর sidebar-01 block-এর মতো: section (`title`) আর তার ভেতরে link (`items`)।

```tsx
// src/components/layout/AppSidebar.tsx  (প্রথম রূপ — menu এখনো ফাইলের ভেতরে)
import {
  Sidebar,
  SidebarContent,
  SidebarFooter,
  SidebarGroup,
  SidebarGroupContent,
  SidebarGroupLabel,
  SidebarHeader,
  SidebarMenu,
  SidebarMenuButton,
  SidebarMenuItem,
  SidebarRail,
} from "@/components/ui/sidebar";
import * as React from "react";
import { Link, useLocation } from "react-router";

const data = {
  navMain: [
    {
      title: "Dashboard",
      items: [{ title: "Overview", url: "/admin" }],
    },
    {
      title: "Catalogue",
      items: [
        { title: "Tours", url: "/admin/tours" },
        { title: "Tour types", url: "/admin/tour-types" },
        { title: "Divisions", url: "/admin/divisions" },
      ],
    },
    {
      title: "Operations",
      items: [
        { title: "Bookings", url: "/admin/bookings" },
        { title: "Users", url: "/admin/users" },
      ],
    },
  ],
};

export function AppSidebar({ ...props }: React.ComponentProps<typeof Sidebar>) {
  const { pathname } = useLocation();

  return (
    <Sidebar {...props}>
      <SidebarHeader>
        <Link to="/" className="flex items-center gap-2.5 p-2">
          <svg width="25" height="29" viewBox="0 0 26 30" aria-hidden="true">
            <path
              d="M13 0C5.8 0 0 5.6 0 12.6 0 21.7 13 30 13 30s13-8.3 13-17.4C26 5.6 20.2 0 13 0z"
              className="fill-sidebar-primary"
            />
            <circle cx="13" cy="12.5" r="5" fill="#fff" />
          </svg>
          <span className="leading-tight">
            <span className="block text-lg font-bold text-white">Phovia Tour</span>
            <span className="text-xs text-sidebar-foreground/70">Dashboard console</span>
          </span>
        </Link>
      </SidebarHeader>

      <SidebarContent>
        {data.navMain.map((group) => (
          <SidebarGroup key={group.title}>
            <SidebarGroupLabel className="text-sidebar-foreground/60">{group.title}</SidebarGroupLabel>
            <SidebarGroupContent>
              <SidebarMenu>
                {group.items.map((item) => (
                  <SidebarMenuItem key={item.title}>
                    <SidebarMenuButton
                      asChild
                      isActive={pathname === item.url}
                      className="data-[active=true]:shadow-[inset_3px_0_0_var(--star)]"
                    >
                      <Link to={item.url}>{item.title}</Link>
                    </SidebarMenuButton>
                  </SidebarMenuItem>
                ))}
              </SidebarMenu>
            </SidebarGroupContent>
          </SidebarGroup>
        ))}
      </SidebarContent>

      <SidebarFooter>
        <SidebarMenu>
          <SidebarMenuItem>
            <SidebarMenuButton asChild>
              <Link to="/">View public site</Link>
            </SidebarMenuButton>
          </SidebarMenuItem>
          <SidebarMenuItem>
            <SidebarMenuButton>Log out</SidebarMenuButton>
          </SidebarMenuItem>
        </SidebarMenu>
      </SidebarFooter>

      <SidebarRail />
    </Sidebar>
  );
}
```

💡 **লাইনগুলোর মানে:**
- **দুইবার `map`:** বাইরের `map` প্রতিটা section (Dashboard, Catalogue, Operations) বানায়, ভেতরের `map` সেই section-এর link-গুলো। (ধাপ ১১-এর `flatMap` বুঝতে এটা মনে রাখুন।)
- **`asChild`:** `SidebarMenuButton` নিজে একটা `<button>` বানায়। কিন্তু আমাদের দরকার link (`<Link>`)। `asChild` দিলে সে নিজের button বানায় না, তার সব style আর আচরণ ভেতরের `<Link>`-এ দিয়ে দেয় (১০ নম্বর নোট: Radix-এ `asChild`)। ফলে দেখতে sidebar-এর button, কাজ করে link হিসেবে।
- **`isActive={pathname === item.url}`:** বর্তমান URL আর item-এর URL মিললে item active দেখায়। `data-[active=true]:shadow-[inset_3px_0_0_var(--star)]` দিয়ে active item-এর বাম পাশে হলুদ (`--star`) দাগ।
- **`SidebarRail`:** sidebar-এর কিনারায় একটা সরু অংশ, সেখানে click করলেও sidebar খোলে-বন্ধ হয়।
- **`{ ...props }`:** `<AppSidebar />`-কে বাইরে থেকে কোনো prop (যেমন `side="right"`) দিলে সেটা ভেতরের `<Sidebar>`-এ চলে যায়।

> ⚠️ সংশোধন (রং-এর নিয়ম): `text-white` আর `fill="#fff"` আমাদের নিয়মে দেওয়া যায় না (hex/সাদা শুধু ছবির ওপরে)। এখন চোখে কোনো সমস্যা নেই, কারণ আপনার `index.css`-এ sidebar-এর রং light আর dark দুই mode-এই গাঢ় নীল (`--sidebar: #1C1A3A`)। কিন্তু কখনো light mode-এ sidebar সাদা করলে "Phovia Tour" লেখা অদৃশ্য হয়ে যাবে। Token দিলে রং sidebar-এর সাথে নিজে মিলে যায়:
> - `text-white` → `text-sidebar-accent-foreground`
> - `fill="#fff"` → `className="fill-sidebar-primary-foreground"`

> নোট: আপনার `index.css`-এ `:root` আর `.dark` দুই জায়গাতেই `--sidebar…` variable-গুলো **দুইবার** লেখা আছে (প্রথমে সাদা/হালকা, তার নিচে গাঢ় নীল)। CSS-এ পরেরটাই কাজ করে, তাই প্রথম set কখনো ব্যবহার হয় না। কোনটা রাখবেন ঠিক করে অন্যটা মুছে দিলে পরে বিভ্রান্তি হবে না।

> নোট: Footer-এর "Log out" button এখনো কিছু করে না, শুধু দেখায়। ১৩ নম্বর নোটের Navbar-এর `handleLogout` (logout → `resetApiState`) এখানেও বসাতে হবে, সাথে logout-এর পরে `/login`-এ পাঠানো।

✅ **যাচাই:** Sidebar-এ তিনটা section; যে page-এ আছেন সেটার item active দেখায়।

---

## ধাপ ৮ — সমস্যা দেখা

🎯 **লক্ষ্য:** এই ব্যবস্থার দুটো সমস্যা চিনে নেওয়া, যেগুলো ভাগ গ আর ঘ-তে ঠিক করব।

✅ **যাচাই (এবং সমস্যা):**

**সমস্যা ১ — সবাই admin-এর menu দেখে:** `/user/bookings` খুলুন। Sidebar-এ দেখাচ্ছে Tours, Users, Divisions… অর্থাৎ **admin-এর** menu, কারণ `data.navMain`-এ শুধু admin-এর item লেখা।

**সমস্যা ২ — একই জিনিস দুই জায়গায়:** ধরুন sidebar-এ "Add tour type" নামে নতুন menu item যোগ করলাম। Link sidebar-এ চলে আসবে, কিন্তু click করলে **কিছুই দেখাবে না**, কারণ `routes/index.tsx`-এ ঐ path-এর route নেই। আবার route লিখতে ভুলে গেলে, বা route-এর path আর sidebar-এর url-এ একটা অক্ষর আলাদা হলে, ভাঙবে।

Teacher-এর সমাধান: **sidebar আর route দুটোই একটা জায়গা থেকে চালানো।** তাতে:
- এক জায়গায় লেখা, তাই মিল না হওয়ার (mis-manage) সুযোগ থাকে না।
- নতুন page যোগ করতে একটা ফাইল খুললেই হয়, এক ফাইল থেকে আরেক ফাইলে দৌড়াতে হয় না (context switching কম)।

---

# ভাগ গ: এক জায়গা থেকে Menu + Route

## ধাপ ৯ — Menu-র data আলাদা ফাইলে

🎯 **লক্ষ্য:** Sidebar-এর item-গুলো `AppSidebar`-এর বাইরে, `routes` folder-এ নিয়ে যাওয়া।

🛠️ **কাজ:**

```ts
// src/routes/adminSidebarItems.ts  (প্রথম রূপ — এখনো component নেই)
export const adminSidebarItems = [
  {
    title: "Dashboard",
    items: [{ title: "Overview", url: "/admin" }],
  },
  {
    title: "Catalogue",
    items: [
      { title: "Tours", url: "/admin/tours" },
      { title: "Tour types", url: "/admin/tour-types" },
      { title: "Divisions", url: "/admin/divisions" },
    ],
  },
];
```

`AppSidebar.tsx`-এ:

```tsx
import { adminSidebarItems } from "@/routes/adminSidebarItems";

const data = {
  navMain: adminSidebarItems,
};
```

💡 **`routes` folder-এ কেন:** এই ফাইলটা শুধু menu-র না, একটু পরে এখান থেকেই **route** তৈরি হবে। তাই route-এর ফাইলগুলোর পাশেই রাখা।

✅ **যাচাই:** Sidebar আগের মতোই দেখায় (data শুধু জায়গা বদলেছে)।

---

## ধাপ ১০ — Item-এ component যোগ আর type

🎯 **লক্ষ্য:** প্রতিটা item-কে জানানো, ঐ URL-এ **কোন page** দেখাবে। এখন item-এ route বানানোর সব তথ্য থাকবে।

💡 **আগে তুলনা করি:** route-এ কী লাগে আর sidebar item-এ কী আছে:

| | Route-এ লাগে | Sidebar item-এ আছে |
|---|---|---|
| Path | `path: "/admin/tours"` | `url: "/admin/tours"` ✅ |
| কোন page | `Component: AddTour` | **নেই** ❌ |
| Menu-র লেখা | লাগে না | `title: "Tours"` |

একটাই জিনিস কম: page (component)। Item-এ `component` যোগ করলেই একই data থেকে route বানানো যাবে।

🛠️ **কাজ ১ — type:**

```ts
// src/types/index.ts  (আগের type-গুলোর নিচে)
import type { ComponentType } from "react";

export interface ISidebarItem {
  title: string;
  items: {
    title: string;
    url: string;
    component: ComponentType;
  }[];
}
```

- **`ComponentType`:** React-এর type, মানে "যেকোনো React component" (function বা class)। লক্ষ্য করুন: এখানে `<AddTour />` (JSX, অর্থাৎ বানানো element) না, শুধু `AddTour` (component নিজে) রাখা হচ্ছে। Route-এর `Component`-ও ঠিক এটাই চায় (৫ নম্বর নোট: `Component` বনাম `element`)।
- **`items: {...}[]`:** শেষের `[]` মানে এমন object-এর একটা **array**।
- `import type`, কারণ `ComponentType` শুধু type।

🛠️ **কাজ ২ — item-এ component:**

```ts
// src/routes/adminSidebarItems.ts
import AddTour from "@/pages/Admin/AddTour";
import AddTourTypes from "@/pages/Admin/AddTourTypes";
import Overview from "@/pages/Admin/Overview";
import type { ISidebarItem } from "@/types";

export const adminSidebarItems: ISidebarItem[] = [
  {
    title: "Dashboard",
    items: [{ title: "Overview", url: "/admin", component: Overview }],
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

💡 **`: ISidebarItem[]` দেওয়ার সুবিধা:** কোনো item-এ `component` দিতে ভুলে গেলে, বা `url`-এর বানান ভুল করলে (`ulr`), TypeScript সাথে সাথে লাল দাগ দেয়। Route ভাঙার আগেই ভুল ধরা পড়ে।

💡 **Overview-এর `url: "/admin"`:** এটা parent-এর (`/admin`) সমান। ফলে কেউ শুধু `/admin` খুললেই Overview দেখায়। এটা আপনার নিজের যোগ করা, teacher-এর কোডে admin-এর এমন page ছিল না (ধাপ ১৯-এ এর গুরুত্ব দেখবেন)।

✅ **যাচাই:** কোনো TypeScript error নেই, sidebar আগের মতো দেখায়।

---

## ধাপ ১১ — `generateRoutes`: item থেকে route বানানো

🎯 **লক্ষ্য:** Sidebar-এর item-গুলোকে React Router-এর route-এ বদলে দেওয়া।

🛠️ **কাজ:**

```ts
// src/utils/generateRoutes.ts
import type { ISidebarItem } from "@/types";

export const generateRoutes = (sidebarItems: ISidebarItem[]) => {
  return sidebarItems.flatMap((section) =>
    section.items.map((route) => ({
      path: route.url,
      Component: route.component,
    })),
  );
};
```

### 💡 `flatMap` কেন (আপনার প্রশ্ন)

আমাদের data **দুই তলা**: বাইরে section-এর array, প্রতিটা section-এর ভেতরে আবার item-এর array।

```ts
[
  { title: "Dashboard",       items: [Overview] },
  { title: "Tour Management", items: [Tours, TourTypes] },
]
```

কিন্তু React Router-এর `children` চায় **এক তলা**, শুধু route-এর একটা সমান array:

```ts
[ OverviewRoute, ToursRoute, TourTypesRoute ]
```

**শুধু `map` দিলে কী হয়:**

```ts
sidebarItems.map((section) => section.items.map((route) => ({ ... })));

// ফলাফল: array-এর ভেতরে array ❌
[
  [ OverviewRoute ],
  [ ToursRoute, TourTypesRoute ],
]
```

বাইরের `map` প্রতিটা section-এর জন্য একটা জিনিস ফেরত দেয়, আর সেই জিনিসটা নিজেই একটা array (ভেতরের `map`-এর ফল)। ফলে array-এর ভেতরে array, React Router এটা চেনে না।

**`flatMap` দিলে:**

```ts
sidebarItems.flatMap((section) => section.items.map((route) => ({ ... })));

// ফলাফল: এক তলা ✅
[ OverviewRoute, ToursRoute, TourTypesRoute ]
```

`flatMap` = **`map` + `flat`**। প্রথমে `map`-এর মতো কাজ করে, তারপর ভেতরের array-গুলো খুলে (চ্যাপ্টা করে, flatten) এক array-তে ঢেলে দেয়। আপনি ঠিকই লিখেছিলেন: দুটো আলাদা array-কে এক array-তে নিয়ে আসে।

একটা ছোট উদাহরণ:

```ts
const boxes = [["🍎"], ["🍌", "🍇"]];

boxes.map((box) => box);      // [["🍎"], ["🍌", "🍇"]]   — বাক্সসহ
boxes.flatMap((box) => box);  // ["🍎", "🍌", "🍇"]       — বাক্স খুলে ফল এক ঝুড়িতে
```

এটা `.map(...).flat()` লেখার সমান, শুধু এক ধাপে।

> নোট: `flatMap` শুধু **এক তলা** খোলে। তিন তলা data (section → sub-section → item) হলে আরেকটা `flatMap` লাগবে। আমাদের দুই তলা, তাই একটাই যথেষ্ট।

### 💡 Item → Route-এ রূপান্তর

```ts
{ title: "Tours", url: "/admin/tours", component: AddTour }
            ↓  ({ path: route.url, Component: route.component })
{ path: "/admin/tours", Component: AddTour }
```

`title` বাদ পড়ে (route-এ লাগে না), `url` হয় `path`, `component` হয় `Component`।

💡 **এখানে absolute path কেন:** `url`-এ পুরো `/admin/tours` আছে, কারণ sidebar-এর `<Link>`-এ পুরো URL লাগে। Route-এও সেটাই বসছে, তাই child route-এ absolute path। এটা বৈধ, কারণ `/admin/tours` parent-এর `/admin` দিয়েই শুরু (ধাপ ৪-এর নিয়ম)। ভুল করে `url: "/tours"` লিখলে app চালুই হবে না।

✅ **যাচাই:** কোনো TypeScript error নেই। (ব্যবহার পরের ধাপে।)

---

## ধাপ ১২ — Route-এ `generateRoutes` বসানো

🎯 **লক্ষ্য:** Admin-এর route আর হাতে না লিখে item থেকে তৈরি করা।

🛠️ **কাজ:**

```tsx
// src/routes/index.tsx  (admin অংশ)
import { generateRoutes } from "@/utils/generateRoutes";
import { adminSidebarItems } from "./adminSidebarItems";

{
  Component: DashboardLayout,
  path: "/admin",
  children: [...generateRoutes(adminSidebarItems)],
},
```

এখন `Analytics`, `AddTour`-এর আলাদা import আর হাতে লেখা route `index.tsx` থেকে মুছে দেওয়া যায়, সব আসছে `adminSidebarItems` থেকে।

### 💡 `[...]` কেন দিতে হচ্ছে (আপনার প্রশ্ন)

`...` হলো **spread operator**: array-এর ভেতরের জিনিসগুলো বের করে ছড়িয়ে দেয়। আর বাইরের `[ ]` একটা নতুন array বানায়, যার ভেতরে ঐ ছড়ানো জিনিসগুলো বসে।

```ts
const routes = generateRoutes(adminSidebarItems);
// routes = [OverviewRoute, ToursRoute, TourTypesRoute]  ← আগে থেকেই array

[...routes]
// = [OverviewRoute, ToursRoute, TourTypesRoute]          ← একই জিনিসের নতুন array
```

অর্থাৎ **এখানে `[...]` আসলে লাগছে না**। `generateRoutes` নিজেই array ফেরত দেয় (ধাপ ১১-এর `flatMap`-এর কারণে), তাই সরাসরি লিখলেও একই কাজ:

```tsx
children: generateRoutes(adminSidebarItems),   // একই ফল
```

**তাহলে `[...]` কখন সত্যিই লাগে?** যখন generate-করা route-এর **সাথে আরও route জুড়তে** হয়। তখন সবাইকে একটা array-তে আনতে হয়:

```tsx
children: [
  { index: true, element: <Navigate to="/user/bookings" replace /> },  // হাতে লেখা একটা
  ...generateRoutes(userSidebarItems),                                  // + generate-করা সবগুলো
],
```

এখানে `...` না দিলে কী হতো:

```tsx
children: [
  { index: true, ... },
  generateRoutes(userSidebarItems),   // ❌ array-এর ভেতরে array
]
// = [ IndexRoute, [BookingsRoute] ]  ← ধাপ ১১-এর সেই "দুই তলা" সমস্যা আবার
```

`...` দিলে ভেতরের array খুলে route-গুলো সরাসরি বাইরের array-এ বসে: `[ IndexRoute, BookingsRoute ]`। ধাপ ১৯-এ user-এর route-এ ঠিক এটাই লাগবে। Admin-এ `[...]` রেখে দিলেও কোনো ক্ষতি নেই, বরং দুই জায়গায় একই ধরনের লেখা থাকে, আর পরে কিছু জুড়তে হলে সহজ।

✅ **যাচাই:**
- `/admin`, `/admin/tours`, `/admin/tour-types` সবগুলো ঠিক page দেখায়।
- এবার পরীক্ষা: `adminSidebarItems.ts`-এ একটা নতুন item যোগ করুন (যেমন Analytics)। `index.tsx` না ছুঁয়েই sidebar-এ link আসে **আর** click করলে page খোলে। 🎉 সমস্যা ২ সমাধান।

### ➕ আপনার যোগ করা: `lazy` দিয়ে Analytics

আপনি `adminSidebarItems.ts`-এ Analytics এভাবে যোগ করেছেন:

```ts
import { lazy } from "react";

const Analytics = lazy(() => import("@/pages/Admin/Analytics"));

// ...
{ title: "Analytics", url: "/admin/analytics", component: Analytics },
```

💡 **`lazy` কী করে:** সাধারণ `import` দিলে Analytics-এর কোড app-এর মূল JavaScript ফাইলেই ঢুকে যায়, প্রথমবার website খুললেই সবাই সেটা download করে, এমনকি যে কখনো admin page-এ যাবে না সে-ও। `lazy(() => import(...))` দিলে Vite Analytics-কে **আলাদা ছোট ফাইলে** রাখে, আর সেটা download হয় **শুধু যখন কেউ ঐ page খোলে**। Chart-এর মতো ভারী page-এর (recharts বড় library) জন্য এটা খুব কাজের।

> ⚠️ সংশোধন (শব্দের মানে): নোটে teacher "code splitting" বলতে বুঝিয়েছেন **route-এর ফাইল ভাগ করা** (এক বড় `index.tsx`-কে role অনুযায়ী ছোট ফাইলে ভাগ), যাতে maintain সহজ হয়। React/Vite-এর জগতে "code splitting" শব্দটা সাধারণত অন্য জিনিস বোঝায়: **JavaScript bundle ভাগ করা**, ঠিক যেটা `lazy` করে। দুটোই "ভাগ করা", কিন্তু একটা কোড সাজানোর জন্য (developer-এর সুবিধা), অন্যটা download কমানোর জন্য (user-এর সুবিধা)। কেউ "code splitting" বললে সাধারণত দ্বিতীয়টাই বোঝায়।

> ➕ সংযোজন: `lazy` page-এর ফাইল download হতে এক মুহূর্ত লাগে। ঐ সময়ে কী দেখাবে, সেটা `<Suspense>` দিয়ে বলে দেওয়া যায়। `DashboardLayout`-এ `<Outlet />`-কে মুড়ে দিলে সব dashboard page-এর জন্য একবারেই হয়ে যায়:
> ```tsx
> import { Suspense } from "react";
>
> <Suspense fallback={<p className="text-muted-foreground">Loading...</p>}>
>   <Outlet />
> </Suspense>
> ```
> আপনার কোড এখন চলছে (React Router নিজে navigation-এর সময় আগের page ধরে রাখে), তাই এটা বাধ্যতামূলক না। কিন্তু loading-এর সময় কী দেখাবে সেটা নিজের হাতে রাখতে চাইলে এটা দেবেন।

---

# ভাগ ঘ: Role অনুযায়ী Menu

## ধাপ ১৩ — Role-এর constant

🎯 **লক্ষ্য:** `"ADMIN"`, `"USER"`-এর মতো string এক জায়গায় রাখা।

🛠️ **কাজ:**

```ts
// src/constants/role.ts
export const role = {
  superAdmin: "SUPER_ADMIN",
  admin: "ADMIN",
  user: "USER",
};
```

💡 **কেন (teacher-এর কথা):** যখনই কোনো string-এর সাথে তুলনা করতে হবে, সেটা কেন্দ্রীয় এক জায়গা থেকে নেওয়া ভালো:

```ts
if (userRole === "SUPER_ADMIN") ...      // ❌ হাতে লেখা string
if (userRole === role.superAdmin) ...    // ✅ constant থেকে
```

| | হাতে লেখা `"SUPER_ADMIN"` | `role.superAdmin` |
|---|---|---|
| বানান ভুল (`"SUPER_ADMN"`) | কোনো error নেই, চুপচাপ কখনো মিলবে না | TypeScript সাথে সাথে বলে `superAdmn` নেই |
| Editor | কোনো suggestion নেই | `role.` লিখলেই সব role-এর list |
| Backend নাম বদলালে (`"SUPERADMIN"`) | পুরো project-এ খুঁজে খুঁজে বদলাতে হয় | এই একটা ফাইলে বদলালেই সব জায়গায় |

> ➕ সংযোজন: শেষে `as const` যোগ করলে আরও ভালো হয়:
> ```ts
> export const role = {
>   superAdmin: "SUPER_ADMIN",
>   admin: "ADMIN",
>   user: "USER",
> } as const;
>
> export type TRole = (typeof role)[keyof typeof role]; // "SUPER_ADMIN" | "ADMIN" | "USER"
> ```
> `as const` ছাড়া TypeScript `role.admin`-কে শুধু "কোনো একটা `string`" ভাবে। `as const` দিলে ঠিক `"ADMIN"` ভাবে। তাতে উপরের লাইনে constant থেকেই `TRole` বানানো যায়, ধাপ ১৫-এ `TRole` আলাদা করে হাতে লিখতে হয় না। নতুন role (যেমন `"GUIDE"`) যোগ করলে শুধু constant-এ লিখলেই type-ও বদলে যায়। আপনার এখনকার কোডও ঠিক চলছে, এটা শুধু একটা উন্নতি।

✅ **যাচাই:** কোনো error নেই।

---

## ধাপ ১৪ — User-এর sidebar item আর route

🎯 **লক্ষ্য:** Admin-এর মতো user-এর জন্যও item-এর ফাইল।

🛠️ **কাজ:**

```ts
// src/routes/userSidebarItems.ts
import Bookings from "@/pages/User/Bookings";
import type { ISidebarItem } from "@/types";

export const userSidebarItems: ISidebarItem[] = [
  {
    title: "History",
    items: [{ title: "Bookings", url: "/user/bookings", component: Bookings }],
  },
];
```

```tsx
// src/routes/index.tsx  (user অংশ)
import { userSidebarItems } from "./userSidebarItems";

{
  Component: DashboardLayout,
  path: "/user",
  children: [...generateRoutes(userSidebarItems)],
},
```

> নোট: আপনার কোডে `import { type ISidebarItem } from "@/types"` লেখা, আর `adminSidebarItems.ts`-এ `import type { ISidebarItem }`। দুটোই কাজ করে। পুরো import শুধু type হলে `import type { ... }` লেখাই পরিষ্কার, তাই দুই ফাইলে একই ধরন রাখুন।

✅ **যাচাই:** `/user/bookings` খোলে। কিন্তু sidebar-এ এখনো **admin-এর** menu (সমস্যা ১), কারণ `AppSidebar` এখনো সরাসরি `adminSidebarItems` নিচ্ছে। শুধু `userSidebarItems` দিয়ে বদলালে তখন admin-এর menu চলে যাবে। দরকার: **কে login করেছে** দেখে ঠিক করা। পরের ধাপ।

---

## ধাপ ১৫ — `getSidebarItems`: role দেখে menu বাছাই

🎯 **লক্ষ্য:** Role দিলেই সেই role-এর menu ফেরত দেয়, এমন একটা function।

🛠️ **কাজ ১ — role-এর type** (যদি ধাপ ১৩-এর `as const` ব্যবহার না করেন):

```ts
// src/types/index.ts
export type TRole = "SUPER_ADMIN" | "ADMIN" | "USER";
```

**`|` মানে "অথবা"।** `TRole` শুধু এই তিনটার একটা হতে পারে। `getSidebarItems("MANAGER")` লিখলে TypeScript error দেবে।

🛠️ **কাজ ২ — function:**

```ts
// src/utils/getSidebarItems.ts
import { role } from "@/constants/role";
import { adminSidebarItems } from "@/routes/adminSidebarItems";
import { userSidebarItems } from "@/routes/userSidebarItems";
import type { TRole } from "@/types";

export const getSidebarItems = (userRole: TRole) => {
  switch (userRole) {
    case role.superAdmin:
      return [...adminSidebarItems];
    case role.admin:
      return [...adminSidebarItems];
    case role.user:
      return [...userSidebarItems];
    default:
      return [];
  }
};
```

### 💡 `switch` কীভাবে কাজ করে

`switch` একটা মান (`userRole`) নিয়ে একে একে `case`-এর সাথে মেলায়। যেটা মিলে, সেটার কোড চলে। `return` দেওয়ায় সেখানেই function শেষ, পরের `case`-এ যায় না।

| `userRole` | ফেরত দেয় |
|---|---|
| `"SUPER_ADMIN"` | admin-এর menu |
| `"ADMIN"` | admin-এর menu |
| `"USER"` | user-এর menu |
| অন্য কিছু / `undefined` (login করা নেই, বা data এখনো আসেনি) | `[]` — খালি menu |

**`default` কেন জরুরি:** কোনো `case` না মিললে এটা চলে। না দিলে function `undefined` ফেরত দিত, আর `AppSidebar`-এ `undefined.map(...)` করতে গিয়ে পুরো page ভেঙে যেত। `[]` দিলে `.map` চলে, শুধু কিছু দেখায় না।

### 💡 `[...adminSidebarItems]` আর super admin

এখানেও spread (ধাপ ১২)। শুধু একটা array ফেরত দিতে `[...]` লাগে না, `return adminSidebarItems;`-ও চলত। কিন্তু **একাধিক array জুড়তে** এটা দরকার। Teacher-এর comment অনুযায়ী: super admin-কে যদি user-এর menu-ও দেখাতে হয়, শুধু comma দিয়ে আরেকটা spread বসালেই হয়:

```ts
case role.superAdmin:
  return [...adminSidebarItems, ...userSidebarItems];
  // = [Dashboard, Tour Management, History]  ← এক তলা, সব section একসাথে
```

`...` ছাড়া `[adminSidebarItems, userSidebarItems]` লিখলে আবার array-এর ভেতরে array হয়ে যেত।

> নোট: super admin-কে user-এর menu দিলে route-ও লাগবে, অর্থাৎ সে `/user/bookings`-এ গেলে page খুলতে হবে। Route তো এখন role দেখে না (সবার জন্য খোলা, ধাপ ২০ দেখুন), তাই এখন সমস্যা নেই।

> ➕ সংযোজন: `super admin` আর `admin` একই জিনিস ফেরত দিচ্ছে, তাই দুটো `case` একসাথে লেখা যায় (একটার নিচে আরেকটা, মাঝে কোনো `return` ছাড়া):
> ```ts
> case role.superAdmin:
> case role.admin:
>   return [...adminSidebarItems];
> ```
> প্রথম `case` মিললে কোড নিচে "গড়িয়ে" পরের `case`-এর `return`-এ পৌঁছায়। তবে পরে super admin-কে আলাদা কিছু দিতে হবে বলে teacher আলাদা রেখেছেন, সেটাও ঠিক।

✅ **যাচাই:** কোনো error নেই। (ব্যবহার পরের ধাপে।)

---

## ধাপ ১৬ — পরীক্ষা: role hardcode করে দেখা

🎯 **লক্ষ্য:** `getSidebarItems` ঠিক কাজ করছে কিনা, login ছাড়াই দেখা।

🛠️ **কাজ:** `AppSidebar.tsx`-এ সাময়িকভাবে:

```tsx
import { getSidebarItems } from "@/utils/getSidebarItems";

const data = {
  navMain: getSidebarItems("ADMIN"),
};
```

✅ **যাচাই:** `"ADMIN"` দিলে admin-এর menu, `"USER"` দিলে user-এর menu। কিন্তু এটা hardcode: যেই login করুক, একই menu। এখন আসল user-এর role বসাব।

---

## ধাপ ১৭ — Login করা user-এর role দিয়ে menu

🎯 **লক্ষ্য:** যে login করেছে, তার role অনুযায়ী sidebar।

🛠️ **কাজ:**

```tsx
// src/components/layout/AppSidebar.tsx  (component-এর ভেতরে)
import { useUserInfoQuery } from "@/redux/features/auth/auth.api";
import { getSidebarItems } from "@/utils/getSidebarItems";

export function AppSidebar({ ...props }: React.ComponentProps<typeof Sidebar>) {
  const { pathname } = useLocation();
  const { data: userData } = useUserInfoQuery(undefined);

  const data = {
    navMain: getSidebarItems(userData?.data?.role),
  };

  // ...বাকি JSX আগের মতো
}
```

💡 **কীভাবে কাজ করে:**
- `useUserInfoQuery` → `/user/me` (১৩ নম্বর নোট)। Backend database থেকে logged-in user-এর তথ্য দেয়, তার ভেতরে `role` আছে।
- `userData?.data?.role` → `"ADMIN"` বা `"USER"` → `getSidebarItems` সেই role-এর menu দেয়।
- **`data: userData` (নাম বদল):** hook-এর ফলাফলের নাম `data`, কিন্তু এই component-এ নিচে আগে থেকেই `const data = { navMain }` আছে। একই নামে দুটো variable রাখা যায় না, তাই hook-এর `data`-কে `userData` নামে নেওয়া হলো।
- **একই query দুই জায়গায় (Navbar আর Sidebar), request কিন্তু একটা:** RTK Query-র cache (১৩ নম্বর নোট) একই data দুই component-কে দেয়, দুইবার backend-এ যায় না।

> নোট: Page প্রথম খোলার মুহূর্তে `/user/me`-র উত্তর আসেনি, তখন `role` হয় `undefined` → `default` → খালি sidebar। এক মুহূর্ত পরে data এলে menu দেখায়। এটা স্বাভাবিক, ধাপ ১৫-এর `default` না থাকলে এখানেই page ভেঙে যেত।

> নোট (type): `userInfo` endpoint-এ এখনো type দেওয়া নেই (১৩ নম্বর নোটে বলা `builder.query<IResponse<IUser>, void>`), তাই `userData` হলো `any`, আর `getSidebarItems(userData?.data?.role)` কোনো error দেয় না। পরে type দিলে TypeScript বলবে `role` `undefined`-ও হতে পারে। তখন function-এর parameter `userRole?: TRole` করে দিলেই হবে (`default` আগেই সেটা সামলায়)।

✅ **যাচাই:**
- Admin account দিয়ে login → `/admin`-এ admin-এর menu।
- User account দিয়ে login → `/user/bookings`-এ শুধু "History › Bookings"। 🎉 সমস্যা ১ সমাধান।

---

## ধাপ ১৮ — Navbar-এ role অনুযায়ী "Dashboard" link

🎯 **লক্ষ্য:** Public site-এর Navbar থেকে dashboard-এ যাওয়ার link, আর সেটা role অনুযায়ী ঠিক জায়গায় নিয়ে যাবে।

🛠️ **কাজ:** `navLinks`-এ প্রতিটা link-এর সাথে কে দেখবে (`role`):

```tsx
// src/components/layout/Navbar.tsx
import { role } from "@/constants/role";
import { Fragment } from "react";

const navLinks = [
  { href: "/", label: "Home", role: "PUBLIC" },
  { href: "/about", label: "About", role: "PUBLIC" },
  { href: "/admin", label: "Dashboard", role: role.admin },
  { href: "/admin", label: "Dashboard", role: role.superAdmin },
  { href: "/user", label: "Dashboard", role: role.user },
];
```

Desktop nav (mobile-এও একই, শুধু `className` আলাদা):

```tsx
{navLinks.map((link, index) => (
  <Fragment key={index}>
    {link.role === "PUBLIC" && (
      <NavLink to={link.href} className="...">
        {link.label}
      </NavLink>
    )}

    {link.role === data?.data?.role && (
      <NavLink to={link.href} className="...">
        {link.label}
      </NavLink>
    )}
  </Fragment>
))}
```

💡 **কীভাবে কাজ করে:**
- **`"PUBLIC"`:** সবাই দেখে (login থাকুক বা না থাকুক)।
- **`link.role === data?.data?.role`:** link-এর role আর logged-in user-এর role মিললে তবেই দেখায়। Admin login করলে শুধু `role.admin`-এর Dashboard (`/admin`), user করলে শুধু `/user`-এরটা। Login না থাকলে `data` `undefined`, কোনোটাই মেলে না, Dashboard দেখায় না।
- **একই link তিনবার কেন:** তিন role-এর Dashboard আলাদা জায়গায় যায় (admin আর super admin `/admin`-এ, user `/user`-এ)। প্রতিটা user-এর একটা role, তাই তিনটার মধ্যে সর্বোচ্চ একটাই দেখাবে।
- **`&&`:** বাম পাশ সত্য হলে তবেই ডান পাশের JSX দেখায়, মিথ্যা হলে কিছুই না।
- **`Fragment key={index}`:** প্রতিটা link-এর জন্য দুটো শর্ত লেখা, তাই একটা মোড়ক লাগে। `<>…</>`-এ `key` দেওয়া যায় না, তাই পুরো নাম `Fragment` লিখে তাতে `key`। (Admin আর super admin-এর Dashboard-এর `label` একই, তাই `key={link.label}` দিলে key মিলে যেত; `index` সবসময় আলাদা।)

> নোট: আপনার কোডে `Fragment`-এ `key` দেওয়ার পরেও ভেতরের `NavLink`-এ `key={link.label}` রয়ে গেছে। `map`-এর সরাসরি সন্তান `Fragment`, তাই React শুধু ওটার `key` দেখে। ভেতরের `key` লাগে না, মুছে দিতে পারেন।

> ➕ সংযোজন (সহজ করার উপায়): একই `NavLink` দুইবার লেখার বদলে আগে `filter` দিয়ে দেখানোর মতো link বেছে নেওয়া যায়, তারপর একবার `map`:
> ```tsx
> const visibleLinks = navLinks.filter(
>   (link) => link.role === "PUBLIC" || link.role === data?.data?.role,
> );
>
> {visibleLinks.map((link) => (
>   <NavLink key={link.href} to={link.href} className="...">
>     {link.label}
>   </NavLink>
> ))}
> ```
> ফল একই, কিন্তু লম্বা `className` desktop আর mobile-এ একবার করেই লিখতে হয়, আর `Fragment` লাগে না। Teacher-এর পদ্ধতিও ঠিক চলে।

✅ **যাচাই:**
- Login ছাড়া: Home, About, কোনো Dashboard নেই।
- Admin login: Dashboard → `/admin` (Overview)।
- User login: Dashboard → `/user` → … **ফাঁকা page!** পরের ধাপে কারণ।

---

## ধাপ ১৯ — `/user`-এ ফাঁকা page: index redirect

🎯 **লক্ষ্য:** কেউ শুধু `/user` খুললে নিজে থেকে `/user/bookings`-এ চলে যাবে।

### 💡 সমস্যা কেন

Navbar-এর Dashboard link যায় `/user`-এ (শুধু parent-এ)। কিন্তু user-এর route-গুলো দেখুন:

```
/user            → DashboardLayout   ✅
/user/bookings   → Bookings          ✅
/user            → (কোনো child নেই) ❌
```

`/user` খুললে layout (sidebar) দেখায়, কিন্তু `<Outlet />`-এ বসানোর মতো কোনো child route মেলে না, তাই মাঝখান ফাঁকা।

**Admin-এ কেন হচ্ছে না:** আপনি admin-এর জন্য `{ title: "Overview", url: "/admin", component: Overview }` বানিয়েছিলেন (ধাপ ১০)। এই child-এর path parent-এর সমান, তাই `/admin` খুললেই Overview দেখায়। User-এর এমন কোনো page নেই। Teacher-এর admin বা user কোনোটাতেই এমন page ছিল না, তাই তাঁর দুটোই ফাঁকা দেখাচ্ছিল।

### 🛠️ কাজ: index route দিয়ে redirect

```tsx
// src/routes/index.tsx  (user অংশ, চূড়ান্ত)
import { createBrowserRouter, Navigate } from "react-router";

{
  Component: DashboardLayout,
  path: "/user",
  children: [
    { index: true, element: <Navigate to="/user/bookings" replace /> },
    ...generateRoutes(userSidebarItems),
  ],
},
```

💡 **লাইনগুলোর মানে:**
- **`index: true`:** "parent-এর ঠিক নিজের URL-এ (`/user`) এলে এটা দেখাও"। Index route-এর কোনো `path` থাকে না, parent-এর path-ই তার path।
- **`<Navigate to="/user/bookings" />`:** এটা কিছু দেখায় না, render হওয়ামাত্র অন্য URL-এ পাঠিয়ে দেয়।
- **`...generateRoutes(...)`:** হাতে লেখা index route-এর সাথে generate-করা route জোড়া, এখানেই spread সত্যিই কাজে লাগল (ধাপ ১২)।

> ⚠️ সংশোধন: Teacher-এর কোডে `<Navigate to="/user/bookings" />`-এ `replace` ছিল না। `replace` না দিলে history-তে `/user` থেকে যায়: user `/user/bookings`-এ এসে Back চাপলে `/user`-এ যায়, সেখান থেকে আবার সাথে সাথে `/user/bookings`-এ… Back বাটন আটকে যায়। `replace` দিলে `/user`-এর জায়গায় `/user/bookings` বসে, Back চাপলে আগের page-এ (যেমন Home) ফেরে। (১৩ নম্বর নোটের Verify page-এর `replace`-এর মতো একই কারণ।)

💡 **এখানে `Component` না, `element` কেন:** আমাদের নিয়মে route-এ `Component: X` লিখি, JSX না (৫ নম্বর নোট)। কিন্তু `Component`-এ শুধু component-এর নাম দেওয়া যায়, **prop দেওয়া যায় না**। `Navigate`-কে কোথায় পাঠাবে (`to`) সেটা prop হিসেবে বলতেই হবে। তাই এখানে `element: <Navigate to="..." />` (JSX)। এটা জেনে-বুঝে করা ব্যতিক্রম। আর এই JSX-এর জন্যই ফাইলটা `.tsx` (ধাপ ৩)।

> নোট: Admin-এর মতো user-এর জন্যও একটা Overview page বানিয়ে `url: "/user"` দিলে redirect লাগত না। কোনটা ভালো, সেটা ডিজাইনের সিদ্ধান্ত: user-এর জন্য আলাদা overview দরকার না হলে redirect-ই সহজ।

✅ **যাচাই:**
- User login → Navbar-এর Dashboard → নিজে থেকে `/user/bookings`, Bookings page দেখায়।
- Address bar-এ `/user` লিখুন → `/user/bookings`-এ যায়।
- এরপর Back চাপলে আগের page-এ ফেরে, `/user`-এ আটকে থাকে না।

---

## ধাপ ২০ — ⚠️ জরুরি: Menu লুকানো মানে page আটকানো না

🎯 **লক্ষ্য:** এখনকার ব্যবস্থার একটা নিরাপত্তার ফাঁক জেনে রাখা, যেটা পরে ঠিক করতে হবে।

✅ **যাচাই (পরীক্ষা):** User account দিয়ে login করে address bar-এ সরাসরি লিখুন `/admin/tours`। Page **খুলে যায়**। Sidebar-এ admin-এর menu নেই ঠিকই, কিন্তু `AddTour` page ঠিকই দেখায়। Login ছাড়াও খোলে।

💡 **কারণ:** এই নোটে আমরা শুধু ঠিক করেছি **কোন menu দেখাবে**। কিন্তু route-গুলো সবার জন্য খোলা, কেউ route-কে বলেনি "শুধু admin ঢুকতে পারবে"। Link লুকানো মানে দরজার সাইনবোর্ড সরানো, দরজায় তালা না।

**কী লাগবে (পরের পাঠে):** ৫ নম্বর নোটে বলা **Protected route / Higher Order Component (HOC)** — একটা মোড়ক (যেমন `withAuth(Component, role)`) যেটা page দেখানোর আগে `/user/me` দেখে: login না থাকলে `/login`-এ পাঠায়, role না মিললে "unauthorized" দেখায়। আমাদের `generateRoutes` থাকায় এটা এক জায়গাতেই বসানো যাবে, প্রতিটা page-এ আলাদা করে না।

> নোট: আসল নিরাপত্তা সবসময় **backend**-এ। Frontend-এ page আটকালেও কেউ Postman দিয়ে সরাসরি API-তে যেতে পারে। তাই backend-এর `checkAuth(role.admin)` (১৩ নম্বর নোট) আসল তালা; frontend-এর protection শুধু ভুল page না দেখানোর জন্য, user-এর সুবিধা।

---

# চূড়ান্ত ফাইল

ধাপে ধাপে বদলানো দুটো মূল ফাইলের শেষ রূপ। (`DashboardLayout.tsx` ধাপ ৬-এ, `generateRoutes.ts` ধাপ ১১-এ, `getSidebarItems.ts` ধাপ ১৫-এ, `userSidebarItems.ts` ধাপ ১৪-এ আগেই সম্পূর্ণ।)

### `src/routes/index.tsx`

```tsx
import App from "@/App";
import DashboardLayout from "@/components/layout/DashboardLayout";
import About from "@/pages/About";
import LoginPage from "@/pages/Login";
import RegisterPage from "@/pages/Register";
import Verify from "@/pages/verify";
import { generateRoutes } from "@/utils/generateRoutes";
import { createBrowserRouter, Navigate } from "react-router";
import { adminSidebarItems } from "./adminSidebarItems";
import { userSidebarItems } from "./userSidebarItems";

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
  { Component: RegisterPage, path: "/signup" }, // Navbar-এর দুই link-এর সাথে নাম মিলিয়ে নিন (ধাপ ৩)
  { Component: Verify, path: "/verify" },
]);
```

### `src/routes/adminSidebarItems.ts`

```ts
import AddTour from "@/pages/Admin/AddTour";
import AddTourTypes from "@/pages/Admin/AddTourTypes";
import Overview from "@/pages/Admin/Overview";
import type { ISidebarItem } from "@/types";
import { lazy } from "react";

const Analytics = lazy(() => import("@/pages/Admin/Analytics"));

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

### নতুন admin page যোগ করার নিয়ম (এখন থেকে)

1. `pages/Admin/`-এ page বানান (যেমন `Divisions.tsx`)।
2. `adminSidebarItems.ts`-এ ঠিক section-এ এক লাইন: `{ title: "Divisions", url: "/admin/divisions", component: Divisions }`।
3. শেষ। Route আর sidebar link দুটোই তৈরি। `routes/index.tsx` ছুঁতে হবে না।

`url` সবসময় `/admin/` দিয়ে শুরু হতে হবে (ধাপ ৪-এর absolute path-এর নিয়ম), নাহলে app চালু হবে না। User-এর page-এর জন্য একই নিয়ম `userSidebarItems.ts`-এ, `/user/` দিয়ে।

---

# শেষ Check-list

**Route ও Layout**
- [ ] `/admin` → Overview, sidebar সহ; ☰ চাপলে sidebar খোলে-বন্ধ হয়
- [ ] `/admin/analytics`, `/admin/tours`, `/admin/tour-types` সব খোলে
- [ ] `/user` → নিজে থেকে `/user/bookings`; Back চাপলে `/user`-এ আটকায় না
- [ ] `adminSidebarItems.ts`-এ নতুন item দিলে `index.tsx` না ছুঁয়েই link আর page দুটোই আসে

**Role**
- [ ] Admin login → sidebar-এ admin-এর menu; Navbar-এ Dashboard → `/admin`
- [ ] User login → sidebar-এ শুধু Bookings; Navbar-এ Dashboard → `/user/bookings`
- [ ] Login ছাড়া Navbar-এ Dashboard link নেই
- [ ] Sidebar-এ যে page খোলা, সেটার item active (বাম পাশে হলুদ দাগ)

**জেনে রাখা (এখনো ঠিক হয়নি)**
- [ ] User login করে `/admin/tours` লিখলে এখনো খোলে → Protected route পরের পাঠে (ধাপ ২০)
- [ ] Sidebar-এর "Log out" এখনো কাজ করে না (ধাপ ৭)
- [ ] Mobile menu-র Sign up আর route-এর নাম মেলানো (ধাপ ৩)
