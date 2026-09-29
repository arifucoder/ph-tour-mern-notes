# Auth Module: Login, JWT আর `checkAuth`

## Step 1: Auth module বানানো

`src/app/modules`-এর ভিতরে `auth` নামে folder বানাব:

```
src/app/modules/auth/
├── auth.route.ts
├── auth.controller.ts
└── auth.service.ts
```

Login, logout, forgot password, reset/change password, এই ধরনের সব API এখানে থাকবে।

Auth module-এর **নিজস্ব কোনো model নেই**, কারণ auth-এর আলাদা কোনো data database-এ রাখা হয় না। এটা মূলত business logic handle করে, আর দরকার হলে `User` model থেকে data নিয়ে আসে।

---

## Step 2: Login (Service → Controller → Route)

### Service

`auth.service.ts`

```ts
import bcryptjs from "bcryptjs";
import httpStatus from "http-status-codes";
import AppError from "../../errorHelpers/AppError";
import type { IUser } from "../user/user.interface";
import { User } from "../user/user.model";

const credentialsLogin = async (payload: Partial<IUser>) => {
	const { email, password } = payload;

	// ১. এই email-এ কোনো user আছে কিনা
	const isUserExist = await User.findOne({ email });

	if (!isUserExist) {
		throw new AppError(httpStatus.BAD_REQUEST, "Email does not exist");
	}

	// ২. password মিলছে কিনা
	const isPasswordMatched = await bcryptjs.compare(password as string, isUserExist.password as string);

	if (!isPasswordMatched) {
		throw new AppError(httpStatus.BAD_REQUEST, "Incorrect password");
	}

	return {
		email: isUserExist.email,
	};
};

export const AuthServices = {
	credentialsLogin,
};
```

- **`Partial<IUser>`**: Login-এ শুধু `email` আর `password` লাগে, `IUser`-এর সব field না।
- **`bcryptjs.compare()`**: Body-তে আসা plain password আর database-এর hashed password মিলিয়ে দেখে।

### Controller

`auth.controller.ts`

```ts
import type { NextFunction, Request, Response } from "express";
import httpStatus from "http-status-codes";
import { catchAsync } from "../../utils/catchAsync";
import { sendResponse } from "../../utils/sendResponse";
import { AuthServices } from "./auth.service";

const credentialsLogin = catchAsync(async (req: Request, res: Response, next: NextFunction) => {
	const loginInfo = await AuthServices.credentialsLogin(req.body);

	sendResponse(res, {
		success: true,
		statusCode: httpStatus.OK,
		message: "User logged in successfully",
		data: loginInfo,
	});
});

export const AuthControllers = {
	credentialsLogin,
};
```

### Route

`auth.route.ts`

```ts
import { Router } from "express";
import { AuthControllers } from "./auth.controller";

const router = Router();

router.post("/login", AuthControllers.credentialsLogin);

export const AuthRoutes = router;
```

`src/app/routes/index.ts`-এর `moduleRoutes`-এ যোগ করব:

```ts
{
	path: "/auth",
	route: AuthRoutes,
},
```

এখন login URL: `POST http://localhost:5000/api/v1/auth/login`

---

## Step 3: JWT কেন দরকার?

Login-এর পর user payment, booking, profile update এর মতো কাজ করবে। প্রতিবার আমাদের জানতে হবে সে আসলেই login করা user কিনা।

### Ticket-এর উদাহরণ

চিড়িয়াখানায় ঢোকার সময় টিকিট কাটলে gateman একটা অংশ রেখে বাকি অংশ তোমাকে দেয়। ভিতরে খাবার বা পানি নিতে গেলে সেই টিকিট দেখাতে হয়। টিকিট না থাকলে কিছু পাবে না। আর একই টিকিট বারবার দেখানো যায়।

ঠিক তেমনি, login করলে আমরা user-কে একটা "টিকিট" দেব, যার নাম **JWT token**। পরে কোনো private কাজ করতে গেলে সে এই token দেখাবে, আর আমরা check করে তাকে সামনে যেতে দেব।

```
Login → Token পায় → Booking করতে চায় → Token দেখায় → Booked / Cancel / Payment
```

### Authentication আর Authorization

- **Authentication**: তুমি কে? (login করা user কিনা, token দিয়ে বোঝা যায়)
- **Authorization**: তোমার কী করার অনুমতি আছে? (যেমন শুধু admin সব user দেখতে পারবে, role দিয়ে বোঝা যায়)

---

## Step 4: JWT token-এর তিনটা অংশ

JWT token-এ তিনটা অংশ থাকে, প্রতিটা dot (`.`) দিয়ে আলাদা করা:

```
xxxxx.yyyyy.zzzzz
Header.Payload.Signature
```

1. **Header**: Token কোন algorithm দিয়ে sign করা হয়েছে (যেমন `HS256`), সেই তথ্য।
2. **Payload**: আমরা token-এ যে data রাখতে চাই, যেমন user-এর `id`, `email`, `role`।
3. **Signature**: Header, payload আর আমাদের **secret** মিলিয়ে তৈরি হয়। এটা দিয়ে check করা হয় token-টা আমাদের server-এরই বানানো কিনা, আর কেউ মাঝপথে বদলে দেয়নি কিনা।

