# Frontend Notes — ২৩: Booking আর Payment — "Book now" থেকে SSLCommerz হয়ে ফলাফলের Page

২২ নম্বর নোটের tour details page-এর "Book now" এবার কাজ করবে। এটা এই project-এর সবচেয়ে গুরুত্বপূর্ণ flow, কারণ এখানে **আসল টাকা** জড়িত, আর তিনটা পক্ষ কাজ করে: আমাদের frontend, আমাদের backend, আর SSLCommerz (payment gateway)। এই নোটে সেই flow, কোড, আর পথে আসা দুটো backend-এর সমস্যা ও তার সমাধান।

প্রতিটা ধাপে 🎯 লক্ষ্য, 🛠️ কাজ, 💡 কেন, ✅ যাচাই। শেষে সমস্যা, উন্নতি আর চূড়ান্ত কোড।

> নোট: আপনার project-এর বর্তমান কোড পড়ে লেখা। পুরনো ধরনের কোড দিয়ে কিছু বদলানো হয়নি।

---

## ভূমিকা: পুরো flow

```
 User              Frontend                 Backend                    SSLCommerz
  │                   │                        │                           │
  │ "Book now"        │                        │                           │
  ├──────────────────►│ login নেই? → /login    │                           │
  │                   │                        │                           │
  │                   │ POST /booking          │                           │
  │                   │ { tour, guestCount } ─►│ Booking (PENDING)          │
  │                   │                        │ Payment (UNPAID) তৈরি      │
  │                   │                        │ payment শুরু ─────────────►│
  │                   │                        │◄──────── GatewayPageURL ───│
  │                   │◄──── { paymentUrl } ───│                           │
  │                   │                        │                           │
  │◄──────── window.location.href = paymentUrl (আমাদের site ছেড়ে যায়) ──►│
  │                   │                        │                           │
  │ টাকা দেয় (bKash, card…) ─────────────────────────────────────────────►│
  │                   │                        │◄──── ফলাফল (success/fail) ─│
  │                   │                        │ Payment PAID, Booking      │
  │                   │                        │ CONFIRMED (বা FAILED…)     │
  │                   │                        │ invoice email              │
  │◄──── redirect: /payment/success?transactionId=…&amount=… ─────────────│
  │                   │ PaymentResult page      │                           │
```

**তিনটা ফলাফলের page** (backend-এর `.env`-এ এই URL-গুলো লেখা থাকে):
- `/payment/success?transactionId=…&amount=…&message=…`
- `/payment/fail?…`
- `/payment/cancel?…`

### ধাপের মানচিত্র

| ধাপ | বিষয় |
|---|---|
| ১ | `booking.api.ts` আর `BOOKING` tag |
| ২ | "Book now": login যাচাই, booking তৈরি, payment-এ পাঠানো |
| ৩ | Login-এর পরে আগের page-এ ফেরা |
| ৪ | একটা component-এ তিনটা ফলাফলের page |
| ৫ | Route-এ `element` দিয়ে prop |
| ৬ | Backend সমস্যা ১: URL-এ `[object Object]` |
| ৭ | Backend সমস্যা ২: email না যাওয়ায় payment cancel |

---

## ধাপ ১ — `booking.api.ts`

🎯 **লক্ষ্য:** Booking তৈরির API।

🛠️ **কাজ:**

```ts
// src/redux/features/Booking/booking.api.ts
import { baseApi } from "@/redux/baseApi";
import type { IResponse } from "@/types";

type TCreateBookingPayload = { tour: string; guestCount: number };
type TCreateBookingResponse = { paymentUrl: string; booking: unknown };

export const bookingApi = baseApi.injectEndpoints({
  endpoints: (builder) => ({
    createBooking: builder.mutation<TCreateBookingResponse, TCreateBookingPayload>({
      query: (payload) => ({
        url: "/booking",
        method: "POST",
        data: payload,
      }),
      transformResponse: (response: IResponse<TCreateBookingResponse>) => response.data,
      invalidatesTags: ["BOOKING"],
    }),
  }),
});

export const { useCreateBookingMutation } = bookingApi;
```

```ts
// src/redux/baseApi.ts
tagTypes: ["USER", "TOUR", "DIVISION", "BOOKING"],
```

