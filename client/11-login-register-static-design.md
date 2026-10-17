# Frontend Notes — ১১: Login ও Register Page Design

Phovia Tour-এর Login আর Register page-এর ডিজাইন (শুধু UI, এখনো backend-এর সাথে যুক্ত না)। Form-এর আসল কাজ (validation, API call) পরের ধাপে **React Hook Form** দিয়ে করা হবে।

> আমাদের প্রজেক্ট এখন **Radix UI** দিয়ে চলছে (১০ নম্বর নোট)। এই page-এর component-গুলো (Button, Input, Label, Separator) Radix আর Base UI দুটোতেই একইভাবে লেখা হয়, তাই কোড বদলাতে হবে না।

---

## ১. ডিজাইনের গঠন

```
Desktop (lg ও বড়) — দুই ভাগ:
┌────────────────────────┬────────────────────────┐
│ ┌────────────────────┐ │                        │
│ │ [Logo]    Photo tag│ │   Welcome back         │
│ │                    │ │   [Continue w/ Google] │
│ │      (ছবি)         │ │   ── or log in with ── │
│ │                    │ │   Email                │
│ │ Your next trip...  │ │   Password      Forgot?│
│ └────────────────────┘ │   [Log in]             │
│      Image panel        │   New? Create account  │
└────────────────────────┴────────────────────────┘

Mobile — ছবি লুকানো, উপরে Logo, নিচে শুধু form
```

Login আর Register-এর **বাম দিকের ছবির অংশ আর পুরো কাঠামো একই**, বদলায় শুধু ছবির ওপরের লেখা আর ডান দিকের form।

---

## ২. প্রস্তুতি

### shadcn component

```sh
bunx --bun shadcn@latest add button input label separator
```

(আগে যেগুলো add করা আছে, CLI সেগুলো আবার নামাবে না।)

### lucide-react (icon)

Docs: https://lucide.dev/guide/react/getting-started

shadcn init সাধারণত `lucide-react` আগেই install করে দেয়। `package.json`-এ না থাকলে:

```sh
bun add lucide-react
```

### ছবি আর icon

```
src/
├── assets/
│   └── images/
│       └── auth/
│           └── inani-beach.jpeg     ← বাম দিকের ছবি
└── components/
    ├── Logo.tsx                     ← ৯ নম্বর নোটে বানানো
    └── icons/
        └── GoogleIcon.tsx           ← Google-এর logo (SVG component)
```

> ⚠️ **সংশোধন — import-এর জায়গা:**
> - Teacher-এর কোডে `Logo` import হচ্ছে `@/assets/icons/Logo` থেকে। আমাদের প্রজেক্টে Logo আছে **`@/components/Logo`**-এ (৯ নম্বর নোট)। Logo-র রং theme token (`fill-primary`), তাই এটা `.tsx` component, `assets`-এ রাখা static ফাইল না।
> - `GoogleIcon.tsx`-ও একটা React component, তাই এটা `assets/`-এ না রেখে **`components/icons/`**-এ রাখা ভালো (৬ নম্বর নোট: `assets` শুধু ছবি আর `.svg` ফাইলের মতো static জিনিসের জন্য)।
> - বিকল্প: Google logo-র রং নির্দিষ্ট (theme-এর সাথে বদলায় না), তাই চাইলে `assets/icons/google.svg` ফাইল বানিয়ে `<img src={googleIcon} alt="" />` দিয়েও দেখানো যায়।

