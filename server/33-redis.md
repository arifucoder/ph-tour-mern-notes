# 33 — Redis দিয়ে OTP: Email Verification

এই note-এ যা বানাব:

- **Redis** কী, কেন, কোথায় ব্যবহার হয়
- Project-এ Redis connect করা
- ৬ digit-এর **OTP** বানিয়ে Redis-এ ২ মিনিটের জন্য রাখা আর email-এ পাঠানো
- OTP **verify** করে user-কে `isVerified: true` করা
- Redis দিয়েই **বারবার OTP পাঠানো** আর **অনুমান করে OTP মেলানো** আটকানো

## পুরো Flow এক নজরে

```
① POST /api/v1/otp/send      { email }
      │  user আছে? আগেই verified না তো?
      │  ১ মিনিটের মধ্যে আবার চাইছে না তো?  (Redis cooldown)
      │  ৬ digit OTP বানাও → Redis: "otp:arif@gmail.com" = "274536"  (২ মিনিট পর নিজে মুছে যাবে)
      │  email পাঠাও
      ▼
② User email থেকে OTP দেখে frontend-এ লেখে
      ▼
③ POST /api/v1/otp/verify    { email, otp }
      │  Redis থেকে OTP আনো → না থাকলে "expired"
      │  মেলে না? → ভুলের সংখ্যা গোনো, ৫ বার ভুল হলে OTP বাতিল
      │  মিলে গেলে → MongoDB: isVerified = true, Redis থেকে OTP মুছে দাও
      ▼
   "OTP verified successfully" ✅
```

---

# Part 1: Redis কী?

## Step 1: In-memory Database

| | MongoDB / SQL | Redis |
|---|---|---|
| Data মূলত কোথায় থাকে | Disk (hard drive / SSD) | **RAM** (memory) |
| গতি | দ্রুত | **অনেক বেশি দ্রুত** (RAM disk-এর চেয়ে অনেক গুণ fast) |
| Data দেখতে কেমন | Collection, document, relation | সাধারণত **key → value** (`"otp:arif@gmail.com" → "274536"`) |
| কীসের জন্য | মূল, স্থায়ী data (user, tour, booking) | ছোট, দ্রুত দরকারি, অনেক সময় অস্থায়ী data |

Data RAM-এ থাকে বলে Redis-কে বলে **in-memory database**।

### RAM-এর উদাহরণ

Computer-এ কোনো app খুললে সেটা RAM-এ load হয়, তাই দ্রুত কাজ করা যায়। কাজ শেষে বন্ধ করলে RAM থেকে সরে যায়, অন্য জিনিস আসে। বিদ্যুৎ চলে গেলে RAM-এর সব হারিয়ে যায়, PC চালু করে app আবার খুলতে হয়। RAM-এর কাজ: **দ্রুত পড়া-লেখার জন্য data সাময়িক রাখা।**

Redis-ও data RAM-এ রাখে, তাই খুব দ্রুত।

> ✏️ **দুটো সংশোধন:**
>
> **① "RAM ভরে গেলে পুরোনো data নিজে সরিয়ে দেয়"** — Redis-এ এটা নির্ভর করে setting-এর (*eviction policy*) উপর। কিছু setting-এ সত্যিই পুরোনো/কম ব্যবহৃত data সরায়, আবার default অনেক জায়গায় memory ভরে গেলে **নতুন লেখাই বন্ধ করে দেয়** (error দেয়)। তাই "Redis নিজে নিজে পরিষ্কার হয়" ভেবে নেওয়া ঠিক না।
>
> **② OTP ২ মিনিট পর মুছে যায় RAM-এর কারণে না, আমাদের দেওয়া TTL-এর কারণে।** Redis-এ প্রতিটা key-র সাথে **TTL (Time To Live)** — মেয়াদ — বসিয়ে দেওয়া যায়: "এই key ১২০ second পর নিজে থেকে মুছে দাও"। RAM-এ থাকা data-ও TTL না দিলে থেকেই যায়। Redis চাইলে disk-এও backup রাখতে পারে (persistence), তাই restart-এ সব হারাবে এমনও না।
>
> OTP-র জন্য Redis পারফেক্ট মূলত এই **TTL feature**-এর জন্য।

