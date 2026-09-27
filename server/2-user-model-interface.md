# User Module

প্রথমে আমরা user module নিয়ে কাজ করব। এজন্য এই folder বানাব:

```
src/app/modules/user/
├── user.interface.ts
└── user.model.ts
```

---

## Step 1: User Interface বানানো

`user` folder-এ `user.interface.ts` নামে একটা file নেব। এখানে user-এর data দেখতে কেমন হবে, তার TypeScript type লিখব।

`src/app/modules/user/user.interface.ts`

```ts
import type { Types } from "mongoose";

export enum Role {
	SUPER_ADMIN = "SUPER_ADMIN",
	ADMIN = "ADMIN",
	USER = "USER",
	GUIDE = "GUIDE",
}

// Auth providers:
// 1. Credentials (email, password)
// 2. Google authentication
export interface IAuthProvider {
	provider: string; // "google", "credentials"
	providerId: string;
}

export enum IsActive {
	ACTIVE = "ACTIVE",
	INACTIVE = "INACTIVE",
	BLOCKED = "BLOCKED",
}

export interface IUser {
	name: string;
	email: string;
	password?: string;
	phone?: string;
	picture?: string;
	address?: string;
	isDeleted?: boolean;
	isActive?: IsActive;
	isVerified?: boolean;
	role: Role;
	auths: IAuthProvider[];
	bookings?: Types.ObjectId[];
	guides?: Types.ObjectId[];
}
```

### কিছু ব্যাখ্যা

- **`enum`**: নির্দিষ্ট কিছু value-র বাইরে যেন কিছু দেওয়া না যায়, সেজন্য `enum` ব্যবহার করেছি। যেমন `role` শুধু `SUPER_ADMIN`, `ADMIN`, `USER` বা `GUIDE` হতে পারবে।
- **`password?`** optional কারণ Google দিয়ে login করলে user-এর কোনো password থাকে না।
- **`auths`**: একজন user কীভাবে login করেছে (email-password নাকি Google), সেই তথ্য এখানে থাকবে। একজন user একাধিক উপায়ে login করতে পারে, তাই এটা array।
- **`Types.ObjectId[]`**: MongoDB-তে প্রতিটা document-এর একটা `_id` থাকে, যার type হলো `ObjectId`। `bookings`-এ user-এর সব booking-এর `_id` গুলো একটা array-তে রাখা থাকবে।
- **`guides`**-ও একইভাবে কাজ করবে। একজন tourist জীবনে শুধু একজন guide পাবে না, অনেক guide পাবে। যেমন Cox's Bazar-এর জন্য একজন, আবার Rangamati-র জন্য আরেকজন।

---

## Step 2: User Model বানানো

`user` folder-এ `user.model.ts` নামে file নেব।

`src/app/modules/user/user.model.ts`

```ts
import { model, Schema } from "mongoose";
import { IsActive, Role, type IAuthProvider, type IUser } from "./user.interface";

const authProviderSchema = new Schema<IAuthProvider>(
	{
		provider: { type: String, required: true },
		providerId: { type: String, required: true },
	},
	{
		versionKey: false,
		_id: false,
	},
);

const userSchema = new Schema<IUser>(
	{
		name: { type: String, required: true },
		email: { type: String, required: true, unique: true },
		password: { type: String },
		role: {
			type: String,
			enum: Object.values(Role),
			default: Role.USER,
		},
		phone: { type: String },
		picture: { type: String },
		address: { type: String },
		isDeleted: { type: Boolean, default: false },
		isActive: {
			type: String,
			enum: Object.values(IsActive),
			default: IsActive.ACTIVE,
		},
		isVerified: { type: Boolean, default: false },
		auths: [authProviderSchema],
	},
	{
		timestamps: true,
		versionKey: false,
	},
);

export const User = model<IUser>("User", userSchema);
```

### Schema options

- **`timestamps: true`**: প্রতিটা document-এ `createdAt` আর `updatedAt` নিজে থেকেই যোগ হবে।
- **`versionKey: false`**: MongoDB প্রতিটা document-এ `__v` নামে একটা field যোগ করে। এটা বন্ধ করার জন্য `versionKey: false` দিয়েছি।
- **`enum: Object.values(Role)`**: `Role` enum-এর সব value একটা array হিসেবে দেয়, তাই database-ও এর বাইরে কোনো value নেবে না।

### `authProviderSchema` আলাদা কেন?

`auths` field-এর ভিতরে নিজস্ব একটা structure আছে। এটাকে `userSchema`-এর ভিতরেই লিখে ফেলা যেত, কিন্তু আলাদা schema বানালে:

- কোড পরিষ্কার ও পড়তে সহজ হয়।
- দরকার হলে অন্য জায়গাতেও reuse করা যায়।
- আলাদা করে option দেওয়া যায়, যেমন `_id: false`।

এটাকে **embedded (sub) schema** বলে, এটা আলাদা কোনো collection না। Default হিসেবে Mongoose প্রতিটা sub-document-এও একটা `_id` যোগ করে। আমাদের এখানে সেটার দরকার নেই, তাই `_id: false` দিয়েছি।

### `bookings` আর `guides` কোথায়?

এগুলোর referencing এখন করব না। যখন `booking` আর `guide`-এর model বানাব, তখন `ref` দিয়ে referencing করব। Interface-এ এগুলো optional, তাই আপাতত কোনো সমস্যা হবে না।

> **Note:** `unique: true` কোনো validation না, এটা MongoDB-তে একটা unique index বানায়। তাই একই email দিয়ে দুইবার user বানাতে গেলে Mongoose-এর validation error না এসে MongoDB-র **duplicate key error** (code `11000`) আসবে। পরে error handling করার সময় এটা মাথায় রাখতে হবে।