> **খুব জরুরি:** JWT **encrypt** করা থাকে না, শুধু **sign** করা থাকে। যে কেউ [jwt.io](https://www.jwt.io/)-তে গিয়ে payload পড়ে ফেলতে পারে। তাই token-এ কখনো password বা গোপন তথ্য রাখা যাবে না।

---

## Step 5: Token বানানো

`jsonwebtoken` আর তার types আমরা project setup-এর সময়ই install করেছি। না করে থাকলে:

```bash
npm i jsonwebtoken
npm i -D @types/jsonwebtoken
```

`jwt.sign()` তিনটা জিনিস নেয়:

```ts
const accessToken = jwt.sign(jwtPayload, "secret", {
	expiresIn: "1d",
});
```

1. **Payload**: Token-এ যে data রাখব।
2. **Secret**: একটা গোপন key, যেটা কারো সাথে share করা যাবে না।
3. **Options**: যেমন `expiresIn`, token কতক্ষণ কাজ করবে (`"1d"` মানে ১ দিন)।

Login করে পাওয়া token [jwt.io](https://www.jwt.io/)-তে দিলে payload-এর তথ্য দেখা যাবে। সঠিক secret দিলে signature **verified** দেখাবে, ভুল secret দিলে দেখাবে না।

### Secret `.env`-এ রাখা

Secret কখনো কোডে সরাসরি লিখব না। `.env`-এ রাখব:

```env
JWT_ACCESS_SECRET=your_super_secret_key
JWT_ACCESS_EXPIRES=1d
```

`src/app/config/env.ts`-এ `EnvConfig` interface, `requiredEnvVariables` array আর return object, তিন জায়গাতেই এই দুটো যোগ করতে হবে:

```ts
JWT_ACCESS_SECRET: process.env.JWT_ACCESS_SECRET as string,
JWT_ACCESS_EXPIRES: process.env.JWT_ACCESS_EXPIRES as string,
```

> **`.env` আপডেট করার পর অবশ্যই server restart করতে হবে।**

---

## Step 6: JWT Helper বানানো

`sign` আর `verify`-এর জন্য ছোট দুটো helper function বানাব।

`src/app/utils/jwt.ts`

```ts
import jwt, { type JwtPayload, type SignOptions } from "jsonwebtoken";

export const generateToken = (payload: JwtPayload, secret: string, expiresIn: string) => {
	const token = jwt.sign(payload, secret, {
		expiresIn,
	} as SignOptions);

	return token;
};

export const verifyToken = (token: string, secret: string) => {
	const verifiedToken = jwt.verify(token, secret);

	return verifiedToken;
};
```

> **`as SignOptions` কেন?** `expiresIn` যেকোনো `string` নেয় না, শুধু `"1d"`, `"2h"` এর মতো নির্দিষ্ট format নেয়। আমাদের value `.env` থেকে আসে বলে TypeScript এটাকে সাধারণ `string` ধরে, তাই error দেয়। `as SignOptions` দিয়ে সেটা ঠিক করেছি।

### Service-এ token যোগ করা

Password check-এর পরে:

```ts
const jwtPayload = {
	userId: isUserExist._id,
	email: isUserExist.email,
	role: isUserExist.role,
};

const accessToken = generateToken(jwtPayload, envVars.JWT_ACCESS_SECRET, envVars.JWT_ACCESS_EXPIRES);

return {
	accessToken,
};
```

---

## Step 7: `checkAuth` middleware দিয়ে route protect করা

User private route-এ request পাঠানোর সময় token-টা **headers-এর `authorization`** property-তে দিয়ে পাঠাবে। Postman-এ: Headers → Key: `Authorization`, Value: token।

সব private route-এ token check করার জন্য একটা **higher order function** middleware বানাব।

`src/app/middlewares/checkAuth.ts`

```ts
import type { NextFunction, Request, Response } from "express";
import type { JwtPayload } from "jsonwebtoken";
import { envVars } from "../config/env";
import AppError from "../errorHelpers/AppError";
import { verifyToken } from "../utils/jwt";

export const checkAuth =
	(...authRoles: string[]) =>
	async (req: Request, res: Response, next: NextFunction) => {
		try {
			const accessToken = req.headers.authorization;

			// ১. token আছে কিনা
			if (!accessToken) {
				throw new AppError(401, "No token received");
			}

			// ২. token সঠিক কিনা (ভুল বা expired হলে এখানেই error)
			const verifiedToken = verifyToken(accessToken, envVars.JWT_ACCESS_SECRET) as JwtPayload;

			// ৩. এই role-এর অনুমতি আছে কিনা
			if (!authRoles.includes(verifiedToken.role)) {
				throw new AppError(403, "You are not permitted to view this route!");
			}

			req.user = verifiedToken;
			next();
		} catch (error) {
			next(error);
		}
	};
```

### কিছু ব্যাখ্যা

- **`if (!accessToken)` কেন?** `req.headers.authorization`-এর type `string | undefined`। এই check না দিলে TypeScript জানে না token আছে কিনা, তাই `verifyToken`-এ error দেয়।
- **`jwt.verify()`**: Token ভুল হলে, secret না মিললে বা মেয়াদ শেষ (expired) হলে এটা নিজেই error `throw` করে। তাই আলাদা করে `if (!verifiedToken)` check করার দরকার নেই।
- **`...authRoles`**: একটা route একাধিক role-এর জন্য খোলা থাকতে পারে (যেমন admin আর super admin দুজনেই)। তাই rest parameter দিয়ে যতগুলো খুশি role নেওয়া যায়। `authRoles.includes(verifiedToken.role)` check করে user-এর role তালিকায় আছে কিনা।
- **`401` আর `403`**: Token না থাকলে বা ভুল হলে `401` (Unauthorized, মানে তুমি কে জানি না)। Token ঠিক আছে কিন্তু অনুমতি নেই, তখন `403` (Forbidden)।
- **`req.user = verifiedToken`**: Token-এর data `req`-এ রেখে দিচ্ছি, যাতে পরে controller জানতে পারে কে request করছে।

### `req.user`-এর type যোগ করা

Express-এর `Request`-এ আগে থেকে `user` নামে কিছু নেই। তাই `checkAuth`-এর এই লাইনে VS Code লাল দাগ দেখাবে:

```ts
req.user = verifiedToken;
// Error: Property 'user' does not exist on type 'Request'
```

এজন্য একটা type file বানাব:

`src/app/interfaces/index.d.ts`

```ts
import type { JwtPayload } from "jsonwebtoken";

declare global {
	namespace Express {
		interface Request {
			user: JwtPayload;
		}
	}
}
```

#### এটা ছাড়াও তো server চলে, তাহলে কেন লাগবে?

আমরা project চালাই `tsx` দিয়ে, আর `tsx` type check করে না, শুধু কোড চালিয়ে দেয়। তাই type error থাকলেও server ঠিকমতো চলে। কিন্তু:

- VS Code-এ লাল দাগ থেকে যাবে।
- `npx tsc` চালালে error দেখাবে।
- `req.user.role` লিখতে গেলে কোনো suggestion পাওয়া যাবে না।

তাই সঠিক TypeScript কোডের জন্য এই file রাখা জরুরি।

#### এটা কীভাবে কাজ করে? (Declaration Merging)

Express-এর `Request` একটা interface, যেখানে `body`, `headers`, `params` এর মতো property আগে থেকেই লেখা আছে। TypeScript-এ একই নামের interface আবার লিখলে নতুন property গুলো পুরোনোটার সাথে **যোগ হয়ে যায়**, পুরোনোটা মুছে যায় না। একে বলে **declaration merging**।

```
Express-এর Request          আমাদের যোগ করা          ফলাফল
─────────────────          ──────────────          ──────────────
body                                               body
headers             +      user          =         headers
params                                             params
...                                                ...
                                                   user ✅
```

#### লাইন ধরে ব্যাখ্যা

- **`namespace Express { interface Request { ... } }`**: Express-এর `Request` interface-এর ভিতরে ঢুকে বলছি, "এখানে `user` নামে একটা property যোগ করো, যার type হবে `JwtPayload`"।
- **`declare global { ... }`**: File-এর উপরে `import` আছে, তাই TypeScript এটাকে একটা আলাদা module ধরে। `declare global` দিয়ে বলছি, "এই পরিবর্তন শুধু এই file-এ না, **পুরো project-এ** প্রযোজ্য"।
- **`.d.ts` file**: `d` মানে declaration। এখানে শুধু type থাকে, কোনো চলমান কোড থাকে না।

#### কোথাও import করতে হবে না

`tsconfig.json`-এ `"include": ["src"]` আছে, তাই `src`-এর ভিতরের এই file TypeScript নিজে থেকেই খুঁজে নেয়।

এরপর project-এর যেকোনো জায়গায় `req.user` লিখলে আর error আসবে না, বরং `req.user.role`, `req.user.email` লিখতে গেলে suggestion-ও দেখাবে। পরে controller-এ "কে request করছে" জানতে এটা অনেক কাজে লাগবে।

### Route-এ ব্যবহার

`user.route.ts`

```ts
router.get("/all-users", checkAuth(Role.ADMIN, Role.SUPER_ADMIN), UserControllers.getAllUsers);
```

এখন:

- Token ছাড়া → data পাবে না
- ভুল token বা ভুল secret → data পাবে না
- Token expired (যেমন `expiresIn: "2s"` দিয়ে test করলে) → data পাবে না
- সাধারণ `USER` role → data পাবে না
- `ADMIN` বা `SUPER_ADMIN` → data পাবে ✅

> **Note:** JWT-এর error (যেমন expired token) এখন `globalErrorHandler`-এ `500` নিয়ে যাবে। আসলে এটা `401` হওয়া উচিত। Mongoose আর Zod error-এর মতো এটাও পরে আলাদা করে handle করব।