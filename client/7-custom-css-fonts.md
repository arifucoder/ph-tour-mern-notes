# Frontend Notes — ০৮: Custom CSS Color (Phovia Theme)

Phovia Tour-এর ডিজাইনে যে রংগুলো ব্যবহার হয়েছে, সেগুলো Tailwind class হিসেবে (`bg-phovia-primary`, `text-phovia-ink` …) ব্যবহার করার জন্য `src/index.css`-এ custom CSS variable যোগ করব।

> ডিজাইনের সব রং নেওয়া হয়েছে Phovia Tour design canvas থেকে (Home, Admin Overview আর States artboard)।

> ✅ **আমাদের প্রজেক্টে যেটা ব্যবহার করছি:** নোটের শেষে **৮ নম্বর section** — shadcn-এর token-এ সরাসরি Phovia-র রং বসানো। ১–৬ নম্বর section-এ `phovia-` prefix দিয়ে আলাদা variable বানানোর পদ্ধতিটা ধারণা বোঝার জন্য রাখা হলো।

---

## ১. মূল ধারণা — ৩টা অংশ

```css
:root {
  --phovia-primary: #5B4CF0;      /* ১. light mode-এর রং */
}

.dark {
  --phovia-primary: #9D92FF;      /* ২. dark mode-এর রং */
}

@theme inline {
  --color-phovia-primary: var(--phovia-primary);   /* ৩. Tailwind-কে জানানো */
}
```

| অংশ | কাজ |
|---|---|
| `:root` | সাধারণ (light) mode-এ variable-এর মান |
| `.dark` | `<html class="dark">` থাকলে একই variable-এর নতুন মান — dark mode |
| `@theme inline` | Tailwind v4-কে বলে: `--color-phovia-primary` নামে একটা color আছে। এতে `bg-phovia-primary`, `text-phovia-primary`, `border-phovia-primary` ইত্যাদি class তৈরি হয় |

**নামের নিয়ম:** `@theme`-এ `--color-` দিয়ে শুরু হওয়া variable-গুলো Tailwind color হিসেবে ধরে। `--color-phovia-primary` → class-এ `-color` বাদ দিয়ে লেখা হয় `bg-phovia-primary`।

**`inline` কেন?** `inline` দিলে Tailwind সরাসরি `var(--phovia-primary)` বসিয়ে দেয়। ফলে `.dark` class যোগ হলে variable-এর মান বদলায়, আর class-এর রংও নিজে থেকে বদলে যায় — আলাদা `dark:` class লিখতে হয় না।

**`phovia-` prefix কেন?** shadcn `index.css`-এ আগেই `--primary`, `--muted`, `--background` ইত্যাদি নাম ব্যবহার করে। একই নাম দিলে shadcn-এর রং overwrite হয়ে যাবে। `phovia-` prefix দিলে কোনো সংঘর্ষ হয় না।

> নোট: আপনার দেওয়া snippet-টা সঠিক ছিল। শুধু `@theme inline`-এর বন্ধনী `}`-এর পরে কিছু লেখা যাবে না — প্রতিটা block আলাদা লাইনে রাখুন।

---

## ২. কোথায় বসাবেন

`src/index.css`-এ shadcn init যা যোগ করেছে (`@import`, `@custom-variant dark`, `@theme inline`, `:root`, `.dark`, `@layer base`) — **সেগুলোর একদম নিচে** নিচের পুরো কোড বসান। shadcn-এর block-গুলোতে হাত দেবেন না।

> একই ফাইলে একাধিক `:root` বা `@theme inline` block থাকলে সমস্যা নেই — browser আর Tailwind সব মিলিয়ে নেয়।

---

## ৩. পুরো কোড — `src/index.css`-এর শেষে

