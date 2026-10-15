# Frontend Notes — ০৪: React + TypeScript Project Scaffold

এই নোটে Vite দিয়ে একটা React + TypeScript প্রজেক্ট তৈরি করে তাতে **React Router**, **React Hook Form**, **Tailwind CSS** আর **shadcn/ui** setup করার ধাপগুলো আছে।

> আমরা package manager হিসেবে **bun** ব্যবহার করছি। নিচে কোথাও pnpm-এর command থাকলে পাশে bun-এর সমতুল্য command দেওয়া আছে।

---

## ১. Vite প্রজেক্ট তৈরি

প্রজেক্ট কয়েকভাবে শুরু করা যায়।

### উপায় ১: আগে folder বানিয়ে, তার ভেতরে প্রজেক্ট তৈরি

Folder বানিয়ে সেই folder-এ terminal খুলে command চালাতে হয়। প্রজেক্টের নামের জায়গায় শুধু **`.`** দিলে বর্তমান folder-এই প্রজেক্ট তৈরি হয়।

```sh
# pnpm
pnpm create vite . --template react-ts

# bun
bun create vite . --template react-ts
```

### উপায় ২: নাম দিয়ে নতুন folder সহ প্রজেক্ট তৈরি

```sh
# pnpm
pnpm create vite project-name --template react-ts

# bun
bun create vite project-name --template react-ts
```

এতে `project-name` নামে নতুন একটা folder তৈরি হবে এবং প্রজেক্ট তার ভেতরে থাকবে।

> ⚠️ সংশোধন: এই উপায়ে terminal খুলতে হবে **যে folder-এর ভেতরে** নতুন প্রজেক্ট folder-টা তৈরি করতে চান সেখানে (parent folder), প্রজেক্ট folder-এর ভেতরে না। কারণ command নিজেই `project-name` folder বানাবে।

### উপায় ৩: Interactive prompt দিয়ে

```sh
bun create vite
```

এরপর prompt-গুলো আসবে:
1. Project name দিন (বর্তমান folder-এ করতে চাইলে `.`)
2. Framework → **React**
3. Variant → **TypeScript**
4. Linter → **Oxlint** (ঐচ্ছিক — নিচের নোট দেখুন)
5. Install and run → হ্যাঁ

> নোট — **Oxlint:** Rust-এ লেখা একটা linter (Oxc প্রজেক্টের অংশ), ESLint-এর চেয়ে অনেক গুণ দ্রুত। তবে ESLint-এর সব rule আর plugin এখনো এতে নেই। যেকোনোটা বেছে নেওয়া যায়।
>
> Create Vite-এর prompt-গুলো version অনুযায়ী একটু বদলাতে পারে।

Install না করে থাকলে:

```sh
bun install
bun run dev
```

---

## ২. React Router যোগ করা

আমরা React Router-এর **Data mode** ব্যবহার করব।
Docs: https://reactrouter.com/start/data/installation

```sh
bun add react-router
```

> নোট: React Router **v7**-এ আলাদা `react-router-dom` package লাগে না — সবকিছু `react-router` package-এই আছে। নোটে "react router dom add করব" লেখা ছিল, কিন্তু command-এ সঠিকভাবেই `react-router` install করা হয়েছে।
>
> **Data mode** মানে `createBrowserRouter` দিয়ে route define করা এবং `RouterProvider` দিয়ে render করা। এই mode-এ `loader` (data আনা), `action` (form submit), pending state ইত্যাদি feature পাওয়া যায়।

---

## ৩. React Hook Form যোগ করা

Docs: https://react-hook-form.com/get-started

```sh
bun add react-hook-form
```

> নোট: Form validation-এর জন্য সাধারণত সাথে **Zod** আর **@hookform/resolvers** ব্যবহার করা হয় (`bun add zod @hookform/resolvers`)। প্রয়োজন হলে পরে যোগ করা যাবে।

---

## ৪. shadcn/ui-এর জন্য environment setup

Docs: https://ui.shadcn.com/docs/installation → **Vite** → **Existing Project** section।

প্রথম ধাপ (Vite প্রজেক্ট তৈরি) আমাদের আগেই হয়ে গেছে, তাই দ্বিতীয় ধাপ থেকে শুরু।

### ৪.১ Tailwind CSS install

```sh
bun add tailwindcss @tailwindcss/vite
```

`src/index.css`-এর **সব কিছু মুছে** শুধু এটা রাখুন:

```css
@import "tailwindcss";
```

