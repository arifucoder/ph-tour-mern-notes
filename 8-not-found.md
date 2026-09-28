# Not Found Route

## কেন দরকার?

কেউ যদি এমন কোনো URL-এ request পাঠায় যেটা আমাদের app-এ নেই (যেমন `/api/v1/abc`), তাহলে Express নিজে থেকে একটা HTML page পাঠায়। কিন্তু আমাদের API সবসময় JSON-এ response দেয়, তাই এখানেও সুন্দর একটা JSON response পাঠাব। এজন্য একটা **Not Found** middleware বানাব।

---

## Step 1: `http-status-codes` install করা

Status code-এর জন্য এখানে [`http-status-codes`](https://www.npmjs.com/package/http-status-codes) package ব্যবহার করব:

```bash
npm i http-status-codes
```

---

## Step 2: `notFound.ts` file বানানো

এটাও একটা middleware, তাই `middlewares` folder-এ রাখব।

```
src/app/middlewares/
├── globalErrorHandler.ts
└── notFound.ts   ← নতুন
```

`src/app/middlewares/notFound.ts`

```ts
import type { Request, Response } from "express";
import httpStatus from "http-status-codes";

const notFound = (req: Request, res: Response) => {
	res.status(httpStatus.NOT_FOUND).json({
		success: false,
		message: "Oops! The route you are looking for does not exist.",
	});
};

export default notFound;
```

---

## Step 3: `app.ts`-এ যোগ করা

`src/app.ts`

```ts
import type { Request, Response } from "express";
import express from "express";
import cors from "cors";
import { router } from "./app/routes";
import { globalErrorHandler } from "./app/middlewares/globalErrorHandler";
import notFound from "./app/middlewares/notFound";

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

// Global error handler
app.use(globalErrorHandler);

// Not found route (সবার শেষে)
app.use(notFound);

export default app;
```

---

## এটা কীভাবে কাজ করে?

Express উপর থেকে নিচে একটা একটা করে route মিলিয়ে দেখে। কোনো route না মিললে request একদম নিচে `notFound`-এ চলে আসে, আর সেখান থেকে `404` response যায়।

```
Request → Routes (মিলল না) → notFound → 404 Response
```

> **`globalErrorHandler`-এর পরে রাখলে সমস্যা হবে না?** না। Express সাধারণ request-এর সময় error handler (৪ parameter-এর middleware) এড়িয়ে যায়, শুধু `next(err)` call হলে সেখানে যায়। তাই কোনো error না থাকলে request সরাসরি `notFound`-এ পৌঁছে যায়।

### Response

`GET http://localhost:5000/api/v1/abc` এ request পাঠালে status `404` সহ এই response আসবে:

```json
{
	"success": false,
	"message": "Oops! The route you are looking for does not exist."
}
```