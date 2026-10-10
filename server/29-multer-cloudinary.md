# 31 — File Upload: Multer + Cloudinary

এই note-এ যা শিখব:

- **Multer** দিয়ে form-data থেকে image নেওয়া
- **Cloudinary**-তে image রাখা আর URL পাওয়া
- Division-এ **একটা** image (`thumbnail`), Tour-এ **অনেকগুলো** image (`images`)
- API fail করলে upload হওয়া image **মুছে ফেলা**
- Update-এর সময় পুরোনো image মোছা, নতুন যোগ করা, বেছে বেছে delete করা

## পুরো Flow এক নজরে

```
Frontend / Postman
   │  form-data: { file: <image>, data: '{"name":"Dhaka"}' }
   ▼
multerUpload.single("file")      ← multer form-data ভাগ করে
   │   ├─ image → সরাসরি Cloudinary-তে upload → req.file.path = image URL
   │   └─ text  → req.body.data (JSON string)
   ▼
validateRequest(zodSchema)        ← req.body.data → JSON.parse → Zod check
   ▼
controller                        ← { ...req.body, thumbnail: req.file.path }
   ▼
service → MongoDB                 ← DB-তে শুধু URL (string) save হয়
   │
   └─ কোথাও error? → globalErrorHandler → এই request-এর image Cloudinary থেকে মুছে দাও
```

> **মূল ধারণা:** MongoDB-তে image file রাখি না। File থাকে Cloudinary-তে, আর DB-তে থাকে শুধু তার **URL**।

## Folder Structure

```
src/app/
├── config/
│   ├── env.ts                 ← CLOUDINARY variable যোগ
│   ├── cloudinary.config.ts   ← নতুন (config + delete helper)
│   └── multer.config.ts       ← নতুন (multerUpload)
├── middlewares/
│   ├── validateRequest.ts     ← update (req.body.data parse)
│   └── globalErrorHandler.ts  ← update (error হলে image মোছা)
└── modules/
    ├── division/              ← thumbnail upload
    └── tour/                  ← images upload + deleteImages
```

---

# Part 1: Multer আর Cloudinary-র ধারণা

## Step 1: JSON দিয়ে File পাঠানো যায় না কেন?

এতদিন সব data পাঠিয়েছি **JSON** দিয়ে:

```json
{ "name": "Dhaka", "description": "Capital city" }
```

JSON শুধু **text** (string, number, boolean, array, object) বহন করতে পারে। একটা image হলো **binary data** (০ আর ১-এর লম্বা ধারা), তাই সেটা JSON-এ রাখা যায় না।

সমাধান: **`multipart/form-data`** — এমন একটা format, যেখানে একই request-এ text আর file দুটোই আলাদা আলাদা "part" হিসেবে পাঠানো যায়। HTML form-এ file upload এভাবেই হয়।

কিন্তু Express নিজে `multipart/form-data` পড়তে পারে না (`express.json()` শুধু JSON পড়ে)। এই কাজটাই করে **Multer**।

## Step 2: Multer কী করে?

Multer হলো একটা **middleware**, যেটা `multipart/form-data` পড়ে দুই ভাগে ভাগ করে:

```
form-data
   │
   ▼
 Multer
   ├─ file part → req.file   (একটা হলে) / req.files (অনেকগুলো হলে)
   └─ text part → req.body
```

### File কোথায় রাখে? — Storage

Multer-কে বলে দিতে হয় file কোথায় রাখবে। একে বলে **storage engine**:

| Storage | কোথায় রাখে | `req.file`-এ কী থাকে |
|---|---|---|
| `multer({ dest: "uploads/" })` (Disk) | আমাদের project-এর `uploads/` folder-এ | local file path |
| `CloudinaryStorage` ✅ | সরাসরি **Cloudinary**-তে | Cloudinary-র image **URL** |

> ✏️ **সংশোধন:** Note-এ লেখা ছিল "multer আগে project-এর একটা temporary folder-এ রাখে, তারপর সেখান থেকে Cloudinary-তে upload হয়"। এটা **disk storage**-এর ক্ষেত্রে সত্যি। কিন্তু আমরা যে `multer-storage-cloudinary` ব্যবহার করছি, সেটা file-কে **সরাসরি Cloudinary-তে stream** করে দেয় — আমাদের project-এ বা computer-এ কোনো folder-ই তৈরি হয় না। Upload শেষে Cloudinary যে URL দেয়, সেটা `req.file.path`-এ বসিয়ে দেয়।

## Step 3: Package Install

```bash
npm i multer
npm i -D @types/multer
npm i cloudinary
npm i multer-storage-cloudinary --legacy-peer-deps
```

### npm-এ `DT` আর `TS` badge

npm-এ package-এর নামের পাশে ছোট একটা badge থাকে:

| Badge | মানে | কী করতে হবে |
|---|---|---|
| **TS** | Package নিজের ভেতরেই TypeScript type নিয়ে আসে | আলাদা কিছু install লাগবে না (যেমন `cloudinary`) |
| **DT** | Type আছে, তবে আলাদাভাবে **DefinitelyTyped**-এ (`@types/...`) | `npm i -D @types/<name>` (যেমন `@types/multer`) |

> ✏️ "DT মানে type declaration file আছে, TS মানে ওটা TypeScript file" — প্রায় ঠিক। সঠিকভাবে বললে: TS = type **package-এর ভেতরেই** আছে; DT = type **আলাদা `@types` package-এ** আছে।

### ❓ `--legacy-peer-deps` কেন লাগে?

`multer-storage-cloudinary` বানানো হয়েছিল `cloudinary` **v1**-এর জন্য। তার `package.json`-এ লেখা আছে "আমার সাথে cloudinary v1 লাগবে" (একে বলে **peer dependency**)। কিন্তু আমরা install করেছি **v2**। npm তখন conflict দেখে install আটকে দেয় (`ERESOLVE` error)।

`--legacy-peer-deps` বলে: "peer dependency-র এই মিল-অমিল check কোরো না, install করে দাও।" v2-এর API v1-এর সাথে মেলে, তাই বাস্তবে ঠিকই কাজ করে।

> 💡 Package-টা অনেক দিন update হয়নি বলেই এই সমস্যা। ভবিষ্যতে কাজ না করলে বিকল্প হলো `multer.memoryStorage()` দিয়ে file নিয়ে নিজে `cloudinary.uploader.upload_stream()` দিয়ে upload করা।

---

# Part 2: Setup

