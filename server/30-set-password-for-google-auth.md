# 32 — Google User-এর জন্য Password Set করা

## কেন দরকার?

যে user **Google** দিয়ে login করেছে, তার DB-তে কোনো password নেই:

```js
{
  email: "arif@gmail.com",
  password: undefined,              // ← নেই
  auths: [{ provider: "google", providerId: "1098..." }]
}
```

এখন সে যদি **একই Gmail আর একটা password** দিয়েও login করতে চায়, পারবে না। Passport local strategy ([note 24](./24-passport-local.md)) তাকে বলে দেয়: "You have authenticated through Google... set a password..."

তাই একটা API বানাব যেটা দিয়ে Google user **অতিরিক্ত** একটা password set করতে পারবে। এরপর সে দুইভাবেই login করতে পারবে:

```
আগে:  auths = [google]                 → শুধু "Continue with Google"
পরে:  auths = [google, credentials]    → Google ✅  অথবা  email + password ✅
```

## Set Password vs Change Password

দুটো আলাদা কাজ, গুলিয়ে ফেলা যাবে না:

| | Set Password | Change Password |
|---|---|---|
| Route | `POST /auth/set-password` | `POST /auth/change-password` |
| কার জন্য | যার **কোনো password নেই** (Google user) | যার **আগে থেকেই password আছে** |
| পুরোনো password লাগে? | না (পুরোনো বলে কিছু নেই) | **হ্যাঁ**, পুরোনোটা মিলতে হবে |
| একবার set হওয়ার পর | আর ব্যবহার করা যাবে না | যতবার খুশি |

> 🔑 **মূল নিয়ম:** Set Password পুরোনো password চায় না। তাই এটা শুধু তাদের জন্যই খোলা থাকতে পারে যাদের **password নেই**। Password থাকলে অবশ্যই Change Password দিয়ে যেতে হবে।

## Flow

```
POST /api/v1/auth/set-password
Authorization: <accessToken>      (Google login থেকে পাওয়া)
Body: { "password": "Arif@1234" }
   │
   ▼
checkAuth            → login করা আছে? token থেকে userId
   ▼
validateRequest      → password-এর rule মানছে?
   ▼
controller → service
   ├─ user আছে?                 না → 404
   ├─ password আগে থেকেই আছে?    হ্যাঁ → 400 "change-password ব্যবহার করো"
   ├─ bcrypt দিয়ে hash
   ├─ auths-এ "credentials" যোগ (আগে না থাকলে)
   └─ save
   ▼
200 "Password set successfully"
```

---

## Step 1: Route

```ts
// src/app/modules/auth/auth.route.ts
import { validateRequest } from "../../middlewares/validateRequest";
import { setPasswordZodSchema } from "./auth.validation";

router.post(
	"/set-password",
	checkAuth(...Object.values(Role)),
	validateRequest(setPasswordZodSchema), // ← যোগ করেছি
	AuthControllers.setPassword,
);
```

- **`checkAuth(...Object.values(Role))`**: যেকোনো role-এর login করা user। Token থেকেই বুঝব কার password set হচ্ছে, তাই body-তে email বা userId নেব না (নাহলে কেউ অন্যের account-এ password বসিয়ে দিত)।

> ⚠️ **যোগ করেছি — Validation:** Teacher-এর route-এ কোনো validation ছিল না। ফলে:
> - Body-তে `password` না পাঠালে `bcryptjs.hash(undefined)` → `Illegal arguments` → **500**।
> - `"1"` এর মতো দুর্বল password-ও set হয়ে যেত, অথচ registration-এ আমরা শক্ত password চাই।

## Step 2: Validation

```ts
// src/app/modules/auth/auth.validation.ts
import { z } from "zod";

export const setPasswordZodSchema = z.object({
	password: z
		.string({ error: "Password is required" })
		.min(8, { error: "Password must be at least 8 characters long" })
		.regex(/[A-Z]/, { error: "Password must contain at least 1 uppercase letter" })
		.regex(/\d/, { error: "Password must contain at least 1 number" })
		.regex(/[!@#$%^&*]/, { error: "Password must contain at least 1 special character (!@#$%^&*)" }),
});
```

> 💡 `user.validation.ts`-এ registration-এর জন্য যে password rule আছে, এখানে **হুবহু সেটাই** ব্যবহার করো। সবচেয়ে ভালো হলো rule-টা একটা আলাদা `passwordZodSchema` হিসেবে export করে দুই জায়গাতেই import করা, যাতে একটা বদলালে দুটোই বদলায়।

