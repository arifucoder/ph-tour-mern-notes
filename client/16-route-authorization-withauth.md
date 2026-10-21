# Frontend Notes — ১৬: Route Authorization — `withAuth` (HOC) দিয়ে Page-এর দরজায় পাহারা

১৪ নম্বর নোটের শেষ ধাপে (ধাপ ২০) একটা ফাঁক দেখেছিলাম: sidebar-এ admin-এর menu লুকানো থাকলেও, user (এমনকি login ছাড়াও কেউ) address bar-এ `/admin/tours` লিখলে page খুলে যায়। সেখানে বলেছিলাম, link লুকানো মানে দরজার সাইনবোর্ড সরানো, দরজায় তালা দেওয়া না। এই নোটে সেই তালা বসাব।

প্রতিটা ধাপে আগের মতো:

- 🎯 **লক্ষ্য**
- 🛠️ **কাজ**
- 💡 **কেন**
- ✅ **যাচাই**

নোটের মূল অংশ teacher-এর ক্রমে, teacher-এর কোড দিয়ে। আপনি যে সমস্যাটা খেয়াল করেছেন (redirect হওয়ার আগে এক ঝলক protected content দেখা যায়), সেটার কারণ আর সমাধান আছে **শেষে "আমার পরামর্শ" অংশে**, আপনার অনুরোধমতো।

> নোট: আপনার project-এর version মেনে লেখা (React 19, React Router 8, shadcn `radix-nova`)।

---

## ভূমিকা: Authentication বনাম Authorization

দুটো শব্দ প্রায় একই শোনায়, কিন্তু প্রশ্ন আলাদা:

| | Authentication | Authorization |
|---|---|---|
| প্রশ্ন | **তুমি কে?** | **তুমি কী করতে পারো?** |
| উদাহরণ | Login: email + password দিয়ে প্রমাণ করা আমি Rahim | Rahim `USER`, তাই `/admin`-এ ঢুকতে পারবে না |
| ব্যর্থ হলে | `/login`-এ পাঠাও | `/unauthorized`-এ পাঠাও |
| আমরা কোথায় করেছি | ১৩ নম্বর নোট (login, cookie, `/user/me`) | **এই নোট** |

অফিসের উদাহরণ: গেটে ID card দেখানো = authentication। ID card থাকলেও server room-এ ঢুকতে পারবেন কিনা, সেটা আপনার পদ = authorization।

### ধাপের মানচিত্র

| ভাগ | ধাপ | শেষে কী হবে |
|---|---|---|
| **ক. HOC বোঝা** | ১–২ | একটা component-কে মুড়ে নতুন component বানাতে পারব |
| **খ. পাহারা বসানো** | ৩–৫ | Login ছাড়া → `/login`; ভুল role → `/unauthorized` |
| **গ. Unauthorized page ও Route** | ৬–৭ | সব dashboard route সুরক্ষিত |
| **আমার পরামর্শ** | — | এক ঝলক content দেখা বন্ধ, একাধিক role, আরও ভালো উপায় |

---

# ভাগ ক: HOC বোঝা

## ধাপ ১ — Higher Order Component (HOC) কী

🎯 **লক্ষ্য:** একটা function বানানো, যেটা কোনো component নেয়, তার চারপাশে পাহারা বসিয়ে নতুন component ফেরত দেয়।

### 💡 ধারণা

**Higher Order Function:** এমন function, যেটা অন্য function নেয় বা ফেরত দেয় (যেমন `map` একটা function নেয়)।
**Higher Order Component (HOC):** এমন function, যেটা একটা **component নেয়** আর একটা **নতুন component ফেরত দেয়**।

```
withAuth( DashboardLayout )  ──→  AuthWrapper  (নতুন component)
          ↑ ভেতরের আসল জিনিস        ↑ আগে পাহারা, তারপর DashboardLayout দেখায়
```

উপহারের মোড়কের মতো: ভেতরের জিনিস (component) বদলায় না, শুধু বাইরে একটা মোড়ক যোগ হয়। আমাদের মোড়কটা একজন **পাহারাদার**: দরজা খোলার আগে দেখে নেয় ঢোকার অনুমতি আছে কিনা।

**কেন HOC:** আমাদের অনেক page সুরক্ষিত করতে হবে। প্রতিটা page-এর ভেতরে একই "login আছে? role ঠিক?" কোড লিখলে একই জিনিস বারবার, আর একটা ভুলে গেলে ফাঁক। HOC-এ একবার লিখে যেকোনো component-কে মুড়ে দেওয়া যায়।

### 🛠️ কাজ (প্রথম রূপ — শুধু দেখার জন্য)

```tsx
// src/utils/withAuth.tsx
import { useUserInfoQuery } from "@/redux/features/auth/auth.api";
import type { ComponentType } from "react";

export const withAuth = (Component: ComponentType) => {
  return function AuthWrapper() {
    const { data } = useUserInfoQuery(undefined);
    console.log("Inside withAuth", data);
    return <Component />;
  };
};
```

পরীক্ষার জন্য `routes/index.tsx`-এ About-কে মুড়ে দিন:

