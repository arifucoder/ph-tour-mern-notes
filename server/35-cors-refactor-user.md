# 35 — User Update Refactor, Single User, Change Password Check আর CORS

এই note-এ কয়েকটা ছোট কিন্তু জরুরি পরিবর্তন:

1. User update API থেকে **password বাদ** — password বদলানো যাবে শুধু change password দিয়ে
2. **Admin কোনো Super Admin-এর profile বদলাতে পারবে না**
3. **Single user get** — শুধু admin আর super admin
4. Change password-এ **নতুন আর পুরোনো password এক কিনা** check
5. **CORS** ঠিকমতো setup — frontend আলাদা domain-এ deploy হলেও যেন কাজ করে

> আগের user update-এর পুরো ব্যাখ্যা: [16-user-update.md](./16-user-update.md)। এখানে শুধু যা বদলাচ্ছে।

---

# Part 1: User Update থেকে Password বাদ

## Step 1: কেন?

[Note 16](./16-user-update.md)-এ update API-তে password দিলে hash করে বদলে দেওয়া হতো:

```ts
if (payload.password) {
	payload.password = await bcryptjs.hash(payload.password, Number(envVars.BCRYPT_SALT_ROUND));
}
```

সমস্যা: এখানে **পুরোনো password চাওয়া হয় না**। মানে কেউ কয়েক মিনিটের জন্য user-এর খোলা browser বা token হাতে পেলে update API দিয়ে নতুন password বসিয়ে account দখল করতে পারত।

এখন আমাদের কাছে password-এর জন্য আলাদা, নিরাপদ API আছে:

| কাজ | API | পুরোনো password লাগে? |
|---|---|---|
| Password বদলানো | `/auth/change-password` ([note 20](./20-change-password.md)) | ✅ হ্যাঁ |
| Google user-এর প্রথম password | `/auth/set-password` ([note 30](./30-set-password-for-google-auth.md)) | না (আগে password ছিলই না) |
| Password ভুলে গেলে | `/auth/forgot-password` → `/auth/reset-password` ([note 32](./32-forgot-reset-password-nodemailer.md)) | না, email যাচাই করে |

তাই update API থেকে password পুরোপুরি সরিয়ে দিই।

## Step 2: দুই জায়গা থেকে সরানো

**① Zod schema থেকে** (এটাই আসল পাহারাদার):

```ts
// src/app/modules/user/user.validation.ts
export const updateUserZodSchema = z.object({
	name: ...,
	// password: ...   ← মুছে ফেলো
	phone: ...,
	address: ...,
	role: ...,
	isActive: ...,
	isDeleted: ...,
	isVerified: ...,
});
```

**② Service থেকে** — উপরের `if (payload.password) { ... }` block মুছে ফেলো, আর `bcryptjs`, `envVars` import অন্য কোথাও ব্যবহার না হলে সেগুলোও।

### ❓ Zod থেকে সরালে কী হয়?

Zod-এর `z.object()` default-ভাবে schema-তে **না থাকা field চুপচাপ বাদ** দেয় (strip)। তাই এখন কেউ body-তে `"password": "hack123"` পাঠালেও `validateRequest`-এর পর `req.body`-তে password থাকবেই না — service পর্যন্ত পৌঁছায়ই না।

```
Body:        { "name": "Arif", "password": "hack123" }
Zod-এর পর:   { "name": "Arif" }          ← password নেই
```

Service থেকে মোছাটা শুধু অপ্রয়োজনীয় code পরিষ্কার করা। আসল নিরাপত্তা দিচ্ছে Zod।

---

# Part 2: Update-এ কে কার Profile বদলাতে পারবে

## Step 3: দুটো Role — কে বদলাচ্ছে আর কাকে বদলাচ্ছে

Teacher-এর condition বুঝতে প্রথমে দুটো জিনিস আলাদা করতে হবে:

| Variable | কে | কোথা থেকে আসে |
|---|---|---|
| `decodedToken.role` | **যে update করছে** (request পাঠাচ্ছে) | Login token থেকে |
| `isUserExist.role` | **যার profile update হচ্ছে** (target) | DB থেকে, URL-এর `:id` দিয়ে খুঁজে |

উদাহরণ: Admin "Rahim" একজন user "Karim"-এর phone বদলাচ্ছে:

```
PATCH /api/v1/user/<Karim-এর id>
Authorization: <Rahim-এর token>

decodedToken.role = "ADMIN"    ← Rahim (যে বদলাচ্ছে)
isUserExist.role  = "USER"     ← Karim (যাকে বদলানো হচ্ছে)
```

## Step 4: Teacher-এর দুটো Condition

### Condition ১: USER/GUIDE শুধু নিজেরটা

```ts
if (decodedToken.role === Role.USER || decodedToken.role === Role.GUIDE) {
	if (userId !== decodedToken.userId) {
		throw new AppError(401, "You are not authorized");
	}
}
```

"যে বদলাচ্ছে সে সাধারণ user বা guide হলে, URL-এর id আর তার নিজের id এক হতে হবে।"

> 📝 এটা আমাদের [note 16](./16-user-update.md)-এ আগেই আছে (`isNormalUser && decodedToken.userId !== userId`) — নতুন করে যোগ করতে হবে না।

### Condition ২: Admin, Super Admin-কে ছুঁতে পারবে না

```ts
if (decodedToken.role === Role.ADMIN && isUserExist.role === Role.SUPER_ADMIN) {
	throw new AppError(401, "You are not authorized");
}
```

সহজ ভাষায়: **"যে বদলাচ্ছে সে ADMIN, আর যাকে বদলাচ্ছে সে SUPER_ADMIN — এমন হলে থামাও।"**

Super admin হলো system-এর সবচেয়ে উপরের মানুষ। Admin তার নিচে। নিচের কেউ উপরের জনের name, phone বদলাবে, তাকে block বা delete করে দেবে — এটা হতে দেওয়া যায় না। নাহলে একজন admin চাইলে super admin-কে `isActive: "BLOCKED"` করে পুরো system দখল করে নিতে পারত।

### সব Case এক নজরে

| যে update করছে ↓ / যাকে → | USER / GUIDE | ADMIN | SUPER_ADMIN |
|---|---|---|---|
| **USER / GUIDE** | শুধু নিজেকে ✅ | ❌ | ❌ |
| **ADMIN** | ✅ | ✅ | ❌ (Condition ২) |
| **SUPER_ADMIN** | ✅ | ✅ | ✅ |

> 💡 খেয়াল করো, এখন একজন admin **অন্য admin**-কে update/block করতে পারে। এটা teacher-এর design। চাইলে আরও কড়া করা যায় — admin শুধু USER/GUIDE-কে update করতে পারবে, অন্য admin-কে না:
> ```ts
> if (decodedToken.role === Role.ADMIN && isUserExist.role === Role.ADMIN && decodedToken.userId !== userId) {
> 	throw new AppError(httpStatus.FORBIDDEN, "Admins cannot update other admins.");
> }
> ```
> এটা project-এর নিয়মের উপর নির্ভর করে, বাধ্যতামূলক না।

### ❓ `401` নাকি `403`?

| Code | মানে | কখন |
|---|---|---|
| `401 Unauthorized` | "তুমি কে, জানি না" | Token নেই, ভুল, মেয়াদ শেষ |
| `403 Forbidden` | "তোমাকে চিনি, কিন্তু এই কাজের অনুমতি নেই" | Login করা আছে, কিন্তু role-এ কুলায় না |

এখানে admin login করা আছে, শুধু অনুমতি নেই — তাই **`403`**। (Note 16-এ আমরা আগেই `httpStatus.FORBIDDEN` ব্যবহার করছি।)

## Step 5: Updated `updateUser` Service

