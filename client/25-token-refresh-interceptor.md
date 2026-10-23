# Frontend Notes — ২৫: Token-এর মেয়াদ শেষ হলে নিজে থেকে নতুন Token — Axios Interceptor আর Request Queue

১৩ নম্বর নোটের ধাপ ৭-এ interceptor-এর ধারণা দেখেছিলাম, আর বলেছিলাম "token-এর মেয়াদ শেষ হলে interceptor নিজে নতুন token এনে আবার চেষ্টা করবে, teacher পরে দেখাবেন"। এই নোটে ঠিক সেটা, দুই ধাপে:

1. **৪০-১২:** মেয়াদ শেষের error ধরা, refresh token দিয়ে নতুন access token আনা, ব্যর্থ request আবার পাঠানো।
2. **৪০-১৩:** একসাথে অনেক request ব্যর্থ হলে সবগুলোকে **লাইনে দাঁড় করিয়ে** (queue) একবারই refresh করা।

দ্বিতীয় ধাপটা ২৪ নম্বর নোটের "resolve/reject বাইরে রেখে পরে ডাকা" কৌশলের সরাসরি ব্যবহার, তাই আগে সেটা একবার দেখে নিলে বোঝা সহজ হবে।

প্রতিটা ধাপে 🎯 লক্ষ্য, 🛠️ কোড, 💡 ব্যাখ্যা, ✅ যাচাই। শেষে সমস্যা, উন্নতি আর চূড়ান্ত কোড।

> নোট: আপনার project-এর বর্তমান `lib/axios.ts` পড়ে লেখা (axios 1.x)। পুরনো ধরনের কোড দিয়ে কিছু বদলানো হয়নি।

---

## ভূমিকা: সমস্যাটা কী

আমাদের backend দুটো token দেয় (১৩ নম্বর নোট):

| Token | মেয়াদ (`.env`) | কাজ |
|---|---|---|
| **Access token** | `JWT_ACCESS_EXPIRES=1d` | প্রতিটা protected request-এ যায়: "আমি কে" |
| **Refresh token** | `JWT_REFRESH_EXPIRES=30d` | শুধু একটা কাজ: নতুন access token আনা |

**এখনকার অবস্থা:** access token-এর মেয়াদ শেষ হলে `/user/me` ব্যর্থ হয় → Navbar আর `withAuth` ভাবে user logged out → user-কে আবার login করতে হয়। অথচ তার কাছে ৩০ দিনের refresh token তখনো আছে!

**আমরা যা চাই:**

```
access token মেয়াদোত্তীর্ণ
   → request ব্যর্থ ("jwt expired")
   → interceptor ধরে ফেলে (component পর্যন্ত যেতে দেয় না)
   → refresh token দিয়ে নতুন access token (backend নতুন cookie বসায়)
   → ব্যর্থ request আবার পাঠায় → এবার সফল
   → component data পায়, user কিছুই টের পায় না
```

আপনার নোটের কথাটা: এখনকার application **unsafe না** (মেয়াদ শেষ হলে ঠিকই logout হয়), সমস্যা শুধু **অভিজ্ঞতায়**: প্রতিদিন আবার login। আমরা নিরাপত্তা না কমিয়ে (access token ছোট মেয়াদেরই থাকছে) সেটা ঠিক করছি।

### ধাপের মানচিত্র

| ধাপ | বিষয় |
|---|---|
| ১ | পরীক্ষার প্রস্তুতি: access token ৫ সেকেন্ড |
| ২ | Response interceptor-এর নিয়ম: সবসময় কিছু ফেরত দাও |
| ৩ | "jwt expired" ধরা |
| ৪ | Refresh: নতুন token এল, তবু logout কেন |
| ৫ | ব্যর্থ request আবার পাঠানো |
| ৬ | অসীম loop থেকে বাঁচা: `_retry` |
| ৭ | নতুন সমস্যা: একসাথে অনেক request |
| ৮ | সমাধান: queue (লাইন) |

---

## ধাপ ১ — পরীক্ষার প্রস্তুতি: access token ৫ সেকেন্ড

🎯 **লক্ষ্য:** ১ দিন অপেক্ষা না করে সমস্যাটা সাথে সাথে দেখা।

🛠️ **কাজ (backend, VPS-এ):**

```env
# server-এর .env
JWT_ACCESS_EXPIRES=5s
JWT_REFRESH_EXPIRES=30d
```

```sh
pm2 reload 0 --update-env
```

💡 **Command-টা টুকরো করে:**
- **`pm2`:** server-এ Node app চালু রাখার process manager (বন্ধ হলে নিজে আবার চালু করে)।
- **`reload`:** app নতুন করে চালু করা, কিন্তু **zero-downtime**: নতুনটা তৈরি হওয়া পর্যন্ত পুরনোটা চলতে থাকে, user-রা কোনো বিরতি টের পায় না (`restart` হলে এক মুহূর্ত বন্ধ থাকে)।
- **`0`:** কোন app, তার id। `pm2 list` দিলে id দেখা যায়; আপনার app-এর id 0।
- **`--update-env`:** নতুন environment-এর মান নিয়ে চালু করো। এটা না দিলে pm2 অনেক সময় **আগের** environment দিয়েই আবার চালায়, `.env` বদলেও কিছু হয় না।

