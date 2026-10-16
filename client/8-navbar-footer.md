# Frontend Notes — ০৯: Header (Navbar) ও Footer

Design canvas থেকে Claude-এর সাহায্যে Phovia Tour-এর Header আর Footer বানানো হয়েছে। এই নোটে কোড, প্রতিটা অংশের ব্যাখ্যা, কিছু সংশোধন আর "footer উপরে উঠে আসা" সমস্যার সমাধান আছে।

ফাইলগুলো থাকবে:

```
src/components/
├── Logo.tsx                ← Header আর Footer দুই জায়গায় ব্যবহৃত
└── layout/
    ├── CommonLayout.tsx
    ├── Navbar.tsx
    └── Footer.tsx
```

> সব জায়গায় রং দেওয়া হয়েছে **semantic token** দিয়ে (`text-foreground`, `bg-background`, `fill-primary`) — কোথাও `text-black`/`bg-white` নেই। তাই dark mode-এ নিজে থেকেই ঠিক দেখাবে (৮ নম্বর নোটের notice দেখুন)।

---

## ১. Logo — আলাদা component

Header আর Footer দুই জায়গাতেই একই logo (map pin icon + "Phovia Tour")। একই কোড দুইবার লিখলে পরে logo বদলাতে হলে দুই জায়গায় বদলাতে হবে। তাই logo-কে আলাদা component বানিয়ে দুই জায়গা থেকে import করব।

> ⚠️ **সংশোধন — ফাইলের জায়গা:** নোটে লেখা ছিল `src/assets/src/Logo.tsx`। এটা ঠিক জায়গা না:
> - `assets/` folder **static ফাইলের** জন্য (ছবি, `.svg` ফাইল) — React component (`.tsx`) রাখার জন্য না (৬ নম্বর নোট দেখুন)।
> - `assets/src/` — এমন কোনো folder আমাদের structure-এ নেই।
>
> Logo একটা reusable **component**, তাই এটা যাবে **`src/components/Logo.tsx`**-এ।

> **কেন `.svg` ফাইল না, inline SVG?** Logo-র রং `fill-primary` আর `fill-background` — এগুলো theme token, dark mode-এ বদলায়। `<img src="logo.svg">` দিয়ে দিলে SVG-র ভেতরে Tailwind class কাজ করে না, তাই রং বদলাবে না। Inline SVG (JSX-এর ভেতরে `<svg>`) দিলে class কাজ করে।

Header-এ logo একটু বড় (26×30, `text-[22px]`), Footer-এ একটু ছোট (24×28, `text-xl`)। তাই `size` prop দিয়ে দুটোই সামলানো যায়:

```tsx
// src/components/Logo.tsx
import { cn } from "@/lib/utils";
import { Link } from "react-router";

const sizes = {
  md: { width: 26, height: 30, text: "text-[22px] tracking-tight" }, // Header
  sm: { width: 24, height: 28, text: "text-xl" },                     // Footer
};

interface LogoProps {
  size?: keyof typeof sizes;
  className?: string;
}

export default function Logo({ size = "md", className }: LogoProps) {
  const s = sizes[size];

  return (
    <Link to="/" className={cn("flex items-center gap-2 text-foreground", className)}>
      <svg width={s.width} height={s.height} viewBox="0 0 26 30" aria-hidden="true">
        <path
          d="M13 0C5.8 0 0 5.6 0 12.6 0 21.7 13 30 13 30s13-8.3 13-17.4C26 5.6 20.2 0 13 0z"
          className="fill-primary"
        />
        <circle cx="13" cy="12.5" r="5" className="fill-background" />
      </svg>
      <span className={cn("font-bold", s.text)}>Phovia Tour</span>
    </Link>
  );
}
```

ব্যবহার:

```tsx
<Logo />            {/* Header — default md */}
<Logo size="sm" />  {/* Footer */}
```

