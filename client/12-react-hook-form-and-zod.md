# Frontend Notes — ১২: React Hook Form + Zod (Login ও Register form)

এই প্রজেক্টে form সামলানোর জন্য **React Hook Form** আর validation-এর জন্য **Zod** ব্যবহার করব।

Docs: https://ui.shadcn.com/docs/forms/react-hook-form

এই নোট ১১ নম্বর নোটের (Login/Register page design) পরের ধাপ। সেখানে বানানো `AuthLayout`, `PasswordInput` আর ফাইলের structure এখানেও একই থাকছে।

---

## ১. Teacher-এর কোডের সাথে কেন মিলছে না

Teacher পুরনো version-এ কাজ করেছেন, তাই অনেক কোড মিলবে না। তবে পার্থক্যটা মূলত **React Hook Form-এ না**। React Hook Form-এর v7-এর মূল API (`useForm`, `handleSubmit`, `Controller`) এখনো প্রায় একই। আসল পরিবর্তন এসেছে দুই জায়গায়:

### ক) shadcn-এর form component বদলেছে

| পুরনো shadcn (Teacher) | নতুন shadcn (আমরা) |
|---|---|
| `bunx shadcn add form` | `bunx shadcn add field` |
| `<Form>`, `<FormField>`, `<FormItem>` | `<Controller>` (সরাসরি React Hook Form থেকে) + `<Field>` |
| `<FormLabel>` | `<FieldLabel>` |
| `<FormControl>` | লাগে না, Input সরাসরি বসে |
| `<FormMessage />` | `<FieldError errors={[fieldState.error]} />` |

### খ) Zod 3 থেকে Zod 4

| Zod 3 (Teacher) | Zod 4 (আমরা) |
|---|---|
| `z.string().email()` | `z.email()` |
| `{ message: "..." }` | `{ error: "..." }` |
| `{ required_error, invalid_type_error }` | একটাই `{ error: "..." }` |

> পুরনো লেখাগুলো (`message`, `z.string().email()`) Zod 4-এও কাজ করে, তবে **deprecated**: পরের version-এ মুছে যেতে পারে, editor-এ দাগ কাটা দেখায়। তাই নতুন কোডে নতুন লেখা ব্যবহার করব।

---

## ২. Install

```sh
bun add react-hook-form zod @hookform/resolvers
bunx --bun shadcn@latest add field input button textarea separator
```

> ⚠️ সংশোধন:
> - আমরা bun ব্যবহার করছি, তাই `npm install`/`npx`-এর বদলে `bun add`/`bunx --bun` (৩ নম্বর নোট)।
> - Register form-এর Address field-এ **`Textarea`** ব্যবহার হয়েছে, কিন্তু install command-এ `textarea` ছিল না। না দিলে import error দেবে।

| Package | কাজ |
|---|---|
| `react-hook-form` | Form-এর value, error, submit সামলায় |
| `zod` | Data কেমন হলে valid, সেই নিয়ম (schema) লেখা |
| `@hookform/resolvers` | Zod-কে **React Hook Form**-এর সাথে জোড়া লাগায় |

> ⚠️ সংশোধন: নোটে লেখা ছিল "resolver Zod-কে **React**-এর সাথে connect করে"। আসলে connect করে **React Hook Form**-এর সাথে। React Hook Form নিজে Zod চেনে না। Resolver দুইজনের মধ্যে অনুবাদকের কাজ করে: Zod-এর error-কে React Hook Form-এর বোঝার মতো error-এ বদলে দেয়।

---

## ৩. ছোট একটা উদাহরণ: পুরো প্রবাহ

