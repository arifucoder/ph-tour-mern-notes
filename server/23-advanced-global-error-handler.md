# 25 — Mongoose আর Zod Error Handling

আগের note-গুলোতে ([06-global-error-handler](./06-global-error-handler.md), [07-extend-js-error-to-apperror](./07-extend-js-error-to-apperror.md)) global error handler আর `AppError` বানিয়েছিলাম। সেখানে মাত্র দুই ধরনের error handle হতো:

| Error | Status | Message |
|---|---|---|
| `AppError` (আমাদের custom) | আমরা যেটা দিই | আমরা যেটা দিই |
| সাধারণ JS `Error` | `500` | `err.message` |

এগুলো আমাদের **app-এর নিজের** error। কিন্তু আমরা অনেক package ব্যবহার করি, আর তাদেরও নিজস্ব error আছে:

- **Mongoose / MongoDB:** DB-র সাথে কাজ করতে গিয়ে error (duplicate, ভুল id, ভুল type)
- **Zod:** request body-র rule না মানলে error

এগুলো আলাদা করে handle না করলে সব `500` হিসেবে যায়, আর message হয় অগোছালো।

---

## সমস্যাটা কী?

ধরো registration-এর সময় `email` field দিইনি। এখনকার response:

```json
{
  "success": false,
  "message": "[\n  {\n    \"expected\": \"string\",\n    \"code\": \"invalid_type\",\n    \"path\": [\n      \"email\"\n    ],\n    \"message\": \"Email must be string\"\n  }\n]",
  "err": { "name": "ZodError", "message": "..." },
  "stack": "ZodError: ... (অনেক লম্বা)"
}
```

Status `500`, আর message একটা JSON string, যেটা পড়াই কঠিন। Frontend developer বুঝতে পারবে না কোন field-এ কী সমস্যা।

**Backend developer-এর কাজ:** frontend-কে এমনভাবে error পাঠানো যাতে সে সহজেই বোঝে কী সমস্যা হয়েছে। নিজেরাও debug করার সময় যেন এক নজরে বুঝতে পারি।

আমরা চাই response দেখতে এরকম হোক:

```json
{
  "success": false,
  "message": "Validation failed! Please check the highlighted fields.",
  "errorSources": [
    { "path": "email", "message": "Email must be string" }
  ],
  "err": null,
  "stack": null
}
```

---

## পুরো Flow

```
Service / Middleware-এ error
        │
        ▼
next(err) → globalErrorHandler
        │
        ├─ err.code === 11000         → handlerDuplicateError   (400)
        ├─ err.name === "CastError"   → handleCastError         (400)
        ├─ err.name === "ZodError"    → handlerZodError         (400)
        ├─ err.name === "ValidationError" → handlerValidationError (400)
        ├─ err instanceof AppError    → err.statusCode, err.message
        └─ err instanceof Error       → 500, err.message
        │
        ▼
সব error একই format-এ response:
{ success, message, errorSources, err, stack }
```

প্রতিটা error-এর জন্য আলাদা **handler function** বানাব `src/app/errorHelpers/` folder-এ। প্রতিটা handler একই shape-এর object return করবে (`TGenericErrorResponse`), তাই globalErrorHandler-এ সব error একভাবে ব্যবহার করা যাবে।

```
src/app/
├── errorHelpers/
│   ├── AppError.ts
│   ├── handleCastError.ts        ← নতুন
│   ├── handleDuplicateError.ts   ← নতুন
│   ├── handlerValidationError.ts ← নতুন
│   └── handlerZodError.ts        ← নতুন
├── interfaces/
│   └── error.types.ts            ← নতুন
└── middlewares/
    └── globalErrorHandler.ts     ← update
```

---

## Step 1: Error-এর Type বানানো

সব handler একই format-এ data return করবে।

