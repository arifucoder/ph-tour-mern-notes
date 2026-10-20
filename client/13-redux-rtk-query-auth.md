# Frontend Notes — ১৩: Redux Toolkit + RTK Query + Axios দিয়ে Authentication

এই নোটে আমাদের Register → Verify (OTP) → Login → Logout পুরো flow-টা backend-এর সাথে জোড়া লাগাব। সাথে বুঝব token কেন cookie-তে রাখা হয়, frontend থেকে cookie কেন set হচ্ছিল না, আর logout-এর পরেও Navbar কেন বদলাচ্ছিল না।

Docs:
- Redux Toolkit Quick Start: https://redux.js.org/tutorials/quick-start
- RTK Query Overview: https://redux-toolkit.js.org/rtk-query/overview
- Axios: https://axios.rest/pages/getting-started/first-steps
- API Slice Utils (`resetApiState`): https://redux-toolkit.js.org/rtk-query/api/created-api/api-slice-utils

এই নোট ১২ নম্বর নোটের (React Hook Form + Zod) পরের ধাপ। সেখানে বানানো `RegisterForm`, `LoginForm`, `auth.schema.ts` আর `PasswordInput` এখানেও একই থাকছে, শুধু `onSubmit`-এর ভেতরে API call বসবে।

---

## ০. আগে পুরো ছবিটা দেখে নিই

অনেকগুলো নতুন জিনিস একসাথে আসছে, তাই আগে দেখি কে কী কাজ করে:

| অংশ | কাজ | উদাহরণ (রেস্টুরেন্ট) |
|---|---|---|
| **Redux store** | পুরো app-এর একটা কেন্দ্রীয় ভাণ্ডার, যেকোনো component সেখান থেকে data নিতে পারে | রেস্টুরেন্টের রান্নাঘরের তাক |
| **RTK Query** | API call, loading/error state, **cache** সব নিজে সামলায় | Waiter: order নেয়, খাবার আনে, মনে রাখে কে কী নিয়েছে |
| **Axios** | আসল HTTP request পাঠায় (fetch-এর বিকল্প) | রান্নাঘরে যাওয়ার দরজা |
| **Interceptor** | প্রতিটা request/response দরজা পার হওয়ার সময় থামিয়ে দেখে | দরজার পাহারাদার |
| **httpOnly cookie** | token রাখে, JavaScript পড়তে পারে না, browser নিজে প্রতি request-এ পাঠায় | হাতে বাঁধা wristband, খুলে কাউকে দেওয়া যায় না |

### Request যায় কোন পথে

```
Component (LoginForm)
   │  login({ email, password })          ← RTK Query-র hook
   ▼
auth.api.ts  (endpoint: url, method, data)
   ▼
baseApi.ts   (createApi → baseQuery)
   ▼
axiosBaseQuery.ts   (RTK Query-র ভাষা → axios-এর ভাষা)
   ▼
lib/axios.ts  (axiosInstance: baseURL, withCredentials, interceptors)
   ▼
Backend (Express + Passport)  → response + Set-Cookie
   ▼
ফেরার পথে উল্টো দিকে → Component-এ data/error
```

### নতুন ফাইলগুলো

```
src/
├── config/
│   └── index.ts                ← .env থেকে baseUrl
├── lib/
│   └── axios.ts                ← axiosInstance + interceptors
├── redux/
│   ├── store.ts
│   ├── hook.ts                 ← useAppDispatch, useAppSelector
│   ├── baseApi.ts              ← createApi (একটাই)
│   ├── axiosBaseQuery.ts
│   └── features/
│       └── auth/
│           └── auth.api.ts     ← login, register, otp, me, logout
├── types/
│   ├── index.ts                ← IResponse, TApiError + re-export
│   └── auth.type.ts
├── utils/
│   └── getErrorMessage.ts
└── pages/
    └── Verify.tsx
```

> নোট: ৬ নম্বর নোটে ফাইলের নাম `hooks.ts` লেখা ছিল, teacher এখানে `hook.ts` দিয়েছেন। যেকোনো একটা নাম রাখলেই হবে, শুধু import-এ একই নাম ব্যবহার করবেন। এই নোটে teacher-এরটা (`hook.ts`) রাখা হলো।

---

## ১. Install

```sh
bun add @reduxjs/toolkit react-redux axios
bunx --bun shadcn@latest add sonner card input-otp
```

| Package | কাজ |
|---|---|
| `@reduxjs/toolkit` | Redux লেখার আধুনিক উপায়। `configureStore`, `createSlice` আর **RTK Query** (`createApi`) এর ভেতরেই আছে, আলাদা install লাগে না |
| `react-redux` | React component-কে Redux store-এর সাথে জোড়া লাগায় (`Provider`, `useSelector`, `useDispatch`) |
| `axios` | HTTP client, request পাঠানোর জন্য |
| `sonner` (shadcn) | Toast notification ("Logged in successfully") |
| `card`, `input-otp` (shadcn) | Verify page-এর জন্য |

> ⚠️ সংশোধন: নোটে `axios` install-এর কথা অনেক পরে ছিল, কিন্তু `lib/axios.ts` লেখার আগেই এটা লাগবে। না থাকলে import error দেবে। তাই সব install শুরুতে একসাথে রাখা হলো।

---

## ২. Redux Store

```ts
// src/redux/store.ts  (প্রথম version, পরে RTK Query যোগ হবে)
import { configureStore } from "@reduxjs/toolkit";

export const store = configureStore({
  reducer: {},
});

export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;
```

- **`configureStore`:** store বানায়। ভেতরে নিজে থেকেই Redux DevTools চালু করে আর কিছু দরকারি middleware (যেমন `thunk`) বসিয়ে দেয়।
- **`reducer: {}`:** store-এর ভেতরে কোন কোন "তাক" থাকবে। এখন খালি, একটু পরে RTK Query-র তাক বসবে।
- **`RootState`:** পুরো store-এর data দেখতে কেমন, তার TypeScript type। `store.getState()` যা return করে, `ReturnType` দিয়ে তার type বের করে নেওয়া হচ্ছে। Store-এ নতুন কিছু যোগ হলে এই type নিজে থেকেই update হয়।
- **`AppDispatch`:** আমাদের store-এর `dispatch` function-এর type।

---

## ৩. `main.tsx`-এ Redux Provider (আর Toaster)

```tsx
// src/main.tsx
import { StrictMode } from "react";
import { createRoot } from "react-dom/client";
import { Provider as ReduxProvider } from "react-redux";
import { RouterProvider } from "react-router";
import "./index.css";
import { Toaster } from "@/components/ui/sonner";
import { ThemeProvider } from "@/providers/theme.provider";
import { store } from "@/redux/store";
import { router } from "@/routes";

createRoot(document.getElementById("root")!).render(
  <StrictMode>
    <ReduxProvider store={store}>
      <ThemeProvider defaultTheme="system" storageKey="phovia-ui-theme">
        <RouterProvider router={router} />
        <Toaster />
      </ThemeProvider>
    </ReduxProvider>
  </StrictMode>,
);
```

- **`ReduxProvider` সবার বাইরে কেন:** Provider-এর ভেতরে যে component থাকবে, শুধু সেই store পাবে। সবার বাইরে রাখলে route, layout, Navbar সবাই store পায়।
- **`Provider as ReduxProvider`:** `react-redux`-এর component-এর নাম শুধু `Provider`। পরে আরও Provider (Theme, Auth …) আসবে, তাই নাম বদলে `ReduxProvider` রাখা হলো যাতে পড়েই বোঝা যায় কোনটা কী।
- **`<Toaster />` `ThemeProvider`-এর ভেতরে:** toast-কে বর্তমান theme (light/dark) জানতে হয়, তাই theme-এর context-এর ভেতরে থাকতে হবে। `RouterProvider`-এর পাশে রাখলে যেকোনো page থেকে `toast()` ডাকলে দেখাবে।

> ⚠️ সংশোধন: নোটের কোডে import ছিল `"./providers/theme.provider.tsx"`, `"./redux/store.ts"` এভাবে (relative path + extension)। আমাদের নিয়মে import হবে `@/` দিয়ে, extension ছাড়া, যেমন `@/redux/store`।

---

## ৪. Typed hook: `hook.ts`

```ts
// src/redux/hook.ts
import { useDispatch, useSelector } from "react-redux";
import type { AppDispatch, RootState } from "./store";

export const useAppSelector = useSelector.withTypes<RootState>();
export const useAppDispatch = useDispatch.withTypes<AppDispatch>();
```

সাধারণ `useSelector` জানে না আমাদের store-এ কী কী আছে। তাই প্রতিবার লিখতে হতো `useSelector((state: RootState) => ...)`। `.withTypes<RootState>()` দিয়ে একবার type বসিয়ে নতুন hook বানালে আর লিখতে হয় না, আর editor-এ suggestion-ও আসে।

**নিয়ম:** project-এ সবসময় `useAppSelector` আর `useAppDispatch` ব্যবহার করব, সাধারণ `useSelector`/`useDispatch` না।

---

## ৫. Slice কেন এখন লাগছে না → RTK Query

Redux-এ data দুই রকম:

| ধরন | মানে | উদাহরণ | কী দিয়ে সামলাব |
|---|---|---|---|
| **Client state** (local state) | শুধু browser-এ থাকে, server জানে না | sidebar খোলা না বন্ধ, cart-এর জিনিস, filter | `createSlice` |
| **Server state** | আসল data server-এ থাকে, আমরা শুধু এনে দেখাই/পাঠাই | logged-in user, tour list, booking | **RTK Query** |

আমাদের এখনকার কাজ: form থেকে data নিয়ে backend-এ পাঠানো আর backend থেকে user-এর তথ্য আনা। এটা পুরোটাই **server state**। তাই এই মুহূর্তে কোনো slice লাগছে না, সরাসরি RTK Query দিয়ে কাজ।

RTK Query না থাকলে প্রতিটা API-র জন্য নিজে লিখতে হতো: `useState` দিয়ে `loading`, `error`, `data`, `useEffect` দিয়ে fetch, আবার একই data অন্য component-এ লাগলে আবার fetch। RTK Query এগুলো সব নিজে করে, আর একই data দুই জায়গায় লাগলে একবারই আনে (cache)।

---

## ৬. `.env` আর `config`

```env
# .env  (project-এর root-এ, src-এর বাইরে)
VITE_BASE_URL=http://localhost:5000/api/v1
```

```ts
// src/config/index.ts
const config = {
  baseUrl: import.meta.env.VITE_BASE_URL,
};

export default config;
```

### Teacher-এর তিনটা নিয়ম, ব্যাখ্যা সহ

**১. নামের শুরুতে অবশ্যই `VITE_`**
Vite নিরাপত্তার জন্য `.env`-এর সব variable browser-এর কোডে দেয় না। শুধু যেগুলোর নাম `VITE_` দিয়ে শুরু, সেগুলোই `import.meta.env` দিয়ে পাওয়া যায়। `BASE_URL=...` লিখলে `undefined` আসবে।

> ⚠️ সংশোধন (গুরুত্বপূর্ণ): নোটে লেখা ছিল "public যেগুলো সেগুলোর সামনে VITE লিখতে হয়"। কথাটা উল্টো দিক থেকেও বোঝা দরকার: **`VITE_` দিলেই সেটা public হয়ে যায়।** Build-এর সময় মানটা JavaScript ফাইলে সরাসরি বসে যায়, যেকোনো user DevTools খুলে দেখতে পারে। তাই frontend-এর `.env`-এ কখনো secret (JWT secret, DB password, Stripe secret key) রাখা যাবে না। এখানে শুধু এমন জিনিস রাখব যেটা সবাই দেখলেও সমস্যা নেই, যেমন API-র URL।

**২. URL-এর শেষে `/` না**
আমাদের endpoint-গুলো `/` দিয়ে শুরু (`/auth/login`)। Base URL-এর শেষেও `/` থাকলে জোড়া লাগিয়ে হয় `…/api/v1//auth/login`। Axios নিজে এই ডাবল slash ঠিক করে নেয়, কিন্তু যেখানে আমরা নিজে string জোড়া লাগাই (যেমন Google login-এ `` `${config.baseUrl}/auth/google` ``), সেখানে ভাঙবে। তাই নিয়ম: শেষে slash না।

**৩. `dotenv` install লাগে না**
Backend-এ (Node) `.env` পড়তে `dotenv` লাগে। Vite নিজেই `.env` পড়ে, তাই frontend-এ লাগে না।

> ➕ সংযোজন:
> - `.env` বদলানোর পর **dev server restart** করতে হবে (`Ctrl + C`, তারপর `bun run dev`)। নাহলে পুরনো মানই থাকবে।
> - `.env` git-এ push করবেন না (`.gitignore`-এ আছে কিনা দেখে নিন)। বদলে একটা `.env.example` রাখুন, যাতে অন্যরা জানে কোন variable লাগবে:
>   ```env
>   VITE_BASE_URL=
>   ```
> - TypeScript-কে variable-এর নাম চেনাতে `src/vite-env.d.ts`-এ (না থাকলে বানিয়ে নিন) এটা যোগ করুন। তাহলে `import.meta.env.`-এর পরে suggestion আসবে আর type হবে `string`:
>   ```ts
>   /// <reference types="vite/client" />
>
>   interface ImportMetaEnv {
>     readonly VITE_BASE_URL: string;
>   }
>
>   interface ImportMeta {
>     readonly env: ImportMetaEnv;
>   }
>   ```

---

## ৭. Axios instance — `lib/axios.ts`

### Axios কেন, fetch থাকতে?

| | `fetch` | `axios` |
|---|---|---|
| Base URL | প্রতিবার পুরো URL লিখতে হয় | একবার `baseURL` দিলেই হয় |
| JSON | `res.json()` নিজে করতে হয়, body-তে `JSON.stringify` | নিজে থেকেই করে |
| Error | 404/500 হলেও error throw করে না, `res.ok` দেখতে হয় | 2xx না হলেই error throw করে |
| Cookie পাঠানো | `credentials: "include"` | `withCredentials: true` |
| **Interceptor** | নেই | **আছে** — সবচেয়ে বড় সুবিধা |

### কোড

```ts
// src/lib/axios.ts
import config from "@/config";
import axios from "axios";

export const axiosInstance = axios.create({
  baseURL: config.baseUrl,
  withCredentials: true, // cookie পাঠানো/নেওয়ার অনুমতি (১৭ নম্বর section)
});

// Request interceptor: প্রতিটা request backend-এ যাওয়ার ঠিক আগে এখানে আসে
axiosInstance.interceptors.request.use(
  (requestConfig) => {
    // এখানে request বদলানো যায় (header যোগ, log ইত্যাদি)
    return requestConfig; // ফেরত না দিলে request যাবেই না
  },
  (error) => Promise.reject(error),
);

// Response interceptor: backend-এর উত্তর আমাদের কোডে পৌঁছানোর ঠিক আগে এখানে আসে
axiosInstance.interceptors.response.use(
  (response) => response, // 2xx হলে এখানে
  (error) => Promise.reject(error), // 2xx না হলে এখানে
);
```

