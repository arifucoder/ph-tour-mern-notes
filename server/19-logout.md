# Logout

## Logout মানে কী?

Login করার সময় আমরা browser-এর cookie-তে `accessToken` আর `refreshToken` রেখেছিলাম। এই দুটো token cookie-তে থাকা মানেই user **login অবস্থায়** আছে।

তাই logout করার জন্য শুধু cookie থেকে এই দুটো token **মুছে** দিলেই হবে।

> Logout-এ database-এর কোনো কাজ নেই, শুধু cookie মুছতে হয়। তাই এর জন্য আলাদা service লাগবে না, controller-এই কাজ শেষ।

---

## Step 1: Controller

`auth.controller.ts`

```ts
const logout = catchAsync(async (req: Request, res: Response, next: NextFunction) => {
	res.clearCookie("accessToken", {
		httpOnly: true,
		secure: false,
		sameSite: "lax",
	});

	res.clearCookie("refreshToken", {
		httpOnly: true,
		secure: false,
		sameSite: "lax",
	});

	sendResponse(res, {
		success: true,
		statusCode: httpStatus.OK,
		message: "User logged out successfully",
		data: null,
	});
});

export const AuthControllers = {
	credentialsLogin,
	getNewAccessToken,
	logout,
};
```

### কিছু ব্যাখ্যা

- **`res.clearCookie("name", options)`**: ওই নামের cookie browser থেকে মুছে দেয়।
- **`sameSite: "lax"`**: অন্য website থেকে আসা request-এ cookie কখন যাবে, সেটা ঠিক করে। `"lax"` মানে সাধারণ link-এ click করে আসলে cookie যাবে, কিন্তু অন্য site থেকে গোপনে পাঠানো request-এ (যেমন form submit) যাবে না। এটা নিরাপত্তার জন্য।
- **`data: null`**: Logout-এ পাঠানোর মতো কোনো data নেই, তাই `null`।

### ⚠️ Set আর clear করার option একই রাখা

Cookie মুছতে হলে `clearCookie`-এর option (যেমন `httpOnly`, `secure`, `sameSite`) সাধারণত set করার সময়ের option-এর সাথে **মিলিয়ে** দিতে হয়। নাহলে কিছু browser cookie মুছতে পারে না।

এখন `clearCookie`-তে `sameSite: "lax"` আছে, কিন্তু `setAuthCookie`-তে নেই। তাই `setCookie.ts`-এও যোগ করে দেওয়া ভালো:

```ts
res.cookie("accessToken", tokenInfo.accessToken, {
	httpOnly: true,
	secure: false,
	sameSite: "lax",
});

res.cookie("refreshToken", tokenInfo.refreshToken, {
	httpOnly: true,
	secure: false,
	sameSite: "lax",
});
```

---

## Step 2: Route

`auth.route.ts`

```ts
router.post("/logout", AuthControllers.logout);
```

URL: `POST http://localhost:5000/api/v1/auth/logout`

---

## Test করা

1. Postman-এ login করো। **Cookies** অংশে `accessToken` আর `refreshToken` দেখা যাবে।
2. `/logout`-এ `POST` request দাও।
3. আবার Cookies দেখো, দুটো token মুছে গেছে। ✅
4. এখন `/refresh-token`-এ request দিলে "No refresh token received" error আসবে।

---

> **জেনে রাখো:** Logout শুধু browser থেকে token মুছে দেয়, token-টা নিজে বাতিল হয় না। JWT-এর মেয়াদ শেষ না হওয়া পর্যন্ত সেটা valid থাকে। তাই কেউ যদি আগেই token কপি করে রাখে, সে মেয়াদ শেষ হওয়া পর্যন্ত সেটা ব্যবহার করতে পারবে। এজন্যই access token-এর মেয়াদ ছোট রাখা হয়।