✅ **যাচাই (এবং সমস্যা):**
- Frontend-এ আগের cookie মুছে নতুন করে login → ৫ সেকেন্ড পরে কোনো page বদলালে বা reload দিলে **logout** হয়ে যায়।
- Postman-এ ৫ সেকেন্ড পরে protected route-এ: `"message": "jwt expired"`।

> ⚠️ **পরীক্ষা শেষে অবশ্যই ফেরত দিন:** `JWT_ACCESS_EXPIRES=1d` (বা আপনার আসল মান), তারপর আবার `pm2 reload 0 --update-env`। ৫ সেকেন্ড রেখে দিলে live site-এ প্রতি ৫ সেকেন্ডে refresh হবে, server-এর ওপর অকারণ চাপ।

---

## ধাপ ২ — Response interceptor-এর নিয়ম: সবসময় কিছু ফেরত দাও

🎯 **লক্ষ্য:** Interceptor-এর দুই function কী ফেরত দিলে কী হয়, বোঝা।

```ts
axiosInstance.interceptors.response.use(
  (response) => {
    // ১. সফল (2xx) হলে
    return response;
  },
  (error) => {
    // ২. ব্যর্থ হলে
    return Promise.reject(error);
  },
);
```

💡 **Interceptor হলো মাঝখানের পাহারাদার** (১৩ নম্বর নোট): backend-এর উত্তর component-এ পৌঁছানোর আগে এখানে আসে। Interceptor যা **ফেরত দেয়**, component (আমাদের ক্ষেত্রে `axiosBaseQuery`) সেটাই পায়।

| Interceptor ফেরত দেয় | যে request পাঠিয়েছিল সে পায় |
|---|---|
| `return response` | সফল উত্তর ✅ |
| `return Promise.reject(error)` | error, তার `catch` চলে ✅ |
| কিছুই না (`undefined`) | **সফল**, কিন্তু data `undefined` ❌ |
| নতুন একটা promise | সেই promise যা হবে, সেটাই (ধাপ ৫-এর কৌশল) |

আপনার নোটে লেখা ছিল: *"এটা server-এর `next`-এর মতো, response নিয়ে return করতে হবে।"* তুলনাটা ঠিক: Express middleware-এ `next()` না ডাকলে request আটকে যায়, এখানে কিছু ফেরত না দিলে উত্তর হারিয়ে যায়। Interceptor একটা শিকলের অংশ, পরের জনকে জিনিসটা হাতে দিতে হবে।

> ⚠️ **সংশোধন:** নোটের প্রথম পরীক্ষার কোডে error-এর function ছিল খালি: `() => {}`। এটা বিপজ্জনক: কিছু ফেরত না দেওয়া মানে **error চুপচাপ গিলে ফেলা**। ব্যর্থ request-কে axios তখন "সফল, data `undefined`" ধরে, ফলে `axiosBaseQuery`-র `catch` চলে না, toast-এ error আসে না, আর component `undefined.data` পড়তে গিয়ে অন্য কোথাও ভাঙে। Error-এর function সবসময় শেষে `return Promise.reject(error)`।

---

## ধাপ ৩ — "jwt expired" ধরা

🎯 **লক্ষ্য:** সব error-এর মধ্যে থেকে শুধু "token-এর মেয়াদ শেষ" চেনা।

🛠️ **কোড:**

```ts
axiosInstance.interceptors.response.use(
  (response) => response,
  async (error) => {
    if (error.response.status === 500 && error.response.data.message === "jwt expired") {
      console.log("Your token is expired");
      // ...পরের ধাপে
    }

    // বাকি সব error যেমন আছে তেমন সামনে যাক
    return Promise.reject(error);
  },
);
```

💡 **লাইনগুলোর মানে:**
- **`error.response`:** backend যা উত্তর দিয়েছে (`status`, `data`)। `console.log(error.response)` দিয়ে দেখা যায়।
- **`error.response.data.message === "jwt expired"`:** আমাদের backend এই লেখাটা পাঠায়। আসলে এটা backend-এর লেখা না, `jsonwebtoken` library-র: মেয়াদোত্তীর্ণ token যাচাই করতে গেলে সে `TokenExpiredError` ছোড়ে, যার message `"jwt expired"`।
- **শেষের `return Promise.reject(error)`:** "For Everything", অর্থাৎ যে error আমরা সামলাচ্ছি না (ভুল password, 404, validation), সেগুলো আগের মতোই component-এ যাক।

