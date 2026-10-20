# Frontend Notes — ০৩: Package Manager ও Vite

## Package Manager কী

Node.js-এর জন্য বিভিন্ন সময়ে মানুষ অনেক package (library) বানিয়েছে। সেই কোডগুলো সবার সাথে share করার জন্য একটা **registry**-তে রাখা হয় (মূলত **npm registry**, `registry.npmjs.org`)। **Package manager** সেই registry থেকে package নামিয়ে প্রজেক্টে install করে, version আর dependency সামলায়।

আমরা এখন **npm** ব্যবহার করছি। Node install করলে npm সাথে সাথেই চলে আসে। npm ছাড়াও বেশ কিছু বিকল্প আছে:
- **yarn**
- **pnpm**
- **bun**

এগুলো সব একরকম না।

> নোট: yarn, pnpm, bun — সবাই by default **একই npm registry** থেকে package নামায়। পার্থক্য হলো package কীভাবে install, store আর link করে তাতে।

প্রতিটার নিজস্ব **lockfile** আছে — একটা প্রজেক্টে একটাই package manager ব্যবহার করা উচিত, মেশানো উচিত না:

| Package manager | Lockfile |
|---|---|
| npm | `package-lock.json` |
| yarn | `yarn.lock` |
| pnpm | `pnpm-lock.yaml` |
| bun | `bun.lock` |

---

## ১. npm

npm শুধু একটা package manager (CLI)। "npm" নামটা দিয়ে registry-কেও বোঝানো হয়।

**অসুবিধা:**
- **তুলনামূলক slow** — বিশেষ করে বড় প্রজেক্টে install-এ yarn/pnpm/bun-এর চেয়ে বেশি সময় লাগে।
- **Disk space বেশি লাগে** — প্রতিটা প্রজেক্টে আলাদা `node_modules` তৈরি হয় এবং প্রতিটায় package-এর পুরো কপি থাকে। ১০টা প্রজেক্টে একই package থাকলে disk-এ ১০টা কপি।

> ⚠️ সংশোধন — "npm safe না, vulnerability বেশি":
> - Vulnerability থাকে **package-গুলোর মধ্যে**, আর সেই package সব package manager একই registry থেকেই নামায়। তাই npm ব্যবহার করলে vulnerability বেশি আর pnpm ব্যবহার করলে কম — এমন না।
> - Registry-তে যে কেউ package publish করতে পারে, কঠোর review হয় না — এটা **পুরো npm ecosystem-এর** সমস্যা (supply chain attack), শুধু npm CLI-র না।
> - আসল পার্থক্য: npm by default dependency-র **install script** (`postinstall` ইত্যাদি) চালিয়ে দেয়, যা দিয়ে malicious কোড চলতে পারে। pnpm (v10+) আর bun by default dependency-র এই script চালায় না — এই অর্থে এগুলো তুলনামূলক নিরাপদ default দেয়।
> - npm-এ `npm audit` দিয়ে known vulnerability check করা যায়।
>
> পুরনো npm (v5-এর আগে) সত্যিই slow ছিল এবং lockfile ছিল না — yarn আসার পেছনে এটা বড় কারণ। এখনকার npm অনেক উন্নত হয়েছে।

---

## ২. yarn

yarn-ও একটা package manager। npm তখন slow আর অনির্ভরযোগ্য ছিল বলে **Facebook (Meta)** ২০১৬ সালে yarn নিয়ে আসে।

তখনকার npm-এর তুলনায় সুবিধা:
- **Faster** — parallel download, offline cache।
- **Deterministic install** — `yarn.lock` দিয়ে সবার মেশিনে হুবহু একই version install হয়।
- **Integrity check** — checksum দিয়ে package যাচাই করে।

**Version-এর ঝামেলা:** yarn-এর দুটো ধারা আছে —
- **Yarn Classic (v1)** — এখনো অনেক প্রজেক্টে ব্যবহার হয়, তবে এখন শুধু **maintenance mode**-এ (নতুন feature আসে না)।
- **Yarn Berry (v2+)** — নতুন architecture। এর **Plug'n'Play (PnP)** mode-এ `node_modules` ফোল্ডারই থাকে না। তবে কিছু tool-এর সাথে compatibility সমস্যা হতে পারে।