💡 **লাইনগুলোর মানে:**
- **নতুন feature, নতুন ফাইল:** booking-এর সব API (তৈরি, পরে "আমার booking", বাতিল) এক জায়গায়, module pattern (১৩ নম্বর নোট)।
- **Payload-এ শুধু `{ tour, guestCount }`:** কোন tour (id) আর কতজন। **দাম পাঠানো হয় না**: backend নিজে database থেকে tour-এর দাম নিয়ে হিসাব করে (২২ নম্বর নোটের ধাপ ৯-এর নীতি)। Frontend দাম পাঠালে কেউ browser-এ বদলে ৳১ করে দিতে পারত।
- **`booking: unknown`:** backend booking-এর পুরো তথ্যও পাঠায়, কিন্তু এখন আমরা সেটা ব্যবহার করছি না, শুধু `paymentUrl`। `unknown` মানে "কিছু একটা আছে, কিন্তু ব্যবহার করতে চাইলে আগে type ঠিক করতে হবে" (২১ নম্বর নোটের ধাপ ৪)। `any`-এর চেয়ে নিরাপদ: ভুল করে `res.booking.status` লিখলে TypeScript আটকায়।
- **`invalidatesTags: ["BOOKING"]`:** এখন কোনো query `BOOKING` provide করে না, তাই এখনই কিছু হয় না। পরে user-এর "My bookings" তালিকা (`providesTags: ["BOOKING"]`) বানালে, নতুন booking-এর পরে সেটা নিজে আপডেট হবে। আগাম প্রস্তুতি।
- **`tagTypes`-এ যোগ না করলে:** `invalidatesTags: ["BOOKING"]`-এ TypeScript error দেয়, কারণ RTK Query শুধু ঘোষিত tag নেয়।

> নোট: আপনার `tagTypes`-এ এখন `"TOUR_TYPE"` নেই, অর্থাৎ ১৭ আর ২০ নম্বর নোটের tag আলাদা করার পরামর্শ এখনো বসানো হয়নি। বসানোর সময় `["USER", "TOUR", "TOUR_TYPE", "DIVISION", "BOOKING"]`।

---

## ধাপ ২ — "Book now": login যাচাই, booking, payment-এ পাঠানো

🎯 **লক্ষ্য:** Button চাপলে সঠিক user-কে payment page-এ পাঠানো।

🛠️ **কাজ (`TourDetails.tsx`-এ):**

```tsx
import { useUserInfoQuery } from "@/redux/features/auth/auth.api";
import { useCreateBookingMutation } from "@/redux/features/Booking/booking.api";
import { useLocation, useNavigate } from "react-router";

const navigate = useNavigate();
const location = useLocation();
const { data: userData } = useUserInfoQuery(undefined);
const [createBooking, { isLoading: isBooking }] = useCreateBookingMutation();

const isLoggedIn = Boolean(userData?.data?.email);
```

```tsx
const handleBook = async () => {
  // login নেই → login page, পরে এখানেই ফিরে আসবে
  if (!isLoggedIn) {
    toast.info("Please log in to book this tour");
    navigate("/login", { state: { from: location.pathname } });
    return;
  }

  const toastId = toast.loading("Creating your booking…");
  try {
    const res = await createBooking({ tour: tour._id, guestCount: guests }).unwrap();

    if (res.paymentUrl) {
      toast.success("Redirecting to payment…", { id: toastId });
      window.location.href = res.paymentUrl;   // SSLCommerz-এ যাও
    } else {
      toast.error("Payment link not received. Please try again.", { id: toastId });
    }
  } catch (err) {
    const error = err as TApiError;
    toast.error(error.data?.message || "Booking failed. Please try again.", { id: toastId });
  }
};
```

```tsx
<Button
  type="button"
  onClick={handleBook}
  disabled={isBooking || tour.costFrom === undefined}
  className="h-13 gap-2 rounded-lg text-[15.5px] font-semibold"
>
  {isBooking ? (<><Loader2 className="size-4.5 animate-spin" /> Processing…</>) : "Book now"}
</Button>
```

### 💡 লাইন ধরে ধরে

**১. আগে login যাচাই (frontend-এ)**
Backend-এর `POST /booking`-এ `checkAuth` আছে, login ছাড়া পাঠালে 401/403 ফেরত দেবে। তাহলে frontend-এ আগে দেখা কেন? কারণ user-কে ভালো অভিজ্ঞতা দেওয়া: অপ্রয়োজনীয় request না পাঠিয়ে সরাসরি "Please log in" বলে login page-এ নেওয়া। (নিরাপত্তা তবু backend-এর `checkAuth`-এই, ১৬ নম্বর নোটের নীতি।)

**২. `state: { from: location.pathname }`**
Login page-কে জানিয়ে রাখা "login-এর পরে `/tours/sajek-valley`-এ ফিরিয়ে দিও" (ধাপ ৩)। ১৩ নম্বর নোটে Verify page-এ email যেভাবে `state` দিয়ে গিয়েছিল, একই কৌশল।

**৩. `window.location.href`, `navigate` না**
`navigate()` শুধু **আমাদের app-এর ভেতরের** route-এ যেতে পারে। SSLCommerz সম্পূর্ণ আলাদা website (`sandbox.sslcommerz.com/…`), তাই browser-কে সত্যিই নতুন page-এ পাঠাতে হয়: `window.location.href = url`। ১৩ নম্বর নোটের Google login-এর একই কারণ। (আর `<a href>`-এর নিষেধ এখানে প্রযোজ্য না, কারণ এটা কোডের ভেতর থেকে redirect, link না।)

