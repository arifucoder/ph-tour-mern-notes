# Passport Local দিয়ে Credentials Login

আগে email + password login-এর সব business logic ছিল `auth.service.ts`-এর `credentialsLogin`-এ। এখন সেই কাজটা **`passport-local` strategy** দিয়ে করব। Google login-এর মতো local login-ও তখন Passport-এর মাধ্যমে চলবে।

---

## পুরো Flow

```
POST /api/v1/auth/login  (body: { email, password })
        │
        ▼
AuthControllers.credentialsLogin
        │
        ▼
passport.authenticate("local", callback)(req, res, next)
        │
        ▼
LocalStrategy-র verify function (email, password, done)
   ├─ user নেই?                → done(...)  ❌
   ├─ Google user, password নেই? → done(...)  ❌
   ├─ password মিলল না?        → done(...)  ❌
   └─ সব ঠিক                   → done(null, user) ✅
        │
        ▼
controller-এর callback (err, user, info)
   ├─ err থাকলে   → return next(AppError) → globalErrorHandler
   ├─ user না থাকলে → return next(AppError(info.message))
   └─ user থাকলে  → token বানাও → cookie set → sendResponse
```

---

## Step 1: `passport.ts`-এ Local Strategy যোগ করা

`passport-local` আর `passport-google-oauth20` — দুটো package-ই তাদের class-এর নাম `Strategy` দিয়েছে। তাই নাম clash এড়াতে rename করে import করি: `Strategy as LocalStrategy`।

`LocalStrategy` constructor-এ **দুটো argument** দিতে হয়:

| Argument | কী থাকে |
|---|---|
| ১ম: `options` object | কোন field-কে username আর password ধরা হবে (`usernameField`, `passwordField`) |
| ২য়: `verify` function | `async (email, password, done) => {}` — এখানে business logic validation |

> Google strategy-তে ১ম argument-এ `clientID`, `clientSecret`, `callbackURL` দিতাম। Local-এ এগুলো লাগে না।

### `usernameField` alias কেন?

`passport-local` default-ভাবে `req.body.username` আর `req.body.password` খোঁজে। আমাদের project-এ username নেই, আছে email। তাই alias করে দিই:

```ts
{
	usernameField: "email",   // req.body.email কে username হিসেবে ধরো
	passwordField: "password",
}
```

এরপর Passport নিজেই `req.body.email` আর `req.body.password` বের করে verify function-এর parameter হিসেবে দিয়ে দেয়।

### Code

```ts
// src/app/config/passport.ts
import bcryptjs from "bcryptjs";
import passport from "passport";
import { Strategy as LocalStrategy } from "passport-local";
import { User } from "../modules/user/user.model";

passport.use(
	new LocalStrategy(
		{
			usernameField: "email",
			passwordField: "password",
		},
		async (email: string, password: string, done) => {
			try {
				const isUserExist = await User.findOne({ email });

				// Pattern 1 (error হিসেবে পাঠানো)
				// if (!isUserExist) {
				//     return done("User does not exist");
				// }

				// Pattern 2 (fail হিসেবে পাঠানো) ✅ recommended
				if (!isUserExist) {
					return done(null, false, { message: "User does not exist" });
				}

				// if (!isUserExist) {
				// 	return done("User does not exist");
				// }

				if (!isUserExist.isVerified) {
					// throw new AppError(httpStatus.BAD_REQUEST, "User is not verified")
					return done("User is not verified");
				}

				if (isUserExist.isActive === IsActive.BLOCKED || isUserExist.isActive === IsActive.INACTIVE) {
					// throw new AppError(httpStatus.BAD_REQUEST, `User is ${isUserExist.isActive}`)
					return done(`User is ${isUserExist.isActive}`);
				}
				if (isUserExist.isDeleted) {
					// throw new AppError(httpStatus.BAD_REQUEST, "User is deleted")
					return done("User is deleted");
				}

				const isGoogleAuthenticated = isUserExist.auths.some(
					(providerObjects) => providerObjects.provider === "google",
				);

				// Google দিয়ে account খুলেছে কিন্তু এখনো password set করেনি
				if (isGoogleAuthenticated && !isUserExist.password) {
					return done(null, false, {
						message:
							"You have authenticated through Google. So if you want to login with credentials, then at first login with google and set a password for your Gmail and then you can login with email and password.",
					});
				}

				// কোনো কারণে password না থাকলে bcrypt.compare crash করবে, তাই আগেই আটকাই
				if (!isUserExist.password) {
					return done(null, false, { message: "Password is not set for this account" });
				}

				const isPasswordMatched = await bcryptjs.compare(password, isUserExist.password);

				if (!isPasswordMatched) {
					return done(null, false, { message: "Password does not match" });
				}

				return done(null, isUserExist);
			} catch (error) {
				return done(error);
			}
		},
	),
);
```

