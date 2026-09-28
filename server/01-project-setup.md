# Project Setup আর Server Error Handling

## Step 1: Folder আর Git setup

1. `phovia-tour` নামে একটা folder বানাব, তার ভিতরে আরেকটা folder বানাব `phovia-tour-server` নামে।
2. `phovia-tour-server` folder-টা code editor-এ open করব।
3. Git চালু করব আর `.gitignore` file বানাব:

```bash
git init
```

`.gitignore`-এ অন্তত এগুলো রাখব:

```gitignore
node_modules
dist
.env
```

4. npm project শুরু করব:

```bash
npm init -y
```

---

## Step 2: TypeScript setup

TypeScript install করে `tsconfig.json` file বানাব:

```bash
npm i -D typescript
npx tsc --init
```

`tsconfig.json`-এ `rootDir` খুঁজে `./src` দেব, আর `outDir`-এ `./dist` দেব:

```json
{
	"compilerOptions": {
		// File Layout
		"rootDir": "./src",
		"outDir": "./dist",

		// Environment Settings
		"module": "preserve",
		"moduleResolution": "bundler",
		"noEmit": true,
		"target": "es2023",
		"lib": ["es2023"],
		"types": ["node"],

		// Output
		"sourceMap": true,

		// Stricter Typechecking
		"strict": true,
		"noUncheckedIndexedAccess": true,
		"noImplicitReturns": true,
		"noImplicitOverride": true,
		"noFallthroughCasesInSwitch": true,

		// Module & Interop
		"verbatimModuleSyntax": true,
		"isolatedModules": true,
		"noUncheckedSideEffectImports": true,
		"moduleDetection": "force",
		"esModuleInterop": true,
		"resolveJsonModule": true,
		"forceConsistentCasingInFileNames": true,
		"skipLibCheck": true
	},
	"include": ["src"],
	"exclude": ["node_modules", "dist"]
}
```

> **Note:** `"noEmit": true` দেওয়ার মানে `tsc` কোনো JavaScript file বানাবে না, শুধু type check করবে। Build-এর কাজটা আমরা `tsup` দিয়ে করব।

---

## Step 3: Package install

```bash
npm install express mongoose zod jsonwebtoken cors dotenv
npm install -D tsx @types/node @types/express @types/jsonwebtoken @types/cors tsup
```

| Package | কাজ |
| --- | --- |
| `express` | Server আর API বানানো |
| `mongoose` | MongoDB database-এর সাথে কাজ |
| `zod` | Data validation |
| `jsonwebtoken` | Authentication-এর জন্য JWT token |
| `cors` | অন্য domain (যেমন frontend) থেকে request আসতে দেওয়া |
| `dotenv` | `.env` file থেকে variable পড়া |
| `tsx` | Development-এ সরাসরি `.ts` file চালানো |
| `tsup` | Production-এর জন্য build করা |

`package.json`-এর `scripts`-এ এগুলো যোগ করব:

```json
"scripts": {
  "dev": "tsx watch src/server.ts",
  "build": "tsup src/server.ts",
  "start": "node dist/server.js",
  "lint": "eslint ./src"
}
```

---

## Step 4: Folder structure (Modular MVC Pattern)

আমরা project গুলো **modular MVC pattern**-এ করব। মানে প্রতিটা feature (module)-এর জন্য আলাদা folder থাকবে, আর সেই folder-এর ভিতরে ওই feature-এর model, controller, interface ইত্যাদি থাকবে।

```
src/
├── server.ts
├── app.ts
└── app/
    ├── config/
    │   └── env.ts
    └── modules/
        ├── user/
        │   ├── user.interface.ts
        │   ├── user.model.ts
        │   └── user.controller.ts
        └── tour/
            ├── tour.interface.ts
            ├── tour.model.ts
            └── tour.controller.ts
```

### `server.ts` আর `app.ts` এর পার্থক্য

- **`server.ts`**: Server-related কাজ করে, যেমন database-এর সাথে connect করা, port-এ server চালু করা, server-এর error handle করা।
- **`app.ts`**: Express app-এর কাজ করে, যেমন middleware, route ইত্যাদি সেট করা।

---

## Step 5: `app.ts`

```ts
import type { Request, Response } from "express";
import express from "express";
import cors from "cors";

const app = express();

app.use(express.json()); // request body থেকে JSON পড়ার জন্য
app.use(cors());

app.get("/", (req: Request, res: Response) => {
	res.json({
		message: "Welcome to Phovia Tour!",
	});
});

export default app;
```

---

## Step 6: `server.ts`

