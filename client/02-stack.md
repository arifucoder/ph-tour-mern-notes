# Frontend Notes — ০২: Frontend Structure (Auth, State, View)

## Frontend Structure

Backend-এ আমরা **modular pattern** follow করেছিলাম। Frontend-এ সাধারণত একে আমরা **structure** বলি। আমাদের প্রজেক্টে একটা নির্দিষ্ট structure থাকবে এবং সবাই সেটাই follow করবে।

> ⚠️ সংশোধন: "Frontend-এ ওরকম কোনো pattern নেই" — পুরোপুরি ঠিক না। Frontend-এও প্রচলিত কিছু structure/pattern আছে, যেমন:
> - **Feature-based structure** — প্রতিটা feature (যেমন `auth/`, `products/`, `cart/`) নিজের component, hook, API call নিয়ে একটা ফোল্ডারে থাকে। Backend-এর modular pattern-এর সবচেয়ে কাছাকাছি।
> - **Feature-Sliced Design (FSD)** — layer আর slice ভিত্তিক একটা বিস্তারিত methodology।
> - **Atomic Design** — UI component-কে atoms → molecules → organisms → templates → pages হিসেবে সাজানো।
>
> তবে backend framework-এর (যেমন NestJS) মতো React কোনো নির্দিষ্ট structure **চাপিয়ে দেয় না**, তাই team-কে নিজেরা ঠিক করে নিতে হয়।

লক্ষ্য: প্রজেক্ট বড় হলেও যেন structure ঝামেলা না করে এবং ফাইল সাজানো, খুঁজে পাওয়া সহজ থাকে।

Frontend-এ ৩টা জিনিস খুবই গুরুত্বপূর্ণ:
1. **Auth**
2. **State**
3. **View**

---

## ১. State

State দুই রকম:

| | Local / Client State | Remote / Server State |
|---|---|---|
| কী | শুধু frontend-এ থাকা data — modal খোলা কিনা, form input, theme, cart UI ইত্যাদি | Backend/database-এ থাকা data, যার একটা কপি frontend-এ আনা হয় — user list, product list ইত্যাদি |
| কীভাবে manage | `useState`, `useReducer`, Context, Zustand, Redux | Data fetching library — TanStack Query, RTK Query, SWR |

Remote state-কে server-এর সাথে sync রাখাই হলো **data fetching**। শুধু fetch না — caching, loading/error state, refetch, background update, invalidation সবই এর অংশ।

### Data fetching — framework অনুযায়ী আলাদা

প্রতিটা framework-এ (React, Vue, Solid, Svelte) data fetching-এর ধরন আলাদা। React-এর ভেতরেও অনেক উপায় আছে:
- কোনো library ছাড়া — `fetch`/`axios` + `useEffect` (ছোট প্রজেক্টে চলে, কিন্তু caching, race condition, loading state নিজেকে সামলাতে হয়)
- **TanStack Query**
- **RTK Query** (Redux Toolkit-এর অংশ)
- **SWR**

Next.js-এ আবার আলাদাভাবে handle করা হয় (নিচে দেখুন)।

প্রজেক্ট শুরুর আগে ভাবা উচিত: যে framework-এ কাজ করছি, তার data fetching option কী কী এবং প্রতিটার pros/cons কী।

### State management — ecosystem অনুযায়ী option

কোন tool ব্যবহার করবেন সেটা মূলত **ecosystem**-এর ওপর নির্ভর করে।

**React-এ:**
- Client state: **Zustand**, **Redux (Redux Toolkit)**, Jotai, Context API
- Server state / data fetching: **TanStack Query**, **RTK Query**, SWR

**অন্য framework-এ:**
> ⚠️ সংশোধন: "Svelte, Vue-তে এগুলো পাবেন না" — আংশিক ভুল।
> - **TanStack Query** framework-agnostic: React, **Vue**, **Svelte**, **Solid**, Angular সবগুলোর জন্য official adapter আছে।
> - Client state-এর জন্য প্রতিটা framework-এর নিজস্ব tool আছে: Vue-তে **Pinia**, Svelte-এ built-in **stores/runes**, Solid-এ **signals/stores**। Zustand/Redux মূলত React ecosystem-এর জন্য জনপ্রিয়।

**কোনটা নেব — সাধারণ নিয়ম:**
- Global client state খুব বেশি না লাগলে → **TanStack Query** (server state) + `useState`/Context বা হালকা কিছু (যেমন Zustand) client state-এর জন্য।
- Global client state অনেক বেশি ও জটিল হলে → **Redux Toolkit + RTK Query** (একই ecosystem-এ দুটোই পাওয়া যায়)।