## Step 4: Cloudinary Account আর Secret

1. https://cloudinary.com-এ sign up করো।
2. Dashboard / **Home → API Keys**-এ পাবে: **Cloud name**, **API key**, **API secret**।
3. Upload করা image দেখতে: **Assets** (Media Library) বা https://console.cloudinary.com/app/product-explorer

### `.env`

```bash
CLOUDINARY_CLOUD_NAME=your_cloud_name
CLOUDINARY_API_KEY=your_api_key
CLOUDINARY_API_SECRET=your_api_secret
```

> 🚨 **খুব জরুরি:** তোমার raw note-এ আসল `CLOUDINARY_API_SECRET` লেখা ছিল। এই note যেহেতু public GitHub repo-তে যায়, এখানে placeholder দিয়েছি।
> - **API secret কখনো note, code বা GitHub-এ রাখবে না।** এটা দিয়ে যে কেউ তোমার account-এ file upload/delete করতে পারে।
> - Secret যেহেতু অন্য কোথাও লেখা হয়ে গেছে (chat, note, হয়তো commit-এও), Cloudinary console থেকে **নতুন API secret generate** করে পুরোনোটা বাতিল করে দাও, আর `.env`-এ নতুনটা বসাও।
> - `.env` যেন অবশ্যই `.gitignore`-এ থাকে।

### `env.ts` — ৩ জায়গায় যোগ

SSL-এর মতোই একটা nested object:

```ts
// src/app/config/env.ts

// ① interface-এ
interface EnvConfig {
	// ... আগের সব
	CLOUDINARY: {
		CLOUDINARY_CLOUD_NAME: string;
		CLOUDINARY_API_KEY: string;
		CLOUDINARY_API_SECRET: string;
	};
}

// ② requiredEnvVariables-এ
const requiredEnvVariables: string[] = [
	// ... আগের সব
	"CLOUDINARY_CLOUD_NAME",
	"CLOUDINARY_API_KEY",
	"CLOUDINARY_API_SECRET",
];

// ③ return object-এ
return {
	// ... আগের সব
	CLOUDINARY: {
		CLOUDINARY_CLOUD_NAME: process.env.CLOUDINARY_CLOUD_NAME as string,
		CLOUDINARY_API_KEY: process.env.CLOUDINARY_API_KEY as string,
		CLOUDINARY_API_SECRET: process.env.CLOUDINARY_API_SECRET as string,
	},
};
```

---

## Step 5: Cloudinary Config + Delete Helper

```ts
// src/app/config/cloudinary.config.ts
import { v2 as cloudinary } from "cloudinary";
import { envVars } from "./env";

cloudinary.config({
	cloud_name: envVars.CLOUDINARY.CLOUDINARY_CLOUD_NAME,
	api_key: envVars.CLOUDINARY.CLOUDINARY_API_KEY,
	api_secret: envVars.CLOUDINARY.CLOUDINARY_API_SECRET,
});

// Image URL থেকে public_id বের করা
// https://res.cloudinary.com/<cloud>/image/upload/v1791525834/a48qzh7s99o-1791525831404-product-1.jpg
//                                                            └──────────── public_id ────────────┘
const getPublicIdFromUrl = (url: string): string | null => {
	const match = /\/upload\/(?:v\d+\/)?(.+)\.[a-z0-9]+$/i.exec(url);
	return match?.[1] ?? null;
};

// একটা image মোছা
export const deleteImageFromCloudinary = async (url: string) => {
	const publicId = getPublicIdFromUrl(url);
	if (!publicId) return;

	await cloudinary.uploader.destroy(publicId);
};

// অনেকগুলো image একসাথে মোছা
// কোনোটা fail করলেও বাকিগুলো মুছবে, আর কখনো error throw করবে না (শুধু log করবে)
export const deleteImagesFromCloudinary = async (urls: string[]) => {
	const results = await Promise.allSettled(urls.map((url) => deleteImageFromCloudinary(url)));

	results.forEach((result, index) => {
		if (result.status === "rejected") {
			// eslint-disable-next-line no-console
			console.error(`Failed to delete image from Cloudinary: ${urls[index]}`, result.reason);
		}
	});
};

export const cloudinaryUpload = cloudinary;
```

### প্রতিটা অংশ

- **`v2 as cloudinary`**: Cloudinary package `v2` নামে export করে, আমরা সেটাকে `cloudinary` নামে ব্যবহার করছি।
- **`cloudinary.config()`**: Cloudinary-কে বলা "আমি কোন account"। এই file যেখানেই import হবে, config তখনই চলবে।
- **`export const cloudinaryUpload = cloudinary`**: শুধু নাম বদলে export করা, যাতে অন্য file-এ দেখেই বোঝা যায় এটা configured cloudinary।
- **`cloudinary.uploader.upload()`**: Cloudinary দিয়ে নিজে upload করার function। আমরা এটা ব্যবহার করছি না, কারণ upload-এর কাজটা `multer-storage-cloudinary` করে দেবে।

### Delete-এর জন্য public_id কেন?

