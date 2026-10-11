# 34 — Forgot Password, Reset Password আর Nodemailer দিয়ে Email

এই note-এ যা বানাব:

- **Nodemailer + Gmail** দিয়ে email পাঠানোর ব্যবস্থা (`sendEmail` utility)
- **EJS** দিয়ে সুন্দর HTML email template
- **Forgot Password**: email দিলে reset link যাবে
- **Reset Password**: link থেকে এসে নতুন password বসানো

## পুরো Flow এক নজরে

```
① Frontend: "Forgot password?" → email দিল
        │
        ▼
POST /api/v1/auth/forgot-password   { email }          ← public
        │  user আছে? blocked / deleted / unverified না তো?
        │  ১০ মিনিটের reset token বানাও (আলাদা secret দিয়ে)
        │  email পাঠাও: http://localhost:5173/reset-password?token=<resetToken>
        ▼
② User email খুলে "Reset Password" button-এ click
        │
        ▼
Frontend /reset-password page
        │  URL-এর query থেকে token বের করে
        │  user-এর কাছ থেকে নতুন password নেয়
        ▼
POST /api/v1/auth/reset-password                        ← public, কিন্তু reset token লাগে
Authorization: <resetToken>
Body: { newPassword }
        │  token verify (reset secret দিয়ে, মেয়াদ আছে কিনা)
        │  password hash → save
        ▼
③ "Password reset successfully" → user নতুন password দিয়ে login করে
```

## Folder Structure

```
src/app/
├── config/env.ts                    ← SMTP + JWT_RESET_SECRET যোগ
├── utils/
│   ├── sendEmail.ts                 ← নতুন
│   └── templates/
│       └── forgetPassword.ejs       ← নতুন
└── modules/auth/
    ├── auth.validation.ts           ← forgot + reset schema
    ├── auth.route.ts
    ├── auth.controller.ts
    └── auth.service.ts
```

---

# Part 1: Email পাঠানোর Setup

## Step 1: Package Install

```bash
npm i nodemailer ejs
npm i -D @types/nodemailer @types/ejs
```

### ❓ এখানেও `--legacy-peer-deps` কেন লাগছে?

> ✏️ **একটু সংশোধন:** Note-এ লেখা ছিল "nodemailer-এর জন্য লাগছে, কারণ multer-storage-cloudinary-তে conflict আছে"। কারণটা ঠিক, তবে পুরোটা এমন:
>
> `npm install` যেকোনো নতুন package install করার সময় **পুরো project-এর** সব dependency আবার check করে। `multer-storage-cloudinary`-র peer dependency conflict (cloudinary v1 চায়, আছে v2 — [note 31](./31-file-upload-multer-cloudinary.md)) project-এ থেকেই গেছে। তাই এখন **যেকোনো** package install করতে গেলেই একই `ERESOLVE` error আসবে, nodemailer বা ejs-এর নিজের কোনো দোষ নেই।

**প্রতিবার flag না লিখে সহজ উপায়:** project-এর root-এ একটা `.npmrc` file বানাও:

```ini
# .npmrc
legacy-peer-deps=true
```

এখন থেকে সাধারণ `npm i <package>`-ই কাজ করবে। Deploy server-এও এই setting কাজ করবে।

### কোন package কী করে?

| Package | কাজ |
|---|---|
| **nodemailer** | Email **পাঠানো** — Gmail-এর SMTP server-এর সাথে যোগাযোগ করে email ডেলিভার করে |
| **ejs** | Email-এর **HTML বানানো** — template-এ `name`, `link`-এর মতো dynamic data বসায় |

> ✏️ **সংশোধন:** Note-এ লেখা ছিল "auto-generated email বানানোর কাজ nodemailer-এর"। আসলে email-এর ভেতরের HTML বানায় **EJS**, আর nodemailer শুধু সেটা **পাঠায়**। Multer + Cloudinary-র তুলনাটা ভালো: multer file নিয়ে Cloudinary-তে দেয়, nodemailer email নিয়ে Gmail-কে দেয়, Gmail সেটা ঠিকানায় পৌঁছে দেয়।

**SMTP কী?** Simple Mail Transfer Protocol — email পাঠানোর জন্য internet-এর নিয়ম। Gmail-এর একটা SMTP server আছে (`smtp.gmail.com`)। আমরা নিজেদের Gmail-এর username/password দিয়ে সেই server-এ ঢুকে email পাঠাব।

