# Frontend Notes — ০৭: UI Resources (shadcn-এর সহায়ক জিনিস)

shadcn/ui দিয়ে মূল component (button, input, dialog …) পাওয়া যায়। কিন্তু দ্রুত সুন্দর UI বানাতে আরও কিছু resource কাজে লাগে — তৈরি section/block, icon, logo, illustration ইত্যাদি। এই নোটে সেগুলোর তালিকা।

> ⚠️ **আমাদের প্রজেক্টের জন্য জরুরি নোট:** আমরা shadcn init-এ **Base UI** বেছে নিয়েছি। কিন্তু বাইরের অনেক shadcn-style library (Origin UI, Magic UI ইত্যাদি) **Radix UI**-এর ওপর বানানো। এসব component কপি করলে:
> - অতিরিক্ত Radix package install হতে পারে, অথবা
> - আমাদের Base UI-ভিত্তিক `components/ui/`-এর সাথে API মিলবে না (যেমন Radix-এর `asChild`-এর বদলে Base UI-তে `render` prop)।
>
> তাই কপি করার আগে দেখে নিন component-টা কোন primitive-এর ওপর বানানো।

---

## ১. Component ও Block Library

### Origin UI
- **আসল সাইট:** https://originui.com (এখন **coss.com**-এর অংশ)
- নোটে দেওয়া লিংক **https://www.originui-ng.com/** — ⚠️ **সংশোধন:** এটা আসল Origin UI না। এটা **Angular**-এর জন্য বানানো একটা আলাদা port, এবং সাইটেই লেখা আছে যে এর সাথে আসল Origin UI-র কোনো সম্পর্ক নেই। আমরা React ব্যবহার করছি, তাই এটা আমাদের কাজে লাগবে না।