- `aria-hidden="true"` — SVG-টা শুধু সাজসজ্জা, পাশে "Phovia Tour" লেখা আছে। তাই screen reader-কে SVG পড়তে হবে না।
- `viewBox` একই থাকায় width/height বদলালে icon শুধু ছোট-বড় হয়, আকার বিকৃত হয় না।

---

## ২. Navbar (Header)

### Design-এর গঠন

```
Desktop (lg ও বড়):
[Logo]      Home  Tours  Divisions  Short tours  Long tours      Log in  [Sign up]

Tablet (sm–lg):
[Logo]                                             Log in  [Sign up]  [☰]

Mobile (sm-এর ছোট):
[Logo]                                                                [☰]
```

`☰` চাপলে ডান দিক থেকে একটা **Sheet** (slide-in panel) খোলে, তাতে সব link আর mobile-এ Log in / Sign up button।

Sheet component আগে add করে নিতে হবে:

```sh
bunx --bun shadcn@latest add sheet
```

### কোড

```tsx
// src/components/layout/Navbar.tsx
import Logo from "@/components/Logo";
import { buttonVariants } from "@/components/ui/button";
import { Sheet, SheetContent, SheetHeader, SheetTitle, SheetTrigger } from "@/components/ui/sheet";
import { cn } from "@/lib/utils";
import { Menu } from "lucide-react";
import { useState } from "react";
import { Link, useLocation } from "react-router";

const navLinks = [
  { label: "Home", href: "/" },
  { label: "Tours", href: "/tours" },
  { label: "Divisions", href: "/divisions" },
  { label: "Short tours", href: "/tours?type=short" },
  { label: "Long tours", href: "/tours?type=long" },
];

const primaryBtn = cn(
  buttonVariants(),
  "h-11 rounded-lg px-[22px] text-[14.5px] font-medium shadow-lg shadow-primary/25 hover:bg-primary-hover",
);

const outlineBtn = cn(
  buttonVariants({ variant: "outline" }),
  "h-11 rounded-lg border-primary text-[14.5px] font-medium text-primary hover:bg-accent hover:text-accent-foreground",
);

export default function Navbar() {
  const [open, setOpen] = useState(false);
  const { pathname, search } = useLocation();

  // "/tours?type=short" আর "/tours" আলাদা করে ধরার জন্য path + query দুটোই মেলানো হচ্ছে
  const isActive = (href: string) => pathname + search === href;

  return (
    <header className="w-full bg-background">
      <div className="mx-auto flex h-21 max-w-300 items-center justify-between gap-6 px-4 sm:px-10">
        <Logo />

        {/* Desktop nav */}
        <nav aria-label="Main" className="hidden items-center gap-9 lg:flex">
          {navLinks.map((link) => (
            <Link
              key={link.label}
              to={link.href}
              aria-current={isActive(link.href) ? "page" : undefined}
              className={cn(
                "py-1.5 text-[14.5px] font-medium text-foreground transition-colors hover:text-primary",
                isActive(link.href) && "text-primary",
              )}
            >
              {link.label}
            </Link>
          ))}
        </nav>

        {/* Right side */}
        <div className="flex items-center gap-3.5">
          <Link
            to="/login"
            className="hidden py-1.5 text-[14.5px] font-medium text-foreground transition-colors hover:text-primary sm:inline-block"
          >
            Log in
          </Link>
          <Link to="/register" className={cn(primaryBtn, "hidden sm:inline-flex")}>
            Sign up
          </Link>

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
                  <Link
                    key={link.label}
                    to={link.href}
                    onClick={() => setOpen(false)}
                    aria-current={isActive(link.href) ? "page" : undefined}
                    className={cn(
                      "rounded-lg px-3 py-2.5 text-[15px] font-medium text-foreground transition-colors hover:bg-accent hover:text-accent-foreground",
                      isActive(link.href) && "bg-accent text-accent-foreground",
                    )}
                  >
                    {link.label}
                  </Link>
                ))}
              </nav>

              <div className="mt-4 flex flex-col gap-3 px-4 sm:hidden">
                <Link to="/login" onClick={() => setOpen(false)} className={outlineBtn}>
                  Log in
                </Link>
                <Link to="/register" onClick={() => setOpen(false)} className={primaryBtn}>
                  Sign up
                </Link>
              </div>
            </SheetContent>
          </Sheet>
        </div>
      </div>
    </header>
  );
}
```