---

## Step 2: Gmail App Password বানানো

নিজের আসল Gmail password code-এ দেওয়া **নিরাপদ না** (Google অনুমতিও দেয় না)। তার বদলে Google একটা আলাদা **App Password** দেয় — শুধু এই app-এর জন্য, যেকোনো সময় বাতিল করা যায়।

1. **2-Step Verification চালু করো** (বাধ্যতামূলক): Google Account → **Security** → 2-Step Verification।
2. Google Account-এর search box-এ লেখো **"App passwords"**।
   - সরাসরি option চলে এলে → ভালো।
   - না এলে, সাধারণত 2-Step Verification চালু নেই। (Help doc "Sign in with app passwords" → "Create and manage your app passwords" থেকেও যাওয়া যায়।)
3. একটা নাম দাও (যেমন `PH Tour Server`) → **Create**।
4. একটা **১৬ অক্ষরের password** দেখাবে। সাথে সাথে copy করে `.env`-এ রাখো — **এটা আর কখনো দেখাবে না।**

> 💡 Google password-টা ৪টা ভাগে space দিয়ে দেখায় (`abcd efgh ijkl mnop`)। `.env`-এ **space ছাড়া** লেখো (`abcdefghijklmnop`), যাতে login সমস্যা না হয়।

---

## Step 3: Environment Variable

### `.env`

```bash
# Email (Gmail SMTP)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=465
SMTP_USER=yourname@gmail.com
SMTP_PASS=abcdefghijklmnop
SMTP_FROM="PH Tour <yourname@gmail.com>"

# Reset password token — access token-এর secret থেকে আলাদা!
JWT_RESET_SECRET=a_long_random_different_secret
```

| Variable | কী |
|---|---|
| `SMTP_HOST` | Gmail-এর SMTP server: `smtp.gmail.com` |
| `SMTP_PORT` | `465` (SSL) — অথবা `587` (তখন `secure: false`, Step 4 দেখো) |
| `SMTP_USER` | যে Gmail থেকে App Password বানিয়েছ |
| `SMTP_PASS` | সেই ১৬ অক্ষরের App Password |
| `SMTP_FROM` | Email কার নামে যাবে। Gmail-এ এটা **`SMTP_USER`-এর একই email** হতে হয়, শুধু সামনে একটা নাম দেওয়া যায় |
| `JWT_RESET_SECRET` | Reset token-এর জন্য **আলাদা** secret (কেন — Step 9-এ) |

> ⚠️ `SMTP_PASS` আর সব secret শুধু `.env`-এ। কখনো note বা GitHub-এ না।

### `env.ts` — ৩ জায়গায়

```ts
// src/app/config/env.ts

// ① interface-এ
interface EnvConfig {
	// ... আগের সব
	JWT_RESET_SECRET: string;
	EMAIL_SENDER: {
		SMTP_HOST: string;
		SMTP_PORT: string;
		SMTP_USER: string;
		SMTP_PASS: string;
		SMTP_FROM: string;
	};
}

// ② requiredEnvVariables-এ
"JWT_RESET_SECRET",
"SMTP_HOST",
"SMTP_PORT",
"SMTP_USER",
"SMTP_PASS",
"SMTP_FROM",

// ③ return object-এ
JWT_RESET_SECRET: process.env.JWT_RESET_SECRET as string,
EMAIL_SENDER: {
	SMTP_HOST: process.env.SMTP_HOST as string,
	SMTP_PORT: process.env.SMTP_PORT as string,
	SMTP_USER: process.env.SMTP_USER as string,
	SMTP_PASS: process.env.SMTP_PASS as string,
	SMTP_FROM: process.env.SMTP_FROM as string,
},
```

---

## Step 4: `sendEmail` Utility

