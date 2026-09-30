# Express-এ Custom Type Declaration (`req.user`)

## সমস্যা কী?

Verified token দরকার হলে প্রতিবার আমাদের token আবার verify করতে হচ্ছে। যেমন `checkAuth.ts`-এ একবার verify করছি, আবার `user.controller.ts`-এও করতে হচ্ছে:

```ts
const token = req.headers.authorization;
const verifiedToken = verifyToken(token as string, envVars.JWT_ACCESS_SECRET) as JwtPayload;
```

অথচ এই route আগে থেকেই `checkAuth` দিয়ে protected, মানে `checkAuth` token-টা **আগেই verify করে ফেলেছে**। একই কাজ দুইবার করার কোনো মানে নেই।

## সমাধান

`checkAuth`-এ verify করা token-টা `req`-এর ভিতরে রেখে দেব:

```ts
// checkAuth.ts
req.user = verifiedToken;
```

`req` object middleware থেকে controller পর্যন্ত একই থাকে। তাই controller-এ শুধু এটুকু লিখলেই verified token পেয়ে যাব:

```ts
// user.controller.ts
const verifiedToken = req.user;
```

```
Request → checkAuth (verify করে req.user-এ রাখে) → Controller (req.user থেকে নেয়)
```

কিন্তু `checkAuth.ts`-এ `req.user` লিখলেই TypeScript error দেয়:

```
Property 'user' does not exist on type 'Request'
```

কারণ Express-এর `Request` type-এ `user` নামে কোনো property নেই। এই error ঠিক করতে আমাদের **custom type declaration** করতে হবে।

---

## Custom Type Declaration কী আর কেন?

আমরা যে third-party package (যেমন `express`) ব্যবহার করি, সেগুলোর কোডের উপর আমাদের কোনো নিয়ন্ত্রণ নেই। তাদের কোড বা type declaration আমরা বদলাতে পারি না। কিন্তু চাইলে তাদের type-এ **নিজেদের কিছু property যোগ** করে দিতে পারি।

এটা দুই ক্ষেত্রে দরকার হয়:

1. **কোনো package-এর type-এ নতুন কিছু যোগ করতে চাইলে।** যেমন আমরা Express-এর `Request`-এ `user` যোগ করতে চাই।
2. **কোনো package-এর type file-ই না থাকলে।** কিছু package (যেমন ShurjoPay) TypeScript support দেয় না। TypeScript project-এ সেগুলো ব্যবহার করতে গেলে অনেক error আসে। তখন নিজেদেরই type declaration file বানাতে হয়।

---

## Step 1: Global `interfaces` folder বানানো

`src/app`-এর ভিতরে `interfaces` নামে একটা folder বানাব। এখানের interface গুলো কোনো নির্দিষ্ট module-এর না, **পুরো app-এ** ব্যবহার হবে। তার ভিতরে `index.d.ts` file নেব।

```
src/app/
├── config/
├── errorHelpers/
├── interfaces/
│   └── index.d.ts   ← নতুন
├── middlewares/
├── modules/
├── routes/
└── utils/
```

`src/app/interfaces/index.d.ts`

```ts
import type { JwtPayload } from "jsonwebtoken";

declare global {
	namespace Express {
		interface Request {
			user: JwtPayload;
		}
	}
}
```

---

## কোডটা বোঝা

- **`declare global`**: এই পরিবর্তন শুধু এই file-এ না, **পুরো application-এ** কাজ করবে।
- **`namespace Express`**: Express-এর type গুলোর ভিতরে ঢুকছি।
- **`interface Request`**: Express-এর যে `Request` type আছে, হুবহু সেই নামটাই দিচ্ছি। TypeScript-এ একই নামের interface আবার লিখলে নতুন property পুরোনোটার সাথে **যোগ হয়ে যায়**, পুরোনোটা মুছে যায় না। একে বলে **declaration merging**।
- **`user: JwtPayload`**: Token verify করার পর যে data পাই (`userId`, `email`, `role`), সেটাই `req.user`-এ রাখছি। তাই এর type `JwtPayload`।

---

## ❓ File-এর নাম কি `index.d.ts`-ই দিতে হবে?

না। নাম যেকোনো কিছু হতে পারে, যেমন `express.d.ts`। শুধু দুটো জিনিস জরুরি:

- **Extension `.d.ts` হতে হবে।** `d` মানে declaration। এই file-এ শুধু type থাকে, কোনো চলমান কোড থাকে না।
- **`index` নামটা একটা প্রচলিত নিয়ম**, যাতে বোঝা যায় এটা এই folder-এর মূল file।

## ❓ `interfaces` folder কি অবশ্যই `app`-এর ভিতরেই হতে হবে?

না, `app`-এর ভিতরেই হতে হবে এমন কোনো নিয়ম নেই। আসল শর্ত হলো file-টা এমন জায়গায় থাকতে হবে যেটা TypeScript দেখতে পায়। আমাদের `tsconfig.json`-এ আছে:

```json
"include": ["src"]
```

তাই `src`-এর ভিতরে যেকোনো জায়গায় রাখলেই চলবে। `src`-এর **বাইরে** রাখলে TypeScript file-টা খুঁজে পাবে না। আমরা গুছিয়ে রাখার জন্য `src/app/interfaces`-এ রেখেছি।

এই কারণে file-টা কোথাও import-ও করতে হয় না, TypeScript নিজেই খুঁজে নেয়।

---

## Step 2: ব্যবহার করা

**`checkAuth.ts`**-এ token verify করে `req.user`-এ রাখা:

```ts
const verifiedToken = verifyToken(accessToken, envVars.JWT_ACCESS_SECRET) as JwtPayload;
req.user = verifiedToken;
next();
```

**`user.controller.ts`**-এ সরাসরি নেওয়া:

```ts
const verifiedToken = req.user;
```

এখন আর error দেখাবে না, আর `req.user.role`, `req.user.email` লিখতে গেলে suggestion-ও পাওয়া যাবে।

> **Error তবুও না গেলে:** VS Code-এ `Ctrl + Shift + P` চেপে **"TypeScript: Restart TS Server"** চালাও, অথবা VS Code reload করো।