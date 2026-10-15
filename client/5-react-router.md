# Frontend Notes — ০৫: React Router (Data Mode) ও Layout

## ০. শুরুর আগে: Dev server-এর port বদলানো

Vite-এর dev server by default `5173` port-এ চলে। আমরা চাইলে `3000`-এ চালাতে পারি। এজন্য `vite.config.ts`-এ `plugins` আর `resolve`-এর পরে `server` যোগ করব:

```ts
export default defineConfig({
  plugins: [react(), tailwindcss()],
  resolve: {
    alias: {
      "@": path.resolve(__dirname, "./src"),
    },
  },
  server: {
    port: 3000,
  },
})
```

Config বদলানোর পর **dev server restart** করতে হবে (terminal-এ `Ctrl + C`, তারপর আবার `bun run dev`)।

> টিপ: `3000` port অন্য কোনো app ব্যবহার করলে Vite নিজে থেকে পরের খালি port-এ চলে যায়। চাইলে `strictPort: true` দিলে অন্য port-এ না গিয়ে error দেবে।

---

## ১. React Router-এর পরিচিতি

Docs: https://reactrouter.com/start/data/installation

নোট লেখার সময় React Router-এর latest version ছিল **8.4.0** (নিজের প্রজেক্টে কোন version আছে তা `package.json`-এ দেখে নিন)।

React Router ব্যবহারের **৩টা mode** আছে:

| Mode | কী | কখন |
|---|---|---|
| **Declarative** | `<BrowserRouter>`, `<Routes>`, `<Route>` দিয়ে JSX-এ route লেখা | সবচেয়ে সহজ, শুধু URL অনুযায়ী page দেখানো |
| **Data** ✅ | `createBrowserRouter` দিয়ে object আকারে route লেখা | loader, action, pending state ইত্যাদি feature লাগলে — **আমরা এটা ব্যবহার করব** |
| **Framework** | Vite plugin সহ পুরো framework (আগের Remix) | SSR, file-based routing লাগলে |

> ⚠️ সংশোধন: নোটে "৩ ধরনের **version**" লেখা ছিল — আসলে এগুলো **mode**, version না। একই version-এর ভেতরেই ৩টা mode।

Install (আমাদের আগেই করা আছে):

```sh
bun add react-router
```

> নোট: নোটে `npm i react-router` লেখা ছিল। আমরা bun ব্যবহার করছি, তাই `bun add react-router`।

---

## ২. Router তৈরি

### ২.১ Page তৈরি

আগে `src/pages/` folder-এ কয়েকটা সাধারণ page বানিয়ে নিই:

```tsx
// src/pages/About.tsx
export default function About() {
  return <h1>About</h1>
}
```

একইভাবে `Home.tsx`, `Dashboard.tsx`, `Analytics.tsx` বানানো যায়।

### ২.২ Route define করা

`src/routes/index.tsx` ফাইল তৈরি করে সেখানে `createBrowserRouter` দিয়ে route define করব:

```tsx
import App from "@/App";
import About from "@/pages/About";
import Dashboard from "@/pages/Dashboard";
import Home from "@/pages/Home";
import { createBrowserRouter } from "react-router";

export const router = createBrowserRouter([
  {
    path: "/",
    Component: App,
    children: [
      {
        index: true,          // "/" — home page
        Component: Home,
      },
      {
        path: "about",        // "/about"
        Component: About,
      },
      {
        path: "dashboard",    // "/dashboard"
        Component: Dashboard,
      },
    ],
  },
]);
```

> ⚠️ সংশোধন (গুরুত্বপূর্ণ): নোটের কোডে লেখা ছিল —
> ```tsx
> { Component: <div>Home</div>, path: "" }
> { Component: <Dashboard />, path: "dashboard" }
> ```
> এটা **ভুল**, error দেবে। `Component`-এ শুধু component-এর **নাম** দিতে হয় (`Dashboard`), JSX (`<Dashboard />`) না। JSX দিতে চাইলে `element` ব্যবহার করতে হবে:
> ```tsx
> { element: <div>Home</div>, index: true }
> { element: <Dashboard />, path: "dashboard" }
> ```
>
> আর home page-এর জন্য `path: ""`-এর বদলে **`index: true`** ব্যবহার করা নিয়ম। Index route মানে — parent-এর path-এ ঠিক গেলে (এখানে `/`) এটা দেখাবে।

### ২.৩ `Component` বনাম `element`

Route-এ কী দেখাবে সেটা দুইভাবে বলা যায়:

| | `Component` | `element` |
|---|---|---|
| কী দিতে হয় | React **component** (function-টা নিজে) | React **element** (JSX) |
| উদাহরণ | `Component: About` | `element: <About />` |
| Props পাঠানো যায়? | না | হ্যাঁ — `element: <About title="Hi" />` |

সহজ ভাষায়: `About` হলো **রেসিপি** (component), আর `<About />` হলো রেসিপি দিয়ে **রান্না করা খাবার** (element)।

### ২.৪ `.ts` না `.tsx`?

নিয়ম একটাই: **যে ফাইলে JSX (`<...>`) লেখা আছে, সেটা `.tsx` হতে হবে।**

- `element: <About />` ব্যবহার করলে ফাইলে JSX আছে → **`.tsx` বাধ্যতামূলক**।
- শুধু `Component: About` ব্যবহার করলে ফাইলে কোনো JSX নেই → **`.ts` দিলেও চলবে** (`.tsx` দিলেও সমস্যা নেই)।

### ২.৫ Protected route ও Higher Order Component (HOC)

Login ছাড়া কিছু page (যেমন dashboard) দেখানো যাবে না — এগুলো **protected route**। দুইভাবে করা যায়:

**১) `element` দিয়ে — wrapper component:**
```tsx
{ path: "dashboard", element: <ProtectedRoute><Dashboard /></ProtectedRoute> }
```

**২) `Component` দিয়ে — Higher Order Component (আমরা এটা করব):**
```tsx
{ path: "dashboard", Component: withAuth(Dashboard) }
```

**HOC** মানে এমন একটা function যে একটা component নেয় এবং **নতুন একটা component ফেরত দেয়** (যার ভেতরে auth check থাকে)। যেহেতু ফলাফল একটা component, তাই সরাসরি `Component`-এ দেওয়া যায়, JSX লাগে না।

> নোট: নোটে "higher order **function**" লেখা ছিল। React-এ component-এর ক্ষেত্রে একে **Higher Order Component (HOC)** বলা হয় — এটা আসলে higher order function-এরই একটা ধরন।
>
> ⚠️ সংশোধন: "element দিলে protected route-এর জন্য আলাদা component বানাতে হয়" — HOC পদ্ধতিতেও `withAuth` বানাতে হয়। দুটো পদ্ধতিই কাজ করে; পার্থক্য শুধু লেখার ধরনে।

---

## ৩. `main.tsx`-এ RouterProvider

`src/main.tsx`-এ `<App />` সরিয়ে `RouterProvider`-এ আমাদের `router` দেব:

```tsx
import { StrictMode } from "react";
import { createRoot } from "react-dom/client";
import { RouterProvider } from "react-router";
import "./index.css";
import { router } from "@/routes";

createRoot(document.getElementById("root")!).render(
  <StrictMode>
    <RouterProvider router={router} />
  </StrictMode>,
);
```

এখন কোন URL-এ কী দেখাবে, সেটা router ঠিক করবে। `App` এখন আর সরাসরি render হয় না — `/` route-এর component হিসেবে router তাকে দেখায়।

> নোট: `"./routes/index.tsx"`-এর বদলে `"@/routes"` লেখা পরিষ্কার — folder-এর নাম দিলে `index` ফাইল নিজে থেকেই খুঁজে নেয়।

---

## ৪. `<Outlet />` — child route কোথায় দেখাবে

`/about`-এ গেলে router আগে parent (`App`) render করে, তারপর child (`About`) render করে। কিন্তু child-টা `App`-এর **ঠিক কোন জায়গায়** বসবে, সেটা বলতে হয় **`<Outlet />`** দিয়ে।

```tsx
import { Outlet } from "react-router";

function App() {
  return (
    <div>
      <p>উপরে — সব page-এ দেখাবে</p>
      <Outlet />   {/* এখানে About / Dashboard / Home বসবে */}
      <p>নিচে — সব page-এ দেখাবে</p>
    </div>
  );
}
```

- `<Outlet />`-এর **উপরে** যা আছে → উপরে দেখাবে
- `<Outlet />`-এর **নিচে** যা আছে → নিচে দেখাবে
- `<Outlet />` না দিলে child route-এর content কোথাও দেখাবে না।

---

## ৫. Layout ডিজাইন: Sibling বনাম Nested route

### Sibling route — কোনো সম্পর্ক নেই

```tsx
export const router = createBrowserRouter([
  { path: "/", Component: App },
  { path: "/about", Component: About },
]);
```

`/`-এ গেলে শুধু `App`, `/about`-এ গেলে শুধু `About`। দুটো সম্পূর্ণ আলাদা route — একটার layout (navbar ইত্যাদি) আরেকটায় যায় না।

### Nested route — layout share করা