```tsx
{
  Component: withAuth(About),
  path: "about",
},
```

> ⚠️ সংশোধন: নোটে পরীক্ষাটা ছিল `withAuth(About, null)`। চূড়ান্ত কোডে দ্বিতীয় parameter-এর type `TRole` আর সেটা optional (`?`), তাই সেখানে `null` দিলে TypeScript error দেবে (`null` আর "না দেওয়া" এক জিনিস না)। Role না লাগলে দ্বিতীয় parameter একদম বাদ দিন: `withAuth(About)`।

✅ **যাচাই:**
- কোনো error নেই, About আগের মতোই দেখায়।
- About-এ গেলে console-এ `Inside withAuth` আর user-এর data আসে।
- Home বা অন্য page-এ গেলে log আসে **না**, কারণ শুধু About-কেই মোড়ানো হয়েছে। মোড়ক শুধু তখনই চলে, যখন মোড়ানো component দেখানো হয়।

দেখা হলে About থেকে `withAuth` সরিয়ে দিন (About সবার জন্য খোলা)।

---

## ধাপ ২ — ভেতরে আরেকটা function কেন (Hook-এর নিয়ম)

🎯 **লক্ষ্য:** `withAuth`-এর গঠন কেন এমন, সেটা বোঝা।

### 💡 সমস্যা

আমরা `withAuth`-এর ভেতরে `useUserInfoQuery` (একটা hook) ডাকতে চাই। কিন্তু React-এর নিয়ম:

> **Hook শুধু React component-এর ভেতরে (বা অন্য custom hook-এর ভেতরে) ডাকা যায়।**

`withAuth` নিজে একটা সাধারণ function, component না। এটা route ফাইলে **একবার** চলে, app চালুর সময়, React যখন কিছু render-ই করছে না। এখানে সরাসরি hook ডাকলে React error দেবে।

```tsx
// ❌ ভুল
export const withAuth = (Component: ComponentType) => {
  const { data } = useUserInfoQuery(undefined);   // সাধারণ function-এ hook
  return Component;
};
```

### 💡 সমাধান

`withAuth` ভেতরে আরেকটা function বানিয়ে ফেরত দেয়: `AuthWrapper`। এটা একটা **আসল component** (বড় হাতের নাম, JSX ফেরত দেয়, React এটাকে render করে)। তাই এর ভেতরে যেকোনো hook ডাকা যায়।

```
withAuth  (সাধারণ function, একবার চলে)
   └── return AuthWrapper  (component, প্রতিবার page দেখানোর সময় React চালায়)
            ├── useUserInfoQuery()   ✅ এখানে hook চলে
            └── return <Component />
```

### 💡 `Component` বড় হাতের `C` দিয়ে কেন

JSX-এ `<component />` (ছোট হাতের) লিখলে React ভাবে এটা HTML tag, যেমন `<div>` বা `<span>`, আর `<component>` নামে একটা অচেনা HTML element বানায়। বড় হাতের `<Component />` দেখলে React বোঝে এটা একটা React component, তাই ঐ নামের variable-এর ভেতরের component চালায়। তাই parameter-এর নামই বড় হাতের রাখা হয়েছে।

### 💡 `ComponentType`

React-এর type: "যেকোনো component" (১৪ নম্বর নোটের `ISidebarItem`-এও এটা ব্যবহার করেছি)। Lazy component-ও এর মধ্যে পড়ে, তাই lazy page-কেও `withAuth` দিয়ে মোড়া যায়।

> নোট (জরুরি নিয়ম): `withAuth(...)` সবসময় **component-এর বাইরে** ডাকবেন, যেমন route ফাইলে (এখানে একবারই চলে)। কোনো component-এর ভেতরে, render-এর সময় `withAuth(X)` লিখলে প্রতিবার render-এ একটা **নতুন** component তৈরি হবে, React ভাববে আগেরটা সরে গিয়ে নতুন এসেছে, ফলে ভেতরের সব state মুছে যাবে আর page বারবার নতুন করে বসবে।

---

# ভাগ খ: পাহারা বসানো

## ধাপ ৩ — Login না থাকলে `/login`-এ পাঠানো

🎯 **লক্ষ্য:** Login ছাড়া কেউ সুরক্ষিত page দেখতে পারবে না।

🛠️ **কাজ:**

```tsx
import { Navigate } from "react-router";

return function AuthWrapper() {
  const { data } = useUserInfoQuery(undefined);

  if (!data?.data?.email) {
    return <Navigate to="/login" />;
  }

  return <Component />;
};
```

💡 **কীভাবে:** `/user/me` (১৩ নম্বর নোট) user-এর তথ্য দিলে logged in, না দিলে (403) logged out। Email না থাকলে আসল page না দেখিয়ে `<Navigate>` ফেরত দিই, যেটা render হওয়ামাত্র `/login`-এ পাঠিয়ে দেয়।

✅ **যাচাই (এবং সমস্যা):** Logout অবস্থায় `/admin`-এ গেলে `/login`-এ যায় ✅। কিন্তু **login করা অবস্থায়** `/admin`-এ থেকে page **reload** দিলে… আবার `/login`-এ চলে যায় ❌, অথচ আমি তো login করা!

