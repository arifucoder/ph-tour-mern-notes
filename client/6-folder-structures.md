# Frontend Notes — ০৬: Folder Structure ও Layout

> আগের নোটে route বোঝার জন্য যে `AdminLayout` বানিয়েছিলাম, সেটা মুছে দিয়েছি।

এই নোটে teacher যে folder structure follow করেন সেটা লেখা হয়েছে। তাঁর কথায় — এটাই "সেরা" structure এমন না, কিন্তু এতে তাঁর কোড maintain করতে সুবিধা হয়। শেষে বড় প্রজেক্টের জন্য একটা বিকল্প structure-এর পরামর্শও দেওয়া আছে।

---

## ১. একটা Page কীভাবে ভাগ হয় — Layout, Page, Module, UI

ধরা যাক `/about` route। এর page ফাইল `About.tsx`। স্ক্রিনে যা দেখা যায়, সেটাকে কয়েকটা স্তরে ভাগ করা যায়:

```
/about  →  Page = About.tsx
┌──────────────────────────────────────────┐
│  Navbar                                   │ ← Layout
├──────────────────────────────────────────┤
│  ┌────────────────────────────────────┐  │
│  │  Our Mission section        [btn]  │  │ ← Module   ([btn] = UI)
│  └────────────────────────────────────┘  │
│                                           │
│  ┌────────┐  ┌────────┐  ┌────────┐      │
│  │        │  │        │  │        │      │ ← Module (যেমন Our Team)
│  └────────┘  └────────┘  └────────┘      │
├──────────────────────────────────────────┤
│  Footer                                   │ ← Layout
└──────────────────────────────────────────┘
```

| স্তর | কী | উদাহরণ | কোথায় থাকে |
|---|---|---|---|
| **Layout** | অনেক page-এ একই থাকে এমন কাঠামো | Navbar, Footer, Sidebar | `components/layout/` |
| **Page** | একটা route-এ যা দেখায় — module-গুলো সাজিয়ে বানানো | `About.tsx` | `pages/` |
| **Module** | Page-এর এক একটা বড় অংশ/section | Our Mission, Our Team, Hero | `components/modules/` |
| **UI** | সবচেয়ে ছোট, বারবার ব্যবহারযোগ্য অংশ | Button, Input, Card | `components/ui/` |

অর্থাৎ: **UI দিয়ে Module বানাই → Module দিয়ে Page বানাই → Page বসে Layout-এর ভেতরে।**

---

## ২. Teacher-এর Folder Structure

```
src/
├── assets/
│   ├── images/
│   └── icons/
├── components/
│   ├── ui/
│   ├── layout/
│   └── modules/
│       ├── homepage/
│       └── about/
├── constants/
├── hooks/
├── lib/
├── pages/
├── redux/
├── routes/
├── types/
└── utils/
```

### `src/assets/`
- `images/` — প্রজেক্টে ব্যবহৃত static image (logo, banner ইত্যাদি)।
- `icons/` — SVG icon।

> নোট: Vite প্রজেক্টে `src/` এর বাইরে একটা **`public/`** folder-ও থাকে। পার্থক্য:
> - **`src/assets/`** — কোডে `import` করে ব্যবহার করা হয়। Build-এর সময় Vite এগুলো optimize করে আর ফাইলের নামে hash যোগ করে (cache-এর জন্য ভালো)।
> - **`public/`** — import না করে সরাসরি fixed URL দিয়ে ব্যবহার হয় (যেমন `/favicon.ico`, `robots.txt`)। এগুলো হুবহু কপি হয়।

### `src/components/`

#### `ui/` — atomic অংশ
- একক, ছোট জিনিস: **Button, Input, Badge, Dialog** ইত্যাদি।
- সাধারণত **shadcn** থেকেই এগুলো আসে (`shadcn add button` করলে এখানে আসে)।
- এখানে খুব বেশি কোড লেখা হয় না — শুধু **atomic level**-এর অংশ।
- নিজেদের ছোট reusable wrapper (যেমন একটা `Container` component যা content-কে মাঝখানে নির্দিষ্ট width-এ রাখে) এখানে রাখা যায়।

#### `layout/` — app-এর কাঠামো

> ⚠️ সংশোধন: নোটে লেখা ছিল "layout হলো individual page-এর structure, যেমন home page-এর structure"। আসলে উল্টো — **layout হলো অনেক page মিলে যে কাঠামো share করে**, যেমন header, footer আর মাঝখানে content। একটা নির্দিষ্ট page-এর ভেতরের গঠন (Hero, Features ইত্যাদি) layout না, সেগুলো **module**।

