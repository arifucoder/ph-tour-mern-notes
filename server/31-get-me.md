# 33 — Get Me: নিজের Profile দেখা

## কেন দরকার?

Login করা যেকোনো user নিজের profile দেখতে চাইবে: নাম, email, phone, address, role ইত্যাদি। Frontend-এ "My Profile" page, বা navbar-এ user-এর নাম ও ছবি দেখাতে এই API লাগে।

```
GET /api/v1/user/me
Authorization: <accessToken>
        │
        ▼
checkAuth → token verify → req.user = { userId, email, role }
        │
        ▼
controller → service: User.findById(userId).select("-password")
        │
        ▼
200 + নিজের profile (password ছাড়া)
```

## `/me` কেন, `/:id` কেন না?

| | `GET /user/:id` | `GET /user/me` ✅ |
|---|---|---|
| কার profile | URL-এ যার id দেবে | **Token যার**, শুধু তার |
| অন্যের profile দেখা | id বদলালেই সম্ভব (আলাদা check না দিলে) | অসম্ভব — token বদলানো যায় না |
| Frontend-কে id জানতে হয়? | হ্যাঁ | না, শুধু token পাঠালেই হয় |

`/me`-তে user কে, সেটা আসে **token থেকে**, URL বা body থেকে না। তাই কেউ চাইলেও অন্য কারো profile দেখতে পারবে না। এটাই এই route-এর নিরাপত্তা।

---

## Step 1: Route

```ts
// src/app/modules/user/user.route.ts
router.get("/me", checkAuth(...Object.values(Role)), UserControllers.getMe);
```

- **Private route:** `checkAuth` ছাড়া কেউ ঢুকতে পারবে না।
- **`...Object.values(Role)`**: SUPER_ADMIN, ADMIN, USER, GUIDE — যেকোনো role-এর login করা user নিজের profile দেখতে পারবে।

> ✏️ **সংশোধন:** Note-এ লেখা ছিল "যে যার profile **update** করবে"। এই route শুধু profile **দেখার** (GET)। Update আলাদা route (`PATCH`) দিয়ে হয়।

> ⚠️ **Route order:** `user.route.ts`-এ যদি `GET /:id`-এর মতো কোনো dynamic route থাকে, `/me` অবশ্যই তার **আগে** রাখতে হবে। নাহলে Express "me"-কে একটা id ভেবে `/:id`-এ পাঠিয়ে দেবে → `findById("me")` → CastError ([note 28](./28-get-single-by-slug.md))।
>
> ```ts
> router.get("/me", ...);   // ✅ static আগে
> router.get("/:id", ...);  // dynamic পরে
> ```

---

## Step 2: Controller

```ts
// src/app/modules/user/user.controller.ts
const getMe = catchAsync(async (req: Request, res: Response) => {
	const decodedToken = req.user as JwtPayload;
	const result = await UserServices.getMe(decodedToken.userId as string);

	sendResponse(res, {
		success: true,
		statusCode: httpStatus.OK,
		message: "Your profile retrieved successfully",
		data: result.data,
	});
});
```

- **`req.user`**: `checkAuth` token verify করে ভেতরের data (`userId`, `email`, `role`) এখানে বসিয়ে দেয় ([note 30](./30-booking-payment-sslcommerz.md)-এ বিস্তারিত)।
- **`decodedToken.userId as string`**: `JwtPayload`-এর custom field-এর type TypeScript জানে না।

### Teacher-এর code থেকে যা বদলেছি

| বিষয় | আগে | এখন | কেন |
|---|---|---|---|
| ⚠️ Status | `httpStatus.CREATED` (201) | `httpStatus.OK` (200) | 201 মানে "নতুন কিছু তৈরি হয়েছে"। এখানে শুধু data পড়া হচ্ছে, কিছু তৈরি হয়নি। GET-এ সফল হলে সবসময় 200। |
| `next` | Parameter-এ ছিল | সরানো | ব্যবহার হয় না, ESLint `no-unused-vars` error দেয় |
| Comment করা পুরোনো code | `res.status(...).json(...)` | সরানো | এখন `sendResponse` ব্যবহার হয়, আর message-ও ভুল ছিল ("All Users") |