```css
/* ============================================
   Phovia Tour — custom colors
   ============================================ */

:root {
  /* Background & surface */
  --phovia-bg: #FFFFFF;
  --phovia-surface: #FFFFFF;
  --phovia-surface2: #F7F6FD;
  --phovia-soft: #FFF3F5;
  --phovia-line: #ECEAF5;
  --phovia-grid: #EEECF6;

  /* Text */
  --phovia-ink: #1C1A3A;
  --phovia-muted: #6A6785;

  /* Brand */
  --phovia-primary: #5B4CF0;
  --phovia-primary-hover: #4A3BD9;
  --phovia-primary-ink: #FFFFFF;
  --phovia-primary-soft: #EEEBFF;

  /* Accent (pink) */
  --phovia-accent-soft: #FFE8EC;
  --phovia-accent-text: #B4335A;

  /* Soft tile backgrounds (icon tile, badge) */
  --phovia-pink-soft: #FFE8EC;
  --phovia-yellow-soft: #FFF4DC;
  --phovia-mint-soft: #E3F7EF;

  /* Rating star */
  --phovia-star: #FFB547;

  /* Image placeholder */
  --phovia-ph: #EEEBF8;
  --phovia-ph-ink: #8A86AE;

  /* Status: success / warning / danger / info */
  --phovia-ok: #17714A;
  --phovia-ok-soft: #E2F3EA;
  --phovia-warn: #8F5400;
  --phovia-warn-soft: #FFF0D4;
  --phovia-danger: #B42318;
  --phovia-danger-ink: #FFFFFF;
  --phovia-danger-soft: #FDECEA;
  --phovia-info: #21539E;
  --phovia-info-soft: #E5EDFA;

  /* Booking status (admin badge) */
  --phovia-s-complete: #2E8B57;
  --phovia-s-pending: #D9A21B;
  --phovia-s-cancelled: #9AA3A0;
  --phovia-s-failed: #C8402F;

  /* Chart series */
  --phovia-c1: #6152F0;
  --phovia-c2: #D9771F;

  /* Shadow */
  --phovia-shadow: 0 12px 34px rgba(28, 26, 58, 0.08);
}

.dark {
  /* Background & surface */
  --phovia-bg: #12111F;
  --phovia-surface: #1B1A2E;
  --phovia-surface2: #1F1D35;
  --phovia-soft: #211B2E;
  --phovia-line: #2E2B4A;
  --phovia-grid: #2A2845;

  /* Text */
  --phovia-ink: #EEEDF7;
  --phovia-muted: #A9A6C4;

  /* Brand */
  --phovia-primary: #9D92FF;
  --phovia-primary-hover: #B3AAFF;
  --phovia-primary-ink: #14112E;
  --phovia-primary-soft: #2A2550;

  /* Accent (pink) */
  --phovia-accent-soft: #3A2230;
  --phovia-accent-text: #FF9DB2;

  /* Soft tile backgrounds */
  --phovia-pink-soft: #3A2230;
  --phovia-yellow-soft: #3A3020;
  --phovia-mint-soft: #1D3530;

  /* Rating star — dark-এও একই */
  --phovia-star: #FFB547;

  /* Image placeholder */
  --phovia-ph: #26243F;
  --phovia-ph-ink: #7E7AA6;

  /* Status */
  --phovia-ok: #6DCB98;
  --phovia-ok-soft: #14301F;
  --phovia-warn: #F1C46E;
  --phovia-warn-soft: #382B10;
  --phovia-danger: #F28B80;
  --phovia-danger-ink: #2A0E0B;   /* ডিজাইনে নেই — প্রস্তাবিত */
  --phovia-danger-soft: #3B1916;
  --phovia-info: #9DBBF0;         /* ডিজাইনে নেই — প্রস্তাবিত */
  --phovia-info-soft: #172338;    /* ডিজাইনে নেই — প্রস্তাবিত */

  /* Booking status */
  --phovia-s-complete: #4FB67C;
  --phovia-s-pending: #E2B23A;
  --phovia-s-cancelled: #6E7A77;
  --phovia-s-failed: #E0604E;

  /* Chart series */
  --phovia-c1: #7F73F5;
  --phovia-c2: #D8762A;

  /* Shadow */
  --phovia-shadow: 0 12px 34px rgba(0, 0, 0, 0.35);
}

@theme inline {
  /* Background & surface */
  --color-phovia-bg: var(--phovia-bg);
  --color-phovia-surface: var(--phovia-surface);
  --color-phovia-surface2: var(--phovia-surface2);
  --color-phovia-soft: var(--phovia-soft);
  --color-phovia-line: var(--phovia-line);
  --color-phovia-grid: var(--phovia-grid);

  /* Text */
  --color-phovia-ink: var(--phovia-ink);
  --color-phovia-muted: var(--phovia-muted);

  /* Brand */
  --color-phovia-primary: var(--phovia-primary);
  --color-phovia-primary-hover: var(--phovia-primary-hover);
  --color-phovia-primary-ink: var(--phovia-primary-ink);
  --color-phovia-primary-soft: var(--phovia-primary-soft);

  /* Accent */
  --color-phovia-accent-soft: var(--phovia-accent-soft);
  --color-phovia-accent-text: var(--phovia-accent-text);

  /* Soft tiles */
  --color-phovia-pink-soft: var(--phovia-pink-soft);
  --color-phovia-yellow-soft: var(--phovia-yellow-soft);
  --color-phovia-mint-soft: var(--phovia-mint-soft);

  /* Star & placeholder */
  --color-phovia-star: var(--phovia-star);
  --color-phovia-ph: var(--phovia-ph);
  --color-phovia-ph-ink: var(--phovia-ph-ink);

  /* Status */
  --color-phovia-ok: var(--phovia-ok);
  --color-phovia-ok-soft: var(--phovia-ok-soft);
  --color-phovia-warn: var(--phovia-warn);
  --color-phovia-warn-soft: var(--phovia-warn-soft);
  --color-phovia-danger: var(--phovia-danger);
  --color-phovia-danger-ink: var(--phovia-danger-ink);
  --color-phovia-danger-soft: var(--phovia-danger-soft);
  --color-phovia-info: var(--phovia-info);
  --color-phovia-info-soft: var(--phovia-info-soft);

  /* Booking status */
  --color-phovia-s-complete: var(--phovia-s-complete);
  --color-phovia-s-pending: var(--phovia-s-pending);
  --color-phovia-s-cancelled: var(--phovia-s-cancelled);
  --color-phovia-s-failed: var(--phovia-s-failed);

  /* Chart */
  --color-phovia-c1: var(--phovia-c1);
  --color-phovia-c2: var(--phovia-c2);

  /* Shadow — `--shadow-` দিয়ে শুরু হলে Tailwind shadow class বানায় */
  --shadow-phovia: var(--phovia-shadow);
}
```

---

## ৪. কোন রং কোথায় — রেফারেন্স টেবিল

### Background, surface, text