**৪. `if (res.paymentUrl)`**
Backend সফল বললেও link না এলে (backend-এর bug, ধাপ ৬) user-কে কোথাও পাঠানো যায় না; তখন পরিষ্কার error।

**৫. Button বন্ধ থাকে কখন**
- **`isBooking`:** request চলার সময়। দুবার চাপলে দুটো booking আর দুটো payment তৈরি হতো।
- **`tour.costFrom === undefined`:** দাম ছাড়া tour-এর booking backend নেয় না ("This tour has no price set yet")। Button চালু রেখে error দেখানোর চেয়ে আগেই বন্ধ রাখা ভালো।

✅ **যাচাই:**
- Logout অবস্থায় Book now → toast "Please log in…" → `/login`
- Login অবস্থায় → "Creating your booking…" → SSLCommerz-এর sandbox page খোলে
- দ্রুত দুবার চাপা যায় না
- দাম ছাড়া tour-এ button বন্ধ

---

## ধাপ ৩ — Login-এর পরে আগের page-এ ফেরা

🎯 **লক্ষ্য:** Tour থেকে login-এ গেলে, login-এর পরে সেই tour-এ ফেরা।

🛠️ **কাজ:**

```tsx
// src/components/modules/Authentication/LoginForm.tsx
import { useLocation, useNavigate } from "react-router";

const location = useLocation();
const from = (location.state as { from?: string } | null)?.from ?? "/";

// onSubmit-এ login সফল হলে
navigate(from, { replace: true });
```

💡 **লাইনগুলোর মানে:**
- **`location.state as { from?: string } | null`:** `state` যেকোনো কিছু হতে পারে (TypeScript `unknown` ধরে), কেউ সরাসরি `/login` খুললে `null`। তাই আকারটা বলে দেওয়া, আর `?.from` দিয়ে নিরাপদে পড়া।
- **`?? "/"`:** কোথাও থেকে না এলে Home।
- **`replace: true`:** history-তে `/login` থাকে না। Tour-এ ফিরে Back চাপলে আবার login page-এ যায় না (১৩ নম্বর নোটের `replace`)।

এই একই ব্যবস্থা ১৬ নম্বর নোটের `withAuth`-এর পরামর্শেও (`state={{ from: location.pathname }}`) কাজ করে: protected page থেকে login-এ পাঠালে login-এর পরে সেখানে ফেরে। এক `from` দিয়ে দুই কাজ।

✅ **যাচাই:** Logout অবস্থায় একটা tour → Book now → login → login-এর পরে সেই tour-এর page; Back চাপলে login page-এ যায় না।

---

## ধাপ ৪ — একটা component-এ তিনটা ফলাফলের page

🎯 **লক্ষ্য:** Success, fail, cancel তিনটার জন্য তিনটা আলাদা page না লিখে একটা।

🛠️ **কাজ (মূল অংশ):**

```tsx
// src/pages/PaymentResult.tsx
import { Ban, CircleCheck, CircleX } from "lucide-react";
import { Link, useSearchParams } from "react-router";

type TStatus = "success" | "fail" | "cancel";

const CONTENT: Record<TStatus, { title: string; text: string; icon: typeof CircleCheck; iconCls: string }> = {
  success: {
    title: "Payment successful",
    text: "Your booking is confirmed. A confirmation and invoice have been sent to your email.",
    icon: CircleCheck,
    iconCls: "bg-mint-soft text-success",
  },
  fail: { title: "Payment failed", text: "...", icon: CircleX, iconCls: "bg-destructive/10 text-destructive" },
  cancel: { title: "Payment cancelled", text: "...", icon: Ban, iconCls: "bg-yellow-soft text-warning" },
};

export default function PaymentResult({ status }: { status: TStatus }) {
  const [searchParams] = useSearchParams();
  const transactionId = searchParams.get("transactionId");
  const amount = searchParams.get("amount");

  const { title, text, icon: Icon, iconCls } = CONTENT[status];
  // ...icon, title, text, transaction-এর তথ্য, button
}
```

### 💡 লাইন ধরে ধরে

**১. `Record<TStatus, {...}>`**
"`TStatus`-এর **প্রতিটা** মানের জন্য একটা করে entry, আর প্রতিটার আকার এই"। তিনটার একটা লিখতে ভুলে গেলে, বা `"faild"` বানান ভুল করলে, TypeScript সাথে সাথে বলে। পরে নতুন status (যেমন `"pending"`) `TStatus`-এ যোগ করলে TypeScript জোর করবে `CONTENT`-এও যোগ করতে।