### Status code মনে রাখার নিয়ম

| Code | কখন |
|---|---|
| `200 OK` | সফলভাবে পড়া, update, delete |
| `201 Created` | নতুন কিছু **তৈরি** হলে (register, create tour, booking) |

---

## Step 3: Service

```ts
// src/app/modules/user/user.service.ts
const getMe = async (userId: string) => {
	const user = await User.findById(userId).select("-password");

	if (!user) {
		throw new AppError(httpStatus.NOT_FOUND, "User not found.");
	}

	return {
		data: user,
	};
};
```

### `.select("-password")` কেন?

`select()` দিয়ে কোন field আসবে তা ঠিক করা যায় ([note 27](./27-query-builder-search-filter-pagination.md)-এর field limiting)। নামের আগে **`-`** দিলে সেই field **বাদ** যায়, বাকি সব আসে।

Password hash করা থাকলেও কখনো response-এ পাঠানো উচিত না। Hash হাতে পেলে কেউ offline-এ বসে অনেক password চেষ্টা করে মেলানোর চেষ্টা করতে পারে।

```js
// .select("-password")-এর পর response
{
  _id: "6650...",
  name: "Arif",
  email: "arif@gmail.com",
  role: "USER",
  phone: "017...",
  address: "Dhaka",
  auths: [{ provider: "google", providerId: "1098..." }],
  // password নেই ✅
}
```

> 💡 Credentials login-এ আমরা `const { password, ...rest } = user.toObject()` দিয়ে password বাদ দিয়েছিলাম ([note 24](./24-passport-local.md))। দুটোই কাজ করে, পার্থক্য হলো:
> - `select("-password")`: DB থেকেই password **আনা হয় না**।
> - Destructure: আগে আনা হয়, পরে JavaScript-এ বাদ দেওয়া হয়। Login-এ এটা লাগে, কারণ সেখানে password **compare করতে হয়**।
>
> শুধু দেখানোর জায়গায় `select("-password")` বেশি ভালো।

> ⚠️ **যোগ করেছি — না পেলে 404:** Teacher-এর code-এ user না পেলে `200` সহ `data: null` যেত। `checkAuth` আগেই DB-তে user আছে কিনা check করে, তাই সাধারণত এমন হবে না — কিন্তু check আর এই query-র মাঝে user delete হয়ে গেলে হতে পারে। তখন "সফল কিন্তু data নেই" না পাঠিয়ে পরিষ্কার 404 পাঠানো ভালো।

---

## Step 4: Test

```
GET /api/v1/user/me
Authorization: <accessToken>
```

| অবস্থা | ফলাফল |
|---|---|
| সঠিক token | `200` + নিজের profile (password ছাড়া) |
| Token নেই | `403`/`401` — checkAuth আটকাবে |
| মেয়াদ শেষ token | checkAuth error |
| User blocked/deleted | checkAuth error |

> খেয়াল করো, URL-এ কোনো id নেই। একই URL-এ ভিন্ন ভিন্ন user-এর token দিলে প্রত্যেকে **শুধু নিজের** profile পাবে।

---

## সারাংশ

| বিষয় | মনে রাখার কথা |
|---|---|
| `/me` | User কে তা token থেকে, URL থেকে না → অন্যের profile দেখা যায় না |
| Route | Private, সব role; dynamic `/:id`-এর **আগে** |
| Status | GET → `200 OK`, `201` শুধু নতুন কিছু তৈরি হলে |
| Password | `.select("-password")` → DB থেকেই আনা হয় না |
| না পেলে | `AppError` 404, `data: null` না |