- **`axios.create()`:** একটা নিজস্ব axios বানায়, যার নিজের setting আছে (baseURL, withCredentials)। পুরো app এই একটা `axiosInstance` দিয়েই request পাঠাবে, তাই setting একবার দিলেই সব জায়গায় কাজ করে।

> ⚠️ সংশোধন:
> - **নামের সংঘর্ষ:** উপরে `import config from "@/config"`, আবার interceptor-এর parameter-এর নামও `config`। ভেতরের `config` বাইরেরটাকে ঢেকে দেয় (shadowing)। কোড চলে, কিন্তু পড়তে গিয়ে গুলিয়ে যায়: কোন `config` কোনটা? তাই parameter-এর নাম `requestConfig` রাখা হলো।
> - **`console.log("Axios", config)` সরানো হয়েছে:** request config-এর ভেতরে `data` থাকে। Login-এর সময় এতে **password** থাকে, আর সেটা console-এ ছাপা হয়ে যেত। শেখার সময় দেখার জন্য সাময়িক log ঠিক আছে, কিন্তু কাজ শেষে মুছে দেবেন।
> - **`withCredentials: true`** শুরু থেকেই রাখা হলো। নোটে এটা পরে যোগ হয়েছিল (১৭ নম্বর section-এ কারণ আছে)।

### Interceptor কী, সহজ ভাষায়

ধরুন আপনার বাসা থেকে সব চিঠি একটাই পোস্ট অফিস দিয়ে যায় আর আসে। পোস্ট অফিসে একজন কর্মী বসানো আছে:
- **যাওয়ার সময় (request interceptor):** প্রতিটা চিঠি খামে ভরার আগে দেখে, চাইলে ঠিকানায় কিছু যোগ করে বা stamp লাগায়।
- **আসার সময় (response interceptor):** প্রতিটা চিঠি আপনার হাতে দেওয়ার আগে খুলে দেখে। কোনো সমস্যা থাকলে আপনাকে না জানিয়েই ঠিক করার চেষ্টা করে।

আমাদের app-এর সব request `axiosInstance` দিয়েই যায়, তাই interceptor-এ একবার কিছু লিখলে **সব** request-এর ওপর কাজ করে। আলাদা আলাদা component-এ লিখতে হয় না।

**কোথায় কাজে লাগে:**

| Interceptor | ব্যবহার |
|---|---|
| Request | header যোগ করা, request log করা, global loading spinner চালু করা |
| Response | সব error এক জায়গায় ধরা, 401 পেলে login page-এ পাঠানো, **token expire হলে নতুন token এনে আবার চেষ্টা করা** |

### Teacher যেটা বললেন: token expire হলে নতুন token আনা

আমাদের backend দুটো token দেয়:

| Token | মেয়াদ | কাজ |
|---|---|---|
| **Access token** | কম (যেমন ১ দিন বা কয়েক ঘণ্টা) | প্রতিটা protected request-এ যায়, "আমি কে" প্রমাণ করে |
| **Refresh token** | বেশি (যেমন ৩০ দিন) | শুধু একটা কাজ: access token-এর মেয়াদ শেষ হলে নতুন access token আনা |

Access token-এর মেয়াদ কম রাখা হয়, যাতে কোনোভাবে চুরি হলেও বেশিক্ষণ কাজে না লাগে। কিন্তু প্রতিবার মেয়াদ শেষে user-কে আবার login করতে বললে বিরক্ত হবে। তখন response interceptor এভাবে কাজ করে:

```
১. Component → GET /user/me  (access token-এর মেয়াদ শেষ)
২. Backend → 401 "token expired"
৩. Response interceptor error ধরে ফেলে, component পর্যন্ত যেতে দেয় না
৪. Interceptor → POST /auth/refresh-token  (refresh token cookie-তে আপনাআপনি যায়)
৫. Backend → নতুন access token cookie-তে বসিয়ে দেয়
৬. Interceptor → আগের GET /user/me আবার পাঠায়
৭. এবার সফল → component data পায়
```

User কিছুই টের পায় না, শুধু request-টা একটু দেরিতে আসে।

> ➕ সংযোজন: ধারণাটা কোডে দেখতে এমন হয় (এখনই বসাতে হবে না, teacher পরে দেখাবেন; refresh route-এর নাম আপনার backend থেকে মিলিয়ে নেবেন):
> ```ts
> import type { InternalAxiosRequestConfig } from "axios";
>
> axiosInstance.interceptors.response.use(
>   (response) => response,
>   async (error) => {
>     const originalRequest = error.config as InternalAxiosRequestConfig & { _retry?: boolean };
>
>     const isTokenExpired = error.response?.status === 401;
>     const isRefreshCall = originalRequest.url?.includes("/auth/refresh-token");
>
>     // _retry: একই request-এর জন্য একবারই চেষ্টা, নাহলে অসীম loop
>     if (isTokenExpired && !isRefreshCall && !originalRequest._retry) {
>       originalRequest._retry = true;
>       try {
>         await axiosInstance.post("/auth/refresh-token");
>         return axiosInstance(originalRequest); // আগের request আবার
>       } catch (refreshError) {
>         return Promise.reject(refreshError); // refresh token-ও মেয়াদোত্তীর্ণ → আবার login লাগবে
>       }
>     }
>
>     return Promise.reject(error);
>   },
> );
> ```

---

## ৮. `axiosBaseQuery` — RTK Query আর Axios-এর মধ্যে অনুবাদক

### সমস্যাটা কী

RTK Query নিজে HTTP request পাঠায় না। সে একটা **`baseQuery`** function ডাকে আর বলে "এই url-এ এই data পাঠাও"। তারপর উত্তর চায় নির্দিষ্ট আকারে:

- সফল হলে: `{ data: ... }`
- ব্যর্থ হলে: `{ error: ... }`

RTK Query-র সাথে একটা তৈরি baseQuery আসে, `fetchBaseQuery` (ভেতরে `fetch` ব্যবহার করে)। আমরা `fetch`-এর বদলে axios চাই, তাই নিজে একটা baseQuery লিখতে হচ্ছে, যেটা RTK Query-র কথা শুনে axios দিয়ে request পাঠাবে আর উত্তর RTK Query-র পছন্দের আকারে ফেরত দেবে। এটাই `axiosBaseQuery`। (RTK Query docs-এ "axios base query" লিখে search করলে এই কোড পাওয়া যায়।)

### কোড

```ts
// src/redux/axiosBaseQuery.ts
import { axiosInstance } from "@/lib/axios";
import type { TApiError } from "@/types";
import type { BaseQueryFn } from "@reduxjs/toolkit/query";
import type { AxiosError, AxiosRequestConfig } from "axios";

// endpoint থেকে আমরা যা যা পাঠাতে পারি
type AxiosBaseQueryArgs = {
  url: string;
  method?: AxiosRequestConfig["method"];
  data?: AxiosRequestConfig["data"];
  params?: AxiosRequestConfig["params"];
  headers?: AxiosRequestConfig["headers"];
};

const axiosBaseQuery =
  (): BaseQueryFn<AxiosBaseQueryArgs, unknown, TApiError> =>
  async ({ url, method, data, params, headers }) => {
    try {
      const result = await axiosInstance({ url, method, data, params, headers });
      return { data: result.data }; // সফল → { data }
    } catch (axiosError) {
      const err = axiosError as AxiosError;
      return {
        // ব্যর্থ → { error }
        error: {
          status: err.response?.status,
          data: (err.response?.data as TApiError["data"]) ?? { message: err.message },
        },
      };
    }
  };

export default axiosBaseQuery;
```

### লাইন ধরে ধরে

**১. function-এর ভেতরে function কেন (`() => async (...) => {}`)**

```ts
const axiosBaseQuery = () => async (args) => { ... };
//                     ↑ বাইরের        ↑ ভেতরের
```

- **বাইরের function** একবারই চলে, `baseApi.ts`-এ: `baseQuery: axiosBaseQuery()`। এটা ভেতরের function-টা বানিয়ে ফেরত দেয়।
- **ভেতরের function** প্রতিটা API call-এ চলে। RTK Query এটাকে ডাকে endpoint-এর `{ url, method, data }` দিয়ে।

বাইরের function রাখার সুবিধা: চাইলে setting পাঠানো যায়, যেমন `axiosBaseQuery({ baseUrl: "..." })`। `fetchBaseQuery({ baseUrl })`-ও ঠিক এভাবেই লেখা। আমাদের axiosInstance-এ baseURL আগেই দেওয়া, তাই এখানে কিছু পাঠাচ্ছি না, খালি `()`।

**২. `BaseQueryFn<Args, Result, Error>` — তিনটা type**

| স্থান | আমাদের মান | মানে |
|---|---|---|
| ১ম: Args | `AxiosBaseQueryArgs` | endpoint থেকে কী কী পাঠানো যাবে (`url` বাধ্যতামূলক, বাকিগুলো `?` মানে ঐচ্ছিক) |
| ২য়: Result | `unknown` | সফল হলে data কেমন। সব endpoint-এর আলাদা, তাই এখানে `unknown`; আসল type প্রতিটা endpoint-এ দেব (১১ নম্বর section) |
| ৩য়: Error | `TApiError` | ব্যর্থ হলে error কেমন দেখতে |

**`AxiosRequestConfig["method"]` মানে কী:** axios-এর config type-এর ভেতরে `method`-এর যে type আছে (`"GET" | "POST" | …`), সেটাই তুলে আনা। নিজে আলাদা করে লিখতে হলো না।

**৩. `try` — সফল হলে**
`axiosInstance(...)` request পাঠায়। Axios-এর উত্তরের ভেতরে অনেক কিছু থাকে (`status`, `headers`, `config` …), আসল body থাকে `result.data`-তে। শুধু সেটাই `{ data }` আকারে ফেরত দিচ্ছি।

**৪. `catch` — ব্যর্থ হলে**
Axios 2xx ছাড়া সব status-এ error throw করে। আমরা সেটা ধরে `{ error: { status, data } }` আকারে ফেরত দিই:
- `status`: 400, 401, 403, 500 …
- `data`: backend-এর পাঠানো error body (`{ message, errorSources }`)। Backend পর্যন্ত request পৌঁছায়নি (internet নেই, server বন্ধ) হলে `err.response` থাকে না, তখন axios-এর নিজের message (`"Network Error"`) রাখা হয়।

> ⚠️ সংশোধন:
> - **Error-এর type `unknown` থেকে `TApiError`:** এতে component-এ error-এর ভেতরে কী আছে TypeScript জানে।
> - **`err.response?.data || err.message` থেকে `?? { message: err.message }`:** আগে network error হলে `data` হয়ে যেত একটা string (`"Network Error"`), অথচ component-এ আমরা পড়ি `error.data.message`। String-এর ভেতরে `.message` নেই, তাই user শুধু সাধারণ fallback message দেখত। এখন সবসময় object, তাই `"Network Error"` সঠিকভাবে toast-এ দেখাবে।
> - **`import { AxiosError, type AxiosRequestConfig }` থেকে `import type { AxiosError, AxiosRequestConfig }`:** দুটোই শুধু type হিসেবে ব্যবহার হচ্ছে (আমাদের নিয়ম: type import-এ `import type`)।
> - **`url: url` থেকে শুধু `url`:** নাম একই হলে JavaScript-এ একবার লিখলেই হয় (shorthand)।

### `body` না, `data` কেন

| | `fetchBaseQuery` | আমাদের `axiosBaseQuery` |
|---|---|---|
| Request-এর body পাঠানোর key | `body: userInfo` | `data: userInfo` |

কারণ fetch-এর ভাষায় শব্দটা `body`, axios-এর ভাষায় `data`। আমরা `axiosBaseQuery`-র Args-এ `data` রেখেছি, তাই endpoint-এ `data: userInfo` লিখতে হবে। ভুল করে `body` লিখলে TypeScript error দেবে আর backend খালি body পাবে।

> ⚠️ সংশোধন: নোটে লেখা ছিল fetchBaseQuery-তে body দিলে "network tab-এ email password দেখা যাবে"। আসলে **যেকোনো** উপায়ে পাঠালেই (fetch বা axios) DevTools-এর Network tab → request → **Payload** tab-এ নিজের পাঠানো email/password দেখা যায়। এটা স্বাভাবিক, কারণ এটা আপনার নিজের browser। বাইরের কেউ যাতে রাস্তায় পড়তে না পারে, সেটা নিশ্চিত করে **HTTPS** (deploy-করা backend-এ)।

---

## ৯. `baseApi.ts` — একটাই API, বাকিরা এতে যোগ হয়

```ts
// src/redux/baseApi.ts
import { createApi } from "@reduxjs/toolkit/query/react";
import axiosBaseQuery from "./axiosBaseQuery";

export const baseApi = createApi({
  reducerPath: "baseApi",
  baseQuery: axiosBaseQuery(),
  tagTypes: ["USER"],
  endpoints: () => ({}),
});
```

### `createApi`-এর অংশগুলো

| অংশ | কাজ |
|---|---|
| `reducerPath` | Redux store-এর ভেতরে এই API-র cache কোন নামে থাকবে (`state.baseApi`) |
| `baseQuery` | **কী দিয়ে** request পাঠাবে। আমরা `axiosBaseQuery()` |
| `tagTypes` | Cache-এর কোন কোন "লেবেল" ব্যবহার হবে (২১ নম্বর section-এ বিস্তারিত) |
| `endpoints` | কোন কোন API আছে। এখানে খালি, কারণ module pattern-এ প্রতিটা feature নিজের ফাইল থেকে endpoint যোগ করবে |

### Module pattern কেন

সব endpoint এক ফাইলে লিখলে project বড় হলে ফাইলটা হাজার লাইনের হয়ে যায়। তাই:

```
baseApi.ts             ← খালি "কাঠামো", একটাই
   ▲
   ├── features/auth/auth.api.ts      → injectEndpoints (login, register …)
   ├── features/tour/tour.api.ts      → injectEndpoints (getTours …)   (পরে)
   └── features/booking/booking.api.ts                                  (পরে)
```

প্রতিটা feature `baseApi.injectEndpoints()` দিয়ে নিজের endpoint ঢুকিয়ে দেয়। কিন্তু সবাই আসলে **একই** `baseApi`-তে যোগ হয়, তাই store-এ একবারই connect করতে হয়।

> ⚠️ সংশোধন: নোটে "`src/redux/features/auth/auth.api.ts` file-এ" লিখে আবার `baseApi`-র কোড দেওয়া ছিল (`createApi`, `import axiosBaseQuery from "./axiosBaseQuery"`)। ওটা ভুল ফাইল। `createApi` থাকবে **শুধু** `redux/baseApi.ts`-এ। `auth.api.ts`-এ থাকবে `baseApi.injectEndpoints(...)` (১১ নম্বর section)। দুই জায়গায় `createApi` থাকলে দুটো আলাদা API হয়ে যাবে, আর দ্বিতীয়টা store-এ connect না থাকায় কাজই করবে না।

> নোট: `createApi` import হচ্ছে `@reduxjs/toolkit/query/react` থেকে (শেষে `/react`)। এখান থেকে নিলে `useLoginMutation`-এর মতো React hook নিজে থেকে তৈরি হয়। শুধু `@reduxjs/toolkit/query` থেকে নিলে hook তৈরি হবে না।