### কোডের ব্যাখ্যা

**`navLinks` array:** link-গুলো JSX-এ বারবার না লিখে একটা array-তে রেখে `.map()` দিয়ে render করা হয়েছে। নতুন link যোগ করতে শুধু array-তে একটা লাইন যোগ করলেই desktop আর mobile দুই জায়গায় চলে আসে।

**`buttonVariants()` + `<Link>`:** "Sign up" দেখতে button-এর মতো, কিন্তু কাজ করে অন্য page-এ যাওয়ার link হিসেবে। তাই `<Button>` না দিয়ে `<Link>`-এ shadcn-এর `buttonVariants()` class বসানো হয়েছে — দেখতে button, আচরণে link। (Page বদলানোর জন্য সবসময় link, কোনো কাজ করার জন্য — যেমন form submit — button।)

**`cn()`:** shadcn-এর default class-এর সাথে নিজের class মেলায়। Conflict হলে (যেমন default `h-9` আর আমাদের `h-11`) শেষেরটা রাখে।

**Responsive class:**

| Class | মানে |
|---|---|
| `hidden lg:flex` | ছোট স্ক্রিনে লুকানো, `lg` (1024px) থেকে দেখাবে — desktop nav |
| `lg:hidden` | `lg` থেকে লুকাবে — ☰ button শুধু ছোট স্ক্রিনে |
| `hidden sm:inline-flex` | `sm` (640px) থেকে দেখাবে — header-এর Sign up |
| `sm:hidden` | `sm` থেকে লুকাবে — Sheet-এর ভেতরের Log in/Sign up শুধু mobile-এ |

**Tailwind v4-এর সংখ্যা class:** `h-21`, `max-w-300`, `w-75`, `size-5.5` — Tailwind v4-এ যেকোনো সংখ্যা দেওয়া যায়, প্রতিটা একক = `0.25rem` (4px)। যেমন `h-21` = 84px, `max-w-300` = 1200px। v3-এ এগুলোর জন্য `h-[84px]` লিখতে হতো।

**Accessibility:**
- `aria-label="Main"` / `"Mobile"` — দুটো `<nav>` থাকায় screen reader আলাদা করতে পারে।
- `aria-label="Open menu"` — ☰ button-এ শুধু icon, লেখা নেই, তাই screen reader-এর জন্য নাম।
- `aria-current="page"` — বর্তমান page-এর link চিহ্নিত করে।
- `SheetTitle` — dialog-এর একটা title থাকা বাধ্যতামূলক (accessibility), তাই সেখানে Logo রাখা হয়েছে।

### আপনার দেওয়া কোডে যা ঠিক/যোগ করা হয়েছে

> ⚠️ **সংশোধন ১ — Mobile menu-তে link চাপলে Sheet বন্ধ হচ্ছিল না।** আগের কোডে Sheet নিজে নিজে খোলা/বন্ধ হতো (uncontrolled)। Link চাপলে page বদলায়, কিন্তু Sheet খোলাই থেকে যায় — user-কে হাত দিয়ে বন্ধ করতে হয়। সমাধান: `useState` দিয়ে Sheet-কে **controlled** করা (`open={open} onOpenChange={setOpen}`), আর প্রতিটা link-এ `onClick={() => setOpen(false)}`।

> ⚠️ **সংশোধন ২ — নাম মিলছিল না।** ফাইলের নাম `Navbar.tsx` কিন্তু component-এর নাম ছিল `Header`। Default export হওয়ায় কাজ করে, কিন্তু খুঁজতে গেলে বিভ্রান্তি হয়। ফাইল আর component-এর নাম একই রাখা নিয়ম — তাই `Navbar`।

