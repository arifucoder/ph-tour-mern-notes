# Dark Theme Toggle (shadcn + Vite + Base UI)

এই নোটে shadcn-এর নিয়ম মেনে একটা **Light / Dark / System** theme toggle বানানো হয়েছে। এর তিনটা অংশ:

1. **ThemeProvider** — বর্তমান theme মনে রাখে, `<html>`-এ `dark` class যোগ করে বা সরায়।
2. **`main.tsx`-এ wrap করা** — যাতে পুরো app থেকে theme পাওয়া যায়।
3. **ModeToggle button** — user এখান থেকে theme বদলায়।

Docs: https://ui.shadcn.com/docs/dark-mode/vite
(shadcn-এর সাইটে "dark mode" লিখে search করে **Vite** বেছে নিলে এই page আসে।)

> ⚠️ মনে রাখুন: theme toggle শুধু `<html>`-এ `dark` class বসায়। রং বদলানোর আসল কাজ করে `index.css`-এর `.dark { ... }` block আর semantic token। তাই কোথাও `text-black`/`bg-white`-এর মতো hard-coded রং থাকলে toggle চাপলেও সেটা বদলাবে না (৮ নম্বর নোটের notice দেখুন)।

---

## ১. ThemeProvider

shadcn docs-এ ফাইলটা `components/` folder-এ রাখতে বলা আছে। আমরা রাখব আলাদা **`src/providers/`** folder-এ। কারণ এটা কোনো UI component না, পুরো app-কে একটা context দেয়। পরে Redux Provider বা Auth Provider-ও একই folder-এ যাবে।

```
src/
└── providers/
    └── theme.provider.tsx
```

shadcn থেকে কোড কপি করে `theme.provider.tsx`-এ বসান:

```tsx
// src/providers/theme.provider.tsx
import { createContext, useContext, useEffect, useState } from "react"

type Theme = "dark" | "light" | "system"

type ThemeProviderProps = {
  children: React.ReactNode
  defaultTheme?: Theme
  storageKey?: string
}

type ThemeProviderState = {
  theme: Theme
  setTheme: (theme: Theme) => void
}

const initialState: ThemeProviderState = {
  theme: "system",
  setTheme: () => null,
}

const ThemeProviderContext = createContext<ThemeProviderState>(initialState)

export function ThemeProvider({
  children,
  defaultTheme = "system",
  storageKey = "vite-ui-theme",
  ...props
}: ThemeProviderProps) {
  const [theme, setTheme] = useState<Theme>(
    () => (localStorage.getItem(storageKey) as Theme) || defaultTheme
  )

  useEffect(() => {
    const root = window.document.documentElement

    root.classList.remove("light", "dark")

    if (theme === "system") {
      const systemTheme = window.matchMedia("(prefers-color-scheme: dark)")
        .matches
        ? "dark"
        : "light"

      root.classList.add(systemTheme)
      return
    }

    root.classList.add(theme)
  }, [theme])

  const value = {
    theme,
    setTheme: (theme: Theme) => {
      localStorage.setItem(storageKey, theme)
      setTheme(theme)
    },
  }

  return (
    <ThemeProviderContext.Provider {...props} value={value}>
      {children}
    </ThemeProviderContext.Provider>
  )
}

export const useTheme = () => {
  const context = useContext(ThemeProviderContext)

  if (context === undefined)
    throw new Error("useTheme must be used within a ThemeProvider")

  return context
}
```

### কোডটা কী করে

| অংশ | কাজ |
|---|---|
| `type Theme` | Theme হতে পারে তিনটার একটা: `"dark"`, `"light"` বা `"system"` |
| `createContext` | একটা context বানায়, যাতে যেকোনো component থেকে theme পড়া আর বদলানো যায় |
| `useState(() => localStorage.getItem(...))` | শুরুতে browser-এর **localStorage** থেকে আগের বেছে নেওয়া theme পড়ে। কিছু না থাকলে `defaultTheme` |
| `useEffect` | Theme বদলালেই `<html>` থেকে পুরনো `light`/`dark` class সরিয়ে নতুনটা বসায় |
| `"system"` হলে | `matchMedia("(prefers-color-scheme: dark)")` দিয়ে computer/phone-এর setting দেখে, সেই অনুযায়ী `dark` বা `light` বসায় |
| `setTheme` | নতুন theme localStorage-এ save করে, তাই page reload করলেও মনে থাকে |
| `useTheme()` | Component-এ theme ব্যবহারের hook: `const { theme, setTheme } = useTheme()` |