---

## ধাপ ৪ — Reload-এ login page-এ চলে যাওয়া: `isLoading`

🎯 **লক্ষ্য:** Data আসার আগেই "login নেই" সিদ্ধান্ত না নেওয়া।

### 💡 কেন এমন হলো

Reload দিলে RTK Query-র cache খালি (১৩ নম্বর নোট)। তখন:

```
মুহূর্ত ১: page render → useUserInfoQuery শুরু হলো, data এখনো আসেনি
          data = undefined → !data?.data?.email = true → <Navigate to="/login" />  ❌
মুহূর্ত ২: (দেরিতে) /user/me-র উত্তর এল, user আছে… কিন্তু ততক্ষণে আমরা /login-এ চলে গেছি
```

"Data এখনো আসেনি" আর "data এসেছে, user নেই", দুটোতেই `data` হলো `undefined`। কোডটা এই দুই অবস্থার পার্থক্য করছিল না। Teacher যেটাকে বললেন "weird behaviour"।

### 🛠️ কাজ

RTK Query বলে দেয় request এখনো চলছে কিনা: **`isLoading`**।

```tsx
const { data, isLoading } = useUserInfoQuery(undefined);

if (!isLoading && !data?.data?.email) {
  return <Navigate to="/login" />;
}
```

মানে: "লোড **শেষ** হওয়ার পরেও email না থাকলে, তবেই login-এ পাঠাও।"

| অবস্থা | `isLoading` | `data` | ফল |
|---|---|---|---|
| Request চলছে | `true` | `undefined` | Redirect **না** |
| এসেছে, user আছে | `false` | user | Page দেখায় |
| এসেছে, login নেই (403) | `false` | `undefined` | `/login` |

> নোট: `isLoading` সত্য থাকে শুধু **প্রথমবার**, যখন cache-এ কিছুই নেই। একবার data এসে গেলে পরে আবার আনার সময় (refetch) `isLoading` আর `true` হয় না (তখন `isFetching` হয়)। আমাদের দরকার ঠিক এটাই: প্রথমবার "জানি না" অবস্থা ধরা।

✅ **যাচাই:** Login করা অবস্থায় `/admin` reload দিন → আর `/login`-এ যায় না। ✅

> ⚠️ খেয়াল করুন: এই কোডে **লোড চলার সময় কী দেখায়?** দুটো `if`-ই মিথ্যা, তাই নিচে নেমে `<Component />`, অর্থাৎ **আসল page**! আপনি যে "এক ঝলক content দেখা যায়" সমস্যা খেয়াল করেছেন, তার মূল এখানেই। সমাধান শেষের "আমার পরামর্শ"-এ।

---

## ধাপ ৫ — Role মেলানো: `/unauthorized`

🎯 **লক্ষ্য:** Login থাকলেও, ঠিক role না হলে ঢুকতে না দেওয়া।

🛠️ **কাজ (teacher-এর চূড়ান্ত `withAuth`):**

```tsx
// src/utils/withAuth.tsx
import { useUserInfoQuery } from "@/redux/features/auth/auth.api";
import type { TRole } from "@/types";
import type { ComponentType } from "react";
import { Navigate } from "react-router";

export const withAuth = (Component: ComponentType, requiredRole?: TRole) => {
  return function AuthWrapper() {
    const { data, isLoading } = useUserInfoQuery(undefined);

    if (!isLoading && !data?.data?.email) {
      return <Navigate to="/login" />;
    }

    if (requiredRole && !isLoading && requiredRole !== data?.data?.role) {
      return <Navigate to="/unauthorized" />;
    }

    return <Component />;
  };
};
```

### 💡 দুটো parameter

| Parameter | মানে |
|---|---|
| `Component` | কোন component-কে পাহারা দেব (যেটায় ঢোকার অনুমতি দেখতে হবে) |
| `requiredRole?` | কোন role থাকলে দেখতে পারবে। `?` মানে **optional**: না দিলে শুধু login দেখা হয়, role দেখা হয় না (যেমন "যেকোনো logged-in user-এর profile page") |

### 💡 দ্বিতীয় শর্তটা পড়া

```ts
requiredRole && !isLoading && requiredRole !== data?.data?.role
```

`&&` মানে **সবগুলো** সত্য হলে তবেই redirect:

1. `requiredRole` — role দেওয়া আছে (না থাকলে role দেখারই দরকার নেই, এখানেই থামে)
2. `!isLoading` — লোড শেষ (ধাপ ৪-এর কারণ)
3. `requiredRole !== data?.data?.role` — চাওয়া role আর user-এর role **মেলে না**

তিনটাই সত্য → `/unauthorized`। কোনো একটা মিথ্যা → নিচে নেমে `<Component />` দেখায়।