```tsx
// "use client" — এটা শুধু Next.js-এ লাগে, Vite-এ লাগে না
import { zodResolver } from "@hookform/resolvers/zod";
import { Controller, useForm } from "react-hook-form";
import * as z from "zod";

import { Button } from "@/components/ui/button";
import { Field, FieldError, FieldLabel } from "@/components/ui/field";
import { Input } from "@/components/ui/input";

// ১. Schema: কোন data valid, তার নিয়ম
const formSchema = z.object({
  email: z.email({ error: "সঠিক email দিন।" }),
});

// ২. Schema থেকে TypeScript type বানানো
type FormValues = z.infer<typeof formSchema>;   // { email: string }

export function EmailForm() {
  // ৩. Form তৈরি, Zod দিয়ে validate হবে
  const form = useForm<FormValues>({
    resolver: zodResolver(formSchema),
    defaultValues: { email: "" },
  });

  // ৪. Data valid হলেই এটা চলবে
  function onSubmit(data: FormValues) {
    console.log(data); // { email: "abc@gmail.com" }
  }

  return (
    <form onSubmit={form.handleSubmit(onSubmit)} className="max-w-sm space-y-4" noValidate>
      {/* ৫. Field */}
      <Controller
        name="email"
        control={form.control}
        render={({ field, fieldState }) => (
          <Field data-invalid={fieldState.invalid}>
            <FieldLabel htmlFor="email">Email</FieldLabel>
            <Input {...field} id="email" placeholder="you@example.com" aria-invalid={fieldState.invalid} />
            {/* ৬. Error থাকলে দেখাবে */}
            {fieldState.invalid && <FieldError errors={[fieldState.error]} />}
          </Field>
        )}
      />

      <Button type="submit">Submit</Button>
    </form>
  );
}
```

> নোট: উদাহরণে `z.string().email("...")` ছিল, এটা Zod 3-এর লেখা। Zod 4-এ লেখা হয় `z.email({ error: "..." })`।

### Submit চাপলে কী হয় — ধাপে ধাপে

```
Submit চাপা
   ↓
form.handleSubmit()  →  resolver Zod দিয়ে পুরো data যাচাই করে
   ↓                          ↓
 valid?  ── হ্যাঁ ──→  onSubmit(data) চলে
   │
   না
   ↓
error-গুলো fieldState.error-এ বসে  →  FieldError দেখায়, onSubmit চলে না
```

| অংশ | কাজ |
|---|---|
| `useForm()` | Form-এর সব কিছু রাখে: value, error, submit অবস্থা |
| `resolver: zodResolver(formSchema)` | Validation-এর কাজ Zod-কে দেয় |
| `defaultValues` | প্রতিটা field-এর শুরুর মান (নিচে controlled অংশে কেন জরুরি) |
| `form.handleSubmit(onSubmit)` | Submit আটকে আগে যাচাই করে। Valid হলেই `onSubmit` ডাকে। Page reload-ও নিজেই আটকায় |
| `Controller` | একটা field-কে form-এর সাথে জোড়া লাগায় |
| `field` | `value`, `onChange`, `onBlur`, `name`, `ref`। `{...field}` দিয়ে একবারে Input-এ দেওয়া হয় |
| `fieldState.invalid` / `.error` | এই field-এ error আছে কিনা / error message |
| `data-invalid`, `aria-invalid` | Error থাকলে Field আর Input লাল দেখায়, screen reader-ও জানে |
| `noValidate` | Browser-এর নিজস্ব validation বন্ধ করে, যাতে Zod-এর message দেখায় (নিচে দেখুন) |
| `z.infer<typeof formSchema>` | Schema থেকে TypeScript type। একই জিনিস দুইবার লিখতে হয় না |

---

## ৪. Controlled বনাম Uncontrolled — সহজ ব্যাখ্যা

প্রশ্নটা হলো: **Input-এ যা লেখা হচ্ছে, সেই মান কার কাছে থাকে?**

### Uncontrolled: মান থাকে **browser (DOM)**-এর কাছে

```tsx
<input defaultValue="" ref={inputRef} />
// দরকার হলে পড়ি: inputRef.current.value
```

User টাইপ করে, browser নিজেই মান রাখে। React প্রতিটা অক্ষর জানে না, দরকার হলে (যেমন submit-এর সময়) গিয়ে পড়ে নেয়।

### Controlled: মান থাকে **React (state)**-এর কাছে

