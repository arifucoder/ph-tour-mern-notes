# Global Error Handler

## Global Error Handler কী?

এখন প্রতিটা controller-এর `catch` block-এ আলাদা করে error response পাঠাচ্ছি। Controller বাড়তে থাকলে একই কোড বারবার লিখতে হবে, আর সব জায়গায় error-এর format একরকম থাকবে না।

তাই আমরা একটা **Global Error Handler** বানাব। এটা একটা বিশেষ middleware, যেখানে পুরো app-এর সব error এসে জমা হবে, আর এক জায়গা থেকেই সব error-এর response পাঠানো হবে।

---

## Step 1: `app.ts`-এ একটা সাধারণ Error Handler

প্রথমে `app.ts`-এ একটা ছোট global error handler বানিয়ে দেখি:

```ts
app.use((err: any, req: Request, res: Response, next: NextFunction) => {
	res.status(500).json({
		success: false,
		message: err.message || "Something went wrong!",
		err,
		stack: envVars.NODE_ENV === "development" ? err.stack : null,
	});
});
```

### কিছু ব্যাখ্যা

- **৪টা parameter কেন?** Express শুধু parameter-এর সংখ্যা দেখে বোঝে কোনটা error middleware। সাধারণ middleware-এ ৩টা parameter থাকে (`req`, `res`, `next`), আর error middleware-এ ৪টা (`err`, `req`, `res`, `next`)। তাই `next` ব্যবহার না করলেও এটা অবশ্যই রাখতে হবে।
- **`stack`**: Error-টা কোডের কোন file-এর কোন লাইনে হয়েছে, সেটা `stack`-এ থাকে। এটা শুধু `development`-এ দেখাব। Production-এ দেখালে আমাদের কোডের ভিতরের তথ্য বাইরের মানুষ দেখে ফেলবে, যেটা নিরাপদ না।

---

## Step 2: Controller-এ `next(err)` ব্যবহার করা

এবার controller-এর `catch` block থেকে error response পাঠানোর কোড সরিয়ে দেব:

```ts
res.status(httpStatus.BAD_REQUEST).json({
	message: `Something went wrong! ${err.message}`,
	err,
});
```

তার বদলে লিখব:

```ts
next(err);
```

`next(err)` মানে "এই error-টা global error handler-এর কাছে পাঠিয়ে দাও"।

> **খেয়াল রাখো:** `next` ব্যবহার করতে হলে controller-এর parameter-এ `next: NextFunction` যোগ করতে হবে, আর `NextFunction` import করতে হবে। নাহলে `next` পাওয়া যাবে না।

`src/app/modules/user/user.controller.ts`

```ts
import type { NextFunction, Request, Response } from "express";
import httpStatus from "http-status";
import { UserServices } from "./user.service";

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

export const UserControllers = {
	createUser,
};
```

---

## Step 3: Error Handler আলাদা file-এ রাখা

Error handler-এর কোড `app.ts`-এ না রেখে আলাদা file-এ রাখব। `src/app`-এর ভিতরে `middlewares` নামে একটা folder বানাব, আর তার ভিতরে `globalErrorHandler.ts` file নেব।

```
src/app/
├── config/
├── middlewares/
│   └── globalErrorHandler.ts   ← নতুন
├── modules/
└── routes/
```

`src/app/middlewares/globalErrorHandler.ts`

```ts
import type { NextFunction, Request, Response } from "express";
import { envVars } from "../config/env";

// eslint-disable-next-line @typescript-eslint/no-explicit-any, @typescript-eslint/no-unused-vars
export const globalErrorHandler = (err: any, req: Request, res: Response, next: NextFunction) => {
	const statusCode = 500;
	const message = err.message || "Something went wrong!";

	res.status(statusCode).json({
		success: false,
		message,
		err,
		stack: envVars.NODE_ENV === "development" ? err.stack : null,
	});
};
```

> **ESLint comment কেন?** `any` ব্যবহার করেছি আর `next` কোথাও ব্যবহার করিনি, তাই ESLint দুটো error দেবে। কিন্তু উপরে বলেছি, `next` রাখতেই হবে, নাহলে Express এটাকে error handler হিসেবে চিনবে না। তাই এই লাইনের জন্য rule দুটো বন্ধ রেখেছি।

---

## Step 4: `app.ts`-এ যোগ করা

`src/app.ts`

```ts
import type { Request, Response } from "express";
import express from "express";
import cors from "cors";
import { router } from "./app/routes";
import { globalErrorHandler } from "./app/middlewares/globalErrorHandler";

const app = express();

// Middlewares
app.use(express.json());
app.use(cors());

// Routes
app.use("/api/v1", router);

app.get("/", (req: Request, res: Response) => {
	res.status(200).json({
		message: "Welcome to Tour Management System Backend",
	});
});

// Global error handler (সবার শেষে)
app.use(globalErrorHandler);

export default app;
```

> **Order গুরুত্বপূর্ণ:** `globalErrorHandler` অবশ্যই সব route-এর **পরে**, একদম শেষে রাখতে হবে। Route-এর আগে রাখলে route থেকে আসা error এটা ধরতে পারবে না।

---

## Error-এর Flow

```
Service-এ error → Controller-এর catch → next(err) → globalErrorHandler → Response
```

> **Note:** এখন সব error-এর জন্য status code `500` যাচ্ছে। কিন্তু সব error server-এর দোষে হয় না। যেমন ভুল data পাঠালে `400` হওয়া উচিত। পরে আমরা নিজস্ব error class বানিয়ে আলাদা আলাদা status code পাঠানোর ব্যবস্থা করব।