```ts
// src/app/utils/sendEmail.ts
import path from "node:path";
import { fileURLToPath } from "node:url";
import ejs from "ejs";
import httpStatus from "http-status-codes";
import nodemailer from "nodemailer";
import { envVars } from "../config/env";
import AppError from "../errorHelpers/AppError";

// ESM-এ __dirname নেই, তাই নিজে বানাতে হয়
const __filename = fileURLToPath(import.meta.url);
const __dirname = path.dirname(__filename);

const smtpPort = Number(envVars.EMAIL_SENDER.SMTP_PORT);

// Gmail-এর সাথে connection — একবার বানিয়ে বারবার ব্যবহার
const transporter = nodemailer.createTransport({
	host: envVars.EMAIL_SENDER.SMTP_HOST,
	port: smtpPort,
	secure: smtpPort === 465, // 465 → true, 587 → false
	auth: {
		user: envVars.EMAIL_SENDER.SMTP_USER,
		pass: envVars.EMAIL_SENDER.SMTP_PASS,
	},
});

// নতুন template বানালে এখানে নাম যোগ করবে
type TEmailTemplate = "forgetPassword";

interface SendEmailOptions {
	to: string;
	subject: string;
	templateName: TEmailTemplate;
	templateData?: Record<string, unknown>;
	attachments?: {
		filename: string;
		content: Buffer | string;
		contentType: string;
	}[];
}

export const sendEmail = async ({ to, subject, templateName, templateData = {}, attachments }: SendEmailOptions) => {
	try {
		// ① template-এ data বসিয়ে HTML বানানো
		const templatePath = path.join(__dirname, "templates", `${templateName}.ejs`);
		const html = await ejs.renderFile(templatePath, templateData);

		// ② Gmail দিয়ে পাঠানো
		await transporter.sendMail({
			from: envVars.EMAIL_SENDER.SMTP_FROM,
			to,
			subject,
			html,
			attachments,
		});
	} catch (error) {
		// eslint-disable-next-line no-console
		console.error("Email sending error:", error);
		throw new AppError(httpStatus.INTERNAL_SERVER_ERROR, "Failed to send email. Please try again later.");
	}
};
```

### প্রতিটা অংশ

- **`__dirname` নিজে বানানো কেন?** CommonJS-এ `__dirname` (এই file কোন folder-এ আছে) এমনিই পাওয়া যায়। আমাদের project **ESM** (`import`/`export`), সেখানে এটা নেই। তাই `import.meta.url` (এই file-এর পুরো URL) থেকে folder বের করি।
  > 💡 Node 20.11+ হলে সরাসরি `import.meta.dirname` লিখলেও হয়, ওপরের দুই লাইন লাগে না।
- **`createTransport`**: Gmail-এর সাথে একটা connection। File-এর উপরে একবার বানাই, প্রতিটা email-এ নতুন করে বানাতে হয় না।
- **`secure: smtpPort === 465`**: Port 465 শুরু থেকেই encrypted (SSL), তাই `true`। Port 587 প্রথমে সাধারণ connection খুলে পরে encrypt করে (STARTTLS), তাই `false`। ভুল দিলে connection timeout বা SSL error আসে।
- **`ejs.renderFile`**: template file পড়ে, `<%= %>`-এর জায়গায় data বসিয়ে HTML string বানায়।
- **`attachments`**: চাইলে email-এর সাথে file (যেমন invoice PDF) পাঠানো যায়। পরে কাজে লাগবে।

### Teacher-এর code থেকে যা বদলেছি

| বিষয় | আগে | এখন | কেন |
|---|---|---|---|
| `secure` | সবসময় `true` | `port === 465` | Port 587 দিলে `true` কাজ করে না |
| `templateName` | যেকোনো `string` | `"forgetPassword"` (নির্দিষ্ট) | বানান ভুল করলে TypeScript আগেই ধরবে, autocomplete-ও পাবে |
| `any` | `Record<string, any>`, `error: any` | `unknown` | ESLint error ছাড়া, তাই eslint-disable লাগে না |
| Attachments | একই field-গুলো আবার map করে নতুন object | সরাসরি পাঠানো | একই shape, map অপ্রয়োজনীয় |
| Error | `AppError(500, ...)` | `httpStatus.INTERNAL_SERVER_ERROR` | |
| Debug code | SMTP config log করার comment (pass-এর অংশ সহ) | সরানো | নিচে দেখো ⬇️ |

### 🔍 Email যাচ্ছে না? — Debug করার নিরাপদ উপায়

Note-এ পুরো SMTP config log করার একটা debug code ছিল (password-এর length, প্রথম-শেষ অক্ষর ইত্যাদি)। Secret-এর কোনো অংশ log করা ঠিক না, log অনেক জায়গায় জমা থাকে। তার বদলে nodemailer-এর নিজের check:

```ts
// server চালুর সময় একবার — শুধু development-এ পরীক্ষার জন্য
transporter.verify().then(
	// eslint-disable-next-line no-console
	() => console.log("✅ SMTP connected"),
	// eslint-disable-next-line no-console
	(error) => console.error("❌ SMTP error:", error),
);
```

Common সমস্যা:

| Error | কারণ |
|---|---|
| `Invalid login` / `535` | App Password ভুল, space আছে, বা 2-Step Verification বন্ধ |
| `wrong version number` / SSL error | Port 587-এ `secure: true` |
| `ETIMEDOUT` | Port বা host ভুল, বা network SMTP port আটকায় |

### ⚠️ Build করার পর template পাওয়া যাবে না!

`tsup` শুধু `.ts` file build করে, **`.ejs` file `dist`-এ copy করে না**। ফলে development-এ (`tsx`) কাজ করলেও `npm run build` করে চালালে error: `ENOENT: no such file or directory ... forgetPassword.ejs`।

সমাধান: build script-এ template folder copy করে দাও। যেমন tsup সব code একটা `dist/server.js`-এ bundle করলে (তখন `__dirname` = `dist`):

```json
// package.json
"scripts": {
	"build": "tsup && cp -r src/app/utils/templates dist/templates"
}
```

> তোমার tsup config যেভাবে output দেয়, সেই অনুযায়ী path মেলাতে হবে। Deploy-এর আগে একবার `npm run build && npm start` চালিয়ে email পাঠিয়ে দেখে নিও।

---

## Step 5: EJS Template

`utils`-এর ভেতরে `templates` folder, তার ভেতরে template:

```html
<!-- src/app/utils/templates/forgetPassword.ejs -->
<!DOCTYPE html>
<html>
  <head>
    <meta charset="UTF-8" />
    <title>Forgot Password</title>
    <style>
      body { font-family: Arial, sans-serif; background: #f7f7f7; color: #333; }
      .container { max-width: 500px; margin: 40px auto; background: #fff; padding: 30px; border-radius: 8px; box-shadow: 0 2px 8px rgba(0, 0, 0, 0.08); }
      .btn { display: inline-block; padding: 12px 24px; background: #007bff; color: #fff; text-decoration: none; border-radius: 4px; margin-top: 20px; }
      .footer { margin-top: 30px; font-size: 12px; color: #888; }
    </style>
  </head>
  <body>
    <div class="container">
      <h2>Password Reset Request</h2>
      <p>Hello <%= name %>,</p>
      <p>We received a request to reset your password. Click the button below to set a new password:</p>
      <a class="btn" href="<%= resetUILink %>">Reset Password</a>
      <p>If you did not request a password reset, please ignore this email. This link will expire in 10 minutes.</p>
      <div class="footer">&copy; PH Tour Management. All rights reserved.</div>
    </div>
  </body>
</html>
```

### `<%= %>` কী?

`sendEmail`-এ যে `templateData` পাঠাই, তার value এখানে বসে:

```ts
templateData: { name: "Arif", resetUILink: "http://localhost:5173/reset-password?token=eyJ..." }
```

```
<p>Hello <%= name %>,</p>        →   <p>Hello Arif,</p>
href="<%= resetUILink %>"         →   href="http://localhost:5173/reset-password?token=eyJ..."
```

`<%= %>` value বসানোর আগে HTML-এর বিশেষ অক্ষর (`<`, `>`) **escape** করে দেয়। তাই কেউ নামে `<script>` লিখে রাখলেও email-এ code হিসেবে চলবে না, শুধু লেখা হিসেবে দেখাবে।

---

# Part 2: Forgot Password

## Step 6: Validation

```ts
// src/app/modules/auth/auth.validation.ts
import { z } from "zod";

export const forgotPasswordZodSchema = z.object({
	email: z.email({ error: "Please provide a valid email address" }),
});

export const resetPasswordZodSchema = z.object({
	newPassword: z
		.string({ error: "New password is required" })
		.min(8, { error: "Password must be at least 8 characters long" })
		.regex(/[A-Z]/, { error: "Password must contain at least 1 uppercase letter" })
		.regex(/\d/, { error: "Password must contain at least 1 number" })
		.regex(/[!@#$%^&*]/, { error: "Password must contain at least 1 special character (!@#$%^&*)" }),
});
```

> 💡 Password rule registration আর set-password-এর ([note 32](./32-set-password-for-google-users.md)) সাথে হুবহু এক রাখো — সবচেয়ে ভালো একটা `passwordZodSchema` বানিয়ে সব জায়গায় import করা।