> ⚠️ **সংশোধন (backend): status 500 কেন?** 500 মানে "server-এর ভেতরে অপ্রত্যাশিত সমস্যা"। মেয়াদোত্তীর্ণ token server-এর সমস্যা না, user-এর পরিচয় যাচাই ব্যর্থ, এর সঠিক status **401 (Unauthorized)** (১৬ নম্বর নোটের শেষ অংশ)। 500 আসছে কারণ backend-এর `checkAuth`-এ `jwt.verify`-এর error কেউ ধরছে না, তাই global error handler সেটাকে সাধারণ "অজানা error" (500) বানাচ্ছে। এর দুটো ক্ষতি:
> 1. Frontend-কে **message-এর লেখা** দেখে চিনতে হচ্ছে, যেটা ভঙ্গুর (১৪ নম্বর নোটের "message দেখে চেনা ভালো উপায় না")। Library-র version বদলালে লেখা বদলাতে পারে।
> 2. সত্যিকারের server-এর সমস্যা (500) আর token-এর সমস্যা আলাদা করা যায় না।
>
> Backend-এ ঠিক করার উপায় উন্নতি ৩-এ। তবে teacher-এর backend এখন 500 দিচ্ছে, তাই frontend আপাতত সেটাই মেলাচ্ছে, ঠিক আছে।

✅ **যাচাই:** ৫ সেকেন্ড পরে কিছু করলে Console-এ "Your token is expired"।

---

## ধাপ ৪ — Refresh: নতুন token এল, তবু logout কেন

🎯 **লক্ষ্য:** Refresh token দিয়ে নতুন access token আনা, আর বোঝা কেন শুধু এটুকু যথেষ্ট না।

🛠️ **কোড:**

```ts
if (error.response.status === 500 && error.response.data.message === "jwt expired") {
  try {
    const res = await axiosInstance.post("/auth/refresh-token");
    console.log("New Token arrived", res);
  } catch (error) {
    console.error(error);
  }
}

return Promise.reject(error);
```

💡 **`/auth/refresh-token` কী করে:** browser cookie-তে থাকা refresh token নিজে থেকেই পাঠায় (`withCredentials: true`, ১৩ নম্বর নোট)। Backend সেটা যাচাই করে নতুন access token বানায় আর **নতুন cookie বসিয়ে দেয়**। Frontend-কে token হাতেও নিতে হয় না, কোথাও রাখতেও হয় না, এটাই httpOnly cookie-র সুবিধা।

✅ **যাচাই (এবং সমস্যা):** Console-এ "New Token arrived", Application → Cookies-এ নতুন `accessToken`। **তবু user logout হয়ে যায়!** আপনার নোটেও এটা লেখা।

### 💡 কেন logout হলো

সময়ের ক্রমটা দেখুন:

```
১. Navbar: GET /user/me  ──────────►  ✗ "jwt expired"
২. Interceptor: POST /auth/refresh-token ──► ✓ নতুন cookie
৩. Interceptor: return Promise.reject(error)   ← মূল error-টাই ফেরত দিল!
৪. /user/me-র উত্তর হিসেবে Navbar পেল: ব্যর্থ  → "logged out"
```

নতুন token এসেছে ঠিকই, কিন্তু **যে request ব্যর্থ হয়েছিল (`/user/me`), সেটা আর কেউ আবার পাঠায়নি।** Interceptor refresh করে পুরনো error-টাই সামনে পাঠিয়ে দিল। Navbar জানে না পেছনে নতুন token এসেছে; সে শুধু দেখে "আমার request ব্যর্থ"।

(Page reload দিলে আবার ঠিক হয়ে যায়, কারণ তখন `/user/me` নতুন cookie দিয়ে নতুন করে যায়।)

---

## ধাপ ৫ — ব্যর্থ request আবার পাঠানো

🎯 **লক্ষ্য:** নতুন token আসার পরে, যে request ব্যর্থ হয়েছিল সেটা আবার পাঠানো, আর তার ফলাফল আসল caller-কে দেওয়া।

🛠️ **কোড:**

```ts
async (error) => {
  const originalRequest = error.config;   // যে request ব্যর্থ হয়েছিল, তার পুরো setting

  if (error.response.status === 500 && error.response.data.message === "jwt expired") {
    try {
      await axiosInstance.post("/auth/refresh-token");
      return axiosInstance(originalRequest);   // একই request আবার
    } catch (error) {
      console.error(error);
    }
  }

  return Promise.reject(error);
}
```

### 💡 `error.config` — ব্যর্থ request-এর "রেসিপি"

Axios-এর প্রতিটা error-এর সাথে `error.config` থাকে: যে request ব্যর্থ হয়েছিল তার সব তথ্য, URL, method, data, headers। আপনার নোটের কথা: *console.log করলে original request পাব*। এটা একটা রেসিপির মতো: একই রেসিপি দিয়ে আবার রান্না করা যায়।

`axiosInstance(originalRequest)`: axios-কে বলা "এই setting দিয়ে আবার একটা request পাঠাও"। এবার browser **নতুন cookie** পাঠায়, তাই সফল।

### 💡 সবচেয়ে জরুরি অংশ: `return`

```ts
return axiosInstance(originalRequest);
```

ধাপ ২-এর টেবিল মনে করুন: interceptor **একটা নতুন promise ফেরত দিলে, caller সেই promise-এর ফলই পায়।** তাই:

```
Navbar: GET /user/me ───────────────┐
                                     │ (Navbar অপেক্ষা করছে)
   ✗ jwt expired                     │
   interceptor: refresh ✓            │
   interceptor: return axiosInstance(originalRequest)
                  └─ GET /user/me (নতুন cookie) ✓ ──► Navbar পায়: সফল, user-এর data
```

Navbar-এর দিক থেকে মনে হয় তার request একটু দেরিতে, কিন্তু **সফলভাবে** এসেছে। মাঝখানে ব্যর্থতা, refresh, আবার চেষ্টা, কিছুই সে জানে না। RTK Query-ও জানে না: `axiosBaseQuery`-র `await axiosInstance(...)` সরাসরি সফল ফল পায় (১৩ নম্বর নোট), তাই কোনো error state, কোনো toast, কিছুই না।

আপনার নোটের comment: *"যেই request fail হচ্ছিল সেটা আমি axiosInstance করে পাঠিয়ে দিচ্ছি, so এখন আর আমার request fail হয়ে logout হবে না"*, ঠিক।

✅ **যাচাই:** ৫ সেকেন্ড পরে page বদলান → Network-এ পরপর: `me` (500) → `refresh-token` (200) → `me` (200); Navbar-এ user থেকে যায়, logout হয় না।

---

## ধাপ ৬ — অসীম loop থেকে বাঁচা: `_retry`

🎯 **লক্ষ্য:** আবার পাঠানো request-ও "jwt expired" দিলে যেন আবার-আবার-আবার না চলে।

### 💡 সমস্যা

ধরুন কোনো কারণে refresh সফল হলো কিন্তু নতুন token-ও কাজ করল না (backend-এ ভুল, ঘড়ির গোলমাল)। তখন:

```
/user/me ✗ → refresh ✓ → /user/me আবার ✗ → refresh ✓ → /user/me আবার ✗ → … চিরকাল
```

আবার পাঠানো request-ও interceptor দিয়ে যায়, আবার "jwt expired", আবার refresh… browser আটকে যায়, server-এ হাজার request।

### 🛠️ সমাধান: একটা চিহ্ন

```ts
const originalRequest = error.config as InternalAxiosRequestConfig & { _retry?: boolean };

if (
  error.response?.status === 500 &&
  error.response?.data?.message === "jwt expired" &&
  !originalRequest._retry                          // আগে আবার চেষ্টা করা হয়নি
) {
  originalRequest._retry = true;                    // চিহ্ন: এটা একবার আবার চেষ্টা হচ্ছে
  // ...refresh আর আবার পাঠানো
}
```

💡 **কীভাবে কাজ করে:** `_retry` হলো আমাদের নিজের বানানো একটা চিহ্ন, request-এর setting object-এর গায়ে লাগানো। প্রথমবার নেই (`undefined`), তাই refresh হয়, আর চিহ্ন লাগে। একই `originalRequest` আবার পাঠানো হয়; সেটাও ব্যর্থ হলে, এবার `_retry === true`, তাই শর্ত মিথ্যা, সরাসরি `Promise.reject`। **প্রতিটা request সর্বোচ্চ একবার** আবার চেষ্টা।

💡 **নামের শুরুতে `_` কেন:** axios-এর নিজের কোনো setting-এর সাথে নাম যেন না মেলে; `_` দিয়ে বোঝানো "এটা আমাদের নিজস্ব, axios-এর না"।

💡 **Type:** axios-এর config type-এ `_retry` নেই, তাই `& { _retry?: boolean }` দিয়ে যোগ করা (২১ নম্বর নোটের `&`)। `?` কারণ প্রথমবার এটা থাকে না।

---

## ধাপ ৭ — নতুন সমস্যা: একসাথে অনেক request

🎯 **লক্ষ্য:** এখনকার কোডের লুকানো সমস্যা চেনা (teacher-এর "ছোট ২টা issue")।

Admin-এর dashboard খুললে একসাথে কয়েকটা request যায়: `/user/me` (Navbar), `/user/me` (Sidebar, cache থেকে), `/tour`, `/division`, `/tour/tour-types` …। Token মেয়াদোত্তীর্ণ হলে **সবগুলো একসাথে** ব্যর্থ হয়:

```
সময় →
/tour      ✗ ─► refresh #1 ───────────► ✓ ─► /tour আবার ✓
/division  ✗ ─► refresh #2 ─────────► ✓ ─► /division আবার ✓
/tour-types✗ ─► refresh #3 ───────────► ✓ ─► …
/user/me   ✗ ─► refresh #4 ──────────► ✓ ─► …
```

💡 **সমস্যা:**
1. **একই কাজ ৪ বার:** ৪টা refresh request, অথচ একটাই দরকার ছিল।
2. **Refresh token rotation থাকলে ভাঙে:** অনেক backend নিরাপত্তার জন্য প্রতিবার refresh-এ refresh token-ও বদলায় আর পুরনোটা বাতিল করে। তখন refresh #1 সফল হয়ে পুরনো refresh token বাতিল করল, আর #2, #3, #4 পুরনোটা পাঠিয়ে **ব্যর্থ** হলো → user logout! (এই backend-এ এখন rotation নেই কিনা নিশ্চিত না, কিন্তু রাখার মতো ভালো অভ্যাস না।)