### OTP কেন Redis-এ, MongoDB-তে না?

OTP = **One Time Password**:
- **একবারই** কাজে লাগে, verify হলেই শেষ।
- নির্দিষ্ট সময়ের (যেমন ২ মিনিট) মধ্যে ব্যবহার না হলে অচল।

MongoDB-তে রাখলে আমাদের নিজে মেয়াদ check করতে হতো, পুরোনো OTP মুছতে আলাদা ব্যবস্থা লাগত। Redis-এ শুধু বলে দিই "১২০ second পর মুছে দাও" — বাকিটা Redis নিজেই করে। Time শেষ হলে OTP আর কখনো পাওয়া যাবে না।

## Step 2: Redis আর কোথায় কাজে লাগে?

| ব্যবহার | উদাহরণ |
|---|---|
| **Caching** | যে data বারবার পড়া হয় (যেমন সব tour-এর list) MongoDB থেকে একবার এনে Redis-এ রাখা; পরের request গুলো Redis থেকে দ্রুত পায় |
| **OTP / অস্থায়ী token** | Email/phone verification, password reset code |
| **Session store** | Login session server-এ রাখা (express-session-এর store হিসেবে) |
| **Rate limiting** | "এক মিনিটে ১০০-র বেশি request না" — IP ধরে count রাখা (এই note-এর OTP cooldown-ও এরকম) |
| **Counter** | Page view, like count — `INCR` দিয়ে খুব দ্রুত |
| **Leaderboard** | Sorted Set দিয়ে score অনুযায়ী সাজানো top 10 |
| **Real-time / Pub-Sub** | Chat, notification — এক server message ছাড়ে, বাকিরা শোনে |
| **Job queue** | Email পাঠানো, image process করার মতো কাজ পরে background-এ করা (যেমন BullMQ library) |
| **Distributed lock** | একই সময়ে দুটো server একই কাজ যেন না করে (যেমন শেষ seat দুজন একসাথে book) |

---

# Part 2: Setup

## Step 3: Redis Cloud Account

1. https://redis.io → **Try Free** → account খোলো।
2. **Free** plan বেছে একটা database বানাও (ছোট, তবে শেখার জন্য যথেষ্ট)।
3. Database-এর **Connect** অংশে host, port, username (সাধারণত `default`), password পাবে।
4. Data দেখতে: **Redis Insight** (browser-এ "Launch Redis Insight web") — key, value আর কত সময় বাকি (TTL) সব দেখা যায়।

> ✏️ **সংশোধন:** "Redis আমাদের একটা AWS EC2 server দেয়" — ঠিক না। Redis Cloud একটা **managed service**: তারা AWS (বা Google Cloud / Azure)-এর server-এ Redis চালায়, আর আমাদের শুধু একটা database-এর ঠিকানা আর password দেয়। আমরা কোনো server/computer পাই না, শুধু database ব্যবহার করি — MongoDB Atlas-এর মতো।

Docs: https://redis.io/docs/latest/

## Step 4: Install আর Environment Variable

```bash
npm i redis
```

`redis` package নিজেই TypeScript type নিয়ে আসে, `@types` লাগে না।

### `.env`

```bash
REDIS_HOST=redis-12345.c1.asia-seast1-1.gce.redns.redis-cloud.com
REDIS_PORT=12345
REDIS_USERNAME=default
REDIS_PASSWORD=your_redis_password
```

### `env.ts` — ৩ জায়গায়

```ts
// ① interface
REDIS_HOST: string;
REDIS_PORT: string;
REDIS_USERNAME: string;
REDIS_PASSWORD: string;

// ② requiredEnvVariables
"REDIS_HOST",
"REDIS_PORT",
"REDIS_USERNAME",
"REDIS_PASSWORD",

// ③ return object
REDIS_HOST: process.env.REDIS_HOST as string,
REDIS_PORT: process.env.REDIS_PORT as string,
REDIS_USERNAME: process.env.REDIS_USERNAME as string,
REDIS_PASSWORD: process.env.REDIS_PASSWORD as string,
```

## Step 5: Redis Config

> ✏️ Note-এ লেখা ছিল "config **folder** বানাব"। আসলে `config` folder-এর ভেতরে একটা **file**: `redis.config.ts`।