**২. লেখা আর রং data-তে, JSX-এ না**
তিনটা page-এর কাঠামো এক (icon, শিরোনাম, লেখা, button), শুধু লেখা, icon আর রং আলাদা। পার্থক্যগুলো একটা object-এ রেখে, JSX একবার লেখা। ২০ নম্বর নোটের তিনটা সংখ্যার input-এর `map`-এর মতো একই ধারণা: যা বদলায় সেটা data, যা একই সেটা কোড।

**৩. `icon: typeof CircleCheck`**
Icon component নিজেই রাখা হচ্ছে (`<CircleCheck />` না, `CircleCheck`)। `typeof CircleCheck` = "lucide-এর একটা icon component-এর type"; তিনটা icon-এরই একই type।

**৪. `icon: Icon` — নাম বদলে বড় হাতের**
```tsx
const { icon: Icon } = CONTENT[status];
<Icon className="size-8" />
```
JSX-এ component-এর নাম বড় হাতের হতে হয় (১৬ নম্বর নোট: `<icon />` লিখলে React HTML tag ভাবত)। তাই destructure-এর সময় `icon`-কে `Icon` নামে নেওয়া।

**৫. `useSearchParams()`**
URL-এর `?transactionId=TRX123&amount=7500` পড়ার React Router hook। `searchParams.get("amount")` → `"7500"` (লেখা; না থাকলে `null`)। দেখানোর সময় `Number(amount).toLocaleString("en-IN")` → "৳7,500"।

**৬. ফলাফল অনুযায়ী button**
Success হলে "View my bookings" (`/user/bookings`), নাহলে "Back to tours"; সাথে সবসময় "Go home"।

---

## ধাপ ৫ — Route: `element` দিয়ে prop

🎯 **লক্ষ্য:** তিনটা URL, একটা component, আলাদা `status`।

🛠️ **কাজ:**

```tsx
// src/routes/index.tsx  (App-এর children-এ)
import PaymentResult from "@/pages/PaymentResult";

{ path: "payment/success", element: <PaymentResult status="success" /> },
{ path: "payment/fail", element: <PaymentResult status="fail" /> },
{ path: "payment/cancel", element: <PaymentResult status="cancel" /> },
```

💡 **`Component` না, `element` কেন:** আমাদের নিয়ম `Component: X`, কিন্তু `Component`-এ prop দেওয়া যায় না (১৪ নম্বর নোটের `<Navigate>`-এর একই কারণ)। এখানে `status` বলে দিতেই হবে, তাই `element: <PaymentResult status="..." />`।

💡 **`App`-এর ভেতরে:** ফলাফলের page-এ Navbar/Footer থাকুক, user সাথে সাথে site-এ ঘুরতে পারুক।

💡 **URL-গুলো backend-এর `.env`-এর সাথে হুবহু:** backend SSLCommerz-কে বলে রাখে "success হলে `http://localhost:3000/payment/success`-এ পাঠাও"। Frontend-এর route `payment/success` না হয়ে `payment-success` হলে user 404 পাবে, অথচ টাকা কেটে গেছে।

✅ **যাচাই:** সরাসরি `/payment/success?transactionId=TEST&amount=1200` খুলুন → সবুজ ✓, "Payment successful", Transaction ID আর ৳1,200। `/payment/fail` আর `/payment/cancel`-ও আলাদা রং আর লেখায়।

---

## ধাপ ৬ — Backend সমস্যা ১: URL-এ `[object Object]`

🎯 **লক্ষ্য:** Book now চাপার পরে browser `http://localhost:3000/[object Object]`-এ চলে যাচ্ছিল, কেন তা বোঝা আর ঠিক করা।

### 💡 কী হচ্ছিল

Backend SSLCommerz-এর **পুরো response object** `paymentUrl` হিসেবে পাঠাচ্ছিল:

```json
"paymentUrl": { "status": "SUCCESS", "GatewayPageURL": "https://sandbox.sslcommerz.com/...", "sessionkey": "...", ... }
```

আর frontend করছিল `window.location.href = res.paymentUrl`। `href`-এ object দিলে JavaScript সেটাকে লেখায় রূপান্তর করে, আর যেকোনো সাধারণ object-এর লেখা রূপ হলো **`"[object Object]"`**। Browser সেটাকে একটা relative URL ভেবে `/[object Object]`-এ চলে গেল।

```js
String({ a: 1 })   // "[object Object]"
```

(১৯ নম্বর নোটে FormData-তে object দিলে যে `"[object Object]"` যায়, একই কারণ।)

### 🛠️ আসল সমাধান: backend শুধু link-টা পাঠাবে

```ts
// backend: modules/sslCommerz/sslCommerz.service.ts
const response = await axios({
  method: "POST",
  url: envVars.SSL.SSL_PAYMENT_API,
  data: data,
  headers: { "Content-Type": "application/x-www-form-urlencoded" },
});

// SSLCommerz ফেরত দেয় { status: "SUCCESS" | "FAILED", GatewayPageURL, failedreason, ... }
if (response.data?.status !== "SUCCESS" || !response.data?.GatewayPageURL) {
  throw new Error(response.data?.failedreason || "Could not start payment.");
}

return response.data.GatewayPageURL as string;
```