---

## ১০. Store-এ `baseApi` connect করা

```ts
// src/redux/store.ts  (চূড়ান্ত)
import { configureStore } from "@reduxjs/toolkit";
import { setupListeners } from "@reduxjs/toolkit/query";
import { baseApi } from "./baseApi";

export const store = configureStore({
  reducer: {
    [baseApi.reducerPath]: baseApi.reducer,
  },
  middleware: (getDefaultMiddleware) => getDefaultMiddleware().concat(baseApi.middleware),
});

setupListeners(store.dispatch);

export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;
```

> ⚠️ সংশোধন: নোটে import ছিল `import { baseApi } from "./features/auth.api"`। `baseApi` আছে `./baseApi` ফাইলে। (আর `features/auth.api` path-টাও ভুল, আসল ফাইল `features/auth/auth.api.ts`।)

আপনি তিনটা জিনিস বুঝতে চেয়েছিলেন। একটা একটা করে:

### ক) `[baseApi.reducerPath]: baseApi.reducer` — reducer-এ এভাবে কেন

**Reducer কী:** store-এর একটা অংশ (তাক) কে সামলায়, অর্থাৎ ঐ অংশের data কীভাবে বদলাবে। RTK Query-র cache-ও store-এর ভেতরে থাকে: কোন API call হয়েছে, data কী এসেছে, loading না error। এই cache রাখার জন্য RTK Query-র নিজের একটা reducer আছে: `baseApi.reducer`। এটা store-এ না দিলে cache রাখার জায়গাই থাকবে না।

**`[ ]` (square bracket) কেন:** এটা JavaScript-এর **computed property name**। মানে key-র নাম হবে variable-এর **মান**:

```ts
baseApi.reducerPath  // = "baseApi"

{ [baseApi.reducerPath]: baseApi.reducer }
// একই জিনিস:
{ baseApi: baseApi.reducer }
```

সরাসরি `baseApi: baseApi.reducer` লিখলেও কাজ করবে। কিন্তু পরে কেউ `createApi`-তে `reducerPath` বদলে দিলে store-এর নাম আর মিলবে না, app ভেঙে যাবে। `[baseApi.reducerPath]` লিখলে নাম সবসময় নিজে থেকে মিলে যায়।

Redux DevTools-এ store খুললে এখন দেখবেন:

```
state
└── baseApi
    ├── queries       ← userInfo-এর মতো GET-এর cache
    ├── mutations     ← login, register-এর অবস্থা
    ├── provided      ← কোন tag কোন data দিচ্ছে
    ├── subscriptions ← কোন component কোন data ব্যবহার করছে
    └── config
```

### খ) `middleware` — এটা কেন যোগ করলাম

**Middleware কী:** dispatch করা action আর reducer-এর মাঝখানে বসে থাকা একজন। প্রতিটা action reducer-এ পৌঁছানোর আগে middleware-এর হাত দিয়ে যায়, সে চাইলে বাড়তি কাজ করতে পারে।

**Reducer শুধু data রাখতে পারে, কোনো কাজ (side effect) করতে পারে না।** API call পাঠানো, timer চালানো, অন্য request আবার পাঠানো, এগুলো reducer-এর কাজ না। এগুলো করে `baseApi.middleware`:

| `baseApi.middleware`-এর কাজ | উদাহরণ |
|---|---|
| Cache কতক্ষণ থাকবে হিসাব রাখা | কোনো component আর data ব্যবহার না করলে ৬০ সেকেন্ড পরে cache মুছে দেয় |
| Tag invalidate হলে আবার fetch করা | logout হলে `USER` tag invalidate → `/user/me` আবার আনা |
| Polling, refetchOnFocus ইত্যাদি | নির্দিষ্ট সময় পর পর data আনা |

সহজ কথায়: **reducer = মেমরি, middleware = হাত-পা।** Middleware না দিলে cache থাকবে, কিন্তু auto refetch, cache invalidation কিছুই কাজ করবে না। RTK Query তখন console-এ সতর্কও করে: *"Middleware for RTK-Query API at reducerPath "baseApi" has not been added to the store."*

**`getDefaultMiddleware().concat(...)`:** `configureStore` নিজে কিছু middleware দেয় (thunk, ভুল ধরার কিছু check)। `middleware` লিখলে সেগুলো বাদ পড়ে যায়, তাই আগে `getDefaultMiddleware()` দিয়ে default-গুলো নিয়ে তার শেষে `.concat()` দিয়ে আমাদেরটা জুড়ে দিচ্ছি।

### গ) `setupListeners` — refetchOnFocus/refetchOnReconnect

Docs-এ লেখা: optional, but required for `refetchOnFocus`/`refetchOnReconnect`। মানে:

| Option | কখন আবার data আনবে |
|---|---|
| `refetchOnFocus` | user অন্য tab-এ গিয়ে আবার আমাদের tab-এ ফিরলে |
| `refetchOnReconnect` | internet চলে গিয়ে আবার এলে |

উদাহরণ: user Phovia Tour খোলা রেখে অন্য tab-এ গেল, ১০ মিনিট পর ফিরল। এর মধ্যে tour-এর seat কমে গেছে। `refetchOnFocus` চালু থাকলে tab-এ ফেরামাত্র নতুন data আসবে।

কিন্তু "tab-এ ফিরল" বা "internet এল", এটা জানতে হলে browser-এর `focus`, `visibilitychange`, `online` event শুনতে হয়। **`setupListeners` এই event-গুলো শোনার ব্যবস্থা করে দেয়।** এটা না থাকলে option দুটো চালু করলেও কাজ করবে না।

> নোট: শুধু `setupListeners` দিলেই refetch শুরু হয় না। Option-টাও চালু করতে হয়, `createApi`-তে (সব endpoint-এর জন্য) বা hook-এ (একটার জন্য):
> ```ts
> useUserInfoQuery(undefined, { refetchOnFocus: true });
> ```
> এখন চালু না করলেও `setupListeners` রেখে দিন, পরে লাগবে।

---

## ১১. `auth.api.ts` — endpoint-গুলো

### Types আগে

API কী নেয় আর কী দেয়, তার type। JSON response থেকে খুব সহজে type বানানো যায়: https://transform.tools/json-to-typescript (Postman-এর response কপি করে paste করুন)।

```ts
// src/types/auth.type.ts
export interface IRegister {
  name: string;
  email: string;
  password: string;
  phone?: string;
  address?: string;
}

export interface ILogin {
  email: string;
  password: string;
}

export interface ISendOtp {
  email: string;
}

export interface IVerifyOtp {
  email: string;
  otp: string;
}

// নিচের দুটো উদাহরণ — Postman-এ আপনার backend-এর আসল response দেখে field মিলিয়ে নিন
export interface IUser {
  _id: string;
  name: string;
  email: string;
  role: string;
  phone?: string;
  address?: string;
  isVerified: boolean;
}

export interface ILoginResponse {
  accessToken: string;
  refreshToken: string;
  user: IUser;
}
```

```ts
// src/types/index.ts
export type { ILogin, ILoginResponse, IRegister, ISendOtp, IUser, IVerifyOtp } from "./auth.type";

// Backend-এর সব response এই আকারে আসে, শুধু data-র ভেতরটা আলাদা
export interface IResponse<T> {
  statusCode: number;
  success: boolean;
  message: string;
  data: T;
}

// Backend-এর error এই আকারে আসে (axiosBaseQuery এটাকে { status, data }-এ মোড়ায়)
export type TApiError = {
  status?: number;
  data?: {
    message?: string;
    errorSources?: { path: string; message: string }[];
  };
};
```

- **`IResponse<T>` (generic):** `T` একটা ফাঁকা জায়গা। `IResponse<IUser>` মানে `data`-র ভেতরে `IUser`, `IResponse<null>` মানে `data` খালি। একটা interface দিয়ে সব response-এর type লেখা যায়।
- **`types/index.ts` থেকে re-export:** component-এ সবসময় `@/types` থেকে import করা যায়, ভেতরে কোন ফাইলে আছে মনে রাখতে হয় না।

> ⚠️ সংশোধন: নোটে `TApiError` লেখা ছিল `RegistrationForm.tsx`-এর ভেতরে, আর `LoginForm` সেটা `"./RegistrationForm"` থেকে import করছিল। একটা form-এর type অন্য form থেকে আনলে দুই component অকারণে জড়িয়ে যায়। একাধিক জায়গায় লাগা type থাকবে `types/`-এ (৬ নম্বর নোট)। Verify page-এর কোডেও আপনি `@/types` থেকেই import করেছিলেন, তাই এখানে এনে রাখা হলো।

### Error message বের করার ছোট helper

Register, Login, Verify তিন জায়গাতেই একই লাইন লাগে, তাই একটা utility:

```ts
// src/utils/getErrorMessage.ts
import type { TApiError } from "@/types";

export const getErrorMessage = (err: unknown, fallback: string) => {
  const error = err as TApiError;
  return error?.data?.errorSources?.[0]?.message || error?.data?.message || fallback;
};
```

ক্রম: আগে Zod-এর field error (`errorSources`-এর প্রথমটা), না থাকলে backend-এর সাধারণ `message`, সেটাও না থাকলে আমাদের দেওয়া `fallback`।

### Endpoint-গুলো

```ts
// src/redux/features/auth/auth.api.ts
import { baseApi } from "@/redux/baseApi";
import type { ILogin, ILoginResponse, IRegister, IResponse, ISendOtp, IUser, IVerifyOtp } from "@/types";

export const authApi = baseApi.injectEndpoints({
  endpoints: (builder) => ({
    register: builder.mutation<IResponse<IUser>, IRegister>({
      query: (userInfo) => ({
        url: "/user/register",
        method: "POST",
        data: userInfo,
      }),
    }),

    login: builder.mutation<IResponse<ILoginResponse>, ILogin>({
      query: (userInfo) => ({
        url: "/auth/login",
        method: "POST",
        data: userInfo,
      }),
      invalidatesTags: ["USER"],
    }),

    sendOtp: builder.mutation<IResponse<null>, ISendOtp>({
      query: (userInfo) => ({
        url: "/otp/send",
        method: "POST",
        data: userInfo,
      }),
    }),

    verifyOtp: builder.mutation<IResponse<null>, IVerifyOtp>({
      query: (userInfo) => ({
        url: "/otp/verify",
        method: "POST",
        data: userInfo,
      }),
    }),

    // ...userInfo আর logout ২০ ও ২১ নম্বর section-এ
  }),
});

export const { useRegisterMutation, useLoginMutation, useSendOtpMutation, useVerifyOtpMutation } = authApi;
```

- **`injectEndpoints`:** `baseApi`-তে নতুন endpoint ঢুকিয়ে দেয় আর একই API ফেরত দেয়। তাই `authApi` আর `baseApi` আসলে **একই জিনিস**, শুধু `authApi`-র type-এ auth-এর endpoint-গুলো জানা আছে।
- **`builder.mutation<Response, Arg>`:** প্রথম type = backend কী ফেরত দেবে, দ্বিতীয় type = আমরা কী পাঠাব। এটা দিলে component-এ `sendOtp({ ... })` লিখতে গেলে editor বলে দেয় কী কী field লাগবে, আর `res.success`, `res.data` লিখলে suggestion আসে। (Teacher-এর comment-ও এটাই বলছে।)
- **Hook-এর নাম নিজে থেকে তৈরি হয়:** endpoint-এর নাম `login` + ধরন `mutation` → `useLoginMutation`। `userInfo` + `query` → `useUserInfoQuery`। নাম বানানোর নিয়ম: `use` + নাম (প্রথম অক্ষর বড়) + `Query`/`Mutation`।
- **`export const authApi`:** Navbar-এ `authApi.util.resetApiState()` লাগবে, তাই `authApi`-ও export করা হয়েছে।

> ⚠️ সংশোধন: নোটে এই ফাইলের path কয়েক জায়গায় আলাদা লেখা ছিল: `src/features/auth/auth.api.ts` (মাঝে `redux` নেই) আর `redux/features/auth/auth.type.ts` (শিরোনামে `.type`, কিন্তু কোড `auth.api`-র)। সঠিক path একটাই: **`src/redux/features/auth/auth.api.ts`**।

> ➕ সংযোজন — login-এ `invalidatesTags: ["USER"]`: login সফল হলে "user-এর data বদলে গেছে" বলে দেওয়া হলো। তাহলে আগে cache-এ "logged out" অবস্থা থাকলেও Navbar login-এর পরে নিশ্চিতভাবে নতুন করে `/user/me` আনবে। (Tag কী, ২১ নম্বর section-এ।)

### Mutation কী? Query আর Mutation-এর পার্থক্য

RTK Query-তে endpoint দুই রকম:

| | **Query** (`builder.query`) | **Mutation** (`builder.mutation`) |
|---|---|---|
| কাজ | server থেকে data **পড়া** | server-এর data **বদলানো** (তৈরি, update, delete) |
| HTTP method | সাধারণত `GET` | `POST`, `PUT`, `PATCH`, `DELETE` |
| কখন চলে | component render হলেই **নিজে থেকে** | আমরা function **ডাকলে তবেই** (যেমন submit চাপলে) |
| Cache | রাখে, একই data আবার লাগলে request পাঠায় না | রাখে না |
| Hook ফেরত দেয় | **object**: `{ data, isLoading, error, … }` | **array (tuple)**: `[triggerFunction, { isLoading, … }]` |
| আমাদের উদাহরণ | `userInfo` (`/user/me`) | `register`, `login`, `sendOtp`, `verifyOtp`, `logout` |

"Mutation" শব্দের মানে পরিবর্তন। Login-এ server-এ নতুন session/token তৈরি হয়, register-এ নতুন user তৈরি হয়, তাই এগুলো mutation। Login-এ আসলে "পড়ছি" মনে হলেও server-এ অবস্থা বদলাচ্ছে, আর আমরা চাই এটা submit চাপলে তবেই চলুক, page খুললেই না।

### Hook কেন একটা object, একটা array (আপনার প্রশ্ন)

```ts
const { data, isLoading } = useUserInfoQuery(undefined);   // query → object
const [logout, { isLoading }] = useLogoutMutation();        // mutation → array
```

- **Query** নিজে নিজেই চলে যায়, তাই আমাদের হাতে শুধু **ফলাফল** দরকার। ফলাফল একটা object।
- **Mutation** নিজে চলে না, আমাদের ডাকতে হয়। তাই দুটো জিনিস দরকার: **১. ডাকার function**, **২. ফলাফল**। এই দুটো একটা array-তে আসে, ঠিক `useState`-এর মতো:

```ts
const [count, setCount] = useState(0);              // [মান, বদলানোর function]
const [login, { isLoading }] = useLoginMutation();  // [ডাকার function, ফলাফল]
```

Array-তে আসায় সুবিধা: নিজের পছন্দের নাম দেওয়া যায়, যেমন `const [registerUser] = useRegisterMutation()`।

### `.unwrap()` কী

```ts
const res = await login(userInfo).unwrap();
```