**`App.css` পরিষ্কার করা:**
- Vite-এর default `src/App.css` আর লাগবে না, মুছে ফেলা যায়।
- মুছলে `src/App.tsx`-এ থাকা `import './App.css'` লাইনটাও মুছতে হবে, নাহলে error দেবে।
- `App.tsx`-এ Vite-এর default demo কোড (logo, counter ইত্যাদি) আছে, যেটা App.css আর logo ফাইলের ওপর নির্ভর করে। তাই `App.tsx` পরিষ্কার করে শুধু একটা sample text রাখুন:

```tsx
function App() {
  return <h1 className="text-3xl font-bold">Hello World</h1>
}

export default App
```

Tailwind class (`text-3xl font-bold`) কাজ করলে বোঝা যাবে Tailwind ঠিকমতো setup হয়েছে।

### ৪.২ `tsconfig.json`-এ path alias

`@/` দিয়ে import করার জন্য (যেমন `@/components/ui/button`) path alias যোগ করতে হবে।

> ⚠️ সংশোধন: `compilerOptions`, `references`-এর **ভেতরে না** — এটা top level-এ, `references`-এর **পরে আলাদা property** হিসেবে বসবে।

```json
{
  "files": [],
  "references": [
    { "path": "./tsconfig.app.json" },
    { "path": "./tsconfig.node.json" }
  ],
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
  }
}
```

### ৪.৩ `tsconfig.app.json`-এ path alias

এখানে `compilerOptions`-এর **ভেতরে** (যেমন `skipLibCheck`-এর পরে) যোগ করুন:

```json
{
  "compilerOptions": {
    // ...
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
    // ...
  }
}
```

> **কেন দুই জায়গায়?** Vite TypeScript config কয়েকটা ফাইলে ভাগ করে রাখে। `tsconfig.app.json` আসলে `src/`-এর কোড check করে, তাই editor-এর `@/` import বোঝার জন্য এখানে alias লাগে। `tsconfig.json`-এ দেওয়া হয় মূলত shadcn CLI-এর জন্য, যাতে সে alias খুঁজে পায়।

> **`baseUrl` নিয়ে নোট:** TypeScript 4.1 থেকে `paths` ব্যবহারের জন্য `baseUrl` বাধ্যতামূলক না — `paths` নিজেই tsconfig ফাইলের অবস্থান থেকে resolve হয়। নতুন TypeScript version-এ `baseUrl` deprecated-ও হয়ে যাচ্ছে। তাই এটা না দিলেও চলে। (shadcn docs-এ এখনো `baseUrl` দেওয়া আছে; দিলে সমস্যা নেই, তবে TypeScript deprecation warning দিলে মুছে ফেলবেন।)

### ৪.৪ `@types/node` install

```sh
bun add -D @types/node
```

> **কেন?** পরের ধাপে `vite.config.ts`-এ Node-এর `path` module আর `__dirname` ব্যবহার হবে। এগুলোর TypeScript type পাওয়ার জন্য `@types/node` লাগে। `-D` মানে এটা **devDependency** — শুধু development-এ লাগে।

### ৪.৫ `vite.config.ts` আপডেট

পুরো ফাইলটা এই কোড দিয়ে replace করুন:

```ts
import path from "path"
import tailwindcss from "@tailwindcss/vite"
import react from "@vitejs/plugin-react"
import { defineConfig } from "vite"

// https://vite.dev/config/
export default defineConfig({
  plugins: [react(), tailwindcss()],
  resolve: {
    alias: {
      "@": path.resolve(__dirname, "./src"),
    },
  },
})
```

- `tailwindcss()` — Vite-এ Tailwind plugin চালু করে।
- `resolve.alias` — Vite-কে বলে `@` মানে `src/` folder।

> **tsconfig আর vite.config দুটোতেই alias কেন?** tsconfig-এর alias শুধু **TypeScript/editor**-কে বোঝায় (type check, autocomplete)। কোড আসলে **চালায় আর bundle করে Vite**, তাই Vite-কেও আলাদাভাবে alias বলে দিতে হয়। একটা দিলে আরেকটা কাজ করে না।
>
> নোট: Create Vite-এর template ভেদে React plugin-এর package নাম আলাদা হতে পারে (যেমন `@vitejs/plugin-react-swc`)। Replace করার আগে নিজের `vite.config.ts`-এ কোন plugin import করা আছে দেখে নিন, এবং সেটাই রাখুন।