Nested route-এ child route-গুলো parent-এর layout-এর ভেতরে দেখায়। এর সুবিধা: ভিন্ন অংশের জন্য ভিন্ন layout রাখা যায়। যেমন:
- **Public page** (home, about) → Navbar + Footer
- **Dashboard/Admin page** → Navbar নেই, বদলে sidebar menu

তাই public route-গুলোর জন্য একটা nested group, আর private route-গুলোর জন্য আরেকটা nested group।

---

## ৬. Public Layout: CommonLayout

`src/components/layout/` folder তৈরি করে সেখানে ফাইলগুলো বানাব।

### `CommonLayout.tsx`

```tsx
import type { ReactNode } from "react";
import Footer from "./Footer";
import Navbar from "./Navbar";

interface IProps {
  children: ReactNode;
}

export default function CommonLayout({ children }: IProps) {
  return (
    <div className="min-h-screen flex flex-col">
      <Navbar />
      <main className="grow">{children}</main>
      <Footer />
    </div>
  );
}
```

- `min-h-screen flex flex-col` + মাঝখানে `grow` → content কম থাকলেও footer সবসময় স্ক্রিনের নিচে থাকে।

> ⚠️ সংশোধন: নোটে `import { ReactNode } from "react"` লেখা ছিল। Vite-এর TypeScript template-এ `verbatimModuleSyntax` চালু থাকে, তাই **type import করতে `import type` লিখতে হবে** — নাহলে error দেবে।
>
> নোট: Tailwind-এ `grow-1`-এর বদলে সাধারণত শুধু **`grow`** (বা `flex-1`) লেখা হয়।

### `ReactNode` বনাম `ReactElement`

`children`-এর type দেওয়ার জন্য দুটো option:

| Type | কী ধরে |
|---|---|
| `ReactElement` | শুধু **JSX element** — যেমন `<div />`, `<About />` |
| `ReactNode` | **render করা যায় এমন সবকিছু** — JSX, string, number, `null`, `undefined`, boolean, array |

তাই `children`-এর জন্য **`ReactNode`** সবচেয়ে উপযুক্ত — এতে চাইলে text-ও পাঠানো যায়।

> ⚠️ সংশোধন: নোটে লেখা ছিল "ReactNode component আর JSX দুটোই ধরে"। আসলে দুটোর কোনোটাই **component নিজে** (যেমন `About` function-টা) ধরে না — ধরে component **render করার পর পাওয়া element** (`<About />`)। Component নিজেকে prop হিসেবে পাঠাতে চাইলে type হবে `ComponentType`।

### `Navbar.tsx`

```tsx
import { Link } from "react-router";

export default function Navbar() {
  return (
    <nav>
      <Link to="/">Home</Link>
      <Link to="/about">About</Link>
    </nav>
  );
}
```

> সংযোজন: Page-এর মধ্যে যাওয়ার জন্য `<a href>`-এর বদলে **`<Link to>`** ব্যবহার করতে হবে। `<a>` দিলে পুরো page reload হয় (state হারিয়ে যায়); `<Link>` reload ছাড়া শুধু route বদলায়। Active link-এ আলাদা style দিতে চাইলে **`<NavLink>`**।

### `Footer.tsx`

```tsx
export default function Footer() {
  return (
    <footer>
      <p>Footer</p>
    </footer>
  );
}
```

### `App.tsx`-এ CommonLayout ব্যবহার

```tsx
import { Outlet } from "react-router";
import CommonLayout from "@/components/layout/CommonLayout";

function App() {
  return (
    <CommonLayout>
      <Outlet />
    </CommonLayout>
  );
}

export default App;
```

এখন `/`-এর নিচের সব page (Home, About) Navbar আর Footer-এর মাঝখানে দেখাবে।

---

## ৭. Admin Layout

### `AdminLayout.tsx`

```tsx
import { Outlet } from "react-router";

export default function AdminLayout() {
  return (
    <div>
      <h1>This is admin page header</h1>
      <Outlet />
    </div>
  );
}
```

### Route-এ দুটো আলাদা layout group

```tsx
import App from "@/App";
import AdminLayout from "@/components/layout/AdminLayout";
import About from "@/pages/About";
import Analytics from "@/pages/Analytics";
import { createBrowserRouter } from "react-router";

export const router = createBrowserRouter([
  {
    path: "/",
    Component: App,              // Public layout
    children: [
      { path: "about", Component: About },          // → /about
    ],
  },
  {
    path: "/admin",
    Component: AdminLayout,      // Admin layout
    children: [
      { path: "analytics", Component: Analytics },  // → /admin/analytics
    ],
  },
]);
```

