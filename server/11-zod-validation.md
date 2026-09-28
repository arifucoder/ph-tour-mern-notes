# Zod Validation

## Zod কেন?

Client `req.body`-তে যা খুশি পাঠাতে পারে। যেমন name-এর জায়গায় number, ভুল format-এর email, বা খুব দুর্বল password। এগুলো database-এ যাওয়ার আগেই আটকাতে হবে।

Zod দিয়ে আমরা `req.body`-র:

- **Validation** করব: data ঠিক আছে কিনা check করা।
- **Sanitization** করব: অপ্রয়োজনীয় data বাদ দিয়ে শুধু দরকারি data রাখা।

### কখন ব্যবহার হবে?

যখন client `req.body`-তে data পাঠায়, মূলত দুই সময়ে:

1. নতুন data **create** করার সময়
2. কোনো data **update** করার সময়

> `zod` আমরা project setup-এর সময়েই install করেছি। না করে থাকলে: `npm install zod`

---

## Step 1: Validation file বানানো

Validation চাইলে সরাসরি route-এর ভিতরেও লেখা যায়, কিন্তু সেটা অগোছালো হয়ে যায়। তাই আমরা ভালো নিয়ম মেনে user module-এ আলাদা একটা file বানাব।

```
src/app/modules/user/
├── user.interface.ts
├── user.model.ts
├── user.validation.ts   ← নতুন
├── user.service.ts
├── user.controller.ts
└── user.route.ts
```

### Create User Schema

`src/app/modules/user/user.validation.ts`

```ts
import { z } from "zod";
import { IsActive, Role } from "./user.interface";

export const createUserZodSchema = z.object({
	name: z
		.string({ error: "Name must be string" })
		.min(2, { error: "Name must be at least 2 characters long." })
		.max(50, { error: "Name cannot exceed 50 characters." }),
	email: z
		.email({ error: "Invalid email address format." })
		.min(5, { error: "Email must be at least 5 characters long." })
		.max(100, { error: "Email cannot exceed 100 characters." }),
	password: z
		.string({ error: "Password must be string" })
		.min(8, { error: "Password must be at least 8 characters long." })
		.regex(/^(?=.*[A-Z])/, {
			error: "Password must contain at least 1 uppercase letter.",
		})
		.regex(/^(?=.*[!@#$%^&*])/, {
			error: "Password must contain at least 1 special character.",
		})
		.regex(/^(?=.*\d)/, {
			error: "Password must contain at least 1 number.",
		}),
	phone: z
		.string({ error: "Phone number must be string" })
		.regex(/^(?:\+8801\d{9}|01\d{9})$/, {
			error: "Phone number must be valid for Bangladesh. Format: +8801XXXXXXXXX or 01XXXXXXXXX",
		})
		.optional(),
	address: z
		.string({ error: "Address must be string" })
		.max(200, { error: "Address cannot exceed 200 characters." })
		.optional(),
});
```

---

## Zod v4-এ error message লেখা

Zod v4-এ `invalid_type_error` আর `required_error` দুটোই বাদ দেওয়া হয়েছে। এখন সবকিছুর জন্য একটাই parameter: **`error`**।

দুইভাবে লেখা যায়:

```ts
// ১. object-এর ভিতরে error দিয়ে
name: z
	.string({ error: "Name must be string" })
	.min(2, { error: "Name must be at least 2 characters long." })
	.max(50, { error: "Name cannot exceed 50 characters." }),

// ২. সরাসরি string দিয়ে (ছোট করে)
name: z
	.string("Name must be string")
	.min(2, "Name must be at least 2 characters long.")
	.max(50, "Name cannot exceed 50 characters."),
```

> **Note:** পুরোনো `{ message: "..." }` এখনও কাজ করে, কিন্তু v4-এ এটা deprecated। নতুন কোডে `error` ব্যবহার করাই ভালো।

> **Email:** Zod v4-এ `z.string().email()` deprecated হয়ে গেছে। এখন সরাসরি `z.email()` লিখতে হয়।

### Regex গুলোর মানে

| Regex | কী check করে |
| --- | --- |
| `/^(?=.*[A-Z])/` | কমপক্ষে ১টা বড় হাতের অক্ষর আছে কিনা |
| `/^(?=.*[!@#$%^&*])/` | কমপক্ষে ১টা special character আছে কিনা |
| `/^(?=.*\d)/` | কমপক্ষে ১টা সংখ্যা আছে কিনা |
| `/^(?:\+8801\d{9}\|01\d{9})$/` | বাংলাদেশি phone number কিনা (`+8801XXXXXXXXX` বা `01XXXXXXXXX`) |

---

## Step 2: `validateRequest` middleware বানানো

Zod schema দিয়ে route validate করার জন্য একটা **higher order function** বানাব। এটা একটা zod schema নেয়, আর একটা middleware return করে।

`src/app/middlewares/validateRequest.ts`

```ts
import type { NextFunction, Request, Response } from "express";
import type { ZodObject } from "zod";

export const validateRequest =
	(zodSchema: ZodObject) => async (req: Request, res: Response, next: NextFunction) => {
		try {
			req.body = await zodSchema.parseAsync(req.body);
			next();
		} catch (error) {
			next(error);
		}
	};
```

### কোডটা বোঝা

