# 39 — Project Packages আর Scripts

`phovia-tour-server` project-এ যেসব npm package আর script ব্যবহার হয়েছে, তার পুরো তালিকা। `dependencies` আর `devDependencies` আলাদা table-এ, প্রতিটার পাশে কোন note-এ এসেছে।

---

## `dependencies` (Production-এ লাগবে)

### Server আর Database

| Package Name | Description |
|---|---|
| express | Node.js-এ server ও API বানানোর জন্য ব্যবহৃত সবচেয়ে জনপ্রিয় web framework। আমরা v5 ব্যবহার করছি — async error নিজে থেকেই ধরে, `req.body` না থাকলে `undefined`।<br>https://www.npmjs.com/package/express |
| mongoose | MongoDB-এর সাথে connect হয়ে schema/model দিয়ে data structure, validation, CRUD, populate, aggregation আর transaction করার জন্য ব্যবহৃত ODM লাইব্রেরি।<br>https://www.npmjs.com/package/mongoose |
| dotenv | `.env` file-এ রাখা environment variable (DB URL, secret, API key) code-এ load করার জন্য।<br>https://www.npmjs.com/package/dotenv |
| cors | Frontend (অন্য domain/port) থেকে backend-এ request আর cookie পাঠানোর অনুমতি দেওয়ার middleware ([note 35](./35-cors-refactor-user.md))।<br>https://www.npmjs.com/package/cors |
| cookie-parser | Request-এর cookie (access/refresh token) পড়ে `req.cookies`-এ বসিয়ে দেওয়ার middleware।<br>https://www.npmjs.com/package/cookie-parser |
| http-status-codes | `200`, `404`-এর মতো সংখ্যার বদলে `httpStatus.OK`, `httpStatus.NOT_FOUND` — পড়তে সহজ status code-এর নাম।<br>https://www.npmjs.com/package/http-status-codes |

### Validation

| Package Name | Description |
|---|---|
| zod | Schema define করে request body validate আর অচেনা field বাদ দেওয়ার জন্য TypeScript-friendly validation লাইব্রেরি (v4)।<br>https://www.npmjs.com/package/zod |

### Authentication

| Package Name | Description |
|---|---|
| bcryptjs | Password hash করে নিরাপদে store আর login-এর সময় compare করার জন্য। `bcrypt`-এর pure JavaScript রূপ — native build লাগে না, তাই যেকোনো OS আর Vercel-এ সহজে install হয়।<br>https://www.npmjs.com/package/bcryptjs |
| jsonwebtoken | Access token, refresh token আর reset token (JWT) বানানো ও verify করার জন্য।<br>https://www.npmjs.com/package/jsonwebtoken |
| passport | Authentication-এর framework — বিভিন্ন login পদ্ধতি (strategy) একভাবে চালানোর ব্যবস্থা ([note 21](./21-passport-oauth.md))।<br>https://www.npmjs.com/package/passport |
| passport-local | Email + password দিয়ে login-এর Passport strategy ([note 22](./22-passport-local.md))।<br>https://www.npmjs.com/package/passport-local |
| passport-google-oauth20 | "Continue with Google" login-এর Passport strategy ([note 21](./21-passport-oauth.md))।<br>https://www.npmjs.com/package/passport-google-oauth20 |
| express-session | Server-এ session রাখার middleware। Google OAuth-এর redirect চলাকালীন Passport এটা ব্যবহার করে।<br>https://www.npmjs.com/package/express-session |

### File Upload আর PDF

| Package Name | Description |
|---|---|
| multer | `multipart/form-data` (file + text) পড়ে `req.file` / `req.files`-এ রাখার middleware ([note 29](./29-multer-cloudinary.md))।<br>https://www.npmjs.com/package/multer |
| cloudinary | Image আর PDF Cloudinary-তে upload, delete করার official SDK।<br>https://www.npmjs.com/package/cloudinary |
| pdfkit | Code লিখে PDF বানানোর লাইব্রেরি — booking-এর invoice ([note 34](./34-pdf-invoice-pdfkit.md))।<br>https://www.npmjs.com/package/pdfkit |

### Email, Payment, Cache

| Package Name | Description |
|---|---|
| nodemailer | Gmail-এর SMTP দিয়ে email পাঠানোর জন্য — reset link, OTP, invoice ([note 32](./32-forgot-reset-password-nodemailer.md))।<br>https://www.npmjs.com/package/nodemailer |
| ejs | Email-এর HTML template-এ `<%= name %>`-এর মতো dynamic data বসানোর template engine।<br>https://www.npmjs.com/package/ejs |
| axios | অন্য server-এ HTTP request পাঠানোর জন্য — SSLCommerz-এর payment init আর validation API ([note 28](./28-booking-payment-sslcommerz.md))।<br>https://www.npmjs.com/package/axios |
| redis | Redis in-memory database-এর client — OTP রাখা, cooldown আর attempt গোনা ([note 33](./33-redis.md))।<br>https://www.npmjs.com/package/redis |