- **`TErrorSources`**: কোন field-এ (`path`) কী সমস্যা (`message`)। Frontend এটা দিয়ে নির্দিষ্ট input-এর নিচে error দেখাতে পারে।
- **`TGenericErrorResponse`**: প্রতিটা handler যা return করবে। `errorSources` optional, কারণ সব error-এর field থাকে না।

---

## Step 2: Duplicate Error (MongoDB code `11000`)

**কখন আসে:** `unique: true` field-এ একই value দুইবার save করলে। যেমন একই email দিয়ে দুইবার register।

> ✏️ **সংশোধন:** `11000` আসলে **MongoDB server**-এর error code, Mongoose-এর না। Mongoose শুধু সেটা পাস করে দেয়। এই code শুধু duplicate key বোঝায়, অন্য কিছু না, তাই `err.code === 11000` দিয়ে নিশ্চিন্তে চেনা যায়।

আমাদের `createUser` service-এ আগেই check করি:

```ts
if (isUserExist) {
	throw new AppError(httpStatus.BAD_REQUEST, "User already exist");
}
```

এই check সরিয়ে দিলে MongoDB নিজেই `11000` error দেবে। তবুও এই handler রাখা দরকার, কারণ অন্য module-এ (যেমন tour-এর `slug`) সবসময় আগে check করা থাকবে না।

MongoDB-র error দেখতে এরকম:

```
E11000 duplicate key error collection: tour.users index: email_1 dup key: { email: "arif@gmail.com" }
```

**Teacher-এর approach:** regex `/"([^"]*)"/` দিয়ে quote-এর ভেতরের অংশ (`arif@gmail.com`) বের করা।

> ⚠️ **সমস্যা:** কোনো কারণে message-এ quote না থাকলে `match()` `null` return করে, তখন `matchedArray[1]` পড়তে গিয়ে **error handler নিজেই crash করবে**। তাছাড়া message-এ শুধু value আসে, কোন field সেটা বোঝা যায় না।
>
> **সমাধান:** MongoDB-র duplicate error-এ একটা `keyValue` object থাকে, যেমন `{ email: "arif@gmail.com" }`। এখান থেকে field আর value দুটোই নিরাপদে পাওয়া যায়, regex লাগে না।

---

## Step 3: Cast Error (ভুল ObjectId)

**কখন আসে:** Mongoose যখন কোনো value-কে দরকারি type-এ রূপান্তর (cast) করতে পারে না। সবচেয়ে common উদাহরণ: user update করার সময় URL-এ ভুল id পাঠানো।

```
PATCH /api/v1/user/123abc   ← valid ObjectId না
```

`findById("123abc")` তখন `CastError` দেয়। আমরা `err.name === "CastError"` দিয়ে ধরে সুন্দর message পাঠাই।

> 💡 **Improve:** Teacher-এর message fixed ছিল। কিন্তু `CastError`-এর ভেতরে `path` (কোন field), `value` (কী পাঠানো হয়েছে), `kind` (কোন type দরকার ছিল) থাকে। এগুলো দিয়ে message আরও পরিষ্কার করা যায়: `Invalid _id: '123abc'. Please provide a valid MongoDB ObjectId.`

---

## Step 4: Mongoose Validation Error

**কখন আসে:** `create()` / `save()` করার সময় (আর update-এ `runValidators: true` দিলে) schema-র rule না মানলে। যেমন `required` field নেই, `enum`-এর বাইরের value, বা type মেলে না।

### Mongoose type casting

Mongoose ভুল type পেলে সাথে সাথে error দেয় না, আগে **cast** করার চেষ্টা করে:

| Schema type | পাঠালাম | ফলাফল |
|---|---|---|
| `String` | `9` | `"9"` ✅ |
| `Number` (age) | `"9"` | `9` ✅ |
| `Number` (age) | `"hello"` | ❌ ValidationError |
| `Boolean` | `"true"` | `true` ✅ |
| `Boolean` | `"hello"` | ❌ ValidationError |