> ⚠️ সংশোধন (ব্যাখ্যায়): নোটে লেখা ছিল "তোমার role থাকলে, loading না হলে, আর required role যদি logged-in role হয়, তাহলে navigate করতে পারবে, নাহলে unauthorized"। শেষ অংশটা উল্টো পড়া হয়েছে: শর্তে আছে `!==` (**মেলে না**)। অর্থাৎ শর্ত সত্য মানে role **মেলেনি** → `/unauthorized`। শর্ত মিথ্যা (role মিলেছে) → page দেখায়। সংক্ষেপে: **মিললে ঢুকতে দাও, না মিললে unauthorized।**

### 💡 ক্রম কেন এমন

আগে login, তারপর role। Login না থাকলে role দেখার মানেই হয় না (role-ও `undefined`), তখন user-কে "unauthorized" না, "login করো" বলাই ঠিক।

✅ **যাচাই:** `/unauthorized` route এখনো নেই, তাই আপাতত ভুল role-এ গেলে কিছু দেখাবে না। পরের ধাপে page বানাব।

---

# ভাগ গ: Unauthorized page ও Route

## ধাপ ৬ — `Unauthorized` page

🎯 **লক্ষ্য:** ভুল role-এ এলে একটা পরিষ্কার, সুন্দর বার্তা।

🛠️ **কাজ:** আপনার page (design-এর রং token দিয়ে ঠিক করা, নিচে সংশোধন দেখুন):

```tsx
// src/pages/Unauthorized.tsx
import Logo from "@/assets/icons/Logo";
import { Button } from "@/components/ui/button";
import { Link, useLocation, useNavigate } from "react-router";

export default function Unauthorized() {
  const navigate = useNavigate();
  const location = useLocation();

  // সরাসরি এই page-এ এলে (আগের কোনো page নেই) Back চাপলে site থেকেই বেরিয়ে যেত, তাই তখন Home
  const goBack = () => (location.key === "default" ? navigate("/") : navigate(-1));

  return (
    <div className="flex min-h-svh flex-col bg-background">
      <header className="px-6 py-5">
        <Logo />
      </header>

      <main className="flex flex-1 flex-col items-center justify-center gap-[18px] px-4 pt-8 pb-16 text-center">
        <div aria-hidden="true">
          <span className="text-[120px] leading-[0.9] font-extrabold tracking-[-0.03em] text-primary sm:text-[200px]">
            4<span className="text-pink-bright">0</span>3
          </span>
          <svg width="120" height="40" viewBox="0 0 120 40" className="mx-auto mt-2 block">
            <path
              d="M2 30 C 22 8, 40 8, 60 22 S 98 36, 118 10"
              fill="none"
              stroke="currentColor"
              strokeWidth="3"
              strokeLinecap="round"
              strokeDasharray="2 9"
              className="text-pink-bright"
            />
          </svg>
        </div>

        <p className="text-[13px] font-bold tracking-[0.08em] text-pink uppercase">No entry</p>

        <h1 className="max-w-[640px] text-[28px] font-bold tracking-tight text-foreground sm:text-[40px]">
          This trail is for staff only
        </h1>

        <p className="max-w-[520px] text-[17px] text-muted-foreground">
          You don't have permission to view this page. Log in with an account that has access, or head back to where you
          were.
        </p>

        <div className="flex flex-wrap justify-center gap-2.5 pt-1.5">
          <Button asChild className="h-12 px-5 text-[15px] font-semibold">
            <Link to="/">Go to home</Link>
          </Button>
          <Button variant="outline" onClick={goBack} className="h-12 px-5 text-[15px] font-semibold">
            Go back
          </Button>
        </div>

        <p className="pt-2 text-sm text-muted-foreground">
          Wrong account?{" "}
          <Link to="/login" className="font-semibold text-primary hover:underline">
            Log in again
          </Link>
        </p>
      </main>
    </div>
  );
}
```

💡 **কিছু অংশ:**
- **`<Button asChild><Link /></Button>`:** দেখতে button, কাজ করে link (Radix-এর `asChild`, ১০ নম্বর নোট)। "Go to home" অন্য page-এ যায়, তাই link; "Go back" একটা কাজ (history-তে পেছনে যাওয়া), তাই সাধারণ button।
- **`navigate(-1)`:** browser-এর Back button-এর মতো, history-তে এক ধাপ পেছনে।
- **`aria-hidden="true"`:** বড় "403" আর আঁকাবাঁকা দাগ শুধু সাজসজ্জা, screen reader এগুলো পড়বে না; আসল বার্তা `<h1>` আর `<p>`-এ।

> ⚠️ সংশোধন (রং-এর নিয়ম): `text-[#FF7A93] dark:text-[#FF8FA6]` hex রং, আমাদের নিয়মে দেওয়া যায় না। আপনার `index.css`-এ `--pink` token আছে, কিন্তু সেটার রং আলাদা (light-এ `#B4335A`, গাঢ়), তাই সরাসরি `text-pink` দিলে ডিজাইন বদলে যেত। ডিজাইন হুবহু রাখতে একটা নতুন token যোগ করুন:
> ```css
> /* index.css */
> @theme inline {
>   --color-pink-bright: var(--pink-bright);
>   /* ...আগের সব */
> }
>
> :root {
>   --pink-bright: #FF7A93;
> }
>
> .dark {
>   --pink-bright: #FF8FA6;
> }
> ```
> তারপর দুই জায়গাতেই শুধু `text-pink-bright`। এখন `dark:` আলাদা লিখতে হয় না, token নিজেই mode অনুযায়ী বদলায়, আর এই রং অন্য page-এও ব্যবহার করা যাবে।