**যা চাই (teacher-এর কথায়):** refresh চলার সময় যে request-গুলো আসবে, সেগুলো **কোথাও জমা রাখব**; নতুন token এলে সেই token দিয়ে সবগুলো সামলাব। আর এর জন্য একটা সাধারণ **array**-কে লাইনের (queue) মতো ব্যবহার করলেই চলবে।

---

## ধাপ ৮ — সমাধান: Queue (লাইন)

🎯 **লক্ষ্য:** Refresh একবারই; বাকিরা লাইনে দাঁড়িয়ে অপেক্ষা করবে, refresh শেষে সবাই আবার যাবে।

### 💡 ব্যাংকের উদাহরণ

ব্যাংকে সার্ভার বন্ধ, কাউন্টারে ৫ জন। একজন কর্মী সার্ভার চালু করতে গেলেন। বাকি ৪ জনকে বলা হলো "লাইনে বসুন, সার্ভার চালু হলে ডাকব"। চালু হলে সবাইকে একসাথে ডাকা হলো, সবাই নিজের কাজ করল। সার্ভার চালু না হলে সবাইকে বলা হলো "আজ হবে না"।

| ব্যাংকে | কোডে |
|---|---|
| "সার্ভার চালু করা হচ্ছে" বোর্ড | `isRefreshing = true` |
| অপেক্ষার লাইন | `pendingQueue` (array) |
| প্রত্যেকের হাতে "ডাকলে আসব" টোকেন | প্রত্যেকের promise-এর `resolve`/`reject` |
| "সবাই আসুন" | `processQueue(null)` → সবার `resolve()` |
| "আজ হবে না" | `processQueue(error)` → সবার `reject(error)` |

### 🛠️ কোড, অংশ ১: অবস্থা আর লাইন

```ts
let isRefreshing = false;   // এখন কি refresh চলছে?

let pendingQueue: {
  resolve: (value: unknown) => void;
  reject: (reason: unknown) => void;
}[] = [];                   // লাইনে অপেক্ষমাণদের resolve/reject

// লাইনের সবাইকে ফল জানানো, তারপর লাইন খালি
const processQueue = (error: unknown) => {
  pendingQueue.forEach((promise) => {
    if (error) promise.reject(error);   // refresh ব্যর্থ → সবাই ব্যর্থ
    else promise.resolve(null);         // refresh সফল → সবাই আবার চেষ্টা করো
  });
  pendingQueue = [];
};
```

💡 **এগুলো interceptor-এর বাইরে কেন:** interceptor-এর function প্রতিটা ব্যর্থ request-এর জন্য **আলাদা করে** চলে। ভেতরে `let isRefreshing` লিখলে প্রতিবার নতুন `false` হতো, কেউ কারো খবর জানত না। বাইরে (ফাইলের স্তরে) রাখলে **সব** request একই `isRefreshing` আর একই লাইন দেখে।

💡 **লাইনে কী রাখা হচ্ছে:** request নিজে না, প্রতিটা অপেক্ষমাণ promise-এর **`resolve` আর `reject` function** (২৪ নম্বর নোটের ধাপ ৩: "চাবি বাইরে রাখা")। পরে `processQueue` সেই চাবি ঘুরিয়ে সিদ্ধান্ত জানায়।

### 🛠️ কোড, অংশ ২: interceptor

```ts
axiosInstance.interceptors.response.use(
  (response) => response,
  async (error) => {
    const originalRequest = error.config as InternalAxiosRequestConfig & { _retry?: boolean };

    if (error.response?.status === 500 && error.response?.data?.message === "jwt expired" && !originalRequest._retry) {
      originalRequest._retry = true;

      // ── ক. অন্য কেউ ইতিমধ্যে refresh করছে → লাইনে দাঁড়াও ──
      if (isRefreshing) {
        return new Promise((resolve, reject) => {
          pendingQueue.push({ resolve, reject });
        }).then(() => axiosInstance(originalRequest));
      }

      // ── খ. আমিই প্রথম → refresh আমি করব ──
      isRefreshing = true;
      try {
        await axiosInstance.post("/auth/refresh-token");
        processQueue(null);                       // লাইনের সবাইকে: "যাও"
        return axiosInstance(originalRequest);    // নিজেও আবার যাই
      } catch (refreshError) {
        processQueue(refreshError);               // লাইনের সবাইকে: "হবে না"
        return Promise.reject(refreshError);
      } finally {
        isRefreshing = false;                     // সফল বা ব্যর্থ, বোর্ড নামাও
      }
    }

    return Promise.reject(error);
  },
);
```

### 💡 ধাপে ধাপে: তিনটা request একসাথে

ধরুন `/tour`, `/division`, `/user/me` তিনটাই প্রায় একসাথে "jwt expired" পেল:

```
সময় ─────────────────────────────────────────────────────────────►

/tour       ✗ → isRefreshing? না → isRefreshing = true
                → POST /auth/refresh-token ……………………… ✓
                                                          → processQueue(null)
                                                          → /tour আবার ✓
/division     ✗ → isRefreshing? হ্যাঁ → লাইনে (promise pending) ……… resolve() → /division আবার ✓
/user/me        ✗ → isRefreshing? হ্যাঁ → লাইনে (promise pending) … resolve() → /user/me আবার ✓
                                                          finally: isRefreshing = false
```

**প্রথম request (`/tour`)** — শাখা খ:
1. `isRefreshing` মিথ্যা, তাই সে-ই refresh করবে। বোর্ড তোলে: `isRefreshing = true`।
2. `await` refresh। এর মধ্যে বাকিরা আসে।

**দ্বিতীয় আর তৃতীয় (`/division`, `/user/me`)** — শাখা ক:
1. `isRefreshing` সত্য, তাই নিজে refresh করে না।
2. একটা **নতুন promise** বানায় যেটা এখনই resolve/reject হয় না, শুধু নিজের `resolve`/`reject` লাইনে (`pendingQueue`) রেখে দেয়। Promise **pending**।
3. সেই promise-এর পেছনে `.then(() => axiosInstance(originalRequest))`: "আমাকে যখন resolve করা হবে, তখন আমার request আবার পাঠাব"।
4. এই পুরো জিনিসটা `return` করে। ধাপ ৫-এর নিয়ম: caller (RTK Query) এখন এই promise-এর ফলের অপেক্ষা করে।

**Refresh সফল হলে** (আবার প্রথম request):
1. `processQueue(null)`: লাইনের প্রত্যেকের `resolve(null)` → তাদের `.then` চলে → প্রত্যেকে নিজের request আবার পাঠায় (নতুন cookie সহ) → সফল → নিজের caller-কে data।
2. নিজের request-ও আবার: `return axiosInstance(originalRequest)`।
3. `finally`: `isRefreshing = false`, বোর্ড নামানো।

**Refresh ব্যর্থ হলে** (refresh token-এরও মেয়াদ শেষ, ৩০ দিন পরে):
1. `processQueue(refreshError)`: প্রত্যেকের `reject(error)` → তাদের promise ব্যর্থ → caller-রা error পায়।
2. নিজেও `Promise.reject`।
3. সবার `/user/me` ব্যর্থ → `withAuth` আর Navbar দেখে "logged out" → login page। এটাই সঠিক: ৩০ দিন পরে সত্যিই আবার login লাগে।

### 💡 কেন `resolve(null)`, নতুন token না

অনেক online উদাহরণে দেখবেন `processQueue(null, newToken)`, আর লাইনের request-গুলো `originalRequest.headers.Authorization = "Bearer " + token` বসিয়ে আবার যায়। কারণ তারা token **header**-এ পাঠায়। আমরা token রাখি **httpOnly cookie**-তে (১৬–১৭ নম্বর নোটে এই সিদ্ধান্ত): refresh সফল মানেই browser-এ নতুন cookie, আর আবার পাঠানো প্রতিটা request নিজে থেকে নতুন cookie নিয়ে যায়। তাই শুধু "যাও" সংকেত (`resolve(null)`) যথেষ্ট, token হাতে দেওয়ার দরকার নেই। Cookie-ভিত্তিক auth-এর আরেকটা সুবিধা।

### 💡 `finally` কেন

`isRefreshing = false` যদি শুধু `try`-এর শেষে থাকত, আর refresh ব্যর্থ হতো, তাহলে বোর্ড চিরকাল উঠে থাকত। পরের সব ব্যর্থ request লাইনে দাঁড়াত, কেউ কখনো ডাকত না, সব app আটকে যেত। `finally` সফল-ব্যর্থ দুই ক্ষেত্রেই চলে (২৪ নম্বর নোটের ধাপ ৭)।

✅ **যাচাই:**
- Access token ৫ সেকেন্ড রেখে login → ৫ সেকেন্ড পরে admin dashboard reload → Network-এ কয়েকটা request 500, কিন্তু **`refresh-token` মাত্র একবার**, তারপর সব request আবার 200।
- User logout হয় না, কোনো error toast নেই।

---

# কী বাকি: সমস্যা চেনা

আপনার project-এর `lib/axios.ts`-এ ধাপ ৮-এর queue বসানো আছে ✅। পড়ে কয়েকটা জায়গা পেলাম:

| # | সমস্যা | ফল |
|---|---|---|
| ১ | **Refresh request নিজেও এই interceptor দিয়ে যায়।** Refresh token-এর মেয়াদ শেষ হলে `/auth/refresh-token` নিজেও "jwt expired" (500) দেয়। তখন interceptor সেটাকে সাধারণ ব্যর্থ request ভাবে: `isRefreshing` সত্য (আমরা তো refresh-এর ভেতরেই), তাই refresh request-কে **লাইনে দাঁড় করায়**। অথচ লাইন খালি করবে যে, সে ঐ refresh request-এরই উত্তরের অপেক্ষায়! | **Deadlock**: দুজন একে অপরের অপেক্ষায়, কেউ শেষ হয় না। ৩০ দিন পরে (বা পরীক্ষায় refresh-কেও ছোট মেয়াদ দিলে) page চিরকাল loading-এ আটকে থাকে, login page-এও যায় না। **সবচেয়ে জরুরি** |
| ২ | `error.response.status`: internet নেই বা server বন্ধ থাকলে `error.response` থাকেই না (`undefined`) | Interceptor নিজেই `TypeError: Cannot read properties of undefined` দিয়ে ভাঙে; আসল "Network Error"-এর বদলে বিভ্রান্তিকর error |
| ৩ | Type `AxiosRequestConfig & { _retry: boolean }` | Interceptor-এ axios দেয় `InternalAxiosRequestConfig` (headers সবসময় থাকে); আর `_retry` প্রথমে থাকে না, তাই `?` হওয়া উচিত |
| ৪ | `.catch((error) => Promise.reject(error))` লাইনে দাঁড়ানো promise-এ | কিছুই করে না: error ধরে আবার একই error ছোড়া। না লিখলেও error নিজে থেকে সামনে যায় |
| ৫ | Request interceptor-এ `console.log("Axios", config)`, আর `console.log("New Token arrived", res)` | Login-এর password console-এ ছাপা হয় (১৩ নম্বর নোট); production-এ অকারণ log |
| ৬ | Backend-এ মেয়াদোত্তীর্ণ token-এ 500 | ধাপ ৩-এর সংশোধন |
| ৭ | `JWT_ACCESS_EXPIRES=5s` | পরীক্ষা শেষে ফেরত না দিলে live site-এ প্রতি ৫ সেকেন্ডে refresh |

---

# উন্নতি

## উন্নতি ১ — Refresh request-কে interceptor-এর নিয়মের বাইরে রাখা (Deadlock বন্ধ)

```ts
const REFRESH_URL = "/auth/refresh-token";

const isRefreshCall = originalRequest.url?.includes(REFRESH_URL);

if (isTokenExpired && !originalRequest._retry && !isRefreshCall) {
  // ...queue আর refresh
}
```

Refresh request নিজে ব্যর্থ হলে সে আর লাইনে দাঁড়ায় না, সরাসরি `Promise.reject` → `await axiosInstance.post(REFRESH_URL)`-এর `catch` চলে → `processQueue(error)` → সবাই ব্যর্থ → login page। Deadlock নেই।

> ⚠️ **পরীক্ষা করে দেখুন:** backend-এ `JWT_REFRESH_EXPIRES=20s`, `JWT_ACCESS_EXPIRES=5s` দিয়ে login, ২০ সেকেন্ডের বেশি অপেক্ষা, তারপর dashboard reload। উন্নতি ১ ছাড়া page আটকে থাকে; উন্নতি ১-এর পরে login page-এ যায়। পরীক্ষা শেষে দুটোই আসল মানে ফেরত।

## উন্নতি ২ — নিরাপদে পড়া (`?.`) আর সঠিক type

```ts
import axios, { type InternalAxiosRequestConfig } from "axios";

const originalRequest = error.config as InternalAxiosRequestConfig & { _retry?: boolean };

const isTokenExpired =
  error.response?.status === 500 && error.response?.data?.message === "jwt expired";
```

`?.`: response না থাকলে (network error) পুরো শর্ত মিথ্যা, interceptor না ভেঙে শেষের `Promise.reject(error)`-এ আসল "Network Error" সামনে যায়। আর `error.config` না থাকলেও (খুব বিরল) ভাঙবে না, কারণ শর্তে পৌঁছানোর আগেই মিথ্যা।

## উন্নতি ৩ — Backend: মেয়াদোত্তীর্ণ token-এ 401

```ts
// backend: checkAuth.ts (উদাহরণ — আপনার কোডের গঠন অনুযায়ী)
import jwt from "jsonwebtoken";

try {
  const verifiedToken = jwt.verify(accessToken, envVars.JWT_ACCESS_SECRET);
  // ...
} catch (error) {
  if (error instanceof jwt.TokenExpiredError) {
    throw new AppError(401, "jwt expired");   // মেয়াদ শেষ: 401, message একই রাখা
  }
  throw new AppError(401, "Invalid token");
}
```

Backend 401 দিলে frontend-এর শর্ত:

```ts
const isTokenExpired =
  (error.response?.status === 401 || error.response?.status === 500) &&
  error.response?.data?.message === "jwt expired";
```

(দুটোই রাখা হলো যাতে backend বদলানোর আগে-পরে দুই অবস্থাতেই চলে; backend বদলে গেলে `500` বাদ দেবেন।) Message একই রাখায় পুরনো frontend-ও ভাঙে না।

## উন্নতি ৪ — `console.log` সরানো, অপ্রয়োজনীয় `.catch` বাদ

চূড়ান্ত কোডে।

## উন্নতি ৫ — পরীক্ষার পরে `.env` ফেরত

```env
JWT_ACCESS_EXPIRES=1d
JWT_REFRESH_EXPIRES=30d
```

```sh
pm2 reload 0 --update-env
```

---

# চূড়ান্ত কোড

### `src/lib/axios.ts`