| Variable | Light | Dark | কোথায় ব্যবহার | Tailwind class |
|---|---|---|---|---|
| `bg` | `#FFFFFF` | `#12111F` | পুরো page-এর background | `bg-phovia-bg` |
| `surface` | `#FFFFFF` | `#1B1A2E` | Card, modal, dropdown | `bg-phovia-surface` |
| `surface2` | `#F7F6FD` | `#1F1D35` | হালকা ধূসর section, admin page-এর background, table header | `bg-phovia-surface2` |
| `soft` | `#FFF3F5` | `#211B2E` | হালকা গোলাপি section background | `bg-phovia-soft` |
| `line` | `#ECEAF5` | `#2E2B4A` | Border, divider | `border-phovia-line` |
| `grid` | `#EEECF6` | `#2A2845` | Chart-এর grid line | `stroke-phovia-grid` |
| `ink` | `#1C1A3A` | `#EEEDF7` | মূল লেখা, heading | `text-phovia-ink` |
| `muted` | `#6A6785` | `#A9A6C4` | কম গুরুত্বপূর্ণ লেখা, caption | `text-phovia-muted` |

### Brand ও accent

| Variable | Light | Dark | কোথায় ব্যবহার | Tailwind class |
|---|---|---|---|---|
| `primary` | `#5B4CF0` | `#9D92FF` | মূল button, link, active icon | `bg-phovia-primary`, `text-phovia-primary` |
| `primary-hover` | `#4A3BD9` | `#B3AAFF` | Primary button hover | `hover:bg-phovia-primary-hover` |
| `primary-ink` | `#FFFFFF` | `#14112E` | Primary button-এর ওপরের লেখা | `text-phovia-primary-ink` |
| `primary-soft` | `#EEEBFF` | `#2A2550` | হালকা বেগুনি badge, outline button hover | `bg-phovia-primary-soft` |
| `accent-soft` | `#FFE8EC` | `#3A2230` | গোলাপি badge background | `bg-phovia-accent-soft` |
| `accent-text` | `#B4335A` | `#FF9DB2` | গোলাপি badge-এর লেখা | `text-phovia-accent-text` |
| `pink-soft` | `#FFE8EC` | `#3A2230` | Icon tile background | `bg-phovia-pink-soft` |
| `yellow-soft` | `#FFF4DC` | `#3A3020` | Icon tile background | `bg-phovia-yellow-soft` |
| `mint-soft` | `#E3F7EF` | `#1D3530` | Icon tile background | `bg-phovia-mint-soft` |
| `star` | `#FFB547` | `#FFB547` | Rating star | `text-phovia-star`, `fill-phovia-star` |
| `ph` | `#EEEBF8` | `#26243F` | ছবি load না হওয়া পর্যন্ত placeholder | `bg-phovia-ph` |
| `ph-ink` | `#8A86AE` | `#7E7AA6` | Placeholder-এর icon/লেখা | `text-phovia-ph-ink` |

### Status (alert, toast, form error)

| Variable | Light | Dark | কোথায় ব্যবহার | Tailwind class |
|---|---|---|---|---|
| `ok` / `ok-soft` | `#17714A` / `#E2F3EA` | `#6DCB98` / `#14301F` | Success message, "Paid" badge | `text-phovia-ok bg-phovia-ok-soft` |
| `warn` / `warn-soft` | `#8F5400` / `#FFF0D4` | `#F1C46E` / `#382B10` | Warning | `text-phovia-warn bg-phovia-warn-soft` |
| `danger` / `danger-soft` | `#B42318` / `#FDECEA` | `#F28B80` / `#3B1916` | Error, delete button, form error | `text-phovia-danger bg-phovia-danger-soft` |
| `danger-ink` | `#FFFFFF` | `#2A0E0B`* | Danger button-এর ওপরের লেখা | `text-phovia-danger-ink` |
| `info` / `info-soft` | `#21539E` / `#E5EDFA` | `#9DBBF0`* / `#172338`* | তথ্যমূলক message | `text-phovia-info bg-phovia-info-soft` |

\* ডিজাইনে এগুলোর dark version নেই — আমি অন্য status রঙের ধাঁচে মিলিয়ে প্রস্তাব করেছি। ডিজাইনে dark version যোগ হলে বদলে নেবেন।

### Booking status ও chart (admin)

| Variable | Light | Dark | কোথায় ব্যবহার |
|---|---|---|---|
| `s-complete` | `#2E8B57` | `#4FB67C` | Complete booking-এর dot/badge |
| `s-pending` | `#D9A21B` | `#E2B23A` | Pending booking |
| `s-cancelled` | `#9AA3A0` | `#6E7A77` | Cancelled booking |
| `s-failed` | `#C8402F` | `#E0604E` | Failed payment |
| `c1` | `#6152F0` | `#7F73F5` | Chart-এর প্রথম series (যেমন revenue) |
| `c2` | `#D9771F` | `#D8762A` | Chart-এর দ্বিতীয় series |

### Shadow

| Variable | Light | Dark | Tailwind class |
|---|---|---|---|
| `shadow` | `0 12px 34px rgba(28,26,58,.08)` | `0 12px 34px rgba(0,0,0,.35)` | `shadow-phovia` |

> নোট: shadow রং না, তাই `@theme`-এ `--color-` না দিয়ে **`--shadow-`** দিয়ে লেখা হয়েছে — এতে Tailwind `shadow-phovia` class বানায়।

> নোট: ডিজাইনে public page-এর background `#FFFFFF`, কিন্তু admin page-এর background `#F7F6FD`। তাই admin layout-এ `bg-phovia-surface2` ব্যবহার করুন।

---

## ৫. ব্যবহারের উদাহরণ