Cloudinary-র [destroy API](https://cloudinary.com/documentation/image_upload_api_reference#destroy) URL দিয়ে না, **public_id** দিয়ে delete করে। DB-তে আমাদের কাছে আছে শুধু URL, তাই regex দিয়ে URL থেকে public_id কেটে বের করি:

```
/upload/v1791525834/a48qzh7s99o-1791525831404-product-1.jpg
        └─ v + সংখ্যা ─┘└──────── (.+) = public_id ────────┘└ .jpg ┘
                       (version, বাদ)                        (extension, বাদ)
```

`match[1]` = regex-এর প্রথম `()` group-এ যা ধরা পড়েছে = public_id।

### Teacher-এর code থেকে যা বদলেছি

| বিষয় | আগে | এখন | কেন |
|---|---|---|---|
| Regex | শুধু `jpg\|jpeg\|png\|gif\|webp` | যেকোনো extension (`[a-z0-9]+`) | `.avif`, `.svg` ইত্যাদিও মুছতে পারবে |
| ⚠️ Error | `throw new AppError(401, "...", error.message)` | কখনো throw না, শুধু log | নিচে দেখো ⬇️ |
| অনেকগুলো মোছা | globalErrorHandler-এ `Promise.all` | `deleteImagesFromCloudinary` (`Promise.allSettled`) | একটা fail হলেও বাকিগুলো মুছবে |
| `console.log` | ছিল | সরানো | ESLint `no-console` |

> ⚠️ **AppError-এ ৩টা ভুল ছিল:**
> 1. **`401`** মানে Unauthorized (login সমস্যা)। Image মুছতে না পারা login সমস্যা না।
> 2. **৩য় parameter `error.message`**: AppError-এর ৩য় parameter হলো **`stack`** (note 07), message না। ফলে error-এর stack trace-এর জায়গায় একটা message বসে যেত।
> 3. **Throw করাই ঠিক না:** এই function globalErrorHandler-এর ভেতর থেকে call হয় (Step 11)। সেখানে error throw করলে error handler **নিজেই crash** করে, আর user আসল error message-টাও পায় না। Cleanup-এর কাজ fail করলে শুধু log করাই যথেষ্ট — user-এর request-কে তার জন্য fail করানো উচিত না।

> 💡 **`Promise.all` vs `Promise.allSettled`:**
> - `Promise.all`: একটা fail করলেই পুরোটা fail (বাকিগুলোর result হারিয়ে যায়)।
> - `Promise.allSettled`: সবগুলো শেষ হওয়া পর্যন্ত অপেক্ষা করে, প্রতিটার আলাদা result দেয় (`fulfilled` বা `rejected`)। কখনো নিজে reject করে না। Cleanup-এর জন্য এটাই ঠিক।

---

## Step 6: Multer Config

```ts
// src/app/config/multer.config.ts
import { randomUUID } from "node:crypto";
import path from "node:path";
import httpStatus from "http-status-codes";
import multer from "multer";
import { CloudinaryStorage } from "multer-storage-cloudinary";
import AppError from "../errorHelpers/AppError";
import { cloudinaryUpload } from "./cloudinary.config";

const storage = new CloudinaryStorage({
	cloudinary: cloudinaryUpload, // configured cloudinary (api key সহ)
	params: {
		public_id: (_req, file) => {
			// "My Special.Image#!@.png" → name: "My Special.Image#!@" → "my-special-image"
			const fileName = path
				.parse(file.originalname)
				.name.toLowerCase()
				.replace(/\s+/g, "-") // space → -
				.replace(/\./g, "-") // . → -
				.replace(/[^a-z0-9-]/g, ""); // অক্ষর, সংখ্যা, - ছাড়া সব বাদ

			// a1b2c3d4-1791525831404-my-special-image
			return `${randomUUID().slice(0, 8)}-${Date.now()}-${fileName}`;
		},
	},
});

export const multerUpload = multer({
	storage,
	limits: { fileSize: 5 * 1024 * 1024 }, // সর্বোচ্চ 5 MB
	fileFilter: (_req, file, cb) => {
		if (file.mimetype.startsWith("image/")) {
			cb(null, true); // image → নাও
		} else {
			cb(new AppError(httpStatus.BAD_REQUEST, "Only image files are allowed.")); // অন্য কিছু → বাদ
		}
	},
});
```

### `CloudinaryStorage`-এ কী দিচ্ছি?

- **`cloudinary`**: Cloudinary-তে upload করতে API key, secret লাগবে। Package নিজে এগুলো জানে না, তাই configured cloudinary-টা দিয়ে দিই।
- **`params.public_id`**: Upload হওয়া file-এর **নাম/পরিচয়**।

### public_id কী?

Cloudinary-র documentation অনুযায়ী: public_id হলো যে **identifier** দিয়ে upload হওয়া file-কে access আর deliver করা হয়। এটা দিয়েই image-এর URL তৈরি হয়, আর URL পেলে যে কেউ image দেখতে পারে (public)।

- না দিলে Cloudinary নিজে random নাম দেয় (বা `use_filename` true থাকলে original নাম)।
- দুটো file-এর একই public_id হলে নতুনটা পুরোনোটাকে **overwrite** করে দেয়। তাই প্রতিটা নাম **unique** হতে হবে।
- নিয়ম: `? & # \ % < > +` থাকা যাবে না, আর **image-এর public_id-তে extension দেওয়া যাবে না**।

`public_id`-এ একটা **callback function** দিই, যেটা প্রতিটা file-এর জন্য আলাদা unique নাম বানায়: `random-time-নাম`।

### ❓ Image URL-এ `.jpg.jpg` কেন আসছিল?

প্রথম version-এ নামের শেষে নিজেরাই extension যোগ করা হচ্ছিল:

```
public_id = "re3o6bmiv6-1791513163895-product-1-jpg.jpg"   ← আমরা .jpg দিলাম
URL       = ".../re3o6bmiv6-1791513163895-product-1-jpg.jpg.jpg"   ← Cloudinary আবার .jpg যোগ করল
```

Cloudinary নিজেই file-এর format দেখে URL-এর শেষে extension বসায়। তাই উপরের নিয়ম: "public_id-তে extension দেবে না"। `path.parse(file.originalname).name` শুধু নামটা দেয়, extension ছাড়া:

```ts
path.parse("product-1.jpg");
// { name: "product-1", ext: ".jpg", ... }
```

এখন URL: `.../a48qzh7s99o-1791525831404-product-1.jpg` ✅

### Teacher-এর code থেকে যা বদলেছি

- **`Math.random().toString(36).substring(2)` → `randomUUID().slice(0, 8)`**: দুটোই কাজ করে। `toString(36)` মানে number-কে **base 36**-এ লেখা (০-৯ আর a-z, মোট ৩৬টা চিহ্ন): `0.2312345121` → `"0.8bsl6gv2e"`, তারপর `substring(2)` দিয়ে শুরুর `"0."` বাদ। কিন্তু `Math.random()` সত্যিকারের random না; Node-এর built-in `randomUUID()` আরও নির্ভরযোগ্য (note 30-এর transaction id-এর মতো)।
- **`fileFilter` যোগ:** আগে যেকোনো file (`.exe`, `.pdf`, video) upload করা যেত। এখন শুধু image।
- **`limits.fileSize` যোগ:** আগে কোনো সীমা ছিল না, কেউ 1 GB file দিয়ে Cloudinary-র জায়গা ভরে ফেলতে পারত।

---

## Step 7: `validateRequest` Update — `req.body.data` Parse

### সমস্যা

Form-data-তে file-এর সাথে বাকি data-কে একটা **text field**-এ JSON string হিসেবে পাঠাই:

```
form-data
├─ file : <product-1.jpg>
└─ data : '{"name": "Test", "description": "..."}'   ← string!
```

Multer এর পর `req.body` হয়:

```js
req.body = { data: '{\r\n "name": "Test", ... }' }   // object-এর ভেতরে string
```

আর আমাদের Zod schema খোঁজে `req.body.name`। কিন্তু `name` আছে `req.body.data`-র ভেতরে, তাও string হয়ে! তাই Zod error দিচ্ছিল।

### সমাধান

Zod-এর আগে `req.body.data` থাকলে সেটাকে `JSON.parse` করে আসল `req.body` বানিয়ে দিই:

```ts
// src/app/middlewares/validateRequest.ts
import type { NextFunction, Request, Response } from "express";
import httpStatus from "http-status-codes";
import type { ZodObject } from "zod";
import AppError from "../errorHelpers/AppError";

export const validateRequest = (zodSchema: ZodObject) => async (req: Request, res: Response, next: NextFunction) => {
	try {
		// form-data হলে আসল data আসে "data" নামের text field-এ, JSON string হিসেবে
		if (typeof req.body?.data === "string") {
			try {
				req.body = JSON.parse(req.body.data);
			} catch {
				throw new AppError(httpStatus.BAD_REQUEST, 'The "data" field must be valid JSON.');
			}
		}

		req.body = await zodSchema.parseAsync(req.body ?? {});
		next();
	} catch (error) {
		next(error);
	}
};
```

```
আগে:  req.body = { data: '{"name":"Test"}' }
পরে:  req.body = { name: "Test" }          ← Zod এখন খুশি
```

### Teacher-এর code থেকে যা বদলেছি

| বিষয় | আগে | এখন | কেন |
|---|---|---|---|
| `req.body.data` | `req.body.data` | `req.body?.data` | Express 5-এ body না থাকলে `req.body` হয় `undefined` → `undefined.data` crash |
| String check | `if (req.body.data)` | `typeof ... === "string"` | কেউ JSON body-তেই `data` নামে object পাঠালে `JSON.parse(object)` ভুল করত |
| ভুল JSON | `SyntaxError` → 500 | `AppError` 400 | Client-এর ভুল, server-এর না |

> 💡 **কেন সব field আলাদা form field হিসেবে না পাঠিয়ে একটা `data` JSON-এ?** Form-data-তে সব value **string** হয়ে যায়। `costFrom: 2800` পাঠালে আসত `"2800"`, `included: [...]` array পাঠানোই কঠিন। JSON string-এর ভেতরে number number-ই থাকে, array array-ই থাকে — parse করলে সব ঠিক type-এ ফিরে আসে, Zod-ও খুশি।

> 💡 এই পরিবর্তনের পরেও আগের মতো সাধারণ JSON body পাঠানো API-গুলো একইভাবে কাজ করবে, কারণ তখন `req.body.data` থাকে না।

### ❓ `express.urlencoded()` কি form-data-র জন্য লাগে?

> ✏️ **সংশোধন:** না। Note-এ লেখা ছিল "form data handle করতে `express.urlencoded()` লাগবে"। কিন্তু দুটো আলাদা format:
>
> | Format | কে পড়ে |
> |---|---|
> | `application/json` | `express.json()` |
> | `application/x-www-form-urlencoded` (শুধু text, `a=1&b=2`) | `express.urlencoded()` |
> | `multipart/form-data` (text + file) | **Multer** |
>
> File upload `multipart/form-data`, তাই এটা পড়ে Multer। তবে `express.urlencoded()` রেখে দাও — SSLCommerz-এর জন্য এটা লাগে ([note 30](./30-booking-payment-sslcommerz.md))।

---

# Part 3: Division — একটা Image (`thumbnail`)

## Step 8: Route-এ `multerUpload.single()`

```ts
// src/app/modules/division/division.route.ts
import { multerUpload } from "../../config/multer.config";

router.post(
	"/create",
	checkAuth(Role.ADMIN, Role.SUPER_ADMIN),
	multerUpload.single("file"), // ← নতুন
	validateRequest(createDivisionSchema),
	DivisionController.createDivision,
);

router.patch(
	"/:id",
	checkAuth(Role.ADMIN, Role.SUPER_ADMIN),
	multerUpload.single("file"), // ← নতুন
	validateRequest(updateDivisionSchema),
	DivisionController.updateDivision,
);
```

### Middleware-এর Order কেন এমন?

```
checkAuth → multerUpload → validateRequest → controller
```

1. **`checkAuth` সবার আগে**: Login না করা কেউ যেন Cloudinary-তে file upload-ই করতে না পারে। Multer আগে থাকলে admin না হলেও image upload হয়ে যেত, তারপর error আসত।
2. **`multerUpload` তারপর `validateRequest`**: Multer form-data না পড়া পর্যন্ত `req.body` খালিই থাকে। তাই Zod-এর আগে Multer লাগবে।

### `"file"` নামটা কোথা থেকে?

`single("file")` মানে: form-data-র **`file`** নামের field থেকে **একটা** file নাও, `req.file`-এ রাখো। Postman/frontend-এ ঠিক এই নামেই key দিতে হবে।

- `single("image")` লিখলে key-ও হবে `image`।
- নাম না মিললে Multer error দেয়: `Unexpected field` (Step 12-এ এটা সুন্দর করে handle করেছি)।

---

## Step 9: Postman দিয়ে Test

1. Body → **form-data** (raw না)
2. দুটো row:

| Key | Type | Value |
|---|---|---|
| `file` | **File** | একটা image বেছে নাও |
| `data` | **Text** | `{"name": "Test", "description": "Explore the vibrant capital city."}` |

> Text field-এ JSON সুন্দর করে format হয়ে দেখায় না, কিন্তু ঠিকই কাজ করে।

Controller-এ সাময়িকভাবে log করে দেখি কী আসে:

```ts
console.log({ file: req.file, body: req.body });
```

```js
{
  file: {
    fieldname: 'file',
    originalname: 'product-1.jpg',
    encoding: '7bit',
    mimetype: 'image/jpeg',
    path: 'https://res.cloudinary.com/<cloud>/image/upload/v1791514250/gwkttnzm1g7-1791514247988-product-1.jpg', // ← URL
    size: 114961,
    filename: 'gwkttnzm1g7-1791514247988-product-1' // ← public_id
  },
  body: {
    name: 'Test',
    description: 'Explore the vibrant capital city.'
  }
}
```

- **`req.file.path`** = Cloudinary-র image **URL** → এটাই DB-তে save করব।
- **`req.file.filename`** = public_id।
- `body` এখন parse হওয়া, Zod দিয়ে পরিষ্কার করা object।

Cloudinary-র **Assets**-এ গেলে upload হওয়া image দেখা যাবে।

---

## Step 10: Division Controller আর Service

### Controller

```ts
// src/app/modules/division/division.controller.ts
import type { IDivision } from "./division.interface";

const createDivision = catchAsync(async (req: Request, res: Response) => {
	const payload: IDivision = {
		...req.body,
		thumbnail: req.file?.path, // image দিলে URL, না দিলে undefined
	};

	const result = await DivisionService.createDivision(payload);
	sendResponse(res, {
		statusCode: httpStatus.CREATED,
		success: true,
		message: "Division created successfully",
		data: result,
	});
});

const updateDivision = catchAsync(async (req: Request, res: Response) => {
	const id = req.params.id as string;
	const payload: Partial<IDivision> = {
		...req.body,
		thumbnail: req.file?.path,
	};

	const result = await DivisionService.updateDivision(id, payload);
	sendResponse(res, {
		statusCode: httpStatus.OK,
		success: true,
		message: "Division updated successfully",
		data: result,
	});
});
```

- **`req.file?.path`**: Image ছাড়াও division বানানো যায় (`thumbnail` optional), তখন `req.file` থাকে না। `?.` না দিলে crash করত।
- Update-এ image না দিলে `thumbnail: undefined` যায়। Mongoose update থেকে `undefined` field নিজেই বাদ দেয়, তাই পুরোনো thumbnail মুছে যায় না।

### Service — Update-এ পুরোনো Thumbnail মোছা

নতুন thumbnail দিলে পুরোনোটা Cloudinary-তে অকারণে জায়গা নিয়ে পড়ে থাকবে। তাই মুছে দিই:

```ts
// src/app/modules/division/division.service.ts
import { deleteImagesFromCloudinary } from "../../config/cloudinary.config";

const updateDivision = async (id: string, payload: Partial<IDivision>) => {
	const existingDivision = await Division.findById(id);
	if (!existingDivision) {
		throw new AppError(httpStatus.NOT_FOUND, "Division not found.");
	}

	if (payload.name) {
		const duplicateDivision = await Division.findOne({ name: payload.name, _id: { $ne: id } });
		if (duplicateDivision) {
			throw new AppError(httpStatus.BAD_REQUEST, "A division with this name already exists.");
		}
	}

	// ① আগে DB update
	const updatedDivision = await Division.findByIdAndUpdate(id, payload, { new: true, runValidators: true });

	// ② DB update সফল হলে তবেই পুরোনো image মোছা
	if (payload.thumbnail && existingDivision.thumbnail) {
		await deleteImagesFromCloudinary([existingDivision.thumbnail]);
	}

	return updatedDivision;
};
```

### ❓ পুরোনো image মোছা DB update-এর **পরে** কেন?

```
❌ আগে মুছলে:
   পুরোনো image মুছলাম → DB update-এ error (যেমন duplicate name)
   → DB-তে এখনো পুরোনো URL, কিন্তু image আর নেই → ভাঙা ছবি 💔

✅ পরে মুছলে:
   DB update → error হলে এখানেই থামবে, পুরোনো image অক্ষত
           → সফল হলে তবেই পুরোনো image মুছি
```

আর **`payload.thumbnail && existingDivision.thumbnail`**: নতুন image এসেছে **এবং** আগে কোনো image ছিল — দুটো সত্যি হলেই মোছার কিছু আছে।

### ❓ Image মোছা কি transaction-এর ভেতরে রাখা যায়?

> ✏️ **সংশোধন:** Note-এ লেখা ছিল "image delete-এ error হলে transaction rollback দিয়ে ঠিক করা যায়"। এটা সম্ভব না। Transaction শুধু **MongoDB-র ভেতরের** কাজ rollback করতে পারে। Cloudinary একটা আলাদা বাইরের service — একবার কোনো image মুছে গেলে MongoDB transaction সেটা ফেরত আনতে পারে না।
>
> তাই সঠিক কৌশল হলো উপরের **order**: আগে DB (যেটা নিশ্চিত হওয়া জরুরি), পরে Cloudinary cleanup। Cleanup fail করলে বড় কোনো ক্ষতি নেই — শুধু একটা অব্যবহৃত image Cloudinary-তে থেকে যায়। এজন্যই `deleteImagesFromCloudinary` error throw করে না, শুধু log করে।

---

## Step 11: API Fail করলে Upload হওয়া Image মোছা

### সমস্যা

Multer middleware **controller-এর আগে** চলে। মানে service-এ পৌঁছানোর আগেই image Cloudinary-তে upload হয়ে গেছে। এখন যদি:

- Zod validation fail করে, বা
- Service-এ "already exists" error আসে, বা
- DB-তে কোনো error হয়

তাহলে DB-তে কিছুই save হয় না, কিন্তু image Cloudinary-তে থেকে যায় — **কোনো কিছুর সাথে যুক্ত না, শুধু জায়গা খায়।**

MongoDB-র সমস্যা transaction দিয়ে rollback করেছিলাম (note 30), কিন্তু Cloudinary MongoDB-র অংশ না। তাই নিজেদের মুছতে হবে।

### সমাধান: globalErrorHandler-এ

যেকোনো error শেষ পর্যন্ত `catchAsync` → `next(err)` → **globalErrorHandler**-এ আসে। তাই সেখানে এক জায়গায় লিখলেই সব API-র জন্য কাজ করবে:

```ts
// src/app/middlewares/globalErrorHandler.ts
import { deleteImagesFromCloudinary } from "../config/cloudinary.config";

export const globalErrorHandler = (err: any, req: Request, res: Response, next: NextFunction) => {
	if (envVars.NODE_ENV === "development") {
		// eslint-disable-next-line no-console
		console.log(err);
	}

	// ── এই request-এ upload হওয়া image Cloudinary থেকে মুছে ফেলা ──
	const uploadedImageUrls: string[] = [];

	if (req.file) {
		uploadedImageUrls.push(req.file.path); // single()
	}
	if (Array.isArray(req.files)) {
		uploadedImageUrls.push(...req.files.map((file) => file.path)); // array()
	}
	if (uploadedImageUrls.length > 0) {
		void deleteImagesFromCloudinary(uploadedImageUrls); // অপেক্ষা না করে background-এ
	}

	// ... বাকি আগের মতো (errorSources, statusCode, message, if-else chain, res.json)
};
```

### কিছু point

- **`req.file`** আসে `single()` থেকে, **`req.files`** আসে `array()` থেকে। দুটোই check করি।
- **`Array.isArray(req.files)`**: Multer-এর type অনুযায়ী `req.files` array **অথবা** object হতে পারে (`fields()` ব্যবহার করলে object হয়)। `Array.isArray` দিয়ে check করলে TypeScript-ও বোঝে এটা array, তাই আলাদা `as Express.Multer.File[]` লাগে না।
- **`void` আর `await` না দেওয়া**: Image মোছার জন্য user-কে error response পেতে অপেক্ষা করানোর দরকার নেই। `void` মানে "Promise-টা চালিয়ে দাও, ফলাফলের জন্য অপেক্ষা করব না" — এতে globalErrorHandler আগের মতো সাধারণ (sync) function থাকে। `deleteImagesFromCloudinary` কখনো throw করে না, তাই এটা নিরাপদ।
- Teacher-এর code-এ `console.log({ file: req.files })` ছিল, সেটা সরিয়েছি।

### Test করার উপায়

Tour service-এর `createTour`-এ create-এর আগে সাময়িকভাবে একটা fake error দাও:

```ts
throw new AppError(httpStatus.BAD_REQUEST, "Fake error for testing");
```

Image সহ request পাঠাও → error response আসবে → Cloudinary Assets-এ দেখো image মুছে গেছে কিনা। Test শেষে fake error অবশ্যই মুছে দিও।

---

## Step 12: Multer Error সুন্দর করা

ভুল field নাম বা বড় file দিলে Multer নিজের **`MulterError`** দেয়, আর আমাদের handler-এ সেটা 500 হিসেবে যেত। কিন্তু এটা client-এর ভুল, তাই 400। globalErrorHandler-এর if-else chain-এ (`AppError`-এর আগে) যোগ করি:

```ts
import multer from "multer";

// ...
else if (err instanceof multer.MulterError) {
	statusCode = 400;
	if (err.code === "LIMIT_FILE_SIZE") {
		message = "File is too large. Maximum size is 5 MB.";
	} else if (err.code === "LIMIT_UNEXPECTED_FILE") {
		message = `Unexpected file field "${err.field ?? ""}". Please check the field name (file / files).`;
	} else {
		message = err.message;
	}
}
```

| ভুল | আগে | এখন |
|---|---|---|
| Key `image` দিলাম, route-এ `single("file")` | 500 "Unexpected field" | 400 `Unexpected file field "image"...` |
| 10 MB image | (limit-ই ছিল না) | 400 "File is too large..." |
| PDF upload | upload হয়ে যেত | 400 "Only image files are allowed." (fileFilter) |

---

# Part 4: Tour — অনেকগুলো Image (`images`)

## Step 13: Route-এ `multerUpload.array()`

```ts
// src/app/modules/tour/tour.route.ts
import { multerUpload } from "../../config/multer.config";

router.post(
	"/create",
	checkAuth(Role.ADMIN, Role.SUPER_ADMIN),
	multerUpload.array("files"), // ← নতুন
	validateRequest(createTourZodSchema),
	TourController.createTour,
);

router.patch(
	"/:id",
	checkAuth(Role.ADMIN, Role.SUPER_ADMIN),
	multerUpload.array("files"), // ← নতুন
	validateRequest(updateTourZodSchema),
	TourController.updateTour,
);
```

| Method | কী নেয় | কোথায় রাখে | Postman key |
|---|---|---|---|
| `single("file")` | একটা file | `req.file` (object) | `file` |
| `array("files")` | অনেকগুলো file | `req.files` (array) | `files` (একই key একাধিকবার) |

Postman-এ multiple image: `files` নামে **কয়েকটা row** বানাও (প্রতিটায় একটা image), অথবা একটা row-তে একসাথে কয়েকটা file বেছে নাও।

> 💡 চাইলে সর্বোচ্চ সংখ্যা দেওয়া যায়: `multerUpload.array("files", 10)` — ১০টার বেশি দিলে `LIMIT_UNEXPECTED_FILE` error।

Log করলে দেখা যায়:

```js
{
  body: { title: 'Dhaka Family Fishing Day Tour', costFrom: 2800, included: [...], ... },
  files: [
    { fieldname: 'files', originalname: 'product-1.jpg', path: 'https://res.cloudinary.com/.../a48qzh7s99o-...-product-1.jpg', ... },
    { fieldname: 'files', originalname: 'product-2.png', path: 'https://res.cloudinary.com/.../dnxc7gwofla-...-product-2.png', ... }
  ]
}
```

খেয়াল করো, `costFrom: 2800` number হিসেবেই এসেছে, `included` array হিসেবেই — কারণ `data` JSON-এর ভেতরে পাঠানো (Step 7)।

---

## Step 14: Tour Controller

```ts
// src/app/modules/tour/tour.controller.ts
import type { ITour, TUpdateTourPayload } from "./tour.interface";

// req.files থেকে শুধু URL-গুলোর array
const getUploadedImageUrls = (req: Request): string[] =>
	Array.isArray(req.files) ? req.files.map((file) => file.path) : [];

const createTour = catchAsync(async (req: Request, res: Response) => {
	const payload: ITour = {
		...req.body,
		images: getUploadedImageUrls(req),
	};

	const result = await TourService.createTour(payload);
	sendResponse(res, {
		statusCode: httpStatus.CREATED,
		success: true,
		message: "Tour created successfully",
		data: result,
	});
});

const updateTour = catchAsync(async (req: Request, res: Response) => {
	const id = req.params.id as string;
	const payload: TUpdateTourPayload = { ...req.body };

	const newImages = getUploadedImageUrls(req);
	if (newImages.length > 0) {
		payload.images = newImages; // শুধু নতুন image এলেই images পাঠাই
	}

	const result = await TourService.updateTour(id, payload);
	sendResponse(res, {
		statusCode: httpStatus.OK,
		success: true,
		message: "Tour updated successfully",
		data: result,
	});
});
```

- **`.map((file) => file.path)`**: File object-এর array থেকে শুধু URL-এর array: `[{ path: "url1", ... }, { path: "url2", ... }]` → `["url1", "url2"]`।
- **`Express.Multer.File[]` type:** Teacher লিখেছিলেন `(req.files as Express.Multer.File[])`। `Array.isArray` দিয়ে check করলে TypeScript নিজেই বুঝে যায়, তাই `as` লাগে না। আর `req.files` না থাকলেও crash করে না।

> ⚠️ **Bug fix — Update-এ সব image মুছে যেত:** Teacher-এর update controller-এ সবসময় `images: req.files.map(...)` ছিল। Image না দিয়ে শুধু title update করলে `req.files` = `[]`, তাই `images: []` গিয়ে DB-র **সব পুরোনো image মুছে দিত**! এখন নতুন image থাকলেই শুধু `images` পাঠাই।

---

## Step 15: Update-এর জন্য `deleteImages` Field

User tour edit করার সময় কিছু পুরোনো image বেছে বাদ দিতে চাইতে পারে। Frontend সেই image-গুলোর URL একটা array-তে পাঠাবে:

```
form-data
├─ files : <new-1.jpg>, <new-2.jpg>          ← নতুন যোগ করতে চাই
└─ data  : '{"title": "...", "deleteImages": ["https://.../old-b.jpg"]}'   ← এগুলো বাদ দিতে চাই
```

### Interface

```ts
// src/app/modules/tour/tour.interface.ts
export interface ITour {
	// ... আগের সব field (deleteImages এখানে না)
}

// Update-এর সময় client যা পাঠাতে পারে
export type TUpdateTourPayload = Partial<ITour> & {
	deleteImages?: string[];
};
```

> ✏️ **একটু পরিবর্তন:** Teacher `deleteImages`-কে সরাসরি `ITour`-এ রেখেছিলেন। কাজ করে, কারণ Mongoose schema-তে না থাকা field চুপচাপ বাদ দেয়। কিন্তু `ITour` হলো **DB-তে tour দেখতে কেমন** তার বর্ণনা — আর `deleteImages` DB-তে কখনো save হয় না, এটা শুধু update-এর একটা **নির্দেশ**। তাই আলাদা type `TUpdateTourPayload`-এ রেখেছি, যাতে `ITour` পরিষ্কার থাকে।

### Validation

```ts
// src/app/modules/tour/tour.validation.ts
export const updateTourZodSchema = createTourZodSchema.partial().extend({
	deleteImages: z.array(z.string()).optional(),
});
```

`.partial()` দিয়ে create-এর সব field optional ([note 26](./26-division-tour-module-and-query-builder.md)), আর `.extend()` দিয়ে শুধু update-এর জন্য `deleteImages` যোগ।

---

## Step 16: Update Service — Image-এর Logic সহজ করে

### আগে বুঝি: Teacher-এর code কী করছিল

```ts
// ধাপ ১: নতুন + পুরোনো জোড়া লাগাও
if (payload.images?.length && existingTour.images?.length) {
	payload.images = [...payload.images, ...existingTour.images];
}

// ধাপ ২: delete থাকলে
if (payload.deleteImages?.length && existingTour.images?.length) {
	// পুরোনোগুলোর মধ্যে যেগুলো delete list-এ নেই
	const restDBImages = existingTour.images.filter((url) => !payload.deleteImages?.includes(url));

	// payload.images (ধাপ ১-এ নতুন+পুরোনো মেশানো) থেকে
	const updatedPayloadImages = (payload.images || [])
		.filter((url) => !payload.deleteImages?.includes(url)) // delete-এরগুলো বাদ
		.filter((url) => !restDBImages.includes(url)); // পুরোনোগুলোও বাদ → শুধু নতুনগুলো থাকে

	payload.images = [...restDBImages, ...updatedPayloadImages];
}
```

**দ্বিতীয় `.filter()` কেন লাগছিল?** ধাপ ১-এ `payload.images`-এ নতুন আর পুরোনো **মিশিয়ে** ফেলা হয়েছিল। তারপর ধাপ ২-এ `restDBImages` (পুরোনো) আবার যোগ করলে পুরোনোগুলো **দুবার** চলে আসত। তাই দ্বিতীয় filter দিয়ে মেশানো list থেকে পুরোনোগুলো আবার বের করে শুধু নতুনগুলো রাখা হয়েছে। মানে ধাপ ১-এ মেশানো, ধাপ ২-এ আবার আলাদা করা — একই কাজ ঘুরিয়ে করা।

### ❓ `map` না `filter` — কেন প্রথমে boolean-এর array আসছিল?

দুটোই array-র প্রতিটা item-এ function চালায়, কিন্তু ফলাফল আলাদা:

```ts
const images = ["A", "B", "C"];
const deleteImages = ["B"];

images.map((url) => !deleteImages.includes(url));
// [true, false, true]   ← প্রতিটা item-এর জায়গায় function-এর return value (boolean) বসে

images.filter((url) => !deleteImages.includes(url));
// ["A", "C"]            ← return true হলে item রাখে, false হলে বাদ দেয়
```

| | `map` | `filter` |
|---|---|---|
| কাজ | প্রতিটা item-কে **বদলায়** | প্রতিটা item **রাখবে নাকি বাদ দেবে** ঠিক করে |
| Result-এর length | সবসময় আগের সমান | সমান বা কম |
| Callback return করে | নতুন value | `true` (রাখো) / `false` (বাদ দাও) |
| কখন ব্যবহার | `files.map(f => f.path)` — object থেকে URL বানাতে | `images.filter(url => ...)` — কিছু বাদ দিতে |

**মনে রাখার সহজ নিয়ম:** "বাদ দিতে চাই" → `filter`। "রূপ বদলাতে চাই" → `map`।

### Teacher-এর code-এ সমস্যা

1. **Image না দিয়ে update করলে সব image মুছে যেত** (Step 14-এর bug): `payload.images = []` কোনো if-এ ঢোকে না, সরাসরি DB-তে `images: []` চলে যায়।
2. **অন্য tour-এর image মুছে ফেলা যেত:** `deleteImages`-এ client যেকোনো URL দিতে পারে। Code check করত না URL-টা **এই tour-এরই** কিনা — সরাসরি Cloudinary থেকে মুছে দিত। কেউ অন্য tour-এর image URL পাঠালে সেই tour-এর ছবি ভেঙে যেত।
3. **Order এলোমেলো:** ধাপ ১-এ নতুন image আগে (`[...new, ...old]`), ধাপ ২-এ পুরোনো আগে (`[...old, ...new]`)।
4. `deleteImages` payload-এ রেখেই DB update-এ পাঠানো হচ্ছিল, `runValidators`-ও ছিল না।

### সহজ নতুন Logic — এক ধাপে

চিন্তাটা খুব সোজা:

```
শেষ image list = (পুরোনো − যেগুলো মুছতে বলেছে) + নতুন
```

```ts
// src/app/modules/tour/tour.service.ts
import { deleteImagesFromCloudinary } from "../../config/cloudinary.config";
import type { ITour, TUpdateTourPayload } from "./tour.interface";

const updateTour = async (id: string, payload: TUpdateTourPayload) => {
	const existingTour = await Tour.findById(id);
	if (!existingTour) {
		throw new AppError(httpStatus.NOT_FOUND, "Tour not found.");
	}

	// payload থেকে image-এর অংশ আলাদা করে নিই, বাকিটা rest-এ
	const { images: newImages = [], deleteImages = [], ...rest } = payload;
	const oldImages = existingTour.images ?? [];

	// ① শুধু এই tour-এরই image মুছতে দেব (অন্যের image URL পাঠালে উপেক্ষা)
	const imagesToDelete = deleteImages.filter((url) => oldImages.includes(url));

	const updateData: Partial<ITour> = { ...rest };

	// ② image-এ কোনো পরিবর্তন থাকলে তবেই images field update করব
	if (newImages.length > 0 || imagesToDelete.length > 0) {
		const keptImages = oldImages.filter((url) => !imagesToDelete.includes(url));
		updateData.images = [...keptImages, ...newImages];
	}

	// ③ আগে DB update
	const updatedTour = await Tour.findByIdAndUpdate(id, updateData, { new: true, runValidators: true });

	// ④ DB সফল হলে তবেই Cloudinary থেকে মোছা
	if (imagesToDelete.length > 0) {
		await deleteImagesFromCloudinary(imagesToDelete);
	}

	return updatedTour;
};
```

### উদাহরণ দিয়ে চালিয়ে দেখি

DB-তে আছে: `[A, B, C]`। User নতুন দিল `[D, E]`, মুছতে বলল `[B]`।

| ধাপ | Code | Result |
|---|---|---|
| শুরু | `oldImages` | `[A, B, C]` |
| ① | `imagesToDelete = [B].filter(url => [A,B,C] এ আছে?)` | `[B]` |
| ② | `keptImages = [A,B,C].filter(url => [B] এ নেই?)` | `[A, C]` |
| ② | `updateData.images = [...keptImages, ...newImages]` | `[A, C, D, E]` |
| ③ | DB update | DB-তে `[A, C, D, E]` |
| ④ | Cloudinary থেকে `B` মোছা | `B` আর নেই |

### সব Case এক নজরে

| নতুন image | deleteImages | DB-তে শেষে | মন্তব্য |
|---|---|---|---|
| — | — | `[A, B, C]` (বদলায় না) | শুধু title ইত্যাদি update; teacher-এর code-এ এখানে সব মুছে যেত ❌ |
| `[D, E]` | — | `[A, B, C, D, E]` | নতুনগুলো শেষে যোগ |
| — | `[B]` | `[A, C]` | শুধু মোছা |
| `[D]` | `[B]` | `[A, C, D]` | একসাথে দুটোই |
| — | `[X]` (অন্য tour-এর) | `[A, B, C]` | উপেক্ষা, X মোছা হয় না ✅ |

### কেন এই order (③ তারপর ④)?

Division-এর মতোই (Step 10): Cloudinary থেকে আগে মুছে দিলে আর DB update fail করলে, DB-তে URL থাকবে কিন্তু image থাকবে না। তাই **আগে DB, পরে Cloudinary**।

আর DB update fail করলে নতুন upload হওয়া `[D, E]`? সেগুলো globalErrorHandler নিজেই মুছে দেবে (Step 11)। ✅

### Destructuring-টা বুঝি

```ts
const { images: newImages = [], deleteImages = [], ...rest } = payload;
```

- **`images: newImages`**: `payload.images`-কে নতুন নাম `newImages` দিলাম (পড়তে সুবিধা)।
- **`= []`**: না থাকলে (`undefined`) default খালি array। তাই পরে `newImages.length` লিখতে `?.` লাগে না।
- **`...rest`**: বাকি সব field (title, description...)। `deleteImages` আর `images` এর মধ্যে নেই, তাই DB-তে ভুল করে যায় না।

---

## Step 17 (Bonus): Delete করলে Image-ও মোছা

Tour বা Division delete করলে তাদের image Cloudinary-তে থেকে যায়। একই helper দিয়ে মুছে দেওয়া যায়:

```ts
// tour.service.ts
const deleteTour = async (id: string) => {
	const deletedTour = await Tour.findByIdAndDelete(id);
	if (!deletedTour) {
		throw new AppError(httpStatus.NOT_FOUND, "Tour not found.");
	}

	await deleteImagesFromCloudinary(deletedTour.images ?? []);
	return deletedTour;
};

// division.service.ts
const deleteDivision = async (id: string) => {
	const deletedDivision = await Division.findByIdAndDelete(id);
	if (!deletedDivision) {
		throw new AppError(httpStatus.NOT_FOUND, "Division not found.");
	}

	if (deletedDivision.thumbnail) {
		await deleteImagesFromCloudinary([deletedDivision.thumbnail]);
	}
	return null;
};
```

`findByIdAndDelete` মুছে ফেলা document-টাই return করে, তাই আলাদা `findById` লাগে না।

---

## সারাংশ

| বিষয় | মনে রাখার কথা |
|---|---|
| File পাঠানো | JSON-এ না, `multipart/form-data`-তে; পড়ে Multer |
| Storage | `CloudinaryStorage` → সরাসরি Cloudinary, local folder নেই; `req.file.path` = URL |
| DB-তে | শুধু URL (string) |
| public_id | Unique নাম; extension দেবে না (নাহলে `.jpg.jpg`) |
| Secret | কখনো note/GitHub-এ না; ফাঁস হলে regenerate |
| Middleware order | `checkAuth` → `multerUpload` → `validateRequest` → controller |
| `single` / `array` | `req.file` (`file`) / `req.files` (`files`); key নাম হুবহু মিলতে হবে |
| Data | `data` text field-এ JSON → `validateRequest`-এ `JSON.parse` |
| Error হলে | globalErrorHandler এই request-এর upload মুছে দেয় |
| পুরোনো image মোছা | আগে DB update, পরে Cloudinary; Cloudinary rollback হয় না |
| Tour update | `(পুরোনো − deleteImages) + নতুন`; image না দিলে `images` পাঠাই না |
| Security | `deleteImages` শুধু এই tour-এর image; fileFilter + fileSize limit |
| `map` vs `filter` | রূপ বদলাতে `map`, বাদ দিতে `filter` |