```ts
// src/app/config/redis.config.ts
import { createClient } from "redis";
import { envVars } from "./env";

export const redisClient = createClient({
	username: envVars.REDIS_USERNAME,
	password: envVars.REDIS_PASSWORD,
	socket: {
		host: envVars.REDIS_HOST,
		port: Number(envVars.REDIS_PORT),
	},
});

// connection-এ পরে কোনো সমস্যা হলে (network কেটে গেলে) এখানে জানাবে
redisClient.on("error", (err) => {
	// eslint-disable-next-line no-console
	console.error("Redis Client Error:", err);
});

export const connectRedis = async () => {
	if (!redisClient.isOpen) {
		await redisClient.connect();
		// eslint-disable-next-line no-console
		console.log("✅ Redis connected");
	}
};
```

- **`createClient`**: Redis-এর সাথে কথা বলার client বানায় — এখনো connect করে না।
- **`connect()`**: আসল connection। App চালুর সময় একবার।
- **`isOpen`**: আগে থেকেই connected থাকলে আবার connect করার চেষ্টা করবে না (দুবার `connect()` করলে error দেয়)।
- **`on("error")`**: এটা না দিলে connection-এ কোনো error হলে Node সেটাকে unhandled error ভেবে **server crash** করে দিতে পারে।

### ❓ `connectRedis`-এ try/catch নেই কেন?

> Note-এ বাক্যটা অসম্পূর্ণ ছিল ("try catch kora hoi nai cause...")। কারণটা এমন:

এখানে error ধরে চুপ করে থাকার মানে হয় না — Redis connect না হলে OTP-র কোনো feature কাজই করবে না। তাই error **উপরে যেতে দেওয়া হয়েছে**: `server.ts` সেটা পেয়ে server চালুই হবে না, terminal-এ আসল error দেখা যাবে। একে বলে **fail fast** — ভাঙা অবস্থায় চালু থাকার চেয়ে শুরুতেই থেমে যাওয়া ভালো।

## Step 6: `server.ts`-এ চালু করা

```ts
// src/server.ts
(async () => {
	await connectRedis();
	await startServer();
	await seedSuperAdmin();
})();
```

Server চালুর আগে Redis connect। `npm run dev` করলে terminal-এ `✅ Redis connected` দেখলে config ঠিক আছে।

> 💡 Connect fail করলে পরিষ্কার message আর exit চাইলে:
> ```ts
> (async () => {
> 	try {
> 		await connectRedis();
> 		await startServer();
> 		await seedSuperAdmin();
> 	} catch (error) {
> 		// eslint-disable-next-line no-console
> 		console.error("❌ Failed to start server:", error);
> 		process.exit(1);
> 	}
> })();
> ```

### Redis-এর মূল command (বোঝার জন্য)

```ts
await redisClient.set("foo", "bar");                                    // রাখো
await redisClient.get("foo");                                           // "bar"
await redisClient.set("foo", "bar", { expiration: { type: "EX", value: 120 } }); // ১২০ second পর মুছে যাবে
await redisClient.del("foo");                                           // মুছে দাও
await redisClient.incr("count");                                        // 1, 2, 3... (সংখ্যা বাড়ায়)
```

> 💡 `{ expiration: { type: "EX", value } }` হলো `redis` package **v5**-এর লেখার ধরন। অনেক tutorial-এ v4-এর `{ EX: 120 }` দেখবে — একই কাজ, শুধু version আলাদা। `EX` = seconds, `PX` = milliseconds।

---

# Part 3: OTP Module

## Step 7: Folder আর Validation

```
src/app/modules/otp/
├── otp.validation.ts
├── otp.route.ts
├── otp.controller.ts
└── otp.service.ts
```

```ts
// src/app/modules/otp/otp.validation.ts
import { z } from "zod";

export const sendOtpZodSchema = z.object({
	email: z.email({ error: "Please provide a valid email address" }),
});

export const verifyOtpZodSchema = z.object({
	email: z.email({ error: "Please provide a valid email address" }),
	otp: z.string({ error: "OTP is required" }).regex(/^\d{6}$/, { error: "OTP must be exactly 6 digits" }),
});
```