> নোট: `context === undefined` শর্তটা আসলে কখনো সত্যি হয় না, কারণ context-এর default মান (`initialState`) দেওয়া আছে। তাই ThemeProvider-এর বাইরে `useTheme()` ডাকলে error না দিয়ে চুপচাপ কিছুই করবে না। shadcn-এর কোড এভাবেই আছে, বদলানোর দরকার নেই। শুধু মনে রাখুন, সব component যেন ThemeProvider-এর ভেতরে থাকে।

> নোট: একই ফাইল থেকে component (`ThemeProvider`) আর hook (`useTheme`) দুটোই export হওয়ায় linter (ESLint/Oxlint) "only export components" warning দিতে পারে। এটা শুধু Vite-এর fast refresh (HMR) সংক্রান্ত warning, app ঠিকমতোই চলবে। চাইলে `useTheme` আলাদা ফাইলে সরিয়ে warning দূর করা যায়।

---

## ২. `main.tsx`-এ ThemeProvider দিয়ে wrap করা

পুরো app যাতে theme পায়, তাই `RouterProvider`-কে `ThemeProvider` দিয়ে wrap করব:

```tsx
// src/main.tsx
import { StrictMode } from "react";
import { createRoot } from "react-dom/client";
import { RouterProvider } from "react-router";
import "./index.css";
import { router } from "@/routes";
import { ThemeProvider } from "@/providers/theme.provider";

createRoot(document.getElementById("root")!).render(
  <StrictMode>
    <ThemeProvider defaultTheme="system" storageKey="phovia-ui-theme">
      <RouterProvider router={router} />
    </ThemeProvider>
  </StrictMode>,
);
```

- **`defaultTheme="system"`:** user প্রথমবার এলে (কিছু বেছে নেওয়ার আগে) তার computer/phone-এ যে theme দেওয়া আছে সেটাই দেখাবে। এটাই সবচেয়ে ভালো default।
- **`storageKey="phovia-ui-theme"`:** localStorage-এ কোন নামে theme save হবে। নিজের প্রজেক্টের নাম দিলে একই browser-এ অন্য প্রজেক্টের theme-এর সাথে গুলিয়ে যায় না।

> ⚠️ সংশোধন: নোটে লেখা ছিল "root layout-কে wrap করতে হবে"। আসলে wrap করা হচ্ছে **`RouterProvider`**-কে। আর `App`/`CommonLayout` শুধু public route-এর জন্য render হয়, তাই ThemeProvider সেখানে রাখলে admin page theme পেত না। `main.tsx`-এ রাখলে **সব route** (public + admin) theme পায় (৫ নম্বর নোটের শেষ অংশ দেখুন)।

---

## ৩. Dropdown Menu install

ModeToggle-এ একটা dropdown লাগবে (Light / Dark / System)। তাই component লেখার **আগেই** shadcn থেকে dropdown install করে নিন:

```sh
bunx --bun shadcn@latest add dropdown-menu
```

> নোট: নোটে dropdown install-এর কথা component লেখার পরে ছিল। আগে install না করলে `@/components/ui/dropdown-menu` খুঁজে না পেয়ে import error দেবে, তাই ধাপটা আগে আনা হলো।

---

## ৪. ModeToggle component

ফাইল: `src/components/layout/ModeToggle.tsx`

> ⚠️ সংশোধন: নোটে ফাইলের নাম `ModeToggler.tsx`, কিন্তু component-এর নাম `ModeToggle`। ফাইল আর component-এর নাম একই রাখা নিয়ম, তাই ফাইলের নাম **`ModeToggle.tsx`**।

### shadcn docs-এর কোড আমাদের প্রজেক্টে কেন error দেয়

shadcn-এর dark mode page-এর কোড **Radix UI**-র জন্য লেখা:

```tsx
<DropdownMenuTrigger asChild>
  <Button variant="outline" size="icon">...</Button>
</DropdownMenuTrigger>
```

কিন্তু আমরা shadcn init-এ **Base UI** বেছে নিয়েছিলাম (৪ নম্বর নোট)। ফলে দুটো error আসে:

**Error ১ — `asChild` নেই।** `asChild` হলো Radix-এর prop। Base UI-তে এই prop নেই, তাই TypeScript error দেয়।

**Error ২ — button-এর ভেতরে button।** `asChild` মুছে দিলে `DropdownMenuTrigger` নিজেই একটা `<button>` render করে, আর তার ভেতরে আমাদের `<Button>`-ও একটা `<button>`। HTML-এর নিয়মে **button-এর ভেতরে button রাখা যায় না**। React console-এ warning দেয় (`<button> cannot be a descendant of <button>`), আর click-ও ঠিকমতো কাজ করে না।