**আসল Origin UI-র কী হয়েছে:**
- Origin UI ছিল shadcn-style, Radix-ভিত্তিক copy-paste component-এর বড় সংগ্রহ — বিশেষ করে form control-এ খুব ভালো।
- পরে **coss.com** (Cal.com-এর) এটা acquire করে। পুরনো Origin UI component-গুলো এখন একটা **legacy snapshot** হিসেবে MIT license-এ আছে, কিন্তু support ও maintenance সীমিত।
- নতুন উত্তরসূরি **coss ui** (https://coss.com/ui) — এটা **Base UI**-এর ওপর বানানো (আমাদের setup-এর সাথে মেলে!), কিন্তু এর license **AGPLv3**, যা MIT-এর চেয়ে অনেক কড়া। ব্যবহারের আগে license পড়ে নেওয়া উচিত।

**সিদ্ধান্ত:** নোটের কথাটা মোটামুটি ঠিক — Origin UI এখন আগের মতো সক্রিয়ভাবে maintain হয় না, তাই নতুন প্রজেক্টে এর ওপর নির্ভর না করাই ভালো। পুরনো component-গুলো দেখতে চাইলে 21st.dev-এ Origin UI-র collection পাওয়া যায়।

### HyperUI — https://hyperui.dev/
- **Free, open-source Tailwind CSS component**-এর সংগ্রহ (MIT license)।
- কিছু **install করতে হয় না** — কোড কপি করে বসিয়ে দিলেই হয়, কারণ এগুলো শুধু Tailwind class দেওয়া HTML।
- Application component (badge, dropdown, chart …), Marketing component (CTA, banner, blog card, contact form …), Neobrutalism style আর পুরো template আছে।
- Interactive অংশের (dropdown খোলা ইত্যাদি) জন্য আলাদা **HyperUX** আছে।
- নোট: এখন মূলত **HTML** কোড দেয়, JSX option আর দেখা যায় না। React-এ ব্যবহার করতে হলে `class` → `className` করতে হবে, আর self-closing tag (যেমন `<img>` → `<img />`) ঠিক করতে হবে।
- **কখন ব্যবহার করব:** landing page-এর section (hero, CTA, footer) দ্রুত বানাতে।

### shadcn Blocks — https://ui.shadcn.com/blocks
- shadcn-এর নিজস্ব তৈরি **block** — dashboard, sidebar, login page, signup page ইত্যাদি।
- CLI দিয়ে সরাসরি add করা যায় (`bunx --bun shadcn@latest add <block-name>`)।
- আমাদের setup-এর সাথে সবচেয়ে বেশি মানানসই — **প্রথমে এখানেই খুঁজুন।**

### shadcn Registry Directory — https://ui.shadcn.com/docs/registry/registry-index
- shadcn CLI দিয়ে install করা যায় এমন **third-party registry**-গুলোর অফিসিয়াল তালিকা। নতুন library খুঁজতে ভালো জায়গা।

### Magic UI — https://magicui.design
- Animated component — marquee, animated text, shimmer button, bento grid ইত্যাদি।
- Landing page-কে আকর্ষণীয় করতে ভালো। shadcn CLI দিয়ে install হয়।

### Aceternity UI — https://ui.aceternity.com
- চোখ ধাঁধানো animation-ওয়ালা component (Framer Motion/Motion দিয়ে)।
- Portfolio বা landing page-এ ভালো, কিন্তু বেশি ব্যবহার করলে site ভারী আর ধীর হয় — অল্প ব্যবহার করুন।

### 21st.dev — https://21st.dev
- Community-র বানানো shadcn-style React component-এর বিশাল marketplace। অনেক library-র component (Origin UI সহ) এক জায়গায় preview করা যায়।

---

## ২. Theme ও Design

### tweakcn — https://tweakcn.com
- shadcn-এর **theme editor** — রং, font, radius বদলে live দেখা যায়, তারপর CSS variable কপি করে `index.css`-এ বসিয়ে দেওয়া যায়।

### shadcn/create — https://ui.shadcn.com/create
- shadcn-এর অফিসিয়াল preset builder — style, রং, font, icon বেছে নিয়ে নতুন প্রজেক্টের command তৈরি করে দেয়।

---

## ৩. Logo, Icon, Image

### Logoipsum — https://logoipsum.com/
- **Placeholder logo** — ডিজাইনের সময় দ্রুত ব্যবহারের জন্য বানানো fake logo।
- কাজে লাগে: navbar-এর logo, "Trusted by" / partner section-এ কোম্পানির logo-র জায়গায়।
- ⚠️ এগুলো **placeholder** — আসল brand-এর logo হিসেবে ব্যবহারের জন্য না। Production-এ নিজের logo বসাতে হবে।

### Lucide Icons — https://lucide.dev
- shadcn-এর **default icon library** (`lucide-react`)। shadcn init করলে সাধারণত এটা আগেই install হয়ে যায়।
- ব্যবহার: `import { Search } from "lucide-react"` → `<Search className="size-4" />`

### অন্যান্য
| Resource | কী | লিংক |
|---|---|---|
| Tabler Icons | ৫০০০+ free icon | https://tabler.io/icons |
| unDraw | Free illustration, রং নিজের মতো বদলানো যায় | https://undraw.co |
| Unsplash | Free, উচ্চমানের ছবি | https://unsplash.com |

---

## ৪. কোনটা কখন ব্যবহার করব — সংক্ষেপে

| দরকার | প্রথম পছন্দ |
|---|---|
| Button, Input, Dialog-এর মতো মূল component | shadcn/ui |
| Login, dashboard, sidebar-এর পুরো block | shadcn Blocks |
| Landing page-এর section (hero, CTA, footer) | HyperUI, shadcn Blocks |
| Animation / চমক | Magic UI (অল্প পরিমাণে Aceternity) |
| Theme-এর রং ঠিক করা | tweakcn |
| Placeholder logo | Logoipsum |
| Icon | Lucide |

> **License-এর নিয়ম:** কোনো কোড কপি করার আগে license দেখে নিন। **MIT** হলে নিশ্চিন্তে ব্যবহার করা যায়; **AGPL** বা "Pro/paid" লেখা থাকলে শর্তগুলো পড়ে নিতে হবে।