> ⚠️ **যোগ করেছি:** Teacher-এর route-এ validation ছিল না। বিশেষ করে `otp`: Postman/frontend থেকে কেউ `"otp": 274536` (number) পাঠালে Redis-এর `"274536"` (string)-এর সাথে `!==` সবসময় `true` → সঠিক OTP দিয়েও "Invalid OTP"! Zod এখন string আর ঠিক ৬ digit নিশ্চিত করে।

## Step 8: Route

```ts
// src/app/modules/otp/otp.route.ts
import express from "express";
import { validateRequest } from "../../middlewares/validateRequest";
import { OTPController } from "./otp.controller";
import { sendOtpZodSchema, verifyOtpZodSchema } from "./otp.validation";

const router = express.Router();

router.post("/send", validateRequest(sendOtpZodSchema), OTPController.sendOTP); // OTP বানিয়ে email + Redis
router.post("/verify", validateRequest(verifyOtpZodSchema), OTPController.verifyOTP); // OTP মিলিয়ে verify

export const OtpRoutes = router;
```

```ts
// src/app/routes/index.ts — moduleRoutes-এ
{
	path: "/otp",
	route: OtpRoutes,
},
```

## Step 9: Controller

```ts
// src/app/modules/otp/otp.controller.ts
import type { Request, Response } from "express";
import httpStatus from "http-status-codes";
import { catchAsync } from "../../utils/catchAsync";
import { sendResponse } from "../../utils/sendResponse";
import { OTPService } from "./otp.service";

const sendOTP = catchAsync(async (req: Request, res: Response) => {
	const { email } = req.body as { email: string };
	await OTPService.sendOTP(email);

	sendResponse(res, {
		statusCode: httpStatus.OK,
		success: true,
		message: "OTP sent to your email successfully",
		data: null, // OTP response-এ না, শুধু email-এ যায়
	});
});

const verifyOTP = catchAsync(async (req: Request, res: Response) => {
	const { email, otp } = req.body as { email: string; otp: string };
	await OTPService.verifyOTP(email, otp);

	sendResponse(res, {
		statusCode: httpStatus.OK,
		success: true,
		message: "OTP verified successfully",
		data: null,
	});
});

export const OTPController = { sendOTP, verifyOTP };
```

### Teacher-এর code থেকে যা বদলেছি

- **Body থেকে `name` নেওয়া বাদ:** Teacher `{ email, name }` body থেকে নিয়ে email-এ বসাতেন। User তো DB-তে আছেই, তাই নাম **DB থেকে** নেব (Step 11)। Body থেকে নিলে যে কেউ যেকোনো email-এ আমাদের নাম দিয়ে নিজের লেখা পাঠাতে পারত (`"name": "Click here to win..."`)।
- **`data: null`-এর comment:** "OTP email-এ যায় তাই data নেই" কথাটা **sendOTP**-এর জন্য, verify-এর না — জায়গা ঠিক করেছি। (ভুল করেও OTP কখনো response-এ পাঠাবে না, তাহলে email যাচাইয়ের মানেই থাকে না।)
- **`200` → `httpStatus.OK`।**

## Step 10: OTP Email Template

```html
<!-- src/app/utils/templates/otp.ejs -->
<h2>Hello <%= name %>,</h2>
<p>Your OTP code is: <strong><%= otp %></strong></p>
<p>This code is valid for <%= expiresInMinutes %> minutes. Do not share it with anyone.</p>
```

`sendEmail.ts`-এ template-এর নাম যোগ করো ([note 32](./32-forgot-reset-password-nodemailer.md)):

```ts
type TEmailTemplate = "forgetPassword" | "otp";
```

> 💡 Teacher-এর template-এ "valid for 2 minutes" hard-coded ছিল। Service-এ সময় বদলালে email-এ ভুল লেখা যেত। এখন সময় data হিসেবে পাঠাই, দুটো সবসময় মিলবে।

## Step 11: Service

### OTP বানানো

```ts
import { randomInt } from "node:crypto";

const generateOtp = (length = 6) => randomInt(10 ** (length - 1), 10 ** length).toString();
```