---

## Step 3: Controller

```ts
// src/app/modules/auth/auth.controller.ts
const setPassword = catchAsync(async (req: Request, res: Response) => {
	const decodedToken = req.user as JwtPayload;
	const { password } = req.body as { password: string };

	await AuthServices.setPassword(decodedToken.userId as string, password);

	sendResponse(res, {
		success: true,
		statusCode: httpStatus.OK,
		message: "Password set successfully. You can now also log in with your email and password.",
		data: null,
	});
});
```

### Teacher-এর code থেকে যা বদলেছি

- **Message:** `"Password Changed Successfully"` ছিল, কিন্তু এখানে password **change** হয়নি, প্রথমবার **set** হয়েছে। Change Password API-র message-এর সাথে গুলিয়ে যেত।
- **`next` সরিয়েছি:** ব্যবহার হচ্ছিল না, ESLint `no-unused-vars` error দেয়। `catchAsync` নিজেই error `next`-এ পাঠায়।
- **`decodedToken.userId as string`**: `JwtPayload`-এর custom field-এর type TypeScript জানে না (note 30)।

---

## Step 4: Service — Teacher-এর Logic-এ কী সমস্যা?

### Teacher-এর check

```ts
if (user.password && user.auths.some((providerObject) => providerObject.provider === "google")) {
	throw new AppError(httpStatus.BAD_REQUEST, "You have already set you password...");
}
```

মানে: **"password আছে এবং Google user"** — দুটোই সত্যি হলে তবেই আটকাও।

তুমি ঠিকই সন্দেহ করেছ, এই logic ভুল। তিন ধরনের user দিয়ে চালিয়ে দেখি:

| User | `password` | `auths` | Teacher-এর code | কী হওয়া উচিত |
|---|---|---|---|---|
| ① শুধু Google | নেই | `[google]` | Set হয় ✅ | Set হবে ✅ |
| ② Google + আগেই set করেছে | আছে | `[google, credentials]` | আটকায় ✅ | আটকাবে ✅ |
| ③ শুধু email/password দিয়ে registered | আছে | `[credentials]` | **Set হয়ে যায়** ❌ | আটকাবে |

**User ③-এর ক্ষেত্রে কী হয়?** তার password আছে, কিন্তু সে Google user না। তাই `user.password && isGoogle` → `true && false` → `false` → check পার হয়ে যায়। ফলে:

1. **🚨 পুরোনো password ছাড়াই password বদলে যায়!** Change Password-এ পুরোনো password চাওয়ার পুরো উদ্দেশ্যই এটা আটকানো। ধরো কেউ এক মুহূর্তের জন্য user-এর খোলা browser বা চুরি হওয়া token পেল। সে `/set-password` দিয়ে নতুন password বসিয়ে দেবে, আসল মালিক আর login-ই করতে পারবে না — **account চুরি**।
2. **`auths`-এ দুটো `credentials` হয়ে যায়:** `[credentials, credentials]` — duplicate data।

### সমাধান: শুধু একটা প্রশ্ন — "password আছে কি?"

User Google দিয়ে এসেছে নাকি অন্যভাবে — সেটা এখানে গুরুত্বপূর্ণ না। গুরুত্বপূর্ণ শুধু:

```
password নেই → set করতে দাও
password আছে → আটকাও, change-password ব্যবহার করতে বলো
```

| User | `!user.password` | ফলাফল |
|---|---|---|
| ① শুধু Google | `true` | Set ✅ |
| ② Google + set করা | `false` | আটকায় ✅ |
| ③ শুধু credentials | `false` | আটকায় ✅ |

### Code