`login(...)` ডাকলে সে **কখনো error throw করে না**, সবসময় একটা `{ data }` বা `{ error }` ফেরত দেয়। তাই `try/catch`-এর `catch` কখনো চলত না। `.unwrap()` এটাকে "খুলে" দেয়:
- সফল হলে → সরাসরি `data` (backend-এর response) দেয়
- ব্যর্থ হলে → error **throw** করে, ফলে `catch`-এ চলে যায়

তাই `try/catch` ব্যবহার করলে সবসময় `.unwrap()` লাগবে।

---

## ১২. Sonner (toast) setup

```sh
bunx --bun shadcn@latest add sonner
```

`main.tsx`-এ `<Toaster />` বসানো হয়েছে (৩ নম্বর section)। ব্যবহার:

```ts
import { toast } from "sonner";

toast.success("Logged in successfully");
toast.error("Invalid OTP");

// loading → পরে একই toast-কে success/error-এ বদলে দেওয়া
const toastId = toast.loading("Sending OTP...");
toast.success("OTP sent", { id: toastId }); // একই id → নতুন toast না, আগেরটাই বদলায়
```

> ⚠️ সংশোধন (dark mode-এর জন্য জরুরি): shadcn-এর `components/ui/sonner.tsx` ফাইলে theme আনা হয় `next-themes` package থেকে:
> ```tsx
> import { useTheme } from "next-themes";
> ```
> কিন্তু আমরা `next-themes` ব্যবহার করছি না, আমাদের নিজের `ThemeProvider` আছে (`dark-theme-toggle.md`)। ফলে toast সবসময় computer-এর theme মানবে, আমাদের ModeToggle-এ বদলালে বদলাবে না। ঠিক করতে ঐ লাইনটা বদলে দিন:
> ```tsx
> import { useTheme } from "@/providers/theme.provider";
> ```
> তারপর `next-themes` আর লাগবে না: `bun remove next-themes`।

---

## ১৩. Register form-এ API যোগ করা

১২ নম্বর নোটের `RegisterForm`-এর JSX পুরোটাই একই থাকছে। বদলাচ্ছে শুধু import, component-এর শুরু আর submit button:

```tsx
// src/components/modules/Authentication/RegisterForm.tsx  (শুধু বদলানো অংশ)
import { useRegisterMutation } from "@/redux/features/auth/auth.api";
import type { IRegister } from "@/types";
import { getErrorMessage } from "@/utils/getErrorMessage";
import { useNavigate } from "react-router";
import { toast } from "sonner";
// ...আগের import-গুলো (Button, Field, PasswordInput, auth.schema …) একই

export default function RegisterForm() {
  const [registerUser, { isLoading }] = useRegisterMutation();
  const navigate = useNavigate();

  const form = useForm<RegisterFormValues>({
    // ...আগের মতোই (resolver + সব defaultValues)
  });

  async function onSubmit(data: RegisterFormValues) {
    // confirmPassword backend-এ যায় না; খালি optional field-ও পাঠাব না
    const userInfo: IRegister = {
      name: data.name,
      email: data.email,
      password: data.password,
      ...(data.phone ? { phone: data.phone } : {}),
      ...(data.address ? { address: data.address } : {}),
    };

    try {
      await registerUser(userInfo).unwrap();
      toast.success("Account created! Please verify your email.");
      navigate("/verify", { state: data.email }); // email সাথে নিয়ে verify page-এ
    } catch (err) {
      toast.error(getErrorMessage(err, "Registration failed. Please try again."));
    }
  }

  // ...JSX আগের মতো, শুধু submit button:
  // <Button type="submit" className="w-full" disabled={isLoading}>
  //   {isLoading ? "Creating account..." : "Create account"}
  // </Button>
}
```

**`...(data.phone ? { phone: data.phone } : {})` কী করছে:** phone থাকলে `{ phone: "017…" }` object-টা ছড়িয়ে (spread) দেয়, ফলে `userInfo`-তে `phone` যোগ হয়। না থাকলে খালি `{}` ছড়ায়, কিছুই যোগ হয় না। ১২ নম্বর নোটের 💡-তে যে সমস্যার কথা ছিল (খালি `""` backend-এ যাওয়া), এটা তার সমাধান।

> ⚠️ সংশোধন (Register):
> - **`navigate("/verify")`-এ email যাচ্ছিল না (সবচেয়ে বড় bug):** Verify page `location.state` থেকে email পড়ে, আর email না পেলে Home-এ ফেরত পাঠায়। কিন্তু register-এর পরে `state` ছাড়াই navigate হচ্ছিল, ফলে register করামাত্র user Home-এ চলে যেত, কখনো verify করতে পারত না। এখন `navigate("/verify", { state: data.email })`।
> - **`register` থেকে `registerUser`:** React Hook Form-এও `register` নামে একটা function আছে। একই নাম দুই অর্থে থাকলে পড়তে গিয়ে গুলিয়ে যায়, আর পরে কেউ `const { register } = useForm()` লিখলে নাম সংঘর্ষও হবে।
> - **`let error` থেকে `const`, আর `console.log(result)`/`console.error` সরানো:** মান বদলানো হয় না তাই `const`। আর error-এর লেখা `getErrorMessage`-এ চলে গেছে।
> - **`disabled={isLoading}`:** request চলার সময় button বন্ধ থাকে, দুইবার চাপলে দুইবার register হয় না। (`form.formState.isSubmitting`-ও কাজ করত; RTK Query-র `isLoading` সরাসরি API-র অবস্থা বলে।)
> - **দেওয়া কোড পুরনো version-এর ছিল:** আবার hard-coded রং (`text-slate-900`, `bg-indigo-600`), `<a href>`, চোখের button-এর জন্য দুটো `useState`, `@/assets/icons/Logo` আর page-এর কাঠামো form-এর ভেতরে চলে এসেছিল। এগুলো ১১ ও ১২ নম্বর নোটে ঠিক করা হয়েছিল, সেই version-ই রাখুন। Schema-ও `auth.schema.ts` থেকে আসবে, form ফাইলে না।

---

## ১৪. Login form-এ API যোগ করা

```tsx
// src/components/modules/Authentication/LoginForm.tsx  (শুধু বদলানো অংশ)
import config from "@/config";
import { useLoginMutation } from "@/redux/features/auth/auth.api";
import type { TApiError } from "@/types";
import { getErrorMessage } from "@/utils/getErrorMessage";
import { useNavigate } from "react-router";
import { toast } from "sonner";
// ...আগের import-গুলো একই

export default function LoginForm() {
  const [login, { isLoading }] = useLoginMutation();
  const navigate = useNavigate();

  const form = useForm<LoginFormValues>({
    // ...আগের মতোই
  });

  async function onSubmit(data: LoginFormValues) {
    try {
      await login(data).unwrap();
      toast.success("Logged in successfully");
      navigate("/");
    } catch (err) {
      const error = err as TApiError;

      // Verify না করা account → OTP page-এ পাঠাও
      // (backend-এর আসল status/message দেখে মিলিয়ে নিন)
      if (error.status === 401 && error.data?.message?.toLowerCase().includes("verif")) {
        toast.error("Your account is not verified");
        navigate("/verify", { state: data.email });
        return;
      }

      toast.error(getErrorMessage(err, "Login failed. Please try again."));
    }
  }

  return (
    <>
      {/* ...heading আগের মতো */}

      <Button
        variant="outline"
        className="mt-6 w-full gap-2"
        type="button"
        onClick={() => {
          window.location.href = `${config.baseUrl}/auth/google`;
        }}
      >
        <GoogleIcon />
        Continue with Google
      </Button>

      {/* ...form আগের মতো, শুধু submit button: */}
      {/* <Button type="submit" className="w-full" disabled={isLoading}>
            {isLoading ? "Logging in..." : "Log in"}
          </Button> */}
    </>
  );
}
```

> ⚠️ সংশোধন (Login):
> - **`toast.message` থেকে `toast.success`:** `toast.message` সাধারণ (রংহীন) toast। সফল হওয়ার জন্য `success` (সবুজ টিক)।
> - **`if (res.success)` বাদ:** `.unwrap()` শুধু 2xx হলেই এই লাইনে আসতে দেয়, ব্যর্থ হলে সরাসরি `catch`-এ যায়। তাই আলাদা করে `success` দেখা অপ্রয়োজনীয়। রেখে দিলেও ভুল না।
> - **`userInfo` object বাদ, সরাসরি `login(data)`:** form-এর `data`-তে ঠিক `{ email, password }`-ই আছে, আবার একই object বানানোর দরকার নেই।
> - **`/signup` থেকে `/register`**, `<a href>` থেকে `<Link>`, hard-coded রং থেকে token: ১২ নম্বর নোটের version রাখুন।
> - **Google button-এ `window.open` থেকে `window.location.href`** (১৯ নম্বর section-এ কারণ)।

### Error ধরা: message দেখে চেনা ভালো উপায় না কেন

Teacher বললেন, password ভুল নাকি account verify করা নেই, সেটা আমরা `message` পড়ে চিনছি (`includes("verif")`), আর এটা ভালো উপায় না। কারণ backend developer message-এর একটা শব্দ বদলালেই ("verified" থেকে "confirmed") frontend-এর এই check চুপচাপ ভেঙে যাবে।

> ⚠️ সংশোধন: নোটে লেখা ছিল "Vercel বা বড় company ৫-৬ অঙ্কের status code দেয়"। HTTP status code সবসময় **৩ অঙ্কের** (100–599), ৫-৬ অঙ্কের HTTP status পাঠানো যায় না। Teacher সম্ভবত যেটা বোঝাতে চেয়েছেন: বড় company-রা status-এর **পাশাপাশি** response body-তে নিজেদের একটা **error code** পাঠায়, যেমন:
> ```json
> { "success": false, "message": "Please verify your email first", "code": "EMAIL_NOT_VERIFIED" }
> ```
> (Vercel-এর error-এ `"code": "NOT_FOUND"`, Stripe-এ `"code": "card_declined"` এরকম থাকে।) Frontend তখন message না, `code` দেখে সিদ্ধান্ত নেয়। Message বদলালেও কিছু ভাঙে না। নিজের backend-এ সুযোগ থাকলে এভাবে করবেন।

> ➕ সংযোজন — নিরাপত্তা: Login-এ "email পাওয়া যায়নি" আর "password ভুল" দুটোর জন্য **একই** message দেওয়া উচিত, যেমন "Invalid email or password"। আলাদা message দিলে খারাপ উদ্দেশ্যের কেউ একের পর এক email দিয়ে দেখে বুঝে ফেলতে পারে কোন email-এ account আছে (একে বলে *user enumeration*)। Teacher "malicious user যাতে track করতে না পারে" বলতে এটাই বুঝিয়েছেন।

---

## ১৫. Verify page (OTP)

### Flow

```
Register ──→ /verify (email সাথে) ──→ OTP দিয়ে verify ──→ /login ──→ Home
                ▲
Login (verify করা নেই) ─┘
```

যতক্ষণ email verify না হচ্ছে, ততক্ষণ login করা যাবে না।

### Route

Verify page-এও Navbar/Footer নেই, তাই login/register-এর মতো top level-এ:

```tsx
// src/routes/index.tsx
import Verify from "@/pages/Verify";

export const router = createBrowserRouter([
  { path: "/", Component: App, children: [ /* ... */ ] },
  { path: "/login", Component: Login },
  { path: "/register", Component: Register },
  { path: "/verify", Component: Verify },
  { path: "*", Component: NotFound },
]);
```

> ⚠️ সংশোধন: নোটে ফাইলের নাম `verify.tsx`, কিন্তু component-এর নাম `Verify`। আমাদের নিয়ম: ফাইলের নাম = component-এর নাম → **`src/pages/Verify.tsx`**। (Linux server-এ build করলে `verify` আর `Verify` আলাদা ফাইল ধরা হয়, তাই import-এর সাথে নাম না মিললে deploy-এ ভাঙতে পারে।)

### Page-এ email পৌঁছায় কীভাবে: `navigate` + `state`

```tsx
// পাঠানো (LoginForm / RegisterForm)
navigate("/verify", { state: data.email });

// নেওয়া (Verify)
const location = useLocation();
const email = location.state as string | undefined;
```

- `navigate`-এর দ্বিতীয় parameter-এ `state` দিলে সেটা URL-এ দেখা যায় না, চুপচাপ পরের page-এ চলে যায়।
- পরের page-এ `useLocation()` দিয়ে `location.state` পড়ে পাওয়া যায়।
- কেউ সরাসরি address bar-এ `/verify` লিখে এলে `state` থাকে না → `undefined`। তখন তাকে Home-এ ফেরত পাঠাব।

**কেন URL-এ (`/verify?email=…`) না:** URL browser history, server log, screenshot সব জায়গায় থেকে যায়। আর URL-এ যেকোনো email লিখে যে কেউ page খুলতে পারত। `state`-এ রাখলে শুধু আমাদের app-এর ভেতর থেকে আসলেই email পাওয়া যায়।

> নোট: React Router `state` রাখে browser-এর history-তে, তাই page **reload** দিলেও email থেকে যায়। কিন্তু নতুন tab-এ link খুললে থাকে না।

### `useEffect` — ভালো করে বোঝা

#### useEffect কী

React component-এর মূল কাজ: data দেখে UI **বানানো** (render)। Render-এর সময় বাইরের কোনো কাজ করা যায় না, যেমন:
- অন্য page-এ পাঠানো (`navigate`)
- API call
- timer (`setTimeout`, `setInterval`)
- browser-এর event শোনা

এগুলোকে বলে **side effect** (পার্শ্বকাজ)। `useEffect` বলে: "**আগে UI screen-এ দেখাও, তারপর এই কাজটা করো।**"

```tsx
useEffect(() => {
  // ১. effect: render শেষ হওয়ার পর যা করতে হবে

  return () => {
    // ২. cleanup (ঐচ্ছিক): পরের বার effect চলার আগে বা component সরে গেলে আগের কাজ গুছিয়ে ফেলা
  };
}, [dep1, dep2]); // ৩. dependency array: কখন আবার চালাবে
```

#### Dependency array — কখন চলে

| লেখা | কখন চলে |
|---|---|
| `useEffect(fn)` (array নেই) | **প্রতিটা** render-এর পরে (সাধারণত ভুল, বিরল ব্যবহার) |
| `useEffect(fn, [])` (খালি array) | শুধু **প্রথমবার** page/component আসার পরে |
| `useEffect(fn, [email])` | প্রথমবার, আর তারপর **যতবার `email` বদলাবে** |

React প্রতিটা render-এর পরে array-র মানগুলো আগের বারের সাথে মিলিয়ে দেখে। কোনোটা বদলালে effect আবার চলে, না বদলালে চলে না।

**নিয়ম:** effect-এর ভেতরে বাইরের যে variable/function ব্যবহার করছেন (`email`, `navigate`), সেগুলো array-তে দিতে হবে। নাহলে effect পুরনো মান নিয়ে কাজ করতে পারে। (ESLint-এর `react-hooks/exhaustive-deps` rule এটা মনে করিয়ে দেয়।)

#### Cleanup কেন লাগে