💡 **লাইনগুলোর মানে:**
- **`application/x-www-form-urlencoded`:** SSLCommerz-এর API JSON না, পুরনো HTML form-এর ধরনে data চায় (`key1=value1&key2=value2`)। এটা ওদের নিয়ম।
- **`status !== "SUCCESS"` যাচাই:** ভুল store id/password হলে SSLCommerz error-ও **২০০ status দিয়ে** পাঠায় (`status: "FAILED"`, `failedreason: "Store Credential Error"`)। axios সেটাকে সফল ভাবে। তাই নিজে `status` দেখে ব্যর্থ হলে আসল কারণসহ error।
- **শুধু `GatewayPageURL` ফেরত:** function-এর কাজ "payment-এর link দেওয়া", তাই শুধু link (string)। Frontend-কে SSLCommerz-এর ভেতরের আকার জানতে হয় না।

### 💡 Frontend-এ বাড়তি সুরক্ষা

```tsx
// server string বা পুরো SSLCommerz object, দুটোতেই চলবে
const paymentUrl =
  typeof res.paymentUrl === "string"
    ? res.paymentUrl
    : (res.paymentUrl as { GatewayPageURL?: string } | undefined)?.GatewayPageURL;
```

Backend ঠিক হওয়ার পরে এটা লাগে না, কিন্তু রাখলে backend কখনো আবার পুরো object পাঠালেও ভাঙবে না। (উন্নতি ৪-এ এর type পরিষ্কার করা।)

✅ **যাচাই:** Book now → সরাসরি `sandbox.sslcommerz.com/...`; আর কখনো `/[object Object]` না।

---

## ধাপ ৭ — Backend সমস্যা ২: email না যাওয়ায় payment cancel

🎯 **লক্ষ্য:** Payment সফল হলেও user cancel/fail page-এ যাচ্ছিল, কারণ invoice-এর email পাঠানো ব্যর্থ হচ্ছিল।

### 💡 কী হচ্ছিল

Payment সফল হলে backend invoice email পাঠায়, আর email-এর নকশা একটা **EJS template** ফাইল (`invoice.ejs`)। Build-এর পরে:

```
dist/templates/templates/invoice.ejs   ← ফাইল এখানে (দুইবার templates)
dist/templates/invoice.ejs             ← কোড এখানে খোঁজে, পায় না
```

Template না পেয়ে email পাঠানো error দিল, সেই error payment success-এর কাজের মাঝখানে ঘটায় পুরো কাজ ব্যর্থ ধরা হলো, আর user-কে fail/cancel page-এ পাঠানো হলো, যদিও টাকা দেওয়া হয়ে গেছে।

### 💡 কেন দুইবার `templates`

**১. `tsup` শুধু TypeScript build করে:** backend-এর `tsup` `.ts` ফাইলগুলো JavaScript বানিয়ে `dist/`-এ রাখে। `.ejs` TypeScript না, তাই `tsup` এটা ছুঁয়েও দেখে না, `dist/`-এ যায়ই না। তাই নিজে কপি করতে হয়।

**২. `cp -r`-এর একটা কৌশল:**

```sh
cp -r src/app/utils/templates dist/templates
```

এর ফল নির্ভর করে `dist/templates` **আগে থেকে আছে কিনা**:

| `dist/templates` আগে | ফল |
|---|---|
| নেই | `templates` folder-টা কপি হয়ে নাম হয় `dist/templates` ✅ |
| **আছে** | folder-টা তার **ভেতরে** কপি হয় → `dist/templates/templates` ❌ |

প্রথম build-এ ঠিক, দ্বিতীয় build থেকে (folder আগেই আছে) দুইবার templates। এজন্যই সমস্যাটা মাঝে মাঝে আসে, ধরতে কঠিন।

### 🛠️ স্থায়ী সমাধান

```ts
// backend: tsup.config.ts
// build শেষ হলে templates-এর ভেতরের ফাইলগুলো কপি করো
onSuccess: "mkdir -p dist/templates && cp -r src/app/utils/templates/. dist/templates/",
```

💡 **লাইনগুলোর মানে:**
- **`onSuccess`:** tsup build সফলভাবে শেষ হলে এই command চালায়।
- **`mkdir -p dist/templates`:** folder না থাকলে বানাও; থাকলে (`-p`) error দিও না।
- **`templates/.`** (শেষে `/.`): "folder-টা না, folder-এর **ভেতরের সবকিছু**"। তাই `dist/templates` আগে থাকুক বা না থাকুক, ফাইলগুলো সবসময় সরাসরি `dist/templates/invoice.ejs`-এ যায়। দুইবার templates আর কখনো হবে না।