```ts
// src/app/modules/auth/auth.service.ts
import bcryptjs from "bcryptjs";
import httpStatus from "http-status-codes";
import { envVars } from "../../config/env";
import AppError from "../../errorHelpers/AppError";
import type { IAuthProvider } from "../user/user.interface";
import { User } from "../user/user.model";

const setPassword = async (userId: string, plainPassword: string) => {
	const user = await User.findById(userId);

	if (!user) {
		throw new AppError(httpStatus.NOT_FOUND, "User not found.");
	}

	// Password আগে থেকেই থাকলে (যেভাবেই আসুক) এখানে বদলানো যাবে না
	if (user.password) {
		throw new AppError(
			httpStatus.BAD_REQUEST,
			"You already have a password. Please use change password to update it.",
		);
	}

	user.password = await bcryptjs.hash(plainPassword, Number(envVars.BCRYPT_SALT_ROUND));

	// "credentials" provider আগে না থাকলে তবেই যোগ করো (duplicate এড়াতে)
	const hasCredentialsProvider = user.auths.some((providerObject) => providerObject.provider === "credentials");

	if (!hasCredentialsProvider) {
		const credentialProvider: IAuthProvider = {
			provider: "credentials",
			providerId: user.email,
		};
		user.auths = [...user.auths, credentialProvider];
	}

	await user.save();
};
```

### প্রতিটা অংশ

- **`user.password` check:** উপরের table-এর মতো, একটা শর্তেই তিন ধরনের user ঠিকভাবে সামলানো যায়।
- **`bcryptjs.hash`:** Plain password কখনো DB-তে রাখি না, registration-এর মতোই hash করে রাখি।
- **`credentials` provider যোগ:** এতে DB-তেই লেখা থাকে user কোন কোন উপায়ে login করতে পারে। `providerId` হিসেবে email, কারণ credentials login-এ email-ই পরিচয়।
- **`hasCredentialsProvider` check:** কোনো কারণে (পুরোনো data, আগের bug) `credentials` আগে থেকেই থাকলে আবার যোগ হবে না।
- **`user.save()`:** `findById` দিয়ে আনা document-এ বদল করে `save()` করলে Mongoose schema validation চালায়।

### Teacher-এর code থেকে যা বদলেছি

| বিষয় | আগে | এখন |
|---|---|---|
| 🚨 Check | `user.password && isGoogle` | `user.password` — credentials user আর old-password ছাড়া password বদলাতে পারবে না |
| Duplicate provider | সবসময় যোগ | না থাকলে তবেই যোগ |
| Status | `404` (সংখ্যা) | `httpStatus.NOT_FOUND` |
| Message | "You have already set **you** password. Now you can change the password from your profile password update" | "You already have a password. Please use change password to update it." |
| `auths` | আলাদা variable বানিয়ে assign | একই কাজ, if-এর ভেতরে |

---

## Step 5: Test

### ১. Google user — প্রথমবার set

```
POST /api/v1/auth/set-password
Authorization: <Google login-এর accessToken>
{ "password": "Arif@1234" }
```

→ `200 "Password set successfully..."`। DB-তে:

```js
{
  email: "arif@gmail.com",
  password: "$2b$10$Xk...",    // hash
  auths: [
    { provider: "google", providerId: "1098..." },
    { provider: "credentials", providerId: "arif@gmail.com" }
  ]
}
```

### ২. এখন email + password দিয়ে login

```
POST /api/v1/auth/login
{ "email": "arif@gmail.com", "password": "Arif@1234" }
```

→ ✅ Login সফল। Passport local strategy-র `isGoogleAuthenticated && !isUserExist.password` check এখন আর আটকায় না, কারণ password আছে। Google login-ও আগের মতোই চলবে।

### ৩. একই user আবার set করতে চাইলে

→ `400 "You already have a password. Please use change password to update it."`

### ৪. সাধারণ (credentials) user set করতে চাইলে

→ `400` একই message। (Teacher-এর code-এ এখানে পুরোনো password ছাড়াই password বদলে যেত।)

### ৫. Password ছাড়া বা দুর্বল password

→ `400` Zod error, `errorSources`-এ কোন rule ভাঙল তা দেখাবে (note 25)।

---

## সারাংশ

| বিষয় | মনে রাখার কথা |
|---|---|
| উদ্দেশ্য | Google user অতিরিক্ত password বসিয়ে email + password দিয়েও login করতে পারবে |
| Set vs Change | Set = password নেই, old password লাগে না; Change = password আছে, old password লাগে |
| 🚨 মূল check | `if (user.password)` → আটকাও। Provider দেখে না, শুধু password আছে কিনা |
| Teacher-এর bug | `password && isGoogle` → credentials user old password ছাড়াই password বদলাতে পারত |
| `auths` | `credentials` যোগ, তবে duplicate না |
| User কে | Token থেকে (`checkAuth`), body থেকে না |
| Validation | Registration-এর মতোই শক্ত password rule |