## Step 7: Route

```ts
// src/app/modules/auth/auth.route.ts
router.post("/forgot-password", validateRequest(forgotPasswordZodSchema), AuthControllers.forgotPassword);
```

**অবশ্যই public (`checkAuth` নেই)।** User password ভুলে গেছে বলেই এখানে এসেছে — সে login করতে পারছে না, তার কাছে কোনো token নেই।

## Step 8: Controller

```ts
// src/app/modules/auth/auth.controller.ts
const forgotPassword = catchAsync(async (req: Request, res: Response) => {
	const { email } = req.body as { email: string };

	await AuthServices.forgotPassword(email);

	sendResponse(res, {
		success: true,
		statusCode: httpStatus.OK,
		message: "If an account exists with this email, a password reset link has been sent.",
		data: null,
	});
});
```

Frontend-এর form থেকে email নেব। Message কেন "If an account exists..." — Step 9-এ।

## Step 9: Service

### Reset token কেন আলাদা secret দিয়ে?

> 🚨 **সবচেয়ে বড় সমস্যা:** Teacher reset token বানিয়েছিলেন **`JWT_ACCESS_SECRET`** দিয়ে, আর `/reset-password` route-এ `checkAuth` দিয়ে check করছিলেন। এতে দুই দিক থেকেই বিপদ:
>
> **① Reset token = পুরো access token।** `checkAuth` access secret দিয়েই verify করে। তাই email-এর link-এর token দিয়ে ১০ মিনিট ধরে user-এর হয়ে **সব API** চালানো যেত — booking, profile, এমনকি admin হলে admin-এর সব কাজ। Email forward বা screenshot হলেই account হাতছাড়া।
>
> **② যেকোনো access token দিয়ে password reset।** উল্টো দিক থেকে, login করে পাওয়া সাধারণ access token দিয়েও `/reset-password` call করা যেত — **পুরোনো password ছাড়াই** password বদলে যেত। Change Password-এ পুরোনো password চাওয়ার নিরাপত্তা পুরো বাইপাস! (Note 32-এর set-password bug-এর মতোই।)
>
> **সমাধান:** Reset token বানাব **আলাদা secret (`JWT_RESET_SECRET`)** দিয়ে। তাহলে:
> - Reset token দিয়ে `checkAuth` পার হওয়া যাবে না (secret মেলে না) → অন্য কোনো API-তে কাজ করবে না।
> - Access token দিয়ে reset করা যাবে না (secret মেলে না)।
>
> দুটো token এখন পুরোপুরি আলাদা কাজের।

### ❓ "User does not exist" বলে দেওয়া কেন ঠিক না?

Teacher-এর code email না পেলে বলত `"User does not exist"`। এতে যে কেউ বিভিন্ন email দিয়ে চেষ্টা করে জেনে নিতে পারে **কোন কোন email আমাদের site-এ registered** (একে বলে *user enumeration*)। সেই list দিয়ে পরে phishing বা password guessing করা সহজ হয়।

তাই email না থাকলে চুপচাপ ফিরে যাই, আর response সবসময় একই: "If an account exists with this email, a reset link has been sent." Gmail, GitHub — সবাই এভাবেই করে।

### ❓ `sendEmail`-এ `await` না দিলে site crash করত কেন?

`await` না দিলে:
1. `sendEmail` background-এ চলতে থাকে, service সাথে সাথে শেষ হয়ে যায়, user "email sent" response পেয়ে যায় — email আসলে না গেলেও।
2. পরে email পাঠাতে fail করলে `sendEmail` যে error throw করে, সেটা ধরার কেউ থাকে না (`catchAsync`-এর কাজ ততক্ষণে শেষ)। একে বলে **unhandled promise rejection**, আর Node.js (v15+) এমন হলে পুরো **server process বন্ধ করে দেয়**।

`await` দিলে error service → `catchAsync` → globalErrorHandler-এ ঠিকমতো যায়।

### একই check দুই জায়গায় → helper

User blocked/deleted/unverified কিনা — এই check forgot আর reset দুই জায়গাতেই লাগবে (link-এর ১০ মিনিটের মধ্যে admin user-কে block করে দিতে পারে)। তাই একটা helper:

### Code

