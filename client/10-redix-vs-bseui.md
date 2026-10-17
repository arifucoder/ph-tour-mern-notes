# Frontend Notes — ১০: Theme বদলানো, Navbar link, Radix বনাম Base UI

> Login/Register page-এর কাজ শুরুর আগের প্রস্তুতিগুলো এই নোটে রাখা হয়েছে। Login/Register-এর নোট আলাদা: `frontend-notes-11-login-register-pages.md`।

---

## ১. shadcn/create থেকে theme (রং, font) নেওয়া

Theme-এর রং, font, radius ইত্যাদি তৈরি অবস্থায় নিতে চাইলে এই page ব্যবহার করা যায়: https://ui.shadcn.com/create

- এখানে **Vite** আর **Next.js** দুটোর option আছে।
- প্রজেক্ট আগেই install করা থাকলে **শুধু CSS** কপি করার option-ও আছে। সেখান থেকে `:root` আর `.dark` block-এর কোড পাওয়া যায়।

### কোথায় বসাবেন

> ⚠️ **পুরো `index.css` replace করবেন না।** শুধু `:root { ... }` আর `.dark { ... }` block দুটো replace করবেন।
>
> যেগুলো **যেমন আছে তেমন থাকবে:**
> - উপরের `@import` লাইনগুলো (Tailwind, font ইত্যাদি)
> - `@custom-variant dark`
> - `@theme inline { ... }`
> - `@layer base { ... }`

> ⚠️ **আমাদের নিজের যোগ করা token হারাবেন না।** আমাদের `:root` আর `.dark`-এ shadcn-এর token ছাড়াও Phovia ডিজাইনের বাড়তি token আছে, যেমন `--success`, `--warning`, `--info`, `--star`, `--status-*`, `--primary-hover` (৮ নম্বর নোট)। shadcn/create-এর কোডে এগুলো নেই। তাই পুরো block কপি-পেস্ট করলে এগুলো মুছে যাবে, আর `text-success`-এর মতো class চুপচাপ কাজ করা বন্ধ করে দেবে। নিরাপদ উপায়: shadcn/create থেকে শুধু shadcn-এর token-গুলোর (`--background`, `--primary`, `--muted` …) মান বদলান, আমাদের বাড়তি token-গুলো রেখে দিন।

---

## ২. Navbar-এর link

`src/components/layout/Navbar.tsx`-এ শুধু যে link-গুলো লাগবে সেগুলো রাখব। Tours আর Divisions page এখনো বানানো হয়নি, তবে আপাতত রেখে দিচ্ছি:

```tsx
const navLinks = [
  { label: "Home", href: "/" },
  { label: "About", href: "/about" },
  { label: "Tours", href: "/tours" },
  { label: "Divisions", href: "/divisions" },
  { label: "Short tours", href: "/tours?type=short" },
  { label: "Long tours", href: "/tours?type=long" },
];
```

### Navigation-এর নিয়ম

- **Navigation link-এ অবশ্যই `NavLink` ব্যবহার করব।** Active state-এর জন্য বাড়তি কিছু (`useLocation` ইত্যাদি) যোগ করব না (৯ নম্বর নোটের "বিকল্প উপায়: NavLink")।
- **Navbar আর Footer-এ কোথাও `<a href>` থাকলে `<Link to>` দিয়ে replace করব।** `<a>` দিলে পুরো page reload হয়, `<Link>` দিলে হয় না।

> ⚠️ মনে রাখুন: `NavLink` query (`?type=`) দেখে না। তাই `/tours`, `/tours?type=short` আর `/tours?type=long` তিনটাই একসাথে active দেখাবে। Tours page বানানোর সময় `/tours/short` আর `/tours/long` আলাদা route করে নিলে সমস্যা মিটবে।

### প্রতিবার কোড লেখার পরে console দেখা

প্রতিটা কোড লেখার পর browser-এর **Console** (`F12`) খুলে দেখে নেব কোনো warning বা error আসছে কিনা। সবচেয়ে common সমস্যা হলো **একটা interactive element-এর ভেতরে আরেকটা**:

| Warning | কখন হয় |
|---|---|
| `<a> cannot be a descendant of <a>` | একটা `Link`-এর ভেতরে আরেকটা `Link` (যেমন Logo একটা Link, তাকে আবার Link দিয়ে মুড়ে দিলে) |
| `<button> cannot be a descendant of <button>` | Trigger-এর ভেতরে `<Button>` (dark-theme-toggle নোট দেখুন) |
| `<a>`-এর ভেতরে `<button>` | `<Link><Button>…</Button></Link>`। এটা HTML-এ invalid। |

Link-কে button-এর মতো দেখাতে চাইলে নিচের যেকোনো একটা উপায় নিন:

```tsx
// উপায় ১: buttonVariants — Radix আর Base UI দুটোতেই কাজ করে (Navbar-এ এটাই করা হয়েছে)
<Link to="/register" className={buttonVariants()}>Sign up</Link>

// উপায় ২: Radix-এ asChild
<Button asChild>
  <Link to="/register">Sign up</Link>
</Button>

// উপায় ২: Base UI-তে render
<Button render={<Link to="/register" />}>Sign up</Button>
```

---

## ৩. Radix UI বনাম Base UI

### আগে আর এখন

