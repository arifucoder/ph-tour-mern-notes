# Change Password

## Change Password আর Reset Password কি একই?

না। Professional developer-রা এই দুটোকে আলাদা জিনিস হিসেবে ধরে:

| | Change Password | Reset Password (Forgot Password) |
| --- | --- | --- |
| **কখন** | User login অবস্থায় আছে | User login করতে পারছে না, password ভুলে গেছে |
| **কী লাগে** | পুরোনো password + নতুন password | Email-এ পাঠানো একটা link/code + নতুন password |
| **Route protected?** | হ্যাঁ, `checkAuth` লাগবে | না, কারণ user তখন logged out |

আমরা এখানে যেটা বানাচ্ছি, সেখানে user login করা আছে আর পুরোনো password দিয়ে নতুন password দিচ্ছে। তাই এটা আসলে **Change Password**।

> Teacher এটাকে `resetPassword` নাম দিয়েছেন। কোড একই, শুধু নামটা `changePassword` রাখলে পরে **Forgot/Reset Password** বানানোর সময় নাম নিয়ে বিভ্রান্তি হবে না।

---

## Step 1: Service

`auth.service.ts`

```ts
const changePassword = async (oldPassword: string, newPassword: string, decodedToken: JwtPayload) => {
	const user = await User.findById(decodedToken.userId);

	if (!user) {
		throw new AppError(httpStatus.NOT_FOUND, "User not found");
	}

	// ১. পুরোনো password মিলছে কিনা
	const isOldPasswordMatch = await bcryptjs.compare(oldPassword, user.password as string);

	if (!isOldPasswordMatch) {
		throw new AppError(httpStatus.UNAUTHORIZED, "Old password does not match");
	}

	// ২. নতুন password hash করে save করা
	user.password = await bcryptjs.hash(newPassword, Number(envVars.BCRYPT_SALT_ROUND));

	await user.save();
};

export const AuthServices = {
	credentialsLogin,
	getNewAccessToken,
	changePassword,
};
```

### কিছু ব্যাখ্যা

- **`decodedToken.userId`**: `checkAuth` token verify করে `req.user`-এ রেখেছিল। সেখান থেকে user-এর id নিয়ে database-এ খুঁজছি। তাই user নিজের password-ই বদলাতে পারবে, অন্য কারো না।
- **`user.save()`**: User-এর password বদলে তারপর database-এ save করছি।
- **Blocked/deleted check এখানে কেন নেই?** এই route `checkAuth` দিয়ে protected, আর `checkAuth` আগেই check করে নিয়েছে user আছে কিনা, blocked/inactive কিনা, deleted কিনা। তাই service-এ এসে নিশ্চিন্তে কাজ করা যায়।

### ⚠️ `await user.save()` কেন জরুরি?

`await` ছাড়া লিখলে (`user.save()`):

- Save শেষ হওয়ার আগেই "Password changed successfully" response চলে যাবে।
- Save করতে গিয়ে কোনো error হলে সেটা কেউ ধরবে না, অথচ user ভাববে password বদলে গেছে।

তাই অবশ্যই `await user.save()` লিখতে হবে।

### `user!` না লিখে `if (!user)` কেন?

`user!` (non-null assertion) দিয়ে TypeScript-কে জোর করে বলা হয় "user অবশ্যই আছে"। কিন্তু আমাদের ESLint-এর `strict` config এটাকে error দেখায়। তাছাড়া `if (!user)` দিয়ে check করলে কোড বেশি নিরাপদ হয়, আর এর পরে TypeScript নিজেই বুঝে যায় user আছে, তাই আর `!` লাগে না।

---

## Step 2: Controller

`auth.controller.ts`

```ts
const changePassword = catchAsync(async (req: Request, res: Response, next: NextFunction) => {
	const { oldPassword, newPassword } = req.body;
	const decodedToken = req.user;

	await AuthServices.changePassword(oldPassword, newPassword, decodedToken);

	sendResponse(res, {
		success: true,
		statusCode: httpStatus.OK,
		message: "Password changed successfully",
		data: null,
	});
});
```

- **`data: null`**: Password কখনো response-এ পাঠানো উচিত না, hashed হলেও না। আর এখানে পাঠানোর মতো অন্য কোনো data-ও নেই, তাই `null`।
- **`decodedToken as JwtPayload` লাগছে না**, কারণ custom type declaration-এ আমরা আগেই `req.user`-এর type `JwtPayload` বলে দিয়েছি।

---

## Step 3: Route

`auth.route.ts`

```ts
router.post("/change-password", checkAuth(...Object.values(Role)), AuthControllers.changePassword);
```

- **`checkAuth`**: শুধু login করা user password বদলাতে পারবে। Logged out অবস্থায় এই route কাজ করবে না।
- **`...Object.values(Role)`**: সব role-এর user নিজের password বদলাতে পারবে।

URL: `POST http://localhost:5000/api/v1/auth/change-password`

---

## Test করা (Postman)

- **Headers:** `Authorization: <login-এর token>`
- **Body:**

```json
{
	"oldPassword": "Arif@1234",
	"newPassword": "Arif@5678"
}
```

তারপর নতুন password দিয়ে login করে দেখো কাজ করছে কিনা। ✅

---

> **Note:** এখন `newPassword`-এর কোনো validation নেই, তাই কেউ `"123"` এর মতো দুর্বল password-ও দিতে পারবে। Register-এর মতো এখানেও Zod schema দিয়ে নতুন password-এর নিয়ম (কমপক্ষে ৮ অক্ষর, বড় হাতের অক্ষর, সংখ্যা, special character) check করা উচিত।

> **Google user:** যে user Google দিয়ে account খুলেছে, তার কোনো password নেই। সে change password করতে গেলে `bcryptjs.compare` error দেবে। পরে Google login যোগ করার সময় এই অবস্থাটা আলাদা করে handle করতে হবে।