```ts
import config from "@/config";
import axios, { type InternalAxiosRequestConfig } from "axios";

export const axiosInstance = axios.create({
  baseURL: config.baseUrl,
  withCredentials: true, // cookie (access + refresh token) সব request-এ যায়
});

/* ───────── Request interceptor ───────── */

axiosInstance.interceptors.request.use(
  (requestConfig) => requestConfig,
  (error) => Promise.reject(error),
);

/* ───────── Token refresh: অবস্থা আর লাইন ───────── */

const REFRESH_URL = "/auth/refresh-token";

let isRefreshing = false; // এখন কি refresh চলছে?

// refresh চলার সময় ব্যর্থ হওয়া request-গুলোর resolve/reject (২৪ নম্বর নোট)
let pendingQueue: {
  resolve: (value: unknown) => void;
  reject: (reason: unknown) => void;
}[] = [];

// লাইনের সবাইকে ফল জানাও, তারপর লাইন খালি
const processQueue = (error: unknown) => {
  pendingQueue.forEach((promise) => {
    if (error) promise.reject(error);
    else promise.resolve(null); // cookie ইতিমধ্যে নতুন, শুধু "আবার যাও" সংকেত
  });
  pendingQueue = [];
};

/* ───────── Response interceptor ───────── */

axiosInstance.interceptors.response.use(
  (response) => response,
  async (error) => {
    const originalRequest = error.config as InternalAxiosRequestConfig & { _retry?: boolean };

    // backend এখন 500 দেয়; 401-এ বদলালেও চলবে (উন্নতি ৩)
    const isTokenExpired =
      (error.response?.status === 401 || error.response?.status === 500) &&
      error.response?.data?.message === "jwt expired";

    // refresh request নিজে ব্যর্থ হলে লাইনে দাঁড়াবে না (deadlock বন্ধ)
    const isRefreshCall = originalRequest?.url?.includes(REFRESH_URL);

    if (isTokenExpired && !originalRequest._retry && !isRefreshCall) {
      originalRequest._retry = true; // প্রতিটা request সর্বোচ্চ একবার আবার চেষ্টা

      // ক. অন্য কেউ refresh করছে → লাইনে দাঁড়াও, শেষ হলে আবার যাও
      if (isRefreshing) {
        return new Promise((resolve, reject) => {
          pendingQueue.push({ resolve, reject });
        }).then(() => axiosInstance(originalRequest));
      }

      // খ. আমিই প্রথম → refresh আমি করব
      isRefreshing = true;
      try {
        await axiosInstance.post(REFRESH_URL); // backend নতুন accessToken cookie বসায়
        processQueue(null);
        return axiosInstance(originalRequest);
      } catch (refreshError) {
        processQueue(refreshError); // refresh token-ও শেষ → সবাই ব্যর্থ → login
        return Promise.reject(refreshError);
      } finally {
        isRefreshing = false;
      }
    }

    // বাকি সব error যেমন আছে তেমন সামনে
    return Promise.reject(error);
  },
);
```

---

# সারসংক্ষেপ

| ধারণা | এক লাইনে |
|---|---|
| Interceptor-এর নিয়ম | সবসময় `return response` বা `return Promise.reject(error)`; খালি রাখলে error গিলে ফেলে |
| `error.config` | ব্যর্থ request-এর রেসিপি; `axiosInstance(config)` দিয়ে আবার পাঠানো যায় |
| `return axiosInstance(originalRequest)` | Caller নতুন request-এর ফল পায়, মাঝখানের ব্যর্থতা জানেই না |
| `_retry` | প্রতিটা request একবারই আবার; অসীম loop বন্ধ |
| `isRefreshing` | একসাথে একটাই refresh |
| `pendingQueue` | অপেক্ষমাণদের `resolve`/`reject`, refresh শেষে একসাথে ডাকা |
| `finally` | সফল-ব্যর্থ যা-ই হোক, বোর্ড নামাও |
| Refresh call বাদ | Refresh নিজে লাইনে দাঁড়ালে deadlock |
| Cookie-র সুবিধা | নতুন token হাতে দিতে হয় না, browser নিজে পাঠায় |

---

# Check-list

- [ ] `JWT_ACCESS_EXPIRES=5s` দিয়ে login, ৫ সেকেন্ড পরে কাজ করলে logout হয় না
- [ ] Network-এ: ব্যর্থ request → `refresh-token` (একবার) → সব আবার সফল
- [ ] Dashboard-এ একসাথে অনেক request ব্যর্থ হলেও `refresh-token` **মাত্র একবার**
- [ ] Refresh token-ও মেয়াদোত্তীর্ণ হলে page আটকে থাকে না, login page-এ যায় (উন্নতি ১)
- [ ] Backend বন্ধ রেখে কিছু করলে "Network Error" আসে, interceptor-এর `TypeError` না (উন্নতি ২)
- [ ] ভুল password-এ login করলে আগের মতোই error toast (অন্য error-এ interceptor হাত দেয় না)
- [ ] Console-এ password বা token-এর log নেই
- [ ] পরীক্ষা শেষে `.env`-এ আসল মেয়াদ ফেরত, `pm2 reload 0 --update-env`