> 💡 ছবি: বড় `.jpeg` page load ধীর করে। [squoosh.app](https://squoosh.app) দিয়ে `.webp`-এ রূপান্তর করে ~1600px চওড়া করে নিলে size অনেক কমে যায়।

---

## ৩. প্রথম ধাপ: পুরো page একবারে

প্রথমে Claude-এর বানানো পুরো Login আর Register page একটা করে ফাইলে বসিয়ে browser-এ দেখে নেওয়া হয়েছে ডিজাইন ঠিক আসছে কিনা। এরপর কোডটা ছোট ছোট অংশে ভাগ করা হয়েছে (নিচে)। পুরো page-এর কোডটা ভাগ করার পরের কোডের মতোই, শুধু সব এক ফাইলে।

### আসল কোডে যা ঠিক করা হয়েছে

> ⚠️ **সংশোধন ১ — Hard-coded রং (dark mode-এ ভাঙবে):** কোডে `bg-white`, `text-slate-900`, `text-slate-600`, `text-indigo-600`, `bg-indigo-600` ছিল। Dark mode-এ এগুলো বদলায় না (৮ নম্বর নোটের notice)। Semantic token দিয়ে বদলানো হয়েছে:
>
> | আগে | এখন |
> |---|---|
> | `bg-white` | `bg-background` |
> | `text-slate-900` | `text-foreground` |
> | `text-slate-600`, `text-slate-500` | `text-muted-foreground` |
> | `text-indigo-600` | `text-primary` |
> | `bg-indigo-600 hover:bg-indigo-700` | মুছে দেওয়া হয়েছে, কারণ `<Button>`-এর default রং-ই primary |
> | `bg-slate-300` (ছবি load হওয়ার আগে) | `bg-muted` |
>
> ছবির **ওপরের** লেখায় `text-white` আর `from-black/70` রাখা হয়েছে। ছবি dark mode-এ বদলায় না, তাই এখানে এটাই ঠিক (৮ নম্বর নোটের ব্যতিক্রম)।

> ⚠️ **সংশোধন ২ — Dark mode-এ Logo অদৃশ্য হয়ে যেত:** ছবির ওপরে Logo একটা `bg-white/90` (সাদা) pill-এর ভেতরে ছিল। কিন্তু আমাদের Logo-র লেখা `text-foreground`, যেটা dark mode-এ হালকা রঙের হয়ে যায়। ফলে সাদা pill-এর ওপর হালকা লেখা প্রায় দেখাই যেত না। Pill-এর রং **`bg-background/90`** করা হয়েছে, এখন দুই mode-এ লেখা আর background একসাথে বদলায়।

> ⚠️ **সংশোধন ৩ — `<a href>` থেকে `<Link to>`:** "Forgot password?", "Create an account", "Log in", "Terms & Privacy" সবগুলো `<a>` ছিল, ক্লিক করলে পুরো page reload হতো। সব `<Link>` করা হয়েছে।

> ⚠️ **সংশোধন ৪ — Route-এর নাম মিলছিল না:** Login page-এ "Create an account" যাচ্ছিল `/signup`-এ, কিন্তু Navbar-এর Sign up যায় `/register`-এ। দুটো আলাদা হলে একটা 404 দেবে। সব জায়গায় **`/register`** করা হয়েছে।

> ⚠️ **সংশোধন ৫ — Eye button কিছুই করছিল না:** Password-এর পাশের চোখের icon শুধু দেখানো ছিল, চাপলে password দেখাত না। এখন একটা `PasswordInput` component বানানো হয়েছে, যেটা password দেখায়/লুকায় আর icon বদলায় (👁 → 🙈)।

> ⚠️ **সংশোধন ৬ — Submit চাপলে page reload:** `<form>`-এ কোনো handler না থাকলে "Log in" চাপলে browser পুরো page reload করে দেয়। আপাতত `e.preventDefault()` দিয়ে আটকানো হয়েছে। পরে React Hook Form-এর `handleSubmit` এর জায়গা নেবে।

> ➕ **সংযোজন — Input-এ `name`, `autoComplete`, `required`:**
> - `autoComplete="email"` / `"current-password"` / `"new-password"` দিলে browser আর password manager ঠিকমতো autofill করে, আর Register-এ নতুন strong password suggest করে।
> - `required` দিলে খালি form submit হয় না।

> নোট: ফাইলের নাম (`pages/Login.tsx`) আর কোডের comment-এ লেখা নাম (`LoginPage.tsx`) আলাদা ছিল। ফাইল আর component-এর নাম একই রাখা হয়েছে: `Login.tsx` → `Login`, `Register.tsx` → `Register`।

> নোট: `min-h-screen`-এর বদলে `min-h-dvh` (mobile-এর address bar সমস্যা, ৯ নম্বর নোট), আর `h-4 w-4`-এর বদলে ছোট করে `size-4`।

---

## ৪. ভাগ করা: কোন ফাইলে কী

Login আর Register-এ একই কোড দুইবার লেখা ছিল: পুরো কাঠামো, ছবির অংশ, password field। তাই এগুলো আলাদা component-এ রাখা হয়েছে:

```
src/
├── components/
│   ├── ui/
│   │   └── password-input.tsx         ← চোখের button সহ password field (দুই page-এ ৩ বার ব্যবহার)
│   ├── layout/
│   │   └── AuthLayout.tsx             ← দুই ভাগের কাঠামো + ছবির অংশ
│   └── modules/
│       └── Authentication/
│           ├── LoginForm.tsx          ← শুধু login form
│           └── RegisterForm.tsx       ← শুধু register form
└── pages/
    ├── Login.tsx                      ← AuthLayout + LoginForm জোড়া লাগায়
    └── Register.tsx                   ← AuthLayout + RegisterForm জোড়া লাগায়
```

| ফাইল | কেন এখানে (৬ নম্বর নোটের structure অনুযায়ী) |
|---|---|
| `ui/password-input.tsx` | ছোট, atomic, যেকোনো form-এ ব্যবহারযোগ্য |
| `layout/AuthLayout.tsx` | একাধিক page-এর share করা কাঠামো, তাই layout |
| `modules/Authentication/*Form.tsx` | একটা নির্দিষ্ট page-এর section, তাই module |
| `pages/*.tsx` | শুধু অংশগুলো সাজায়, নিজের কোড প্রায় নেই |

> নোট: Teacher-এর কোডে `LoginForm`-এর ভেতরে mobile logo আর centering-এর `div`-ও ছিল, অর্থাৎ form-এর সাথে page-এর কাঠামোও মিশে ছিল। এখানে কাঠামো সরানো হয়েছে `AuthLayout`-এ, তাই Form শুধু form-এর কাজ করে, আর Register-এ একই কাঠামো আবার লিখতে হয় না।

---

## ৫. কোড

### ৫.১ `PasswordInput`

```tsx
// src/components/ui/password-input.tsx
import { Input } from "@/components/ui/input";
import { cn } from "@/lib/utils";
import { Eye, EyeOff } from "lucide-react";
import { useState, type ComponentProps } from "react";

type PasswordInputProps = Omit<ComponentProps<typeof Input>, "type">;

export function PasswordInput({ className, ...props }: PasswordInputProps) {
  const [show, setShow] = useState(false);

  return (
    <div className="relative">
      <Input type={show ? "text" : "password"} className={cn("pr-10", className)} {...props} />
      <button
        type="button"
        onClick={() => setShow((prev) => !prev)}
        aria-label={show ? "Hide password" : "Show password"}
        aria-pressed={show}
        className="absolute right-3 top-1/2 -translate-y-1/2 text-muted-foreground transition-colors hover:text-foreground"
      >
        {show ? <EyeOff className="size-4" /> : <Eye className="size-4" />}
      </button>
    </div>
  );
}
```

- **`useState(false)`:** শুরুতে password লুকানো থাকে। চাপলে `show` উল্টে যায়।
- **`type={show ? "text" : "password"}`:** `"text"` হলে অক্ষর দেখা যায়, `"password"` হলে `•••`।
- **`Omit<ComponentProps<typeof Input>, "type">`:** shadcn `Input` যা যা prop নেয় (`id`, `name`, `placeholder` …) সবই নেবে, শুধু `type` বাদে, কারণ সেটা এই component নিজে ঠিক করে।
- **`type="button"`:** form-এর ভেতরের button-এর default type হলো `submit`। এটা না দিলে চোখের icon চাপলেই form submit হয়ে যেত।
- **`aria-label` বদলায়:** screen reader জানতে পারে button এখন দেখাবে নাকি লুকাবে।
- **`{...props}`:** React 19-এ `ref`-ও সাধারণ prop হিসেবে এর মধ্যেই চলে যায় (`dark-theme-toggle.md`-এর forwardRef অংশ)। পরে React Hook Form-এর `register()` ঠিকমতো কাজ করবে।

### ৫.২ `AuthLayout`

```tsx
// src/components/layout/AuthLayout.tsx
import heroImage from "@/assets/images/auth/inani-beach.jpeg";
import Logo from "@/components/Logo";
import type { ReactNode } from "react";

interface AuthLayoutProps {
  title: ReactNode;        // ছবির ওপরের বড় লেখা
  description: string;     // তার নিচের ছোট লেখা
  children: ReactNode;     // ডান দিকের form
}

export default function AuthLayout({ title, description, children }: AuthLayoutProps) {
  return (
    <div className="grid min-h-dvh bg-background lg:grid-cols-2">
      {/* Left: image panel (desktop only) */}
      <div className="relative hidden p-4 lg:block">
        <div className="relative h-full overflow-hidden rounded-3xl bg-muted">
          <img
            src={heroImage}
            alt="Inani Beach, Cox's Bazar"
            className="absolute inset-0 size-full object-cover"
          />
          <div className="absolute inset-0 bg-linear-to-t from-black/70 via-black/10 to-transparent" />

          <div className="absolute left-8 top-8 rounded-full bg-background/90 px-3 py-1.5 backdrop-blur">
            <Logo size="sm" />
          </div>

          <span className="absolute right-6 top-8 rounded bg-black/60 px-2 py-1 text-[10px] font-semibold uppercase tracking-wide text-white">
            Photo · Inani Beach, Cox&apos;s Bazar
          </span>

          <div className="absolute bottom-0 left-0 p-10 text-white">
            <h2 className="text-4xl font-bold leading-tight">{title}</h2>
            <p className="mt-3 max-w-sm text-sm text-white/80">{description}</p>
          </div>
        </div>
      </div>

      {/* Right: form */}
      <div className="flex flex-col px-4 py-8 sm:px-6">
        {/* Mobile-only logo (image panel hidden on small screens) */}
        <div className="lg:hidden">
          <Logo />
        </div>

        <div className="flex flex-1 items-center justify-center">
          <div className="w-full max-w-sm">{children}</div>
        </div>
      </div>
    </div>
  );
}
```

**ছবির অংশ কীভাবে সাজানো:**

| স্তর | Class | কাজ |
|---|---|---|
| ১. বাইরের box | `relative h-full overflow-hidden rounded-3xl` | কোণা গোল, ভেতরের সব এর মধ্যে আটকে থাকে |
| ২. ছবি | `absolute inset-0 size-full object-cover` | পুরো box ভরে দেয়, ছবি চ্যাপ্টা না করে প্রয়োজনে কেটে নেয় |
| ৩. কালো gradient | `bg-linear-to-t from-black/70 …` | নিচ থেকে কালো → স্বচ্ছ। এতে সাদা লেখা যেকোনো ছবির ওপরে পড়া যায় |
| ৪. Logo, tag, লেখা | `absolute` + `top-8`/`bottom-0` | ছবির ওপর নির্দিষ্ট জায়গায় ভাসে |

- `hidden lg:block` → ছবি শুধু desktop-এ দেখায়। Mobile-এ লুকানো থাকে, তাই সেখানে আলাদা Logo (`lg:hidden`) রাখা হয়েছে।
- `title`-এর type `ReactNode`, `string` না। কারণ লেখার মাঝে `<br />` পাঠাতে হয়।
- `{children}` → এখানেই LoginForm বা RegisterForm বসে।

> `bg-linear-to-t` হলো Tailwind v4-এর নাম। v3-এ একে বলা হতো `bg-gradient-to-t`।

### ৫.৩ `LoginForm`

```tsx
// src/components/modules/Authentication/LoginForm.tsx
import GoogleIcon from "@/components/icons/GoogleIcon";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";
import { PasswordInput } from "@/components/ui/password-input";
import { Separator } from "@/components/ui/separator";
import { Link } from "react-router";

export default function LoginForm() {
  return (
    <>
      <h1 className="text-3xl font-bold text-foreground">Welcome back</h1>
      <p className="mt-2 text-sm text-muted-foreground">Log in to manage your bookings.</p>

      <Button variant="outline" className="mt-6 w-full gap-2" type="button">
        <GoogleIcon />
        Continue with Google
      </Button>

      <div className="my-5 flex items-center gap-3">
        <Separator className="flex-1" />
        <span className="text-xs text-muted-foreground">or log in with email</span>
        <Separator className="flex-1" />
      </div>

      {/* পরে React Hook Form-এর handleSubmit দিয়ে বদলানো হবে */}
      <form className="space-y-5" onSubmit={(e) => e.preventDefault()}>
        <div className="space-y-2">
          <Label htmlFor="email">Email</Label>
          <Input id="email" name="email" type="email" autoComplete="email" placeholder="you@example.com" required />
        </div>

        <div className="space-y-2">
          <div className="flex items-center justify-between">
            <Label htmlFor="password">Password</Label>
            <Link to="/forgot-password" className="text-xs font-medium text-primary underline underline-offset-4">
              Forgot password?
            </Link>
          </div>
          <PasswordInput id="password" name="password" autoComplete="current-password" required />
        </div>

        <Button type="submit" className="w-full">
          Log in
        </Button>
      </form>

      <p className="mt-6 text-center text-sm text-muted-foreground">
        New to Phovia Tour?{" "}
        <Link to="/register" className="font-medium text-primary underline underline-offset-4">
          Create an account
        </Link>
      </p>
    </>
  );
}
```

- **`<>…</>` (Fragment):** বাইরে বাড়তি `div` লাগছে না, কারণ কাঠামো (`max-w-sm` ইত্যাদি) `AuthLayout` দিচ্ছে।
- **`<Label htmlFor="email">` + `<Input id="email">`:** দুটো জোড়া লাগে। Label-এ click করলে input-এ cursor যায়, আর screen reader বুঝতে পারে input-টা কীসের জন্য।
- **Google button-এ `type="button"`:** এটা form-এর বাইরে, তবু রাখা ভালো। এটা submit না, শুধু একটা কাজ করবে (পরে Google login)।
- **"or log in with email" লাইন:** দুই পাশে দুটো `Separator`, `flex-1` দিয়ে যতটা জায়গা আছে সমানভাবে ভাগ করে নেয়।

### ৫.৪ `RegisterForm`

```tsx
// src/components/modules/Authentication/RegisterForm.tsx
import GoogleIcon from "@/components/icons/GoogleIcon";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";
import { PasswordInput } from "@/components/ui/password-input";
import { Separator } from "@/components/ui/separator";
import { Link } from "react-router";

export default function RegisterForm() {
  return (
    <>
      <h1 className="text-3xl font-bold text-foreground">Create your account</h1>
      <p className="mt-2 text-sm text-muted-foreground">Sign up to start booking your tours.</p>

      <Button variant="outline" className="mt-6 w-full gap-2" type="button">
        <GoogleIcon />
        Sign up with Google
      </Button>

      <div className="my-5 flex items-center gap-3">
        <Separator className="flex-1" />
        <span className="text-xs text-muted-foreground">or sign up with email</span>
        <Separator className="flex-1" />
      </div>

      {/* পরে React Hook Form-এর handleSubmit দিয়ে বদলানো হবে */}
      <form className="space-y-5" onSubmit={(e) => e.preventDefault()}>
        <div className="space-y-2">
          <Label htmlFor="name">Full name</Label>
          <Input id="name" name="name" type="text" autoComplete="name" placeholder="Your full name" required />
        </div>

        <div className="space-y-2">
          <Label htmlFor="email">Email</Label>
          <Input id="email" name="email" type="email" autoComplete="email" placeholder="you@example.com" required />
        </div>

        <div className="space-y-2">
          <Label htmlFor="password">Password</Label>
          <PasswordInput
            id="password"
            name="password"
            autoComplete="new-password"
            placeholder="At least 8 characters"
            minLength={8}
            required
          />
        </div>

        <div className="space-y-2">
          <Label htmlFor="confirm-password">Confirm password</Label>
          <PasswordInput
            id="confirm-password"
            name="confirmPassword"
            autoComplete="new-password"
            placeholder="Re-enter password"
            required
          />
        </div>

        <Button type="submit" className="w-full">
          Create account
        </Button>
      </form>

      <p className="mt-4 text-center text-xs text-muted-foreground">
        By signing up, you agree to our{" "}
        <Link to="/terms" className="underline underline-offset-4 hover:text-foreground">
          Terms &amp; Privacy
        </Link>
        .
      </p>

      <p className="mt-4 text-center text-sm text-muted-foreground">
        Already have an account?{" "}
        <Link to="/login" className="font-medium text-primary underline underline-offset-4">
          Log in
        </Link>
      </p>
    </>
  );
}
```

- Password field দুটোতেই এখন আলাদা আলাদা কাজ করা চোখের button আছে, কারণ প্রতিটা `PasswordInput`-এর নিজের `useState`।
- `minLength={8}` → "At least 8 characters" লেখাটা এখন browser নিজেও যাচাই করে।
- দুই password মিলছে কিনা, সেটা browser নিজে যাচাই করতে পারে না। এটা পরে React Hook Form + Zod দিয়ে করা হবে।

### ৫.৫ Page

```tsx
// src/pages/Login.tsx
import AuthLayout from "@/components/layout/AuthLayout";
import LoginForm from "@/components/modules/Authentication/LoginForm";

export default function Login() {
  return (
    <AuthLayout
      title={<>Your next trip is a few<br />taps away.</>}
      description="Log in to book tours, pay securely and keep every invoice in one place."
    >
      <LoginForm />
    </AuthLayout>
  );
}
```

```tsx
// src/pages/Register.tsx
import AuthLayout from "@/components/layout/AuthLayout";
import RegisterForm from "@/components/modules/Authentication/RegisterForm";

export default function Register() {
  return (
    <AuthLayout
      title={<>Start exploring<br />Bangladesh with us.</>}
      description="Create an account to book tours, pay securely and keep every invoice in one place."
    >
      <RegisterForm />
    </AuthLayout>
  );
}
```

Page ফাইল এখন মাত্র কয়েক লাইনের। শুধু বলে দেয় কোন লেখা আর কোন form বসবে।

---

## ৬. Route

এই ডিজাইনে **Navbar আর Footer নেই**, পুরো স্ক্রিন জুড়ে নিজের কাঠামো। তাই Login আর Register-কে `App`-এর (CommonLayout) `children`-এ রাখলে চলবে না, তাহলে উপরে Navbar আর নিচে Footer চলে আসবে। এদের রাখতে হবে **top level-এ**, `/`-এর পাশে (sibling):

```tsx
// src/routes/index.tsx
import App from "@/App";
import Login from "@/pages/Login";
import Register from "@/pages/Register";
import NotFound from "@/pages/NotFound";
import { createBrowserRouter } from "react-router";

export const router = createBrowserRouter([
  {
    path: "/",
    Component: App,              // Navbar + Footer সহ
    children: [
      // Home, About, Tours …
    ],
  },
  { path: "/login", Component: Login },         // Navbar/Footer ছাড়া
  { path: "/register", Component: Register },   // Navbar/Footer ছাড়া
  { path: "*", Component: NotFound },
]);
```

(৫ নম্বর নোটের "Sibling বনাম Nested route" অংশের বাস্তব উদাহরণ এটা।)

Navbar-এর "Log in" (`/login`) আর "Sign up" (`/register`) এখন সঠিক page খুলবে। আর ছবির ওপরের Logo-তে click করলে Home-এ ফেরা যাবে।

---

## ৭. Check-list

কোড বসানোর পর একবার দেখে নিন:

- [ ] Browser **Console**-এ কোনো warning নেই (বিশেষ করে `<a>` বা `<button>`-এর ভেতরে আরেকটা, ১০ নম্বর নোট)
- [ ] চোখের icon চাপলে password দেখায়/লুকায়, আর form submit হয় না
- [ ] "Log in" / "Create account" চাপলে page reload হয় না
- [ ] **Dark mode**-এ (`html`-এ `dark` class বা ModeToggle) লেখা, Logo আর input সব পড়া যায়
- [ ] Mobile width-এ (DevTools-এর device toolbar, `Ctrl+Shift+M`) ছবি লুকায়, উপরে Logo দেখায়
- [ ] "Create an account" → `/register`, "Log in" → `/login` ঠিক page খোলে