```tsx
{/* Primary button */}
<button className="bg-phovia-primary text-phovia-primary-ink hover:bg-phovia-primary-hover rounded-[10px] px-5 min-h-11">
  Book now
</button>

{/* Card */}
<div className="bg-phovia-surface shadow-phovia rounded-2xl p-6">
  <h3 className="text-phovia-ink font-semibold">Sajek Valley</h3>
  <p className="text-phovia-muted">3 days 2 nights</p>
</div>

{/* Status badge */}
<span className="bg-phovia-ok-soft text-phovia-ok rounded-full px-3 py-1 text-sm">Paid</span>

{/* Opacity-ও কাজ করে */}
<div className="bg-phovia-primary/10">...</div>
```

Dark mode চালু করতে `<html>`-এ `dark` class যোগ করলেই সব রং নিজে থেকে বদলে যাবে:

```ts
document.documentElement.classList.add("dark");     // dark চালু
document.documentElement.classList.remove("dark");  // light-এ ফেরা
```

> shadcn init `index.css`-এ আগেই `@custom-variant dark (&:is(.dark *));` যোগ করে রাখে, তাই `.dark` class পদ্ধতি কাজ করবে।

---

## ৬. (ঐচ্ছিক) shadcn component-কেও Phovia রঙে আনা

উপরের variable-গুলো শুধু আমাদের নিজেদের লেখা class-এ কাজ করে। shadcn-এর `Button`, `Input` ইত্যাদি এখনো shadcn-এর নিজস্ব `--primary`, `--ring` ব্যবহার করে। এগুলোকেও Phovia রঙে আনতে চাইলে, উপরের `:root` আর `.dark`-এর **ভেতরে** shadcn-এর variable-গুলোকে Phovia variable দিয়ে map করে দিন:

```css
:root {
  /* ...উপরের phovia variable-গুলো... */
  --primary: var(--phovia-primary);
  --primary-foreground: var(--phovia-primary-ink);
  --ring: var(--phovia-primary);
}

.dark {
  /* ...উপরের phovia variable-গুলো... */
  --primary: var(--phovia-primary);
  --primary-foreground: var(--phovia-primary-ink);
  --ring: var(--phovia-primary);
}
```

এতে `<Button>` (default variant) Phovia-র বেগুনি রঙে দেখাবে, আর focus ring-ও একই রঙের হবে। এই অংশটা shadcn-এর block-এর **নিচে** থাকতে হবে, নাহলে shadcn-এর মান আবার জিতে যাবে।

---

## ৭. Font

ডিজাইনে font ব্যবহার হয়েছে **Poppins** (ইংরেজি) আর **Hind Siliguri** (বাংলা)।

Font কাজ করাতে দুটো ধাপ লাগে:
1. **Font load করা** — font ফাইল browser-এ আনা। এটা ৩ভাবে করা যায় (নিচে)।
2. **Tailwind-কে জানানো** — `@theme inline`-এ font-এর নাম দেওয়া। এই ধাপ তিন উপায়েই **একই**।

### ধাপ ২ (সবার জন্য একই): `index.css`-এর `@theme inline`-এ

```css
@theme inline {
  --font-phovia: "Poppins", "Hind Siliguri", system-ui, sans-serif;
}
```

ব্যবহার: `<body className="font-phovia">` বা যেকোনো element-এ `font-phovia`।

> `--font-` দিয়ে শুরু হলে Tailwind font class বানায় (`font-phovia`)। ইংরেজি অক্ষর Poppins-এ দেখাবে; Poppins-এ বাংলা অক্ষর নেই, তাই বাংলা লেখা পরের font **Hind Siliguri**-তে দেখাবে।

এখন ধাপ ১ — font load করার ৩টা উপায়:

---

### Option ১: `index.html`-এ `<link>` — সহজ ও দ্রুত

`index.html`-এর `<head>`-এ:

```html
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Poppins:wght@400;500;600;700;800&family=Hind+Siliguri:wght@500;600&display=swap" rel="stylesheet">
```

- Browser HTML পড়ার সাথে সাথেই font-এর CSS নামানো শুরু করে।
- `preconnect` দিয়ে Google-এর server-এর সাথে connection আগেই তৈরি হয়ে যায়, তাই আরও দ্রুত।
- `display=swap` — font load না হওয়া পর্যন্ত system font-এ লেখা দেখায়, পরে বদলে যায় (লেখা অদৃশ্য থাকে না)।
- Google Fonts নিজেও এই পদ্ধতি recommend করে।

**অসুবিধা:** font আসে Google-এর server থেকে — third-party-র ওপর নির্ভরতা থাকে, আর internet ছাড়া কাজ করে না।

---

### Option ২: CSS-এ `@import url(...)` — ⚠️ এড়িয়ে চলুন

`index.css`-এর **একদম শুরুতে**:

```css
@import url("https://fonts.googleapis.com/css2?family=Poppins:wght@400;500;600;700;800&family=Hind+Siliguri:wght@500;600&display=swap");
@import "tailwindcss";
/* ...বাকি সব... */
```

কাজ করে, কিন্তু **সবচেয়ে ধীর**। কারণ request-গুলো একটার পর একটা হয় (chain):

```
আপনার CSS নামানো → পড়ে বুঝল Google-এর CSS লাগবে → Google-এর CSS নামানো → font ফাইল নামানো
```

`<link>`-এ browser প্রথম ধাপেই Google-এর CSS নামানো শুরু করে দেয়, তাই সেটা দ্রুত।