এখানে থাকবে:
- Layout component — `CommonLayout.tsx`, `DashboardLayout.tsx`
- Layout-এর অংশ — `Navbar.tsx`, `Footer.tsx`, `Sidebar.tsx`

#### `modules/` — page-এর section
প্রতিটা page-এর জন্য একটা folder, তার ভেতরে সেই page-এর section-গুলো:

```
components/modules/
├── homepage/
│   ├── HeroSection.tsx
│   └── FeaturesSection.tsx
└── about/
    ├── OurMission.tsx
    └── OurTeam.tsx
```

Page ফাইল তখন শুধু module-গুলো সাজায়, তাই ছোট আর পরিষ্কার থাকে:

```tsx
// src/pages/Homepage.tsx
import HeroSection from "@/components/modules/homepage/HeroSection";
import FeaturesSection from "@/components/modules/homepage/FeaturesSection";

export default function Homepage() {
  return (
    <>
      <HeroSection />
      <FeaturesSection />
    </>
  );
}
```

### `src/lib/` — third-party connection
Third-party কোনো library বা service-এর সাথে connect করার setup ফাইল। Teacher `lib`-কে "library" হিসেবে ধরেন।

উদাহরণ:
- `firebase.ts` — Firebase initialize করা
- `axios.ts` — base URL, `withCredentials` সহ configured axios instance
- `stripe.ts` — payment SDK setup

> নোট: নোটে Firebase-এর জন্য "driver লাগে" লেখা ছিল — frontend-এ একে সাধারণত **SDK** বা **client** বলা হয়। `lib`-এ সেই SDK একবার configure করে export করা হয়, বাকি app সেখান থেকে import করে।
>
> নোট: shadcn নিজে `lib/utils.ts`-এ `cn()` রেখেছে, যেটা আসলে utility function। এটা shadcn-এর নিয়ম, তাই সেখানেই থাকুক — সরালে `components.json`-এর alias বদলাতে হবে।

### `src/utils/` — ছোট helper function
এখানে ছোট ছোট function থাকে, যার প্রতিটা **শুধু একটা কাজ** করে। সাধারণত এগুলো **pure function** — একই input দিলে একই output, বাইরের কিছু বদলায় না।

উদাহরণ:

```ts
// src/utils/formatTaka.ts
export const formatTaka = (amount: number) =>
  new Intl.NumberFormat("bn-BD", { style: "currency", currency: "BDT" }).format(amount);
// formatTaka(1500) → "১,৫০০.০০৳"

// src/utils/truncateText.ts
export const truncateText = (text: string, max: number) =>
  text.length > max ? text.slice(0, max) + "..." : text;

// src/utils/getInitials.ts
export const getInitials = (name: string) =>
  name.split(" ").map((n) => n[0]).join("").toUpperCase();
// getInitials("Rahim Uddin") → "RU"
```

আরও উদাহরণ: `formatDate`, `slugify` (`"Hello World"` → `"hello-world"`), `generateRoutes` (route config থেকে route list বানানো)।

> নোট: নোটে `generateRoutes.tsx` লেখা ছিল। ফাইলে JSX না থাকলে `.ts` দিলেই হবে — utils-এর ফাইল সাধারণত `.ts`।

**`lib` বনাম `utils` সংক্ষেপে:** `lib` = বাইরের library/service-এর setup; `utils` = নিজেদের লেখা ছোট helper।

### `src/routes/`
Admin, user আর public route-গুলোর জন্য **আলাদা আলাদা ফাইল**:

```
routes/
├── index.tsx          ← createBrowserRouter — সব route এখানে জোড়া লাগে
├── publicRoutes.tsx
├── userRoutes.tsx
└── adminRoutes.tsx
```

ছোট প্রজেক্টে route কম থাকলে একটা ফাইলই যথেষ্ট। কিন্তু প্রজেক্ট যত বড় হবে, route-এর সংখ্যা তত বাড়বে — তখন আলাদা ফাইলে রাখলে খুঁজে পাওয়া আর maintain করা সহজ হয়।

### `src/pages/`
প্রতিটা route-এর page component — `Homepage.tsx`, `About.tsx`, `Login.tsx`, `NotFound.tsx`।

### `src/hooks/`
নিজেদের লেখা **custom hook** — যেমন `useDebounce.ts`, `useMediaQuery.ts`। (Custom hook-এর নাম `use` দিয়ে শুরু হতে হয়।)