```tsx
const [email, setEmail] = useState("");
<input value={email} onChange={(e) => setEmail(e.target.value)} />
```

প্রতিটা অক্ষর টাইপ করলেই `onChange` → state বদলায় → React আবার render করে → input নতুন মান দেখায়। Input নিজে কিছু মনে রাখে না, React যা বলে তাই দেখায়।

### একটা উদাহরণ দিয়ে

- **Uncontrolled:** ছাত্র নিজের খাতায় লিখছে, শিক্ষক পরীক্ষা শেষে খাতা নিয়ে দেখেন।
- **Controlled:** ছাত্র যা বলছে শিক্ষক বোর্ডে লিখছেন। বোর্ডে যা আছে সেটাই সত্যি, শিক্ষক চাইলে যেকোনো সময় মুছে বা বদলে দিতে পারেন।

| | Uncontrolled | Controlled |
|---|---|---|
| মান কোথায় | DOM (browser) | React state |
| শুরুর মান | `defaultValue` | `value` |
| প্রতি অক্ষরে re-render | না, দ্রুত | হ্যাঁ |
| React সবসময় মান জানে? | না, পড়ে নিতে হয় | হ্যাঁ |
| React Hook Form-এ | `register("email")` | `<Controller />` |

### Teacher-এর কথাটা

Teacher বলেছেন "controlled মানে আমার control-এ আছে, default-এ কিছু set করে দিতে হবে, তারপর change করতে পারব।" কথাটা **ঠিক**, তবে পুরো ছবি হলো:

- **"আমার control-এ"** মানে মানটা React-এর state-এ থাকে, React-ই ঠিক করে input-এ কী দেখাবে।
- **"Default set করতে হবে"** কারণ controlled input-এর শুরু থেকেই একটা মান লাগে। `defaultValues` না দিলে শুরুতে `value` হয় `undefined`। React তখন input-টাকে uncontrolled ভাবে, তারপর টাইপ করলে মান আসায় হঠাৎ controlled হয়ে যায়। Console-এ warning আসে:
  > `A component is changing an uncontrolled input to be controlled.`

  তাই **প্রতিটা field-এর** default মান দিতে হয়, খালি হলে `""`। Optional field (phone, address) হলেও।

> **shadcn কেন `Controller` (controlled) ব্যবহার করে?** React Hook Form-এর default আসলে uncontrolled (`register`), কারণ সেটা দ্রুত। কিন্তু Radix-এর Select, Checkbox, Switch সাধারণ `<input>` না, তাই `register` দিয়ে সরাসরি কাজ করে না। `Controller` সব রকম component-এ একইভাবে কাজ করে, তাই shadcn সব জায়গায় একই নিয়ম রাখতে `Controller` ব্যবহার করে। আমাদের মতো ছোট form-এ re-render-এর খরচ টের পাওয়া যায় না।

---

## ৫. Register form-এর Schema

```tsx
import * as z from "zod";

// বাংলাদেশি মোবাইল নম্বর: 01 + (3-9) + আরও 8 অঙ্ক, শুরুতে চাইলে +88
const bdPhoneRegex = /^(?:\+88)?01[3-9]\d{8}$/;

const baseSchema = z.object({
  name: z
    .string()
    .trim()
    .min(2, { error: "Name must be at least 2 characters long." })
    .max(50, { error: "Name cannot exceed 50 characters." }),
  email: z
    .email({ error: "Invalid email address format." })
    .max(100, { error: "Email cannot exceed 100 characters." }),
  password: z
    .string()
    .min(8, { error: "Password must be at least 8 characters long." })
    .regex(/[A-Z]/, { error: "Password must contain at least 1 uppercase letter." })
    .regex(/\d/, { error: "Password must contain at least 1 number." })
    .regex(/[^A-Za-z0-9]/, { error: "Password must contain at least 1 special character." }),
  // Confirm password-এ আলাদা নিয়ম লাগে না, শুধু password-এর সাথে মিলতে হবে (নিচে refine)
  confirmPassword: z.string().min(1, { error: "Please confirm your password." }),
  // Optional: খালি ("") হলে valid, কিছু লিখলে সঠিক নম্বর হতে হবে
  phone: z
    .string()
    .trim()
    .refine((value) => value === "" || bdPhoneRegex.test(value), {
      error: "Enter a valid Bangladeshi number: 01XXXXXXXXX or +8801XXXXXXXXX",
    }),
  address: z.string().trim().max(200, { error: "Address cannot exceed 200 characters." }),
});

export const registerSchema = baseSchema.refine((data) => data.password === data.confirmPassword, {
  error: "Passwords do not match.",
  path: ["confirmPassword"],
  // অন্য field-এ error থাকলেও এই check চালানোর জন্য (নিচে ব্যাখ্যা)
  when(payload) {
    return baseSchema
      .pick({ password: true, confirmPassword: true })
      .safeParse(payload.value).success;
  },
});

export type RegisterFormValues = z.infer<typeof registerSchema>;
```