> ⚠️ সাধারণ ভুল: CSS-এর নিয়মে `@import` ফাইলের **সবার আগে** থাকতে হয় — Tailwind-এর `@import "tailwindcss"`-এরও আগে। অন্য কোনো rule-এর পরে লিখলে browser এটা উপেক্ষা করে, font load হয় না।

---

### Option ৩: npm package import (Fontsource) — ✅ production-এর জন্য সবচেয়ে ভালো

**Fontsource** Google Fonts-এর font-গুলো npm package হিসেবে দেয়।

Install:

```sh
bun add @fontsource/poppins @fontsource/hind-siliguri
```

`src/main.tsx`-এ, `index.css`-এর import-এর **আগে**:

```ts
// Poppins — যে weight লাগবে শুধু সেগুলো
import "@fontsource/poppins/400.css";
import "@fontsource/poppins/500.css";
import "@fontsource/poppins/600.css";
import "@fontsource/poppins/700.css";
import "@fontsource/poppins/800.css";

// Hind Siliguri (বাংলা)
import "@fontsource/hind-siliguri/500.css";
import "@fontsource/hind-siliguri/600.css";

import "./index.css";
```

**সুবিধা:**
- **Self-hosted** — font ফাইল আপনার প্রজেক্টের সাথেই bundle হয়ে deploy হয়। Google-এর server-এ কোনো request যায় না।
- Vite ফাইলের নামে hash দেয়, তাই browser লম্বা সময় cache রাখতে পারে।
- প্রতিটা font ভাষা অনুযায়ী ছোট ছোট ভাগে (subset) থাকে — বাংলা অংশ **শুধু তখনই নামে** যখন page-এ বাংলা লেখা থাকে।
- Offline বা local development-এও কাজ করে।
- User-এর তথ্য Google-এর কাছে যায় না (privacy)।

**অসুবিধা:**
- প্রতিটা weight আলাদা করে import করতে হয়। (যে weight import করেননি, যেমন `font-light` (300), সেটা ঠিকমতো দেখাবে না।)
- Deploy-এর size সামান্য বাড়ে।

> নোট: কিছু font-এর **variable** version আছে (`@fontsource-variable/...`) — একটা import-এই সব weight পাওয়া যায়। Poppins আর Hind Siliguri-র variable version নেই, তাই এখানে weight আলাদা করে import করতে হচ্ছে।

---

### তুলনা

| | `<link>` (HTML) | `@import url()` (CSS) | Fontsource (npm) |
|---|---|---|---|
| Speed | দ্রুত | সবচেয়ে ধীর | দ্রুত |
| Font কোথা থেকে আসে | Google server | Google server | নিজের site (self-hosted) |
| Offline কাজ করে? | না | না | হ্যাঁ |
| Setup | সবচেয়ে সহজ | সহজ, কিন্তু ভুল জায়গায় লিখলে কাজ করে না | Install + weight অনুযায়ী import |
| কখন | দ্রুত prototype, ছোট প্রজেক্ট | এড়িয়ে চলুন | Production, portfolio প্রজেক্ট |

> **আমাদের প্রজেক্টে:** **Fontsource (Option ৩)** — কারণ এই প্রজেক্ট live server-এ যাবে। তিনটার যেকোনো একটাই ব্যবহার করবেন, একসাথে একাধিক না — নাহলে একই font দুইবার load হবে।

---

## ৮. ✅ Best Practice: shadcn token-এ সরাসরি Phovia-র রং (সবচেয়ে optimized)

### কেন এটা সবচেয়ে ভালো

উপরের পদ্ধতিতে shadcn-এর token (`--primary`, `--muted` …) আর আমাদের `--phovia-*` token **দুই সেট** পাশাপাশি থাকে। এই পদ্ধতিতে আলাদা সেট বানাই না — **shadcn-এর token-গুলোতেই সরাসরি Phovia-র রং বসাই**, আর shadcn-এ যা নেই শুধু সেগুলো নতুন যোগ করি।

| | `phovia-` prefix পদ্ধতি (১–৬) | shadcn token পদ্ধতি (এই section) |
|---|---|---|
| Variable সেট | দুটো (shadcn + phovia) | একটা |
| Class-এর নাম | `bg-phovia-primary` | `bg-primary` — ছোট, সব shadcn dev চেনে |
| shadcn component (Button, Input) | আলাদা map না করলে Phovia রঙে আসে না | নিজে থেকেই Phovia রঙে |
| CSS size | বড়, একই রং দুইবার | ছোট, প্রতিটা রং একবার |
| Dark mode | কাজ করে | কাজ করে |

### দুইভাবে বানানো যায়

**১) Claude দিয়ে (দ্রুত):** design canvas-এর লিংক দিয়ে Claude-কে বলা — "এই ডিজাইনের সব রং light/dark সহ বের করে shadcn token-এ map করে `index.css` বানাও; shadcn-এ যে token নেই সেগুলো নতুন নামে যোগ করো।" তারপর ফলাফল নিজে মিলিয়ে দেখা (বিশেষ করে ডিজাইনে না থাকা প্রস্তাবিত মানগুলো)।

**২) নিজে (শেখার জন্য ভালো):**
1. ডিজাইন থেকে সব রং light/dark জোড়া করে তালিকা বানানো।
2. প্রতিটা রংকে কাজ অনুযায়ী shadcn token-এর সাথে মেলানো (নিচের mapping table)।
3. যেগুলো মেলে না (success, warning, star …) সেগুলোর জন্য নতুন semantic নাম দেওয়া।
4. `:root` আর `.dark`-এ মান বসানো, `@theme inline`-এ শুধু **নতুন** token-গুলো যোগ করা।