Disk space: Yarn Classic-ও npm-এর মতো প্রতিটা প্রজেক্টে আলাদা `node_modules` বানায়, তবে কিছুটা optimized।

---

## ৩. pnpm

pnpm-ও একটা package manager। npm আর yarn-এর চেয়ে সাধারণত **অনেক faster** এবং **disk efficient**।

**কীভাবে disk বাঁচায় (সংশোধিত):**
- pnpm কম্পিউটারে একটা **global content-addressable store** রাখে।
- একটা package (নির্দিষ্ট version) একবার download হলে store-এ থাকে।
- অন্য প্রজেক্টে একই package লাগলে আবার download করে না — store থেকে **hard link** করে দেয়, আর `node_modules`-এর ভেতরের structure **symlink** দিয়ে সাজায়।
- ফলে ১০টা প্রজেক্টে একই package থাকলেও disk-এ ফাইলগুলো **একবারই** থাকে।

> নোট: নোটে "symlink আর cache" লেখা ছিল — মূল ধারণা ঠিক, তবে নিখুঁতভাবে বললে ফাইলগুলো store থেকে **hard link** হয়, আর symlink ব্যবহার হয় dependency-গুলোর structure সাজাতে।

**আরেকটা সুবিধা — strict `node_modules`:** `package.json`-এ যে package declare করা নেই, সেটা কোডে import করা যায় না (**phantom dependency** সমস্যা থেকে রক্ষা করে)। npm/yarn classic-এ ভুল করে এমন import কাজ করে যায়।

pnpm এখন production-এ, বিশেষ করে **monorepo**-তে, খুব জনপ্রিয়। (তবে মোট ব্যবহারে এখনো npm সবচেয়ে বেশি।)

---

## ৪. bun

> ⚠️ সংশোধন: bun শুধু package manager না। Bun মূলত একটা **JavaScript runtime** (Node.js-এর বিকল্প), যার সাথে একটা toolkit বিল্ট-ইন:
> - **Runtime** — Node-এর মতো JS/TS কোড চালায় (`bun index.ts`)। Node যেখানে V8 engine ব্যবহার করে, bun সেখানে **JavaScriptCore** (Safari-র engine) ব্যবহার করে।
> - **Package manager** — `bun install`
> - **Task/script runner** — `bun run dev` (`package.json`-এর script চালায়)
> - **Test runner** — `bun test`
> - **Bundler** — `bun build`
> - **TypeScript** সরাসরি চালাতে পারে, আলাদা compile step লাগে না।
>
> নোটে লেখা ছিল "task runner মানে test case run করতে পারে" — এটা মেশানো হয়ে গেছে। **Task runner** script চালায়; **test runner** test চালায়। Bun-এ দুটোই আছে।

এখানে যতগুলো package manager নিয়ে কথা হলো, install speed-এ সাধারণত **bun সবচেয়ে fast**।

**Zig:** Bun লেখা হয়েছে **Zig** নামের একটা low-level systems language দিয়ে। Zig, C/Rust-এর মতো **low-level** — memory নিজে নিয়ন্ত্রণ করা যায়, garbage collector নেই — তাই খুব efficient ও দ্রুত কোড লেখা যায়।
> নোট: "Zig, Rust-এর মতোই" — শুধু এই অর্থে ঠিক যে দুটোই fast systems language। Rust-এর মতো compile-time memory safety guarantee Zig-এ নেই।

**অসুবিধা:** Bun তুলনামূলক নতুন। Runtime হিসেবে Node-এর সব API ও package-এর সাথে এখনো পুরোপুরি compatible না, তাই production-এ (বিশেষ করে **runtime** হিসেবে) অনেক team এখনো Node + pnpm/npm-কেই বেছে নেয়।
> নোট: শুধু **package manager** হিসেবে (`bun install`) bun ব্যবহার করা তুলনামূলক নিরাপদ বিকল্প, এমনকি app Node দিয়ে চললেও।

