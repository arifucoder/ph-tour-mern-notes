# 38 — Deploy: Vercel আর Contabo VPS (PM2 + Nginx)

Backend দুই জায়গায় deploy করব:

| | Vercel | Contabo VPS |
|---|---|---|
| ধরন | **Serverless** — request এলে function চালু হয়, কাজ শেষে ঘুমিয়ে পড়ে | **নিজের server** — একটা computer সারাক্ষণ চলছে |
| Setup | সহজ, কয়েকটা command | বেশি কাজ (PM2, Nginx, SSL) |
| নিয়ন্ত্রণ | কম | পুরোটা নিজের হাতে |
| খরচ | Free plan আছে | মাসিক ভাড়া |

> 📌 **এখনকার অবস্থা:** Deploy হয়েছে, কিন্তু email (nodemailer) আর payment এখনো পুরোপুরি কাজ করছে না। Frontend deploy আর আরেকটা project শেষ করে এখানে ফিরব। কোথা থেকে শুরু করে খুঁজতে হবে, সেটা শেষে **Part 4**-এ লিখে রাখলাম।

---

# Part 1: Deploy-এর আগে — Build ঠিক করা

দুই জায়গাতেই চলবে `dist/server.js` (build করা JavaScript), `.ts` file না। তাই আগে build ঠিকঠাক হওয়া চাই।

## Step 1: `package.json` Scripts

```json
"scripts": {
	"dev": "tsx watch src/server.ts",
	"lint": "eslint src",
	"typecheck": "tsc --noEmit",
	"build": "tsup",
	"start": "node dist/server.js"
}
```

| Script | কাজ |
|---|---|
| `dev` | Development — `.ts` সরাসরি চালায়, file বদলালে নিজে restart |
| `lint` | ESLint দিয়ে code-এর ভুল আর নিয়ম check |
| `typecheck` | TypeScript-এর type error check (কিছু build করে না) |
| `build` | `tsup` দিয়ে `src` → `dist` (JavaScript) |
| `start` | Production-এ build করা `dist/server.js` চালানো — **এটা থাকতেই হবে** |

> ✏️ **ছোট পরিবর্তন:**
> - `"build"` আর `"build:tsup"` দুটোই `tsup` — একই কাজ দুবার। একটাই যথেষ্ট।
> - `"lint": "npx eslint ./src"` → `"eslint src"`। npm script-এর ভেতরে project-এর package এমনিতেই পাওয়া যায়, `npx` লাগে না।

### ⚠️ `tsup` type error check করে না!

এটা অনেকেই জানে না: `tsup` খুব দ্রুত build করে, কারণ সে শুধু TypeScript-এর type **মুছে ফেলে** JavaScript বানায় — type ঠিক আছে কিনা **দেখেই না**। মানে type error থাকলেও `npm run build` সফল হবে, আর ভুলটা ধরা পড়বে live server-এ।

তাই deploy-এর আগে তিনটা command, এই ক্রমে:

```bash
npm run lint        # ১. code-এর নিয়ম
npm run typecheck   # ২. type error
npm run build       # ৩. build
```

তিনটাই error ছাড়া পার হলে তবেই deploy। **Error** অবশ্যই ঠিক করতে হবে; **warning** ঠিক করা ভালো, তবে deploy আটকায় না।

## Step 2: `tsup.config.ts`

Project-এর root-এ:

```ts
// tsup.config.ts
import { defineConfig } from "tsup";

export default defineConfig({
	entry: ["src/server.ts"], // এখান থেকে শুরু করে সব import খুঁজে নেবে
	format: ["esm"], // import/export (ES Module) format-এ output
	target: "esnext",
	outDir: "dist",
	clean: true, // প্রতিবার build-এর আগে পুরোনো dist মুছে দেবে
	bundle: true, // সব file মিলিয়ে একটা dist/server.js
	splitting: false,
	sourcemap: true, // error-এ dist-এর বদলে আসল .ts file-এর লাইন দেখাবে
	banner: {
		js: `
import { createRequire } from 'module';
const require = createRequire(import.meta.url);
`,
	},
	// build শেষ হলে email template copy
	onSuccess: "cp -r src/app/utils/templates dist/templates",
});
```

