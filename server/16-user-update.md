# User Update

## কে কী update করতে পারবে?

| Field | কে update করতে পারবে |
| --- | --- |
| `email` | কেউ না |
| `name`, `phone`, `address`, `password` | সবাই (নিজের) |
| `role`, `isActive`, `isDeleted`, `isVerified` | শুধু `ADMIN` আর `SUPER_ADMIN` |
| কাউকে `SUPER_ADMIN` বানানো | শুধু `SUPER_ADMIN` |

- **Email:** Zod-এর `updateUserZodSchema`-তে `email` রাখিনি, তাই কেউ পাঠালেও Zod সেটা বাদ দিয়ে দেবে। এজন্য কোন user update হবে সেটা email দিয়ে না, **`userId`** দিয়ে খুঁজব।
- **Password:** নতুন password দিলে আবার hash করে তারপর রাখতে হবে।

---

## Step 1: Service

`user.service.ts`

```ts
import bcryptjs from "bcryptjs";
import httpStatus from "http-status-codes";
import type { JwtPayload } from "jsonwebtoken";
import { envVars } from "../../config/env";
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
		throw new AppError(httpStatus.FORBIDDEN, "You are not authorized");
	}

	// ৩. role update
	if (payload.role) {
		if (isNormalUser) {
			throw new AppError(httpStatus.FORBIDDEN, "You are not authorized");
		}

		if (payload.role === Role.SUPER_ADMIN && decodedToken.role === Role.ADMIN) {
			throw new AppError(httpStatus.FORBIDDEN, "You are not authorized");
		}
	}

	// ৪. isActive, isDeleted, isVerified update
	if (
		payload.isActive !== undefined ||
		payload.isDeleted !== undefined ||
		payload.isVerified !== undefined
	) {
		if (isNormalUser) {
			throw new AppError(httpStatus.FORBIDDEN, "You are not authorized");
		}
	}

	// ৫. password থাকলে আবার hash করা
	if (payload.password) {
		payload.password = await bcryptjs.hash(payload.password, Number(envVars.BCRYPT_SALT_ROUND));
	}

	// ৬. update করা
	const newUpdatedUser = await User.findByIdAndUpdate(userId, payload, {
		new: true,
		runValidators: true,
	});

	return newUpdatedUser;
};
```

### ❓ `||` দিয়ে check করলে কি সবগুলো `true` হতে হবে?

না। `||` (OR) মানে **যেকোনো একটা** সত্যি হলেই পুরো condition সত্যি। অর্থাৎ `isActive`, `isDeleted`, `isVerified` এর মধ্যে **যেকোনো একটা** payload-এ থাকলেই ভিতরের check চলবে।

সবগুলো একসাথে সত্যি হওয়া লাগলে `&&` (AND) লিখতে হতো।

### ⚠️ `!== undefined` কেন? (আপনার কোডে একটা লুকানো bug ছিল)

আপনার কোডে ছিল:

```ts
if (payload.isActive || payload.isDeleted || payload.isVerified)
```

সমস্যা হলো `isDeleted` আর `isVerified` হলো `boolean`। কেউ যদি `isDeleted: false` পাঠায়, তাহলে `payload.isDeleted` হবে `false`, আর condition-টা ধরে নেবে "এই field পাঠানো হয়নি"। ফলে check-টা এড়িয়ে যাবে।

মানে একজন soft-deleted সাধারণ user নিজেই `{ "isDeleted": false }` পাঠিয়ে নিজের account ফিরিয়ে আনতে পারত।

`!== undefined` দিলে value `true` হোক বা `false`, **field পাঠানো হলেই** check চলবে।

### Step ২ কেন যোগ করলাম?

এটা না থাকলে একজন সাধারণ `USER` অন্য যেকোনো user-এর `id` দিয়ে তার name, phone, এমনকি **password**-ও বদলে দিতে পারত। তাই সাধারণ user শুধু নিজের `id` (token-এর `userId`) দিয়েই update করতে পারবে। Admin আর super admin সবার profile update করতে পারবে।

### Blocked বা Deleted user-এর check কেন দিইনি?

User আছে কিনা check করার পরে এই condition দিতে পারতাম:

```ts
if (isUserExist.isDeleted || isUserExist.isActive === IsActive.BLOCKED) {
	throw new AppError(httpStatus.FORBIDDEN, "This user cannot be updated");
}
```

কিন্তু এটা দিলে admin-ও কখনো কোনো blocked user-কে আবার active করতে পারবে না, বা deleted user-কে ফিরিয়ে আনতে পারবে না। তাই এটা এখানে দিইনি।

### `findByIdAndUpdate`-এর option

- **`new: true`**: Update হওয়ার **পরের** data return করবে। না দিলে পুরোনো data return করে।
- **`runValidators: true`**: Update-এর সময়ও model-এর নিয়ম (যেমন `enum`) check করবে। Default হিসেবে Mongoose update-এ validation চালায় না।

---

## Step 2: Controller

`user.controller.ts`

```ts
const updateUser = catchAsync(async (req: Request, res: Response, next: NextFunction) => {
	const userId = req.params.id;
	const verifiedToken = req.user;
	const payload = req.body;

	const user = await UserServices.updateUser(userId as string, payload, verifiedToken);

	sendResponse(res, {
		success: true,
		statusCode: httpStatus.OK,
		message: "User updated successfully",
		data: user,
	});
});
```

কোন data কোথা থেকে আসছে:

| Data | কোথা থেকে |
| --- | --- |
| `userId` | URL-এর params থেকে (`req.params.id`) |
| `verifiedToken` | `checkAuth` middleware আগেই token verify করে `req.user`-এ রেখে দিয়েছে |
| `payload` | Request body থেকে (`req.body`) |

> **Token আবার verify করতে হচ্ছে না কেন?** `checkAuth` আগেই token verify করে `req.user`-এ রেখে দিয়েছে, তাই controller-এ শুধু `req.user` নিলেই হবে।

> **Parameter-এর ক্রম:** Service-এ parameter-এর ক্রম হলো `(userId, payload, decodedToken)`। Controller থেকে call করার সময় ঠিক এই ক্রমেই দিতে হবে, নাহলে ভুল data ভুল জায়গায় চলে যাবে।

### ❓ Status code কী হবে?

`httpStatus.OK` (`200`)। `CREATED` (`201`) শুধু নতুন কিছু **তৈরি** হলে ব্যবহার হয়। Update-এ নতুন কিছু তৈরি হচ্ছে না, আগেরটা বদলাচ্ছে, তাই `200`।

---

## Step 3: Route

`user.route.ts`

```ts
router.patch(
	"/:id",
	checkAuth(...Object.values(Role)),
	validateRequest(updateUserZodSchema),
	UserControllers.updateUser,
);
```

- **`/:id`**: `:id` মানে URL-এর এই অংশটা dynamic। যেমন `/api/v1/user/665f1a...` দিলে `req.params.id` হবে `665f1a...`।
- **`PATCH`**: Data-র শুধু কিছু অংশ update করার জন্য `PATCH` ব্যবহার হয়।

### `...Object.values(Role)` কী করছে?

Update সব role-এর user-ই করতে পারবে। তাই প্রতিটা role হাতে না লিখে:

```ts
checkAuth(Role.SUPER_ADMIN, Role.ADMIN, Role.USER, Role.GUIDE);
```

এভাবে লিখেছি:

```ts
checkAuth(...Object.values(Role));
```

- `Object.values(Role)` → `["SUPER_ADMIN", "ADMIN", "USER", "GUIDE"]` (একটা array)
- `...` (**spread operator**) → array-টা ভেঙে আলাদা আলাদা argument হিসেবে পাঠায়

পরে নতুন role যোগ করলে এখানে কিছু বদলাতে হবে না।

### Middleware-এর ক্রম

`checkAuth` আগে, `validateRequest` পরে রেখেছি। আগে দেখব request পাঠানো মানুষটা login করা কিনা। Login করা না থাকলে তার data validate করার কোনো দরকারই নেই।

```
Request → checkAuth → validateRequest → Controller → Service → Database
```

---

## Test করা (Postman)

- **Method:** `PATCH`
- **URL:** `http://localhost:5000/api/v1/user/<userId>`
- **Headers:** `Authorization: <login-এর token>`
- **Body:**

```json
{
	"name": "Arif Hossain",
	"phone": "01712345678"
}
```