### কিছু point

- **এখানে user create করি না।** Google strategy-তে user না থাকলে বানাতাম, কিন্তু local হলো শুধু **login**। User আগে থেকে DB-তে থাকলেই login করতে পারবে (register আলাদা route-এ)।
- **Async function, তাই `try/catch` লাগবে।** DB বা bcrypt-এ unexpected error হলে `catch`-এ গিয়ে `done(error)`।
- **Google check-এ `!isUserExist.password` কেন জরুরি?**
  নিচের commented version ভুল:
  ```ts
  // ❌ শুধু Google কিনা দেখলে
  if (isGoogleAuthenticated) { return done("...") }
  ```
  কারণ, কেউ Google দিয়ে login করে পরে password set করলেও এই code তাকে **কখনোই** email-password দিয়ে login করতে দেবে না। তাই "Google user **এবং** password নেই" — দুটো একসাথে check করতে হবে।

---

## Step 2: `done` function-এর ৩টা ব্যবহার

`done`-এ যা পাঠাব, controller-এর callback `(err, user, info)`-তে ঠিক সেটাই আসবে।

| `done` call | callback-এ যা আসে | কোন branch-এ যাবে | কখন ব্যবহার |
|---|---|---|---|
| `done(error)` বা `done("message")` | `err` = error/string | `if (err)` | Unexpected error (DB down ইত্যাদি) |
| `done(null, false, { message })` | `user = false`, `info = { message }` | `if (!user)` | Business logic fail (user নেই, password ভুল) |
| `done(null, user)` | `user` = DB-র user | success | সব ঠিক |

> **২য় parameter `false` কেন?** কারণ login fail করেছে — দেওয়ার মতো কোনো user নেই। `false` মানে "authentication হয়নি"।

**দুটো error pattern-ই কাজ করে**, যেকোনো একটা দিলেই হয়। তবে ভালো নিয়ম:
- User ভুল তথ্য দিয়েছে (user নেই / password ভুল) → **`done(null, false, { message })`**
- সত্যিকারের crash/exception → **`done(error)`**

---

## Step 3: Controller-এ `passport.authenticate` ব্যবহার

Service-এর `credentialsLogin` আর লাগবে না, কারণ business logic এখন Passport করছে। তাই সেটা comment করে দিই।

### Route-এ middleware হিসেবে না দিয়ে controller-এর ভেতরে কেন?

সাধারণত route-এ function শুধু reference হিসেবে দিই, call করি না। কিন্তু আমরা চাই login-এর পর **নিজের মতো** token বানাব, cookie set করব, response পাঠাব। তাই `passport.authenticate("local", callback)` controller-এর ভেতরে লিখে নিজেরা call করি:

```ts
passport.authenticate("local", callback)(req, res, next);
//                                       ▲ এটাই call করছে
```

`passport.authenticate(...)` একটা middleware **return** করে, তাই শেষে `(req, res, next)` দিয়ে সেটা run করাতে হয়।

- `"local"` = strategy-র নাম (`passport-local` নিজেই এই নাম দিয়েছে)।
- Custom callback দিলে Passport নিজে session/`req.login` করে না — আমাদের সেটা লাগেও না, কারণ আমরা JWT ব্যবহার করি।

### Code

```ts
// src/app/modules/auth/auth.controller.ts
import type { NextFunction, Request, Response } from "express";
import type { HydratedDocument } from "mongoose";
import type { IVerifyOptions } from "passport-local";
import httpStatus from "http-status-codes";
import passport from "passport";
import AppError from "../../errorHelpers/AppError";
import { catchAsync } from "../../utils/catchAsync";
import { sendResponse } from "../../utils/sendResponse";
import { setAuthCookie } from "../../utils/setCookie";
import { createUserTokens } from "../../utils/userTokens";
import type { IUser } from "../user/user.interface";

const credentialsLogin = catchAsync(async (req: Request, res: Response, next: NextFunction) => {
	// const loginInfo = await AuthServices.credentialsLogin(req.body)

	passport.authenticate(
		"local",
		(err: unknown, user: HydratedDocument<IUser> | false, info: IVerifyOptions | undefined) => {
			if (err) {
				// done("User does not exist") → string আসে → 401
				if (typeof err === "string") {
					return next(new AppError(httpStatus.UNAUTHORIZED, err));
				}
				// done(error) → আসল Error → globalErrorHandler (500)
				return next(err);
			}

			if (!user) {
				return next(new AppError(httpStatus.UNAUTHORIZED, info?.message ?? "Login failed"));
			}

			const userTokens = createUserTokens(user);

			// eslint-disable-next-line @typescript-eslint/no-unused-vars
			const { password: pass, ...rest } = user.toObject();

			setAuthCookie(res, userTokens);

			sendResponse(res, {
				success: true,
				statusCode: httpStatus.OK,
				message: "User Logged In Successfully",
				data: {
					accessToken: userTokens.accessToken,
					refreshToken: userTokens.refreshToken,
					user: rest,
				},
			});
		},
	)(req, res, next);
});
```