### `refine` কী

Zod-এ অনেক validation আগে থেকে তৈরি আছে (`min`, `max`, `email`, `regex` …)। কিন্তু **"দুই field-এর মান মিলছে কিনা"**-এর মতো নিজস্ব শর্ত Zod জানে না। সেটা `refine` দিয়ে লেখা হয়:

```ts
.refine(
  (data) => data.password === data.confirmPassword,   // ১. শর্ত: true হলে valid
  {
    error: "Passwords do not match.",                  // ২. false হলে এই message
    path: ["confirmPassword"],                         // ৩. message কোন field-এর নিচে দেখাবে
  },
)
```

- **Callback (`data`):** schema-র সব field-এর মান একসাথে পায়। তাই দুই field তুলনা করা যায়।
- **`path`:** খুব জরুরি। এটা না দিলে error পুরো form-এর হয়ে যায়, কোনো field-এর নিচে দেখায় না। `["confirmPassword"]` দিলে Confirm password-এর নিচে দেখায়।

> নোট: একটা field-এর ভেতরেও `refine` দেওয়া যায়। উপরের `phone`-এ সেটাই করা হয়েছে।

### Schema-তে যা ঠিক করা হয়েছে

> ⚠️ **সংশোধন ১ — Password mismatch error দেরিতে দেখাত:** Zod 4-এ object-এর `refine` by default **তখনই চলে যখন বাকি সব field valid**। ধরুন user দুই password আলাদা দিয়েছে, কিন্তু phone-ও ভুল। তখন শুধু phone-এর error দেখাবে, "Passwords do not match" দেখাবে না। Phone ঠিক করার পরে হঠাৎ নতুন error আসবে। `when` দিয়ে বলা হয়েছে: শুধু password আর confirmPassword ঠিক থাকলেই এই check চালাও, বাকি field যাই হোক। এজন্য schema দুই ভাগে লেখা হয়েছে: `baseSchema` আর তার ওপর `refine`। কারণ `when`-এর ভেতরে `baseSchema.pick()` ব্যবহার করতে হয়।

> ⚠️ **সংশোধন ২ — Phone-এর custom message হারিয়ে যেতে পারত:** আগে লেখা ছিল `z.string().regex(...).or(z.literal(""))`। `.or()` (union)-এ দুটো option-ই fail করলে Zod 4 অনেক সময় নিজের সাধারণ "Invalid input" message দেয়, ফলে আমাদের লেখা বাংলাদেশি নম্বরের message না-ও দেখাতে পারে। `refine` দিয়ে "খালি অথবা সঠিক নম্বর" একটা শর্তে লেখা হয়েছে, এতে সবসময় আমাদের message দেখায়।

> ⚠️ **সংশোধন ৩ — Phone regex:** `01\d{9}` দিলে `010…`, `011…`, `012…`-ও valid হয়ে যেত, অথচ বাংলাদেশে মোবাইল নম্বর শুরু হয় `013` থেকে `019` দিয়ে। এখন `01[3-9]\d{8}` (মোট ১১ অঙ্কই থাকছে)। `(?:\+88)?` মানে শুরুতে `+88` থাকলেও চলবে, না থাকলেও চলবে।