> ➕ সংযোজন — "Go back" ঠিক করা: কেউ সরাসরি `/unauthorized` লিখে এলে (বা নতুন tab-এ খুললে) history-তে আগের কোনো page নেই। তখন `navigate(-1)` user-কে আমাদের site থেকেই বের করে দেয়। React Router প্রথম page-এর `location.key`-কে `"default"` দেয়, সেটা দেখে তখন Home-এ পাঠানো হয়েছে।

> নোট: `<h1>`-এ `text-foreground` যোগ করা হয়েছে, যাতে dark mode-এ রং নিশ্চিতভাবে ঠিক থাকে (আমাদের নিয়ম: লেখায় semantic token)।

---

## ধাপ ৭ — Route-এ `withAuth` বসানো

🎯 **লক্ষ্য:** Admin আর user-এর পুরো dashboard সুরক্ষিত করা।

🛠️ **কাজ (teacher-এর চূড়ান্ত route):**

```tsx
// src/routes/index.tsx
import App from "@/App";
import DashboardLayout from "@/components/layout/DashboardLayout";
import { role } from "@/constants/role";
import About from "@/pages/About";
import LoginPage from "@/pages/Login";
import RegisterPage from "@/pages/Register";
import Unauthorized from "@/pages/Unauthorized";
import Verify from "@/pages/verify";
import type { TRole } from "@/types";
import { generateRoutes } from "@/utils/generateRoutes";
import { withAuth } from "@/utils/withAuth";
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
    Component: withAuth(DashboardLayout, role.superAdmin as TRole),
    path: "/admin",
    children: [...generateRoutes(adminSidebarItems)],
  },
  {
    Component: withAuth(DashboardLayout, role.user as TRole),
    path: "/user",
    children: [
      { index: true, element: <Navigate to="/user/bookings" replace /> },
      ...generateRoutes(userSidebarItems),
    ],
  },
  { Component: LoginPage, path: "/login" },
  { Component: RegisterPage, path: "/signup" },
  { Component: Verify, path: "/verify" },
  { Component: Unauthorized, path: "/unauthorized" },
]);
```

### 💡 শুধু `DashboardLayout` মুড়লেই সব child সুরক্ষিত কেন

Child page (Analytics, Tours, Bookings …) দেখায় layout-এর `<Outlet />`-এর ভেতরে। পাহারাদার layout-কেই না দেখালে (redirect করলে), `<Outlet />`-ও render হয় না, তাই কোনো child page-ও না। **বাড়ির মূল দরজায় তালা দিলে ভেতরের প্রতিটা ঘরে আলাদা তালা লাগে না।** নতুন admin page (`adminSidebarItems`-এ) যোগ করলে সেটাও নিজে থেকেই সুরক্ষিত।

### 💡 `role.superAdmin as TRole` কেন

`role` constant-এ `as const` নেই, তাই TypeScript `role.superAdmin`-কে শুধু "কোনো একটা `string`" ভাবে। কিন্তু `withAuth` চায় `TRole` (`"SUPER_ADMIN" | "ADMIN" | "USER"`)। `as TRole` দিয়ে TypeScript-কে বলা হচ্ছে "বিশ্বাস করো, এটা TRole"। কাজ চলে, তবে `as` দিয়ে জোর করলে TypeScript আর ভুল ধরতে পারে না। ভালো উপায় শেষের পরামর্শে।

### 💡 Unauthorized-কে আটকাব কিনা

আপনি লিখেছেন, চাইলে verify page-এর মতো শর্ত দিয়ে unauthorized page-কে শুধু redirect হয়ে এলেই দেখানো যায়। করা যায়, কিন্তু দরকার নেই: এই page-এ কোনো গোপন তথ্য নেই, কেউ সরাসরি খুললেও শুধু "No entry" বার্তা দেখবে। Verify page-এ শর্ত লাগত কারণ সেখানে email ছাড়া page কাজই করে না।

> ⚠️ সংশোধন: নোটে ফাইলের নাম `routes/index.ts` লেখা, আপনার project-এ `index.tsx`, সেটাই ঠিক (JSX আছে, ১৪ নম্বর নোট)। আর `/user`-এর index redirect-এ `replace` যোগ করা হয়েছে (১৪ নম্বর নোটের ধাপ ১৯)।

✅ **যাচাই:**
- Logout অবস্থায় `/admin/tours` → `/login`।
- USER দিয়ে login করে `/admin` → `/unauthorized`।
- USER দিয়ে `/user` → `/user/bookings` ✅।
- SUPER_ADMIN দিয়ে `/admin` → Overview ✅।
- **ADMIN দিয়ে `/admin` → `/unauthorized` ❌!** (এটা bug, পরের অংশে)
- আর ভুল role-এ যাওয়ার সময় এক ঝলক আসল page দেখা যায় (আপনার খেয়াল করা সমস্যা)।

---