> 📝 **`multer-storage-cloudinary` নেই:** এই `package.json`-এ package-টা আর নেই — মানে deploy-এর সময় teacher-এর মতো সরিয়ে দিয়ে `multer.memoryStorage()` + নিজে Cloudinary upload-এ চলে গেছ ([note 38](./38-deploy-vercel-vps.md))। তাহলে peer dependency conflict-ও আর নেই — `.npmrc`-এর `legacy-peer-deps=true` না রাখলেও এখন install হওয়ার কথা। (রেখে দিলেও ক্ষতি নেই।)

---

## `devDependencies` (শুধু Development-এ লাগবে)

### TypeScript আর Build

| Package Name | Description |
|---|---|
| typescript | JavaScript-এর উপর static type-checking যোগ করার language/compiler। আমরা `tsc --noEmit` দিয়ে শুধু type check করি, build করে `tsup`।<br>https://www.npmjs.com/package/typescript |
| tsx | TypeScript file সরাসরি (build ছাড়াই) চালানো আর file বদলালে নিজে restart করার development টুল (`npm run dev`)।<br>https://www.npmjs.com/package/tsx |
| tsup | `src`-এর TypeScript থেকে production-এর জন্য `dist/server.js` বানানো bundler। খুব দ্রুত, তবে type check **করে না**।<br>https://www.npmjs.com/package/tsup |

### Linting

| Package Name | Description |
|---|---|
| eslint | Code-এর ভুল, খারাপ অভ্যাস আর style-এর নিয়ম ধরার টুল (`npm run lint`)।<br>https://www.npmjs.com/package/eslint |
| @eslint/js | ESLint-এর JavaScript-এর জন্য বানানো recommended নিয়মের set।<br>https://www.npmjs.com/package/@eslint/js |
| typescript-eslint | TypeScript code-এর জন্য ESLint-এর parser আর নিয়ম (আমরা `strict` + `stylistic` config ব্যবহার করি — `any`, `!` ইত্যাদি ধরে)।<br>https://www.npmjs.com/package/typescript-eslint |

### Type Definitions (`@types/...`)

এগুলোর runtime-এ কোনো কাজ নেই — শুধু TypeScript আর editor-কে বলে দেয় package-গুলোর function আর object দেখতে কেমন। যে package নিজে type নিয়ে আসে না (npm-এ **DT** badge), তার জন্য লাগে ([note 29](./29-multer-cloudinary.md))।

| Package Name | কার জন্য |
|---|---|
| @types/node | Node.js-এর নিজের module (`fs`, `path`, `crypto`, `process`, `Buffer`)<br>https://www.npmjs.com/package/@types/node |
| @types/express | `Request`, `Response`, `NextFunction`, `CookieOptions`<br>https://www.npmjs.com/package/@types/express |
| @types/cors | cors<br>https://www.npmjs.com/package/@types/cors |
| @types/cookie-parser | cookie-parser (`req.cookies`)<br>https://www.npmjs.com/package/@types/cookie-parser |
| @types/jsonwebtoken | jsonwebtoken (`JwtPayload`)<br>https://www.npmjs.com/package/@types/jsonwebtoken |
| @types/passport | passport<br>https://www.npmjs.com/package/@types/passport |
| @types/passport-local | passport-local (`IVerifyOptions`)<br>https://www.npmjs.com/package/@types/passport-local |
| @types/passport-google-oauth20 | passport-google-oauth20<br>https://www.npmjs.com/package/@types/passport-google-oauth20 |
| @types/express-session | express-session<br>https://www.npmjs.com/package/@types/express-session |
| @types/multer | multer (`Express.Multer.File`)<br>https://www.npmjs.com/package/@types/multer |
| @types/nodemailer | nodemailer<br>https://www.npmjs.com/package/@types/nodemailer |
| @types/ejs | ejs<br>https://www.npmjs.com/package/@types/ejs |
| @types/pdfkit | pdfkit<br>https://www.npmjs.com/package/@types/pdfkit |

> 💡 `mongoose`, `zod`, `axios`, `cloudinary`, `redis`, `bcryptjs`, `http-status-codes`, `dotenv` — এগুলো নিজেরাই type নিয়ে আসে (**TS** badge), তাই এদের `@types` লাগে না।

---

## একবারে Install (নতুন project-এ)

```bash
# dependencies
npm i express mongoose dotenv cors cookie-parser http-status-codes zod \
  bcryptjs jsonwebtoken passport passport-local passport-google-oauth20 express-session \
  multer cloudinary pdfkit nodemailer ejs axios redis

# devDependencies
npm i -D typescript tsx tsup eslint @eslint/js typescript-eslint \
  @types/node @types/express @types/cors @types/cookie-parser @types/jsonwebtoken \
  @types/passport @types/passport-local @types/passport-google-oauth20 @types/express-session \
  @types/multer @types/nodemailer @types/ejs @types/pdfkit
```

আগে থেকে `package.json` আর `package-lock.json` থাকলে (যেমন clone করার পর) শুধু:

```bash
npm ci   # lock file-এ যে version লেখা, হুবহু সেটাই install
```

---

## Scripts

### এখন যা আছে

