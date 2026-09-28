# `catchAsync` দিয়ে Controller-এর Repetition কমানো

## সমস্যা কী?

এখন আমাদের `createUser` controller দেখতে এরকম:

```ts
const createUser = async (req: Request, res: Response, next: NextFunction) => {
	try {
		const user = await UserServices.createUser(req.body);

		res.status(httpStatus.CREATED).json({
			message: "User created successfully",
			user,
		});
	} catch (err: any) {
		console.log(err);
		next(err);
	}
};
```

এখানে এই অংশটুকু **প্রতিটা controller-এ হুবহু একই** থাকবে:

```ts
async (req: Request, res: Response, next: NextFunction) => {
	try {
		// শুধু এই ভিতরের অংশটা আলাদা
	} catch (err: any) {
		console.log(err);
		next(err);
	}
};
```

১০টা controller থাকলে এই `try...catch` ১০ বার লিখতে হবে। Programming-এ একটা নিয়ম আছে **DRY (Don't Repeat Yourself)**, মানে একই কোড বারবার লেখা যাবে না। তাই এই repetition কমাব।

---

## Service Layer-এ `try...catch` লাগছে না কেন?

নিয়ম হলো `async` function-এ error হলে সেটা ধরতে হবে। কিন্তু service-এ আমরা `try...catch` দিইনি। কারণ:

Service-এ কোনো error হলে (বা আমরা `throw` করলে), সেই error নিজে থেকেই তাকে যে call করেছে তার কাছে চলে যায়। Service-কে call করে controller (`await UserServices.createUser()`), তাই error গিয়ে controller-এ পৌঁছায়, আর সেখানে ধরা হয়।

```
Service-এ error → নিজে থেকে উপরে যায় → Controller ধরে
```

তাই error শুধু এক জায়গায় (controller-এ) ধরলেই যথেষ্ট।

---

## Higher Order Function কী?

যে function **অন্য function-কে argument হিসেবে নেয়**, অথবা **function return করে** (বা দুটোই), তাকে **Higher Order Function** বলে।

আমরা যে `catchAsync` বানাব, সেটা দুটোই করে: একটা function নেয়, আর একটা নতুন function return করে।

---

## Step 1: `catchAsync` বানানো

`src/app`-এর ভিতরে `utils` নামে একটা folder বানাব, আর তার ভিতরে `catchAsync.ts` file নেব।

```
src/app/
├── config/
├── errorHelpers/
├── middlewares/
├── modules/
├── routes/
└── utils/
    └── catchAsync.ts   ← নতুন
```

`src/app/utils/catchAsync.ts`

```ts
/* eslint-disable @typescript-eslint/no-explicit-any */
import type { NextFunction, Request, Response } from "express";

type AsyncHandler = (req: Request, res: Response, next: NextFunction) => Promise<void>;

export const catchAsync =
	(fn: AsyncHandler) => (req: Request, res: Response, next: NextFunction) => {
		Promise.resolve(fn(req, res, next)).catch((err: any) => {
			console.log(err);
			next(err);
		});
	};
```

---

## `catchAsync` সহজ ভাষায় বোঝা

### একটা উদাহরণ দিয়ে

ধরো তুমি একটা রেস্টুরেন্টের মালিক। প্রতিটা রাঁধুনিকে আলাদা করে বলে দিতে হয়, "রান্নায় কোনো সমস্যা হলে ম্যানেজারকে জানাবে।" ১০ জন রাঁধুনি থাকলে ১০ বার একই কথা বলতে হয়।

তার বদলে তুমি একজন **supervisor** রাখলে। প্রতিটা রাঁধুনি supervisor-এর অধীনে কাজ করে। রাঁধুনি শুধু রান্না করে, আর কোনো সমস্যা হলে supervisor নিজেই সেটা ধরে ম্যানেজারকে জানিয়ে দেয়।

এখানে:

- **রাঁধুনি** = আমাদের controller (শুধু আসল কাজ করে)
- **Supervisor** = `catchAsync` (error ধরে)
- **ম্যানেজার** = `globalErrorHandler` (error-এর response পাঠায়)

### `try...catch` আর `.catch()` এর তুলনা

`catchAsync`-এর ভিতরে `try...catch` না লিখে `.catch()` ব্যবহার করা হয়েছে। আমাদের ক্ষেত্রে দুটো **একই কাজ** করে, শুধু লেখার ধরন আলাদা:

```ts
// try...catch দিয়ে (async/await style)
try {
	await fn(req, res, next);   // কাজ
} catch (err) {
	next(err);                  // error ধরা
}

// .catch() দিয়ে (Promise style) — catchAsync-এ এটাই আছে
Promise.resolve(fn(req, res, next))  // কাজ
	.catch((err) => {
		next(err);                     // error ধরা
	});
```

- `fn(req, res, next)` হলো `try`-এর ভিতরের কাজ।
- `.catch()` হলো `catch` block।
- `Promise.resolve()` নিজে `try`-এর মতো কিছু করে না, শুধু নিশ্চিত করে যে হাতে একটা Promise আছে, যাতে `.catch()` লাগানো যায়।

আমাদের controller `async`, আর `async` function-এ error হলে সেটার Promise reject হয়ে যায়। Reject হওয়া Promise-কে ধরে `.catch()`, তাই এটা `try...catch`-এর কাজটাই করে।

> **একটা ছোট পার্থক্য:** `async` ছাড়া সাধারণ (sync) function সরাসরি `throw` করলে `try...catch` সেটা ধরতে পারে, কিন্তু `Promise.resolve().catch()` পারে না। কারণ error-টা Promise তৈরি হওয়ার আগেই ঘটে যায়। তবে আমাদের সব controller `async`, তাই এই পার্থক্য আমাদের কোনো সমস্যা করবে না।

### কোডটা ভেঙে ভেঙে দেখি

**১. `type AsyncHandler`**

```ts
type AsyncHandler = (req: Request, res: Response, next: NextFunction) => Promise<void>;
```

এটা শুধু একটা type। এটা বলছে, `catchAsync`-কে এমন একটা `async` function দিতে হবে যেটা `req`, `res`, `next` নেয়। আমাদের controller ঠিক এরকমই।

**২. প্রথম arrow: `(fn: AsyncHandler) =>`**

```ts
export const catchAsync = (fn: AsyncHandler) => ...
```

`catchAsync` আমাদের controller-এর আসল কাজের function-টা নেয়, আর সেটাকে `fn` নামে রাখে।

**৩. দ্বিতীয় arrow: `(req, res, next) => { ... }`**

```ts
(req: Request, res: Response, next: NextFunction) => { ... }
```

`catchAsync` এই নতুন function-টা **return** করে। Express-এর route আসলে এই function-টাই পায়। Request আসলে Express এটাকে `req`, `res`, `next` দিয়ে call করে।

**৪. ভিতরের আসল কাজ**

```ts
Promise.resolve(fn(req, res, next)).catch((err: any) => {
	console.log(err);
	next(err);
});
```

- `fn(req, res, next)`: আমাদের controller-এর আসল কাজটা চালায়।
- `Promise.resolve(...)`: নিশ্চিত করে যে ফলাফলটা একটা Promise, যাতে `.catch()` ব্যবহার করা যায়।
- `.catch(...)`: কাজের মধ্যে কোনো error হলে এখানে ধরা পড়ে, আর `next(err)` দিয়ে `globalErrorHandler`-এ পাঠিয়ে দেয়।

মানে আগে আমরা প্রতিটা controller-এ যে `try...catch` লিখতাম, সেই কাজটা এখন `.catch()` একবারেই করে দিচ্ছে।

### পুরো Flow

```
Request আসে
   ↓
Express, catchAsync-এর return করা function-কে call করে
   ↓
সেই function আমাদের controller-এর কাজ (fn) চালায়
   ↓
সব ঠিক থাকলে → Response যায়
Error হলে    → .catch() ধরে → next(err) → globalErrorHandler
```

---

## Step 2: Controller-এ `catchAsync` ব্যবহার করা

এখন controller-এ আর `try...catch` লিখতে হবে না। শুধু আসল কাজটা `catchAsync(...)`-এর ভিতরে লিখব।

`src/app/modules/user/user.controller.ts`

```ts
import type { NextFunction, Request, Response } from "express";
import httpStatus from "http-status-codes";
import { catchAsync } from "../../utils/catchAsync";
import { UserServices } from "./user.service";

const createUser = catchAsync(async (req: Request, res: Response, next: NextFunction) => {
	const user = await UserServices.createUser(req.body);

	res.status(httpStatus.CREATED).json({
		message: "User created successfully",
		user,
	});
});

const getAllUsers = catchAsync(async (req: Request, res: Response, next: NextFunction) => {
	const users = await UserServices.getAllUsers();

	res.status(httpStatus.OK).json({
		success: true,
		message: "All users retrieved successfully",
		data: users,
	});
});

export const UserControllers = {
	createUser,
	getAllUsers,
};
```

> **ESLint note:** এখানে `next` কোথাও ব্যবহার হচ্ছে না, তাই ESLint `no-unused-vars` warning দিতে পারে। চাইলে controller থেকে `next` parameter-টা সরিয়ে দিতে পারো, কোড ঠিকমতোই চলবে।

---

## Step 3: বোঝার জন্য একটা `getAllUsers` route

`catchAsync` ভালোভাবে বোঝার জন্য সব user দেখার একটা route বানাই।

**Service** (`user.service.ts`):

```ts
const getAllUsers = async () => {
	const users = await User.find({});
	return users;
};

export const UserServices = {
	createUser,
	getAllUsers,
};
```

**Route** (`user.route.ts`):

```ts
router.get("/all-users", UserControllers.getAllUsers);
```

এখন এই URL-এ request পাঠালে সব user পাওয়া যাবে:

```
GET http://localhost:5000/api/v1/user/all-users
```

---

## আগে আর পরে

| আগে | পরে |
| --- | --- |
| প্রতিটা controller-এ `try...catch` | কোনো controller-এ `try...catch` নেই |
| প্রতিটায় `next(err)` লিখতে হতো | `catchAsync` একবারেই সব error ধরে |
| একই কোড বারবার | কোড ছোট, পরিষ্কার আর DRY |