### আমাদের প্রজেক্টে bun কেন — production-এর জন্যও

এই প্রজেক্ট live server-এ deploy হবে, তবুও frontend-এ **bun** ব্যবহার করা নিরাপদ। কারণ:

**Frontend — bun কোনো ঝুঁকি না:**
- Vite + React প্রজেক্ট build করলে (`bun run build`) একটা `dist/` ফোল্ডার তৈরি হয়, যেখানে শুধু **static HTML, CSS আর JS** ফাইল থাকে।
- Live server-এ শুধু এই ফাইলগুলোই যায়। সেখানে bun বা Node কোনোটাই চলে না।
- তাই bun এখানে শুধু **install আর build**-এর কাজ করে, **production runtime** হিসেবে চলে না।
- আগে bun-এর যে সমস্যার কথা বলা হয়েছে (Node compatibility), সেটা মূলত **runtime** হিসেবে চালানোর সময়ের সমস্যা, তাই এখানে প্রযোজ্য না।
- Vercel, Netlify-এর মতো জনপ্রিয় hosting সাধারণত `bun.lock` দেখে নিজেই bun দিয়ে install করে নেয়। তবে deploy করার আগে hosting-এর docs একবার দেখে নিতে হবে।

**Backend — ভাবনাটা আলাদা:**
- Backend (Express + Passport.js) server-এ সবসময় চলবে, তাই সেখানে **runtime** বেছে নেওয়াটা আসলেই গুরুত্বপূর্ণ।
- **Runtime:** **Node.js** রাখা ভালো। এটা সবচেয়ে stable, প্রায় সব hosting-এ (Render, Railway, VPS, Docker) ঝামেলা ছাড়াই চলে, আর Passport-এর মতো পুরনো library-র সাথে compatibility নিয়ে চিন্তা করতে হয় না।
- **Package manager:** backend-এ bun বা pnpm, যেকোনোটা ব্যবহার করা যায়।

| অংশ | Package manager | Production runtime |
|---|---|---|
| Frontend (Vite + React) | bun | নেই — static ফাইল serve হয় |
| Backend (Express + Passport.js) | bun বা pnpm | Node.js |

---

## Vite

এর সাথে আমরা **Vite** ব্যবহার করব।
- উচ্চারণ: **"ভিট" (veet)** — ফরাসি শব্দ, অর্থ "দ্রুত"।
- বানিয়েছেন **Evan You** — Vue.js-এর creator।
- Vite একটা **frontend build tool** — framework না, শুধু একটা tool। এটা **dev server** আর **production build** দুটোই দেয়।
- শুধু Vue না, **React, Svelte, Solid, Preact, Lit, vanilla JS/TS** সহ অনেক কিছু support করে।

### Development আর Production আলাদা

**Development-এ:**
- Vite পুরো app আগে থেকে bundle করে না। Browser-এর নিজস্ব **ES Modules (ESM)** support ব্যবহার করে — browser যে ফাইল চায়, শুধু সেটা transform করে পাঠায়। তাই dev server প্রায় সাথে সাথেই চালু হয়।
- `node_modules`-এর dependency-গুলো একবার **pre-bundle** করে রাখে (দ্রুত tool দিয়ে), যাতে browser-কে হাজারটা ছোট ফাইল load করতে না হয়।
- **HMR (Hot Module Replacement)** — কোড বদলালে পুরো page reload ছাড়াই শুধু বদলানো অংশ update হয়।

**Production-এ:**
- কোড **bundle**, minify আর optimize করে (code splitting, tree-shaking)। Vite ঐতিহাসিকভাবে এর জন্য **Rollup** ব্যবহার করেছে; নতুন version-গুলোতে Rust-ভিত্তিক **Rolldown**-এ সরে যাচ্ছে।
- কারণ: production-এ শত শত আলাদা ESM ফাইল network দিয়ে load করা ধীর হয়, bundle করা ফাইল দ্রুত।

> নোট: নোটে "ESM module" লেখা ছিল — ESM মানেই "ECMAScript Modules", তাই "ESM" বা "ES Modules" লিখলেই যথেষ্ট।