```json
"scripts": {
	"dev": "tsx watch src/server.ts",
	"build": "tsup",
	"build:tsup": "tsup",
	"start": "node dist/server.js",
	"lint": "npx eslint ./src"
}
```

### Updated (সুপারিশ)

```json
"scripts": {
	"dev": "tsx watch src/server.ts",
	"lint": "eslint src",
	"lint:fix": "eslint src --fix",
	"typecheck": "tsc --noEmit",
	"build": "tsup",
	"start": "node dist/server.js"
}
```

| Script | Command | কাজ | কখন |
|---|---|---|---|
| `dev` | `tsx watch src/server.ts` | `.ts` সরাসরি চালায়; file save করলেই নিজে restart | Coding করার সময় |
| `lint` | `eslint src` | ESLint-এর নিয়ম ভাঙা code খোঁজে (error + warning দেখায়) | Commit / deploy-এর আগে |
| `lint:fix` | `eslint src --fix` | যে ভুলগুলো নিজে ঠিক করা যায় (import-এর ক্রম, `type` import ইত্যাদি) ঠিক করে দেয় | `lint`-এ অনেক ছোট ভুল এলে |
| `typecheck` | `tsc --noEmit` | শুধু TypeScript-এর type error খোঁজে, কোনো file বানায় না | Build-এর আগে — কারণ `tsup` type check করে না |
| `build` | `tsup` | `tsup.config.ts` অনুযায়ী `src` → `dist/server.js` বানায়, email template copy করে | Deploy-এর আগে |
| `start` | `node dist/server.js` | Build করা JavaScript চালায় — production-এ এটাই চলে | Server-এ / local-এ production test |

### কী বদলালাম

| আগে | এখন | কেন |
|---|---|---|
| `"build"` আর `"build:tsup"` দুটোই `tsup` | শুধু `"build"` | একই কাজ দুবার লেখা |
| `"lint": "npx eslint ./src"` | `"eslint src"` | npm script-এর ভেতরে project-এর package এমনিতেই পাওয়া যায়, `npx` লাগে না |
| — | `"typecheck"` যোগ | `tsup` type error ধরে না, এটা না থাকলে ভুল live-এ গিয়ে ধরা পড়ে ([note 38](./38-deploy-vercel-vps.md)) |
| — | `"lint:fix"` যোগ | ছোটখাটো ভুল এক command-এ ঠিক |

### কীভাবে চালাতে হয়

```bash
npm run dev         # "dev" বাদে সবগুলোর আগে "run" লাগে
npm run lint
npm run typecheck
npm run build
npm start           # "start" আর "test" — শুধু এই দুটো "run" ছাড়াও চলে
```

### Deploy-এর আগে ক্রম

```bash
npm run lint && npm run typecheck && npm run build && npm start
```

`&&` মানে আগেরটা সফল হলে তবেই পরেরটা চলবে — কোথাও error হলে সেখানেই থেমে যাবে।

---

## `package.json`-এর অন্য field

| Field | এখানে | মানে |
|---|---|---|
| `"type": "module"` | আছে | Project **ESM** (`import`/`export`)। এজন্যই `__dirname` নিজে বানাতে হয় ([note 32](./32-forgot-reset-password-nodemailer.md)), আর tsup-এর `banner`-এ `createRequire` লাগে |
| `"main": "dist/server.js"` | আছে | Project-এর মূল file — build-এর পরের entry point |
| `"description"`, `"author"` | খালি | চাইলে পূরণ করো, কাজে কোনো প্রভাব নেই |

---

## `dependencies` vs `devDependencies` — পার্থক্য কী?

- **`dependencies`** → যেসব package **production (live server)**-এও লাগবে — app আসলে **চালানোর জন্য** দরকার (যেমন `express`, `mongoose`, `nodemailer`)। Code-এ `import` করে runtime-এ ব্যবহার হয়।
- **`devDependencies`** → যেসব package শুধু **development**-এই লাগে — `typescript` (type check), `tsx` (dev-এ চালানো), `tsup` (build), `eslint` (code check), আর সব `@types/...` (শুধু type, runtime-এ কোনো ভূমিকা নেই)।

```bash
npm install express         # dependencies-এ যোগ হবে
npm install -D typescript   # devDependencies-এ যোগ হবে (-D বা --save-dev)
```

> ⚠️ **Server-এ build করলে devDependencies-ও লাগবে।** VPS-এ `npm run build` চালাতে `tsup` লাগে, আর `tsup` devDependency। তাই সেখানে `npm ci` (সব install) দিয়ে build করতে হবে — `npm ci --omit=dev` দিলে `tsup` থাকবে না, build fail করবে। শুধু আগে থেকে build করা `dist` চালালে তখন `--omit=dev` দেওয়া যায়।
>
> 💡 উল্টো ভুলও সাধারণ: কোনো runtime package (যেমন `axios`) ভুল করে `-D` দিয়ে install করলে local-এ কাজ করবে, কিন্তু `--omit=dev` দিয়ে install করা server-এ `Cannot find package 'axios'` error দেবে।