> ✏️ Note-এ এই code ` ```json `-এর ভেতরে ছিল, কিন্তু এটা TypeScript file (`.ts`)। JSON-এ `import`, comment বা function থাকতে পারে না।

### `banner` কেন?

আমাদের output ESM (`import`)। কিন্তু কিছু পুরোনো package ভেতরে ভেতরে CommonJS-এর `require()` ব্যবহার করে। ESM-এ `require` নেই, তাই সেগুলো crash করত (`require is not defined`)। `banner` প্রতিটা build-এর **একদম শুরুতে** এই দুই লাইন বসিয়ে দেয়, যা ESM-এর ভেতরেও একটা কাজ করা `require` বানিয়ে দেয়।

### `onSuccess` কেন — Email template

`tsup` শুধু **JavaScript/TypeScript** file নিয়ে কাজ করে। `.ejs` file (email template) সে চেনেই না, তাই `dist`-এ যায় না ([note 32](./32-forgot-reset-password-nodemailer.md))। Build-এর পর হাতে copy না করলে production-এ email পাঠানোর সময় error: `ENOENT: no such file or directory ... forgetPassword.ejs`।

| Build tool | Template copy |
|---|---|
| `tsup` | `onSuccess`-এ `cp` command (উপরে) |
| `tsc` | Build script-এ আলাদা করে: `"build": "tsc && cp -r src/app/utils/templates dist/app/utils/templates"` |

**নিয়ম:** `.ts`/`.js` ছাড়া অন্য যেকোনো file (`.ejs`, `.json` data, font, image) যেটা code runtime-এ **পড়ে**, সেটা `dist`-এ গেছে কিনা নিজে check করতে হবে। Build-এর পর একবার দেখে নাও:

```bash
ls dist/templates   # forgetPassword.ejs  invoice.ejs  otp.ejs  থাকা উচিত
```

> 💡 `cp -r` Mac/Linux-এর command। Windows-এ কাজ না করলে `npm i -D cpy-cli` দিয়ে `"cpy 'src/app/utils/templates/**' dist/templates"` ব্যবহার করা যায়।

## Step 3: Build আর Local-এ Production Test

```bash
npm run build
npm start   # dist/server.js চালায় — ঠিক যেভাবে server-এ চলবে
```

Deploy-এর আগে local-এ `npm start` দিয়ে একবার login, booking, email পাঠানো test করে নাও। এখানে কাজ না করলে server-এও করবে না।

**কিছু বদলালেই আবার build:**

```
code বদলাও → lint → typecheck → build → deploy
```

`dist` হলো build-এর সময়ের ছবি — পরে `src` বদলালে `dist` নিজে বদলায় না।

---

# Part 2: Vercel

## Step 4: `vercel.json`

Project-এর root-এ:

```json
{
	"version": 2,
	"builds": [
		{
			"src": "dist/server.js",
			"use": "@vercel/node",
			"config": {
				"includeFiles": ["dist/templates/**"]
			}
		}
	],
	"routes": [
		{
			"src": "/(.*)",
			"dest": "dist/server.js"
		}
	]
}
```

| অংশ | মানে |
|---|---|
| `builds.src` | কোন file চালাতে হবে — build করা `dist/server.js` (যেখানে আছে, হুবহু সেই path) |
| `"use": "@vercel/node"` | Node.js দিয়ে চালাও |
| `routes` | সব URL (`/(.*)`) এই একটা file-এ পাঠাও — বাকিটা Express-এর router সামলাবে |
| `includeFiles` ← **যোগ করেছি** | নিচে দেখো ⬇️ |

> ⚠️ **`includeFiles` কেন — সম্ভবত email না যাওয়ার একটা কারণ:**
> Vercel deploy-এর সময় শুধু সেই file-গুলো নেয়, যেগুলো code `import` করে। `.ejs` template কেউ `import` করে না — `ejs.renderFile()` runtime-এ path দিয়ে পড়ে। তাই `dist/templates` local-এ থাকলেও Vercel-এর server-এ **পৌঁছায় না** → `ENOENT` → email fail। `includeFiles` দিয়ে Vercel-কে বলে দিই: "এই folder-ও সাথে নাও।"

## Step 5: Vercel CLI দিয়ে Deploy

```bash
# Vercel CLI install / update
npm install -g vercel@latest
vercel --version

