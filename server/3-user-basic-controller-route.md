# User Controller আর Routes

এই পর্যন্ত আমাদের user module-এ interface আর model আছে। এবার controller আর route বানাব।

```
src/app/modules/user/
├── user.interface.ts
├── user.model.ts
├── user.controller.ts   ← নতুন
└── user.route.ts        ← নতুন
```

---

## Step 1: `http-status` package install করা

Response-এ `201`, `400` এর মতো সংখ্যা সরাসরি না লিখে নাম দিয়ে লেখার জন্য আমরা [`http-status`](https://www.npmjs.com/package/http-status) package ব্যবহার করব:

```bash
npm i http-status
```

এতে কোড পড়তে সহজ হয়। যেমন `201` না লিখে `httpStatus.CREATED` লিখব।

| নাম | Code | কখন ব্যবহার হয় |
| --- | --- | --- |
| `httpStatus.OK` | `200` | Request সফল হয়েছে |
| `httpStatus.CREATED` | `201` | নতুন কিছু তৈরি হয়েছে |
| `httpStatus.BAD_REQUEST` | `400` | Client ভুল data পাঠিয়েছে |
| `httpStatus.NOT_FOUND` | `404` | কিছু খুঁজে পাওয়া যায়নি |
| `httpStatus.INTERNAL_SERVER_ERROR` | `500` | Server-এ সমস্যা হয়েছে |

---

## Step 2: Controller বানানো

Controller-এর কাজ হলো request নেওয়া, দরকারি কাজ করা, আর response পাঠানো।

`src/app/modules/user/user.controller.ts`

```ts
import type { Request, Response } from "express";
import httpStatus from "http-status";
import { User } from "./user.model";

const createUser = async (req: Request, res: Response) => {
	try {
		const { name, email } = req.body;

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

export const UserControllers = {
	createUser,
};
```

### কিছু ব্যাখ্যা

- **`catch` block:** কোনো সমস্যা হলে (যেমন required field না দিলে বা একই email দুইবার দিলে) error এখানে আসবে, আর আমরা `400` status দিয়ে error message পাঠিয়ে দেব।
- **ESLint note:** `npm run lint` চালালে `any` আর `console.log`-এর জন্য ESLint error/warning দেখাতে পারে। আপাতত এটা স্বাভাবিক, পরে error handling ভালোভাবে শেখার সময় এটা ঠিক করব।
- **`UserControllers` object:** সব controller function একটা object-এর ভিতরে রেখে export করছি। পরে `getAllUsers`, `updateUser` ইত্যাদি যোগ হলে এখানেই যোগ করব।

---

## Step 3: Route বানানো

Route ঠিক করে কোন URL-এ request আসলে কোন controller চলবে।

`src/app/modules/user/user.route.ts`

```ts
import { Router } from "express";
import { UserControllers } from "./user.controller";

const router = Router();

router.post("/register", UserControllers.createUser);

export const UserRoutes = router;
```

### কিছু ব্যাখ্যা

- **Function call না, শুধু reference:** এখানে `UserControllers.createUser` লিখেছি, `UserControllers.createUser()` না। আমরা শুধু function-টা Express-কে দিয়ে দিচ্ছি। Request আসলে Express নিজেই এটাকে call করবে। নিজে `()` দিয়ে call করলে server চালু হওয়ার সময়ই function চলে যাবে, যেটা ভুল।
- **`export const UserRoutes = router`:** এভাবে একটা পরিষ্কার নাম দিয়ে export করছি, যাতে `app.ts`-এ import করার সময় বোঝা যায় এটা user-এর route। এটা মূলত কোড সহজ ও readable রাখার জন্য।

---

## Step 4: `app.ts`-এ route যোগ করা

`src/app.ts`

```ts
import type { Request, Response } from "express";
import express from "express";
import cors from "cors";
import { UserRoutes } from "./app/modules/user/user.route";

const app = express();

// Middlewares
app.use(express.json());
app.use(cors());

// Routes
app.use("/api/v1/user", UserRoutes);

app.get("/", (req: Request, res: Response) => {
	res.status(200).json({
		message: "Welcome to Tour Management System Backend",
	});
});

export default app;
```

### Middleware গুলোর কাজ

- **`express.json()`**: আমরা request body-তে JSON পাঠাব। এই middleware সেই JSON পড়ে `req.body`-তে রেখে দেয়। এটা না দিলে `req.body` হবে `undefined`। খেয়াল রাখতে হবে, এটা function, তাই `()` দিয়ে call করতে হবে।
- **`cors()`**: Frontend সাধারণত অন্য port বা domain থেকে চলে (যেমন `localhost:5173`)। Browser default হিসেবে অন্য domain থেকে API call আটকে দেয়। `cors()` এই অনুমতি দেয়।

> **Order গুরুত্বপূর্ণ:** Middleware গুলো (`express.json()`, `cors()`) অবশ্যই routes-এর **আগে** লিখতে হবে। নাহলে route চলার সময় `req.body` পাওয়া যাবে না।

### Final URL

`app.ts`-এর `/api/v1/user` আর `user.route.ts`-এর `/register` মিলে পুরো URL হবে:

```
POST http://localhost:5000/api/v1/user/register
```

---

## Step 5: Test করা

Postman বা Thunder Client দিয়ে এই request পাঠাব:

- **Method:** `POST`
- **URL:** `http://localhost:5000/api/v1/user/register`
- **Body** (raw → JSON):

```json
{
	"name": "Arif",
	"email": "arif@example.com"
}
```

সফল হলে `201` status আর নতুন user-এর data পাব।