```ts
// src/app/modules/user/user.service.ts
import httpStatus from "http-status-codes";
import type { JwtPayload } from "jsonwebtoken";
import AppError from "../../errorHelpers/AppError";
import { Role, type IUser } from "./user.interface";
import { User } from "./user.model";

const updateUser = async (userId: string, payload: Partial<IUser>, decodedToken: JwtPayload) => {
	// ১. user আছে কিনা
	const isUserExist = await User.findById(userId);

	if (!isUserExist) {
		throw new AppError(httpStatus.NOT_FOUND, "User not found");
	}

	const isNormalUser = decodedToken.role === Role.USER || decodedToken.role === Role.GUIDE;

	// ২. সাধারণ user শুধু নিজের profile update করতে পারবে
	if (isNormalUser && decodedToken.userId !== userId) {
		throw new AppError(httpStatus.FORBIDDEN, "You can only update your own profile.");
	}

	// ৩. Admin কোনো Super Admin-এর profile update করতে পারবে না  ← নতুন
	if (decodedToken.role === Role.ADMIN && isUserExist.role === Role.SUPER_ADMIN) {
		throw new AppError(httpStatus.FORBIDDEN, "Admins cannot update a super admin's profile.");
	}

	// ৪. role update
	if (payload.role) {
		if (isNormalUser) {
			throw new AppError(httpStatus.FORBIDDEN, "You are not allowed to change roles.");
		}

		if (payload.role === Role.SUPER_ADMIN && decodedToken.role === Role.ADMIN) {
			throw new AppError(httpStatus.FORBIDDEN, "Only a super admin can create another super admin.");
		}
	}

	// ৫. isActive, isDeleted, isVerified update
	if (payload.isActive !== undefined || payload.isDeleted !== undefined || payload.isVerified !== undefined) {
		if (isNormalUser) {
			throw new AppError(httpStatus.FORBIDDEN, "You are not allowed to change account status.");
		}
	}

	// password-এর block আর নেই ← সরানো হয়েছে

	// ৬. update — response-এ password hash যেন না যায়
	const newUpdatedUser = await User.findByIdAndUpdate(userId, payload, {
		new: true,
		runValidators: true,
	}).select("-password");

	return newUpdatedUser;
};
```

### কী বদলালাম

| বিষয় | আগে | এখন |
|---|---|---|
| Password | Hash করে update | সরানো (Zod + service) |
| Admin → Super Admin | Check ছিল না | ৩ নম্বর condition |
| Variable নাম | Teacher: `ifUserExist` | আমাদের code-এর মতো `isUserExist` |
| Status | Teacher: `401` | `403` (`httpStatus.FORBIDDEN`) |
| Error message | সব জায়গায় "You are not authorized" | প্রতিটা আলাদা — কোন নিয়মে আটকাল বোঝা যায় |
| ⚠️ Response | Password hash সহ পুরো user | `.select("-password")` |

> ⚠️ **লুকানো সমস্যা ঠিক করলাম:** Note 16-এর code-এ `findByIdAndUpdate` পুরো user document return করত — **password hash সহ** — আর controller সেটাই response-এ পাঠাত। Profile দেখার API-তে আমরা password বাদ দিই ([note 31](./31-get-me.md)), update-এও একই নিয়ম লাগবে।

---

# Part 3: Single User Get (Admin-এর জন্য)

Admin dashboard-এ কোনো একজন user-এর বিস্তারিত দেখতে এই API লাগবে।

## Step 6: Route — Order খুব জরুরি

```ts
// src/app/modules/user/user.route.ts
router.get("/all-users", checkAuth(Role.ADMIN, Role.SUPER_ADMIN), UserControllers.getAllUsers);
router.get("/me", checkAuth(...Object.values(Role)), UserControllers.getMe);

// dynamic route সবার শেষে
router.get("/:id", checkAuth(Role.ADMIN, Role.SUPER_ADMIN), UserControllers.getSingleUser);
```

> ⚠️ **`/:id` অবশ্যই `/me`-র পরে।** উল্টো হলে `GET /user/me` request-এ Express "me"-কে একটা id ভেবে `/:id`-এ পাঠাবে:
> - সাধারণ user-এর জন্য → checkAuth আটকাবে (403), নিজের profile-ই দেখতে পারবে না
> - Admin-এর জন্য → `findById("me")` → CastError
>
> একই কারণে `/all-users`-ও `/:id`-এর আগে ([note 26](./26-get-single-by-slug.md)-এ route order বিস্তারিত)।

## Step 7: Controller

```ts
// src/app/modules/user/user.controller.ts
const getSingleUser = catchAsync(async (req: Request, res: Response) => {
	const id = req.params.id as string;
	const result = await UserServices.getSingleUser(id);

	sendResponse(res, {
		success: true,
		statusCode: httpStatus.OK,
		message: "User retrieved successfully",
		data: result.data,
	});
});
```

| বিষয় | Teacher | এখন | কেন |
|---|---|---|---|
| Status | `httpStatus.CREATED` (201) | `httpStatus.OK` (200) | শুধু পড়া হচ্ছে, কিছু তৈরি হয়নি ([note 31](./31-get-me.md)-এর মতোই) |
| `next` | ছিল | সরানো | ব্যবহার হয় না, ESLint error |
| `req.params.id` | সরাসরি | `as string` | `noUncheckedIndexedAccess`-এ type `string \| undefined` |

## Step 8: Service