> ⚠️ **সংশোধন ৪ — Special character খুব সীমিত ছিল:** `[!@#$%^&*]` দিলে শুধু এই ৮টা চিহ্ন গোনা হতো। কেউ `_`, `-`, `.` বা `?` দিলে error দিত, যদিও সেগুলোও special character। `[^A-Za-z0-9]` মানে "অক্ষর বা সংখ্যা না এমন যেকোনো কিছু"।
>
> 💡 **Backend-এর নিয়মের সাথে মিলিয়ে নিন:** Frontend-এর password/phone নিয়ম আর backend-এর validation (Zod/Mongoose) **একই** হতে হবে। নাহলে frontend পাস করবে কিন্তু backend reject করবে, অথবা উল্টোটা।

> ⚠️ **সংশোধন ৫ — শুধু space দিয়ে নাম:** `"   "` (৩টা space) দিলে `min(2)` পাস করে যেত। `.trim()` আগে দুই পাশের space মুছে তারপর গোনে।

> নোট — যেগুলো সরানো হয়েছে:
> - `z.string({ error: "Name must be string" })`: Input থেকে সবসময় string-ই আসে, তাই এই message কখনো দেখাবে না।
> - Email-এর `.min(5)`: একটা valid email (`a@b.co`) এমনিতেই ৫ অক্ষরের বেশি, তাই এই check কখনো আলাদা করে fail করবে না।
>
> রেখে দিলেও ভুল না, শুধু অপ্রয়োজনীয়।

---

## ৬. Register form — পুরো কোড

> ⚠️ **সংশোধন — আগের নোটের structure-এর সাথে মেলানো:** দেওয়া কোডটা ১১ নম্বর নোটের আগের version-এর ওপর লেখা ছিল। তাই এতে আবার hard-coded রং (`text-slate-900`, `bg-indigo-600`), `<a href>`, `/signup`, `@/assets/icons/Logo` আর form-এর ভেতরে page-এর কাঠামো (mobile logo, centering div) চলে এসেছিল। নিচের কোড ১১ নম্বর নোটের structure মেনে লেখা:
> - ফাইল: `components/modules/Authentication/RegisterForm.tsx` (`components/authentication/RegistrationForm.tsx` না)
> - কাঠামো আর Logo `AuthLayout`-এ, এখানে শুধু form
> - চোখের button-এর জন্য দুইবার `useState` লেখার বদলে আগের `PasswordInput`
> - সব রং semantic token, সব link `<Link>`