```ts
// src/app/modules/auth/auth.service.ts
import bcryptjs from "bcryptjs";
import httpStatus from "http-status-codes";
import jwt, { type JwtPayload } from "jsonwebtoken";
import { envVars } from "../../config/env";
import AppError from "../../errorHelpers/AppError";
import { sendEmail } from "../../utils/sendEmail";
import { type IAuthProvider, IsActive, type IUser } from "../user/user.interface";
import { User } from "../user/user.model";

// password reset করার অনুমতি আছে কিনা (forgot + reset দুই জায়গাতেই)
const assertUserCanResetPassword = (user: IUser) => {
	if (user.isDeleted) {
		throw new AppError(httpStatus.FORBIDDEN, "This account has been deleted.");
	}
	if (user.isActive === IsActive.BLOCKED || user.isActive === IsActive.INACTIVE) {
		throw new AppError(httpStatus.FORBIDDEN, `This account is ${user.isActive.toLowerCase()}.`);
	}
	if (!user.isVerified) {
		throw new AppError(httpStatus.FORBIDDEN, "Please verify your account first.");
	}
};

const forgotPassword = async (email: string) => {
	const user = await User.findOne({ email });

	// email না থাকলে কাউকে জানাব না (user enumeration আটকানো)
	if (!user) return;

	assertUserCanResetPassword(user);

	// শুধু userId — reset-এর জন্য আর কিছু লাগে না
	const resetToken = jwt.sign({ userId: user._id.toString() }, envVars.JWT_RESET_SECRET, {
		expiresIn: "10m",
	});

	const resetUILink = `${envVars.FRONTEND_URL}/reset-password?token=${resetToken}`;

	await sendEmail({
		to: user.email,
		subject: "Reset your PH Tour password",
		templateName: "forgetPassword",
		templateData: {
			name: user.name,
			resetUILink,
		},
	});
};
```

### Teacher-এর code থেকে যা বদলেছি

| বিষয় | আগে | এখন | কেন |
|---|---|---|---|
| 🚨 Secret | `JWT_ACCESS_SECRET` | `JWT_RESET_SECRET` | উপরে বিস্তারিত |
| User না পেলে | 400 "User does not exist" | চুপচাপ return | User enumeration |
| Token payload | `userId`, `email`, `role` | শুধু `userId` | Reset-এর জন্য role লাগে না; token-এ যত কম তথ্য, তত ভালো |
| Link | `?id=...&token=...` | `?token=...` | `id` token-এর ভেতরেই আছে, আলাদা করে লাগে না (Step 12) |
| Status | সব `BAD_REQUEST` | blocked/deleted → `FORBIDDEN` | Request ঠিক আছে, কিন্তু অনুমতি নেই — এটাই 403 |
| Check | শুধু forgot-এ | helper, forgot + reset দুই জায়গায় | ১০ মিনিটে block হয়ে যেতে পারে |

> 📝 Teacher-এর flow comment-এ link-এ `email` ছিল (`?email=...&token=...`), কিন্তু code-এ ছিল `id`। এখন কোনোটাই লাগছে না, শুধু token।

---

# Part 3: Reset Password

## Step 10: Route

```ts
// src/app/modules/auth/auth.route.ts
router.post("/reset-password", validateRequest(resetPasswordZodSchema), AuthControllers.resetPassword);
```

> ✏️ **সংশোধন:** Note-এ লেখা ছিল "reset-password অবশ্যই protected route, কারণ token পাঠাব" — তাই teacher `checkAuth` দিয়েছিলেন। Route-টা **সুরক্ষিত** ঠিকই, কিন্তু `checkAuth` দিয়ে না:
> - `checkAuth` **login-এর access token** চেনে। Reset token আলাদা secret-এ বানানো, তাই `checkAuth` সেটা মানবেই না।
> - আর `checkAuth` রাখলে সাধারণ access token দিয়েও ঢোকা যেত (Step 9-এর সমস্যা ②)।
>
> তাই route থেকে `checkAuth` সরিয়ে, service-এর ভেতরে **reset secret দিয়ে** token verify করি। Token ছাড়া বা ভুল token দিলে service-ই আটকাবে — route তাই কার্যত এখনো protected।

## Step 11: Controller