- **`ZodObject`**: আমাদের সব schema `z.object({...})` দিয়ে বানানো, তাই সবগুলোই `ZodObject`। শুধু ভিতরের field গুলো একেকটায় একেক রকম। তাই type হিসেবে `ZodObject` দিয়েছি, যাতে যেকোনো schema এখানে দেওয়া যায়।
- **`parseAsync(req.body)`**: `req.body`-কে schema-র সাথে মিলিয়ে দেখে।
  - সব ঠিক থাকলে পরিষ্কার data return করে, আর আমরা সেটা আবার `req.body`-তে রেখে দিই। তারপর `next()` দিয়ে controller-এ পাঠাই।
  - কোনো ভুল থাকলে error `throw` করে, আর `next(error)` দিয়ে সেটা `globalErrorHandler`-এ যায়।
- **Sanitization কীভাবে হয়?** Schema-তে নেই এমন কোনো field (যেমন create-এর সময় `"role": "SUPER_ADMIN"`) পাঠালে Zod সেটা নিজে থেকেই বাদ দিয়ে দেয়। তাই controller শুধু পরিষ্কার data পায়।

---

## Step 3: Route-এ ব্যবহার করা

`user.route.ts`

```ts
import { Router } from "express";
import { validateRequest } from "../../middlewares/validateRequest";
import { UserControllers } from "./user.controller";
import { createUserZodSchema } from "./user.validation";

const router = Router();

router.post("/register", validateRequest(createUserZodSchema), UserControllers.createUser);
router.get("/all-users", UserControllers.getAllUsers);

export const UserRoutes = router;
```

এখন request আসলে আগে `validateRequest` চলবে। Data ঠিক থাকলে তবেই controller-এ যাবে।

```
Request → validateRequest (data check) → Controller → Service → Database
                ↓ (ভুল হলে)
         globalErrorHandler
```

> **Note:** এখন Zod-এর error `globalErrorHandler`-এ `500` status নিয়ে যাবে, কারণ এটা `AppError` না। Validation error-এর status `400` হওয়া উচিত। Mongoose error-এর মতো এটাও পরে আলাদা করে handle করব।

---

## Update User Schema

Update-এর সময় user সব field পাঠাবে না, যে যেটা বদলাতে চায় শুধু সেটাই পাঠাবে। তাই update schema-তে **সব field optional**। যেমন create-এর সময় `name` required ছিল, কিন্তু update-এ কেউ হয়তো name বদলাতেই চায় না, তাই এখানে optional।

`user.validation.ts` (একই file-এ)

```ts
export const updateUserZodSchema = z.object({
	name: z
		.string({ error: "Name must be string" })
		.min(2, { error: "Name must be at least 2 characters long." })
		.max(50, { error: "Name cannot exceed 50 characters." })
		.optional(),
	password: z
		.string({ error: "Password must be string" })
		.min(8, { error: "Password must be at least 8 characters long." })
		.regex(/^(?=.*[A-Z])/, {
			error: "Password must contain at least 1 uppercase letter.",
		})
		.regex(/^(?=.*[!@#$%^&*])/, {
			error: "Password must contain at least 1 special character.",
		})
		.regex(/^(?=.*\d)/, {
			error: "Password must contain at least 1 number.",
		})
		.optional(),
	phone: z
		.string({ error: "Phone number must be string" })
		.regex(/^(?:\+8801\d{9}|01\d{9})$/, {
			error: "Phone number must be valid for Bangladesh. Format: +8801XXXXXXXXX or 01XXXXXXXXX",
		})
		.optional(),
	role: z.enum(Role).optional(),
	isActive: z.enum(IsActive).optional(),
	isDeleted: z.boolean({ error: "isDeleted must be true or false" }).optional(),
	isVerified: z.boolean({ error: "isVerified must be true or false" }).optional(),
	address: z
		.string({ error: "Address must be string" })
		.max(200, { error: "Address cannot exceed 200 characters." })
		.optional(),
});
```

### কিছু ব্যাখ্যা

- **`email` নেই কেন?** আমরা user-কে email বদলাতে দেব না, তাই update schema-তে email রাখিনি। কেউ পাঠালেও Zod সেটা বাদ দিয়ে দেবে।
- **`role`, `isActive`, `isDeleted`, `isVerified` কেন আছে?** Admin আর সাধারণ user, দুজনের profile update একই API দিয়ে handle করব। Admin এগুলো বদলাতে পারবে, তাই schema-তে রাখা হয়েছে।

> **খেয়াল রাখো:** Zod শুধু data-র **format** check করে, কে পাঠাচ্ছে সেটা দেখে না। সাধারণ user যেন নিজের `role` বদলে admin হয়ে যেতে না পারে, সেটা আটকানো হবে পরে **authorization**-এর মাধ্যমে।

### Enum লেখার তিনটা উপায়

```ts
// ১. হাতে লিখে
role: z.enum(["SUPER_ADMIN", "ADMIN", "USER", "GUIDE"]).optional(),

// ২. Object.values দিয়ে
role: z.enum(Object.values(Role) as [string]).optional(),

// ৩. সরাসরি TypeScript enum দিয়ে (Zod v4, সবচেয়ে সহজ)
role: z.enum(Role).optional(),
```

হাতে লিখলে পরে `Role` enum-এ নতুন role যোগ করলে এখানেও আলাদা করে যোগ করতে হবে, ভুলে যাওয়ার সম্ভাবনা থাকে। তাই `Role` enum থেকেই value নেওয়া ভালো। Zod v4-এ সরাসরি `z.enum(Role)` লেখা যায়, যেটা সবচেয়ে সহজ আর type-ও ঠিকঠাক থাকে। `as [string]` দিলে TypeScript আসল role গুলো চিনতে পারে না, শুধু `string` ধরে নেয়।