# Create User Service আপডেট: Duplicate Email Check আর Auth Provider

## Step 1: Destructure না করে পুরো `payload` পাঠানো

আগে service-এ আমরা `payload` থেকে শুধু `name` আর `email` বের করে নিচ্ছিলাম:

```ts
const createUser = async (payload: Partial<IUser>) => {
	const { name, email } = payload;

	const user = await User.create({
		name,
		email,
	});

	return user;
};
```

এতে সমস্যা হলো, `password`, `phone`, `address` এর মতো অন্য field গুলো database-এ যাচ্ছিল না। এখন আর আলাদা করে destructure করার দরকার নেই, সরাসরি পুরো `payload` পাঠাব:

```ts
const createUser = async (payload: Partial<IUser>) => {
	const user = await User.create(payload);
	return user;
};
```

এতে required আর optional, সব data-ই database-এ যাবে।

### পুরো `payload` পাঠানো কি নিরাপদ?

সাধারণত পুরো `payload` সরাসরি database-এ পাঠানো ঝুঁকিপূর্ণ, কারণ কেউ `"role": "SUPER_ADMIN"` পাঠিয়ে নিজেকে admin বানিয়ে ফেলতে পারে। কিন্তু আমাদের ক্ষেত্রে এটা নিরাপদ, কারণ data দুই জায়গায় check হয়ে আসছে:

- **Zod** (`validateRequest`): Schema-তে নেই এমন field (যেমন `role`) নিজে থেকেই বাদ দিয়ে দেয়।
- **Mongoose schema**: Data-র type আর নিয়ম ঠিক আছে কিনা check করে।

---

## Step 2: Email আগে থেকে আছে কিনা check করা

সরাসরি `payload` পাঠালে কিছু কাজ বাকি থেকে যায়, যেমন:

- একই email দিয়ে আগে কেউ account খুলেছে কিনা check করা।
- Password hash করে তারপর database-এ রাখা (এটা পরে করব)।

তাই `payload` থেকে `email` আলাদা করে নেব, আর বাকি সব data `rest`-এ রাখব:

```ts
const createUser = async (payload: Partial<IUser>) => {
	const { email, ...rest } = payload;

	const isUserExist = await User.findOne({ email });

	if (isUserExist) {
		throw new AppError(httpStatus.BAD_REQUEST, "User already exists");
	}

	const user = await User.create({
		email,
		...rest,
	});

	return user;
};
```

### Rest আর Spread Operator

এখানে `...` দুই জায়গায় দুই রকম কাজ করছে:

| কোড | নাম | কাজ |
| --- | --- | --- |
| `const { email, ...rest } = payload` | **Rest operator** | `email` আলাদা করে নেয়, আর **বাকি সব** field `rest` নামে একটা object-এ জমা করে। |
| `{ email, ...rest }` (`User.create`-এর ভিতরে) | **Spread operator** | `rest` object-এর সব field **ছড়িয়ে দিয়ে** নতুন object-এ বসিয়ে দেয়। |

মানে rest দিয়ে জিনিস **জড়ো** করি, আর spread দিয়ে **ছড়িয়ে** দিই।

---

## Step 3: Auth Provider যোগ করা

### Interface আপডেট

`IAuthProvider`-এর `provider` আগে যেকোনো `string` হতে পারত। এখন শুধু দুটো নির্দিষ্ট value রাখব, যাতে ভুল value দেওয়া না যায়:

`user.interface.ts`

```ts
export interface IAuthProvider {
	provider: "google" | "credentials";
	providerId: string;
}
```

- **`credentials`**: Email আর password দিয়ে registration করলে।
- **`google`**: Google দিয়ে login করলে।

আমাদের `createUser` হলো email-password দিয়ে registration, তাই এখানে provider হবে `"credentials"`, আর `providerId` হিসেবে user-এর email রাখব।

### Service আপডেট

`user.service.ts`

```ts
import httpStatus from "http-status-codes";
import AppError from "../../errorHelpers/AppError";
import type { IAuthProvider, IUser } from "./user.interface";
import { User } from "./user.model";

const createUser = async (payload: Partial<IUser>) => {
	const { email, ...rest } = payload;

	const isUserExist = await User.findOne({ email });

	if (isUserExist) {
		throw new AppError(httpStatus.BAD_REQUEST, "User already exists");
	}

	const authProvider: IAuthProvider = {
		provider: "credentials",
		providerId: email as string,
	};

	const user = await User.create({
		...rest,
		email,
		auths: [authProvider],
	});

	return user;
};
```

### `email as string` কেন?

`payload`-এর type হলো `Partial<IUser>`, আর `Partial` সব field-কে optional করে দেয়। তাই TypeScript-এর চোখে `email`-এর type হলো `string | undefined`।

কিন্তু `providerId`-এর type হলো `string`, সেখানে `undefined` বসানো যায় না। তাই TypeScript error দেয়।

আমরা জানি route-এ Zod আগেই check করে নিয়েছে যে email অবশ্যই আছে। কিন্তু service সেটা জানে না, কারণ Zod আর service আলাদা জায়গায়। তাই `as string` দিয়ে TypeScript-কে বলে দিচ্ছি, "এটা নিশ্চিতভাবে `string`, চিন্তা করো না"।

### `auths: [authProvider]` array কেন?

Interface-এ `auths`-এর type হলো `IAuthProvider[]`, মানে একটা **array**। কারণ একজন user একাধিক উপায়ে login করতে পারে। যেমন প্রথমে email-password দিয়ে account খুলল, পরে Google দিয়েও login করল। তখন `auths`-এ দুটো provider থাকবে:

```ts
auths: [
	{ provider: "credentials", providerId: "arif@example.com" },
	{ provider: "google", providerId: "1234567890" },
];
```

এখন নতুন account খোলার সময় একটাই provider আছে, তাই সেটাকে `[authProvider]` আকারে array-র ভিতরে দিচ্ছি।

### `...rest` আগে কেন?

`User.create()`-এ `...rest` সবার আগে রেখেছি, তারপর `email` আর `auths`। Object-এ একই নামের field দুইবার থাকলে **পরেরটা** আগেরটাকে বদলে দেয়।

- `rest` আসে user-এর পাঠানো data (`payload`) থেকে।
- `auths: [authProvider]` আমরা নিজেরা service-এর কোডে লিখেছি।

তাই user যদি কোনোভাবে নিজে `auths` পাঠিয়েও দেয়, সেটা `rest`-এর ভিতরে থাকবে আর আগে বসবে। পরে service-এর কোডে লেখা `auths` এসে সেটাকে বদলে দেবে। ফলে শেষ পর্যন্ত আমাদের service-এর `auths`-ই database-এ যাবে।