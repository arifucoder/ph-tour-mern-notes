# Super Admin Seed করা

## কেন দরকার?

আমাদের `/register` route দিয়ে যে-ই account খুলুক, তার role হবে সাধারণ `USER`। কারণ Zod `role` field বাদ দিয়ে দেয়, আর model-এ default role হলো `USER`।

তাহলে প্রশ্ন হলো, প্রথম admin আসবে কোথা থেকে? Admin না থাকলে `/all-users` এর মতো admin-only route কেউ কখনো ব্যবহার করতে পারবে না।

তাই server চালু হওয়ার সাথে সাথে database-এ একজন **Super Admin** নিজে থেকেই তৈরি হয়ে যাবে। আগে থেকে থাকলে আর নতুন করে তৈরি হবে না। Database-এ এভাবে শুরুর data বসিয়ে দেওয়াকে **seeding** বলে।

---

## Step 1: `.env`-এ তথ্য রাখা

Super admin-এর email আর password কোডে না লিখে `.env`-এ রাখব:

```env
SUPER_ADMIN_EMAIL=super@admin.com
SUPER_ADMIN_PASSWORD=Super@1234
BCRYPT_SALT_ROUND=10
```

`src/app/config/env.ts`-এ `EnvConfig` interface, `requiredEnvVariables` array আর return object, তিন জায়গাতেই এগুলো যোগ করতে হবে:

```ts
SUPER_ADMIN_EMAIL: process.env.SUPER_ADMIN_EMAIL as string,
SUPER_ADMIN_PASSWORD: process.env.SUPER_ADMIN_PASSWORD as string,
BCRYPT_SALT_ROUND: process.env.BCRYPT_SALT_ROUND as string,
```

> **`.env` আপডেট করার পর server restart করতে হবে।**

---

## Step 2: `seedSuperAdmin` function বানানো

`src/app/utils/seedSuperAdmin.ts`

```ts
/* eslint-disable no-console */
import bcryptjs from "bcryptjs";
import { envVars } from "../config/env";
import { Role, type IAuthProvider, type IUser } from "../modules/user/user.interface";
import { User } from "../modules/user/user.model";

export const seedSuperAdmin = async () => {
	try {
		// ১. super admin আগে থেকে আছে কিনা
		const isSuperAdminExist = await User.findOne({ email: envVars.SUPER_ADMIN_EMAIL });

		if (isSuperAdminExist) {
			console.log("Super Admin already exists!");
			return;
		}

		console.log("Trying to create Super Admin...");

		// ২. password hash করা
		const hashedPassword = await bcryptjs.hash(
			envVars.SUPER_ADMIN_PASSWORD,
			Number(envVars.BCRYPT_SALT_ROUND),
		);

		const authProvider: IAuthProvider = {
			provider: "credentials",
			providerId: envVars.SUPER_ADMIN_EMAIL,
		};

		// ৩. super admin-এর data
		const payload: IUser = {
			name: "Super Admin",
			role: Role.SUPER_ADMIN,
			email: envVars.SUPER_ADMIN_EMAIL,
			password: hashedPassword,
			isVerified: true,
			auths: [authProvider],
		};

		// ৪. database-এ তৈরি করা
		const superAdmin = await User.create(payload);
		console.log("Super Admin created successfully!\n");
		console.log(superAdmin);
	} catch (error) {
		console.log(error);
	}
};
```

### কিছু ব্যাখ্যা

- **`Role` সাধারণ import, বাকি দুটো `type` import কেন?** `Role` একটা enum, যেটা কোড চলার সময়ও ব্যবহার হয় (`Role.SUPER_ADMIN`)। কিন্তু `IAuthProvider` আর `IUser` শুধু type। আমাদের `tsconfig`-এ `verbatimModuleSyntax: true` আছে, তাই শুধু-type import-এ `type` লিখতেই হবে।
- **`Number(envVars.BCRYPT_SALT_ROUND)`**: `.env`-এর সব value `string` হিসেবে আসে, কিন্তু `bcryptjs.hash()`-এ salt round দিতে হয় `number`। তাই `Number()` দিয়ে বদলে নিচ্ছি।
- **`isVerified: true`**: Super admin-কে আলাদা করে verify করার দরকার নেই।

> **Tip:** Salt round যেহেতু এখন `.env`-এ আছে, তাই `user.service.ts`-এর `createUser`-এও `10` না লিখে `Number(envVars.BCRYPT_SALT_ROUND)` ব্যবহার করা ভালো। এতে পুরো project-এ একই মান থাকবে।

---

## Step 3: Server চালু হওয়ার পর seed করা

`server.ts`-এ আগের `startServer();` লাইনটা সরিয়ে তার জায়গায় এটা লিখব:

```ts
import { seedSuperAdmin } from "./app/utils/seedSuperAdmin";

(async () => {
	await startServer();
	await seedSuperAdmin();
})();
```

> আগের `startServer();` লাইন না সরালে server দুইবার চালু হওয়ার চেষ্টা করবে।

### Order কেন জরুরি?

`seedSuperAdmin` database-এ কাজ করে। আর database-এর সাথে connection হয় `startServer`-এর ভিতরে (`mongoose.connect()`)। তাই আগে `startServer` পুরোপুরি শেষ হতে হবে, তারপর `seedSuperAdmin` চলবে। `await` দিয়ে এই ক্রমটা নিশ্চিত করা হয়েছে।

```
startServer() → DB connect ✅ → Server চালু ✅ → seedSuperAdmin() → Super Admin তৈরি ✅
```

### `(async () => { ... })()` এটা কী?

এটাকে বলে **IIFE (Immediately Invoked Function Expression)**, মানে এমন function যেটা বানানোর সাথে সাথেই নিজে নিজে চলে যায়।

```ts
(async () => {
	// কাজ
})();
//  ↑ শেষের এই () দিয়ে function-টা সাথে সাথে call হয়ে যায়
```

`await` শুধু `async` function-এর ভিতরে ব্যবহার করা যায়। তাই একটা `async` IIFE বানিয়ে তার ভিতরে দুটো function পরপর `await` দিয়ে চালাচ্ছি।

---

## Test করা

Server চালু করলে terminal-এ প্রথমবার দেখাবে:

```
Connected to DB
Server is listening on port 5000
Trying to create Super Admin...
Super Admin created successfully!
```

আবার restart করলে দেখাবে:

```
Super Admin already exists!
```

এখন `.env`-এর `SUPER_ADMIN_EMAIL` আর `SUPER_ADMIN_PASSWORD` দিয়ে login করে যে token পাবে, সেটা দিয়ে `/all-users` route ব্যবহার করা যাবে।