### Design → shadcn token mapping

| ডিজাইনের নাম | shadcn token | Class উদাহরণ |
|---|---|---|
| `bg` | `--background` | `bg-background` |
| `ink` | `--foreground` | `text-foreground` |
| `surface` | `--card`, `--popover` | `bg-card` |
| `surface2` | `--secondary`, `--muted` | `bg-muted` |
| `muted` (লেখা) | `--muted-foreground` | `text-muted-foreground` |
| `line` | `--border`, `--input` | `border-border` |
| `primary` | `--primary`, `--ring` | `bg-primary` |
| `primary-ink` | `--primary-foreground` | `text-primary-foreground` |
| `primary-soft` | `--accent` | `bg-accent` |
| `danger` | `--destructive` | `text-destructive` |
| `c1`, `c2` | `--chart-1`, `--chart-2` | chart-এ |
| — (নতুন) | `--success`, `--warning`, `--info`, `--star`, `--status-*` … | `text-success`, `bg-warning-soft` |

> নোট: `--muted` (background) আর `--muted-foreground` (লেখা) shadcn-এ আলাদা জিনিস। ডিজাইনের "muted" আসলে লেখার রং, তাই সেটা `--muted-foreground`-এ গেছে।

---

### ধাপ ১: Font package বদলানো

shadcn init-এ আমরা **Nova** preset বেছেছিলাম, যেটা **Geist** font install করে। আমাদের ডিজাইনে Geist নেই, তাই সেটা সরিয়ে Poppins আর Hind Siliguri যোগ করব।

```sh
# bun (আমাদের প্রজেক্টে এটাই)
bun add @fontsource/poppins @fontsource/hind-siliguri
bun remove @fontsource-variable/geist

# npm-এর সমতুল্য
npm i @fontsource/poppins @fontsource/hind-siliguri
npm uninstall @fontsource-variable/geist
```

> ⚠️ প্রজেক্ট bun দিয়ে চলছে বলে **bun-এর command-ই চালান**। bun প্রজেক্টে npm চালালে একটা `package-lock.json` তৈরি হয়, ফলে দুটো lockfile হয়ে যায় (৩ নম্বর নোট দেখুন)।
>
> ⚠️ Geist uninstall করার পর `index.css`-এ থাকা `@import "@fontsource-variable/geist";` লাইন আর `--font-sans`-এ থাকা `'Geist Variable'` **মুছতে হবে**। নাহলে package খুঁজে না পেয়ে build error দেবে।

> নোট: Font এখানে `index.css`-এর `@import` দিয়ে আনা হয়েছে, ৭ নম্বর section-এর মতো `main.tsx`-এ না। দুটোই ঠিক — Vite দুই জায়গা থেকেই package-এর CSS bundle করে। CSS-এ রাখলে সব style এক ফাইলে থাকে। (এটা ৭ নম্বর section-এর ধীর `@import url(...)` না — এখানে font আমাদের নিজেদের bundle থেকে আসছে, Google থেকে না।)

---

### ধাপ ২: পুরো `src/index.css`

পুরো ফাইলটা এটা দিয়ে **replace** করুন:

```css
@import "tailwindcss";
@import "tw-animate-css";
@import "shadcn/tailwind.css";

/* Fonts (self-hosted, Fontsource) */
@import "@fontsource/poppins/400.css";
@import "@fontsource/poppins/500.css";
@import "@fontsource/poppins/600.css";
@import "@fontsource/poppins/700.css";
@import "@fontsource/poppins/800.css";
@import "@fontsource/hind-siliguri/500.css";
@import "@fontsource/hind-siliguri/600.css";

@custom-variant dark (&:is(.dark *));

@theme inline {
  --font-heading: var(--font-sans);
  --font-sans: 'Poppins', 'Hind Siliguri', system-ui, sans-serif;

  /* shadcn tokens */
  --color-background: var(--background);
  --color-foreground: var(--foreground);
  --color-card: var(--card);
  --color-card-foreground: var(--card-foreground);
  --color-popover: var(--popover);
  --color-popover-foreground: var(--popover-foreground);
  --color-primary: var(--primary);
  --color-primary-foreground: var(--primary-foreground);
  --color-secondary: var(--secondary);
  --color-secondary-foreground: var(--secondary-foreground);
  --color-muted: var(--muted);
  --color-muted-foreground: var(--muted-foreground);
  --color-accent: var(--accent);
  --color-accent-foreground: var(--accent-foreground);
  --color-destructive: var(--destructive);
  --color-border: var(--border);
  --color-input: var(--input);
  --color-ring: var(--ring);
  --color-chart-1: var(--chart-1);
  --color-chart-2: var(--chart-2);
  --color-chart-3: var(--chart-3);
  --color-chart-4: var(--chart-4);
  --color-chart-5: var(--chart-5);
  --color-sidebar: var(--sidebar);
  --color-sidebar-foreground: var(--sidebar-foreground);
  --color-sidebar-primary: var(--sidebar-primary);
  --color-sidebar-primary-foreground: var(--sidebar-primary-foreground);
  --color-sidebar-accent: var(--sidebar-accent);
  --color-sidebar-accent-foreground: var(--sidebar-accent-foreground);
  --color-sidebar-border: var(--sidebar-border);
  --color-sidebar-ring: var(--sidebar-ring);

  /* Extra tokens from the Phovia design (shadcn has no equivalent) */
  --color-primary-hover: var(--primary-hover);
  --color-destructive-foreground: var(--destructive-foreground);
  --color-destructive-soft: var(--destructive-soft);
  --color-success: var(--success);
  --color-success-soft: var(--success-soft);
  --color-warning: var(--warning);
  --color-warning-soft: var(--warning-soft);
  --color-info: var(--info);
  --color-info-soft: var(--info-soft);

  --color-soft: var(--soft);
  --color-grid: var(--grid);
  --color-pink: var(--pink);
  --color-pink-soft: var(--pink-soft);
  --color-yellow-soft: var(--yellow-soft);
  --color-mint-soft: var(--mint-soft);
  --color-star: var(--star);
  --color-placeholder: var(--placeholder);
  --color-placeholder-foreground: var(--placeholder-foreground);

  --color-status-complete: var(--status-complete);
  --color-status-pending: var(--status-pending);
  --color-status-cancelled: var(--status-cancelled);
  --color-status-failed: var(--status-failed);

  --shadow-card: var(--card-shadow);

  /* Radius: 10px buttons/inputs, 14px cards */
  --radius-sm: calc(var(--radius) * 0.6);
  --radius-md: calc(var(--radius) * 0.8);
  --radius-lg: var(--radius);
  --radius-xl: calc(var(--radius) * 1.4);
  --radius-2xl: calc(var(--radius) * 1.8);
  --radius-3xl: calc(var(--radius) * 2.2);
  --radius-4xl: calc(var(--radius) * 2.6);
}

:root {
  --radius: 0.625rem;

  /* Surfaces & text */
  --background: #FFFFFF;
  --foreground: #1C1A3A;
  --card: #FFFFFF;
  --card-foreground: #1C1A3A;
  --popover: #FFFFFF;
  --popover-foreground: #1C1A3A;
  --secondary: #F7F6FD;
  --secondary-foreground: #1C1A3A;
  --muted: #F7F6FD;
  --muted-foreground: #6A6785;
  --soft: #FFF3F5;
  --grid: #EEECF6;

  /* Brand */
  --primary: #5B4CF0;
  --primary-foreground: #FFFFFF;
  --primary-hover: #4A3BD9;
  --accent: #EEEBFF;
  --accent-foreground: #5B4CF0;

  /* Lines & focus */
  --border: #ECEAF5;
  --input: #ECEAF5;
  --ring: #5B4CF0;

  /* Status */
  --destructive: #B42318;
  --destructive-foreground: #FFFFFF;
  --destructive-soft: #FDECEA;
  --success: #17714A;
  --success-soft: #E2F3EA;
  --warning: #8F5400;
  --warning-soft: #FFF0D4;
  --info: #21539E;
  --info-soft: #E5EDFA;

  /* Icon tiles, rating, image placeholder */
  --pink: #B4335A;
  --pink-soft: #FFE8EC;
  --yellow-soft: #FFF4DC;
  --mint-soft: #E3F7EF;
  --star: #FFB547;
  --placeholder: #EEEBF8;
  --placeholder-foreground: #8A86AE;

  /* Booking status badges */
  --status-complete: #2E8B57;
  --status-pending: #D9A21B;
  --status-cancelled: #9AA3A0;
  --status-failed: #C8402F;

  /* Charts (3–5 reuse status colors) */
  --chart-1: #6152F0;
  --chart-2: #D9771F;
  --chart-3: #2E8B57;
  --chart-4: #D9A21B;
  --chart-5: #C8402F;

  /* Admin sidebar */
  --sidebar: #FFFFFF;
  --sidebar-foreground: #1C1A3A;
  --sidebar-primary: #5B4CF0;
  --sidebar-primary-foreground: #FFFFFF;
  --sidebar-accent: #EEEBFF;
  --sidebar-accent-foreground: #5B4CF0;
  --sidebar-border: #ECEAF5;
  --sidebar-ring: #5B4CF0;

  --card-shadow: 0 12px 34px rgba(28, 26, 58, 0.08);
}

.dark {
  /* Surfaces & text */
  --background: #12111F;
  --foreground: #EEEDF7;
  --card: #1B1A2E;
  --card-foreground: #EEEDF7;
  --popover: #1B1A2E;
  --popover-foreground: #EEEDF7;
  --secondary: #1F1D35;
  --secondary-foreground: #EEEDF7;
  --muted: #1F1D35;
  --muted-foreground: #A9A6C4;
  --soft: #211B2E;
  --grid: #2A2845;

  /* Brand */
  --primary: #9D92FF;
  --primary-foreground: #14112E;
  --primary-hover: #B3AAFF;
  --accent: #2A2550;
  --accent-foreground: #9D92FF;

  /* Lines & focus */
  --border: #2E2B4A;
  --input: #2E2B4A;
  --ring: #9D92FF;

  /* Status */
  --destructive: #F28B80;
  --destructive-foreground: #2A0E0B; /* not in design, proposed */
  --destructive-soft: #3B1916;
  --success: #6DCB98;
  --success-soft: #14301F;
  --warning: #F1C46E;
  --warning-soft: #382B10;
  --info: #9DBBF0;                   /* not in design, proposed */
  --info-soft: #172338;              /* not in design, proposed */

  /* Icon tiles, rating, image placeholder */
  --pink: #FF9DB2;
  --pink-soft: #3A2230;
  --yellow-soft: #3A3020;
  --mint-soft: #1D3530;
  --star: #FFB547;
  --placeholder: #26243F;
  --placeholder-foreground: #7E7AA6;

  /* Booking status badges */
  --status-complete: #4FB67C;
  --status-pending: #E2B23A;
  --status-cancelled: #6E7A77;
  --status-failed: #E0604E;

  /* Charts */
  --chart-1: #7F73F5;
  --chart-2: #D8762A;
  --chart-3: #4FB67C;
  --chart-4: #E2B23A;
  --chart-5: #E0604E;

  /* Admin sidebar */
  --sidebar: #1B1A2E;
  --sidebar-foreground: #EEEDF7;
  --sidebar-primary: #9D92FF;
  --sidebar-primary-foreground: #14112E;
  --sidebar-accent: #2A2550;
  --sidebar-accent-foreground: #9D92FF;
  --sidebar-border: #2E2B4A;
  --sidebar-ring: #9D92FF;

  --card-shadow: 0 12px 34px rgba(0, 0, 0, 0.35);
}

@layer base {
  * {
    @apply border-border outline-ring/50;
  }

  body {
    @apply bg-background text-foreground;
  }

  html {
    @apply font-sans antialiased;
  }
}
```