### সমাধান: Base UI-এর `render` prop

দুই library-তে একই কাজ, শুধু লেখার ধরন আলাদা: "trigger নিজে আলাদা button না বানিয়ে আমার দেওয়া `<Button>`-টাকেই trigger হিসেবে ব্যবহার করো।"

| | Radix UI | Base UI (আমাদের) |
|---|---|---|
| Prop | `asChild` | `render` |
| লেখার ধরন | `<Trigger asChild><Button>…</Button></Trigger>` | `<Trigger render={<Button />}>…</Trigger>` |
| Button-এর ভেতরের জিনিস (icon, text) | `<Button>`-এর ভেতরে | `<Trigger>`-এর ভেতরে |

shadcn-এর dropdown page-এ (https://ui.shadcn.com/docs/components/base/dropdown-menu) উপরে **Base** বেছে নিলে `render` দিয়ে লেখা কোড দেখা যায়, আর **Radix** বেছে নিলে `asChild` দিয়ে। যেকোনো shadcn component-এর কোড কপি করার আগে তাই **Base** বেছে নেওয়া আছে কিনা দেখে নেবেন।

### Final কোড

```tsx
// src/components/layout/ModeToggle.tsx
import { Moon, Sun } from "lucide-react";

import { Button } from "@/components/ui/button";
import {
  DropdownMenu,
  DropdownMenuContent,
  DropdownMenuItem,
  DropdownMenuTrigger,
} from "@/components/ui/dropdown-menu";
import { useTheme } from "@/providers/theme.provider";

export function ModeToggle() {
  const { setTheme } = useTheme();

  return (
    <DropdownMenu>
      <DropdownMenuTrigger render={<Button variant="outline" size="icon" />}>
        <Sun className="h-[1.2rem] w-[1.2rem] scale-100 rotate-0 transition-all dark:scale-0 dark:-rotate-90" />
        <Moon className="absolute h-[1.2rem] w-[1.2rem] scale-0 rotate-90 transition-all dark:scale-100 dark:rotate-0" />
        <span className="sr-only">Toggle theme</span>
      </DropdownMenuTrigger>
      <DropdownMenuContent align="end">
        <DropdownMenuItem onClick={() => setTheme("light")}>Light</DropdownMenuItem>
        <DropdownMenuItem onClick={() => setTheme("dark")}>Dark</DropdownMenuItem>
        <DropdownMenuItem onClick={() => setTheme("system")}>System</DropdownMenuItem>
      </DropdownMenuContent>
    </DropdownMenu>
  );
}
```

> ছোট পরিবর্তন:
> - `render={<Button variant="outline" size="icon"></Button>}`-কে ছোট করে `render={<Button variant="outline" size="icon" />}` লেখা হয়েছে। ভেতরে কিছু না থাকলে self-closing tag দেওয়াই নিয়ম, কাজ একই।
> - `import { Button } from "../ui/button"`-এর বদলে `"@/components/ui/button"`। বাকি সব import `@/` দিয়ে, তাই একই ধরন রাখা হলো। ফাইল সরালেও `@/` path ভাঙে না।

### Sun/Moon icon-এর animation কীভাবে কাজ করে

| | Light mode | Dark mode (`dark:` class কাজ করে) |
|---|---|---|
| ☀️ Sun | `scale-100 rotate-0` → দেখা যায় | `dark:scale-0 dark:-rotate-90` → ঘুরে ছোট হয়ে অদৃশ্য |
| 🌙 Moon | `scale-0 rotate-90` → অদৃশ্য | `dark:scale-100 dark:rotate-0` → ঘুরে বড় হয়ে দেখা যায় |

- `transition-all` থাকায় বদলটা ঘুরে ঘুরে animation হয়ে দেখায়।
- Moon-এ `absolute` থাকায় সে Sun-এর ঠিক ওপরে বসে, পাশে না। তাই একটা button-এ একসাথে একটাই icon দেখা যায়।
- `sr-only` লেখাটা চোখে দেখা যায় না, শুধু screen reader পড়ে। Icon-only button-এর একটা নাম থাকা দরকার, এটাই সেই কাজ করে।

---

## ৫. Navbar-এ ModeToggle বসানো

যেখানে toggle button দেখাতে চান, সেখানে import করে বসিয়ে দিন। সাধারণত Navbar-এর ডান দিকে, Log in-এর আগে:

```tsx
// src/components/layout/Navbar.tsx
import { ModeToggle } from "./ModeToggle";

{/* Right side */}
<div className="flex items-center gap-3.5">
  <ModeToggle />
  <Link to="/login" ...>Log in</Link>
  ...
</div>
```

> নোট: `ModeToggle` **named export** (`export function ModeToggle`), তাই import-এ `{ }` লাগবে: `import { ModeToggle } from "./ModeToggle"`। Default export হলে `{ }` ছাড়া লেখা হতো।

---

## ৬. Teacher-এর কথা: `forwardRef` আর React 19 — সহজ ব্যাখ্যা

### `ref` কী আর কেন লাগে

`ref` দিয়ে কোনো component আসল HTML element-কে (যেমন `<button>`) সরাসরি ধরতে পারে।

Dropdown-এর ক্ষেত্রে এটা জরুরি। Menu খুললে সেটা **button-এর ঠিক নিচে** দেখাতে হবে। এজন্য dropdown-কে জানতে হয় button স্ক্রিনের কোথায় আছে। তাই `render={<Button />}` দিলে `DropdownMenuTrigger` আমাদের `<Button>`-এর ভেতরের আসল `<button>` element-এ একটা `ref` পাঠায়।

### React 19-এর আগে: `forwardRef`

আগে সাধারণ function component-এ `ref` prop হিসেবে পাঠানো যেত না, React সেটা মাঝপথেই আটকে দিত। তাই component-কে `forwardRef` দিয়ে মুড়ে দিতে হতো, যাতে ref ভেতরের element পর্যন্ত পৌঁছায়:

```tsx
// React 18 — পুরনো নিয়ম
const Button = forwardRef<HTMLButtonElement, ButtonProps>((props, ref) => (
  <button ref={ref} {...props} />
));
```

### React 19-এ: `ref` সাধারণ prop

React 19 থেকে `ref` অন্য যেকোনো prop-এর মতোই পাঠানো যায়। `forwardRef` আর লাগে না:

```tsx
// React 19 — নতুন নিয়ম
function Button({ ref, ...props }: ButtonProps) {
  return <button ref={ref} {...props} />;
}
```

shadcn-এর নতুন component-গুলো (আমাদের `components/ui/button.tsx` সহ) এই নতুন নিয়মে লেখা, তাই সেগুলো চালাতে **React 19** লাগে।

### আমাদের কী করতে হবে

**কিছুই না।** Teacher-এর প্রজেক্টে React-এর পুরনো version ছিল, তাই তাঁকে React 19-এ update করতে হয়েছে। আমাদের প্রজেক্ট নতুন Vite template দিয়ে বানানো, তাই `react`, `react-dom` আর তাদের type package (`@types/react`, `@types/react-dom`) আগে থেকেই React 19-এর।

নিশ্চিত হতে `package.json` দেখুন:

```json
"dependencies": {
  "react": "^19.x.x",
  "react-dom": "^19.x.x"
},
"devDependencies": {
  "@types/react": "^19.x.x",
  "@types/react-dom": "^19.x.x"
}
```

চারটার version `19` দিয়ে শুরু হলে সব ঠিক আছে। কোনোটা `18` হলে update করুন:

```sh
bun add react@latest react-dom@latest
bun add -D @types/react@latest @types/react-dom@latest
```

---

## ৭. (ঐচ্ছিক উন্নতি) Page load-এর সময় সাদা ঝলক বন্ধ করা

### সমস্যা

Dark mode বেছে রেখে page reload করলে এক মুহূর্তের জন্য **সাদা (light) page** দেখা যায়, তারপর dark হয়। এটাকে **flash of wrong theme** বলে। কারণ ThemeProvider-এর `useEffect` চলে React পুরো page render করার **পরে**, তাই তার আগ পর্যন্ত `<html>`-এ `dark` class থাকে না।

### সমাধান

`index.html`-এর `<head>`-এ একটা ছোট script বসান। এটা React load হওয়ার **আগেই** চলে আর সঠিক class বসিয়ে দেয়:

```html
<!-- index.html, <head>-এর ভেতরে -->
<script>
  (function () {
    try {
      var theme = localStorage.getItem("phovia-ui-theme") || "system";
      var dark =
        theme === "dark" ||
        (theme === "system" && window.matchMedia("(prefers-color-scheme: dark)").matches);
      document.documentElement.classList.add(dark ? "dark" : "light");
    } catch (e) {}
  })();
</script>
```

> ⚠️ এখানের `"phovia-ui-theme"` আর `main.tsx`-এর `storageKey` **হুবহু একই** হতে হবে, নাহলে script ভুল জায়গায় theme খুঁজবে।
>
> `try/catch` রাখা হয়েছে কারণ কিছু private browsing mode-এ localStorage ব্যবহার করতে গেলে error দেয়। তখন script চুপচাপ থেমে যায়, page ভাঙে না।