ধরুন effect-এ একটা timer চালু করলেন। User অন্য page-এ চলে গেল, কিন্তু timer চলতেই থাকল, এমন একটা component-এর state বদলানোর চেষ্টা করল যেটা আর screen-এ নেই। একে বলে **memory leak**। Cleanup function-এ `clearTimeout`/`clearInterval` দিয়ে timer বন্ধ করলে এটা হয় না।

#### প্রথম effect: email না থাকলে Home-এ পাঠাও

```tsx
useEffect(() => {
  if (!email) navigate("/", { replace: true });
}, [email, navigate]);
```

**ধাপে ধাপে:**
1. Page render হলো (email `undefined`)।
2. Render শেষে effect চলল → `!email` সত্য → Home-এ পাঠাল।
3. email থাকলে কিছুই হয় না।

**`navigate` সরাসরি component-এর body-তে কেন না:**
```tsx
// ❌ ভুল
export default function Verify() {
  if (!email) navigate("/");   // render চলাকালীন navigate
  ...
}
```
React-এর নিয়ম: render-এর সময় শুধু UI হিসাব হবে, অন্য কিছু বদলাবে না। Render-এর মাঝখানে অন্য page-এ যাওয়ার চেষ্টা করলে React Router সতর্ক করে: *"You should call navigate() in a React.useEffect(), not when your component is first rendered."* তাই `useEffect`-এ।

**`{ replace: true }` কেন:** না দিলে history-তে `/verify` থেকে যায়। User Home-এ এসে Back চাপলে আবার `/verify`-তে যায়, সেখান থেকে আবার Home… Back বাটন আটকে যায়। `replace` দিলে `/verify`-এর জায়গায় `/` বসে।

> ➕ সংযোজন: React Router-এ একই কাজ effect ছাড়াও করা যায়, render-এ `<Navigate>` component ফেরত দিয়ে:
> ```tsx
> if (!email) return <Navigate to="/" replace />;
> ```
> এটা আরও সরাসরি, এক মুহূর্তের জন্য খালি card-ও দেখায় না। Teacher `useEffect` দিয়ে দেখিয়েছেন, তাই নিচের কোডে সেটাই রাখা হলো, সাথে hook-গুলোর পরে `if (!email) return null;` যোগ করা হয়েছে যাতে খালি card না দেখায়।

#### দ্বিতীয় effect: page খুললেই একবার OTP পাঠাও

```tsx
const hasSentRef = useRef(false);

useEffect(() => {
  if (!email || hasSentRef.current) return;
  hasSentRef.current = true;
  handleSendOtp();
  // eslint-disable-next-line react-hooks/exhaustive-deps
}, [email]);
```

**সমস্যা:** development-এ `<StrictMode>` ইচ্ছা করে প্রতিটা effect **দুইবার** চালায় (mount → cleanup → আবার mount)। উদ্দেশ্য: cleanup ঠিকমতো লেখা আছে কিনা ধরিয়ে দেওয়া। কিন্তু এর ফলে OTP **দুইবার** যেত, দুটো email আসত, আর প্রথম OTP বাতিল হয়ে যেত।

**সমাধান `useRef`:** `useRef` এমন একটা বাক্স যেটার মান বদলালে re-render হয় না, আর StrictMode-এর দ্বিতীয়বার চালানোতেও মান মুছে যায় না।
1. প্রথমবার: `hasSentRef.current` = `false` → `true` করে দিয়ে OTP পাঠায়।
2. দ্বিতীয়বার (StrictMode): `hasSentRef.current` = `true` → `return`, আর পাঠায় না।

**`eslint-disable` লাইন কেন:** effect-এর ভেতরে `handleSendOtp` ব্যবহার হচ্ছে, তাই ESLint বলবে array-তে দাও। কিন্তু `handleSendOtp` প্রতিটা render-এ নতুন করে তৈরি হয়, array-তে দিলে প্রতি render-এ effect আবার চলত। Ref-এর guard থাকায় আসলে দ্বিতীয়বার পাঠাত না, তবু অকারণ চলা এড়াতে এখানে rule বন্ধ রাখা হয়েছে। এটা জেনে-বুঝে করা ব্যতিক্রম, সব জায়গায় rule বন্ধ করবেন না।

> নোট: Production build-এ StrictMode effect দুইবার চালায় না। তবু guard রাখা ভালো, কারণ development-এ ঠিক না চললে কাজ করা কঠিন।

> নোট: Page **reload** দিলে ref নতুন করে `false` থেকে শুরু হয়, তাই আবার একটা OTP যাবে। এটা স্বাভাবিক আচরণ ধরে নেওয়া যায়।

#### তৃতীয় effect: countdown timer

```tsx
const [timer, setTimer] = useState(0);

useEffect(() => {
  if (timer <= 0) return;
  const id = setTimeout(() => setTimer((t) => t - 1), 1000);
  return () => clearTimeout(id);
}, [timer]);
```

OTP পাঠানো সফল হলে `setTimer(60)`। তারপর একটা শৃঙ্খল চলে:

```
timer = 60 → effect চলে → ১ সেকেন্ড পরে setTimer(59)
timer = 59 → (timer বদলেছে) effect আবার চলে → ১ সেকেন্ড পরে setTimer(58)
...
timer = 0  → effect চলে → `timer <= 0` → return, শৃঙ্খল থেমে যায়
```

- **`[timer]`:** প্রতিবার timer বদলালে effect আবার চলে, তাই প্রতি সেকেন্ডে একটা করে নতুন `setTimeout`।
- **Cleanup `clearTimeout`:** ৩০ সেকেন্ডে user page ছেড়ে গেলে বাকি timer বন্ধ হয়ে যায়।
- **`setTimer((t) => t - 1)` (function দিয়ে):** সবসময় সর্বশেষ মান থেকে ১ কমায়। `setTimer(timer - 1)` লিখলে পুরনো মান ধরে রাখার ঝুঁকি থাকে।

**Teacher-এর `setInterval` version-এর সাথে তুলনা:**

| | Teacher (`setInterval`) | আপনার (`setTimeout` + `[timer]`) |
|---|---|---|
| কীভাবে | একটা interval প্রতি সেকেন্ডে চলতেই থাকে | প্রতি সেকেন্ডে একটা করে নতুন timeout |
| 0-তে পৌঁছালে | interval চলতেই থাকে (প্রতি সেকেন্ডে "Tick" log), শুধু মান 0-তে আটকে থাকে | নিজেই থেমে যায় |
| শুরু করা | আলাদা `confirmed` state লাগে | `setTimer(60)` দিলেই শুরু |

দুটোই ঠিক, তবে আপনারটা 0-তে পৌঁছে নিজেই থেমে যায়, তাই বেশি পরিচ্ছন্ন।

### Teacher-এর দুই ধাপ বনাম আপনার এক ধাপ

| | Teacher | আপনার |
|---|---|---|
| ধাপ | ১. "Confirm" চাপলে OTP যায় → ২. OTP card আসে | Page খুললেই OTP যায়, সরাসরি OTP card |
| Verify সফল হলে | শুধু toast, page-এই থাকে | toast + `/login`-এ পাঠায় |
| Error হলে | শুধু `console.log`, user কিছু জানে না, loading toast ঘুরতেই থাকে | error toast (loading toast-টাই বদলে যায়), OTP box খালি হয় |
| Resend | timer চলাকালীন disabled button + সংখ্যা | timer চলাকালীন "Resend in 42s" লেখা, শেষে button |

Teacher-এর দুই ধাপের সুবিধা: user নিজে "Confirm" না চাপা পর্যন্ত কোনো email যায় না (ভুল করে page খুললে অকারণ OTP যায় না)। আপনার এক ধাপের সুবিধা: register-এর পরে user-কে একটা কম click করতে হয়। দুটোই চলে।

> ⚠️ সংশোধন (Teacher-এর কোডে, শেখার জন্য): `catch`-এ শুধু `console.log(err)` ছিল, ফলে error হলে `toast.loading` চিরকাল ঘুরতে থাকত। Loading toast খুললে সবসময় success **বা** error দিয়ে সেটা বন্ধ করতে হয় (`{ id: toastId }`)। আর `"Resent OPT"` বানান ভুল ছিল → `"Resend OTP"`।

### চূড়ান্ত কোড

```tsx
// src/pages/Verify.tsx
import { Button } from "@/components/ui/button";
import { Card, CardContent, CardDescription, CardFooter, CardHeader, CardTitle } from "@/components/ui/card";
import { Field, FieldDescription, FieldError, FieldLabel } from "@/components/ui/field";
import { InputOTP, InputOTPGroup, InputOTPSlot } from "@/components/ui/input-otp";
import { useSendOtpMutation, useVerifyOtpMutation } from "@/redux/features/auth/auth.api";
import { getErrorMessage } from "@/utils/getErrorMessage";
import { zodResolver } from "@hookform/resolvers/zod";
import { REGEXP_ONLY_DIGITS } from "input-otp";
import { Dot } from "lucide-react";
import { useEffect, useRef, useState } from "react";
import { Controller, useForm } from "react-hook-form";
import { useLocation, useNavigate } from "react-router";
import { toast } from "sonner";
import * as z from "zod";

const RESEND_TIME = 60; // সেকেন্ড

const FormSchema = z.object({
  pin: z.string().length(6, { error: "Your one-time password must be 6 digits." }),
});

type FormValues = z.infer<typeof FormSchema>;

export default function Verify() {
  const location = useLocation();
  const navigate = useNavigate();
  const email = location.state as string | undefined;

  const [sendOtp, { isLoading: isSending }] = useSendOtpMutation();
  const [verifyOtp, { isLoading: isVerifying }] = useVerifyOtpMutation();
  const [timer, setTimer] = useState(0);

  // StrictMode-এ effect দুবার চলে, তাই OTP যেন দুবার না যায়
  const hasSentRef = useRef(false);

  const form = useForm<FormValues>({
    resolver: zodResolver(FormSchema),
    defaultValues: { pin: "" },
  });

  const handleSendOtp = async () => {
    if (!email) return;
    const toastId = toast.loading("Sending OTP...");

    try {
      await sendOtp({ email }).unwrap();
      toast.success("OTP sent to your email", { id: toastId });
      setTimer(RESEND_TIME);
    } catch (err) {
      toast.error(getErrorMessage(err, "Failed to send OTP"), { id: toastId });
    }
  };

  const onSubmit = async (data: FormValues) => {
    if (!email) return;
    const toastId = toast.loading("Verifying OTP...");

    try {
      await verifyOtp({ email, otp: data.pin }).unwrap();
      toast.success("Email verified! Please log in.", { id: toastId });
      navigate("/login", { replace: true });
    } catch (err) {
      toast.error(getErrorMessage(err, "Invalid OTP"), { id: toastId });
      form.reset({ pin: "" });
    }
  };

  // ১. email না থাকলে Home-এ পাঠাও
  useEffect(() => {
    if (!email) navigate("/", { replace: true });
  }, [email, navigate]);

  // ২. Page খুললেই একবার OTP পাঠাও
  useEffect(() => {
    if (!email || hasSentRef.current) return;
    hasSentRef.current = true;
    handleSendOtp();
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [email]);

  // ৩. Countdown: 0 হলে নিজেই থেমে যায়
  useEffect(() => {
    if (timer <= 0) return;
    const id = setTimeout(() => setTimer((t) => t - 1), 1000);
    return () => clearTimeout(id);
  }, [timer]);

  // Redirect হওয়ার আগের মুহূর্তে খালি card না দেখাতে (সব hook-এর পরে)
  if (!email) return null;

  return (
    <div className="grid min-h-dvh place-content-center bg-background px-4">
      <Card className="w-full max-w-sm">
        <CardHeader>
          <CardTitle className="text-xl">Verify your email address</CardTitle>
          <CardDescription>
            Please enter the 6-digit code we sent to <br />
            <span className="font-medium text-foreground">{email}</span>
          </CardDescription>
        </CardHeader>

        <CardContent>
          <form id="otp-form" onSubmit={form.handleSubmit(onSubmit)} className="space-y-6" noValidate>
            <Controller
              control={form.control}
              name="pin"
              render={({ field, fieldState }) => (
                <Field data-invalid={fieldState.invalid}>
                  <FieldLabel htmlFor="otp-pin">One-Time Password</FieldLabel>
                  <InputOTP
                    id="otp-pin"
                    maxLength={6}
                    pattern={REGEXP_ONLY_DIGITS}
                    {...field}
                    aria-invalid={fieldState.invalid}
                  >
                    <InputOTPGroup>
                      <InputOTPSlot index={0} />
                      <InputOTPSlot index={1} />
                      <InputOTPSlot index={2} />
                    </InputOTPGroup>
                    <Dot className="text-muted-foreground" />
                    <InputOTPGroup>
                      <InputOTPSlot index={3} />
                      <InputOTPSlot index={4} />
                      <InputOTPSlot index={5} />
                    </InputOTPGroup>
                  </InputOTP>

                  <FieldDescription>
                    Didn&apos;t get the code?{" "}
                    {timer > 0 ? (
                      <span className="text-muted-foreground">Resend in {timer}s</span>
                    ) : (
                      <Button
                        type="button"
                        variant="link"
                        onClick={handleSendOtp}
                        disabled={isSending}
                        className="h-auto p-0"
                      >
                        Resend OTP
                      </Button>
                    )}
                  </FieldDescription>

                  {fieldState.invalid && <FieldError errors={[fieldState.error]} />}
                </Field>
              )}
            />
          </form>
        </CardContent>

        <CardFooter>
          <Button form="otp-form" type="submit" className="w-full" disabled={isVerifying}>
            {isVerifying ? "Verifying..." : "Verify"}
          </Button>
        </CardFooter>
      </Card>
    </div>
  );
}
```

**কিছু জিনিস যা নতুন:**
- **`form="otp-form"`:** Submit button `<form>`-এর বাইরে (`CardFooter`-এ)। HTML-এ button-এ `form="id"` দিলে সে বাইরে থেকেও ঐ id-র form submit করতে পারে।
- **`pattern={REGEXP_ONLY_DIGITS}`:** OTP box-এ শুধু সংখ্যা টাইপ করা যায়। Mobile-এ সংখ্যার keyboard-ও খোলে।
- **`form.reset({ pin: "" })`:** ভুল OTP দিলে box খালি হয়ে যায়, user নতুন করে লিখতে পারে।

> ⚠️ সংশোধন (আপনার কোডে):
> - **TypeScript error:** `email`-এর type `string | undefined`, কিন্তু `verifyOtp` চায় `IVerifyOtp` (`email: string`)। তাই `verifyOtp({ email, otp })` লাইনে TypeScript error দিত। `onSubmit`-এর শুরুতে `if (!email) return;` দিলে TypeScript বুঝে যায় নিচে email অবশ্যই string।
> - **`text-slate-900`, `text-slate-500` থেকে `text-foreground`, `text-muted-foreground`:** dark mode-এ slate রং প্রায় অদৃশ্য হয়ে যেত।
> - **`h-screen` থেকে `min-h-dvh`:** mobile browser-এ address bar-এর কারণে `h-screen` (100vh) স্ক্রিনের চেয়ে লম্বা হয়ে scroll আসে। `dvh` আসল দেখা অংশ মাপে। (`AuthLayout`-এও `min-h-dvh` ব্যবহার করা।)
> - **Zod-এ `.min(6, { message })` থেকে `.length(6, { error })`:** OTP ঠিক ৬ অঙ্কের, "কমপক্ষে ৬" না। আর Zod 4-এ `message` এর বদলে `error` (১২ নম্বর নোট)। `import z from "zod"` থেকে `import * as z from "zod"`, আগের ফাইলগুলোর সাথে মিল রেখে।
> - **form-এ `noValidate`:** আমাদের নিয়ম মেনে।
> - **`getErrorMessage` ফাইলের ভেতরে না রেখে `utils`-এ:** Register আর Login-ও একই function ব্যবহার করছে।