```tsx
// src/components/modules/Authentication/RegisterForm.tsx
import GoogleIcon from "@/components/icons/GoogleIcon";
import { Button } from "@/components/ui/button";
import { Field, FieldError, FieldLabel } from "@/components/ui/field";
import { Input } from "@/components/ui/input";
import { PasswordInput } from "@/components/ui/password-input";
import { Separator } from "@/components/ui/separator";
import { Textarea } from "@/components/ui/textarea";
import { zodResolver } from "@hookform/resolvers/zod";
import { Controller, useForm } from "react-hook-form";
import { Link } from "react-router";
import { registerSchema, type RegisterFormValues } from "./auth.schema";

export default function RegisterForm() {
  const form = useForm<RegisterFormValues>({
    resolver: zodResolver(registerSchema),
    // প্রতিটা field-এর default value দিতে হবে (controlled input)
    defaultValues: {
      name: "",
      email: "",
      password: "",
      confirmPassword: "",
      phone: "",
      address: "",
    },
  });

  function onSubmit(data: RegisterFormValues) {
    console.log(data); // পরে এখানে API call হবে
  }

  return (
    <>
      <h1 className="text-3xl font-bold text-foreground">Create your account</h1>
      <p className="mt-2 text-sm text-muted-foreground">Sign up to start booking your tours.</p>

      <Button variant="outline" className="mt-6 w-full gap-2" type="button">
        <GoogleIcon />
        Sign up with Google
      </Button>

      <div className="my-5 flex items-center gap-3">
        <Separator className="flex-1" />
        <span className="text-xs text-muted-foreground">or sign up with email</span>
        <Separator className="flex-1" />
      </div>

      <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-5" noValidate>
        <Controller
          name="name"
          control={form.control}
          render={({ field, fieldState }) => (
            <Field data-invalid={fieldState.invalid}>
              <FieldLabel htmlFor="name">Full name</FieldLabel>
              <Input
                {...field}
                id="name"
                type="text"
                placeholder="Your full name"
                autoComplete="name"
                aria-invalid={fieldState.invalid}
              />
              {fieldState.invalid && <FieldError errors={[fieldState.error]} />}
            </Field>
          )}
        />

        <Controller
          name="email"
          control={form.control}
          render={({ field, fieldState }) => (
            <Field data-invalid={fieldState.invalid}>
              <FieldLabel htmlFor="email">Email</FieldLabel>
              <Input
                {...field}
                id="email"
                type="email"
                placeholder="you@example.com"
                autoComplete="email"
                aria-invalid={fieldState.invalid}
              />
              {fieldState.invalid && <FieldError errors={[fieldState.error]} />}
            </Field>
          )}
        />

        <Controller
          name="password"
          control={form.control}
          render={({ field, fieldState }) => (
            <Field data-invalid={fieldState.invalid}>
              <FieldLabel htmlFor="password">Password</FieldLabel>
              <PasswordInput
                {...field}
                id="password"
                placeholder="At least 8 characters"
                autoComplete="new-password"
                aria-invalid={fieldState.invalid}
              />
              {fieldState.invalid && <FieldError errors={[fieldState.error]} />}
            </Field>
          )}
        />

        <Controller
          name="confirmPassword"
          control={form.control}
          render={({ field, fieldState }) => (
            <Field data-invalid={fieldState.invalid}>
              <FieldLabel htmlFor="confirmPassword">Confirm password</FieldLabel>
              <PasswordInput
                {...field}
                id="confirmPassword"
                placeholder="Re-enter password"
                autoComplete="new-password"
                aria-invalid={fieldState.invalid}
              />
              {fieldState.invalid && <FieldError errors={[fieldState.error]} />}
            </Field>
          )}
        />

        <Controller
          name="phone"
          control={form.control}
          render={({ field, fieldState }) => (
            <Field data-invalid={fieldState.invalid}>
              <FieldLabel htmlFor="phone">Phone (optional)</FieldLabel>
              <Input
                {...field}
                id="phone"
                type="tel"
                placeholder="01XXXXXXXXX"
                autoComplete="tel"
                aria-invalid={fieldState.invalid}
              />
              {fieldState.invalid && <FieldError errors={[fieldState.error]} />}
            </Field>
          )}
        />

        <Controller
          name="address"
          control={form.control}
          render={({ field, fieldState }) => (
            <Field data-invalid={fieldState.invalid}>
              <FieldLabel htmlFor="address">Address (optional)</FieldLabel>
              <Textarea
                {...field}
                id="address"
                placeholder="Your address"
                autoComplete="street-address"
                className="min-h-20 resize-none"
                aria-invalid={fieldState.invalid}
              />
              {fieldState.invalid && <FieldError errors={[fieldState.error]} />}
            </Field>
          )}
        />

        <Button type="submit" className="w-full" disabled={form.formState.isSubmitting}>
          Create account
        </Button>
      </form>

      <p className="mt-4 text-center text-xs text-muted-foreground">
        By signing up, you agree to our{" "}
        <Link to="/terms" className="underline underline-offset-4 hover:text-foreground">
          Terms &amp; Privacy
        </Link>
        .
      </p>

      <p className="mt-4 text-center text-sm text-muted-foreground">
        Already have an account?{" "}
        <Link to="/login" className="font-medium text-primary underline underline-offset-4">
          Log in
        </Link>
      </p>
    </>
  );
}
```