# কোন account-এ login আছে
vercel whoami

# login (না থাকলে)
vercel login

# production-এ deploy
vercel --prod
```

প্রথমবার `vercel --prod` দিলে কয়েকটা প্রশ্ন করবে:
- **Set up and deploy?** → Yes
- **Link to existing project?** → No (নতুন project)
- **Project name?** → default বা নিজের পছন্দের নাম
- **In which directory is your code located?** → `./` (default)

শেষে একটা live link দেবে, যেমন `https://phovia-tour-server.vercel.app`।

> 💡 Vercel CLI uninstall করতে: `npm uninstall -g vercel`

### ⚠️ Multer-এর Conflict — Vercel-এ install fail

Vercel deploy-এর সময় নিজে `npm install` চালায়। আমাদের `multer-storage-cloudinary`-র peer dependency conflict-এর জন্য ([note 29](./29-multer-cloudinary.md)) সেখানে `ERESOLVE` error দিয়ে install fail করতে পারে।

দুটো সমাধান:

| উপায় | কী করতে হবে |
|---|---|
| **১. `.npmrc`** (সহজ) | Project root-এ `.npmrc` file-এ `legacy-peer-deps=true` ([note 32](./32-forgot-reset-password-nodemailer.md))। Vercel এই file পড়ে, তাই সেখানেও একই নিয়মে install হবে। |
| **২. Package বাদ** (teacher যা করেছেন) | `multer-storage-cloudinary` uninstall করে `multer.memoryStorage()` দিয়ে file memory-তে নিয়ে নিজে Cloudinary-তে upload করা। এর জন্য দরকারি `uploadBufferToCloudinary` function আমরা invoice-এর সময় আগেই বানিয়েছি ([note 34](./34-pdf-invoice-pdfkit.md))। |

প্রথমে `.npmrc` দিয়ে চেষ্টা করো। কাজ না হলে বা package-টা স্থায়ীভাবে সরাতে চাইলে দ্বিতীয় উপায়।

## Step 6: Environment Variable

Local-এর `.env` file Vercel-এ যায় না (যাওয়া উচিতও না)। সব variable Vercel-এ আলাদা করে দিতে হবে:

**Vercel Dashboard → Project → Settings → Environment Variables** → প্রতিটা key-value যোগ করো (বা `.env`-এর পুরো লেখা একবারে paste করা যায়)।

### কোনটা কী হবে

| Variable | Local | Production |
|---|---|---|
| `NODE_ENV` | `development` | **`production`** |
| `SSL_SUCCESS_BACKEND_URL` ইত্যাদি (৩টা) | `http://localhost:5000/api/v1/payment/...` | `https://<vercel-link>/api/v1/payment/...` |
| `SSL_IPN_URL` | `http://localhost:5000/...` | `https://<vercel-link>/api/v1/payment/validate-payment` |
| `GOOGLE_CALLBACK_URL` | `http://localhost:5000/api/v1/auth/google/callback` | `https://<vercel-link>/api/v1/auth/google/callback` |
| `FRONTEND_URL`, `SSL_*_FRONTEND_URL` | `http://localhost:5173...` | **Frontend-এর** live link (frontend deploy না হওয়া পর্যন্ত localhost থাকবে) |

> ✏️ **সংশোধন:** Note-এ লেখা ছিল ".env-এর **যত জায়গায় localhost:5000** আছে, সব Vercel-এর live link দিয়ে বদলে দেব।" এটা ঠিক — শুধু `localhost:5000` (backend)-গুলোই বদলাবে। কিন্তু `localhost:5173` (**frontend**) ভুল করে backend-এর link দিয়ে বদলে দিও না। Frontend-এর URL হবে frontend-এর নিজের live link।