> ✏️ **সংশোধন:** "Boolean-এ string পাঠালে validation error" — সবসময় না। Mongoose `"true"`, `"false"`, `"1"`, `"0"`, `"yes"`, `"no"`-কে boolean বানিয়ে ফেলে। শুধু cast করা যায় না এমন value (যেমন `"hello"`) দিলে error দেয়।

### Error-এর ভেতরে Error

`ValidationError`-এর ভেতরে একটা `errors` object থাকে, যেখানে **প্রতিটা ভুল field-এর জন্য আলাদা error**:

```js
{
  name: "ValidationError",
  errors: {
    age:  { name: "CastError", path: "age", kind: "Number", message: "Cast to Number failed ..." },
    role: { name: "ValidatorError", path: "role", message: "`HELLO` is not a valid enum value ..." }
  }
}
```

তাই ভেতরে `CastError` দেখা যায়, কিন্তু বাইরের (main) error হলো `ValidationError`। আমরা `Object.values(err.errors)` দিয়ে সব field-এর error-কে একটা array (`errorSources`) বানাই।

> 💡 **Improve:** ভেতরের `CastError`-এর message (`Cast to Number failed for value "hello" (type string) at path "age"`) frontend-এর জন্য অগোছালো। তাই সেটাকে `age must be a valid Number` বানিয়ে দিয়েছি।

### ❓ Zod তো আগেই validate করে, তাহলে এটা লাগে কেন?

ঠিক, যে route-এ `validateRequest(zodSchema)` আছে সেখানে বেশিরভাগ ভুল Zod-ই ধরে ফেলবে (Zod casting করে না, number-এ শুধু number নেয়)। কিন্তু:
- সব route-এ Zod থাকবে না (যেমন seed, Google OAuth দিয়ে user create)
- Service-এর ভেতরে আমরা নিজেরাও data বানিয়ে save করি

এসব জায়গায় শেষ পাহারাদার হলো Mongoose schema, তাই এই handler-ও দরকার।

---

## Step 5: Zod Error

**কখন আসে:** `validateRequest` middleware-এ `schema.parseAsync(req.body)` fail করলে।

`ZodError`-এর গঠন:

```js
{
  name: "ZodError",
  issues: [
    { code: "invalid_type", expected: "string", path: ["email"], message: "Email must be string" },
    { code: "too_small",   path: ["name", "lastName"], message: "..." }
  ]
}
```

> ✏️ **সংশোধন:** `issues` কোনো object না, এটা সরাসরি **object-এর array**। প্রতিটা object একটা ভুল (issue)।

প্রতিটা issue থেকে `path` আর `message` নিয়ে `errorSources` বানাই।

### `path` কীভাবে নেব?

`path` একটা array, nested field হলে একাধিক item থাকে: `["name", "lastName"]`।

| Approach | Result | সমস্যা |
|---|---|---|
| `issue.path[issue.path.length - 1]` (teacher) | `"lastName"` | কোন object-এর lastName বোঝা যায় না; type-এ `string \| number \| symbol \| undefined` আসে, `string` না |
| `issue.path.reverse().join(" inside ")` (commented) | `"lastName inside name"` | `reverse()` মূল array-কেই উল্টে দেয় (mutate) |
| `issue.path.map(String).join(".")` ✅ | `"name.lastName"` | কোনো সমস্যা নেই, frontend form library-ও এই format বোঝে |

---

## Step 6: Global Error Handler update

সব handler-কে globalErrorHandler-এ যোগ করি। প্রতিটা `if` block একই কাজ করে: handler call → `statusCode`, `message`, `errorSources` নেওয়া।

### কিছু point