> নোট: TanStack Query কোনো client state management library না — এটা **server state** manage করে। তাই "state management লাগবে না, তাই TanStack" — কথাটা আসলে বোঝায় "আলাদা global store লাগবে না, server state-এর জন্য TanStack যথেষ্ট"।

### Next.js-এ State ও Data Fetching

- Next.js-এ Redux ব্যবহার **করা যায়**, তবে App Router-এ **Server Components** দিয়ে server-এই data fetch করা যায় বলে অনেক ক্ষেত্রে বড় global store লাগে না।
- Next.js-এর নিজস্ব উপায়: Server Component-এ সরাসরি `async`/`await fetch`, Next-এর caching ও revalidation, এবং data পরিবর্তনের জন্য **Server Actions**।

> ⚠️ সংশোধন: "Next.js-এ TanStack দিয়ে করলে শুধু client side হয়, তাই Next-এর দেওয়া উপায়টাই করতে হবে" — পুরোপুরি ঠিক না।
> - TanStack Query **SSR support করে** — server-এ prefetch করে `HydrationBoundary` দিয়ে client-এ hydrate করা যায়।
> - Next-এর built-in fetching ব্যবহার করা **বাধ্যতামূলক না**, তবে সাধারণত **recommended**। প্রচলিত approach: initial data → Server Component দিয়ে; client-side interactive জিনিস (infinite scroll, polling, optimistic update) → TanStack Query।

---

## ২. View (UI / Styling)

View-এর জন্য অনেক option:
- **Raw CSS** (বা CSS Modules, Sass)
- **Material UI (MUI)**
- **Bootstrap**
- **Tailwind CSS** (শুধু utility class)
- **shadcn/ui**
- **Headless library**

প্রজেক্টের জন্য যেটা সবচেয়ে ভালো, সেটা বেছে নেওয়া উচিত। এই সিদ্ধান্ত গুরুত্বপূর্ণ কারণ এর ওপর **প্রজেক্টের timeline** অনেকটা নির্ভর করে।

Frontend-এ সাধারণত সবচেয়ে বেশি সময় যায় **design/view** অংশে। Backend dev একটা কাজ ২-৩ দিনে করলেও একই feature-এর frontend অংশে ৬-৭ দিন লাগতে পারে, কারণ frontend dev-কে backend-এর API integrate করার পাশাপাশি UI, responsiveness, loading/error state সবকিছু নিয়ে কাজ করতে হয়।
> নোট: এটা একটা সাধারণ অভিজ্ঞতা, নিয়ম না — feature আর team অনুযায়ী সময় অনেক বদলায়।

### Option-গুলোর তুলনা

| Option | Control | সময় | মন্তব্য |
|---|---|---|---|
| Raw CSS | সবচেয়ে বেশি | সবচেয়ে বেশি | সবকিছু নিজে বানাতে হয় |
| Tailwind | বেশি | মাঝারি | Utility class দিয়ে দ্রুত custom design, কিন্তু component নিজে বানাতে হয় |
| Bootstrap | কম | কম | Ready component, কিন্তু সব সাইট দেখতে একরকম লাগে |
| Material UI | মাঝারি | কম | Ready component, Material Design look |
| shadcn/ui | বেশি | কম–মাঝারি | Ready component + পুরো customization |
| Headless | সবচেয়ে বেশি (design-এ) | মাঝারি | Behavior ready, design পুরোটা নিজের |

**Material UI:** অনেক ready component দেয়, তাই সময় কম লাগে।
> ⚠️ সংশোধন: "Control তেমন নেই, MUI যা দেয় সেভাবেই করতে হয়" — পুরোপুরি ঠিক না। MUI-তে theme, `sx` prop, `styled()` দিয়ে customize করা যায়। তবে **Material Design-এর look থেকে বের হয়ে সম্পূর্ণ নিজের design বানানো কঠিন** এবং component-এর ভেতরের structure বদলানো যায় না।

**shadcn/ui:** তুলনামূলক ভালো, কারণ অনেক ready component দেয় এবং প্রতিটা component অনেক customizable।
> সংশোধন/সংযোজন: shadcn/ui আসলে npm package হিসেবে install হওয়া কোনো library না। CLI দিয়ে component-এর **source code সরাসরি আপনার প্রজেক্টে copy হয়**, তাই কোডটা আপনার নিজের — যেভাবে খুশি বদলাতে পারবেন। এটা **Radix UI** (headless) + **Tailwind CSS** দিয়ে তৈরি।