- সব URL-এর **শেষে slash না** ([note 35](./35-cors-refactor-user.md))।
- Vercel-এ env বদলালে **আবার deploy** করতে হয়, তবেই নতুন value কাজ করে।

### `NODE_ENV=production` দিলে যা বদলায়

- Error response-এ `stack` আর `err` যায় না ([note 23](./23-advanced-global-error-handler.md))
- Cookie: `secure: true`, `sameSite: "none"` ([note 35](./35-cors-refactor-user.md)) — note-এ লেখা ছিল "setCookie.ts-এ sameSite none না হলে করতে হবে" — note 35-এর `authCookieOptions` থাকলে এটা `NODE_ENV` দেখে নিজেই হয়ে যায়।

## Step 7: `app.set("trust proxy", 1)`

```ts
// src/app.ts — app বানানোর ঠিক পরেই, সব middleware-এর আগে
const app = express();
app.set("trust proxy", 1);
```

### কেন লাগে? — Proxy কী

Live server-এ user-এর request সরাসরি আমাদের Express-এ আসে না। মাঝে একটা **proxy** থাকে:

```
User (https) → Vercel / Nginx (proxy) → আমাদের Express app (http)
```

| | Vercel-এ | VPS-এ |
|---|---|---|
| Proxy | Vercel-এর নিজের server | Nginx (+ Cloudflare) |

HTTPS-এর কাজ proxy-ই সেরে নেয় (SSL terminate করে), তারপর ভেতরে Express-এর কাছে সাধারণ **HTTP**-তে পাঠায়। ফলে Express নিজে দেখে:
- Request এসেছে **HTTP**-তে (`req.secure = false`), যদিও user আসলে HTTPS ব্যবহার করছে
- User-এর IP না, **proxy-র IP** (`req.ip`)

Proxy অবশ্য সাথে আসল তথ্য header-এ লিখে দেয়: `X-Forwarded-Proto: https`, `X-Forwarded-For: <user-এর আসল IP>`। কিন্তু Express default-ভাবে এই header **বিশ্বাস করে না** (যে কেউ বানিয়ে পাঠাতে পারে)। `trust proxy` দিয়ে বলি: "**আমার সামনে একটা proxy আছে, তার দেওয়া এই header বিশ্বাস করো।**"

### এটা না দিলে কী ভাঙে?

- **`express-session`-এর cookie:** Google login-এর জন্য আমরা `express-session` ব্যবহার করি ([note 21](./21-passport-oauth.md))। Session cookie `secure: true` হলে, `express-session` নিজে check করে request HTTPS-এ এসেছে কিনা — `req.secure` false দেখে **cookie set-ই করে না**। Note-এ লেখা "এটা না দিলে cookie set হবে না" — এই কারণেই।
- `req.ip` সবসময় proxy-র IP দেখাবে (পরে rate limiting বা log-এ সব user একই IP মনে হবে)।
- `req.protocol` `"http"` দেখাবে, ফলে কোথাও URL বানালে `http://` আসবে।

### ❓ Security-র কোনো ঝুঁকি আছে?

`trust proxy` মানে `X-Forwarded-*` header বিশ্বাস করা — আর header যে কেউ বানিয়ে পাঠাতে পারে। তাই **কতটা বিশ্বাস করব** সেটাই আসল প্রশ্ন:

| Setting | মানে | ঝুঁকি |
|---|---|---|
| `false` (default) | কোনো proxy বিশ্বাস করো না | নেই, কিন্তু proxy-র পেছনে cookie/IP ভুল |
| **`1`** ✅ | **ঠিক একটা** proxy-কে বিশ্বাস করো (সবচেয়ে কাছেরটা) | কম — শুধু সরাসরি সামনের proxy-র কথা মানে |
| `true` | সামনের **সব** hop বিশ্বাস করো | ⚠️ কেউ নিজে `X-Forwarded-For: 1.2.3.4` পাঠিয়ে নিজের IP লুকাতে বা বদলাতে পারে |

`1` নিরাপদ, **যদি সত্যিই সামনে একটা proxy থাকে আর app-এ সরাসরি কেউ ঢুকতে না পারে।**