✅ **যাচাই:** Backend দুইবার পরপর build করুন → দুবারই `dist/templates/invoice.ejs` আছে, `dist/templates/templates` নেই। Sandbox-এ payment সফল করলে success page, আর email-এ invoice।

---

# কী বাকি: সমস্যা চেনা

| # | সমস্যা |
|---|---|
| ১ | **Success page URL বিশ্বাস করে।** কেউ নিজে `/payment/success?amount=99999` টাইপ করলেও "Payment successful" দেখায়। দেখানো শুধু, এতে booking নিশ্চিত হয় না; কিন্তু screenshot নিয়ে কেউ ভুল দাবি করতে পারে, বা user বিভ্রান্ত হতে পারে। |
| ২ | **`userInfo` লোড হওয়ার মুহূর্তে Book now চাপলে** `isLoggedIn` মিথ্যা (data এখনো আসেনি), তাই login করা user-কেও login page-এ পাঠিয়ে দেয়। |
| ৩ | **Email ব্যর্থ হলে payment-ই "ব্যর্থ"।** ধাপ ৭-এ template ঠিক হলো, কিন্তু মূল সমস্যা থেকে গেল: email কোনো কারণে ব্যর্থ হলে (email server বন্ধ, ভুল password) আবার টাকা দেওয়া user fail page দেখবে। |
| ৪ | `TCreateBookingResponse`-এ `paymentUrl: string`, অথচ কোডে object-ও সামলানো হচ্ছে; type আর কোড মিলছে না। |
| ৫ | `bg-destructive/10` → `bg-destructive-soft`; `<h1>`-এ `text-foreground`; PaymentResult-এ `` `…${iconCls}` `` দিয়ে class জোড়া (`cn()` না)। |
| ৬ | Error-এ `errorSources` দেখা হয় না (`getErrorMessage` না)। |

---

# উন্নতি

## উন্নতি ১ — Success page-এ আসল অবস্থা backend থেকে

**নীতি:** URL-এ যা আছে সেটা যে কেউ লিখতে পারে। টাকা/booking-এর অবস্থা সবসময় **backend থেকে** জানতে হয়।

**কীভাবে (backend-এর সাথে মিলিয়ে):** backend-এ transaction id দিয়ে payment/booking-এর অবস্থা ফেরত দেওয়ার একটা protected endpoint (যেমন `GET /payment/:transactionId`, শুধু ঐ booking-এর মালিক দেখতে পারবে)। Frontend-এ:

```ts
// booking.api.ts (উদাহরণ — backend-এর route-এর নাম মিলিয়ে নিন)
getPaymentStatus: builder.query<{ status: "PAID" | "UNPAID" | "FAILED" | "CANCELLED"; amount: number }, string>({
  query: (transactionId) => ({ url: `/payment/${transactionId}`, method: "GET" }),
  transformResponse: (response: IResponse<{ status: "PAID" | "UNPAID" | "FAILED" | "CANCELLED"; amount: number }>) =>
    response.data,
}),
```

```tsx
// PaymentResult-এ
const { data, isLoading } = useGetPaymentStatusQuery(transactionId ?? skipToken);
// দেখানো: data.status আর data.amount (URL-এর amount না)
```

তখন success-এর লেখা দেখাবে শুধু backend `PAID` বললে, আর amount আসবে database থেকে। এর আগে পর্যন্ত, এখনকার page ঠিক আছে "user-কে তথ্য জানানোর" জন্য; শুধু মনে রাখুন এটা প্রমাণ না। (আসল নিশ্চিতকরণ ইতিমধ্যে backend-এ হচ্ছে, SSLCommerz-এর ফলাফল দেখে।)

## উন্নতি ২ — Login-এর অবস্থা জানার আগে সিদ্ধান্ত না

```tsx
const { data: userData, isLoading: isUserLoading } = useUserInfoQuery(undefined);

<Button disabled={isBooking || isUserLoading || tour.costFrom === undefined} ...>
```

১৬ নম্বর নোটের মূল নীতি: "জানি না" মানে "না" না। User-এর অবস্থা আসা পর্যন্ত button এক মুহূর্ত বন্ধ; তারপর ঠিক সিদ্ধান্ত।

## উন্নতি ৩ — Backend: email ব্যর্থ হলেও payment সফল

১৯ আর ২০ নম্বর নোটের Cloudinary cleanup-এর একই নীতি: **user-এর আসল কাজ (টাকা দিয়ে booking) হয়ে গেছে; email পাঠানো পার্শ্ব-কাজ।** Payment success-এর কাজে email আলাদা `try/catch`-এ:

```ts
// backend: payment success-এর service-এ (উদাহরণ)
// ১. Payment PAID আর Booking CONFIRMED (database) — এটা জরুরি, ব্যর্থ হলে পুরো কাজ ব্যর্থ
// ...

// ২. Invoice email — ব্যর্থ হলেও payment সফলই
try {
  await sendEmail({ to: user.email, subject: "Your booking invoice", templateName: "invoice", templateData });
} catch (error) {
  console.error("Invoice email failed for transaction", transactionId, error);
}

// ৩. Success-এর redirect
```