---

## ১৬. Token কোথায় রাখব: Authorization header বনাম Cookie

Login-এর পর backend দুটো token দেয়। প্রতিটা protected request-এ access token পাঠাতে হবে। পাঠানোর দুই উপায়:

### উপায় ১: Authorization header

```
GET /user/me
Authorization: eyJhbGciOi...
```

Postman-এ আমরা এভাবেই পাঠাতাম। কিন্তু browser-এ এভাবে পাঠাতে হলে **JavaScript-কে token জানতে হবে**, মানে token কোথাও রাখতে হবে:

| কোথায় রাখব | সমস্যা |
|---|---|
| `localStorage` | যেকোনো JavaScript পড়তে পারে। XSS attack (কোনোভাবে আমাদের page-এ অন্যের script চলে গেলে) এক লাইনে চুরি করে নেবে: `localStorage.getItem("token")` |
| `sessionStorage` | একই সমস্যা, আর tab বন্ধ করলে মুছে যায়, প্রতিবার আবার login |
| Redux/state-এ (memory) | Page reload দিলেই মুছে যায় |

> ⚠️ সংশোধন: নোটে লেখা ছিল "session storage reload দিলেও চলে যায়"। আসলে `sessionStorage` **reload-এ টিকে থাকে**, মুছে যায় **tab বন্ধ করলে**। Reload-এ মুছে যায় শুধু memory-তে (state/Redux) রাখা data। আর "cross site attack" বলতে এখানে মূলত **XSS (Cross-Site Scripting)** বোঝানো হয়েছে।

### উপায় ২: httpOnly cookie (আমরা এটাই ব্যবহার করছি)

Backend login-এর response-এ একটা header পাঠায়:

```
Set-Cookie: accessToken=eyJhb...; HttpOnly; Secure; SameSite=None; Max-Age=86400
```

Browser cookie-টা জমা রাখে, আর এরপর **ঐ backend-এ যাওয়া প্রতিটা request-এ নিজে থেকেই** পাঠিয়ে দেয়। আমাদের JavaScript-কে কিছু করতে হয় না।

**Cookie-র flag-গুলো:**

| Flag | মানে | কেন জরুরি |
|---|---|---|
| `HttpOnly` | JavaScript (`document.cookie`) দিয়ে পড়া যায় না, শুধু HTTP request-এর সাথে যায় | XSS হলেও token চুরি করা যায় না |
| `Secure` | শুধু HTTPS দিয়ে যায় | রাস্তায় কেউ পড়তে পারে না |
| `SameSite` | অন্য site থেকে আসা request-এ cookie যাবে কিনা | CSRF attack ঠেকায় (নিচে টেবিল) |
| `Max-Age` / `Expires` | কতদিন পরে মুছে যাবে | না দিলে browser বন্ধ করলেই মুছে যায় |

**সারকথা:** Header-এ পাঠাতে হলে JavaScript-কে token ধরতে হয়, আর JavaScript যা ধরতে পারে, XSS-ও তা ধরতে পারে। httpOnly cookie-তে token থাকে JavaScript-এর নাগালের বাইরে, তাই এটা তুলনামূলক বেশি নিরাপদ।

> নোট: httpOnly-ও ১০০% নিরাপদ না। XSS হলে attacker token চুরি করতে পারবে না ঠিক, কিন্তু page খোলা থাকা অবস্থায় user-এর হয়ে request পাঠাতে পারবে। তাই XSS ঠেকানো (user-এর লেখা HTML হিসেবে না বসানো ইত্যাদি) আলাদাভাবেও জরুরি। "তুলনামূলক বেশি নিরাপদ" কথাটাই ঠিক।

---

## ১৭. Frontend থেকে login করলে cookie set হচ্ছিল না (***খুব গুরুত্বপূর্ণ***)

### সমস্যা

- Postman-এ login → response-এ token আসে, Postman-এর cookie-তে দুটো token বসে যায় ✅
- Browser থেকে login → response আসে, কিন্তু browser-এর cookie-তে token নেই ❌

### Postman-এ চলে, browser-এ কেন না

**Postman browser না।** Browser-এর নিরাপত্তার নিয়মগুলো (CORS, SameSite, Secure) Postman মানে না। Browser মানে, কারণ browser-এ একই সাথে অনেক website খোলা থাকে, আর একটা site যেন অন্যটার cookie নিয়ে খেলতে না পারে। তাই Postman-এ চললেই browser-এ চলবে, এমন কোনো কথা নেই।

Teacher বলেছেন cookie set হওয়ার দুই শর্ত: **backend পাঠাবে** আর **frontend নেওয়ার জন্য তৈরি থাকবে**। পুরোটা মিলিয়ে চারটা জিনিস লাগে:

### ধাপ ১ (Frontend): `withCredentials: true`

```ts
// src/lib/axios.ts
export const axiosInstance = axios.create({
  baseURL: config.baseUrl,
  withCredentials: true,
});
```

Frontend (`localhost:3000`) আর backend (`localhost:5000` বা deploy-করা URL) আলাদা **origin**। Browser by default অন্য origin-এর request-এ cookie **পাঠায়ও না, response-এর cookie জমাও রাখে না**। `withCredentials: true` বলে: "এই request-এ cookie পাঠাও, আর response-এর cookie-ও রাখো।"

সব request `axiosInstance` দিয়ে যায় বলে এখানে একবার দিলেই সব request-এ কাজ করে।

### ধাপ ২ (Backend): CORS-এ `credentials: true`

> ➕ সংযোজন (নোটে ছিল না, কিন্তু এটা ছাড়া চলবে না): frontend cookie পাঠাতে চাইলে backend-কেও অনুমতি দিতে হয়, নাহলে browser response আটকে দেয় (console-এ CORS error):
> ```ts
> // backend: app.ts
> app.use(
>   cors({
>     origin: ["http://localhost:3000"], // frontend-এর URL; deploy-এর পরে সেটাও যোগ করবেন
>     credentials: true,
>   }),
> );
> ```
> `credentials: true` থাকলে `origin: "*"` দেওয়া যায় না, নির্দিষ্ট URL দিতে হয়। আপনার backend-এ এটা সম্ভবত আগেই করা আছে (নাহলে login-এর response-ই আসত না), একবার দেখে নিন।

### ধাপ ৩ (Backend): cookie-র `sameSite` আর `secure`

`withCredentials` দেওয়ার পরে cookie আসছিল, কিন্তু reload দিলে চলে যাচ্ছিল, আর DevTools-এ cookie-র **Secure** column-এ টিক ছিল না। Backend-এ `setCookie.ts`-এ (search করুন `cookie` লিখে, অথবা login route → controller → service ধরে খুঁজলেই পাবেন):

```ts
// backend: utils/setCookie.ts — accessToken আর refreshToken দুটোতেই
res.cookie("accessToken", tokenInfo.accessToken, {
  httpOnly: true,
  secure: true,
  sameSite: "none",
  maxAge: 1000 * 60 * 60 * 24, // ১ দিন (access token-এর মেয়াদের সমান রাখুন)
});
```

**`sameSite` তিন রকম:**

| মান | অন্য site থেকে আসা request-এ cookie যায়? |
|---|---|
| `"strict"` | কখনোই না |
| `"lax"` (browser-এর default) | শুধু link-এ click করে আসলে; fetch/axios-এর request-এ না |
| `"none"` | সবসময় যায়, কিন্তু **শর্ত: `secure: true` হতেই হবে** |

**"Same site" মানে কী:** site হলো domain-এর মূল অংশ (`phovia.com`)। port বা subdomain আলাদা হলেও site একই:

| Frontend | Backend | Site |
|---|---|---|
| `localhost:3000` | `localhost:5000` | একই (port গোনা হয় না) |
| `phovia.com` | `api.phovia.com` | একই |
| `localhost:3000` | `phovia-api.vercel.app` | **আলাদা** (cross-site) |

Backend deploy করা থাকলে (Vercel ইত্যাদি) আর frontend `localhost`-এ চললে request **cross-site**। তখন `lax` cookie browser-এ আটকে যায়, তাই `sameSite: "none"` লাগে। আর `"none"`-এর সাথে `secure: true` না দিলে browser cookie-টা **পুরোপুরি বাতিল করে দেয়**। এজন্যই Secure-এ টিক না থাকার সাথে cookie হারানোর সম্পর্ক ছিল।

### ধাপ ৪ (Backend): মেয়াদ (`maxAge`)

DevTools-এ cookie-র **Expires** column-এ লেখা ছিল **"Session"**। মানে cookie-তে কোনো মেয়াদ দেওয়া নেই, browser পুরোপুরি বন্ধ করলেই মুছে যাবে, user-কে আবার login করতে হবে। `maxAge` (millisecond-এ) দিলে নির্দিষ্ট দিন পর্যন্ত থাকে। Teacher যেটা বললেন "session রাখব না, নিজেরা ঠিক করে দেব", সেটা এভাবেই। মেয়াদ token-এর নিজের মেয়াদের সমান রাখবেন (access token ছোট, refresh token বড়)।

### `NODE_ENV` নিয়ে

Backend-এর কোডে সাধারণত এমন থাকে:

```ts
secure: envVars.NODE_ENV === "production",
sameSite: envVars.NODE_ENV === "production" ? "none" : "lax",
```

মানে development-এ `lax`, production-এ `none`। Teacher বললেন, শুধু cookie ঠিক করার জন্য `.env`-এ `NODE_ENV=production` করে দেবেন না। বদলে, নিজের বানানো backend না হলে শর্ত সরিয়ে hardcode করে দিতে পারেন (`secure: true`, `sameSite: "none"`), আর অন্যের project হলে senior-কে জিজ্ঞেস করে করবেন।

> ⚠️ সংশোধন: নোটে লেখা ছিল production করলে "অনেক file delete হয়ে যেতে পারে"। `NODE_ENV=production` নিজে কোনো ফাইল delete করে না। আসল ঝুঁকি হলো: **এই একটা variable দেখে অনেক কিছুর আচরণ বদলে যায়**, যা আপনি জানেনও না। যেমন error-এর বিস্তারিত (stack) লুকিয়ে যায়, ফলে debug করা কঠিন হয়; কিছু package development-এর সুবিধা বন্ধ করে দেয়; `bun install --production` দিলে devDependencies install হয় না; কোডের অন্য কোথাও `if (production)` দিয়ে আসল database বা আসল payment চালু হয়ে যেতে পারে। তাই এক জায়গার সমস্যা মেটাতে পুরো app-কে "production" বলা ঠিক না। Teacher-এর পরামর্শটাই ঠিক।

### Cookie কোথায় দেখব

Cookie জমা থাকে **backend-এর domain-এর নামে**, frontend-এর না। কারণ cookie বসিয়েছে backend, আর পাঠানোও হবে backend-এ।

- DevTools → **Network** → `login` request → **Cookies** tab: response-এ কোন cookie এল, কোনোটা বাতিল হলে হলুদ ⚠️ দিয়ে কারণও দেখায় (সবচেয়ে নির্ভরযোগ্য জায়গা)।
- DevTools → **Application** → **Cookies**: বাম দিকের list-এ ঐ page যেসব domain-এর সাথে কথা বলেছে, সেগুলো দেখায়। Login page-এ backend-এর entry না দেখালেও, Home-এ (যেখানে `/user/me` call হয়) দেখা যায়। Teacher এজন্যই বললেন "login route-এ না দেখালেও home route-এ দেখব আছে"।

> ➕ সংযোজন: Safari by default সব cross-site (third-party) cookie আটকে দেয়, `SameSite=None; Secure` দিলেও। তাই production-এ frontend আর backend **একই site-এ** রাখা সবচেয়ে নিরাপদ, যেমন `phovia.com` আর `api.phovia.com`। তখন cookie আর cross-site থাকে না, সব browser-এ চলে।

---

## ১৮. `fetchBaseQuery` ব্যবহার করলে

Axios না চাইলে RTK Query-র নিজের `fetchBaseQuery` দিয়েও একই কাজ হয়:

```ts
// src/redux/baseApi.ts
import { createApi, fetchBaseQuery } from "@reduxjs/toolkit/query/react";
import config from "@/config";

export const baseApi = createApi({
  reducerPath: "baseApi",
  baseQuery: fetchBaseQuery({
    baseUrl: config.baseUrl,   // backend-এর base URL
    credentials: "include",    // = axios-এর withCredentials: true
  }),
  tagTypes: ["USER"],
  endpoints: () => ({}),
});
```

| | `axiosBaseQuery` | `fetchBaseQuery` |
|---|---|---|
| Cookie পাঠানো | `withCredentials: true` (axiosInstance-এ) | `credentials: "include"` |
| Endpoint-এ body | `data: userInfo` | `body: userInfo` |
| Interceptor | আছে | নেই (চাইলে baseQuery-কে মুড়িয়ে নিজে লিখতে হয়) |
| বাড়তি package | `axios` | লাগে না |

ব্যবহার ভিন্ন, কাজ একই। আমরা axios রাখছি মূলত interceptor-এর জন্য।

> ⚠️ সংশোধন: নোটে `baseUrl: "asdfaskdjfha"` hardcode ছিল (উদাহরণ হিসেবে)। আসল কোডে `config.baseUrl` ব্যবহার করবেন, যাতে URL শুধু `.env`-এ বদলালেই হয়।

---

## ১৯. Google Login

Backend-এ `/auth/google`-এ গেলে Passport Google-এর login page-এ পাঠায়, login শেষে Google backend-এর callback-এ ফেরত পাঠায়, backend cookie বসিয়ে frontend-এ redirect করে।

```
[Continue with Google] click
   → browser যায়: backend/auth/google
   → Google-এর login page
   → Google ফেরত পাঠায়: backend/auth/google/callback
   → backend: token বানায়, cookie বসায়, frontend-এ redirect
   → Home (logged in)
```

এখানে frontend-কে কোনো GET/POST request পাঠাতে হয় না, শুধু **browser-কে backend-এর ঐ URL-এ পাঠাতে হয়**:

```tsx
<Button
  variant="outline"
  className="mt-6 w-full gap-2"
  type="button"
  onClick={() => {
    window.location.href = `${config.baseUrl}/auth/google`;
  }}
>
  <GoogleIcon />
  Continue with Google
</Button>
```

`config.baseUrl` ব্যবহার করায় backend-এর URL বদলালে শুধু `.env` বদলালেই হয়। (Register form-এর "Sign up with Google" button-এও একই কোড।)