```ts
// src/app/modules/user/user.service.ts
const getSingleUser = async (id: string) => {
	const user = await User.findById(id).select("-password");

	if (!user) {
		throw new AppError(httpStatus.NOT_FOUND, "User not found.");
	}

	return {
		data: user,
	};
};
```

- **`.select("-password")`**: Admin হলেও password hash দেখার কোনো দরকার নেই।
- ⚠️ **যোগ করেছি:** না পেলে `200` + `data: null` না, পরিষ্কার `404`।
- ভুল format-এর id (`/user/abc`) দিলে Mongoose `CastError` দেবে, যেটা globalErrorHandler সুন্দর করে 400 দেখায় ([note 23](./23-advanced-global-error-handler.md))।

---

# Part 4: Change Password-এ নতুন check

## Step 9: নতুন আর পুরোনো password এক হলে আটকানো

User যদি পুরোনো password-টাই আবার "নতুন" হিসেবে দেয়, তাহলে আসলে কিছুই বদলায় না — অথচ সে ভাববে বদলেছে। নিরাপত্তার দিক থেকেও password বদলানোর উদ্দেশ্যই হলো নতুন কিছু দেওয়া।

```ts
// src/app/modules/auth/auth.service.ts
const changePassword = async (oldPassword: string, newPassword: string, decodedToken: JwtPayload) => {
	// ০. নতুন আর পুরোনো এক কিনা — DB-তে যাওয়ার আগেই  ← নতুন
	if (oldPassword === newPassword) {
		throw new AppError(httpStatus.BAD_REQUEST, "New password must be different from the old password.");
	}

	const user = await User.findById(decodedToken.userId);

	if (!user) {
		throw new AppError(httpStatus.NOT_FOUND, "User not found");
	}

	// Google দিয়ে আসা user-এর password নেই  ← নতুন
	if (!user.password) {
		throw new AppError(
			httpStatus.BAD_REQUEST,
			"You don't have a password yet. Please use set password to create one.",
		);
	}

	// ১. পুরোনো password মিলছে কিনা
	const isOldPasswordMatch = await bcryptjs.compare(oldPassword, user.password);

	if (!isOldPasswordMatch) {
		throw new AppError(httpStatus.UNAUTHORIZED, "Old password does not match");
	}

	// ২. নতুন password hash করে save করা
	user.password = await bcryptjs.hash(newPassword, Number(envVars.BCRYPT_SALT_ROUND));

	await user.save();
};
```

### ❓ Check-টা সবার আগে কেন?

দুটো string সরাসরি তুলনা করা যায় — DB বা bcrypt কিছুই লাগে না। তাই এক মুহূর্তে উত্তর দিয়ে দেওয়া যায়। এক হলে আর DB-তে যাওয়ার, bcrypt-এ সময় খরচ করার দরকার নেই।

| পুরোনো | নতুন | ফলাফল |
|---|---|---|
| `Arif@123` | `Arif@123` | 400 "must be different" |
| `Arif@123` (ভুল) | `New@456` | 401 "Old password does not match" |
| `Arif@123` (সঠিক) | `New@456` | ✅ বদলে যাবে |

### ⚠️ আরেকটা লুকানো সমস্যা: Google user

যে শুধু Google দিয়ে login করেছে, তার `user.password` = `undefined`। আগের code-এ `user.password as string` লিখে TypeScript-কে চুপ করানো হয়েছিল, কিন্তু runtime-এ `bcryptjs.compare("abc", undefined)` → **`Illegal arguments` error → 500**।

এখন password না থাকলে পরিষ্কার 400 message, আর তাকে set-password-এ পাঠাই ([note 30](./30-set-password-for-google-auth.md))। `if (!user.password)` check-এর পর TypeScript নিজেই বোঝে password এখন string, তাই `as string`-ও আর লাগে না।

---

# Part 5: CORS

## Step 10: CORS কী, কেন?

Browser-এর একটা নিরাপত্তার নিয়ম আছে — **Same-Origin Policy**: একটা website-এর JavaScript অন্য **origin**-এর server থেকে data পড়তে পারবে না, যদি না সেই server নিজে অনুমতি দেয়।

**Origin** = protocol + domain + port:

| URL | Origin |
|---|---|
| `http://localhost:5173/tours` | `http://localhost:5173` |
| `http://localhost:5000/api/v1/user` | `http://localhost:5000` ← port আলাদা, তাই **আলাদা origin** |
| `https://ph-tour.vercel.app` | `https://ph-tour.vercel.app` |