# আমার পরামর্শ: ঝলক ছাড়া, ঠিকঠাক Authorization

আপনি যেমন চেয়েছিলেন, নোট শেষে এই অংশে আপনার খেয়াল করা সমস্যাগুলো আর সেগুলোর সমাধান।

## পরামর্শ ১ — এক ঝলক content দেখা: আসল কারণ

### 💡 আপনি ঠিকই ধরেছেন

USER দিয়ে login করে `/admin`-এ গেলে এক মুহূর্তের জন্য admin-এর dashboard দেখা যায়, তারপর `/unauthorized`। এটা শুধু দেখতে খারাপ না, **সমস্যাও**: ঐ মুহূর্তে admin-এর page render হয়, তার ভেতরের API call (যেমন সব user-এর list আনা) চলে যেতে পারে, আর কেউ screenshot নিলে বা ধীর internet-এ ঐ ঝলকটা লম্বা হলে অন্যের তথ্য দেখা যেতে পারে।

### 💡 কারণ: teacher-এর কোডে loading-এর সময় কোন লাইন চলে

```tsx
if (!isLoading && !data?.data?.email) { ... }                  // isLoading = true → মিথ্যা, পার
if (requiredRole && !isLoading && requiredRole !== ...) { ... } // isLoading = true → মিথ্যা, পার
return <Component />;                                            // ← এটাই চলে! আসল page
```

দুটো শর্তেই `!isLoading` আছে, তাই loading চলার সময় দুটোই মিথ্যা, আর কোড নিচে নেমে **আসল component** দেখিয়ে দেয়। Data আসার পর তবেই role মিলিয়ে redirect হয়। অর্থাৎ ধাপ ৪-এর reload-সমস্যা ঠিক করতে গিয়ে loading-এর সময়টাকে "অনুমতি আছে" ধরে নেওয়া হয়েছে।

**নিরাপত্তার মূল নীতি:** যতক্ষণ জানি না, ততক্ষণ দরজা **বন্ধ**। "জানি না" মানে "ঢুকতে দাও" না।

### 🛠️ সমাধান: loading-কে আলাদা করে আগে ধরা

```tsx
if (isLoading) {
  return <PageLoader />;   // জানি না → আসল page না, শুধু loading
}
// এর নিচে পৌঁছানো মানে লোড শেষ, তাই আর !isLoading লেখার দরকার নেই
```

এখন তিনটা অবস্থা তিনটা আলাদা পথে:

| অবস্থা | কী দেখায় |
|---|---|
| লোড চলছে | Loading skeleton (আসল page **না**) |
| Login নেই | `/login` |
| Role মেলে না | `/unauthorized` |
| সব ঠিক | আসল page |

Loading-এর সময় আসল component একবারও render হয় না, তাই ঝলক নেই, ভেতরের API call-ও যায় না।

---

## পরামর্শ ২ — ADMIN `/admin`-এ ঢুকতে পারছে না: একাধিক role

### 💡 Bug

Route-এ লেখা `withAuth(DashboardLayout, role.superAdmin)`। শুধু SUPER_ADMIN ঢুকতে পারে। কিন্তু:
- `getSidebarItems` ADMIN-কে admin-এর menu দেয় (১৪ নম্বর নোট)
- Navbar ADMIN-কে `/admin`-এর Dashboard link দেয়

ADMIN সেই link চাপলে → `/unauthorized`। আপনি নিজেই লিখেছিলেন: একই জায়গায় একাধিক role লাগলে array পাঠাতে হবে। ঠিক তাই।

### 🛠️ সমাধান: `requiredRoles` array

```ts
// আগে: একটা role
withAuth(DashboardLayout, role.superAdmin as TRole)

// পরে: যেকোনো একটা মিললেই হবে
withAuth(DashboardLayout, [role.superAdmin, role.admin])
```

আর শর্তে `!==`-এর বদলে `.includes()`:

```ts
if (requiredRoles && !requiredRoles.includes(user.role)) {
  return <Navigate to="/unauthorized" replace />;
}
```

`[A, B].includes(x)` মানে "x কি A অথবা B?"। User-এর role array-তে না থাকলে unauthorized।

---

## পরামর্শ ৩ — Redirect-এ `replace`, আর login-এর পরে আগের page-এ ফেরা

### 💡 `replace` না দিলে Back-এর ফাঁদ

```
/  →  /admin (withAuth redirect)  →  /unauthorized
```

`replace` ছাড়া history-তে `/admin` থেকে যায়। Unauthorized page-এর **"Go back"** চাপলে → `/admin` → আবার redirect → `/unauthorized` → আবার "Go back" … **user আটকে যায়**। `replace` দিলে `/admin`-এর জায়গায় `/unauthorized` বসে, "Go back" সরাসরি Home-এ ফেরে।

### 💡 Login-এর পরে যেখানে যেতে চেয়েছিল সেখানে ফেরা (➕)

কেউ logout অবস্থায় `/admin/analytics` খুলল → `/login`। Login-এর পরে তাকে Home-এ না পাঠিয়ে সরাসরি Analytics-এ ফেরত দেওয়া ভালো অভিজ্ঞতা। Redirect-এর সময় কোথা থেকে এসেছিল সেটা `state`-এ পাঠিয়ে দিন:

```tsx
// withAuth-এ
const location = useLocation();
return <Navigate to="/login" replace state={{ from: location.pathname }} />;
```

```tsx
// LoginForm.tsx-এ, login সফল হওয়ার পরে
const location = useLocation();
const from = (location.state as { from?: string } | null)?.from ?? "/";
navigate(from, { replace: true });
```

(১৩ নম্বর নোটে Verify page-এ email যেভাবে `state` দিয়ে পাঠানো হয়েছিল, একই কৌশল।)

> নোট: আপনার `login` mutation-এ এখনো `invalidatesTags: ["USER"]` নেই (১৩ নম্বর নোটের ধাপ ২৮-এর পরামর্শ)। Login-এর পরে `withAuth` যেন নিশ্চিতভাবে নতুন user-এর তথ্য পায়, সেজন্য এটা যোগ করে দিন।

---

## পরামর্শ ৪ — `as TRole` বাদ দেওয়া (`as const`)

১৪ নম্বর নোটের ধাপ ১৩-এর পরামর্শ এখানে সরাসরি কাজে লাগে:

```ts
// src/constants/role.ts
export const role = {
  superAdmin: "SUPER_ADMIN",
  admin: "ADMIN",
  user: "USER",
} as const;
```

এখন TypeScript জানে `role.admin` ঠিক `"ADMIN"`, তাই route-এ `as TRole` লাগে না:

```tsx
withAuth(DashboardLayout, [role.superAdmin, role.admin])   // কোনো as নেই
```

`as` দিয়ে জোর না করলে, ভুল করে `role.adimn` লিখলে TypeScript সাথে সাথে ধরে।

---

## চূড়ান্ত `withAuth` (সব পরামর্শ মিলিয়ে)

```tsx
// src/utils/withAuth.tsx
import { PageLoader } from "@/components/ui/page-loader";
import { useUserInfoQuery } from "@/redux/features/auth/auth.api";
import type { TRole } from "@/types";
import type { ComponentType } from "react";
import { Navigate, useLocation } from "react-router";

export const withAuth = (Component: ComponentType, requiredRoles?: TRole[]) => {
  return function AuthWrapper() {
    const { data, isLoading } = useUserInfoQuery(undefined);
    const location = useLocation();

    // ১. এখনো জানি না user কে → আসল page না, শুধু loading
    if (isLoading) {
      return (
        <div className="min-h-dvh bg-background p-6">
          <PageLoader />
        </div>
      );
    }

    const user = data?.data;

    // ২. Login নেই → login-এ, সাথে কোথা থেকে এসেছিল
    if (!user?.email) {
      return <Navigate to="/login" replace state={{ from: location.pathname }} />;
    }

    // ৩. Role দেওয়া আছে কিন্তু মেলে না → unauthorized
    if (requiredRoles && !requiredRoles.includes(user.role)) {
      return <Navigate to="/unauthorized" replace />;
    }

    // ৪. সব ঠিক
    return <Component />;
  };
};
```

`PageLoader` হলো ১৫ নম্বর নোটের ধাপ ৫-এ বানানো skeleton। এখনো না বানিয়ে থাকলে আপাতত যেকোনো ছোট loading UI দিন, শুধু আসল `<Component />` দেবেন না।

Route:

```tsx
// src/routes/index.tsx  (শুধু বদলানো দুই অংশ)
{
  Component: withAuth(DashboardLayout, [role.superAdmin, role.admin]),
  path: "/admin",
  children: [...generateRoutes(adminSidebarItems)],
},
{
  Component: withAuth(DashboardLayout, [role.user]),
  path: "/user",
  children: [
    { index: true, element: <Navigate to="/user/bookings" replace /> },
    ...generateRoutes(userSidebarItems),
  ],
},
```

(`as const` না দিলে array-র প্রতিটায় `as TRole` লাগবে, বা `[role.superAdmin, role.admin] as TRole[]`।)

✅ **যাচাই:**
- USER দিয়ে `/admin` → শুধু loading skeleton, তারপর `/unauthorized`; **admin-এর কিছুই এক ঝলকও দেখা যায় না**। (Network-এ **Slow 4G** দিয়ে দেখুন, ঝলক থাকলে তখন স্পষ্ট বোঝা যায়।)
- ADMIN দিয়ে `/admin` → Overview ✅ (আগে unauthorized হতো)।
- `/unauthorized`-এ "Go back" চাপলে আবার `/admin`-এ আটকে যায় না।
- Logout অবস্থায় `/admin/analytics` → login → login-এর পরে সরাসরি Analytics।
- Login অবস্থায় `/admin` reload → loading, তারপর page; `/login`-এ যায় না।

---

## পরামর্শ ৫ — আরও এগিয়ে (পরে শেখার জন্য): Loader দিয়ে render-এর আগেই যাচাই