> ⚠️ সংশোধন: নোটে ছিল `window.open(url)`। `window.open` একটা **নতুন tab** খোলে। Google login শেষে backend frontend-এ redirect করবে ঐ নতুন tab-এ, আর পুরনো tab-এ login page খোলাই পড়ে থাকবে, user-এর কাছে দুটো tab। `window.location.href = url` একই tab-এ যায়, login শেষে একই tab-এ Home-এ ফেরে।
>
> আর এটা `axios`/RTK Query দিয়ে করা যায় না, কারণ Google-এর login page-এ browser-কে **সত্যিই যেতে হবে** (page বদলাতে হবে), পেছনে request পাঠালে হবে না।

> ⚠️ সংশোধন — Google Console: নোটে লেখা ছিল teacher "Authorized JavaScript origins"-এ frontend-এর URL দিয়েছেন। সেটা দেওয়া যায়, তবে এই flow-তে (server-side redirect) আসল জরুরি জিনিস হলো **"Authorized redirect URIs"**-এ backend-এর callback URL দেওয়া, যেমন `http://localhost:5000/api/v1/auth/google/callback`। এটা না মিললে Google `redirect_uri_mismatch` error দেয়। JavaScript origins লাগে মূলত তখন, যখন frontend নিজে Google-এর script দিয়ে login করায়।

> ⚠️ সংশোধন: cookie-র নাম `connect.oid` না, **`connect.sid`** (session ID)। এটা `express-session`-এর cookie। Passport-এর Google login (OAuth) চলাকালীন কিছু তথ্য session-এ রাখে, সেজন্য এটা আসে। Teacher যেমন বললেন, এটা নিয়ে চিন্তা করতে হবে না, এটা আমাদের auth token না।

---

## ২০. Protected route: `/user/me`

### Endpoint

```ts
// auth.api.ts-এর endpoints-এর ভেতরে
userInfo: builder.query<IResponse<IUser>, void>({
  query: () => ({
    url: "/user/me",
    method: "GET",
  }),
  providesTags: ["USER"],
}),
```

```tsx
// ব্যবহার (Navbar)
const { data } = useUserInfoQuery(undefined);
console.log(data?.data); // backend-এর response { success, message, data: user } → ভেতরের user
```

- **`void` (দ্বিতীয় type):** এই endpoint কোনো argument নেয় না। তাই `useUserInfoQuery()` লিখলেই হয়। Teacher-এর মতো `useUserInfoQuery(undefined)` লিখলেও একই কাজ, `undefined` মানে "কোনো argument নেই"।
- **`data?.data` কেন দুবার:** প্রথম `data` RTK Query-র (backend পুরো যা পাঠিয়েছে), দ্বিতীয় `data` backend-এর response-এর ভেতরের `data` field (`IResponse`-এর `data`)। `?.` কারণ প্রথমে (loading-এর সময়) `data` থাকে না।

> ⚠️ সংশোধন: নোটে লেখা ছিল "method অবশ্যই GET capital-এ দেব, নাহলে মাঝে মাঝে error দেয়"। Axios-এ `"get"` বা `"GET"` দুটোই কাজ করে (axios নিজে ঠিক করে নেয়)। কিন্তু **`fetch`-এ** (fetchBaseQuery) `"patch"` ছোট হাতে দিলে সত্যিই ভাঙে, কারণ fetch শুধু `GET`/`POST`/`PUT`/`DELETE` ইত্যাদি নিজে বড় হাতের করে, `PATCH` করে না, আর server `patch` চেনে না। তাই অভ্যাস হিসেবে সবসময় বড় হাতের লেখাই ঠিক নিয়ম, কারণটা জেনে রাখুন।

### সমস্যা: cookie-তে token আছে, তবু 403

Navbar-এ query বসানোর পর console-এ **403 Forbidden**, Network tab → `me` → Preview-তে `"No token received"`।

কারণ খুঁজতে Network tab → `me` → **Headers**:
- **Request Headers**-এ `Cookie: accessToken=...; refreshToken=...` আছে ✅ (withCredentials কাজ করছে)
- কিন্তু `Authorization` নামে কোনো header নেই ❌

অর্থাৎ token যাচ্ছে **cookie-তে**, কিন্তু backend খুঁজছে **Authorization header-এ**। Backend developer Postman-এর মতো header-এ token আশা করে কোড লিখেছিলেন।

### সমাধান: backend-এর `checkAuth` middleware

Backend-এ `accessToken` লিখে search করলে দেখা যায় token পড়া হচ্ছে `checkAuth.ts` middleware-এ। সব protected route এই middleware দিয়ে মোড়ানো (`router.get("/me", checkAuth(...), ...)`)। তাই শুধু এক জায়গায় বদলালেই সব protected route ঠিক হয়ে যায়:

```ts
// backend: middlewares/checkAuth.ts
// আগে:
const accessToken = req.headers.authorization;

// পরে: header থাকলে header, নাহলে cookie
const accessToken = req.headers.authorization || req.cookies.accessToken;
```

`req.cookies.accessToken`-এর `accessToken` নামটা browser-এর cookie list-এ যে নাম দেখাচ্ছে, হুবহু সেটাই হতে হবে।

এখন `/user/me` → **"User retrieved successfully"** ✅

> ➕ সংযোজন: `req.cookies` তখনই কাজ করে, যখন backend-এ **`cookie-parser`** middleware বসানো থাকে (`app.use(cookieParser())`)। না থাকলে `req.cookies` হবে `undefined`, আর `.accessToken` পড়তে গিয়ে error দেবে। আপনার backend-এ সম্ভবত আছে (refresh token-এর জন্য লাগে), একবার `app.ts`-এ দেখে নিন।

> নোট: Backend deploy করা থাকলে শুধু local কোড বদলালে হবে না। Code push → build → server restart (বা Vercel-এ নতুন deploy), তারপর frontend থেকে দেখুন। নাহলে পুরনো কোডই চলতে থাকবে, data আসবে না।

### Header-এ token দেওয়ার পরীক্ষা (শুধু বোঝার জন্য)

```ts
// ❌ শুধু পরীক্ষা, আসল কোডে করবেন না
export const axiosInstance = axios.create({
  baseURL: config.baseUrl,
  withCredentials: true,
  headers: {
    Authorization: "যেকোনো একটা token",
  },
});
```

এটা দিলে Network tab-এ `me`-র Request Headers-এ `Authorization` দেখা যাবে। মানে header দিয়েও পাঠানো যায়। কিন্তু আসল token এখানে বসাতে হলে JavaScript-কে token জানতে হবে (localStorage ইত্যাদি), যেটা আমরা ১৬ নম্বর section-এ বাদ দিয়েছি। তাই এই পরীক্ষা শেষে মুছে দিন।

### Token-এর পুরো recap

1. Header-এর বদলে **cookie** দিয়ে auth, কারণ httpOnly cookie JavaScript পড়তে পারে না, তুলনামূলক নিরাপদ।
2. **Frontend:** `axios.ts`-এ `withCredentials: true` → browser cookie নেয় আর প্রতিটা request-এ পাঠায়।
3. **Backend `setCookie.ts`:** `httpOnly: true`, `secure: true`, `sameSite: "none"` (cross-site হলে), সাথে `maxAge`।
4. **Backend CORS:** `credentials: true` আর নির্দিষ্ট `origin`।
5. **Backend `checkAuth.ts`:** token `req.headers.authorization` ছাড়াও `req.cookies.accessToken` থেকে পড়ে।

এখন frontend-এর কোনো জায়গায় token ছুঁতে হয় না। Browser নিজে cookie নিয়ে যায়, backend নিজে cookie থেকে পড়ে।

---

## ২১. Logout আর Cache: কেন reload না দিলে বোঝা যায় না

### Logout endpoint

```ts
// auth.api.ts-এর endpoints-এর ভেতরে
logout: builder.mutation<IResponse<null>, void>({
  query: () => ({
    url: "/auth/logout",
    method: "POST",
  }),
  invalidatesTags: ["USER"],
}),
```

Backend-এর `/auth/logout` cookie দুটো মুছে দেয় (Postman-এ hit করলে token চলে যায়)।

### সমস্যা ১: logout করলাম, Navbar-এ তবু "Log out" button

Reload না দেওয়া পর্যন্ত Navbar বুঝতে পারে না user logout করেছে। কারণ **cache**:
- Page খোলার সময় `useUserInfoQuery` একবার `/user/me` এনেছিল, RTK Query উত্তরটা cache-এ রেখে দিয়েছে।
- Logout হলো server-এ, কিন্তু RTK Query জানে না যে user-এর data এখন পুরনো। তাই cache থেকে পুরনো user-ই দেখাচ্ছে।

### সমাধানের চেষ্টা: Tags

RTK Query-কে বলে দিতে হবে: "logout হলে user-এর data আর বিশ্বাসযোগ্য না।" এজন্য **tag** (লেবেল):

```ts
// baseApi.ts
tagTypes: ["USER"],                 // ১. কোন কোন লেবেল থাকবে, আগে ঘোষণা

// auth.api.ts
userInfo: builder.query({ ..., providesTags: ["USER"] }),     // ২. "আমার আনা data-র গায়ে USER লেবেল"
logout:   builder.mutation({ ..., invalidatesTags: ["USER"] }), // ৩. "আমি সফল হলে USER লেবেলের data বাতিল"
```

দোকানের উদাহরণ: `providesTags` মানে তাকের জিনিসে "USER" লেখা stiker লাগানো। `invalidatesTags` মানে ঘোষণা: "USER stiker লাগানো সব জিনিস মেয়াদোত্তীর্ণ"। তখন যে component এই data ব্যবহার করছে (Navbar), RTK Query তার জন্য **নিজে থেকে আবার fetch করে** (auto refetching)।

`"USER"` নামটা আমাদের পছন্দমতো, যেকোনো নাম দেওয়া যায়, শুধু তিন জায়গায় একই হতে হবে।

### সমস্যা ২: tag দেওয়ার পরেও একই অবস্থা!

Tag দিলাম, logout করলাম, তবু reload লাগছে। কেন?

ধাপে ধাপে দেখি কী হলো:
1. Logout সফল → cookie মুছে গেছে।
2. `invalidatesTags: ["USER"]` → userInfo-র cache **stale** (বাসি) চিহ্নিত হলো → RTK Query আবার `/user/me` পাঠাল।
3. কিন্তু এখন token নেই → backend দিল **403**।
4. RTK Query-র নিয়ম: **নতুন fetch ব্যর্থ হলে পুরনো `data` মুছে ফেলে না।** শুধু `error` বসায়, `data`-তে আগের user থেকেই যায়।
5. Navbar দেখে `data?.data?.email` এখনো আছে → "Log out" button দেখাতেই থাকে।

### Redux DevTools-এ teacher যা দেখালেন

Chrome-এর Redux DevTools extension খুলে `baseApi` → `queries` → `userInfo(undefined)` দেখলে:

| সময় | `status` | `data` | `error` |
|---|---|---|---|
| Login-এর পরে | `fulfilled` | user-এর তথ্য | নেই |
| Logout-এর পরে | `rejected` | **user-এর তথ্য এখনো আছে** | 403 |

> ⚠️ সংশোধন (শব্দ): নোটে লেখা ছিল "data steal হয়ে যায়"। শব্দটা **stale** (স্টেল), মানে **বাসি/পুরনো**। Steal মানে চুরি, এখানে কেউ কিছু চুরি করছে না। Teacher বলেছেন: invalidate করলে data **stale হয়ে যায়, কিন্তু remove হয় না।**

অর্থাৎ teacher-এর কথাটা: `providesTags`/`invalidatesTags` নিজে কোনো data দেয় না বা মোছে না। এরা শুধু জানায় data fresh নাকি stale। Stale হলে RTK Query নতুন data আনার চেষ্টা করে, আর নতুন data **এলে** পুরনোটা তা দিয়ে বদলে দেয়। আমাদের ক্ষেত্রে নতুন data আসছেই না (token নেই, এসেছে error), তাই পুরনোটা বদলানোর মতো কিছুই নেই, সেটাই থেকে যাচ্ছে।

> নোট: RTK Query এটা ইচ্ছা করেই করে। ধরুন tour list দেখাচ্ছেন, হঠাৎ internet একটু গেল আর refetch ব্যর্থ হলো। তখন পুরো list মুছে খালি page দেখানোর চেয়ে পুরনো list দেখিয়ে রাখা ভালো। কিন্তু logout-এর মতো ক্ষেত্রে এই আচরণ আমাদের বিপক্ষে যায়।

### আসল সমাধান: `resetApiState`

```ts
dispatch(authApi.util.resetApiState());
```

এটা RTK Query-র **পুরো cache মুছে একদম শুরুর অবস্থায়** নিয়ে যায়: সব query-র data, সব mutation-এর অবস্থা, সব কিছু। Cache খালি হওয়ার পর Navbar-এর `useUserInfoQuery` আবার চেষ্টা করে, 403 পায়, কিন্তু এবার পুরনো data নেই → `data` = `undefined` → "Log in / Sign up" দেখায়। ✅

**এটা action, তাই `dispatch` লাগে:** `resetApiState()` নিজে কিছু করে না, শুধু একটা action (নির্দেশ) বানায়। `dispatch` সেই নির্দেশ store-এ পাঠায়, তখন store-এর reducer cache খালি করে। তাই `useAppDispatch()` দিয়ে `dispatch` নিয়ে তার ভেতরে দিতে হয়।

**"শুধু authApi reset, নাকি পুরো API"** (আপনি যেটা বোঝেননি): আগেই বলেছি `authApi` = `baseApi.injectEndpoints(...)`, আর `injectEndpoints` **একই** API ফেরত দেয়। তাই `authApi.util.resetApiState()` আর `baseApi.util.resetApiState()` **হুবহু একই কাজ করে**: পুরো `baseApi`-র cache মুছে দেয়, শুধু auth-এর না। পরে tour, booking-এর API যোগ হলে logout-এর সময় সেগুলোর cache-ও মুছে যাবে। Logout-এর জন্য এটাই আসলে ঠিক: একজন user-এর booking, profile কিছুই পরের user-এর সামনে থেকে যাওয়া উচিত না।

> ⚠️ সংশোধন: নোটে লেখা ছিল `useLogoutMutation(undefined)`। Mutation hook-এ argument দেওয়া হয় না (দিলে সেটা setting হিসেবে ধরা হয়), argument দেওয়া হয় **ডাকার সময়**: `const [logout] = useLogoutMutation();` তারপর `logout()`। আর endpoint-এ `void` type দেওয়ায় `logout(undefined)` না লিখে শুধু `logout()` লেখা যায়।

> নোট: `resetApiState` দেওয়ার পরে logout-এর `invalidatesTags` আর না দিলেও চলে, কারণ reset নিজেই সব আবার আনায়। রেখে দিলে ক্ষতি নেই, শুধু Network tab-এ একটা বাড়তি 403 দেখবেন। Teacher রেখেছেন, তাই রাখা হলো।

### `async/await` কেন জরুরি

