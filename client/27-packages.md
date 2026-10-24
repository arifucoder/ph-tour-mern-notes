# Frontend Notes — ২৭: Client-এর Packages — কোনটা কী কাজে, কোথায় লাগে

`package.json`-এর প্রতিটা package কেন আছে, project-এর কোথায় ব্যবহার হচ্ছে, আর নতুন project-এ কোন command দিয়ে বসাতে হয়। প্রতিটা package-এর "কোথায়" কলামটা আপনার `src` folder খুঁজে লেখা।

> 🔗 **Repo:** [arifucoder/phovia-tour-client-v0](https://github.com/arifucoder/phovia-tour-client-v0)

> নোট: সব command `bun` দিয়ে। Version-গুলো আপনার `package.json` থেকে নেওয়া, পুরনো কিছু দিয়ে বদলানো হয়নি।

---

## ১. `scripts`

| Script | Command | কাজ |
|---|---|---|
| `dev` | `vite` | Development server চালু (`bun run dev`), কোড বদলালে সাথে সাথে browser-এ দেখায় |
| `build` | `tsc -b && vite build` | আগে TypeScript-এর ভুল পরীক্ষা (`tsc -b`), ভুল না থাকলে production build (`dist` folder)। `&&` মানে প্রথমটা সফল হলেই দ্বিতীয়টা চলবে, তাই type error থাকলে build হবেই না |
| `lint` | `oxlint` | কোডের সম্ভাব্য ভুল আর খারাপ অভ্যাস খোঁজা |
| `preview` | `vite preview` | তৈরি হওয়া `dist` local-এ চালিয়ে দেখা, deploy-এর আগে শেষ পরীক্ষা |

---

## ২. `dependencies` বনাম `devDependencies`

| | কখন লাগে | উদাহরণ |
|---|---|---|
| **dependencies** | App চলার সময় browser-এ যে code যায় | `react`, `axios`, `zod` |
| **devDependencies** | শুধু development আর build-এর সময়, browser-এ যায় না | `typescript`, `vite`, `@types/*`, `oxlint` |

```shell
bun add axios          # dependencies-এ
bun add -d typescript  # devDependencies-এ (-d = dev)
```

> 💡 Vite project-এ এই ভাগটা মূলত সাজানোর জন্য: build-এর সময় Vite শুধু সেই code-ই নেয় যা আসলে `import` করা হয়েছে, তালিকা যেখানেই থাকুক।

---

## ৩. Core: React আর build tool

| Package | কাজ | কোথায় |
|---|---|---|
| `react` | UI library: component, hook (`useState`, `useEffect` …) | প্রায় সব file |
| `react-dom` | React component-কে browser-এর DOM-এ বসানো (`createRoot`) | `main.tsx` |
| `vite` *(dev)* | Dev server আর production build | `vite.config.ts` |
| `@vitejs/plugin-react` *(dev)* | Vite-কে JSX/TSX আর React Fast Refresh বোঝানো | `vite.config.ts` |
| `typescript` *(dev)* | Type পরীক্ষা (`tsc -b`) | `build` script |
| `@types/react`, `@types/react-dom` *(dev)* | React-এর TypeScript type | Editor আর `tsc` নিজে ব্যবহার করে |
| `@types/node` *(dev)* | Node-এর type, `vite.config.ts`-এ `path`, `__dirname`-এর জন্য (`@` alias) | `vite.config.ts` |
| `oxlint` *(dev)* | Linter (ESLint-এর চেয়ে অনেক দ্রুত, Rust-এ লেখা) | `lint` script |

---

## ৪. Routing

| Package | কাজ | কোথায় |
|---|---|---|
| `react-router` | Page-এর route (`createBrowserRouter`), `Link`, `Navigate`, `useNavigate`, `useParams`, `useSearchParams` | `routes/`, page আর component (০৫ নম্বর নোট) |

> 💡 React Router v7 থেকে আলাদা `react-router-dom` লাগে না, সব `react-router`-এর ভেতরে।

---

## ৫. State আর data fetching

| Package | কাজ | কোথায় |
|---|---|---|
| `@reduxjs/toolkit` | Redux store (`configureStore`) আর **RTK Query** (`createApi`, cache, tag) | `redux/store.ts`, `baseApi`, সব `*.api.ts` (১৩ নম্বর নোট) |
| `react-redux` | React-কে store-এর সাথে যুক্ত করা (`Provider`, `useSelector`, `useDispatch`) | `main.tsx`, `redux/hook.ts` |
| `axios` | HTTP request; `withCredentials` দিয়ে cookie, interceptor দিয়ে token refresh | `lib/axios.ts`, `axiosBaseQuery` (২৫ নম্বর নোট) |

---

## ৬. Form আর validation

| Package | কাজ | কোথায় |
|---|---|---|
| `react-hook-form` | Form-এর মান, submit, error (`useForm`, `Controller`) | Login, Register, Tour, Division form (১২ নম্বর নোট) |
| `zod` | Schema দিয়ে data যাচাই (`z.object`, `z.email()`), schema থেকে type (`z.infer`) | সব form-এর schema |
| `@hookform/resolvers` | Zod schema-কে react-hook-form-এর সাথে জোড়া (`zodResolver`) | `useForm({ resolver: zodResolver(schema) })` |

---

## ৭. UI: shadcn আর তার সাথের packages

shadcn কোনো সাধারণ component library না: `bunx --bun shadcn@latest add button` দিলে component-এর **code** আপনার `src/components/ui/`-এ কপি হয়। তারপর সেই code যে package-গুলো `import` করে, সেগুলো `package.json`-এ বসে।

| Package | কাজ | কোথায় |
|---|---|---|
| `shadcn` | CLI (component add করা) + `shadcn/tailwind.css` (base style) | `index.css`-এ `@import "shadcn/tailwind.css"`, তাই dependencies-এ থাকা দরকার |
| `radix-ui` | Accessible headless primitive: Dialog, Popover, Select, Slot (`asChild`) … সব একটা package-এ | `components/ui/*` (১০ নম্বর নোট) |
| `class-variance-authority` | Component-এর variant (`variant: "destructive"`, `size: "sm"`) সাজানো | `button.tsx`, `badge.tsx`, `field.tsx` … |
| `cn` | Tailwind class জোড়া আর conflict মেটানো (`px-2` + `px-4` → `px-4`) | `lib/utils.ts` (`export { cn } from "cn"`) আর ২৪টা ui file |
| `lucide-react` | Icon | ২৭টা file |
| `sonner` | Toast notification (`toast.success`, `toast.error`) | `main.tsx`-এ `<Toaster>`, form-গুলো |
| `next-themes` | Theme (light/dark) জানা | শুধু `ui/sonner.tsx`: toast-এর রং theme অনুযায়ী |
| `cmdk` | খোঁজা যায় এমন command menu | `ui/command.tsx` → `searchable-select.tsx` → `AddTour.tsx` |
| `input-otp` | OTP-এর আলাদা আলাদা ঘর | `ui/input-otp.tsx` → `pages/verify.tsx` |
| `recharts` | Chart | `ui/chart.tsx` (এখনো কোনো page-এ ব্যবহার হয়নি, dashboard-এর chart-এর জন্য রাখা) |
| `tw-animate-css` | shadcn-এর animation class (`animate-in`, `fade-in` …) | `index.css` |

> 💡 **`cn` package:** আগে shadcn `clsx` + `tailwind-merge` দুই package মিলিয়ে `lib/utils.ts`-এ নিজের `cn` বানাত। নতুন shadcn-এ এটা আলাদা একটা ছোট, দ্রুত package `cn` (shadcn-এরই বানানো, "drop-in replacement for clsx + tailwind-merge")। আপনার `utils.ts` এটাই re-export করে, এটা নতুন আর সঠিক পদ্ধতি।

> 💡 **`recharts` version-এ `^` নেই (`3.8.0`):** shadcn-এর chart component নির্দিষ্ট version-এ পরীক্ষিত, তাই shadcn নিজেই version আটকে দেয়। `bun update` দিলেও এটা বদলাবে না; এভাবেই রাখুন।

---

## ৮. Style আর font

| Package | কাজ | কোথায় |
|---|---|---|
| `tailwindcss` | Tailwind v4 | `index.css`-এ `@import "tailwindcss"` |
| `@tailwindcss/vite` | Tailwind-কে Vite-এর plugin হিসেবে চালানো (v4-এ `postcss.config` লাগে না) | `vite.config.ts` |
| `@fontsource/poppins` | Poppins font, নিজের server থেকে (Google Fonts-এ request না) | `index.css`, 400–800 weight |
| `@fontsource/hind-siliguri` | বাংলা font | `index.css`, 500 আর 600 weight (০৭ নম্বর নোট) |

> 💡 প্রতিটা weight আলাদা `@import`, যেটা দরকার শুধু সেটাই। অপ্রয়োজনীয় weight বাদ দিলে site দ্রুত হয় (১৫ নম্বর নোট)।

---

## ৯. ⚠️ যে package-গুলো ভুল করে বসে গেছে

`src` খুঁজে দেখলাম, এগুলো **কোথাও import হয় না**:

| Package | কেন এসেছে | আসলে কী |
|---|---|---|
| `add` | `bun add add …` বা `bunx shadcn add`-এর বদলে `bun add`-এ লেখা | npm-এর একটা অন্য package |
| `button` | `bun add button`, অথচ লেখার কথা ছিল `bunx --bun shadcn@latest add button` | npm-এর অন্য, অপ্রাসঙ্গিক package |
| `input` | একই ভুল | একই |
| `field` | একই ভুল | একই |
| `command` | একই ভুল (আসল কাজ করছে `cmdk`) | একই |
| `popover` | একই ভুল (আসল কাজ করছে `radix-ui`) | একই |

> ⚠️ **সংশোধন:** `src`-এ যেখানে `"button"`, `"input"` লেখা দেখা যায়, সেগুলো `@/components/ui/button`-এর মতো নিজের file, npm-এর package না। shadcn component সবসময় `bunx --bun shadcn@latest add <নাম>` দিয়ে আনতে হয়; `bun add <নাম>` দিলে npm থেকে ঐ নামের অন্য কারো package চলে আসে।

🛠️ **মুছে ফেলুন:**

```shell
bun remove add button input field command popover
```

✅ **যাচাই:** `bun run build` সফল, `bun run dev`-এ সব page আগের মতো চলে।

---

## ১০. নতুন project-এ একবারে বসানো

```shell
# React, routing, state, request
bun add react-router @reduxjs/toolkit react-redux axios

# Form
bun add react-hook-form zod @hookform/resolvers

# Tailwind আর font
bun add tailwindcss @tailwindcss/vite
bun add @fontsource/poppins @fontsource/hind-siliguri

# Dev
bun add -d @types/node

# shadcn (init দিলে radix-ui, cn, class-variance-authority, lucide-react, tw-animate-css নিজে বসে)
bunx --bun shadcn@latest init
bunx --bun shadcn@latest add button input field sonner command popover input-otp chart
```

> 💡 `react`, `react-dom`, `vite`, `typescript`, `@vitejs/plugin-react`, `@types/react*` আসে Vite-এর template থেকে (০৪ নম্বর নোট)। `next-themes`, `cmdk`, `input-otp`, `recharts` আলাদা install করতে হয় না: যে shadcn component এগুলো লাগায়, সেটা add করলে নিজে বসে।

---

# Check-list

- [ ] প্রতিটা package কোন কাজে, জানি
- [ ] shadcn component আনি `bunx --bun shadcn@latest add`, `bun add` না
- [ ] `bun remove add button input field command popover` দিয়েছি
- [ ] তারপর `bun run build` সফল
- [ ] `recharts`-এর version আটকানো রেখেছি