- **Order:** নির্দিষ্ট error আগে, সাধারণ `Error` সবার শেষে। কারণ `ZodError`, `CastError` সবাই আসলে `Error`-এরই child। আগে `instanceof Error` check করলে সবাই সেখানেই ধরা পড়বে।
- **`err.name` / `err.code` দিয়ে কেন চিনি?** প্রতিটা package-এর error-এ নিজস্ব `name` থাকে, এটাই সবচেয়ে সহজ উপায়।
- **`next` ব্যবহার না করলেও রাখতে হবে:** Express ৪টা parameter দেখেই error middleware চেনে (06 নম্বর note)। তাই `no-unused-vars` বন্ধ রাখা।
- **`err` আর `stack` শুধু development-এ:** Production-এ `null`, যাতে কোডের ভেতরের তথ্য বাইরে না যায়।
- **`console.log(err)`:** শুধু development-এ terminal-এ পুরো error দেখার জন্য। `no-console` warning এড়াতে eslint comment দিয়েছি।

---

## Final Code

### `src/app/interfaces/error.types.ts`

```ts
export interface TErrorSources {
	path: string;
	message: string;
}

export interface TGenericErrorResponse {
	statusCode: number;
	message: string;
	errorSources?: TErrorSources[];
}
```

### `src/app/errorHelpers/AppError.ts` (আগের মতোই)

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

### `src/app/errorHelpers/handleDuplicateError.ts`

```ts
import httpStatus from "http-status-codes";
import type { TGenericErrorResponse } from "../interfaces/error.types";

interface IMongoDuplicateError {
	code: number;
	message: string;
	keyValue?: Record<string, unknown>;
}

export const handlerDuplicateError = (err: IMongoDuplicateError): TGenericErrorResponse => {
	// keyValue: { email: "arif@gmail.com" }
	const duplicateEntry = Object.entries(err.keyValue ?? {})[0];

	if (!duplicateEntry) {
		return {
			statusCode: httpStatus.BAD_REQUEST,
			message: "Duplicate value! This data already exists.",
		};
	}

	const [field, value] = duplicateEntry;

	return {
		statusCode: httpStatus.BAD_REQUEST,
		message: `${field} '${String(value)}' already exists! Please use a different ${field}.`,
		errorSources: [{ path: field, message: `This ${field} is already taken` }],
	};
};
```

### `src/app/errorHelpers/handleCastError.ts`

```ts
import httpStatus from "http-status-codes";
import type mongoose from "mongoose";
import type { TGenericErrorResponse } from "../interfaces/error.types";

export const handleCastError = (err: mongoose.Error.CastError): TGenericErrorResponse => {
	const expectedType = err.kind === "ObjectId" ? "MongoDB ObjectId" : err.kind;

	return {
		statusCode: httpStatus.BAD_REQUEST,
		message: `Invalid ${err.path}: '${String(err.value)}'. Please provide a valid ${expectedType}.`,
		errorSources: [{ path: err.path, message: `Expected a valid ${expectedType}` }],
	};
};
```

### `src/app/errorHelpers/handlerValidationError.ts`

```ts
import httpStatus from "http-status-codes";
import mongoose from "mongoose";
import type { TErrorSources, TGenericErrorResponse } from "../interfaces/error.types";

export const handlerValidationError = (err: mongoose.Error.ValidationError): TGenericErrorResponse => {
	const errorSources: TErrorSources[] = Object.values(err.errors).map((errorObject) => ({
		path: errorObject.path,
		// ভেতরের CastError-এর লম্বা message-কে ছোট ও পরিষ্কার করা
		message:
			errorObject instanceof mongoose.Error.CastError
				? `${errorObject.path} must be a valid ${errorObject.kind}`
				: errorObject.message,
	}));

	return {
		statusCode: httpStatus.BAD_REQUEST,
		message: "Database validation failed! Please check the highlighted fields.",
		errorSources,
	};
};
```

### `src/app/errorHelpers/handlerZodError.ts`

```ts
import httpStatus from "http-status-codes";
import type { ZodError } from "zod";
import type { TErrorSources, TGenericErrorResponse } from "../interfaces/error.types";

export const handlerZodError = (err: ZodError): TGenericErrorResponse => {
	const errorSources: TErrorSources[] = err.issues.map((issue) => ({
		// ["name", "lastName"] → "name.lastName"
		path: issue.path.map(String).join("."),
		message: issue.message,
	}));

	return {
		statusCode: httpStatus.BAD_REQUEST,
		message: "Validation failed! Please check the highlighted fields.",
		errorSources,
	};
};
```

