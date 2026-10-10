# `sendResponse` দিয়ে Response একরকম রাখা

## সমস্যা কী?

প্রতিটা controller-এ আমরা এভাবে response পাঠাচ্ছি:

```ts
res.status(httpStatus.CREATED).json({
	message: "User created successfully",
	user,
});
```

এতে দুটো সমস্যা:

- একই ধরনের কোড বারবার লিখতে হচ্ছে।
- একেক controller-এ response-এর format একেক রকম হয়ে যাচ্ছে। যেমন একটায় `user`, আরেকটায় `data`, কোথাও `success` আছে, কোথাও নেই। Frontend developer-এর জন্য এটা ঝামেলার।

তাই একটা function বানাব, যেটা `statusCode`, `message` আর `data` নিয়ে সবসময় **একই format-এ** JSON response বানিয়ে পাঠিয়ে দেবে।

---

## Step 1: `sendResponse` বানানো

`src/app/utils` folder-এ `sendResponse.ts` নামে একটা file নেব।

```
src/app/utils/
├── catchAsync.ts
└── sendResponse.ts   ← নতুন
```

`src/app/utils/sendResponse.ts`

```ts
import type { Response } from "express";

interface TMeta {
    page: number;
    limit: number;
    totalPage: number;
    total: number
}

interface TResponse<T> {
	statusCode: number;
	success: boolean;
	message: string;
	data: T;
	meta?: TMeta;
}

export const sendResponse = <T>(res: Response, data: TResponse<T>) => {
	res.status(data.statusCode).json({
		statusCode: data.statusCode,
		success: data.success,
		message: data.message,
		meta: data.meta,
		data: data.data,
	});
};
```

---

## কোডটা বোঝা

### Generic type `<T>` কেন?

`data`-তে কী থাকবে সেটা আগে থেকে জানা নেই। একেক API একেক রকম data পাঠায়। যেমন `createUser` পাঠায় একটা user object, আর `getAllUsers` পাঠায় user-এর একটা array।

তাই একটা নির্দিষ্ট type না দিয়ে **generic type `T`** নিয়েছি। `T` মানে "যেকোনো type, যেটা ব্যবহারের সময় ঠিক হবে"। আমরা যে data দেব, TypeScript নিজেই বুঝে নেবে `T` কী।

### `meta` কী?

`meta` হলো data সম্পর্কে কিছু বাড়তি তথ্য, যেমন মোট কতগুলো data আছে, pagination, filtering-এর তথ্য ইত্যাদি। সব API-তে এটা লাগে না, তাই এটা **optional** (`meta?`)। আপাতত শুধু `total` রেখেছি, পরে দরকার অনুযায়ী আরও field যোগ করব।

> **Note:** `meta` না দিলে এর value হয় `undefined`, আর JSON-এ `undefined` field বাদ পড়ে যায়। তাই যেসব response-এ `meta` নেই, সেগুলোতে `meta` field দেখাই যাবে না।

### Function-এর parameter

`sendResponse` একটা **generic function**, যেটা দুটো parameter নেয়:

1. **`res`**: Express-এর `Response` object, যেটা দিয়ে response পাঠানো হয়।
2. **`data`**: Response-এর সব তথ্য একটা object-এ (`statusCode`, `success`, `message`, `data`, `meta`)।

---

## Step 2: Controller-এ ব্যবহার করা

### `createUser`

আগে:

```ts
res.status(httpStatus.CREATED).json({
	message: "User created successfully",
	user,
});
```

এখন:

```ts
sendResponse(res, {
	statusCode: httpStatus.CREATED,
	success: true,
	message: "User created successfully",
	data: user,
});
```

---

## Step 3: `meta` test করা

`meta` কাজ করছে কিনা দেখার জন্য `getAllUsers`-এ মোট user-এর সংখ্যা যোগ করব।

### Service

`User.countDocuments()` collection-এ মোট কতগুলো document আছে সেটা গুনে দেয়।

`user.service.ts`

```ts
const getAllUsers = async () => {
	const users = await User.find({});
	const totalUsers = await User.countDocuments();

	return {
		data: users,
		meta: {
			total: totalUsers,
		},
	};
};
```

### Controller

Service এখন একটা object return করছে (`data` আর `meta` সহ), তাই আগে সেটা `result`-এ রেখে তারপর ব্যবহার করব।

`user.controller.ts`

```ts
const getAllUsers = catchAsync(async (req: Request, res: Response, next: NextFunction) => {
	const result = await UserServices.getAllUsers();

	sendResponse(res, {
		statusCode: httpStatus.OK,
		success: true,
		message: "All users retrieved successfully",
		data: result.data,
		meta: result.meta,
	});
});
```

---

## পুরো Controller

`src/app/modules/user/user.controller.ts`

```ts
import type { NextFunction, Request, Response } from "express";
import httpStatus from "http-status-codes";
import { catchAsync } from "../../utils/catchAsync";
import { sendResponse } from "../../utils/sendResponse";
import { UserServices } from "./user.service";

const createUser = catchAsync(async (req: Request, res: Response, next: NextFunction) => {
	const user = await UserServices.createUser(req.body);

	sendResponse(res, {
		statusCode: httpStatus.CREATED,
		success: true,
		message: "User created successfully",
		data: user,
	});
});

const getAllUsers = catchAsync(async (req: Request, res: Response, next: NextFunction) => {
	const result = await UserServices.getAllUsers();

	sendResponse(res, {
		statusCode: httpStatus.OK,
		success: true,
		message: "All users retrieved successfully",
		data: result.data,
		meta: result.meta,
	});
});

export const UserControllers = {
	createUser,
	getAllUsers,
};
```

---

## Response দেখতে কেমন হবে

`GET http://localhost:5000/api/v1/user/all-users`

```json
{
	"statusCode": 200,
	"success": true,
	"message": "All users retrieved successfully",
	"meta": {
		"total": 2
	},
	"data": [
		{ "name": "Arif", "email": "arif@example.com" },
		{ "name": "Rahim", "email": "rahim@example.com" }
	]
}
```

এখন প্রতিটা API-র response একই format-এ যাবে, যেটা frontend-এর জন্য অনেক সুবিধাজনক।