```ts
const handleLogout = async () => {
  try {
    await logout().unwrap();   // ১. আগে logout শেষ হোক (cookie মুছুক)
    toast.success("Logged out");
  } catch (err) {
    toast.error(getErrorMessage(err, "Logout failed"));
  } finally {
    dispatch(authApi.util.resetApiState());   // ২. তারপর cache মোছো
  }
};
```

`await` ছাড়া reset আগে হয়ে যেতে পারে, আর logout request শেষ হওয়ার **আগেই** Navbar আবার `/user/me` চায়। তখন cookie এখনো আছে → user আবার চলে আসে → মনে হবে logout কাজ করেনি। `await` দিলে ক্রম নিশ্চিত: আগে cookie মোছা, তারপর cache মোছা।

**`finally`:** `try` সফল হোক বা `catch`-এ যাক, `finally` সবসময় চলে। Logout ব্যর্থ হলেও cache reset হয়ে আবার `/user/me` আনবে। Cookie তখনো থাকলে user logged in-ই দেখাবে, যা সঠিক।

> ⚠️ সংশোধন: নোটের কোডে ছিল `try { ... } finally { ... }`, কোনো `catch` নেই। `catch` না থাকলে logout ব্যর্থ হলে error **উপরে ছুড়ে দেওয়া হয়**, আর button-এর `onClick` সেটা কেউ ধরে না → console-এ *"Uncaught (in promise)"* error, user কিছুই জানে না। `catch` দিয়ে error toast দেখানো হলো।

---

## ২২. চূড়ান্ত Navbar

```tsx
// src/components/layout/Navbar.tsx
import Logo from "@/components/Logo";
import { Button, buttonVariants } from "@/components/ui/button";
import { Sheet, SheetContent, SheetHeader, SheetTitle, SheetTrigger } from "@/components/ui/sheet";
import { cn } from "@/lib/utils";
import { authApi, useLogoutMutation, useUserInfoQuery } from "@/redux/features/auth/auth.api";
import { useAppDispatch } from "@/redux/hook";
import { getErrorMessage } from "@/utils/getErrorMessage";
import { LogOut, Menu } from "lucide-react";
import { useState } from "react";
import { Link, NavLink } from "react-router";
import { toast } from "sonner";
import { ModeToggle } from "./ModeToggle";

const navLinks = [
  { label: "Home", href: "/" },
  { label: "About", href: "/about" },
  { label: "Tours", href: "/tours" },
  { label: "Divisions", href: "/divisions" },
  { label: "Short tours", href: "/tours?type=short" },
  { label: "Long tours", href: "/tours?type=long" },
];

// আসল <Button>-এ বাড়তি class হিসেবে
const primaryClass =
  "h-11 rounded-lg px-[22px] text-[14.5px] font-medium shadow-lg shadow-primary/25 hover:bg-primary-hover";
const outlineClass =
  "h-11 rounded-lg border-primary text-[14.5px] font-medium text-primary hover:bg-accent hover:text-accent-foreground";

// <Link>-কে button-এর মতো দেখাতে
const primaryBtn = cn(buttonVariants(), primaryClass);
const outlineBtn = cn(buttonVariants({ variant: "outline" }), outlineClass);

export default function Navbar() {
  const [open, setOpen] = useState(false);
  const { data, isLoading } = useUserInfoQuery(undefined);
  const [logout, { isLoading: isLoggingOut }] = useLogoutMutation();
  const dispatch = useAppDispatch();

  const isLoggedIn = Boolean(data?.data?.email);

  const handleLogout = async () => {
    try {
      await logout().unwrap();
      toast.success("Logged out");
    } catch (err) {
      toast.error(getErrorMessage(err, "Logout failed"));
    } finally {
      dispatch(authApi.util.resetApiState());
      setOpen(false);
    }
  };

  return (
    <header className="w-full bg-background">
      <div className="mx-auto flex h-21 max-w-300 items-center justify-between gap-6 px-4 sm:px-10">
        <Logo />

        {/* Desktop nav */}
        <nav aria-label="Main" className="hidden items-center gap-9 lg:flex">
          {navLinks.map((link) => (
            <NavLink
              key={link.label}
              to={link.href}
              className="relative py-1.5 text-[14.5px] font-medium text-foreground transition-colors hover:text-primary after:absolute after:-bottom-1 after:left-1/2 after:size-1.25 after:-translate-x-1/2 after:rounded-full after:bg-primary after:opacity-0 aria-[current=page]:after:opacity-100"
            >
              {link.label}
            </NavLink>
          ))}
        </nav>

        {/* Right side */}
        <div className="flex items-center gap-3.5">
          {/* প্রথমবার user-এর তথ্য আসার আগে কিছু দেখাব না, নাহলে Log in → Log out লাফ দেখা যায় */}
          {!isLoading &&
            (isLoggedIn ? (
              <Button
                variant="outline"
                onClick={handleLogout}
                disabled={isLoggingOut}
                className={cn(outlineClass, "hidden sm:inline-flex")}
              >
                <LogOut className="size-4" />
                Log out
              </Button>
            ) : (
              <>
                <Link
                  to="/login"
                  className="hidden py-1.5 text-[14.5px] font-medium text-foreground transition-colors hover:text-primary sm:inline-block"
                >
                  Log in
                </Link>
                <Link to="/register" className={cn(primaryBtn, "hidden sm:inline-flex")}>
                  Sign up
                </Link>
              </>
            ))}

          {/* Mobile menu */}
          <Sheet open={open} onOpenChange={setOpen}>
            <SheetTrigger
              aria-label="Open menu"
              className={cn(
                buttonVariants({ variant: "outline", size: "icon" }),
                "size-11 rounded-lg border-border text-foreground lg:hidden",
              )}
            >
              <Menu className="size-5.5" />
            </SheetTrigger>

            <SheetContent side="right" className="w-75 bg-background">
              <SheetHeader>
                <SheetTitle className="text-left">
                  <Logo />
                </SheetTitle>
              </SheetHeader>

              <nav aria-label="Mobile" className="flex flex-col gap-1 px-4">
                {navLinks.map((link) => (
                  <NavLink
                    key={link.label}
                    to={link.href}
                    onClick={() => setOpen(false)}
                    className="rounded-lg px-3 py-2.5 text-[15px] font-medium text-foreground transition-colors hover:bg-accent hover:text-accent-foreground aria-[current=page]:bg-accent aria-[current=page]:text-accent-foreground"
                  >
                    {link.label}
                  </NavLink>
                ))}
              </nav>

              <div className="mt-4 flex flex-col gap-3 px-4 sm:hidden">
                {!isLoading &&
                  (isLoggedIn ? (
                    <Button
                      variant="outline"
                      onClick={handleLogout}
                      disabled={isLoggingOut}
                      className={outlineClass}
                    >
                      <LogOut className="size-4" />
                      Log out
                    </Button>
                  ) : (
                    <>
                      <Link to="/login" onClick={() => setOpen(false)} className={outlineBtn}>
                        Log in
                      </Link>
                      <Link to="/register" onClick={() => setOpen(false)} className={primaryBtn}>
                        Sign up
                      </Link>
                    </>
                  ))}
              </div>
            </SheetContent>
          </Sheet>

          <ModeToggle />
        </div>
      </div>
    </header>
  );
}
```

> ⚠️ সংশোধন (Navbar):
> - **Import path:** `Logo` আছে `@/components/Logo`-এ (৮ নম্বর নোট), `@/assets/icons/Logo`-এ না (`assets`-এ শুধু ছবি/static ফাইল)। `ModeToggle` ফাইলের নাম `ModeToggle.tsx` (`ModeToggler` না, `dark-theme-toggle.md`)।
> - **Desktop-এ `/signup` আর mobile-এ `/register`:** একই app-এ দুই নাম ছিল, আর `/signup` route-ই নেই → 404। দুই জায়গাতেই `/register`।
> - **Mobile-এ সাধারণ `<button>`:** আমাদের নিয়মে আসল button-এ shadcn-এর `<Button>`।
> - **`<Button className={outlineBtn}>`:** `outlineBtn`-এ আগে থেকেই `buttonVariants(...)` আছে, আর `<Button>` নিজেও `buttonVariants()` বসায় → একই class দুইবার, default আর outline-এর রং একে অপরকে কাটাকাটি করে। এখন `<Button variant="outline" className={outlineClass}>` (শুধু বাড়তি class), আর `buttonVariants` সহ `outlineBtn` শুধু `<Link>`-এর জন্য।
> - **Sheet-এর `open` state ফিরিয়ে আনা:** ৮ নম্বর নোটে link-এ click করলে menu বন্ধ হতো, শেষ কোডে সেটা হারিয়ে গিয়েছিল। এখন link বা logout চাপলে menu বন্ধ হয়।
> - **`useLogoutMutation(undefined)` → `useLogoutMutation()`, `logout(undefined)` → `logout()`** (২১ নম্বর section)।
> - **`isLoading` check (➕):** page খোলার সাথে সাথে `/user/me`-র উত্তর আসতে একটু সময় লাগে। এর মধ্যে `data` নেই বলে "Log in" দেখাত, তারপর হঠাৎ "Log out"-এ লাফ দিত। প্রথম load শেষ না হওয়া পর্যন্ত কিছু না দেখালে এই ঝাঁকুনি থাকে না।

> নোট: `NavLink` URL-এর `?type=short` অংশ দেখে না, শুধু `/tours` মেলায়। তাই `/tours`-এ থাকলে "Tours", "Short tours", "Long tours" তিনটাই active দেখাবে। ৮ নম্বর নোটে `pathname + search` মিলিয়ে নিজে `isActive` বানিয়ে এটা সামলানো হয়েছিল। Tours page বানানোর সময় এটা আবার দেখে নেবেন।

> নোট: `SheetTrigger`-এ `asChild` ছাড়াই সরাসরি `className` দেওয়া হয়েছে। Radix-এর `SheetTrigger` নিজেই একটা `<button>` render করে, তাই এখানে ভেতরে আলাদা `<Button>` বসাচ্ছি না (button-এর ভেতরে button হবে না)। ভেতরে `<Button>` বসাতে চাইলে তখন `<SheetTrigger asChild>` লাগবে (`dark-theme-toggle.md`)।

---

## ২৩. চূড়ান্ত `auth.api.ts`

```ts
// src/redux/features/auth/auth.api.ts
import { baseApi } from "@/redux/baseApi";
import type { ILogin, ILoginResponse, IRegister, IResponse, ISendOtp, IUser, IVerifyOtp } from "@/types";

export const authApi = baseApi.injectEndpoints({
  endpoints: (builder) => ({
    register: builder.mutation<IResponse<IUser>, IRegister>({
      query: (userInfo) => ({
        url: "/user/register",
        method: "POST",
        data: userInfo,
      }),
    }),

    login: builder.mutation<IResponse<ILoginResponse>, ILogin>({
      query: (userInfo) => ({
        url: "/auth/login",
        method: "POST",
        data: userInfo,
      }),
      invalidatesTags: ["USER"],
    }),

    logout: builder.mutation<IResponse<null>, void>({
      query: () => ({
        url: "/auth/logout",
        method: "POST",
      }),
      invalidatesTags: ["USER"],
    }),

    sendOtp: builder.mutation<IResponse<null>, ISendOtp>({
      query: (userInfo) => ({
        url: "/otp/send",
        method: "POST",
        data: userInfo,
      }),
    }),

    verifyOtp: builder.mutation<IResponse<null>, IVerifyOtp>({
      query: (userInfo) => ({
        url: "/otp/verify",
        method: "POST",
        data: userInfo,
      }),
    }),

    userInfo: builder.query<IResponse<IUser>, void>({
      query: () => ({
        url: "/user/me",
        method: "GET",
      }),
      providesTags: ["USER"],
    }),
  }),
});

export const {
  useRegisterMutation,
  useLoginMutation,
  useLogoutMutation,
  useSendOtpMutation,
  useVerifyOtpMutation,
  useUserInfoQuery,
} = authApi;
```

> নোট: Teacher-এর কোডে `login`, `register`, `logout`-এ type দেওয়া ছিল না (শুধু `sendOtp`, `verifyOtp`-এ ছিল)। Type ছাড়াও চলে, কিন্তু তখন `res.data`-র ভেতরে কী আছে editor জানে না, আর ভুল field পাঠালেও ধরতে পারে না। তাই সবগুলোতে type দেওয়া হলো। `IUser`/`ILoginResponse`-এর field Postman-এ আসল response দেখে মিলিয়ে নেবেন।

---

## ২৪. Check-list

**Setup**
- [ ] `.env`-এ `VITE_BASE_URL` আছে, শেষে `/` নেই; `.env` বদলানোর পর dev server restart করেছি
- [ ] Redux DevTools-এ store-এর ভেতরে `baseApi` দেখা যায়
- [ ] Console-এ "Middleware for RTK-Query API … has not been added" warning নেই

**Register → Verify → Login**
- [ ] Register করলে success toast আসে আর `/verify`-এ email সহ যায় (Home-এ ফেরত যায় না)
- [ ] Verify page খুললে **একটাই** OTP email আসে (দুটো না)
- [ ] Countdown 60 থেকে 0 পর্যন্ত নামে, তারপর "Resend OTP" button আসে
- [ ] ভুল OTP দিলে error toast আসে আর box খালি হয়; loading toast ঘুরতে থাকে না
- [ ] সঠিক OTP দিলে `/login`-এ যায়
- [ ] Address bar-এ সরাসরি `/verify` লিখলে Home-এ চলে যায়, আর Back চাপলে আবার `/verify`-তে আটকায় না
- [ ] Verify না করা account দিয়ে login করলে `/verify`-এ পাঠায়
- [ ] Phone/Address খালি রেখে register করলে Network → Payload-এ `phone`, `address`, `confirmPassword` যায় না

**Cookie**
- [ ] Network → `login` → Cookies tab-এ `accessToken`, `refreshToken` আছে, কোনো হলুদ ⚠️ নেই
- [ ] Cookie-তে **HttpOnly** আর **Secure** টিক আছে, **Expires** "Session" না
- [ ] Reload দিলে আর browser বন্ধ করে খুললেও logged in থাকে
- [ ] Console-এ `document.cookie` লিখলে token **দেখা যায় না** (httpOnly কাজ করছে)
- [ ] Network → `me` → Request Headers-এ `Cookie` আছে, আর response 200

**Navbar ও Logout**
- [ ] Login-এর পরে Navbar-এ (reload ছাড়াই) "Log out" দেখায়
- [ ] Logout চাপলে reload ছাড়াই "Log in / Sign up" ফিরে আসে
- [ ] Logout-এর পরে Redux DevTools-এ `baseApi.queries`-এ পুরনো user-এর data নেই
- [ ] Mobile menu থেকে logout করলে menu বন্ধ হয়
- [ ] Page খোলার সময় "Log in" থেকে "Log out"-এ লাফ দেখা যায় না

**Google ও Dark mode**
- [ ] "Continue with Google" একই tab-এ Google-এ যায় আর login শেষে একই tab-এ ফেরে
- [ ] ModeToggle-এ dark করলে toast-ও dark হয় (`sonner.tsx`-এ `useTheme` ঠিক করা হয়েছে)
- [ ] Verify page dark mode-এ পড়া যায়