```ts
// src/app/modules/auth/auth.controller.ts
const resetPassword = catchAsync(async (req: Request, res: Response) => {
	const resetToken = req.headers.authorization; // frontend URL থেকে নিয়ে header-এ পাঠাবে
	const { newPassword } = req.body as { newPassword: string };

	await AuthServices.resetPassword(resetToken, newPassword);

	sendResponse(res, {
		success: true,
		statusCode: httpStatus.OK,
		message: "Password reset successfully. Please log in with your new password.",
		data: null,
	});
});
```

- Token আগের মতোই `Authorization` header-এ আসে (teacher-এর flow অনুযায়ী), শুধু এখন `checkAuth` না, service verify করে।
- ✏️ Message `"Password Changed Successfully"` → reset হয়েছে বোঝাতে "Password reset successfully"।
- ব্যবহার না হওয়া `next` সরিয়েছি (ESLint)।

## Step 12: Service

```ts
// src/app/modules/auth/auth.service.ts
const resetPassword = async (resetToken: string | undefined, newPassword: string) => {
	if (!resetToken) {
		throw new AppError(httpStatus.UNAUTHORIZED, "Reset token is missing.");
	}

	// ① reset secret দিয়ে verify — মেয়াদ শেষ বা ভুল token হলে এখানেই থামবে
	let decoded: JwtPayload;
	try {
		decoded = jwt.verify(resetToken, envVars.JWT_RESET_SECRET) as JwtPayload;
	} catch {
		throw new AppError(httpStatus.UNAUTHORIZED, "Reset link is invalid or has expired. Please request a new one.");
	}

	// ② user এখনো আছে আর অনুমতি আছে কিনা
	const user = await User.findById(decoded.userId);
	if (!user) {
		throw new AppError(httpStatus.NOT_FOUND, "User not found.");
	}
	assertUserCanResetPassword(user);

	// ③ নতুন password hash করে save
	user.password = await bcryptjs.hash(newPassword, Number(envVars.BCRYPT_SALT_ROUND));

	// Google user forgot-password দিয়ে প্রথমবার password পেলে, credentials provider যোগ (note 32-এর মতো)
	const hasCredentialsProvider = user.auths.some((providerObject) => providerObject.provider === "credentials");
	if (!hasCredentialsProvider) {
		const credentialProvider: IAuthProvider = { provider: "credentials", providerId: user.email };
		user.auths = [...user.auths, credentialProvider];
	}

	await user.save();
};
```

### প্রতিটা অংশ

- **`jwt.verify`-কে `try/catch`-এ রাখা কেন?** Token মেয়াদোত্তীর্ণ হলে `jwt.verify` `TokenExpiredError` throw করে। আমাদের globalErrorHandler JWT error আলাদা করে চেনে না ([note 25](./25-mongoose-zod-error-handling.md)), তাই সেটা **500** হয়ে যেত। এখানে ধরে পরিষ্কার 401 message দিই, যাতে frontend বলতে পারে "link expired, আবার চাও"।
- **`decoded.userId`**: কার password বদলাব সেটা **token থেকেই** নিই। Token reset secret দিয়ে sign করা, তাই এর ভেতরের userId কেউ বদলাতে পারে না।
- **`credentials` provider**: শুধু Google দিয়ে আসা user forgot-password দিয়েও password পেতে পারে। তখন set-password-এর মতোই `auths`-এ credentials যোগ করি, যাতে DB-তে সঠিক থাকে সে কীভাবে login করতে পারে।

### ❓ Teacher-এর `payload.id != decodedToken.userId` check কেন সরালাম?

Teacher frontend থেকে body-তে `id` পাঠাতেন (link-এর `?id=` থেকে), তারপর মিলিয়ে দেখতেন token-এর userId-র সাথে। কিন্তু:

- Token-এর ভেতরেই `userId` আছে, আর সেটা বদলানো যায় না। তাই আলাদা `id` থেকে নতুন কোনো তথ্য বা নিরাপত্তা আসে না — শুধু token-এর userId ব্যবহার করলেই হয়।
- বরং সমস্যা ছিল: `!=` (loose comparison, `!==` হওয়া উচিত), message "You can not reset **your** password" (বিভ্রান্তিকর), আর `payload: Record<string, any>` — কোনো validation ছাড়া যেকোনো কিছু।

তাই `id` বাদ, শুধু token। Link-ও ছোট হলো।

### Teacher-এর code থেকে যা বদলেছি

