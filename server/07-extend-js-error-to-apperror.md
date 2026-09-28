# Custom Error Class (`AppError`)

## কেন দরকার?

এখন আমরা কোনো error `throw` করলে global error handler সবসময় `500` status code পাঠায়। কারণ JavaScript-এর built-in `Error` class-এ শুধু `message` থাকে, কোনো `statusCode` থাকে না।

```ts
throw new Error("User not found"); // এখানে status code দেওয়ার উপায় নেই
```

কিন্তু সব error server-এর দোষে হয় না। যেমন user খুঁজে না পেলে `404`, ভুল data পাঠালে `400` হওয়া উচিত।

তাই JavaScript-এর `Error` class-কে **extend** করে আমরা নিজের একটা error class বানাব, যেখানে `message`-এর সাথে `statusCode`-ও যোগ করা যাবে।

### সুবিধা

- নিজের ইচ্ছামতো status code পাঠানো যায় (`400`, `404`, `401` ইত্যাদি)।
- কোডের যেকোনো জায়গা (service, controller) থেকে error `throw` করা যায়, আর সেটা সঠিক status code নিয়ে global error handler-এ পৌঁছায়।
- পুরো app-এ সব error একই format-এ যায়।

---

## Step 1: `AppError` class বানানো

`src/app`-এর ভিতরে `errorHelpers` নামে একটা folder বানাব, আর তার ভিতরে `AppError.ts` file নেব।

```
src/app/
├── config/
├── errorHelpers/
│   └── AppError.ts   ← নতুন
├── middlewares/
├── modules/
└── routes/
```

`src/app/errorHelpers/AppError.ts`

```ts
class AppError extends Error {
	public statusCode: number;

	constructor(statusCode: number, message: string, stack = "") {
		super(message);
		this.statusCode = statusCode;

		if (stack) {
			this.stack = stack;
		} else {
			Error.captureStackTrace(this, this.constructor);
		}
	}
}

export default AppError;
```

### লাইন ধরে ব্যাখ্যা

- **`extends Error`**: `AppError` হলো JavaScript-এর `Error` class-এর child। তাই `Error`-এর সব বৈশিষ্ট্য (`message`, `stack` ইত্যাদি) এটাও পেয়ে যায়।
- **`super(message)`**: Child class-এর constructor-এ আগে parent class (`Error`)-এর constructor call করতে হয়, আর সেটাই করে `super()`। `super(message)` মূলত `new Error("Something went wrong")` এর কাজটাই করে, মানে `message` সেট করে দেয়।
- **`this.statusCode = statusCode`**: এটাই আমাদের নতুন যোগ করা অংশ, যেটা `Error`-এ ছিল না।
- **`stack`**: Error-টা কোডের কোথায় হয়েছে, সেই তথ্য। `stack` property আসে `Error` class থেকে।
  - কেউ নিজে `stack` দিলে সেটাই ব্যবহার হবে।
  - না দিলে `Error.captureStackTrace()` নিজে থেকে stack তৈরি করবে। দ্বিতীয় argument হিসেবে `this.constructor` দেওয়ার মানে, stack-এ `AppError`-এর constructor-এর ভিতরের লাইনগুলো বাদ যাবে, শুধু আসল জায়গাটা দেখাবে যেখান থেকে error `throw` করা হয়েছে।

---

## Step 2: Global Error Handler-এ `AppError` যোগ করা

`src/app/middlewares/globalErrorHandler.ts`

```ts
/* eslint-disable @typescript-eslint/no-unused-vars */
/* eslint-disable @typescript-eslint/no-explicit-any */
import type { NextFunction, Request, Response } from "express";
import { envVars } from "../config/env";
import AppError from "../errorHelpers/AppError";

export const globalErrorHandler = (err: any, req: Request, res: Response, next: NextFunction) => {
	let statusCode = 500;
	let message = err.message || "Something went wrong!";

	if (err instanceof AppError) {
		statusCode = err.statusCode;
		message = err.message;
	} else if (err instanceof Error) {
		statusCode = 500;
		message = err.message;
	}

	res.status(statusCode).json({
		success: false,
		message,
		err,
		stack: envVars.NODE_ENV === "development" ? err.stack : null,
	});
};
```

### এটা কীভাবে কাজ করে?

- Error যদি `AppError` হয়, তাহলে তার নিজের `statusCode` আর `message` ব্যবহার হবে।
- সাধারণ `Error` হলে `500` যাবে।

> **Order গুরুত্বপূর্ণ:** `AppError`-এর check অবশ্যই `Error`-এর **আগে** রাখতে হবে। কারণ `AppError` নিজেও একটা `Error` (সে `Error`-কে extend করেছে)। তাই `Error`-এর check আগে থাকলে `AppError`-ও সেখানেই ধরা পড়ে যাবে, আর সবসময় `500` যাবে।

---

## Step 3: ব্যবহার করা

এখন যেকোনো জায়গা থেকে status code সহ error `throw` করা যাবে। যেমন একই email দিয়ে দুইবার user বানানো আটকাতে চাই।

### Service-এ error `throw` করা

`src/app/modules/user/user.service.ts`

```ts
import httpStatus from "http-status";
import AppError from "../../errorHelpers/AppError";
import type { IUser } from "./user.interface";
import { User } from "./user.model";

const createUser = async (payload: Partial<IUser>) => {
	const { name, email } = payload;

	const isUserExist = await User.findOne({ email });

	if (isUserExist) {
		throw new AppError(httpStatus.BAD_REQUEST, "User already exists");
	}

	const user = await User.create({ name, email });

	return user;
};

export const UserServices = {
	createUser,
};
```

### Error-এর Flow

```
Service-এ throw new AppError() → Controller-এর catch → next(err) → globalErrorHandler → Response
```

Global error handler দেখবে error-টা `AppError`, তাই তার নিজের status code `400` আর message পাঠাবে। Response-এ এরকম থাকবে (সাথে `err` আর `stack`-ও থাকবে):

```json
{
	"success": false,
	"message": "User already exists"
}
```

---

## Step 4: Controller-এ `try...catch` দিয়ে error ধরা

Service থেকে `throw` করা error controller-এর `catch` block-এ এসে ধরা পড়বে। সেখান থেকে `next(err)` দিয়ে global error handler-এ পাঠিয়ে দেব।

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

> **Mongoose error-এর কী হবে?** Mongoose-এর error (যেমন required field না দিলে validation error) `AppError` না, সাধারণ `Error`। তাই এগুলোও controller-এর `catch` হয়ে global error handler-এ যাবে, কিন্তু status code যাবে `500`। Mongoose error গুলো আলাদা করে handle করা আমরা পরে শিখব।