এখন পর্যন্ত আমরা shadcn install করার জন্য যে **environment** লাগে, সেটা setup করলাম।

---

## ৫. shadcn/ui init

```sh
bunx --bun shadcn@latest init
```

> `bunx` হলো bun-এর `npx`/`pnpm dlx`-এর সমতুল্য — package install না করেই চালায়। `--bun` flag দিলে Node-এর বদলে bun runtime দিয়ে চালায়।

Prompt-এ:
- **Component library (base):** আমরা **Base UI** বেছে নিচ্ছি (অন্য option Radix UI)।
- **Preset/style:** যেকোনোটা নেওয়া যায় — আমি **Nova** বেছে নিয়েছি।

এরপর shadcn install হয়ে যাবে।

> নোট: init কমান্ড শুধু ফাইল তৈরি করে না — প্রয়োজনীয় dependency (যেমন `clsx`, `tailwind-merge`, icon library) install করে এবং **`src/index.css`-এ theme-এর CSS variable** (রং, radius ইত্যাদি) যোগ করে।

---

## ৬. Button component দিয়ে test

```sh
bunx --bun shadcn@latest add button
```

`src/App.tsx`:

```tsx
import { Button } from "@/components/ui/button"

function App() {
  return (
    <div className="flex min-h-svh flex-col items-center justify-center">
      <Button>Click me</Button>
    </div>
  )
}

export default App
```

Button ঠিকমতো styled হয়ে দেখালে বুঝবেন shadcn সফলভাবে install ও configure হয়েছে।

---

## ৭. shadcn যা তৈরি করে

```
src/
├── components/
│   └── ui/
│       └── button.tsx    ← shadcn add করা component এখানে আসে
└── lib/
    └── utils.ts          ← cn() function
components.json           ← shadcn-এর configuration (project root-এ)
```

### `src/components/ui/`
shadcn দিয়ে যে component add করবেন (`button`, `card`, `dialog` …), সেগুলো এখানে আসে। কোডটা আপনার নিজের — প্রয়োজন মতো বদলাতে পারবেন।

### `src/lib/utils.ts` — `cn()` function
`cn()` দিয়ে dynamic/conditional class name যোগ করা যায়। এটা দুটো library মিলিয়ে বানানো:
- **`clsx`** — শর্ত অনুযায়ী class যোগ করে
- **`tailwind-merge`** — conflict হওয়া Tailwind class ঠিকভাবে merge করে (যেমন `p-2` আর `p-4` একসাথে থাকলে শেষেরটা রাখে)

```tsx
<div className={cn("p-2 text-sm", isActive && "bg-blue-500", className)} />
```

### `components.json`
shadcn-এর configuration ফাইল। এতে style, base library, alias (`@/components`, `@/lib/utils`), CSS ফাইলের path, icon library ইত্যাদি লেখা থাকে। shadcn CLI প্রতিবার `add` করার সময় এই ফাইল পড়ে ঠিক করে component কোথায় রাখবে আর কীভাবে import করবে।

> ⚠️ সংশোধন: "এতে হাত দেওয়ার দরকার নেই, shadcn-এর internal ব্যবহারের জন্য" — সাধারণত হাত দিতে হয় না ঠিকই, তবে এটা internal বা লুকানো ফাইল না। প্রয়োজনে (যেমন folder structure বা alias বদলালে) এটা **edit করা যায় এবং করতে হয়**। এটা git-এ commit করতে হবে।

---

## ৮. shadcn install-এ ভুল হলে

Install করার সময় কিছু ভুল হলে:
1. `src/components/`, `src/lib/` folder আর `components.json` ফাইল মুছে ফেলুন।
2. `src/index.css`-এ shadcn যে theme/CSS variable যোগ করেছিল, সেগুলোও মুছে শুধু `@import "tailwindcss";` রাখুন।
3. আবার init command চালান:

```sh
bunx --bun shadcn@latest init
```

> ⚠️ সংযোজন: শুধু ফাইল মুছলে `index.css`-এর পরিবর্তন আর install হওয়া dependency থেকে যায়, যা পরের init-এ গোলমাল করতে পারে।
>
> 💡 সবচেয়ে নিরাপদ উপায়: shadcn init-এর **আগে একটা git commit** করে রাখুন। ভুল হলে `git restore .` আর `git clean -fd` দিয়ে (সাবধানে — এটা untracked ফাইল মুছে দেয়) commit-এর অবস্থায় ফিরে যেতে পারবেন।