> ⚠️ **সংশোধন ৩ — Logo ভেতরে না রেখে import।** আগের কোডে `Logo` function Navbar-এর ভেতরেই ছিল, Footer-এ আবার আলাদা করে লেখা। এখন `@/components/Logo` থেকে import।

> ➕ **সংযোজন — Active link।** বর্তমান page-এর link `text-primary` রঙে দেখাবে। এজন্য `useLocation()` দিয়ে বর্তমান URL নিয়ে মেলানো হয়েছে।
>
> **কেন `NavLink` না?** React Router-এর `<NavLink>` নিজেই active বোঝে, কিন্তু শুধু **path** দেখে, query (`?type=short`) দেখে না। ফলে `/tours?type=short`-এ গেলে "Tours", "Short tours", "Long tours" — তিনটাই active দেখাত। তাই এখানে path + query দুটোই মেলানো হয়েছে। Query ছাড়া সাধারণ link হলে `NavLink` সহজ উপায় — নিচে দেখুন।

### Active link — বিকল্প উপায়: `NavLink` (design-এর মতো নিচে dot সহ)

Design-এ active link-এর **নিচে একটা ছোট বেগুনি dot** আছে। `NavLink` দিয়ে এটা বানাতে Navbar-এ শুধু তিনটা জিনিস বদলাতে হবে।

**১. `Link`-এর জায়গায় `NavLink` import:**

```tsx
import { NavLink } from "react-router";
```

(Logo, Log in, Sign up-এর জন্য `Link` থাকছে — তাই দুটোই import থাকবে: `import { Link, NavLink } from "react-router";`)

**২. Desktop nav link:**

```tsx
<NavLink
  key={link.label}
  to={link.href}
  end={link.href === "/"}
  className="relative py-1.5 text-[14.5px] font-medium text-foreground transition-colors hover:text-primary after:absolute after:-bottom-1 after:left-1/2 after:size-[5px] after:-translate-x-1/2 after:rounded-full after:bg-primary after:opacity-0 aria-[current=page]:after:opacity-100"
>
  {link.label}
</NavLink>
```

Active state-এর জন্য নতুন অংশটুকু:

```
relative after:absolute after:-bottom-1 after:left-1/2 after:size-[5px] after:-translate-x-1/2 after:rounded-full after:bg-primary after:opacity-0 aria-[current=page]:after:opacity-100
```

| Class | কাজ |
|---|---|
| `relative` | dot-এর অবস্থান link-এর সাপেক্ষে ঠিক করার জন্য |
| `after:absolute` | `::after` দিয়ে একটা আলাদা element (dot) বানানো, যেটা link-এর ওপর ভাসবে |
| `after:-bottom-1` | link-এর নিচে ৪px দূরে |
| `after:left-1/2 after:-translate-x-1/2` | ঠিক মাঝখানে |
| `after:size-[5px] after:rounded-full` | ৫px-এর গোল dot |
| `after:bg-primary` | বেগুনি রং (dark mode-এ নিজে থেকে বদলাবে) |
| `after:opacity-0` | সাধারণভাবে dot লুকানো |
| `aria-[current=page]:after:opacity-100` | **শুধু active link-এ** dot দেখাবে |

> Tailwind v4-এ `after:` দিলে `content: ""` নিজে থেকেই বসে যায়, আলাদা করে `after:content-['']` লিখতে হয় না।

**৩. Mobile menu link** — `Link`-কে `NavLink` বানিয়ে `className`-এর শেষে যোগ করুন:

```
aria-[current=page]:bg-accent aria-[current=page]:text-accent-foreground
```

```tsx
<NavLink
  key={link.label}
  to={link.href}
  end={link.href === "/"}
  onClick={() => setOpen(false)}
  className="rounded-lg px-3 py-2.5 text-[15px] font-medium text-foreground transition-colors hover:bg-accent hover:text-accent-foreground aria-[current=page]:bg-accent aria-[current=page]:text-accent-foreground"
>
  {link.label}
</NavLink>
```

এই উপায়ে `useLocation()`, `isActive()` আর হাতে লেখা `aria-current` আর লাগে না — মুছে দিন।