- **`randomInt(min, max)`**: `min` থেকে `max`-এর **ঠিক আগ পর্যন্ত** (max বাদ) একটা random পূর্ণসংখ্যা।
- `length = 6` হলে: `randomInt(10 ** 5, 10 ** 6)` = `randomInt(100000, 1000000)` → **100000 থেকে 999999** — সবসময় ঠিক ৬ digit।
- **`Math.random()` না কেন?** `crypto.randomInt` cryptographically secure, অনুমান করা যায় না। OTP-র মতো নিরাপত্তার জিনিসে এটাই লাগে।

> ✏️ **হিসাবের সংশোধন:** Note-এ ছিল `10 ** 5 => 10*10*10*10*10*10 => 1000000`। কিন্তু `10 ** 5` মানে ১০-কে **৫ বার** গুণ = **100000** (৬ digit-এর সবচেয়ে ছোট সংখ্যা)। `1000000` হলো `10 ** 6`।

### Redis key-র নামকরণ

```ts
`otp:${email}`; // "otp:arif@gmail.com"
```

Redis-এ সব key এক জায়গায় থাকে, তাই key **unique** আর বোঝা যায় এমন হতে হবে। Convention হলো `কী-ধরনের:কার` — colon (`:`) দিয়ে ভাগ। Redis Insight-এ এগুলো folder-এর মতো গুছিয়ে দেখায়।

### দুটো নিরাপত্তা সমস্যা — আর Redis দিয়েই সমাধান

> ⚠️ **সমস্যা ১ — OTP spam:** `/otp/send`-এ কোনো সীমা ছিল না। কেউ loop চালিয়ে কারো email-এ হাজারটা OTP পাঠাতে পারত। তাছাড়া Gmail দিনে নির্দিষ্ট সংখ্যার বেশি email পাঠাতে দেয় না — limit শেষ হলে **সব user-এর** email (forgot password-ও) বন্ধ।
>
> **সমাধান — Cooldown:** OTP পাঠানোর পর Redis-এ ৬০ second-এর একটা চিহ্ন রাখি। চিহ্ন থাকলে নতুন OTP পাঠাব না।

> ⚠️ **সমস্যা ২ — Brute force:** ৬ digit = মাত্র ৯ লাখ সম্ভাবনা। ২ মিনিটে কোনো সীমা ছাড়া script দিয়ে হাজার হাজার বার চেষ্টা করে OTP মিলিয়ে ফেলা সম্ভব।
>
> **সমাধান — Attempt limit:** প্রতিটা ভুলে Redis-এ count বাড়াই (`INCR`)। ৫ বার ভুল হলে OTP বাতিল, নতুন OTP চাইতে হবে।

### Code