তাই আমাদের frontend (5173) থেকে backend (5000)-এ request — browser-এর চোখে অন্য origin। Backend যদি না বলে "5173 থেকে আসা request ঠিক আছে", browser response আটকে দেবে।

**CORS** (Cross-Origin Resource Sharing) হলো backend-এর সেই অনুমতি দেওয়ার ব্যবস্থা — response-এর header-এ জানিয়ে দেয় কোন origin অনুমতি পাবে।

> 💡 CORS শুধু **browser** মানে। Postman বা server থেকে server-এ request-এ CORS-এর কোনো প্রভাব নেই — এজন্যই Postman-এ সব কাজ করলেও frontend-এ গিয়ে হঠাৎ CORS error দেখা যায়।

## Step 11: `app.ts`-এ CORS

এতদিন ছিল `app.use(cors())` ([note 01](./01-project-setup.md)) — মানে **সব origin**-কে অনুমতি, কিন্তু cookie ছাড়া। এখন বদলাই:

```ts
// src/app.ts
import cors from "cors";
import { envVars } from "./app/config/env";

app.use(
	cors({
		origin: envVars.FRONTEND_URL, // শুধু আমাদের frontend
		credentials: true, // cookie আদান-প্রদানের অনুমতি
	}),
);
```

- **Route-গুলোর আগে** রাখতে হবে (`app.use("/api/v1", router)`-এর উপরে), নাহলে route-এর response-এ CORS header যাবে না।
- **`origin`**: শুধু এই URL থেকে আসা request-কে অনুমতি।
- **`credentials: true`**: Cookie (আমাদের access/refresh token) cross-origin request-এ পাঠাতে আর নিতে দেয়।

### ❓ `credentials: true` কেন বাধ্যতামূলক?

> ✏️ **একটু পরিষ্কার করে:** Note-এ লেখা ছিল "cookie-তে security issue-এর জন্য token set হবে না"। আসলে ঘটনাটা এমন:
>
> Browser cross-origin request-এ default-ভাবে **cookie পাঠায়ও না, নেয়ও না**। Cookie আদান-প্রদান করতে হলে **দুই পক্ষেরই** রাজি হতে হয়:
>
> | কে | কী করবে |
> |---|---|
> | Backend | `credentials: true` → response-এ `Access-Control-Allow-Credentials: true` header |
> | Frontend | Request-এ cookie পাঠাতে বলবে: axios-এ `withCredentials: true`, fetch-এ `credentials: "include"` |
>
> একজনও না বললে login-এর পর cookie set হবে না, আর পরের request-এ token যাবে না → user বারবার logged out।
>
> আর `credentials: true` দিলে `origin`-এ `"*"` (সবাই) দেওয়া **যায় না** — browser নিরাপত্তার কারণে মানে না। নির্দিষ্ট URL দিতেই হবে।

## Step 12: URL-এর শেষে Slash নয়

```bash
# .env
FRONTEND_URL=https://ph-tour.vercel.app     ✅
FRONTEND_URL=https://ph-tour.vercel.app/    ❌
```

**কেন?** Browser request-এর সাথে `Origin` header পাঠায়, আর সেখানে **কখনো শেষে slash থাকে না**: `Origin: https://ph-tour.vercel.app`। `cors` package এটাকে আমাদের `origin`-এর সাথে **হুবহু** মেলায়:

```
"https://ph-tour.vercel.app" === "https://ph-tour.vercel.app/"  → false → CORS error ❌
```

একটা slash-এর জন্য পুরো frontend কাজ করবে না, আর error দেখে বোঝাও কঠিন। চাইলে code-এই নিশ্চিন্ত হওয়া যায়:

```ts
origin: envVars.FRONTEND_URL.replace(/\/+$/, ""), // শেষের সব / কেটে দাও
```

> 💡 এটা শুধু CORS না — `FRONTEND_URL` দিয়ে আমরা link-ও বানাই (`${FRONTEND_URL}/reset-password`)। শেষে slash থাকলে link হয় `...app//reset-password`। তাই `.env`-এর সব URL-এ শেষের slash বাদ রাখাই নিয়ম।

## Step 13: Local আর Live — দুটো Frontend একসাথে

Deploy করার পরও local-এ কাজ চালাতে চাইলে array দেওয়া যায়:

```ts
app.use(
	cors({
		origin: ["http://localhost:5173", envVars.FRONTEND_URL],
		credentials: true,
	}),
);
```