### Schema আলাদা ফাইলে

Schema আর type component ফাইল থেকে সরিয়ে একই folder-এ `auth.schema.ts`-এ রাখা হয়েছে:

```
components/modules/Authentication/
├── auth.schema.ts      ← registerSchema, loginSchema, আর তাদের type
├── LoginForm.tsx
└── RegisterForm.tsx
```

কারণ:
- Component ফাইল শুধু UI নিয়ে থাকে, ছোট আর পড়তে সহজ হয়।
- পরে API call বা অন্য কোথাও একই type (`RegisterFormValues`) লাগলে এখান থেকে import করা যায়।
- `.ts` ফাইল, কারণ এতে কোনো JSX নেই (৫ নম্বর নোট)।

`auth.schema.ts`-এ উপরের ৫ নম্বর section-এর register schema আর নিচের login schema দুটোই থাকবে।

### যা ঠিক বা যোগ করা হয়েছে

> - **`PasswordInput`-এ `{...field}`:** `field`-এর ভেতরে `ref`-ও থাকে। React 19-এ `ref` সাধারণ prop হিসেবে `PasswordInput` হয়ে ভেতরের `Input`-এ পৌঁছায় (`dark-theme-toggle.md`-এর forwardRef অংশ)। তাই error হলে React Hook Form ঠিক সেই field-এ cursor নিয়ে যেতে পারে।
> - **১১ নম্বর নোটের `required`, `minLength` মুছে দেওয়া হয়েছে:** এখন validation করছে Zod। `noValidate` দেওয়ায় browser এগুলো উপেক্ষাও করে। রেখে দিলে বোঝা যায় না নিয়ম আসলে কোথায় লেখা।
> - **`noValidate` (ঠিকই ছিল):** এটা না দিলে `type="email"`-এর কারণে browser নিজের ইংরেজি popup দেখাত, আর Zod-এর message আসার আগেই submit আটকে দিত।
> - **`disabled={form.formState.isSubmitting}`:** পরে API call যোগ হলে submit চলার সময় button বন্ধ থাকবে, user দুইবার চেপে দুটো account বানাতে পারবে না।
> - **Address-এ `autoComplete="street-address"`:** browser ঠিকানা autofill করতে পারবে।

> 💡 **Optional field খালি থাকলে `""` যায়:** user phone না দিলে `onSubmit`-এ `phone: ""` আসবে। Backend যদি খালি string না নেয়, তাহলে API-তে পাঠানোর আগে খালি field সরিয়ে দিতে হবে। এটা API connect করার সময় দেখা হবে।

---

## ৭. Login form — পুরো কোড

```ts
// src/components/modules/Authentication/auth.schema.ts (register schema-র নিচে)
export const loginSchema = z.object({
  email: z.email({ error: "Invalid email address format." }),
  password: z.string().min(1, { error: "Password is required." }),
});

export type LoginFormValues = z.infer<typeof loginSchema>;
```

> Login-এ password-এর জন্য শুধু "খালি না" নিয়ম, কোনো uppercase/number শর্ত নেই। কারণ login-এ নতুন password বানানো হচ্ছে না, পুরনোটা মেলানো হচ্ছে। সেটা backend করবে। এখানে শক্ত নিয়ম দিলে পুরনো নিয়মে বানানো account-এর user login-ই করতে পারবে না। (আপনার কোডে এটা ঠিকই ছিল।)