তাহলে email server বন্ধ থাকলেও user success page দেখবে, booking নিশ্চিত থাকবে; শুধু email পরে আবার পাঠানোর ব্যবস্থা (বা "My bookings" থেকে invoice download) লাগবে। (Backend-এর আসল function-এর নাম আর গঠন আপনার কোড অনুযায়ী।)

## উন্নতি ৪ — Type আর কোড এক করা

```ts
// booking.api.ts
type TCreateBookingResponse = {
  paymentUrl: string | { GatewayPageURL?: string }; // পুরনো backend পুরো object পাঠাত
  booking: unknown;
};
```

```tsx
// handleBook-এ
const paymentUrl = typeof res.paymentUrl === "string" ? res.paymentUrl : res.paymentUrl?.GatewayPageURL;
```

এখন `as` লাগে না: type-এই লেখা আছে দুই রকম আসতে পারে, আর `typeof` দেখে TypeScript নিজে বোঝে কোন শাখায় কোনটা। Backend নিশ্চিতভাবে ঠিক হওয়ার পরে দুই জায়গা থেকেই object-এর অংশ মুছে শুধু `string` রাখুন।

## উন্নতি ৫ — ছোটখাটো

- PaymentResult: `fail`-এর `iconCls` → `"bg-destructive-soft text-destructive"`; `<h1>`-এ `text-foreground`; `` className={`… ${iconCls}`} `` → `className={cn("flex size-16 items-center justify-center rounded-full", iconCls)}`।
- `handleBook`-এর catch-এ `getErrorMessage(err, "Booking failed. Please try again.")` (১৭ নম্বর নোট)।

---

# চূড়ান্ত কোড

### `src/redux/features/Booking/booking.api.ts`

```ts
import { baseApi } from "@/redux/baseApi";
import type { IResponse } from "@/types";

type TCreateBookingPayload = { tour: string; guestCount: number };
type TCreateBookingResponse = {
  paymentUrl: string | { GatewayPageURL?: string }; // পুরনো backend পুরো SSLCommerz object পাঠাত
  booking: unknown;
};

export const bookingApi = baseApi.injectEndpoints({
  endpoints: (builder) => ({
    createBooking: builder.mutation<TCreateBookingResponse, TCreateBookingPayload>({
      query: (payload) => ({
        url: "/booking",
        method: "POST",
        data: payload,
      }),
      transformResponse: (response: IResponse<TCreateBookingResponse>) => response.data,
      invalidatesTags: ["BOOKING"],
    }),
  }),
});

export const { useCreateBookingMutation } = bookingApi;
```

### `TourDetails.tsx` — booking-এর অংশ (উন্নতিসহ)

```tsx
import { getErrorMessage } from "@/utils/getErrorMessage";

const { data: userData, isLoading: isUserLoading } = useUserInfoQuery(undefined);
const [createBooking, { isLoading: isBooking }] = useCreateBookingMutation();
const isLoggedIn = Boolean(userData?.data?.email);

const handleBook = async () => {
  // login নেই → login page, পরে এখানেই ফিরে আসবে
  if (!isLoggedIn) {
    toast.info("Please log in to book this tour");
    navigate("/login", { state: { from: location.pathname } });
    return;
  }

  const toastId = toast.loading("Creating your booking…");
  try {
    const res = await createBooking({ tour: tour._id, guestCount: guests }).unwrap();
    const paymentUrl = typeof res.paymentUrl === "string" ? res.paymentUrl : res.paymentUrl?.GatewayPageURL;

    if (paymentUrl) {
      toast.success("Redirecting to payment…", { id: toastId });
      window.location.href = paymentUrl; // বাইরের site, তাই navigate না
    } else {
      toast.error("Payment link not received. Please try again.", { id: toastId });
    }
  } catch (err) {
    toast.error(getErrorMessage(err, "Booking failed. Please try again."), { id: toastId });
  }
};

// Button
<Button
  type="button"
  onClick={handleBook}
  disabled={isBooking || isUserLoading || tour.costFrom === undefined}
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
```

### `src/pages/PaymentResult.tsx` — আপনার বর্তমান কোড

> উন্নতি ৫ বসাতে: `fail`-এর `iconCls`-এ `bg-destructive-soft`, `<h1>`-এ `text-foreground`, icon-এর `span`-এ `cn(...)` (import `@/lib/utils`)। বাকি সব হুবহু।