## Step 14: ⚠️ Deploy-এর পর Cookie — CORS একা যথেষ্ট না

Note-এ লেখা ছিল "Vercel-এ frontend আর backend আলাদা deploy করলে cookie set হবে না"। CORS-এর `credentials: true` এর একটা অংশ, কিন্তু আরেকটা জিনিস না বদলালে তবুও হবে না — **cookie-র নিজের option**।

এখন `setAuthCookie`-এ আছে ([note 18](./18-refresh-token.md), [note 19](./19-logout.md)):

```ts
{ httpOnly: true, secure: false, sameSite: "lax" }
```

Frontend `ph-tour.vercel.app` আর backend `ph-tour-api.vercel.app` — browser-এর চোখে এরা **আলাদা site** (cross-site)। আর `sameSite: "lax"` মানে: cross-site request-এ cookie **পাঠাবে না**। ফলে CORS ঠিক থাকলেও cookie যাবে না।

Cross-site-এ cookie পাঠাতে লাগে:

| Option | Local (HTTP) | Production (HTTPS, আলাদা domain) |
|---|---|---|
| `secure` | `false` | `true` (বাধ্যতামূলক) |
| `sameSite` | `"lax"` | `"none"` |

আর browser নিয়ম: **`sameSite: "none"` দিলে `secure: true` দিতেই হবে**, নাহলে cookie বাতিল। তাই environment দেখে ঠিক করি:

```ts
// src/app/utils/setCookie.ts
const isProduction = envVars.NODE_ENV === "production";

const cookieOptions = {
	httpOnly: true,
	secure: isProduction,
	sameSite: isProduction ? ("none" as const) : ("lax" as const),
};

// res.cookie("accessToken", token, cookieOptions) — দুটো token-এই একই option
// logout-এর clearCookie-তেও একই option (note 19)
```

> Deploy-এর সময় এটা আবার কাজে লাগবে — তখন বিস্তারিত দেখব। এখন শুধু মনে রাখো: **production-এ cookie লাগলে তিনটা জিনিস একসাথে লাগে** — CORS `credentials: true` + frontend `withCredentials` + cookie `secure: true, sameSite: "none"`।

---

## Test

| # | Request | Expected |
|---|---|---|
| 1 | `PATCH /user/<নিজের id>` body-তে `"password": "x"` | 200, কিন্তু password বদলায়নি (Zod বাদ দিয়েছে) |
| 2 | Admin token দিয়ে `PATCH /user/<super admin-এর id>` | 403 "Admins cannot update a super admin's profile." |
| 3 | Super admin token দিয়ে যেকোনো user | 200 |
| 4 | Update-এর response | `password` field নেই |
| 5 | Admin token দিয়ে `GET /user/<id>` | 200, password ছাড়া |
| 6 | User token দিয়ে `GET /user/<অন্য id>` | 403 (checkAuth) |
| 7 | যেকোনো token দিয়ে `GET /user/me` | 200, নিজের profile (route order ঠিক থাকলে) |
| 8 | `POST /auth/change-password` পুরোনো = নতুন | 400 "must be different" |
| 9 | Google-only user change-password | 400 "use set password" |
| 10 | Frontend (5173) থেকে login → পরের request | Cookie যায়, CORS error নেই |

---

## সারাংশ

| বিষয় | মনে রাখার কথা |
|---|---|
| Update-এ password | Zod থেকে সরালেই body থেকে বাদ; password শুধু change/set/reset দিয়ে |
| `decodedToken.role` vs `isUserExist.role` | কে বদলাচ্ছে vs কাকে বদলানো হচ্ছে |
| Admin → Super Admin | ❌ 403 |
| 401 vs 403 | 401 = চিনি না (token সমস্যা); 403 = চিনি, অনুমতি নেই |
| Response | Update আর get — সবখানে `.select("-password")` |
| `GET /:id` | শুধু admin; `/me`, `/all-users`-এর **পরে** |
| Change password | পুরোনো = নতুন → 400; password না থাকলে set-password-এ পাঠাও |
| CORS | Browser-এর নিয়ম; `origin` নির্দিষ্ট + `credentials: true`; `"*"` চলবে না |
| URL | `.env`-এ শেষে slash না — `Origin`-এর সাথে হুবহু মিলতে হয় |
| Production cookie | CORS + `withCredentials` + `secure: true, sameSite: "none"` |