```tsx
// src/components/modules/Authentication/LoginForm.tsx
import GoogleIcon from "@/components/icons/GoogleIcon";
import { Button } from "@/components/ui/button";
import { Field, FieldError, FieldLabel } from "@/components/ui/field";
import { Input } from "@/components/ui/input";
import { PasswordInput } from "@/components/ui/password-input";
import { Separator } from "@/components/ui/separator";
import { zodResolver } from "@hookform/resolvers/zod";
import { Controller, useForm } from "react-hook-form";
import { Link } from "react-router";
import { loginSchema, type LoginFormValues } from "./auth.schema";

export default function LoginForm() {
  const form = useForm<LoginFormValues>({
    resolver: zodResolver(loginSchema),
    defaultValues: {
      email: "",
      password: "",
    },
  });

  function onSubmit(data: LoginFormValues) {
    console.log(data); // পরে এখানে API call হবে
  }

  return (
    <>
      <h1 className="text-3xl font-bold text-foreground">Welcome back</h1>
      <p className="mt-2 text-sm text-muted-foreground">Log in to manage your bookings.</p>

      <Button variant="outline" className="mt-6 w-full gap-2" type="button">
        <GoogleIcon />
        Continue with Google
      </Button>

      <div className="my-5 flex items-center gap-3">
        <Separator className="flex-1" />
        <span className="text-xs text-muted-foreground">or log in with email</span>
        <Separator className="flex-1" />
      </div>

      <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-5" noValidate>
        <Controller
          name="email"
          control={form.control}
          render={({ field, fieldState }) => (
            <Field data-invalid={fieldState.invalid}>
              <FieldLabel htmlFor="email">Email</FieldLabel>
              <Input
                {...field}
                id="email"
                type="email"
                placeholder="you@example.com"
                autoComplete="email"
                aria-invalid={fieldState.invalid}
              />
              {fieldState.invalid && <FieldError errors={[fieldState.error]} />}
            </Field>
          )}
        />

        <Controller
          name="password"
          control={form.control}
          render={({ field, fieldState }) => (
            <Field data-invalid={fieldState.invalid}>
              <div className="flex items-center justify-between">
                <FieldLabel htmlFor="password">Password</FieldLabel>
                <Link to="/forgot-password" className="text-xs font-medium text-primary underline underline-offset-4">
                  Forgot password?
                </Link>
              </div>
              <PasswordInput
                {...field}
                id="password"
                autoComplete="current-password"
                aria-invalid={fieldState.invalid}
              />
              {fieldState.invalid && <FieldError errors={[fieldState.error]} />}
            </Field>
          )}
        />

        <Button type="submit" className="w-full" disabled={form.formState.isSubmitting}>
          Log in
        </Button>
      </form>

      <p className="mt-6 text-center text-sm text-muted-foreground">
        New to Phovia Tour?{" "}
        <Link to="/register" className="font-medium text-primary underline underline-offset-4">
          Create an account
        </Link>
      </p>
    </>
  );
}
```

> ⚠️ সংশোধন (Login):
> - **"Forgot password?" link হারিয়ে গিয়েছিল:** দেওয়া কোডে ডিজাইনের এই link ছিল না। Label-এর পাশে আবার যোগ করা হয়েছে।
> - **`autoComplete="current-password"` ছিল না:** এটা দিলে browser সেভ করা password autofill করে। Register-এ `new-password` (নতুন password suggest করে), Login-এ `current-password`।
> - **`placeholder="Password"` সরানো হয়েছে:** উপরেই "Password" label আছে, তাই একই লেখা placeholder-এ দেওয়া অপ্রয়োজনীয়।
> - **`/signup` থেকে `/register`:** route-এর নাম Navbar-এর সাথে মেলানো (১১ নম্বর নোট)।

---

## ৮. Check-list

- [ ] খালি form submit করলে প্রতিটা required field-এর নিচে লাল message দেখায়
- [ ] দুই password আলাদা দিলে, অন্য field ভুল থাকলেও, "Passwords do not match" দেখায়
- [ ] Phone খালি রাখলে কোনো error নেই। `012…` দিলে error, `017…` দিলে ঠিক
- [ ] Error ঠিক করে টাইপ করলে message সাথে সাথে চলে যায় (প্রথম submit-এর পরে React Hook Form প্রতি অক্ষরে আবার যাচাই করে)
- [ ] সব ঠিক থাকলে Console-এ `data` object আসে, page reload হয় না
- [ ] Console-এ `changing an uncontrolled input to be controlled` warning নেই
- [ ] Dark mode-এ error message আর লাল border পড়া যায়