```ts
// src/app/modules/otp/otp.service.ts
import { randomInt } from "node:crypto";
import httpStatus from "http-status-codes";
import { redisClient } from "../../config/redis.config";
import AppError from "../../errorHelpers/AppError";
import { sendEmail } from "../../utils/sendEmail";
import { User } from "../user/user.model";

const OTP_EXPIRATION = 2 * 60; // ২ মিনিট (Redis-এ সময় second-এ)
const OTP_RESEND_COOLDOWN = 60; // ১ মিনিটের মধ্যে আবার পাঠানো যাবে না
const OTP_MAX_ATTEMPTS = 5; // সর্বোচ্চ ৫ বার ভুল

// Redis key — এক জায়গায় রাখলে বানান ভুল হয় না
const otpKey = (email: string) => `otp:${email}`;
const cooldownKey = (email: string) => `otp:cooldown:${email}`;
const attemptsKey = (email: string) => `otp:attempts:${email}`;

const generateOtp = (length = 6) => randomInt(10 ** (length - 1), 10 ** length).toString();

// user আছে আর এখনো verified না — দুই জায়গাতেই লাগে
const getUnverifiedUser = async (email: string) => {
	const user = await User.findOne({ email });

	if (!user) {
		throw new AppError(httpStatus.NOT_FOUND, "User not found.");
	}
	if (user.isVerified) {
		throw new AppError(httpStatus.BAD_REQUEST, "Your account is already verified.");
	}
	return user;
};

/* ─────────────── OTP পাঠানো ─────────────── */
const sendOTP = async (email: string) => {
	const user = await getUnverifiedUser(email);

	// cooldown: key না থাকলে (NX) ৬০ second-এর জন্য বসাও; আগে থেকে থাকলে null দেয়
	const canSend = await redisClient.set(cooldownKey(email), "1", {
		condition: "NX",
		expiration: { type: "EX", value: OTP_RESEND_COOLDOWN },
	});

	if (!canSend) {
		throw new AppError(
			httpStatus.TOO_MANY_REQUESTS,
			`Please wait ${OTP_RESEND_COOLDOWN} seconds before requesting a new OTP.`,
		);
	}

	const otp = generateOtp();

	// নতুন OTP রাখো (পুরোনো থাকলে overwrite) আর ভুলের count নতুন করে শুরু
	await redisClient.set(otpKey(email), otp, {
		expiration: { type: "EX", value: OTP_EXPIRATION },
	});
	await redisClient.del(attemptsKey(email));

	await sendEmail({
		to: email,
		subject: "Your PH Tour verification code",
		templateName: "otp",
		templateData: {
			name: user.name, // DB থেকে, body থেকে না
			otp,
			expiresInMinutes: OTP_EXPIRATION / 60,
		},
	});
};

/* ─────────────── OTP verify ─────────────── */
const verifyOTP = async (email: string, otp: string) => {
	const user = await getUnverifiedUser(email);

	const savedOtp = await redisClient.get(otpKey(email));

	// মেয়াদ শেষ (Redis নিজে মুছে দিয়েছে) বা কখনো চাওয়াই হয়নি
	if (!savedOtp) {
		throw new AppError(httpStatus.BAD_REQUEST, "OTP has expired or was not requested. Please request a new one.");
	}

	if (savedOtp !== otp) {
		const attempts = await redisClient.incr(attemptsKey(email)); // 1, 2, 3...

		if (attempts === 1) {
			// প্রথম ভুলের সময় count-এরও মেয়াদ দিয়ে দাও, যাতে চিরকাল থেকে না যায়
			await redisClient.expire(attemptsKey(email), OTP_EXPIRATION);
		}

		if (attempts >= OTP_MAX_ATTEMPTS) {
			await redisClient.del([otpKey(email), attemptsKey(email)]); // OTP বাতিল
			throw new AppError(httpStatus.TOO_MANY_REQUESTS, "Too many wrong attempts. Please request a new OTP.");
		}

		throw new AppError(
			httpStatus.BAD_REQUEST,
			`Invalid OTP. You have ${OTP_MAX_ATTEMPTS - attempts} attempt(s) left.`,
		);
	}

	// ① আগে DB-তে verified করো
	user.isVerified = true;
	await user.save();

	// ② তারপর OTP মুছে দাও — একবারই ব্যবহার করা যাবে
	await redisClient.del([otpKey(email), attemptsKey(email)]);
};

export const OTPService = {
	sendOTP,
	verifyOTP,
};
```

### নতুন Redis জিনিস

| Code | মানে |
|---|---|
| `condition: "NX"` | **N**ot e**X**ists — key **না থাকলে তবেই** set করো। থাকলে কিছু করে না, `null` return করে। Cooldown-এর জন্য নিখুঁত। |
| `redisClient.incr(key)` | Key-র সংখ্যা ১ বাড়ায় আর নতুন সংখ্যা return করে। Key না থাকলে ০ ধরে ১ বানায়। |
| `redisClient.expire(key, sec)` | আগে থেকে থাকা key-তে মেয়াদ বসানো। |
| `redisClient.del([k1, k2])` | একসাথে কয়েকটা key মোছা। |

### OTP verify — ধাপে ধাপে

| অবস্থা | ফলাফল |
|---|---|
| Redis-এ OTP নেই (২ মিনিট পার / চাওয়াই হয়নি) | 400 "OTP has expired..." |
| ভুল OTP (১ম–৪র্থ বার) | 400 "Invalid OTP. N attempt(s) left." |
| ভুল OTP (৫ম বার) | 429, OTP বাতিল → নতুন চাইতে হবে |
| সঠিক OTP | `isVerified = true`, Redis-এর key মুছে দেওয়া |
| আবার একই OTP দিয়ে verify | 400 "already verified" (OTP-ও আর নেই) |

### Teacher-এর code থেকে যা বদলেছি

