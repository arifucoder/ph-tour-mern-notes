# Routing System গুছিয়ে রাখা

## সমস্যা কী?

এখন `app.ts`-এ আমরা route এভাবে যোগ করছি:

```ts
app.use("/api/v1/user", UserRoutes);
```

কিন্তু পরে আরও অনেক module আসবে, যেমন tour, guide, booking ইত্যাদি। সবগুলো `app.ts`-এ লিখতে থাকলে file-টা বড় আর অগোছালো হয়ে যাবে, আর প্রতিবার `/api/v1` বারবার লিখতে হবে:

```ts
app.use("/api/v1/user", UserRoutes);
app.use("/api/v1/tour", TourRoutes);
app.use("/api/v1/guide", GuideRoutes);
app.use("/api/v1/booking", BookingRoutes);
// ...
```

তাই সব route এক জায়গায় গুছিয়ে রাখব।

---

## Step 1: `routes` folder বানানো

`src/app`-এর ভিতরে `routes` নামে একটা folder নেব, আর তার ভিতরে `index.ts` file:

```
src/app/
├── config/
├── modules/
└── routes/
    └── index.ts   ← নতুন
```

---

## Step 2: `index.ts`-এ সব route রাখা

`src/app/routes/index.ts`

```ts
import { Router } from "express";
import { UserRoutes } from "../modules/user/user.route";

export const router = Router();

const moduleRoutes = [
	{
		path: "/user",
		route: UserRoutes,
	},
];

moduleRoutes.forEach((route) => {
	router.use(route.path, route.route);
});
```

### এটা কীভাবে কাজ করে?

`moduleRoutes` array-তে প্রতিটা module-এর `path` আর `route` রাখছি। তারপর `forEach` দিয়ে প্রতিটার জন্য `router.use()` চালাচ্ছি।

মানে এই পুরো array আর `forEach` আসলে এটাই করছে:

```ts
router.use("/user", UserRoutes);
```

### নতুন module যোগ করা

পরে নতুন module আসলে শুধু array-তে একটা object যোগ করলেই হবে, আর কিছু বদলাতে হবে না:

```ts
const moduleRoutes = [
	{
		path: "/user",
		route: UserRoutes,
	},
	{
		path: "/tour",
		route: TourRoutes,
	},
];
```

---

## Step 3: `app.ts`-এ যোগ করা

`app.ts` থেকে আগের `app.use("/api/v1/user", UserRoutes)` লাইন আর `UserRoutes`-এর import সরিয়ে দেব। তার বদলে শুধু এটা লিখব:

`src/app.ts`

```ts
import type { Request, Response } from "express";
import express from "express";
import cors from "cors";
import { router } from "./app/routes";

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

export default app;
```

> **Note:** `import { router } from "./app/routes"` লিখলে আলাদা করে `/index` লিখতে হয় না। Folder-এর নাম দিলেই সেই folder-এর `index.ts` file নিজে থেকেই import হয়ে যায়।

---

## Final URL

URL আগের মতোই থাকবে, শুধু তিনটা অংশ তিন জায়গা থেকে আসছে:

| অংশ | কোথা থেকে আসছে |
| --- | --- |
| `/api/v1` | `app.ts` |
| `/user` | `routes/index.ts` |
| `/register` | `user.route.ts` |

```
POST http://localhost:5000/api/v1/user/register
```