> ⚠️ **VPS-এ সাবধান:** কেউ যদি Nginx এড়িয়ে সরাসরি `http://<server-ip>:5000`-এ request পাঠাতে পারে, তাহলে সে নিজেই "proxy" হয়ে বানানো header পাঠাতে পারবে। তাই port 5000 বাইরে থেকে **বন্ধ** রাখতে হবে (Step 13-এর firewall)। শুধু Nginx-ই যেন app-এর কাছে যেতে পারে।
>
> 💡 VPS-এ Cloudflare + Nginx মিলে দুটো proxy। `1` দিলে cookie আর HTTPS ঠিকঠাক কাজ করবে (Nginx `X-Forwarded-Proto` দেয়), শুধু `req.ip`-এ user-এর বদলে Cloudflare-এর IP আসতে পারে। আসল user IP লাগলে (যেমন rate limiting) তখন `2` দিতে হবে, বা Cloudflare-এর `CF-Connecting-IP` header পড়তে হবে।

## Step 8: আবার Build আর Deploy

`trust proxy` যোগ করার পর:

```bash
npm run lint && npm run typecheck && npm run build
vercel --prod
```

## Step 9: SSLCommerz আর Google-এ Live URL

### SSLCommerz

Merchant panel-এ login করে store settings-এ **IPN URL** live link দিয়ে update করো: `https://<vercel-link>/api/v1/payment/validate-payment`। (আমরা প্রতিটা payment-এর সাথে `ipn_url` পাঠাচ্ছি — [note 37](./37-validate-payment-sslcommerz.md) — তবুও panel-এ দিয়ে রাখা ভালো।)

### Google Cloud Console (Google Login)

**APIs & Services → Credentials → OAuth Client** → দুই জায়গায় live link যোগ করো (localhost-এরগুলো রেখে দাও, local-এ কাজ চালাতে):

| জায়গা | কী দেবে | উদাহরণ |
|---|---|---|
| **Authorized JavaScript origins** | **Frontend**-এর origin (যেখান থেকে login button চাপা হয়) | `https://phovia-tour.vercel.app` |
| **Authorized redirect URIs** | **Backend**-এর callback route — `.env`-এর `GOOGLE_CALLBACK_URL`-এর সাথে **হুবহু** মিলতে হবে | `https://phovia-tour-server.vercel.app/api/v1/auth/google/callback` |

> ✏️ Note-এ লেখা ছিল "দুই জায়গাতেই আমাদের live url দেব।" একটু সাবধান — **origin-এ frontend-এর** URL, **redirect URI-তে backend-এর** callback। দুটো আলাদা।

Teacher ঠিকই বলেছেন যে এটা এখনো পুরোপুরি কাজ করবে না: Google login শেষে backend user-কে `FRONTEND_URL`-এ redirect করে, যেটা এখনো `localhost:5173`। Frontend deploy হলে আর তার URL বসালে তবেই পুরো flow চলবে।

## Step 10: Vercel-এ যে সীমাবদ্ধতাগুলো মাথায় রাখতে হবে

Vercel **serverless** — সারাক্ষণ চলা কোনো server না। Request এলে function জাগে, response দিয়ে ঘুমিয়ে পড়ে। এতে কিছু জিনিস আলাদাভাবে আচরণ করে:

| বিষয় | কী হয় | আমাদের project-এ প্রভাব |
|---|---|---|
| Response-এর পরের কাজ | Response পাঠানোর পর function থেমে যেতে পারে | `void deleteImagesFromCloudinary(...)` ([note 29](./29-multer-cloudinary.md))-এর মতো "পাঠিয়ে ভুলে যাও" কাজ মাঝপথে থেমে যেতে পারে |
| সময়সীমা | প্রতিটা request-এর সর্বোচ্চ সময় আছে (plan অনুযায়ী) | Payment success → validate + PDF + Cloudinary + email একসাথে অনেক সময় নিলে timeout |
| Cold start | অনেকক্ষণ request না এলে প্রথম request ধীর | MongoDB + Redis connect, `seedSuperAdmin` আবার চলে |
| File system | Read-only (শুধু `/tmp`-এ লেখা যায়) | আমরা disk-এ কিছু লিখি না, তাই ঠিক আছে |

