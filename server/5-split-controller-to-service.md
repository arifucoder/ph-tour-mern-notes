# Controller ভাগ করা আর Service Layer বানানো

## সমস্যা কী?

এখন আমাদের controller-এ database-এর সাথে যোগাযোগের কোডও লেখা হচ্ছে:

`user.controller.ts`

```ts
const createUser = async (req: Request, res: Response) => {
	try {
		const { name, email } = req.body;

		// এই অংশটুকু database-এর সাথে যোগাযোগ
		const user = await User.create({
			name,
			email,
		});

		res.status(httpStatus.CREATED).json({
			message: "User created successfully",
			user,
		});
	} catch (err: any) {
		console.log(err);

		res.status(httpStatus.BAD_REQUEST).json({
			message: `Something went wrong! ${err.message}`,
			err,
		});
	}
};
```

পরে যখন password hash করা, password মিলছে কিনা check করা ইত্যাদি কাজ আসবে, তখন controller অনেক বড় আর জটিল হয়ে যাবে।

তাই database-এর সাথে যোগাযোগ আর সব **business logic** আমরা আলাদা একটা layer-এ রাখব, যার নাম **Service Layer**।

---

## কার কী কাজ?

| Layer | কাজ |
| --- | --- |
| **Controller** | শুধু `req` আর `res` handle করবে। Request থেকে data নিয়ে service-কে দেবে, আর service যা return করবে সেটা response হিসেবে পাঠাবে। |
| **Service** | Business logic আর database-এর সাথে যোগাযোগ। যেমন password hash করা, password match করা, user তৈরি করা ইত্যাদি। |

---

## Step 1: Service file বানানো

`src/app/modules/user` folder-এ `user.service.ts` নামে একটা file নেব।

`src/app/modules/user/user.service.ts`

```ts
import type { IUser } from "./user.interface";
import { User } from "./user.model";

const createUser = async (payload: Partial<IUser>) => {
	const { name, email } = payload;

	const user = await User.create({
		name,
		email,
	});

	return user;
};

export const UserServices = {
	createUser,
};
```

### `Partial<IUser>` কেন?

`IUser`-এ অনেক field আছে, আর কিছু field required (যেমন `role`, `auths`)। কিন্তু user তৈরি করার সময় আমরা সব data পাঠাচ্ছি না, শুধু `name` আর `email` পাঠাচ্ছি।

`Partial<IUser>` দিলে `IUser`-এর সব field optional হয়ে যায়, তাই কিছু field পাঠালেও TypeScript error দেবে না।

### Service-এ destructure কেন করছি?

`payload` থেকে শুধু `name` আর `email` বের করে নিচ্ছি। এটা নিরাপত্তার জন্য জরুরি। কেউ চাইলে request body-তে `"role": "SUPER_ADMIN"` পাঠিয়ে দিতে পারে। আমরা যদি পুরো `payload` সরাসরি `User.create()`-এ দিয়ে দিতাম, তাহলে সে নিজেকে admin বানিয়ে ফেলতে পারত।

---

## Step 2: Controller আপডেট করা

এবার controller-এ আর destructure করব না, আর `User` model-ও সরাসরি ব্যবহার করব না। শুধু `req.body` service-কে দিয়ে দেব।

`src/app/modules/user/user.controller.ts`

```ts
import type { Request, Response } from "express";
import httpStatus from "http-status";
import { UserServices } from "./user.service";

const createUser = async (req: Request, res: Response) => {
	try {
		const user = await UserServices.createUser(req.body);

		res.status(httpStatus.CREATED).json({
			message: "User created successfully",
			user,
		});
	} catch (err: any) {
		console.log(err);

		res.status(httpStatus.BAD_REQUEST).json({
			message: `Something went wrong! ${err.message}`,
			err,
		});
	}
};

export const UserControllers = {
	createUser,
};
```

এখন controller অনেক ছোট আর পরিষ্কার। এখন থেকে password hashing বা API-র যত business logic আছে, সব service-এ লিখব।

---

## Request-এর Flow

একটা request আসলে এই পথে যায়:

```
Route → Controller → Service → Model → Database
```

তারপর data একই পথে উল্টো দিকে ফিরে আসে, আর controller সেটা response হিসেবে পাঠায়।

## কাজ করার Order

তাই নতুন module বানানোর সময় আমরা নিচ থেকে উপরের দিকে কাজ করব:

```
Interface → Model → Service → Controller → Route
```

```
src/app/modules/user/
├── user.interface.ts
├── user.model.ts
├── user.service.ts      ← নতুন
├── user.controller.ts
└── user.route.ts
```