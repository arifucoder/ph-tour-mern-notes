# Passport.js দিয়ে Google OAuth Login

## Passport.js কী?

Google বা Facebook দিয়ে login করাতে হলে ওদের platform-এ আমাদের একটা app বানাতে হয়, তারপর ওদের দেওয়া নিয়ম মেনে project-এ social login চালু করতে হয়। এই কাজটা বেশ জটিল।

[**Passport.js**](https://www.passportjs.org/) হলো Node.js-এর একটা **authentication middleware**। এটা আমাদের আর Google/Facebook-এর মাঝখানে একটা **wrapper** হিসেবে কাজ করে, আর পুরো কাজটা অনেক সহজ করে দেয়।

Passport-এ প্রতিটা login পদ্ধতিকে বলে **Strategy**। আমরা দুটো ব্যবহার করব:

1. [**passport-google-oauth20**](https://www.passportjs.org/packages/passport-google-oauth20/): Google দিয়ে login (এই note-এ)
2. [**passport-local**](https://www.passportjs.org/packages/passport-local/): আমাদের email-password login, এবার Passport দিয়ে (পরের note-এ)

> **Tip:** Passport-এর documentation খুব বিস্তারিত না। ভালোভাবে শিখতে চাইলে ওদের GitHub repo-র example গুলো দেখা ভালো।

---

## পুরো Flow একনজরে

```
Frontend (localhost:5173)
   ↓ "Login with Google" এ click
Backend: /api/v1/auth/google
   ↓ Passport user-কে Google-এ পাঠায়
Google Consent Screen → Gmail দিয়ে login
   ↓ সফল হলে Google ফেরত পাঠায়
Backend: /api/v1/auth/google/callback
   ↓ User database-এ আছে কিনা দেখি, না থাকলে তৈরি করি
JWT token বানিয়ে cookie-তে রাখি
   ↓
Frontend-এ redirect
```

### Google আর আমাদের database-এর মধ্যে "Bridge"

Email-password-এ user নিজে register করে, আমরা database-এ রাখি, আর login-এর সময় token দিই।

Google login-এ Google শুধু বলে "এই মানুষটা আসলেই এই Gmail-এর মালিক"। কিন্তু Google আমাদের app-এর `role` জানে না। অথচ JWT token-এ `userId`, `email`, `role` লাগে।

তাই Google login সফল হলে user-কে **আমাদের database-এ রাখতে হবে**, তারপর সেখান থেকে token বানাতে হবে। এজন্যই `IUser`-এ `password` optional রেখেছিলাম, কারণ Google user-এর কোনো password নেই।

নিয়ম হবে:

- User আগে থেকে database-এ আছে → সরাসরি login করিয়ে দাও
- না থাকলে → নতুন user তৈরি করো, তারপর login করাও

---

## Step 1: Google Cloud-এ app বানানো

1. [Google Cloud Console](https://console.cloud.google.com/)-এ login করো।
2. **APIs & Services → OAuth consent screen** setup করো।
3. **Clients (Credentials) → Create OAuth Client** → Application type: **Web application**।
4. এই দুটো ঘর পূরণ করো:

| ঘর | Local-এ কী দেব |
| --- | --- |
| **Authorized JavaScript origins** | `http://localhost:5000` |
| **Authorized redirect URIs** | `http://localhost:5000/api/v1/auth/google/callback` |

> **Redirect URI** হলো Google login সফল হওয়ার পর Google যে URL-এ user-কে ফেরত পাঠাবে। এখানে **backend-এর callback route**-ই দিতে হবে, কারণ token বানানোর কাজটা backend করে। আর এটা `.env`-এর `GOOGLE_CALLBACK_URL`-এর সাথে **হুবহু** মিলতে হবে, নাহলে Google error দেবে। Deploy করার পর live URL যোগ করতে হবে।

5. Create দিলে **Client ID** আর **Client Secret** পাবে।

---

## Step 2: `.env` আপডেট

```env
GOOGLE_CLIENT_ID=your_client_id
GOOGLE_CLIENT_SECRET=your_client_secret
GOOGLE_CALLBACK_URL=http://localhost:5000/api/v1/auth/google/callback
EXPRESS_SESSION_SECRET=your_session_secret
FRONTEND_URL=http://localhost:5173
```

`env.ts`-এর তিন জায়গাতেই (`EnvConfig`, `requiredEnvVariables`, return object) এগুলো যোগ করতে হবে।

---

## Step 3: Package install

```bash
npm i passport passport-local passport-google-oauth20 express-session
npm i -D @types/passport @types/passport-local @types/passport-google-oauth20 @types/express-session
```

---

## Step 4: `app.ts`-এ Passport চালু করা

```ts
import expressSession from "express-session";
import passport from "passport";
import "./app/config/passport";

const app = express();

app.use(
	expressSession({
		secret: envVars.EXPRESS_SESSION_SECRET,
		resave: false,
		saveUninitialized: false,
	}),
);
app.use(passport.initialize());
app.use(passport.session());

app.use(cookieParser());
app.use(express.json());
// ... বাকি সব আগের মতো
```

### ❓ এই অংশটুকু কেন লাগে?

Google login-এর পর Passport নিজে থেকে user-কে একটা **session**-এ মনে রাখতে চায়। Session মানে server-এর কাছে রাখা একটা ছোট "মনে রাখার খাতা", যেটা দিয়ে পরের request-এ server চিনতে পারে এই user আগে login করেছিল।

| কোড | কাজ |
| --- | --- |
| `expressSession({...})` | Session ব্যবস্থা চালু করে |
| `secret` | Session-এর cookie sign করার গোপন key |
| `resave: false` | কিছু না বদলালে প্রতি request-এ session আবার save করবে না |
| `saveUninitialized: false` | খালি session (যেখানে কিছু রাখা হয়নি) save করবে না |
| `passport.initialize()` | Passport চালু করে |
| `passport.session()` | Passport-কে session ব্যবহার করতে দেয় |

> **Order জরুরি:** `expressSession` অবশ্যই `passport.session()`-এর **আগে** থাকতে হবে।

### `import "./app/config/passport"` কেন?

পরের step-এ `passport.ts` file-এ Google strategy setup করব। কিন্তু file বানালেই Express সেটা জানে না। এই import দিয়ে file-টা চালু করে দিচ্ছি, যাতে Passport জানে "google" strategy আছে।

---

## Step 5: Google Strategy setup

`src/app/config/passport.ts`

```ts
/* eslint-disable @typescript-eslint/no-explicit-any */
import passport from "passport";
import { Strategy as GoogleStrategy, type Profile, type VerifyCallback } from "passport-google-oauth20";
import { IsActive, Role } from "../modules/user/user.interface";
import { User } from "../modules/user/user.model";
import { envVars } from "./env";

passport.use(
	new GoogleStrategy(
		{
			clientID: envVars.GOOGLE_CLIENT_ID,
			clientSecret: envVars.GOOGLE_CLIENT_SECRET,
			callbackURL: envVars.GOOGLE_CALLBACK_URL,
		},
		async (accessToken: string, refreshToken: string, profile: Profile, done: VerifyCallback) => {
			try {
				const email = profile.emails?.[0]?.value;

				if (!email) {
					return done(null, false, { message: "No email found" });
				}

				let user = await User.findOne({ email });

				if (user) {
					if (user.isDeleted) {
						return done(null, false, { message: "User is deleted" });
					}

					if (user.isActive === IsActive.BLOCKED || user.isActive === IsActive.INACTIVE) {
						return done(null, false, { message: `User is ${user.isActive}` });
					}

					if (!user.isVerified) {
						return done(null, false, { message: "User is not verified" });
					}

					// Optional: link Google provider to existing account
					const hasGoogle = user.auths?.some((a) => a.provider === "google");
					if (!hasGoogle) {
						user.auths.push({ provider: "google", providerId: profile.id });
						await user.save();
					}
				} else {
					user = await User.create({
						email,
						name: profile.displayName,
						picture: profile.photos?.[0]?.value,
						role: Role.USER,
						isVerified: true,
						auths: [{ provider: "google", providerId: profile.id }],
					});
				}

				return done(null, user);
			} catch (error) {
				console.log("Google Strategy Error", error);
				return done(error);
			}
		},
	),
);

passport.serializeUser((user: any, done: (err: any, id?: unknown) => void) => {
	done(null, user._id);
});

passport.deserializeUser(async (id: string, done: any) => {
	try {
		const user = await User.findById(id);
		done(null, user);
	} catch (error) {
		console.log(error);
		done(error);
	}
});
```

### `GoogleStrategy`-এর দুটো parameter

1. **প্রথম parameter (object)**: Google-এর সাথে যোগাযোগের তথ্য (`clientID`, `clientSecret`, `callbackURL`)।
2. **দ্বিতীয় parameter (function)**: Google login সফল হলে কী করব, সেটা **আমরা** ঠিক করি। আমাদের database-এ user কীভাবে তৈরি হবে, কী কী field লাগবে, সেটা Google বা Passport কেউই জানে না। তাই Passport এই function দিয়ে কাজটা আমাদের হাতে ছেড়ে দেয়।

এই function-এ ৪টা parameter পাই:

| Parameter | কী |
| --- | --- |
| `accessToken`, `refreshToken` | Google-এর নিজের token (Google-এর API ব্যবহার করতে লাগে, আমাদের এখন দরকার নেই) |
| `profile` | Google থেকে আসা user-এর তথ্য (নাম, email, ছবি, id) |
| `done` | কাজ শেষে Passport-কে ফলাফল জানানোর function |

### `done()` কীভাবে কাজ করে?

| লেখা | মানে |
| --- | --- |
| `done(null, user)` | সফল, এই user login করল |
| `done(null, false, { message })` | কোনো error হয়নি, কিন্তু login দেওয়া যাবে না |
| `done(error)` | কোনো অপ্রত্যাশিত error হয়েছে |

> **এখানে `AppError` throw করি না কেন?** আমরা এখন Express-এর controller-এ না, Passport-এর ভিতরে আছি। Passport ফলাফল জানে শুধু `done()` দিয়ে। তাই "email নেই" বা "user blocked" এর মতো অবস্থায় `done(null, false, ...)` দিয়ে জানাতে হয়।

### `?.[0]?.value` কেন?

Google সবসময় email বা ছবি নাও দিতে পারে। আর আমাদের `tsconfig`-এ `noUncheckedIndexedAccess: true` আছে, তাই TypeScript ধরে নেয় `[0]` খালি হতে পারে। তাই `[0]`-এর পরেও `?.` দিতে হয়।

### `serializeUser` আর `deserializeUser`

Session-এ পুরো user object রাখা ভারী, তাই শুধু user-এর `_id` রাখি।

- **`serializeUser`**: Login-এর সময় user থেকে শুধু `_id` বের করে session-এ রাখে।
- **`deserializeUser`**: পরের request-এ session-এর `_id` দিয়ে database থেকে পুরো user আবার খুঁজে আনে।

এগুলো না দিলে `/google`-এ request দিলে Passport error দেয় যে session-এ user রাখতে পারছে না।

> **জেনে রাখো:** আমরা আসলে login মনে রাখার কাজ JWT দিয়ে করছি, session দিয়ে না। চাইলে `passport.authenticate("google", { session: false })` দিয়ে session বন্ধ রাখা যায়, তখন `express-session` আর serialize/deserialize লাগে না। তবে course-এর সাথে মিল রাখতে আমরা session রেখেই করছি।

---

## Step 6: Routes

`auth.route.ts`

```ts
import passport from "passport";
import { envVars } from "../../config/env";

// Google login শুরু
router.get("/google", (req: Request, res: Response, next: NextFunction) => {
	const redirect = req.query.redirect || "/";

	passport.authenticate("google", {
		scope: ["profile", "email"],
		state: redirect as string,
	})(req, res, next);
});

// Google login শেষে Google এখানে ফেরত পাঠাবে
router.get(
	"/google/callback",
	passport.authenticate("google", { failureRedirect: `${envVars.FRONTEND_URL}/login` }),
	AuthControllers.googleCallbackController,
);
```

### কিছু ব্যাখ্যা

- **`GET` কেন?** এখানে body-তে কিছু পাঠাতে হয় না। User শুধু একটা link-এ যায়, তাই `GET`।
- **`scope: ["profile", "email"]`**: Google-এর কাছে user-এর profile আর email চাইছি।
- **`state: redirect`**: User login-এর আগে কোন page-এ ছিল (যেমন `/booking`), সেটা Google-কে দিয়ে রাখছি। Google login শেষে এটা আবার ফেরত দেয়, যাতে user-কে সেই page-এই পাঠানো যায়।
- **`failureRedirect`**: Login ব্যর্থ হলে (যেমন user blocked) frontend-এর login page-এ পাঠাবে।

### Redirect-এর উদাহরণ

```
User /booking page-এ যেতে চাইল → login নেই → /login page-এ পাঠানো হলো
   ↓
Frontend: localhost:5000/api/v1/auth/google?redirect=/booking
   ↓
Google login সফল
   ↓
Callback: /api/v1/auth/google/callback?state=/booking
   ↓
User-কে আবার /booking page-এ পাঠানো হলো ✅
```

---

## Step 7: Callback Controller

`auth.controller.ts`

```ts
const googleCallbackController = catchAsync(async (req: Request, res: Response, next: NextFunction) => {
	let redirectTo = req.query.state ? (req.query.state as string) : "";

	// "/booking" → "booking", "/" → ""
	if (redirectTo.startsWith("/")) {
		redirectTo = redirectTo.slice(1);
	}

	const user = req.user;

	if (!user) {
		throw new AppError(httpStatus.NOT_FOUND, "User not found");
	}

	const tokenInfo = createUserTokens(user);

	setAuthCookie(res, tokenInfo);

	res.redirect(`${envVars.FRONTEND_URL}/${redirectTo}`);
});
```

### `req.user` কোথা থেকে এলো?

Email-password login-এ আমরা নিজে `checkAuth`-এ `req.user` বসিয়েছিলাম। কিন্তু এখানে **Passport নিজেই** কাজটা করে। `passport.ts`-এ `done(null, user)` দিয়ে যে user পাঠিয়েছিলাম, `passport.authenticate()` middleware সেটাকে `req.user`-এ বসিয়ে দেয়। তাই controller-এ সরাসরি পেয়ে যাই।

### Response না পাঠিয়ে redirect কেন?

এই request আসে Google থেকে, browser-এর মাধ্যমে, Postman বা frontend-এর `fetch` থেকে না। তাই JSON response পাঠালে user শুধু একটা JSON লেখা দেখবে। এজন্য token cookie-তে রেখে user-কে frontend-এ **redirect** করে দিই।

---

## Test করা

Google login **browser** দিয়ে test করতে হবে, Postman দিয়ে না। কারণ Google-এর consent screen (Gmail বেছে নেওয়ার page) Postman-এ দেখা যায় না।

Browser-এ যাও:

```
http://localhost:5000/api/v1/auth/google
```

Gmail দিয়ে login করলে database-এ নতুন user তৈরি হবে, cookie-তে token বসবে, আর frontend URL-এ redirect হবে। (Frontend চালু না থাকলে redirect page খুলবে না, কিন্তু database আর cookie check করে দেখা যাবে কাজ হয়েছে।)

---

> **TypeScript error দেখালে:** `@types/passport` নিজেও Express-এর `Request`-এ `user` property যোগ করে। আমাদের `index.d.ts`-এ `user: JwtPayload` থাকায় দুটো declaration মিলে না গিয়ে error দেখাতে পারে (*Subsequent property declarations must have the same type*)। তখন `index.d.ts`-এ `Request`-এর বদলে Passport-এর `User` interface-এ `JwtPayload` যোগ করতে হয়:
>
> ```ts
> import type { JwtPayload } from "jsonwebtoken";
>
> declare global {
> 	namespace Express {
> 		// eslint-disable-next-line @typescript-eslint/no-empty-object-type
> 		interface User extends JwtPayload {}
> 	}
> }
> ```
>
> এতে `req.user`-এর type হয় `User | undefined`, তাই যেসব controller-এ `req.user` service-এ পাঠাই (যেমন update user, change password), সেখানে `req.user as JwtPayload` লিখতে হবে।