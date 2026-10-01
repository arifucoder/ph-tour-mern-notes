# Refresh Token

## সমস্যা কী?

আগে আমরা [access token](https://github.com/arifucoder/ph-tour-mern-notes/blob/main/server/14-login-jwt-access-token.md) দেখেছিলাম, যার মেয়াদ (`expiresIn`) দিয়েছিলাম `1d`।

এখন ধরো তুমি কাজ করছ, আর কাজের মাঝখানেই token-এর মেয়াদ শেষ হয়ে গেল। তখন হঠাৎ তোমাকে logout করে দেবে, আবার login করতে হবে। এতে user experience খুব খারাপ হয়।

## সমাধান: Refresh Token

Login-এর সময় user-কে **দুটো token** দেব:

| Token | কাজ | মেয়াদ (আমাদের project-এ) |
| --- | --- | --- |
| **Access Token** | Private route ব্যবহার করার জন্য | `1d` |
| **Refresh Token** | Access token শেষ হলে নতুন access token বানানোর জন্য | `30d` |

Refresh token হলো একটা **backup token**। Access token-এর মেয়াদ শেষ হলে সেটা আর কোনোভাবেই ব্যবহার করা যায় না। তখন refresh token দেখিয়ে নতুন একটা access token নিয়ে নেওয়া যায়, user-কে আর logout হতে হয় না।

### Ticket-এর উদাহরণ

চিড়িয়াখানার টিকিট হারিয়ে গেলে তোমাকে আর ঢুকতে দেবে না। কিন্তু টিকিট কাটার সময় যদি তোমার email বা phone-এ টিকিটের নম্বর পাঠিয়ে দেওয়া হয়, তাহলে টিকিট হারালেও সেটা দেখিয়ে আবার নতুন টিকিট পাবে।

- **কাগজের টিকিট** = Access token
- **Email/SMS-এ পাঠানো নম্বর** = Refresh token

### সাধারণত মেয়াদ কত থাকে?

Access token-এর মেয়াদ **ছোট** রাখা হয় (যেমন ১৫ মিনিট থেকে ১ দিন), কারণ এটা প্রতিটা request-এ পাঠানো হয়, তাই চুরি হওয়ার ঝুঁকি বেশি। Refresh token-এর মেয়াদ **বড়** রাখা হয় (যেমন ৭ দিন থেকে কয়েক মাস)।

### ❓ ৩০ দিন পর refresh token শেষ হলে কি কাজের মাঝে logout হয়ে যাবে?

হ্যাঁ। Refresh token-এর মেয়াদ শেষ হলে নতুন access token বানানো যাবে না, তাই user-কে আবার login করতে হবে। তবে এটা ৩০ দিনে একবার হবে, প্রতিদিন না, তাই খুব একটা অসুবিধা হয় না।

> অনেক বড় app-এ নতুন access token দেওয়ার সময় একটা **নতুন refresh token**-ও দিয়ে দেয় (একে **refresh token rotation** বলে)। এতে নিয়মিত ব্যবহার করা user প্রায় কখনোই logout হয় না।

---

## Step 1: `.env` আপডেট

```env
JWT_REFRESH_SECRET=your_refresh_secret_key
JWT_REFRESH_EXPIRES=30d
```

`src/app/config/env.ts`-এর তিন জায়গায় (`EnvConfig`, `requiredEnvVariables`, return object) এগুলো যোগ করতে হবে।

> Refresh token-এর secret, access token-এর secret থেকে **আলাদা** হতে হবে।

---

## Step 2: `IUser`-এ `_id` যোগ করা

একটু পরে আমরা `user._id` ব্যবহার করব। কিন্তু `IUser` interface-এ `_id` নেই, তাই TypeScript error দেবে। এজন্য `user.interface.ts`-এ যোগ করব:

```ts
import type { Types } from "mongoose";

export interface IUser {
	_id?: Types.ObjectId;
	name: string;
	email: string;
	// ... বাকি সব আগের মতো
}
```

### `Types.ObjectId` কী?

MongoDB প্রতিটা document-এ নিজে থেকেই একটা unique `_id` বানিয়ে দেয়, যেমন `665f1a2b3c4d5e6f7a8b9c0d`। দেখতে string-এর মতো হলেও এটা আসলে একটা বিশেষ type, যার নাম **ObjectId**। আর `Types.ObjectId` হলো Mongoose-এ সেই type-এর TypeScript নাম।

`_id?` optional রেখেছি, কারণ user তৈরি করার আগে `_id` থাকে না, database-এ save হওয়ার পর MongoDB নিজে দেয়।

---

## Step 3: Token বানানোর utility function

Access আর refresh token বানানোর কাজটা আলাদা একটা function-এ রাখব।

`src/app/utils/userTokens.ts`

```ts
import { envVars } from "../config/env";
import type { IUser } from "../modules/user/user.interface";
import { generateToken } from "./jwt";

export const createUserTokens = (user: Partial<IUser>) => {
	const jwtPayload = {
		userId: user._id,
		email: user.email,
		role: user.role,
	};

	const accessToken = generateToken(jwtPayload, envVars.JWT_ACCESS_SECRET, envVars.JWT_ACCESS_EXPIRES);

	const refreshToken = generateToken(jwtPayload, envVars.JWT_REFRESH_SECRET, envVars.JWT_REFRESH_EXPIRES);

	return {
		accessToken,
		refreshToken,
	};
};
```

- দুটো token-এর **payload একই**, শুধু secret আর মেয়াদ আলাদা।
- **User database থেকে এখানে আনছি না কেন?** Login service-এ user আগেই database থেকে আনা হয়েছে। এখানে আবার আনলে একই কাজের জন্য database-এ দুইবার যেতে হতো। তাই আগে থেকে পাওয়া user-টাই parameter হিসেবে নিচ্ছি।

---

## Step 4: Login service আপডেট

`auth.service.ts`

```ts
const credentialsLogin = async (payload: Partial<IUser>) => {
	const { email, password } = payload;

	const isUserExist = await User.findOne({ email });

	if (!isUserExist) {
		throw new AppError(httpStatus.BAD_REQUEST, "Email does not exist");
	}

	const isPasswordMatched = await bcryptjs.compare(password as string, isUserExist.password as string);

	if (!isPasswordMatched) {
		throw new AppError(httpStatus.BAD_REQUEST, "Incorrect password");
	}

	const userTokens = createUserTokens(isUserExist);

	// eslint-disable-next-line @typescript-eslint/no-unused-vars
	const { password: pass, ...rest } = isUserExist.toObject();

	return {
		accessToken: userTokens.accessToken,
		refreshToken: userTokens.refreshToken,
		user: rest,
	};
};
```

### User-এর data কেন পাঠাচ্ছি?

Login-এর পর frontend-এ user-এর নাম, ছবি, role ইত্যাদি দেখাতে হয়। তাই token-এর সাথে user-এর data-ও পাঠাচ্ছি, যাতে frontend সেটা state-এ রেখে ব্যবহার করতে পারে।

### Password বাদ দেওয়া

```ts
const { password: pass, ...rest } = isUserExist.toObject();
```

- **`password: pass`**: `password` বের করে `pass` নামে রাখছি (নাম বদলাচ্ছি, কারণ উপরে `password` নামে আগেই একটা variable আছে)।
- **`...rest`**: Password ছাড়া বাকি সব field `rest`-এ।
- **`.toObject()`**: Mongoose-এর document একটা সাধারণ object না, এর ভিতরে Mongoose-এর অনেক নিজস্ব জিনিস থাকে। `.toObject()` এটাকে সাধারণ JavaScript object বানিয়ে দেয়, যাতে destructure ঠিকমতো কাজ করে।
- `pass` কোথাও ব্যবহার হচ্ছে না, তাই ESLint comment দিয়েছি।

এতে password আর response-এ যাচ্ছে না। ✅

---

## Step 5: Cookie-তে token রাখা

Token গুলো browser-এর **cookie**-তে রাখব। পরে নতুন access token বানানোর সময় cookie থেকেই refresh token পড়ব।

### `cookie-parser` install

Cookie পড়ার জন্য [`cookie-parser`](https://www.npmjs.com/package/cookie-parser) package লাগবে:

```bash
npm i cookie-parser
npm i -D @types/cookie-parser
```

`app.ts`-এ `express.json()`-এর উপরে যোগ করব:

```ts
import cookieParser from "cookie-parser";

app.use(cookieParser());
app.use(express.json());
```

এটা না দিলে `req.cookies` হবে `undefined`।

### `setAuthCookie` utility

`src/app/utils/setCookie.ts`

```ts
import type { Response } from "express";

export interface AuthTokens {
	accessToken?: string;
	refreshToken?: string;
}

export const setAuthCookie = (res: Response, tokenInfo: AuthTokens) => {
	if (tokenInfo.accessToken) {
		res.cookie("accessToken", tokenInfo.accessToken, {
			httpOnly: true,
			secure: false,
		});
	}

	if (tokenInfo.refreshToken) {
		res.cookie("refreshToken", tokenInfo.refreshToken, {
			httpOnly: true,
			secure: false,
		});
	}
};
```

- দুটো token-ই **optional**, কারণ login-এ দুটোই set করব, কিন্তু নতুন access token বানানোর সময় শুধু access token set করব।
- **`httpOnly: true`**: Browser-এর JavaScript (`document.cookie`) এই cookie পড়তে পারবে না, শুধু server পড়তে পারবে। এতে কোনো ক্ষতিকর script token চুরি করতে পারে না।
- **`secure: false`**: `true` দিলে cookie শুধু HTTPS-এ পাঠানো হয়। Local-এ আমরা HTTP ব্যবহার করি, তাই এখন `false`। Production-এ (live server) অবশ্যই `true` হতে হবে।

### Login controller-এ cookie set করা

`auth.controller.ts`

```ts
const credentialsLogin = catchAsync(async (req: Request, res: Response, next: NextFunction) => {
	const loginInfo = await AuthServices.credentialsLogin(req.body);

	setAuthCookie(res, loginInfo);

	sendResponse(res, {
		success: true,
		statusCode: httpStatus.OK,
		message: "User logged in successfully",
		data: loginInfo,
	});
});
```

---

## Step 6: Refresh token দিয়ে নতুন access token বানানো

### Utility function

`userTokens.ts`-এ নতুন function যোগ করব:

```ts
import httpStatus from "http-status-codes";
import type { JwtPayload } from "jsonwebtoken";
import AppError from "../errorHelpers/AppError";
import { IsActive } from "../modules/user/user.interface";
import { User } from "../modules/user/user.model";
import { verifyToken } from "./jwt";

export const createNewAccessTokenWithRefreshToken = async (refreshToken: string) => {
	// ১. refresh token verify (ভুল বা expired হলে নিজেই error দেয়)
	const verifiedRefreshToken = verifyToken(refreshToken, envVars.JWT_REFRESH_SECRET) as JwtPayload;

	// ২. user আছে কিনা, আর active কিনা
	const isUserExist = await User.findOne({ email: verifiedRefreshToken.email });

	if (!isUserExist) {
		throw new AppError(httpStatus.BAD_REQUEST, "User does not exist");
	}

	if (isUserExist.isActive === IsActive.BLOCKED || isUserExist.isActive === IsActive.INACTIVE) {
		throw new AppError(httpStatus.BAD_REQUEST, `User is ${isUserExist.isActive}`);
	}

	if (isUserExist.isDeleted) {
		throw new AppError(httpStatus.BAD_REQUEST, "User is deleted");
	}

	// ৩. আগের utility দিয়ে নতুন access token বানানো
	const userTokens = createUserTokens(isUserExist);

	return userTokens.accessToken;
};
```

- **নতুন করে `jwtPayload` আর `generateToken` লিখছি না কেন?** এই কাজটা Step 3-এর `createUserTokens`-এ আগেই আছে। আবার লিখলে একই কোড দুই জায়গায় থাকত (DRY নিয়মের বিরুদ্ধে)। `createUserTokens` দুটো token বানায়, আমরা এখান থেকে শুধু `accessToken` নিচ্ছি। বাড়তি একটা refresh token তৈরি হলেও সমস্যা নেই, এতে খুবই কম সময় লাগে।

- **Verify-এর পর আলাদা error check লাগছে না**, কারণ `verifyToken` ভুল বা expired token-এ নিজেই error `throw` করে।
- **User-এর অবস্থা check করা খুব জরুরি।** ধরো ১০ দিন আগে কেউ login করেছিল, তারপর admin তাকে block করে দিয়েছে। তার কাছে এখনও ৩০ দিনের refresh token আছে। এই check না থাকলে সে blocked হয়েও নতুন access token পেয়ে যেত।

> **💡 প্রচলিত নিয়ম: Refresh Token Rotation**
>
> বাস্তবে অনেক app নতুন access token দেওয়ার সময় একটা **নতুন refresh token**-ও দিয়ে দেয়, যার মেয়াদ আবার নতুন করে ৩০ দিন শুরু হয়। ফলে:
>
> - যে user নিয়মিত app ব্যবহার করে, তার refresh token বারবার নতুন হতে থাকে, তাই সে কখনো logout হয় না।
> - শুধু যে user টানা ৩০ দিন app-ই খোলেনি, তাকে আবার login করতে হবে। আর সেটা স্বাভাবিক, কারণ সে তো তখন কাজ করছে না।
>
> আমাদের কোডে এটা করা খুব সহজ। শুধু `accessToken` return না করে দুটো token-ই return করলেই হবে:
>
> ```ts
> const userTokens = createUserTokens(isUserExist);
>
> return userTokens; // { accessToken, refreshToken }
> ```
>
> তারপর service-এ দুটোই return করলে controller-এর `setAuthCookie` দুটো token-কেই cookie-তে রেখে দেবে।

### Service

`auth.service.ts`

```ts
const getNewAccessToken = async (refreshToken: string) => {
	const newAccessToken = await createNewAccessTokenWithRefreshToken(refreshToken);

	return {
		accessToken: newAccessToken,
	};
};

export const AuthServices = {
	credentialsLogin,
	getNewAccessToken,
};
```

### Controller

`auth.controller.ts`

```ts
const getNewAccessToken = catchAsync(async (req: Request, res: Response, next: NextFunction) => {
	const refreshToken = req.cookies.refreshToken;

	if (!refreshToken) {
		throw new AppError(httpStatus.BAD_REQUEST, "No refresh token received from cookies");
	}

	const tokenInfo = await AuthServices.getNewAccessToken(refreshToken as string);

	setAuthCookie(res, tokenInfo);

	sendResponse(res, {
		success: true,
		statusCode: httpStatus.OK,
		message: "New access token retrieved successfully",
		data: tokenInfo,
	});
});
```

- **`req.cookies.refreshToken`**: Login-এর সময় cookie-তে যে refresh token রেখেছিলাম, সেটা এখানে পড়ছি। এজন্যই login-এ cookie set করা আর `cookieParser()` যোগ করা জরুরি ছিল।
- নতুন access token পাওয়ার পর সেটাও আবার cookie-তে রেখে দিচ্ছি।

> **Cookie কোথায় কোথায় set/clear করতে হবে:** Login, নতুন access token বানানো আর পরে social (Google) login-এর সময় cookie **set** করতে হবে। আর logout-এর সময় cookie **clear** করতে হবে।

### Route

`auth.route.ts`

```ts
router.post("/refresh-token", AuthControllers.getNewAccessToken);
```

URL: `POST http://localhost:5000/api/v1/auth/refresh-token`

---

## পুরো Flow

```
Login
  ↓
Access token + Refresh token → cookie-তে রাখা
  ↓
Access token দিয়ে private route ব্যবহার
  ↓
Access token expired ❌
  ↓
/refresh-token-এ request (cookie থেকে refresh token যায়)
  ↓
নতুন access token → cookie-তে রাখা
  ↓
আবার private route ব্যবহার ✅
```

---

## Test করা

1. `.env`-এ `JWT_ACCESS_EXPIRES=10s` দিয়ে server restart করো।
2. Postman-এ login করো। Postman নিজেই cookie রেখে দেয়।
3. ১০ সেকেন্ড পর private route-এ request দিলে error আসবে (token expired)।
4. `/refresh-token`-এ `POST` request দাও, নতুন access token পাবে।
5. নতুন token দিয়ে আবার private route-এ request দিলে data পাবে। ✅

Test শেষে `JWT_ACCESS_EXPIRES` আবার `1d` করে দিতে ভুলো না।

> **Frontend-এ কী হবে?** সাধারণত frontend নিজেই check করে access token expired কিনা। Expired হলে user-কে কিছু বুঝতে না দিয়ে refresh token দিয়ে নতুন access token নিয়ে নেয়। এটা আমরা frontend-এর অংশে দেখব।