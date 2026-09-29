# Password Hashing

## Hashing কেন?

User যে password দেয়, সেটা কখনো সরাসরি (plain text) database-এ রাখা যাবে না। কারণ database কোনোভাবে ফাঁস হয়ে গেলে সবার password সবাই দেখে ফেলবে।

তাই password-কে **hash** করে রাখব। Hash হলো password থেকে বানানো একটা এলোমেলো লেখা, যেমন:

```
Arif@1234  →  $2b$10$N9qo8uLOickgx2ZMRZoMye...
```

Hash **একমুখী (one-way)**। মানে hash থেকে আর কখনো আসল password ফিরিয়ে আনা যায় না।

---

## Step 1: `bcryptjs` install করা

Hashing-এর জন্য [`bcryptjs`](https://www.npmjs.com/package/bcryptjs) package ব্যবহার করব:

```bash
npm install bcryptjs
```

> নতুন version-এ TypeScript support আগে থেকেই আছে, তাই `@types/bcryptjs` আলাদা করে install করতে হবে না।

---

## Step 2: Validation আগে, কাজ পরে

Programming-এর একটা ভালো নিয়ম হলো, **সব check (validation) function-এর উপরের দিকে রাখা**। কোনো check fail করলে সাথে সাথে error দিয়ে থেমে যাবে। আর সব check পার হলে বুঝব সব ঠিক আছে, তারপর আসল কাজ (data create) করব।

তাই আগে user আছে কিনা check করব, তারপর password hash করব। User আগে থেকে থাকলে শুধু শুধু hashing করার দরকার নেই।

---

## Step 3: Service-এ password hash করা

`payload` থেকে `password`-ও আলাদা করে নেব, তারপর hash করব।

```ts
const hashedPassword = await bcrypt.hash(password as string, 10);
```

### `10` কী? (Salt Rounds)

এখানে দুটো জিনিস বুঝতে হবে:

- **Salt**: Hash করার আগে bcrypt password-এর সাথে একটা random লেখা যোগ করে দেয়, একেই salt বলে। এতে দুইজন user-এর password একই হলেও তাদের hash আলাদা হয়। ফলে hash দেখে কেউ বুঝতে পারে না কার password কী।
- **Salt Rounds (`10`)**: Hashing-এর কাজটা কতটা কঠিন হবে, সেটা ঠিক করে। Round যত বেশি, hash বানাতে তত বেশি সময় লাগে, আর hacker-এর পক্ষে অনুমান করে password বের করাও তত কঠিন। `10` একটা ভালো মান।

> **খেয়াল রাখো:** Hashing আর encryption এক জিনিস না। Encryption উল্টো করে আগের লেখা ফিরিয়ে আনা যায়, hashing-এ যায় না।

### `await` ভুলে গেলে কী হয়?

`bcrypt.hash()` একটা Promise return করে। `await` না দিলে hash না পেয়ে একটা Promise পাওয়া যায়:

```ts
const hashedPassword = bcrypt.hash(password as string, 10);
console.log(hashedPassword); // Promise { <pending> }
```

তাই অবশ্যই `await` দিতে হবে।

### Test করা

Hash ঠিকমতো হচ্ছে কিনা দেখতে, আপাতত user create না করে শুধু terminal-এ hash দেখতে পারি:

```ts
const hashedPassword = await bcrypt.hash(password as string, 10);
console.log(hashedPassword);

return {};
```

---

## Step 4: Final Code

`src/app/modules/user/user.service.ts`

```ts
import bcrypt from "bcryptjs";
import httpStatus from "http-status-codes";
import AppError from "../../errorHelpers/AppError";
import type { IAuthProvider, IUser } from "./user.interface";
import { User } from "./user.model";

const createUser = async (payload: Partial<IUser>) => {
	const { email, password, ...rest } = payload;

	// ১. আগে check
	const isUserExist = await User.findOne({ email });

	if (isUserExist) {
		throw new AppError(httpStatus.BAD_REQUEST, "User already exists");
	}

	// ২. তারপর কাজ
	const hashedPassword = await bcrypt.hash(password as string, 10);

	const authProvider: IAuthProvider = {
		provider: "credentials",
		providerId: email as string,
	};

	const user = await User.create({
		...rest,
		email,
		password: hashedPassword,
		auths: [authProvider],
	});

	return user;
};
```

> `...rest` আগের note-এর মতোই সবার আগে রেখেছি, যাতে user-এর পাঠানো data কখনো আমাদের hashed `password` বা `auths`-কে বদলে দিতে না পারে।

---

## পরে Password মিলাব কীভাবে?

Hash থেকে আসল password ফিরিয়ে আনা যায় না, কিন্তু **মিলিয়ে দেখা** যায়। এজন্য bcryptjs-এ `compare` function আছে। এটা plain password আর hashed password নেয়, আর `true` বা `false` return করে:

```ts
const isPasswordMatched = await bcrypt.compare("Arif@1234", user.password);
// মিললে true, না মিললে false
```

Login-এর সময় আমরা এটাই ব্যবহার করব।

---

> **নিরাপত্তা note:** এখন `createUser`-এর response-এ hashed password-ও client-এর কাছে চলে যাচ্ছে। Hash হলেও password কখনো response-এ পাঠানো উচিত না। পরে এটা response থেকে বাদ দেওয়ার ব্যবস্থা করতে হবে।