| বিষয় | আগে | এখন |
|---|---|---|
| 🚨 Token verify | `checkAuth` (access secret) | service-এ `JWT_RESET_SECRET` দিয়ে |
| মেয়াদ শেষ token | 500 (JWT error) | 401 "Reset link is invalid or has expired..." |
| `id` check | body-র `id` vs token | সরানো, শুধু token-এর userId |
| User না পেলে | `401` | `404` |
| Blocked/deleted | Reset-এর সময় check ছিল না | `assertUserCanResetPassword` |
| `newPassword` validation | ছিল না (`undefined` → bcrypt 500) | Zod schema |
| Google-only user | auths বদলাত না | credentials যোগ |

---

## Step 13: Frontend কী করবে (বোঝার জন্য)

```
Email-এর link: http://localhost:5173/reset-password?token=eyJhbGci...
```

1. `/reset-password` page খুলবে, URL-এর query থেকে `token` বের করবে।
2. User নতুন password লিখবে (দুবার, মিলিয়ে দেখার জন্য)।
3. Backend-এ পাঠাবে:

```
POST http://localhost:5000/api/v1/auth/reset-password
Authorization: eyJhbGci...
Content-Type: application/json

{ "newPassword": "NewPass@123" }
```

4. সফল হলে login page-এ পাঠাবে।

---

## Step 14: Test (Postman)

| # | Request | Expected |
|---|---|---|
| 1 | `POST /forgot-password` `{ "email": "arif@gmail.com" }` | 200 + inbox-এ email আসবে |
| 2 | `POST /forgot-password` `{ "email": "nobody@gmail.com" }` | 200, একই message, কিন্তু কোনো email যাবে না |
| 3 | `POST /forgot-password` `{ "email": "abc" }` | 400 Zod error |
| 4 | Email-এর link থেকে token নিয়ে `POST /reset-password` (header-এ token) | 200 |
| 5 | একই reset token দিয়ে অন্য API, যেমন `GET /user/me` | ❌ আটকাবে (secret আলাদা) ✅ |
| 6 | Login-এর access token দিয়ে `POST /reset-password` | 401 ✅ |
| 7 | ১০ মিনিট পরে reset token দিয়ে | 401 "Reset link is invalid or has expired..." |
| 8 | নতুন password দিয়ে `POST /auth/login` | 200 ✅ |

> Email না এলে Spam folder দেখো। তারপরও না এলে Step 4-এর debug table।

---

## 💡 আরও নিরাপদ: একবার ব্যবহারযোগ্য Link (Optional)

এখন reset link ১০ মিনিটের মধ্যে **যতবার খুশি** ব্যবহার করা যায়। কেউ পরে সেই email-এর link পেলে (১০ মিনিটের মধ্যে) আবার password বদলে দিতে পারে।

একটা সহজ কৌশল: token sign করার secret-এর সাথে user-এর **বর্তমান password hash** জুড়ে দেওয়া। Password বদলে গেলে hash বদলায় → secret বদলায় → পুরোনো link নিজে থেকেই অচল।

```ts
const getResetSecret = (user: IUser) => `${envVars.JWT_RESET_SECRET}${user.password ?? ""}`;

// forgot: jwt.sign({ userId }, getResetSecret(user), { expiresIn: "10m" })
// reset:  আগে jwt.decode() দিয়ে userId বের → user আনো → jwt.verify(token, getResetSecret(user))
```

এখনকার জন্য দরকার নেই, তবে production-এ ভালো অভ্যাস।

---

## সারাংশ

| বিষয় | মনে রাখার কথা |
|---|---|
| Nodemailer | Email **পাঠায়** (Gmail SMTP দিয়ে); EJS HTML **বানায়** |
| Gmail | 2-Step Verification → App Password (space ছাড়া) |
| Port | 465 → `secure: true`, 587 → `secure: false` |
| `.npmrc` | `legacy-peer-deps=true` — প্রতিবার flag লাগবে না |
| Build | `.ejs` file `dist`-এ copy করতে হবে |
| Forgot route | Public; email না থাকলেও একই response (enumeration আটকানো) |
| 🚨 Reset token | **আলাদা secret** (`JWT_RESET_SECRET`), ১০ মিনিট, শুধু userId |
| Reset route | `checkAuth` না; service-এ reset secret দিয়ে verify |
| `await sendEmail` | না দিলে unhandled rejection → server crash, আর ভুল "success" |
| Expired token | `try/catch` → 401, নাহলে 500 যেত |