---

### আপনার দেওয়া ফাইলে যা ঠিক করা হয়েছে

> ⚠️ **সংশোধন ১ — Shadow variable নিজেকেই reference করছিল:**
> ```css
> @theme inline { --shadow-card: var(--shadow-card); }   /* ❌ */
> :root        { --shadow-card: 0 12px 34px ...; }
> ```
> `--shadow-card`-এর মান `var(--shadow-card)` — অর্থাৎ নিজের দিকেই ইঙ্গিত (circular reference)। এখন হয়তো CSS layer-এর ক্রমের কারণে কাজ করে যাচ্ছে, কিন্তু এটা ভঙ্গুর আর বিভ্রান্তিকর। shadcn-এর অন্য সব token-এ দুই নাম আলাদা (`--color-background: var(--background)`), এখানেও তাই করা হয়েছে:
> ```css
> @theme inline { --shadow-card: var(--card-shadow); }   /* ✅ */
> :root        { --card-shadow: 0 12px 34px ...; }
> ```
> Class একই থাকছে: `shadow-card`।

> ⚠️ **সংশোধন ২ — "proposed" comment ভুল জায়গায় ছিল:** dark mode-এর `--destructive-soft: #3B1916` আসলে **ডিজাইনেই আছে**। ডিজাইনে যেগুলো নেই সেগুলো হলো `--destructive-foreground` (dark), `--info` (dark) আর `--info-soft` (dark)। Comment ঠিক জায়গায় সরানো হয়েছে।

> নোট: `--chart-3` থেকে `--chart-5` ডিজাইনে ছিল না — এগুলোতে status-এর রং (সবুজ, হলুদ, লাল) বসানো হয়েছে। Chart-এ ৩টার বেশি series লাগলে কাজে লাগবে।

> নোট: Hind Siliguri-র শুধু 500 আর 600 weight import করা আছে (ডিজাইনে এগুলোই ব্যবহার হয়েছে)। সাধারণ (400) weight-এর বাংলা লেখাতে browser সবচেয়ে কাছের 500 ব্যবহার করবে। বাংলা লেখা একটু হালকা চাইলে `@import "@fontsource/hind-siliguri/400.css";` যোগ করুন।

---

### `index.css`-এর গঠন — কোন অংশ কী করে

| অংশ | কাজ |
|---|---|
| `@import` | Tailwind, animation, shadcn-এর base style আর font আনা। সবসময় ফাইলের **শুরুতে** |
| `@custom-variant dark` | `.dark` class-এর ভেতরে থাকলে `dark:` variant চালু হবে |
| `@theme inline` | Tailwind-কে token-এর নাম জানানো → class তৈরি হয় (`bg-primary`, `shadow-card`, `rounded-xl`, `font-sans`) |
| `:root` | Light mode-এর আসল রং |
| `.dark` | Dark mode-এর আসল রং — **একই নাম**, ভিন্ন মান |
| `@layer base` | সব element-এর default: border রং, body-র background/লেখা, font |

> `@layer base`-এর কারণে `<body>`-এ আলাদা করে `bg-background text-foreground font-sans` লিখতে হয় না — নিজে থেকেই বসে যায়।

---

### ব্যবহারের উদাহরণ

```tsx
import { Button } from "@/components/ui/button";

{/* shadcn Button — নিজে থেকেই Phovia বেগুনি */}
<Button>Book now</Button>
<Button variant="outline">Details</Button>
<Button variant="destructive">Delete</Button>

{/* Card */}
<div className="bg-card shadow-card rounded-xl p-6">
  <h3 className="font-semibold">Sajek Valley</h3>
  <p className="text-muted-foreground">3 days 2 nights</p>
  <span className="text-star">★ 4.8</span>
</div>

{/* Status badge */}
<span className="bg-success-soft text-success rounded-full px-3 py-1 text-sm">Paid</span>
<span className="bg-warning-soft text-warning rounded-full px-3 py-1 text-sm">Pending</span>

{/* Booking status dot */}
<span className="size-2 rounded-full bg-status-complete" />

{/* Custom primary button-এ hover */}
<a className="bg-primary text-primary-foreground hover:bg-primary-hover rounded-lg px-5 py-3">Explore</a>
```

> 💡 **Tailwind CSS IntelliSense** (VS Code extension) install করে রাখুন — `bg-` লিখলেই সব token-এর তালিকা রঙের preview সহ আসবে, কিছু মুখস্থ রাখতে হবে না।