- `/` দিয়ে শুরু হওয়া URL → `App` (CommonLayout)-এর ভেতরে দেখাবে
- `/admin` দিয়ে শুরু হওয়া URL → `AdminLayout`-এর ভেতরে দেখাবে

**Child route-এর path-এ শুরুতে `/` দেওয়া যাবে না।** Child path হলো parent-এর সাপেক্ষে (**relative**):
- `path: "analytics"` → parent `/admin` + `analytics` = **`/admin/analytics`** ✅
- `path: "/analytics"` → এটা absolute path, parent `/admin`-এর সাথে মেলে না → React Router **error দেবে** ❌

> সংশোধন: নোটে লেখা ছিল "about আর analytics কখনো `/about` বা `/analytics` হবে না, দিলে break করবে" — কথাটা ঠিক, শুধু কারণটা পরিষ্কার করা হলো: child path relative হয়, তাই analytics-এর আসল URL `/admin/analytics`।

---

## ৮. 404 — Page not found

কোনো route না মিললে দেখানোর জন্য একটা catch-all route (`path: "*"`) রাখা ভালো। `*` মানে "যেকোনো path" — অন্য কোনো route না মিললে এটা মিলবে।

প্রথমে `src/pages/NotFound.tsx` বানাই:

```tsx
import { Link } from "react-router";

export default function NotFound() {
  return (
    <div>
      <h1>404 — Page not found</h1>
      <Link to="/">Home-এ ফিরে যান</Link>
    </div>
  );
}
```

এরপর এটা `src/routes/index.tsx`-এ `createBrowserRouter`-এর array-তে বসাতে হবে। **কোথায় বসাবেন, তার ওপর নির্ভর করে 404 page-এ layout থাকবে কিনা।**

### Option ১: Top level-এ — layout ছাড়া (সাধারণত এটাই ব্যবহার হয়)

Array-র **শেষে**, `/` আর `/admin`-এর **sibling** হিসেবে:

```tsx
export const router = createBrowserRouter([
  {
    path: "/",
    Component: App,
    children: [
      { path: "about", Component: About },
    ],
  },
  {
    path: "/admin",
    Component: AdminLayout,
    children: [
      { path: "analytics", Component: Analytics },
    ],
  },
  { path: "*", Component: NotFound },   // ← এখানে
]);
```

ফলাফল: ভুল URL-এ (যেমন `/xyz`) শুধু NotFound page দেখাবে — Navbar/Footer **ছাড়া**।

### Option ২: `App`-এর children-এর ভেতরে — Navbar/Footer সহ

```tsx
{
  path: "/",
  Component: App,
  children: [
    { path: "about", Component: About },
    { path: "*", Component: NotFound },   // ← এখানে
  ],
},
```

ফলাফল: ভুল URL-এ NotFound page দেখাবে CommonLayout-এর **ভেতরে** — উপরে Navbar, নিচে Footer থাকবে। ব্যবহারকারী Navbar দিয়ে সহজে অন্য page-এ যেতে পারবে।

> চাইলে দুটোই রাখা যায় — যেমন `/admin`-এর children-এও একটা `*` দিলে `/admin/xyz`-এ AdminLayout-এর ভেতরে 404 দেখাবে।
>
> নোট: React Router সবচেয়ে **নির্দিষ্ট (specific)** route আগে মেলায়, তাই `*` array-র কোথায় লিখছেন তাতে matching-এ সমস্যা হয় না। তবে পড়ার সুবিধার জন্য শেষে রাখাই নিয়ম।

---

## ৯. App.tsx কেন রেখে দিলাম

`App.tsx` চাইলে মুছে দিয়ে সরাসরি `CommonLayout`-কে route-এ দেওয়া যেত। কিন্তু একটা **entry point** রাখা ভালো — `App`-কে **public route-গুলোর entry point** হিসেবে ধরে কাজ করা হয়।

`App`-এ আরও যা রাখা যায় (public route-গুলোর সব জায়গায় লাগে এমন জিনিস):
- **Global loading indicator** — `useNavigation()` দিয়ে page বদলানোর সময় উপরে loading bar দেখানো
- **`<ScrollRestoration />`** — নতুন page-এ গেলে scroll উপরে ফেরানো
- **Toast/notification container** (যেমন shadcn-এর Toaster)
- **Analytics/page view tracking**
- Theme বা অন্য কোনো Provider

> নোট: যা **পুরো app**-এ (admin সহ) লাগবে, যেমন auth বা theme provider, সেটা `main.tsx`-এ `RouterProvider`-এর বাইরে রাখা ভালো — কারণ `App` শুধু `/` group-এর জন্য render হয়, `/admin`-এ হয় না।