#### কীভাবে কাজ করে

- **`aria-current="page"`:** `NavLink` active link-এ নিজে থেকেই এই attribute বসিয়ে দেয়। তাই `aria-[current=page]:` দিয়ে লেখা class-গুলো **শুধু active link-এই** কাজ করে। একই সাথে screen reader-ও জানতে পারে কোনটা বর্তমান page — style আর accessibility একসাথে।
- **`end`:** `end` দিলে link **শুধু ঠিক ওই path-এ** active হয়। `end` না দিলে নিচের সব path-এও active হয় — যেমন `"/"` সব path-এর শুরুতেই আছে, তাই Home-এ `end` না দিলে **সব page-এই Home active** দেখাবে।

> বিকল্প লেখার ধরন: `className`-এ function দিয়েও active style দেওয়া যায় — `className={({ isActive }) => cn("...", isActive && "text-primary")}`। তবে `aria-[current=page]:` পদ্ধতি ছোট আর শুধু class-এই হয়ে যায়।

#### ⚠️ Query link-এর সমস্যা

Tours, Short tours, Long tours যদি একই `/tours` path-এ `?type=` query দিয়ে থাকে (`/tours`, `/tours?type=short`, `/tours?type=long`), তাহলে **তিনটা একসাথে active** দেখাবে — কারণ `NavLink` query দেখে না, তিনটার path-ই `/tours`।

সমাধান — দুটোর যেকোনো একটা:

1. **আলাদা route বানানো (NavLink রাখতে চাইলে):** `/tours/short` আর `/tours/long` আলাদা path করুন, আর "Tours"-এও `end` দিন (নাহলে `/tours/short`-এ গেলে Tours-ও active হবে, কারণ `/tours` তার শুরুতে আছে):

   ```tsx
   const navLinks = [
     { label: "Home", href: "/" },
     { label: "Tours", href: "/tours" },
     { label: "Divisions", href: "/divisions" },
     { label: "Short tours", href: "/tours/short" },
     { label: "Long tours", href: "/tours/long" },
   ];

   // end: Home আর Tours দুটোতেই
   end={link.href === "/" || link.href === "/tours"}
   ```

2. **Query রাখতে চাইলে:** উপরের `useLocation()` + `isActive()` পদ্ধতি — সেটা path আর query দুটোই মেলায়।

| | `useLocation()` + `isActive()` | `NavLink` |
|---|---|---|
| Query (`?type=`) আলাদা করে | ✅ হ্যাঁ | ❌ না |
| কোড | একটু বেশি | কম |
| `aria-current` | হাতে লিখতে হয় | নিজে থেকে বসে |
| কখন | link-এ query থাকলে | সব link আলাদা path হলে |

---

## ৩. Footer

### কোড