এগুলো VPS-এ নেই, কারণ সেখানে server সারাক্ষণ চলে। Payment-এর মতো ভারী কাজে Vercel-এ সমস্যা হলে এটাও একটা কারণ হতে পারে।

---

# Part 3: Contabo VPS (PM2 + Nginx + Cloudflare)

বিস্তারিত guide: [deploy_express-app_pm2_nginx-reverse-proxy_cloudflare-ssl.md](https://github.com/arifucoder/devops-notes/blob/main/production-notes/express/deploy_express-app_pm2_nginx-reverse-proxy_cloudflare-ssl.md)

```
Internet → Cloudflare (DNS + SSL) → Nginx (:443, reverse proxy) → PM2 → Express app (:5000)
```

## Step 11: Server-এ Code আনা

```bash
# নিজের computer থেকে VPS-এ ঢোকা
ssh root@<server-ip>

# app রাখার জায়গায় যাও
cd /opt

# GitHub থেকে clone
git clone https://github.com/<your-username>/phovia-tour-server.git phovia-server
cd phovia-server
```

> ✏️ Note-এ লেখা ছিল "root folder-এই clone করব"। তোমার নিজের guide-এ `/opt` বলা আছে — সেটাই ভালো। `/root` হলো root user-এর ব্যক্তিগত folder, app-এর জন্য না। `/opt` বানানোই হয়েছে বাড়তি software/app রাখার জন্য।

### `.env` server-এ পাঠানো

`.env` GitHub-এ নেই (থাকাও উচিত না), তাই clone-এ আসবে না। নিজের computer থেকে copy করো:

```bash
# নিজের computer-এর terminal থেকে (VPS-এর ভেতর থেকে না)
scp .env root@<server-ip>:/opt/phovia-server/.env
```

তারপর VPS-এ `.env` খুলে production-এর value বসাও (Step 6-এর table): `NODE_ENV=production`, backend URL-গুলো live domain দিয়ে।

## Step 12: Install, Build, Test

```bash
# Node আছে কিনা
node -v
npm -v

# dependency install (package-lock.json অনুযায়ী হুবহু)
npm ci

# build — dist GitHub-এ নেই, তাই এখানে বানাতে হবে
npm run build
ls dist/templates   # template copy হয়েছে কিনা

# একবার সরাসরি চালিয়ে দেখো
npm start
```

`npm start`-এ "Server is running", "MongoDB connected", "Redis connected" দেখলে ঠিক আছে → `Ctrl + C` দিয়ে বন্ধ করো, এবার PM2 দিয়ে চালাব।

> ✏️ **সংশোধন:**
> - Note-এ test করার কথা ছিল `npm run dev` দিয়ে। কিন্তু production-এ চলবে `dist/server.js` — সেটা `npm start` দিয়েই test করা উচিত। `dev` চলে `.ts` থেকে, তাই build-এর সমস্যা (যেমন template copy না হওয়া) সেখানে ধরা পড়ে না।
> - **Build-এর ধাপটা বাদ পড়েছিল।** `dist` folder সাধারণত `.gitignore`-এ থাকে, তাই clone করলে আসে না। `npm run build` না করলে PM2 `dist/server.js` খুঁজেই পাবে না।
> - `npm install`-এর বদলে `npm ci`: `package-lock.json`-এ যে version লেখা আছে হুবহু সেটাই install করে — local-এ যা কাজ করেছে, server-এও ঠিক তাই।

## Step 13: Port খালি আছে কিনা

App চালানোর আগে দেখে নাও port 5000 অন্য কেউ ব্যবহার করছে কিনা:

```bash
sudo lsof -i :5000
# অথবা
sudo ss -tulpn | grep :5000
```

| ফলাফল | মানে |
|---|---|
| কিছুই আসেনি | Port খালি ✅ |
| কোনো লাইন এসেছে | কেউ ব্যবহার করছে — লাইনে process-এর নাম আর **PID** দেখা যাবে |

Port দখল থাকলে:
- আগের কোনো PM2 app হলে → `pm2 list` দিয়ে দেখো, দরকার হলে `pm2 delete <name>`
- অন্য কিছু হলে → `.env`-এ অন্য port (যেমন `5001`) দাও, আর Nginx config-এর `proxy_pass`-এও সেই port

### Firewall — port 5000 বাইরে থেকে বন্ধ

```bash
sudo ufw allow OpenSSH        # ssh বন্ধ হলে server-এ আর ঢুকতে পারবে না — আগে এটা!
sudo ufw allow 'Nginx Full'   # 80 আর 443
sudo ufw enable
sudo ufw status
```

5000 খোলা হয়নি — তাই বাইরে থেকে শুধু Nginx (80/443) দিয়েই ঢোকা যাবে, সরাসরি app-এ না। `trust proxy` নিরাপদ রাখতে এটা জরুরি (Step 7)।

## Step 14: PM2 দিয়ে চালানো

**PM2** একটা process manager: app crash করলে নিজে আবার চালু করে, terminal বন্ধ করলেও app চলতে থাকে, server restart হলেও app আবার চালু হয়।

```bash
# install (একবার)
npm i -g pm2
pm2 -v

# app চালু
pm2 start dist/server.js --name phovia-server

# server restart হলেও যেন নিজে চালু হয়
pm2 save
pm2 startup   # যে command দেখাবে, সেটা copy করে চালাও
```

### দরকারি PM2 command

| Command | কাজ |
|---|---|
| `pm2 list` | কোন কোন app চলছে |
| `pm2 logs phovia-server` | App-এর log (error খুঁজতে সবচেয়ে কাজের) |
| `pm2 restart phovia-server` | Restart |
| `pm2 stop phovia-server` | বন্ধ |
| `pm2 delete phovia-server` | PM2-এর list থেকে সরানো |

> ✏️ Note-এ `pm2 save` আর `pm2 startup` ছিল না। এগুলো ছাড়া VPS কোনো কারণে restart হলে (Contabo maintenance, update) app আর চালু হবে না।

আরও: [pm2.md](https://github.com/arifucoder/devops-notes/blob/main/devops-toolkit/pm2.md)

## Step 15: Nginx + Cloudflare (Domain আর SSL)

Guide: [nginx.md](https://github.com/arifucoder/devops-notes/blob/main/devops-toolkit/nginx.md) আর উপরের deploy guide।

সংক্ষেপে:
1. Cloudflare-এ domain-এর DNS-এ একটা **A record**: `api.yourdomain.com` → VPS-এর IP (Proxy চালু, কমলা মেঘ)।
2. `/etc/nginx/sites-available/api.yourdomain.com`-এ config: `listen 443 ssl`, Cloudflare origin certificate, আর `proxy_pass http://localhost:5000`।
3. `sudo ln -s ... /etc/nginx/sites-enabled/` → `sudo nginx -t` → `sudo systemctl reload nginx`।

তোমার guide-এর Nginx config-এ এই লাইনটা আছে, আর `trust proxy`-এর জন্য এটাই জরুরি:

```nginx
proxy_set_header X-Forwarded-Proto $scheme;
```

এটা Express-কে জানায় user আসলে HTTPS-এ এসেছে।

Domain ঠিক হলে `.env`-এর backend URL-গুলো (`SSL_*_BACKEND_URL`, `SSL_IPN_URL`, `GOOGLE_CALLBACK_URL`) `https://api.yourdomain.com/...` দিয়ে update করে `pm2 restart phovia-server`।

## Step 16: পরে Code Update করলে

```bash
cd /opt/phovia-server
git pull
npm ci
npm run build
pm2 restart phovia-server
pm2 logs phovia-server   # সব ঠিক আছে কিনা দেখো
```

---

# Part 4: যে সমস্যাগুলো বাকি — ফিরে এসে যেখান থেকে খুঁজব

> এখন ঠিক করছি না। পরে ফিরে এলে এই ক্রমে check করব।

## Email (Nodemailer) যাচ্ছে না

| # | Check | কীভাবে |
|---|---|---|
| 1 | আসল error কী? | Vercel: Dashboard → Logs (বা `vercel logs <url>`); VPS: `pm2 logs phovia-server`। `"Email sending error:"`-এর পরের লাইন দেখো |
| 2 | Template পৌঁছেছে? | Error-এ `ENOENT ... .ejs` থাকলে → Vercel-এ `includeFiles` (Step 4); VPS-এ `ls dist/templates` |
| 3 | Env আছে? | `SMTP_*` সব Vercel/VPS-এ দেওয়া হয়েছে কিনা; App Password-এ space নেই তো |
| 4 | SMTP connect হয়? | Note 32-এর `transporter.verify()` সাময়িক চালিয়ে log দেখো |
| 5 | VPS SMTP port আটকায়? | VPS-এ: `nc -vz smtp.gmail.com 465` → "succeeded" না এলে provider port বন্ধ রেখেছে; তখন `587` চেষ্টা করো (`secure: false`) বা provider-এর support-এ জিজ্ঞেস করো |
| 6 | Gmail আটকাচ্ছে? | নতুন server-এর IP থেকে login হলে Gmail "suspicious sign-in" বলে আটকাতে পারে — Gmail-এর inbox/Security-তে alert দেখো |

## Payment আটকে যাচ্ছে

| # | Check |
|---|---|
| 1 | **Email fail হলে কি payment-ও fail করছে?** তাহলে code-এ এখনো teacher-এর পুরোনো version — email transaction-এর ভেতরে। [Note 34](./34-pdf-invoice-pdfkit.md)-এর version-এ invoice/email commit-এর **পরে**, fail হলেও payment PAID থাকে। এটাই সবচেয়ে সম্ভাব্য কারণ, কারণ তুমি লিখেছ "nodemailer mail পাঠাচ্ছে না, এইজন্য payment আটকে আছে"। |
| 2 | SSLCommerz-এর success/fail/cancel URL-গুলো live backend-এর কিনা (Step 6) |
| 3 | `express.urlencoded()` আছে কিনা — SSLCommerz form data পাঠায় ([note 28](./28-booking-payment-sslcommerz.md)) |
| 4 | Vercel-এ timeout হচ্ছে কিনা (Step 10) — log-এ "Task timed out" |
| 5 | Payment-এর পর redirect হয় `SSL_SUCCESS_FRONTEND_URL`-এ — frontend deploy না হওয়া পর্যন্ত এটা localhost, তাই browser-এ page না খুললেও **DB-তে PAID হয়েছে কিনা** দেখো |

---

## সারাংশ

| বিষয় | মনে রাখার কথা |
|---|---|
| Deploy-এর আগে | `lint` → `typecheck` → `build` → `npm start` দিয়ে local test |
| `tsup` | Type check করে না — `tsc --noEmit` আলাদা চালাও |
| Template | `.ejs` নিজে copy (`onSuccess`); Vercel-এ `includeFiles`-ও |
| Vercel | `vercel.json` → `vercel --prod`; env Dashboard-এ; env বদলালে redeploy |
| Multer conflict | `.npmrc`-এ `legacy-peer-deps=true`, নাহলে package বাদ |
| Env | শুধু backend-এর `localhost:5000` বদলাও; `NODE_ENV=production`; শেষে slash না |
| `trust proxy` | Proxy-র পেছনে HTTPS/IP ঠিক বোঝার জন্য; `1` = শুধু সামনের একটা proxy; app-এ সরাসরি ঢোকা বন্ধ রাখো |
| Google | Origin = frontend, redirect URI = backend callback |
| Vercel serverless | Response-এর পরের কাজ থামতে পারে, timeout, cold start |
| VPS | `/opt`-এ clone → `.env` scp → `npm ci` → build → `npm start` test → PM2 → `pm2 save` + `startup` |
| Port | `lsof -i :5000`; firewall-এ 5000 বন্ধ, শুধু 80/443/SSH |
| Update | `git pull` → `npm ci` → build → `pm2 restart` |
| বাকি কাজ | Email: log → template → env → SMTP port; Payment: email-কে transaction-এর বাইরে (note 34) |