**Headless library:** যখন click handle, modal খোলা/বন্ধ, keyboard navigation, accessibility — এই behavior-গুলো নিজে বানাতে চান না, কিন্তু design পুরোটা নিজে করতে চান, তখন headless library ব্যবহার করা যায়। যেমন:
- **React Aria** (Adobe) — নোটে "react area" লেখা ছিল, সঠিক নাম React Aria
- **Base UI** (আগের "MUI Base")
- **Radix UI**
- **Headless UI** (Tailwind team-এর)

---

## ৩. Auth

Authentication অনেকভাবে করা যায়:
- **Email/password based**
- **Social login** (Google, GitHub, Facebook — OAuth)
- **Custom** (OTP, magic link ইত্যাদি)

আমাদের backend-এ auth আগেই **Passport.js** দিয়ে করা হয়েছে।

> ⚠️ সংশোধন: "Frontend-এ চাইলে Better Auth ব্যবহার করা যায়" — Better Auth শুধু frontend library না; এটা একটা **full-stack TypeScript auth library** (server + client দুই অংশই আছে)। এটা ব্যবহার করলে backend-এর auth-ও এর মাধ্যমে হয়। আমাদের backend-এ Passport.js থাকায় এটা লাগবে না।

### Login system-এর ধরন

#### ১) Session based

> ⚠️ সংশোধন: "যতক্ষণ login থাকব ততক্ষণ logged in থাকব" — এটা session-based-এর সংজ্ঞা না (token-based-এও এটা হয়)।

সঠিক ধারণা:
- Login করলে **server একটা session তৈরি করে নিজের কাছে রাখে** (memory, database বা Redis-এ)।
- Client-কে শুধু একটা **session ID** cookie-তে পাঠানো হয়।
- প্রতিটা request-এ browser সেই cookie পাঠায়, server session store-এ ID খুঁজে বোঝে user কে।
- **Stateful** — server-কে session মনে রাখতে হয়। Logout বা session বাতিল করা সহজ (server থেকে মুছে দিলেই হয়)।

#### ২) Token based (আমাদের প্রজেক্টে এটা)

Flow:
1. Frontend login-এর জন্য backend-এ request পাঠায় (email + password)।
2. Backend password check করে, সঠিক হলে একটা **signed token** (সাধারণত **JWT**) তৈরি করে দেয়।
3. Token frontend-এ আসে, এবং পরের প্রতিটা request-এর সাথে যায়।
4. **Backend** প্রতিটা request-এ token-এর signature ও expiry verify করে — valid হলে access দেয়, invalid হলে `401 Unauthorized`।

- **Stateless** — server-কে কোনো session মনে রাখতে হয় না, token-এর ভেতরেই তথ্য থাকে।

> ⚠️ সংশোধন (গুরুত্বপূর্ণ):
> - নোটে লেখা ছিল "valid থাকলে login করতে দিব না" — উল্টো হয়ে গেছে। **Valid হলে access দেওয়া হয়**, invalid হলে না।
> - Token valid কিনা সেটা **frontend check করে না, backend করে।** Frontend-এ check করা নিরাপদ না, কারণ frontend-এর কোড user বদলাতে পারে।
> - তাছাড়া আমরা token **httpOnly cookie**-তে রাখছি, তাই frontend-এর JavaScript token **পড়তেই পারবে না**। Frontend login অবস্থা জানতে সাধারণত backend-এর একটা endpoint (যেমন `GET /auth/me`) call করে — সেটা user data দিলে logged in, `401` দিলে logged out।

### Token পাঠানো: Cookie

Token আমরা পাঠাব **cookie** দিয়ে। নোটে "http secure cookie" লেখা ছিল — সঠিকভাবে এগুলো আলাদা আলাদা cookie flag:
- **`HttpOnly`** — JavaScript দিয়ে cookie পড়া যায় না, তাই XSS attack-এ token চুরি করা কঠিন।
- **`Secure`** — cookie শুধু HTTPS দিয়ে যায়।
- **`SameSite`** (`Lax`/`Strict`/`None`) — অন্য site থেকে request-এ cookie যাবে কিনা নিয়ন্ত্রণ করে, CSRF থেকে রক্ষা করে।

**Frontend-এ যা মনে রাখতে হবে:**
- Frontend আর backend আলাদা origin-এ থাকলে request-এ cookie পাঠাতে `fetch`-এ `credentials: 'include'` বা axios-এ `withCredentials: true` দিতে হবে।
- Backend-এর CORS-এ `credentials: true` এবং নির্দিষ্ট origin দিতে হবে (credentials-এর সাথে `*` চলে না)। — আগের নোটের CORS অংশের সাথে সম্পর্কিত।
- সাধারণত দুটো token ব্যবহার হয়: স্বল্পমেয়াদী **access token** আর দীর্ঘমেয়াদী **refresh token**। Access token expire হলে refresh token দিয়ে নতুন access token নেওয়া হয়।