> **সতর্কতা:** Database URL বা password কখনো সরাসরি কোডে লিখব না। সবসময় `.env` file-এ রেখে `envVars` দিয়ে ব্যবহার করব ([env guide](https://github.com/arifucoder/dev-starter-notes/blob/main/mern/server/features/2.env-handle.md) দেখো)।

```ts
/* eslint-disable no-console */
import type { Server } from "http";
import mongoose from "mongoose";
import app from "./app";
import { envVars } from "./app/config/env";

let server: Server;

const startServer = async () => {
	try {
		await mongoose.connect(envVars.DB_URL);
		console.log("Connected to database");

		server = app.listen(envVars.PORT, () => {
			console.log(`Server is running on port ${envVars.PORT}`);
		});
	} catch (error) {
		console.log(error);
	}
};

startServer();
```

---

## Step 7: Server-এর error handle করা

### `process` কী?

`process` হলো Node.js-এর একটা global object, যেটা দিয়ে আমাদের চলমান server (process)-এর তথ্য পাওয়া যায় আর সেটাকে control করা যায়। যেমন `process.env` দিয়ে env variable পড়ি, `process.exit()` দিয়ে server বন্ধ করি, আর `process.on()` দিয়ে বিভিন্ন event শুনি।

### চার ধরনের event

**১. Unhandled Rejection (`unhandledRejection`)**
অন্য কোনো server বা database-এর সাথে কাজ করার সময় আমরা promise ব্যবহার করি। Promise resolve বা reject হতে পারে। Reject হলে সাধারণত আমরা `try...catch` দিয়ে error ধরি। কিন্তু কোথাও ভুলে `catch` না লিখলে সেই error ধরা পড়ে না, আর server crash করতে পারে। এই ধরনের error এখানে handle করব।

**২. Uncaught Exception (`uncaughtException`)**
এটা promise-এর বাইরের error। যেমন এমন একটা variable `console.log` করলাম যেটা declare-ই করিনি। কোডের এমন error যেটা কোথাও ধরা হয়নি, সেটাই uncaught exception।

**৩. SIGTERM**
ধরো project-টা AWS, Vercel ইত্যাদিতে live করলাম। কোনো কারণে (যেমন maintenance) hosting platform server বন্ধ করার একটা signal পাঠাল। সেই signal পেয়ে আমরা server-টা **gracefully shutdown** করব, মানে চলমান কাজ শেষ করে তারপর বন্ধ করব।

**৪. SIGINT**
Terminal-এ `Ctrl + C` চাপলে এই signal আসে।

### কোড

একই কোড চারবার না লিখে একটা `shutdown` function বানিয়ে নিলাম:

```ts
const shutdown = (exitCode: number) => {
	if (server) {
		// নতুন request নেওয়া বন্ধ করে, চলমান request শেষ হলে process বন্ধ করবে
		server.close(() => {
			process.exit(exitCode);
		});
	} else {
		process.exit(exitCode);
	}
};

process.on("unhandledRejection", (err) => {
	console.log("Unhandled Rejection detected... Server shutting down..", err);
	shutdown(1);
});

process.on("uncaughtException", (err) => {
	console.log("Uncaught Exception detected... Server shutting down..", err);
	shutdown(1);
});

process.on("SIGTERM", () => {
	console.log("SIGTERM signal received... Server shutting down..");
	shutdown(0);
});

process.on("SIGINT", () => {
	console.log("SIGINT signal received... Server shutting down..");
	shutdown(0);
});
```

> **Exit code:** `0` মানে স্বাভাবিকভাবে বন্ধ হয়েছে, `1` মানে error-এর কারণে বন্ধ হয়েছে। SIGTERM আর SIGINT কোনো error না, তাই সেখানে `0` দিয়েছি।

---

## Final Code

### `src/app.ts`

```ts
import type { Request, Response } from "express";
import express from "express";
import cors from "cors";

const app = express();

app.use(express.json());
app.use(cors());

app.get("/", (req: Request, res: Response) => {
	res.status(200).json({
		message: "Welcome to Tour Management System Backend",
	});
});

export default app;
```

### `src/server.ts`

```ts
/* eslint-disable no-console */
import type { Server } from "http";
import mongoose from "mongoose";
import app from "./app";
import { envVars } from "./app/config/env";

let server: Server;

const PORT = envVars.PORT;

const startServer = async () => {
	try {
		await mongoose.connect(envVars.DB_URL);
		console.log("Connected to DB");

		server = app.listen(PORT, () => {
			console.log(`Server is listening on port ${PORT}`);
		});
	} catch (error) {
		console.log(error);
	}
};

startServer();

const shutdown = (exitCode: number) => {
	if (server) {
		server.close(() => {
			process.exit(exitCode);
		});
	} else {
		process.exit(exitCode);
	}
};

process.on("unhandledRejection", (err) => {
	console.log("Unhandled Rejection detected... Server shutting down..", err);
	shutdown(1);
});

process.on("uncaughtException", (err) => {
	console.log("Uncaught Exception detected... Server shutting down..", err);
	shutdown(1);
});

process.on("SIGTERM", () => {
	console.log("SIGTERM signal received... Server shutting down..");
	shutdown(0);
});

process.on("SIGINT", () => {
	console.log("SIGINT signal received... Server shutting down..");
	shutdown(0);
});
```

---

## Related Guides

- ESLint guide: https://github.com/arifucoder/dev-starter-notes/blob/main/mern/server/features/1-es-lint.md
- Env guide: https://github.com/arifucoder/dev-starter-notes/blob/main/mern/server/features/2.env-handle.md