### `src/redux/`
Redux-এর সবকিছু:

```
redux/
├── store.ts          ← store তৈরি
├── hooks.ts          ← typed useAppDispatch / useAppSelector
├── baseApi.ts        ← RTK Query-র base API
└── features/
    └── auth/
        ├── authSlice.ts
        └── authApi.ts
```

### `src/types/`
একাধিক জায়গায় ব্যবহৃত TypeScript **type/interface** — যেমন `IUser`, `IProduct`, API response-এর type।

> টিপ: শুধু একটা component-এ ব্যবহৃত type সেই component ফাইলেই রাখা ভালো। `types/`-এ রাখুন শুধু shared type।

### `src/constants/`
যে মান বদলায় না এবং অনেক জায়গায় লাগে:

```ts
// src/constants/roles.ts
export const ROLES = { ADMIN: "ADMIN", USER: "USER" } as const;
```

উদাহরণ: user role, navbar-এর menu item list, pagination-এর default limit।

> নোট: API URL বা secret key `constants`-এ না রেখে **`.env`** ফাইলে রাখুন। Vite-এ env variable-এর নাম `VITE_` দিয়ে শুরু করতে হয় (যেমন `VITE_API_URL`) এবং কোডে `import.meta.env.VITE_API_URL` দিয়ে পড়তে হয়।

---

## ৩. পরামর্শ: বড় প্রজেক্টের জন্য Feature-based Structure

Teacher-এর structure **type অনুযায়ী** ভাগ করা (সব hook এক জায়গায়, সব type এক জায়গায়, সব redux এক জায়গায়)। ছোট-মাঝারি প্রজেক্টে এটা খুব ভালো কাজ করে।

কিন্তু প্রজেক্ট অনেক বড় হলে একটা feature-এর (যেমন auth) কোড ৬-৭টা folder-এ ছড়িয়ে যায় — `pages/`, `components/modules/`, `redux/`, `types/`, `hooks/`… একটা feature-এ কাজ করতে গিয়ে বারবার folder বদলাতে হয়।

এর সমাধান হিসেবে অনেক team **feature-based** structure ব্যবহার করে — **একটা feature-এর সবকিছু এক folder-এ**:

```
src/
├── app/                 ← router, store, providers — app চালু করার setup
├── assets/
├── components/          ← শুধু সব feature-এ shared জিনিস
│   ├── ui/
│   └── layout/
├── features/
│   ├── auth/
│   │   ├── components/   ← LoginForm.tsx, RegisterForm.tsx
│   │   ├── api/          ← authApi.ts (RTK Query endpoint)
│   │   ├── hooks/        ← useAuth.ts
│   │   ├── types.ts
│   │   └── authSlice.ts
│   ├── products/
│   │   ├── components/
│   │   ├── api/
│   │   └── types.ts
│   └── cart/
├── pages/               ← শুধু feature-এর component সাজায়
├── hooks/               ← shared hook
├── lib/
├── utils/
├── types/               ← shared type
└── constants/
```

**মূল নিয়ম:**
- কোনো কিছু **একটা feature-এ** ব্যবহার হলে → সেই feature-এর folder-এ।
- **একাধিক feature-এ** লাগলে → উপরের shared folder-এ (`components/`, `hooks/`, `utils/`)।
- Feature-কে মুছতে হলে শুধু তার folder মুছলেই প্রায় সব চলে যায়।

| | Teacher-এর structure (type-based) | Feature-based |
|---|---|---|
| কোথায় ভালো | ছোট-মাঝারি প্রজেক্ট, শেখার সময় | বড় প্রজেক্ট, অনেক developer |
| সুবিধা | সহজ, কোথায় কী আছে দ্রুত বোঝা যায় | এক feature-এর কোড এক জায়গায় |
| অসুবিধা | বড় হলে feature-এর কোড ছড়িয়ে যায় | শুরুতে একটু বেশি folder, কোনটা shared ঠিক করতে হয় |

> **পরামর্শ:** এই প্রজেক্টে teacher-এর structure-ই follow করুন — class-এর সাথে মিলিয়ে চলা সহজ হবে, আর এই আকারের প্রজেক্টের জন্য এটা যথেষ্ট। Teacher-এর `components/modules/<page>/` ধারণাটা আসলে feature-based চিন্তার কাছাকাছি, তাই পরে বড় প্রজেক্টে feature-based-এ যাওয়া কঠিন হবে না।