| বিষয় | আগে | এখন | কেন |
|---|---|---|---|
| ⚠️ Send-এ সীমা | ছিল না | ৬০ second cooldown (`NX`) | OTP spam, Gmail limit শেষ হওয়া |
| ⚠️ Verify-এ সীমা | ছিল না | ৫ বার ভুলে OTP বাতিল (`INCR`) | Brute force |
| `name` | Body থেকে | DB থেকে | কেউ অন্যের email-এ নিজের লেখা ঢোকাতে পারত |
| Verify-এর শেষে | `Promise.all([updateOne, del])` | আগে DB update, তারপর `del` | একসাথে চালালে DB update fail করলেও OTP মুছে যেতে পারত — user verify-ও হলো না, OTP-ও গেল। এখন DB সফল হলেই OTP মোছে |
| "already verified" | `401` | `400` | 401 মানে login নেই/ভুল token — এখানে সেটা না |
| "Invalid OTP" | `401`, মেয়াদ শেষ আর ভুল একই message | `400`, আলাদা message | User বুঝবে নতুন OTP চাইবে নাকি আবার লিখবে |
| User check | দুই function-এ একই code | `getUnverifiedUser` helper | |
| `crypto` | `import crypto from "crypto"` | `import { randomInt } from "node:crypto"` | Built-in module-এ `node:` prefix — কোনটা Node-এর নিজের তা পরিষ্কার |
| Template-এ সময় | "2 minutes" hard-coded | `expiresInMinutes` data | |

---

## Step 12: Postman দিয়ে Test

### ১. OTP পাঠানো

```
POST localhost:5000/api/v1/otp/send
{ "email": "theextraschool@gmail.com" }
```

→ 200 + inbox-এ OTP। **Redis Insight**-এ দেখো:

```
otp:theextraschool@gmail.com            "274536"   TTL: 118s
otp:cooldown:theextraschool@gmail.com   "1"        TTL: 58s
```

TTL কমতে থাকবে, ০ হলে key নিজে থেকেই উধাও।

### ২. সাথে সাথে আবার পাঠাতে চাইলে

→ 429 "Please wait 60 seconds before requesting a new OTP."

### ৩. Verify

> ✏️ Note-এ verify-এর URL ভুল করে `/otp/send` লেখা ছিল। Verify হবে **`/otp/verify`**-এ।

```
POST localhost:5000/api/v1/otp/verify
{ "email": "theextraschool@gmail.com", "otp": "274536" }
```

→ 200 "OTP verified successfully"। MongoDB-তে `isVerified: true`, Redis থেকে key মুছে গেছে।

### ৪. আরও পরীক্ষা

| Request | Expected |
|---|---|
| ২ মিনিট পর verify | 400 "OTP has expired..." |
| ভুল OTP ৫ বার | ৫ম বারে 429, তারপর সঠিক OTP দিলেও "expired" |
| `"otp": 274536` (number) | 400 Zod: "OTP must be exactly 6 digits" |
| Verified user আবার send | 400 "already verified" |

---

## সারাংশ

| বিষয় | মনে রাখার কথা |
|---|---|
| Redis | In-memory (RAM) key-value database, খুব দ্রুত |
| TTL | Key-র মেয়াদ (`EX` seconds); OTP মুছে যায় **TTL-এর জন্য**, RAM-এর জন্য না |
| ব্যবহার | Cache, OTP, session, rate limit, counter, leaderboard, pub/sub, queue, lock |
| Redis Cloud | Managed service (AWS/GCP-এ চলে); আমরা শুধু database পাই |
| Connect | `createClient` → `on("error")` → `connect()` একবার, server চালুর আগে |
| Key | `otp:<email>` — `কী-ধরনের:কার` convention |
| OTP | `crypto.randomInt(100000, 1000000)` → সবসময় ৬ digit |
| Verify | মিলিয়ে → আগে DB, পরে `del` (একবারই ব্যবহার) |
| ⚠️ Cooldown | `set` + `NX` + `EX` → spam বন্ধ |
| ⚠️ Attempt limit | `INCR` → ৫ বার ভুলে OTP বাতিল |
| `otp` field | Zod-এ string + ৬ digit, নাহলে number-এ সবসময় "Invalid" |