```tsx
// src/components/layout/Footer.tsx
import Logo from "@/components/Logo";
import { Link } from "react-router";

const footerSections = [
  {
    title: "Explore",
    links: [
      { label: "All tours", href: "/tours" },
      { label: "Short tours", href: "/tours?type=short" },
      { label: "Long tours", href: "/tours?type=long" },
      { label: "Divisions", href: "/divisions" },
    ],
  },
  {
    title: "Company",
    links: [
      { label: "About us", href: "/about" },
      { label: "Become a guide", href: "/become-a-guide" },
      { label: "Contact", href: "/contact" },
    ],
  },
  {
    title: "Support",
    links: [
      { label: "Help centre", href: "/help" },
      { label: "Cancellation policy", href: "/cancellation-policy" },
    ],
    contact: ["+880 1XXX-XXXXXX", "support@phoviatour.com"],
  },
];

const linkClass = "text-sm text-muted-foreground transition-colors hover:text-primary";

export default function Footer() {
  const year = new Date().getFullYear();

  return (
    <footer className="w-full bg-background pb-7 pt-20">
      <div className="mx-auto flex max-w-300 flex-col gap-14 px-4 sm:px-10">
        <div className="grid grid-cols-1 gap-9 sm:grid-cols-2 lg:grid-cols-4">
          {/* Brand */}
          <div className="flex flex-col gap-3.5">
            <Logo size="sm" />
            <p className="max-w-65 text-sm text-muted-foreground">
              Guided tours across Bangladesh, booked and paid online in taka.
            </p>
          </div>

          {/* Link columns */}
          {footerSections.map((section) => (
            <nav key={section.title} aria-label={section.title} className="flex flex-col gap-3">
              <h2 className="text-[15px] font-semibold text-foreground">{section.title}</h2>
              {section.links.map((link) => (
                <Link key={link.label} to={link.href} className={linkClass}>
                  {link.label}
                </Link>
              ))}
              {section.contact?.map((item) => (
                <span key={item} className="text-sm text-muted-foreground">
                  {item}
                </span>
              ))}
            </nav>
          ))}
        </div>

        {/* Bottom bar */}
        <div className="flex flex-col gap-3 border-t border-border pt-5.5 text-[13px] sm:flex-row sm:items-center sm:justify-between">
          <span className="text-muted-foreground">© {year} Phovia Tour. All rights reserved.</span>
          <div className="flex gap-6">
            <Link to="/terms" className="text-muted-foreground transition-colors hover:text-primary">
              Terms of service
            </Link>
            <Link to="/privacy" className="text-muted-foreground transition-colors hover:text-primary">
              Privacy policy
            </Link>
          </div>
        </div>
      </div>
    </footer>
  );
}
```

### কোডের ব্যাখ্যা

**Grid layout:** `grid-cols-1 sm:grid-cols-2 lg:grid-cols-4` — mobile-এ সব column একটার নিচে আরেকটা, tablet-এ ২টা করে পাশাপাশি, desktop-এ ৪টা (Brand + ৩টা link column) এক সারিতে।

**`footerSections` array:** Navbar-এর মতোই — column আর link array-তে, `.map()` দিয়ে render। নতুন column বা link যোগ করা সহজ।

**Bottom bar:** `border-t border-border` দিয়ে উপরে একটা line, mobile-এ copyright আর link-গুলো একটার নিচে আরেকটা (`flex-col`), বড় স্ক্রিনে দুই প্রান্তে (`sm:flex-row sm:justify-between`)।

### আপনার দেওয়া কোডে যা ঠিক/যোগ করা হয়েছে

> ⚠️ **সংশোধন ১ — Section-এর নাম দিয়ে শর্ত।** আগের কোডে ছিল `section.title === "Support" && (...)` — phone আর email শুধু "Support" নামের section-এ দেখানোর জন্য। সমস্যা: section-এর নাম কখনো বদলালে (যেমন "Help") contact তথ্য চুপচাপ হারিয়ে যাবে, কোনো error-ও দেবে না। সমাধান: contact তথ্য **data-র ভেতরেই** রাখা (`contact: [...]`), আর `section.contact?.map(...)` দিয়ে render। `?.` মানে — `contact` না থাকলে কিছুই করবে না।

> ⚠️ **সংশোধন ২ — সাল hard-coded।** `© 2026` লেখা থাকলে ২০২৭ সালে হাতে বদলাতে হবে। `new Date().getFullYear()` নিজে থেকেই বর্তমান সাল দেয়।

> ⚠️ **সংশোধন ৩ — `<strong>`-এর বদলে `<h2>`।** Column-এর শিরোনাম ("Explore", "Company") আসলে heading। `<strong>` শুধু লেখা মোটা করে; `<h2>` দিলে screen reader ব্যবহারকারী heading ধরে ধরে page-এ ঘুরতে পারে। দেখতে একই থাকবে, কারণ size/weight class দিয়ে ঠিক করা।

> ➕ **Logo এখন `<Logo size="sm" />`** — আগে Footer-এ পুরো SVG আবার লেখা ছিল।

> নোট: Bottom bar-এর link-গুলোতে `text-[13px]` আলাদা করে দেওয়ার দরকার নেই — parent `div`-এ আগেই `text-[13px]` আছে, ভেতরের সবাই সেটা পায়।