### `src/app/middlewares/globalErrorHandler.ts`

```ts
/* eslint-disable @typescript-eslint/no-unused-vars */
/* eslint-disable @typescript-eslint/no-explicit-any */
import type { NextFunction, Request, Response } from "express";
import { envVars } from "../config/env";
import AppError from "../errorHelpers/AppError";
import { handleCastError } from "../errorHelpers/handleCastError";
import { handlerDuplicateError } from "../errorHelpers/handleDuplicateError";
import { handlerValidationError } from "../errorHelpers/handlerValidationError";
import { handlerZodError } from "../errorHelpers/handlerZodError";
import type { TErrorSources } from "../interfaces/error.types";

export const globalErrorHandler = (err: any, req: Request, res: Response, next: NextFunction) => {
	if (envVars.NODE_ENV === "development") {
		// eslint-disable-next-line no-console
		console.log(err);
	}

	let errorSources: TErrorSources[] = [];
	let statusCode = 500;
	let message = "Something went wrong!";

	// MongoDB Duplicate Error
	if (err.code === 11000) {
		const simplifiedError = handlerDuplicateError(err);
		statusCode = simplifiedError.statusCode;
		message = simplifiedError.message;
		errorSources = simplifiedError.errorSources ?? [];
	}
	// Mongoose Cast Error (ভুল ObjectId)
	else if (err.name === "CastError") {
		const simplifiedError = handleCastError(err);
		statusCode = simplifiedError.statusCode;
		message = simplifiedError.message;
		errorSources = simplifiedError.errorSources ?? [];
	}
	// Zod Error
	else if (err.name === "ZodError") {
		const simplifiedError = handlerZodError(err);
		statusCode = simplifiedError.statusCode;
		message = simplifiedError.message;
		errorSources = simplifiedError.errorSources ?? [];
	}
	// Mongoose Validation Error
	else if (err.name === "ValidationError") {
		const simplifiedError = handlerValidationError(err);
		statusCode = simplifiedError.statusCode;
		message = simplifiedError.message;
		errorSources = simplifiedError.errorSources ?? [];
	}
	// আমাদের custom error
	else if (err instanceof AppError) {
		statusCode = err.statusCode;
		message = err.message;
	}
	// সাধারণ JS Error
	else if (err instanceof Error) {
		statusCode = 500;
		message = err.message;
	}

	res.status(statusCode).json({
		success: false,
		message,
		errorSources,
		err: envVars.NODE_ENV === "development" ? err : null,
		stack: envVars.NODE_ENV === "development" ? err.stack : null,
	});
};
```

---

## সারাংশ

| Error | কীভাবে চিনি | কখন আসে | Status | `errorSources` |
|---|---|---|---|---|
| Duplicate | `err.code === 11000` | `unique` field-এ একই value | 400 | ✅ (field) |
| Cast | `err.name === "CastError"` | ভুল ObjectId (`findById`) | 400 | ✅ |
| Zod | `err.name === "ZodError"` | `validateRequest`-এ rule ভাঙলে | 400 | ✅ (প্রতিটা issue) |
| Mongoose Validation | `err.name === "ValidationError"` | `save/create`-এ schema rule ভাঙলে | 400 | ✅ (প্রতিটা field) |
| AppError | `instanceof AppError` | আমরা নিজে `throw` করলে | আমরা দিই | ❌ |
| JS Error | `instanceof Error` | অপ্রত্যাশিত bug | 500 | ❌ |

> 📌 **এখনো যা বাকি:** JWT error (`TokenExpiredError`, `JsonWebTokenError`) এখনো আলাদা handle হয়নি, তাই expired token দিলে `500` যায়। এটা `401` হওয়া উচিত। একই pattern-এ (`err.name` দিয়ে চিনে) পরে যোগ করা যাবে।