> Route আগের মতোই: `router.post("/login", AuthControllers.credentialsLogin)`। আর `app.ts`-এ `passport.initialize()` ও `passport.ts` import আগেই (Google-এর সময়) করা আছে, তাই নতুন কিছু লাগবে না।

### ❓ `await createUserTokens(user)` — await কাজ করছে না কেন?

`createUserTokens` একটা **সাধারণ (sync) function** — ভেতরে `jwt.sign` sync চলে আর সরাসরি `{ accessToken, refreshToken }` object return করে, Promise না। তাই `await` দেওয়ার কিছু নেই; দিলে error হয় না, কিন্তু কোনো কাজও করে না। তাই `await` সরিয়ে দিয়েছি, আর callback-কেও `async` রাখার দরকার নেই।

### ❓ `delete user.toObject().password` কেন কাজ করে না?

`user.toObject()` প্রতিবার call করলে **নতুন** plain object বানায়। ওই নতুন object থেকে password delete হয়, কিন্তু পরে আবার `user.toObject()` করলে password আবার থাকে। তাই destructure করে `rest` নেওয়াই সঠিক: `const { password: pass, ...rest } = user.toObject()`।

---

## Step 4: Callback-এর ভেতরে Error handling — কোনটা ঠিক, কোনটা ভুল

| Code | ফলাফল | কারণ |
|---|---|---|
| `throw new AppError(401, "...")` | ❌ | Callback-টা পরে (strategy শেষ হলে) চলে, ততক্ষণে `catchAsync`-এর কাজ শেষ। তাই throw কেউ catch করে না → request ঝুলে থাকে বা unhandled error-এ server crash করতে পারে। |
| `next(err)` (`return` ছাড়া) | ❌ | Error পাঠানোর পরেও code নিচে চলতে থাকে → `sendResponse` আবার response পাঠাতে চায় → `Cannot set headers after they are sent` error। |
| `return new AppError(401, err)` | ❌ | শুধু একটা object বানিয়ে return করছে, কাউকে পাঠাচ্ছে না। Express কিছুই জানে না → Postman-এ request loading হতেই থাকে। |
| `return next(err)` | ⚠️ আংশিক ঠিক | `done(error)` (আসল Error) হলে ঠিক আছে। কিন্তু `done("User does not exist")` হলে `err` একটা string — globalErrorHandler-এ `err.message` থাকে না, status 500 যায়। |
| `return next(new AppError(401, err))` | ✅ | String error-কে AppError বানিয়ে globalErrorHandler-এ পাঠায় → সঠিক 401 আর message। |
| `return next(new AppError(401, info.message))` | ✅ | `done(null, false, { message })`-এর জন্য — `!user` branch। |

**মূল নিয়ম:** Passport callback-এর ভেতরে error পাঠাতে হলে **সবসময় `return next(...)`** — কখনো `throw` না।

> ⚠️ ভুল ধারণা ঠিক করা: "AppError আমাদের custom error, তাই পাঠানো যাবে না" — এটা ঠিক না। `next(new AppError(...))` পুরোপুরি ঠিক আছে। সমস্যা AppError-এ না, সমস্যা **`throw` করায়**।

---

## সারাংশ

| বিষয় | মনে রাখার কথা |
|---|---|
| Import | `Strategy as LocalStrategy` (Google-এর `Strategy`-র সাথে নাম clash এড়াতে) |
| Field alias | `usernameField: "email"` |
| Business logic | Service থেকে সরে strategy-র verify function-এ |
| `done` | fail → `done(null, false, { message })`, crash → `done(error)`, success → `done(null, user)` |
| Controller | `passport.authenticate("local", cb)(req, res, next)` |
| Callback-এ error | সবসময় `return next(...)`, কখনো `throw` না |