---

## ৪. Footer উপরে উঠে আসার সমস্যা — Sticky Footer

### সমস্যা

যে page-এ content কম (যেমন 404 বা Login page), সেখানে footer content-এর ঠিক নিচে বসে পড়ে — অর্থাৎ স্ক্রিনের মাঝখানে উঠে আসে, আর নিচে খালি জায়গা থাকে। দেখতে খুব বাজে লাগে।

```
সমস্যা:                      সমাধানের পরে:
┌──────────────┐            ┌──────────────┐
│ Navbar       │            │ Navbar       │
│ অল্প content │            │ অল্প content │
│ Footer       │ ← উপরে     │              │
│              │            │   (grow)     │
│  (খালি)      │            │              │
│              │            │ Footer       │ ← একদম নিচে
└──────────────┘            └──────────────┘
```

### সমাধান: `CommonLayout.tsx`

```tsx
// src/components/layout/CommonLayout.tsx
import type { ReactNode } from "react";
import Footer from "./Footer";
import Navbar from "./Navbar";

interface IProps {
  children: ReactNode;
}

export default function CommonLayout({ children }: IProps) {
  return (
    <div className="flex min-h-dvh flex-col">
      <Navbar />
      <main className="grow">{children}</main>
      <Footer />
    </div>
  );
}
```

### কীভাবে কাজ করে — ধাপে ধাপে

1. **`min-h-dvh`** (বা `min-h-screen`) — বাইরের `div`-কে **কমপক্ষে** পুরো স্ক্রিনের সমান উঁচু করে।
2. **`flex flex-col`** — ভেতরের তিনটা অংশ (Navbar, main, Footer) উপর থেকে নিচে সাজায়।
3. শুধু এই দুটো দিলে **কোনো পরিবর্তন দেখা যায় না** — কারণ div বড় হলেও ভেতরের অংশগুলো নিজেদের আকারেই থাকে, বাড়তি জায়গা নিচে খালি পড়ে থাকে।
4. **`grow`** (মাঝের `main`-এ) — এটাই আসল চাবি। `grow` মানে "বাড়তি যত খালি জায়গা আছে, সব আমি নেব"। ফলে `main` লম্বা হয়ে সব খালি জায়গা দখল করে, আর Footer ঠেলে একদম নিচে চলে যায়।

Content বেশি হলে (স্ক্রিনের চেয়ে লম্বা) কোনো খালি জায়গা থাকে না, `grow`-এর কিছু করার থাকে না — page স্বাভাবিকভাবে scroll হয়, footer content-এর পরে থাকে। তাই সব ক্ষেত্রেই ঠিক কাজ করে।

> ✅ নোটে লেখা কারণটা একদম ঠিক ছিল — `min-h-screen` আর `flex flex-col`-এ পরিবর্তন দেখা যায় না, `grow` দিলেই সমস্যা মেটে।

> ⚠️ **সংশোধন — `<div>`-এর বদলে `<main>`:** নোটে মাঝখানে `<div className="grow">` ছিল। Page-এর মূল content-এর জন্য HTML-এর নিজস্ব **`<main>`** tag আছে — screen reader আর search engine এতে বোঝে কোনটা আসল content। দেখতে কোনো পার্থক্য নেই। (৫ নম্বর নোটেও `<main>` ব্যবহার করা হয়েছে।)

> 💡 **`min-h-screen` না `min-h-dvh`?** Mobile browser-এ address bar দেখা দেয় আর লুকায়, তাই `100vh` (`min-h-screen`) আসল দৃশ্যমান উচ্চতার চেয়ে বড় হয় — ফলে footer একটু নিচে কেটে যায়, অকারণে scroll হয়। **`min-h-dvh`** (dynamic viewport height) address bar সহ আসল উচ্চতা নেয়। Desktop-এ দুটো একই।

> নোট: Tailwind-এ `grow` = `flex-grow: 1`। `flex-1`-ও প্রায় একই কাজ করে। পুরনো লেখা `grow-1`-এর দরকার নেই।