আগে shadcn init করলে **Radix UI** নিজে থেকেই install হতো, অন্য কিছু বেছে নেওয়ার option ছিল না। এখন shadcn init বা shadcn/create-এ **Radix UI** আর **Base UI**-এর মধ্যে একটা বেছে নেওয়া যায়।

দুটোই **headless** library (২ নম্বর নোট)। Dropdown খোলা/বন্ধ, keyboard দিয়ে চালানো, focus সামলানো, accessibility — এই আচরণগুলো তারা দেয়। আর shadcn তার ওপর Tailwind দিয়ে design বসায়। ফলে দুটো দেখতে প্রায় একই, পার্থক্য ভেতরের কোডে আর লেখার ধরনে।

### তুলনা

| | Radix UI | Base UI |
|---|---|---|
| কারা বানিয়েছে | WorkOS | Radix, Material UI (MUI) আর Floating UI-এর নির্মাতাদের দল |
| বয়স | পুরনো, অনেক বছর ধরে ব্যবহৃত, খুব stable | নতুন, সম্প্রতি v1 (stable) হয়েছে |
| shadcn-এ | আগের default, shadcn এর ওপরেই প্রথম বানানো | নতুন option |
| Child element-কে trigger বানানো | `asChild` prop | `render` prop |
| বাইরের resource (tutorial, blocks, Origin UI, Magic UI …) | প্রায় সবই Radix-এর জন্য লেখা | কম, তবে বাড়ছে |
| Component সংখ্যা | যথেষ্ট | কিছু বাড়তি component আছে (যেমন Combobox, Number Field) |
| Development | Maintenance চলছে, নতুন feature ধীরে আসে | সক্রিয়ভাবে নতুন feature যোগ হচ্ছে |

### একই কাজ, দুই রকম লেখা

```tsx
// Radix UI
<DropdownMenuTrigger asChild>
  <Button variant="outline" size="icon">
    <Sun />
  </Button>
</DropdownMenuTrigger>

// Base UI
<DropdownMenuTrigger render={<Button variant="outline" size="icon" />}>
  <Sun />
</DropdownMenuTrigger>
```

- **Radix (`asChild`):** "নিজে আলাদা element বানিয়ো না, ভেতরের child-কেই trigger বানাও।" Icon/text থাকে `<Button>`-এর ভেতরে।
- **Base UI (`render`):** "এই element-টা দিয়ে render করো।" Icon/text থাকে Trigger-এর ভেতরে।

### কোনটা বেছে নেব

- **Teacher বা কোনো tutorial follow করলে:** তারা যেটা ব্যবহার করছে সেটাই। নাহলে প্রতিটা কোড অনুবাদ করতে হবে।
- **বাইরের shadcn block/library বেশি ব্যবহার করতে চাইলে:** **Radix**, কারণ বেশিরভাগ resource Radix-এর জন্য লেখা (৭ নম্বর নোট)।
- **একদম নতুন প্রজেক্ট, নিজে লিখবেন:** দুটোই ভালো। Base UI নতুন আর সক্রিয়, Radix পুরনো আর পরীক্ষিত।

> ⚠️ **এক প্রজেক্টে একটাই রাখুন।** দুটো মেশালে একই component-এর দুই রকম API চলে আসে, আর বাইরে থেকে কপি করা কোড কোনটার, সেটা গুলিয়ে যায়।

### আমাদের প্রজেক্ট: Base UI থেকে Radix-এ সরানো হলো

Teacher Radix ব্যবহার করছেন, তাই তাঁকে follow করার সুবিধার জন্য প্রজেক্টে **Radix UI** install করা হয়েছে। এখন থেকে **Radix-এর কোড** follow করতে হবে।

Radix-এ সরার পর যা মিলিয়ে নিতে হবে:

1. **`components.json`:** এখানে Radix বেছে নেওয়া আছে কিনা দেখুন। নাহলে পরের `shadcn add` আবার Base UI-এর component নামাবে।
2. **`components/ui/`-এর পুরনো component:** Base UI দিয়ে আগে add করা component (button, dropdown-menu, sheet …) Radix version দিয়ে বদলে নিন:
   ```sh
   bunx --bun shadcn@latest add button dropdown-menu sheet --overwrite
   ```
3. **নিজের লেখা কোডে `render` থাকলে `asChild`-এ বদলান।** যেমন `ModeToggle.tsx`:
   ```tsx
   <DropdownMenuTrigger asChild>
     <Button variant="outline" size="icon">
       <Sun className="h-[1.2rem] w-[1.2rem] scale-100 rotate-0 transition-all dark:scale-0 dark:-rotate-90" />
       <Moon className="absolute h-[1.2rem] w-[1.2rem] scale-0 rotate-90 transition-all dark:scale-100 dark:rotate-0" />
       <span className="sr-only">Toggle theme</span>
     </Button>
   </DropdownMenuTrigger>
   ```
   (`dark-theme-toggle.md` নোটের Base UI version-টা তখন আর প্রযোজ্য না।)
4. **Docs থেকে কোড কপি করার সময়:** shadcn component page-এর উপরে **Radix** বেছে নেওয়া আছে কিনা দেখে নিন।

> নোট: `buttonVariants()` দিয়ে বানানো link (Navbar-এর Sign up) দুই library-তেই একইভাবে কাজ করে। তাই সেগুলো বদলাতে হবে না।