HOC-এ যাচাই হয় **component render হওয়ার সময়**, তাই অন্তত loading skeleton দেখাতেই হয়। React Router-এর data mode-এ (আমরা `createBrowserRouter` ব্যবহার করছি) আরেকটা উপায় আছে: route-এর **`loader`**। Loader চলে page render হওয়ার **আগে**, আর সেখান থেকে সরাসরি redirect করা যায়। ফলে ভুল role-এর user-এর জন্য dashboard-এর কোনো অংশ, এমনকি skeleton-ও, render হয় না।

ধারণাটা মোটামুটি এমন:

```ts
// src/utils/requireAuth.ts  (ধারণা — প্রয়োগের আগে React Router-এর বর্তমান docs মিলিয়ে নিন)
import { authApi } from "@/redux/features/auth/auth.api";
import { store } from "@/redux/store";
import type { TRole } from "@/types";
import { redirect } from "react-router";

export const requireAuth = (requiredRoles?: TRole[]) => async () => {
  // Component ছাড়াই RTK Query দিয়ে /user/me আনা (cache থাকলে সেখান থেকেই)
  const result = await store.dispatch(
    authApi.endpoints.userInfo.initiate(undefined, { subscribe: false }),
  );
  const user = result.data?.data;

  if (!user?.email) throw redirect("/login");
  if (requiredRoles && !requiredRoles.includes(user.role)) throw redirect("/unauthorized");
  return null;
};
```

```tsx
// route-এ
{
  Component: DashboardLayout,
  path: "/admin",
  loader: requireAuth([role.superAdmin, role.admin]),
  children: [...generateRoutes(adminSidebarItems)],
},
```

| | HOC (`withAuth`) | Loader (`requireAuth`) |
|---|---|---|
| কখন যাচাই | Render-এর সময় | Render-এর **আগে** |
| Loading-এর সময় | নিজে skeleton দেখাতে হয় | Page বদলানোর সময় আগের page থেকে যায়; প্রথম load-এর জন্য React Router-এর `HydrateFallback` দিতে হয় |
| শেখা সহজ | ✅ (teacher এটাই দেখাচ্ছেন) | একটু বেশি ধারণা লাগে |
| ঝলক | পরামর্শ ১-এর পরে নেই | একদমই নেই |

**আমার পরামর্শ:** এখন teacher-এর HOC-এর সাথে থাকুন, শুধু পরামর্শ ১–৪ বসিয়ে নিন। এতেই ঝলকের সমস্যা পুরো মিটে যায়। Course-এ পরে `loader` এলে বা project বড় হলে loader-এ যাওয়ার কথা ভাববেন।

---

## পরামর্শ ৬ — মনে রাখার মতো দুটো কথা

**১. আসল তালা backend-এ।** Frontend-এর `withAuth` শুধু ঠিক user-কে ঠিক page দেখায়; কেউ চাইলে browser-এর কোড বদলে বা Postman দিয়ে সরাসরি API-তে যেতে পারে। তাই backend-এর প্রতিটা admin API-তে `checkAuth(role.admin, role.superAdmin)` (১৩ নম্বর নোট) থাকতেই হবে। Frontend-এর পাহারা user-এর সুবিধার জন্য, backend-এর পাহারা নিরাপত্তার জন্য।

**২. নামের ব্যাপারে ছোট তথ্য:** HTTP-তে **401 Unauthorized** মানে আসলে "login নেই" (authentication), আর **403 Forbidden** মানে "login আছে, অনুমতি নেই" (authorization)। নামটা বিভ্রান্তিকর, অনেকে তাই "unauthorized" শব্দটা দুই অর্থেই ব্যবহার করে। আমাদের page-এ বড় করে **403** লেখা, যেটা এই অবস্থার জন্য একদম সঠিক সংখ্যা; route-এর নাম `/unauthorized` রাখাও প্রচলিত, বদলানোর দরকার নেই।

---

# Check-list

**Teacher-এর অংশ**
- [ ] Logout অবস্থায় `/admin/...` বা `/user/...` → `/login`
- [ ] Login অবস্থায় dashboard reload দিলে `/login`-এ যায় না
- [ ] USER দিয়ে `/admin` → `/unauthorized`
- [ ] Unauthorized page light আর dark দুই mode-এ ঠিক দেখায় (`text-pink-bright` token)
- [ ] সরাসরি `/unauthorized` খুলে "Go back" চাপলে Home-এ যায়, site থেকে বেরিয়ে যায় না

**পরামর্শ বসানোর পরে**
- [ ] Slow 4G-তে USER দিয়ে `/admin` গেলে admin-এর কিছুই এক ঝলকও দেখা যায় না, শুধু skeleton
- [ ] ADMIN আর SUPER_ADMIN দুজনেই `/admin`-এ ঢুকতে পারে
- [ ] Unauthorized-এ "Go back" চাপলে `/admin`-এ ফেরত গিয়ে আটকে যায় না
- [ ] Logout অবস্থায় `/admin/analytics` → login → সরাসরি Analytics
- [ ] Route ফাইলে আর `as TRole` নেই (`as const` দেওয়ার পরে)
- [ ] `login` mutation-এ `invalidatesTags: ["USER"]`
- [ ] Backend-এর admin API-গুলোতেও role check আছে