```tsx
import { Button } from "@/components/ui/button";
import { Ban, CircleCheck, CircleX } from "lucide-react";
import { Link, useSearchParams } from "react-router";

type TStatus = "success" | "fail" | "cancel";

const CONTENT: Record<TStatus, { title: string; text: string; icon: typeof CircleCheck; iconCls: string }> = {
	success: {
		title: "Payment successful",
		text: "Your booking is confirmed. A confirmation and invoice have been sent to your email.",
		icon: CircleCheck,
		iconCls: "bg-mint-soft text-success",
	},
	fail: {
		title: "Payment failed",
		text: "Your payment could not be completed. No money was taken. Please try booking again.",
		icon: CircleX,
		iconCls: "bg-destructive/10 text-destructive",
	},
	cancel: {
		title: "Payment cancelled",
		text: "You cancelled the payment. Your booking was not confirmed.",
		icon: Ban,
		iconCls: "bg-yellow-soft text-warning",
	},
};

export default function PaymentResult({ status }: { status: TStatus }) {
	const [searchParams] = useSearchParams();
	const transactionId = searchParams.get("transactionId");
	const amount = searchParams.get("amount");

	const { title, text, icon: Icon, iconCls } = CONTENT[status];

	return (
		<div className="mx-auto flex max-w-lg flex-col items-center gap-5 px-4 py-20 text-center">
			<span className={`flex size-16 items-center justify-center rounded-full ${iconCls}`}>
				<Icon className="size-8" />
			</span>

			<div className="flex flex-col gap-2">
				<h1 className="text-3xl font-bold tracking-tight">{title}</h1>
				<p className="text-muted-foreground">{text}</p>
			</div>

			{(transactionId || amount) && (
				<dl className="flex w-full flex-col gap-2.5 rounded-xl bg-card p-5 text-left text-sm shadow-card">
					{transactionId && (
						<div className="flex justify-between gap-4">
							<dt className="text-muted-foreground">Transaction ID</dt>
							<dd className="font-medium break-all">{transactionId}</dd>
						</div>
					)}
					{amount && (
						<div className="flex justify-between gap-4">
							<dt className="text-muted-foreground">Amount</dt>
							<dd className="font-semibold">৳{Number(amount).toLocaleString("en-IN")}</dd>
						</div>
					)}
				</dl>
			)}

			<div className="flex flex-wrap justify-center gap-2.5">
				{status === "success" ? (
					<Button asChild className="h-11 rounded-lg px-5">
						<Link to="/user/bookings">View my bookings</Link>
					</Button>
				) : (
					<Button asChild className="h-11 rounded-lg px-5">
						<Link to="/tours">Back to tours</Link>
					</Button>
				)}
				<Button asChild variant="outline" className="h-11 rounded-lg px-5">
					<Link to="/">Go home</Link>
				</Button>
			</div>
		</div>
	);
}
```

### `src/routes/index.tsx` — App-এর children

```tsx
{ index: true, Component: Home },
{ Component: About, path: "about" },
{ Component: Tours, path: "tours" },
{ Component: TourDetails, path: "tours/:slug" },
{ path: "payment/success", element: <PaymentResult status="success" /> },
{ path: "payment/fail", element: <PaymentResult status="fail" /> },
{ path: "payment/cancel", element: <PaymentResult status="cancel" /> },
```

### Backend

- `sslCommerz.service.ts`: ধাপ ৬-এর কোড (শুধু `GatewayPageURL` ফেরত)।
- `tsup.config.ts`: ধাপ ৭-এর `onSuccess`।
- Payment success-এর service: উন্নতি ৩ (email আলাদা `try/catch`-এ)।

---

# Check-list

**Booking**
- [ ] Logout অবস্থায় Book now → "Please log in" → login → login-এর পরে **সেই tour-এ** ফেরে; Back চাপলে login page আসে না
- [ ] Login অবস্থায় Book now → "Creating your booking…" → SSLCommerz sandbox (কখনো `/[object Object]` না)
- [ ] Network → `booking` → Payload-এ শুধু `{ tour, guestCount }`, দাম নেই
- [ ] Request চলার সময় button-এ "Processing…", দুবার চাপা যায় না; দাম ছাড়া tour-এ button বন্ধ
- [ ] Page খোলার মুহূর্তে (user-এর তথ্য আসার আগে) button এক মুহূর্ত বন্ধ (উন্নতি ২)

**Payment-এর ফলাফল**
- [ ] Sandbox-এ সফল payment → `/payment/success`, transaction id আর টাকা দেখায়, email-এ invoice
- [ ] Sandbox-এ fail আর cancel → নিজ নিজ page, আলাদা রং আর লেখা
- [ ] Success-এ "View my bookings", বাকিগুলোয় "Back to tours"
- [ ] Dark mode-এ তিনটা page-এর রং ঠিক

**Backend**
- [ ] দুবার পরপর build-এর পরেও `dist/templates/invoice.ejs` আছে, `dist/templates/templates` নেই
- [ ] (উন্নতি ৩-এর পরে) email server বন্ধ থাকলেও সফল payment-এ success page
