# Frontend Notes — ১৮: Tour Type মোছা — Reusable Delete Confirmation (Alert Dialog)

১৭ নম্বর নোটে tour type যোগ করা আর তালিকা দেখানো হয়েছে। এবার মোছা। কিন্তু মোছা এমন কাজ যা ফেরানো যায় না, তাই ভুল করে click করলেই যেন মুছে না যায়, সেজন্য আগে একটা নিশ্চিতকরণ (confirmation) dialog: "সত্যিই মুছবেন?"

আর যেহেতু পরে tour, division, booking, user অনেক কিছুই মুছতে হবে, dialog-টা বানাব **reusable**: একবার বানিয়ে সব জায়গায় ব্যবহার।

নোট দুই ভাগে:
1. **ধাপে ধাপে (ধাপ ১–৭):** teacher-এর ক্রমে, আপনার কোড দিয়ে।
2. **উন্নতি আর চূড়ান্ত কোড:** teacher-এর কোডে যা বাকি ছিল (dialog-এর লেখা সবসময় "Delete chat?", error-এ loading toast আটকে থাকা ইত্যাদি), আর আপনার "async-এর জন্য common file"-এর ধারণা।

প্রতিটা ধাপে 🎯 লক্ষ্য, 🛠️ কাজ, 💡 কেন, ✅ যাচাই।

> নোট: আপনার project-এর shadcn version নতুন (`radix-nova`)। এতে `AlertDialogContent`-এ `size="sm"`, `AlertDialogMedia` (icon-এর জায়গা), আর `AlertDialogAction`/`AlertDialogCancel`-এ `variant` আছে, যেগুলো পুরনো shadcn-এ ছিল না। পুরনো docs বা ভিডিওর কোড দেখে এগুলো সরাবেন না।

---

## ভূমিকা

### ধাপের মানচিত্র

| ভাগ | ধাপ | শেষে কী হবে |
|---|---|---|
| **ক. Dialog বানানো** | ১–৪ | যেকোনো button-কে মুড়ে "নিশ্চিত?" dialog দেখানো যায় |
| **খ. আসলে মোছা** | ৫–৬ | Dialog-এ Delete চাপলে tour type মুছে যায়, তালিকা আপডেট হয় |
| **গ. কী বাকি** | ৭ | সমস্যা চেনা |
| **উন্নতি ও চূড়ান্ত কোড** | — | Dynamic লেখা, error, মোছা শেষ না হওয়া পর্যন্ত dialog খোলা, reusable toast helper |

### শেষে যে ফাইলগুলো থাকবে

```
src/
├── components/
│   ├── DeleteConfirmation.tsx        ← reusable dialog
│   └── ui/alert-dialog.tsx           ← shadcn
├── pages/Admin/
│   └── AddTourType.tsx               ← Delete button এখানে
├── redux/features/Tour/
│   └── tour.api.ts                   ← removeTourType
└── utils/
    └── runWithToast.ts               ← (উন্নতি) async + toast-এর common helper
```

---

# ভাগ ক: Dialog বানানো

## ধাপ ১ — Alert Dialog install

🎯 **লক্ষ্য:** নিশ্চিতকরণের জন্য shadcn-এর component আনা।

🛠️ **কাজ:**

```sh
bunx --bun shadcn@latest add alert-dialog
```

### 💡 Alert Dialog আর সাধারণ Dialog-এর পার্থক্য

shadcn-এ দুটো আছে: `dialog` আর `alert-dialog`। দেখতে প্রায় একই, কিন্তু উদ্দেশ্য আলাদা:

| | `Dialog` | `AlertDialog` |
|---|---|---|
| কাজ | যেকোনো কিছু দেখানো (form, ছবি, তথ্য) | **গুরুত্বপূর্ণ সিদ্ধান্ত** নেওয়ানো |
| বাইরে click করলে | বন্ধ হয়ে যায় | **বন্ধ হয় না**, user-কে Cancel বা Delete-এর একটা বেছে নিতেই হবে |
| Screen reader | "dialog" | "alertdialog" (জরুরি বার্তা হিসেবে পড়ে) |
| উদাহরণ | Edit form, ছবি বড় করে দেখা | মোছা, logout, payment নিশ্চিত করা |

মোছা ফেরানো যায় না, তাই ভুল করে বাইরে click করে বন্ধ হয়ে যাওয়া বা পাশ কাটিয়ে যাওয়ার সুযোগ রাখা হয় না: **AlertDialog**।

✅ **যাচাই:** `src/components/ui/alert-dialog.tsx` এসেছে।

---

## ধাপ ২ — shadcn-এর উদাহরণ থেকে `DeleteConfirmation`

🎯 **লক্ষ্য:** shadcn docs-এর "destructive" উদাহরণটা নিজের component বানিয়ে page-এ বসানো।

🛠️ **কাজ:** docs-এর উদাহরণ কপি করে নাম `DeleteConfirmation`:

```tsx
// src/components/DeleteConfirmation.tsx  (প্রথম রূপ — shadcn-এর উদাহরণ হুবহু)
import { Trash2Icon } from "lucide-react";
import {
  AlertDialog,
  AlertDialogAction,
  AlertDialogCancel,
  AlertDialogContent,
  AlertDialogDescription,
  AlertDialogFooter,
  AlertDialogHeader,
  AlertDialogMedia,
  AlertDialogTitle,
  AlertDialogTrigger,
} from "@/components/ui/alert-dialog";
import { Button } from "@/components/ui/button";

export function DeleteConfirmation() {
  return (
    <AlertDialog>
      <AlertDialogTrigger asChild>
        <Button variant="destructive">Delete Chat</Button>
      </AlertDialogTrigger>
      <AlertDialogContent size="sm">
        <AlertDialogHeader>
          <AlertDialogMedia className="bg-destructive/10 text-destructive dark:bg-destructive/20 dark:text-destructive">
            <Trash2Icon />
          </AlertDialogMedia>
          <AlertDialogTitle>Delete chat?</AlertDialogTitle>
          <AlertDialogDescription>This will permanently delete this chat conversation.</AlertDialogDescription>
        </AlertDialogHeader>
        <AlertDialogFooter>
          <AlertDialogCancel variant="outline">Cancel</AlertDialogCancel>
          <AlertDialogAction variant="destructive">Delete</AlertDialogAction>
        </AlertDialogFooter>
      </AlertDialogContent>
    </AlertDialog>
  );
}
```

💡 **অংশগুলো:**

| Component | কাজ |
|---|---|
| `AlertDialog` | পুরো dialog-এর মূল; খোলা না বন্ধ, সেই অবস্থা রাখে |
| `AlertDialogTrigger` | যেটা চাপলে dialog খোলে |
| `AlertDialogContent` | মাঝখানে ভাসা box (পেছনের পর্দা/overlay সহ)। `size="sm"`: ছোট, Cancel আর Delete পাশাপাশি সমান দুই ভাগে |
| `AlertDialogMedia` | উপরের icon-এর ঘর (নতুন shadcn-এর অংশ) |
| `AlertDialogTitle`, `AlertDialogDescription` | প্রশ্ন আর ব্যাখ্যা; screen reader এগুলো পড়ে শোনায় |
| `AlertDialogCancel` | বন্ধ করে, কিছু করে না |
| `AlertDialogAction` | মূল কাজ (Delete), click করলে dialog নিজে থেকে বন্ধও হয় |

Page-এ পুরনো Delete button-এর জায়গায় বসান:

```tsx
// src/pages/Admin/AddTourType.tsx
import { DeleteConfirmation } from "@/components/DeleteConfirmation";

<DeleteConfirmation />
```

💡 **ফাইল কোথায়:** `components/DeleteConfirmation.tsx`, অর্থাৎ `ui/`-এ না, `modules/`-এও না। কারণ এটা কোনো এক page-এর অংশ না (তাই `modules` না), আবার shadcn-এর কাঁচা টুকরোও না (তাই `ui` না); এটা পুরো app-এ ব্যবহারের তৈরি একটা component, ঠিক `Logo`-এর মতো (৬ নম্বর নোট)।

✅ **যাচাই:** প্রতিটা row-এ লাল "Delete Chat" button, চাপলে dialog খোলে; বাইরে click করলে বন্ধ হয় **না**; Cancel চাপলে বন্ধ হয়।

---

## ধাপ ৩ — Trigger button বাইরে থেকে দেওয়া: `children`

🎯 **লক্ষ্য:** "Delete Chat" button ভেতরে বাঁধা না রেখে, যে ব্যবহার করবে সে নিজের button দেবে।

### 💡 সমস্যা

এখন button `DeleteConfirmation`-এর **ভেতরে** লেখা: সবসময় লাল, সবসময় "Delete Chat"। কিন্তু আমাদের tour type-এর তালিকায় লাগবে ছোট outline icon button (🗑️), আবার অন্য page-এ হয়তো লেখাসহ বড় button। Button-টা যে ব্যবহার করবে তার হাতে থাকা দরকার।

### 💡 আপনার প্রশ্ন: props আর children-এর পার্থক্য

বাইরে থেকে কিছু পাঠানোর দুই উপায়:

```tsx
// ১. Prop হিসেবে (attribute-এর মতো, tag-এর ভেতরে নাম দিয়ে)
<DeleteConfirmation trigger={<Button>Delete</Button>} />

// ২. Children হিসেবে (খোলা আর বন্ধ tag-এর মাঝখানে)
<DeleteConfirmation>
  <Button>Delete</Button>
</DeleteConfirmation>
```

| | Props | Children |
|---|---|---|
| কোথায় লেখা | Tag-এর ভেতরে, `নাম={মান}` | খোলা `<X>` আর বন্ধ `</X>`-এর **মাঝখানে** |
| Component-এ পাওয়া যায় | নিজের দেওয়া নামে (`trigger`, `title`, `onConfirm`) | সবসময় **`children`** নামে |
| সাধারণত কী পাঠানো হয় | **Setting**: লেখা, সংখ্যা, function, true/false | **যা ভেতরে দেখাবে**: JSX, অন্য component |
| HTML-এর সাথে তুলনা | `<img src="..." alt="...">`-এর `src`, `alt` | `<button>এই লেখাটা</button>`-এর "এই লেখাটা" |

আসলে **`children`-ও একটা prop**, শুধু বিশেষ একটা: React নিজে থেকে tag-এর মাঝখানের সবকিছু `children` নামে পাঠায়। তাই দুটো উপায়েই একই কাজ হয়, পার্থক্য লেখার ধরনে।

**কোনটা কখন:** একটাই "ভেতরের জিনিস" থাকলে **children** স্বাভাবিক লাগে (পড়লে মনে হয় button-টাকে dialog দিয়ে মুড়ে দিলাম)। একাধিক JSX জায়গা থাকলে (যেমন trigger আলাদা, icon আলাদা) নাম দিয়ে **props**। আর লেখা, function, setting সবসময় props। Teacher এখানে button-কে children হিসেবে নিয়েছেন, আর `onConfirm` (function) props হিসেবে, ঠিক এই নিয়মে।

### 🛠️ কাজ

```tsx
import type { ReactNode } from "react";

interface IProps {
  children: ReactNode;
}

export function DeleteConfirmation({ children }: IProps) {
  return (
    <AlertDialog>
      <AlertDialogTrigger asChild>{children}</AlertDialogTrigger>
      {/* ...বাকি আগের মতো */}
    </AlertDialog>
  );
}
```

```tsx
// page-এ: নিজের button মুড়ে দেওয়া
<DeleteConfirmation>
  <Button variant="outline" size="icon" type="button" className="size-10 rounded-lg">
    <Trash2 className="size-4.25" />
  </Button>
</DeleteConfirmation>
```

💡 **`ReactNode`:** React যা কিছু দেখাতে পারে (JSX, লেখা, সংখ্যা, `null`) তার type। children-এর জন্য এটাই সাধারণ type।

💡 **`AlertDialogTrigger asChild` কেন জরুরি:** `AlertDialogTrigger` নিজে একটা `<button>` বানায়। আমাদের children-ও একটা `<Button>` (`<button>`)। `asChild` না দিলে হতো **button-এর ভেতরে button**, যেটা HTML-এ নিষেধ (৯ নম্বর নোটের ModeToggle-এর সেই error)। `asChild` দিলে Trigger নিজের button বানায় না, তার "চাপলে dialog খোলো" কাজটা ভেতরের আমাদের button-এর গায়ে লাগিয়ে দেয়।

✅ **যাচাই:** Row-এ এখন আপনার outline 🗑️ button, চাপলে dialog খোলে; console-এ `<button> cannot be a descendant of <button>` warning নেই।

---

## ধাপ ৪ — Delete চাপলে কী হবে: `onConfirm`

🎯 **লক্ষ্য:** Dialog-এর Delete চাপলে বাইরের দেওয়া কাজটা চালানো।

### 💡 মূল ধারণা: dialog জানে না কী মুছছে

Reusable হওয়ার শর্ত: `DeleteConfirmation` নিজে **কখনো** জানবে না সে tour type মুছছে নাকি booking। সে শুধু জানে "Delete চাপলে আমাকে যে function দেওয়া হয়েছে সেটা চালাব"। কী মুছবে, কোন API ডাকবে, সেটা ঠিক করে যে ব্যবহার করছে (page)।

```
Page (AddTourType)                     DeleteConfirmation
──────────────────                     ──────────────────
"এই function-টা রাখো,                  "ঠিক আছে, Delete চাপলে
 Delete চাপলে চালিও"  ── onConfirm ──→   চালিয়ে দেব। ভেতরে কী আছে
 (tour type মোছার কোড)                   জানি না, জানার দরকারও নেই"
```

Teacher-এর কথা: `removeTourType` (API) যদি সরাসরি `DeleteConfirmation`-এর ভেতরে ডাকি, তাহলে সেটা শুধু tour type মুছতে পারবে, আর reusable থাকবে না।

### 🛠️ কাজ

```tsx
interface IProps {
  children: ReactNode;
  onConfirm: () => void;   // কিছু নেয় না, কিছু ফেরত দেয় না, এমন function
}

export function DeleteConfirmation({ children, onConfirm }: IProps) {
  const handleConfirm = () => {
    onConfirm();
  };

  return (
    <AlertDialog>
      <AlertDialogTrigger asChild>{children}</AlertDialogTrigger>
      <AlertDialogContent size="sm">
        {/* ...header আগের মতো */}
        <AlertDialogFooter>
          <AlertDialogCancel variant="outline">Cancel</AlertDialogCancel>
          <AlertDialogAction variant="destructive" onClick={handleConfirm}>
            Delete
          </AlertDialogAction>
        </AlertDialogFooter>
      </AlertDialogContent>
    </AlertDialog>
  );
}
```

> নোট: এখানে `handleConfirm` শুধু `onConfirm()` ডাকছে, বাড়তি কিছু করছে না। তাই `onClick={onConfirm}` লিখলেও একই কাজ হতো। তবে আলাদা function রাখলে পরে ভেতরে কিছু যোগ করা সহজ (উন্নতি ৩-এ ঠিক এটাই লাগবে)।

> নোট: shadcn-এর উদাহরণে description-এর ভেতরে `<a href="#">Settings</a>` ছিল। আমাদের নিয়মে `<a href>` না, আর এখানে লাগেও না। আপনি পরের version-এ সরিয়ে দিয়েছেন, ঠিক করেছেন।

✅ **যাচাই:** সাময়িকভাবে page-এ `onConfirm={() => console.log("confirmed")}` দিন। Delete চাপলে console-এ লেখা আসে আর dialog বন্ধ হয়; Cancel চাপলে আসে না।

---

# ভাগ খ: আসলে মোছা

## ধাপ ৫ — `removeTourType` endpoint

🎯 **লক্ষ্য:** Backend-এ মোছার API।

🛠️ **কাজ:**

```ts
// src/redux/features/Tour/tour.api.ts  (endpoints-এর ভেতরে)
removeTourType: builder.mutation({
  query: (tourTypeId) => ({
    url: `/tour/tour-types/${tourTypeId}`,
    method: "DELETE",
  }),
  invalidatesTags: ["TOUR"],
}),
```

আর export-এ `useRemoveTourTypeMutation`।

💡 **লাইনগুলোর মানে:**
- **`` `/tour/tour-types/${tourTypeId}` ``:** backtick (`` ` ``) দিয়ে লেখা string-এর ভেতরে `${}` দিয়ে variable বসানো যায় (template literal)। id `665a…` হলে URL হয় `/tour/tour-types/665a…`। REST-এর নিয়ম: **কোনটা** মুছব সেটা URL-এ থাকে, তাই body লাগে না।
- **`method: "DELETE"`:** HTTP-তে মোছার method।
- **`invalidatesTags`:** মোছার পরে তালিকা নিজে আবার আসে (১৭ নম্বর নোটের ধাপ ৯)। Tag-এর নাম `addTourType` আর `getTourTypes`-এ যা আছে, এখানেও **হুবহু তাই** হতে হবে। ১৭ নম্বর নোটের পরামর্শমতো `"TOUR_TYPE"` করে থাকলে এখানেও `"TOUR_TYPE"`; নাহলে তিন জায়গাতেই `"TOUR"`। না মিললে মোছা হবে, কিন্তু তালিকা থেকে যাবে।

✅ **যাচাই:** Postman-এ একটা tour type-এর `_id` দিয়ে `DELETE /tour/tour-types/:id` কাজ করে (admin token দিয়ে)।

---

## ধাপ ৬ — Page-এ জোড়া লাগানো

🎯 **লক্ষ্য:** Dialog-এর Delete চাপলে ঐ row-এর tour type মুছে যাবে।

🛠️ **কাজ (teacher-এর কোড):**

```tsx
// src/pages/Admin/AddTourType.tsx
const [removeTourType] = useRemoveTourTypeMutation();

const handleRemoveTourType = async (tourId: string) => {
  const toastId = toast.loading("Removing...");
  try {
    const res = await removeTourType(tourId).unwrap();
    if (res.success) {
      toast.success("Removed", { id: toastId });
    }
  } catch (err) {
    console.error(err);
  }
};
```

```tsx
// তালিকার প্রতিটা row-এ
<DeleteConfirmation onConfirm={() => handleRemoveTourType(item._id)}>
  <Button variant="outline" size="icon" type="button" className="size-10 rounded-lg">
    <Trash2 className="size-4.25" />
  </Button>
</DeleteConfirmation>
```

### 💡 `onConfirm={() => handleRemoveTourType(item._id)}` — arrow function কেন

এটা এই নোটের সবচেয়ে জরুরি লাইন। তিনভাবে লেখার চেষ্টা করে দেখি:

```tsx
// ❌ ১. বিপজ্জনক
onConfirm={handleRemoveTourType(item._id)}
```
এখানে `()` দিয়ে function **এখনই ডাকা হচ্ছে**, page render হওয়ার সময়েই। অর্থাৎ page খুললেই তালিকার **প্রতিটা** tour type মোছার request চলে যাবে, কেউ কিছু চাপার আগেই! আর `onConfirm` পাবে function-এর ফলাফল (একটা Promise), function না।

```tsx
// ❌ ২. id যায় না
onConfirm={handleRemoveTourType}
```
Function ঠিকই পাঠানো হলো (ডাকা হয়নি), কিন্তু `DeleteConfirmation` সেটা ডাকে `onConfirm()`, কোনো id ছাড়া। তাই কোনটা মুছবে জানবে না।

```tsx
// ✅ ৩. ঠিক
onConfirm={() => handleRemoveTourType(item._id)}
```
একটা **নতুন ছোট function** পাঠানো হলো, যেটা এখনই চলে না। পরে Delete চাপলে `DeleteConfirmation` এটা ডাকে, আর তখন এটা ভেতর থেকে `handleRemoveTourType`-কে **এই row-এর** `item._id` দিয়ে ডাকে। প্রতিটা row-এর নিজের arrow function, নিজের id মনে রাখা।

এক কথায়: **`fn`** = function পাঠানো, **`fn()`** = এখনই চালানো, **`() => fn(x)`** = পরে চালানোর জন্য, সাথে x নিয়ে।

### 💡 `toast.loading` → `toast.success` একই `id` দিয়ে

১৩ নম্বর নোটের Verify page-এর মতো: প্রথমে "Removing..." ঘুরতে থাকে, সফল হলে **একই toast** (একই `id`) সবুজ "Removed"-এ বদলে যায়। নতুন toast আসে না।

✅ **যাচাই:**
- 🗑️ চাপুন → dialog → Delete → "Removing..." → "Removed", আর reload ছাড়াই row তালিকা থেকে চলে যায়।
- Cancel চাপলে কিছু মোছে না।

---

# ভাগ গ: কী বাকি

## ধাপ ৭ — সমস্যা চেনা

🎯 **লক্ষ্য:** Teacher-এর কোড "চলে", কিন্তু ব্যবহারে কোথায় ঝামেলা, খুঁজে বের করা।

✅ **যাচাই (এবং সমস্যা):**

| # | পরীক্ষা | কী হয় | সমস্যা |
|---|---|---|---|
| ১ | যেকোনো row-এর 🗑️ চাপুন | Dialog-এ লেখা **"Delete chat?"** | লেখা shadcn-এর উদাহরণ থেকে রয়ে গেছে, আর সব জায়গায় একই। কোনটা মুছছি সেটা বলে না |
| ২ | Backend বন্ধ করে বা এমন tour type মুছুন যেটা কোনো tour-এ ব্যবহৃত (backend মানা করলে) | "Removing..." **চিরকাল ঘুরতে থাকে** | `catch`-এ শুধু `console.error`, toast বন্ধ হয় না, user কারণ জানে না (১৩ নম্বর নোটে teacher-এর Verify কোডের একই ভুল) |
| ৩ | Slow 4G দিয়ে Delete চাপুন | Dialog **সাথে সাথে বন্ধ**, row কিছুক্ষণ থেকে যায় | `AlertDialogAction` click হওয়ামাত্র dialog বন্ধ করে, মোছা শেষ হওয়ার অপেক্ষা করে না। Error হলে user আর dialog-ও দেখে না |
| ৪ | Delete-এর পরে দ্রুত আরেকবার 🗑️ → Delete | একই জিনিস মোছার দুটো request | চলার সময় কিছু বন্ধ (disabled) হয় না |
| ৫ | Row-এর 🗑️-এর `aria-label` | "Delete Short tour — in use by 46 tours" সব row-এ | Hardcode (১৭ নম্বর নোট) |
| ৬ | Code-এ `data?.map((item: { name: string; _id: string }) => …` | চলে | Type প্রতিবার হাতে লেখা; `ITourType` (১৭ নম্বর নোট) ব্যবহার হচ্ছে না |
| ৭ | `if (res.success)` | চলে | `.unwrap()` সফল না হলে এই লাইনে আসেই না (১৩ নম্বর নোট), তাই অপ্রয়োজনীয় |

আর আপনার নোটের শেষ লাইন: **"async await-এর জন্য একটা common file বানিয়ে reuse করতে পারি।"** একদম ঠিক ধারণা: loading toast → try → success toast → catch → error toast, এই একই নকশা add, delete, edit সব জায়গায় লাগবে। উন্নতি ৪-এ এটা বানানো হলো।

---

# উন্নতি

## উন্নতি ১ — Dialog-এর লেখা বাইরে থেকে (props)

ধাপ ৩-এর নিয়ম অনুযায়ী লেখা (setting) যায় **props** দিয়ে। সাথে **default** মান, যাতে না দিলেও কিছু একটা অর্থপূর্ণ দেখায়:

```tsx
interface IProps {
  children: ReactNode;
  onConfirm: () => void | Promise<unknown>;
  title?: string;
  description?: ReactNode;
  confirmText?: string;
}

export function DeleteConfirmation({
  children,
  onConfirm,
  title = "Delete this item?",
  description = "This action can't be undone.",
  confirmText = "Delete",
}: IProps) { ... }
```

```tsx
// ব্যবহার: কোনটা মুছছি স্পষ্ট
<DeleteConfirmation
  title={`Delete "${item.name}"?`}
  description="This tour type will be removed permanently. Tours still using it can't be saved until you pick another type."
  onConfirm={() => handleRemoveTourType(item)}
>
```

- **`title?`:** `?` মানে optional; না দিলে `= "Delete this item?"`-এর default মান।
- **`description`-এর type `ReactNode`:** শুধু লেখা না, চাইলে `<strong>` দিয়ে নাম মোটা করেও পাঠানো যায়।
- `DeleteConfirmation` এখনো জানে না কী মুছছে; শুধু যা লেখা পায় তা দেখায়। Reusable থাকল।

## উন্নতি ২ — Error হলে toast বন্ধ, কারণ দেখানো

```ts
catch (err) {
  toast.error(getErrorMessage(err, "Couldn't remove the tour type."), { id: toastId });
}
```

Loading toast খুললে সবসময় success **বা** error দিয়ে বন্ধ করতে হয়। Backend কারণ পাঠালে (যেমন "This tour type is used by 3 tours") সেটাই দেখায় (`getErrorMessage`, ১৭ নম্বর নোটের উন্নতি ৩)।

## উন্নতি ৩ — মোছা শেষ না হওয়া পর্যন্ত dialog খোলা রাখা

### 💡 সমস্যা (ধাপ ৭-এর #৩)

`AlertDialogAction` চাপলে Radix নিজে থেকে dialog বন্ধ করে দেয়, আমাদের `onConfirm` শেষ হোক বা না হোক। তাই:
- মোছা চলার সময় কোনো "Deleting…" দেখা যায় না।
- Error হলে dialog আগেই বন্ধ, user-কে আবার 🗑️ চেপে খুলতে হয়।

### 💡 সমাধান

1. **Dialog-এর খোলা/বন্ধ নিজের হাতে নেওয়া (controlled):** `open` আর `onOpenChange` দিয়ে।
2. **Delete চাপলে Radix-কে বন্ধ করতে না দেওয়া:** click-এর `event.preventDefault()`। (Radix-এর নিয়ম: Action-এর click-এ `preventDefault` দিলে সে নিজে বন্ধ করে না।)
3. `onConfirm` **শেষ হওয়া পর্যন্ত অপেক্ষা** (`await`), এর মধ্যে button-এ "Deleting…" আর দুটো button-ই disabled।
4. সফল হলে তবেই নিজে বন্ধ করা। ব্যর্থ হলে খোলা থাকে (error toast ইতিমধ্যে দেখানো), user চাইলে আবার চেষ্টা বা Cancel।

```tsx
const [open, setOpen] = useState(false);
const [isPending, setIsPending] = useState(false);

const handleConfirm = async (event: MouseEvent<HTMLButtonElement>) => {
  event.preventDefault();          // Radix: এখনই বন্ধ কোরো না
  setIsPending(true);
  try {
    await onConfirm();             // মোছা শেষ হওয়া পর্যন্ত অপেক্ষা
    setOpen(false);                // সফল → বন্ধ
  } catch {
    // ব্যর্থ → dialog খোলা থাকে; error-এর বার্তা যে onConfirm দিয়েছে সে দেখিয়েছে
  } finally {
    setIsPending(false);
  }
};
```

আর চলার সময় Escape চেপে বা Cancel দিয়ে বন্ধ করা আটকাতে:

```tsx
<AlertDialog open={open} onOpenChange={(next) => !isPending && setOpen(next)}>
```

💡 **এজন্যই `onConfirm`-এর type এখন `() => void | Promise<unknown>`:** async function (Promise ফেরত দেয়) দিলে dialog অপেক্ষা করে; সাধারণ function দিলে সাথে সাথে বন্ধ হয়। দুটোই চলে।

💡 **ব্যর্থ হলে dialog কীভাবে জানে:** page-এর handler error ধরে toast দেখায়, তারপর error আবার ছুড়ে দেয় (`throw`)। তাতে `await onConfirm()` ব্যর্থ হয়, `setOpen(false)`-এ পৌঁছায় না, dialog খোলা থাকে। পরের উন্নতির helper এটা নিজেই করে।

## উন্নতি ৪ — আপনার ধারণা: async-এর জন্য common helper

### 💡 একই নকশা বারবার

```ts
const toastId = toast.loading("...");
try {
  await something.unwrap();
  toast.success("...", { id: toastId });
} catch (err) {
  toast.error(getErrorMessage(err, "..."), { id: toastId });
}
```

Add tour type, delete, পরে edit, tour যোগ, booking বাতিল… সব জায়গায় এই ৭-৮ লাইন। একটা function-এ রাখলে প্রতিবার এক লাইন:

```ts
// src/utils/runWithToast.ts
import { getErrorMessage } from "@/utils/getErrorMessage";
import { toast } from "sonner";

interface IToastMessages {
  loading: string;
  success: string;
  error: string; // backend কোনো message না দিলে এটা দেখাবে
}

export const runWithToast = async <T>(promise: Promise<T>, messages: IToastMessages): Promise<T> => {
  const toastId = toast.loading(messages.loading);
  try {
    const result = await promise;
    toast.success(messages.success, { id: toastId });
    return result;
  } catch (err) {
    toast.error(getErrorMessage(err, messages.error), { id: toastId });
    throw err; // যে ডেকেছে সে-ও যেন জানে ব্যর্থ হয়েছে (যেমন dialog খোলা রাখতে)
  }
};
```

ব্যবহার:

```ts
const handleRemoveTourType = (item: ITourType) =>
  runWithToast(removeTourType(item._id).unwrap(), {
    loading: `Removing "${item.name}"…`,
    success: `"${item.name}" removed`,
    error: "Couldn't remove the tour type.",
  });
```

💡 **লাইনগুলোর মানে:**
- **`<T>` (generic):** "যেকোনো ধরনের ফলাফল"। Delete-এ ফলাফল এক রকম, add-এ আরেক রকম; `T` সেই ধরন মনে রাখে, আর `return result` একই ধরন ফেরত দেয়। তাই চাইলে `const res = await runWithToast(...)` লিখে `res.data` ব্যবহার করা যায়, type সহ।
- **প্রথম parameter একটা Promise:** `removeTourType(item._id).unwrap()` লেখামাত্র request শুরু হয়ে যায়, helper শুধু তার ফলাফলের অপেক্ষা করে আর toast সামলায়।
- **`throw err` আবার:** helper error দেখিয়ে দিল, কিন্তু যে ডেকেছে তার-ও জানা দরকার হতে পারে (উন্নতি ৩-এর dialog)। আর চাইলে বাইরে আলাদা কিছু করা যায় (যেমন form-এ `setError`)।
- **`getErrorMessage`:** ১৭ নম্বর নোটের উন্নতি ৩-এ বানানো helper (`utils/getErrorMessage.ts`)। এখনো না বানিয়ে থাকলে আগে সেটা বানান।

> নোট — sonner-এর নিজের `toast.promise`: sonner-এ একটা তৈরি `toast.promise(promise, { loading, success, error })` আছে, প্রায় একই কাজ করে। নিজের helper রাখার সুবিধা: আমাদের `getErrorMessage` দিয়ে backend-এর আসল error বের করা, `id` ব্যবস্থাটা ১৩ নম্বর নোটের মতো একই রাখা, আর error আবার throw করা যাতে dialog খোলা থাকে। দুটোই ঠিক; project-এ যেকোনো একটা নিয়ম ঠিক করে সব জায়গায় সেটাই ব্যবহার করুন।

> নোট: চাইলে ১৭ নম্বর নোটের `AddTourTypeForm`-এর `onSubmit`-ও এই helper দিয়ে লেখা যায়। তবে সেখানে error **input-এর নিচে** দেখানো হয়েছে (`setError`), toast-এ না; তাই সেটা যেমন আছে রেখে দেওয়াই ভালো। Helper সবচেয়ে কাজের যেখানে বার্তা toast-এই দেখানো উচিত (মোছা, অবস্থা বদলানো)।

## উন্নতি ৫ — Type আর ছোটখাটো

- `removeTourType`-এ type: `builder.mutation<IResponse<null>, string>` (ফেরত data খালি, পাঠাই id-র string)।
- `data?.map((item: { name: string; _id: string }) => …)`-এর হাতে লেখা type বাদ, ১৭ নম্বর নোটের `ITourType` দেওয়া query থেকে নিজে আসে।
- `handleRemoveTourType` এখন পুরো `item` নেয় (শুধু id না), যাতে toast-এ নাম দেখানো যায়। Parameter-এর নাম `tourId` ছিল, অথচ এটা tour-এর না, tour type-এর id; এখন `item`।
- `aria-label={`Delete ${item.name}`}`: প্রতিটা row-এর নিজের নাম।
- Icon-এর ঘরে `bg-destructive/10 … dark:bg-destructive/20 dark:text-destructive` এর বদলে আপনার project-এর `bg-destructive-soft text-destructive` token: light আর dark দুটোতেই নিজে মিলে যায়, `dark:` আলাদা লাগে না।
- `AlertDialogCancel`-এর `variant="outline"` আসলে default-ই outline (alert-dialog.tsx দেখুন), তাই না লিখলেও চলে; রেখে দিলেও ক্ষতি নেই।

---

# চূড়ান্ত কোড

> ১৭ নম্বর নোটের চূড়ান্ত কোডের ওপর ভিত্তি করে (types, `TOUR_TYPE` tag, `getErrorMessage`, loading/empty/error অবস্থা)। ওগুলো এখনো না বসিয়ে থাকলে আগে সেগুলো বসান।

### `src/redux/features/Tour/tour.api.ts`

```ts
import { baseApi } from "@/redux/baseApi";
import type { IAddTourType, IResponse, ITourType } from "@/types";

export const tourApi = baseApi.injectEndpoints({
  endpoints: (builder) => ({
    addTourType: builder.mutation<IResponse<ITourType>, IAddTourType>({
      query: (tourTypeInfo) => ({
        url: "/tour/create-tour-type",
        method: "POST",
        data: tourTypeInfo,
      }),
      invalidatesTags: ["TOUR_TYPE"],
    }),

    removeTourType: builder.mutation<IResponse<null>, string>({
      query: (tourTypeId) => ({
        url: `/tour/tour-types/${tourTypeId}`,
        method: "DELETE",
      }),
      invalidatesTags: ["TOUR_TYPE"],
    }),

    getTourTypes: builder.query<ITourType[], void>({
      query: () => ({
        url: "/tour/tour-types",
        method: "GET",
      }),
      providesTags: ["TOUR_TYPE"],
      transformResponse: (response: IResponse<ITourType[]>) => response.data,
    }),
  }),
});

export const { useGetTourTypesQuery, useAddTourTypeMutation, useRemoveTourTypeMutation } = tourApi;
```

### `src/utils/runWithToast.ts`

```ts
import { getErrorMessage } from "@/utils/getErrorMessage";
import { toast } from "sonner";

interface IToastMessages {
  loading: string;
  success: string;
  error: string;
}

export const runWithToast = async <T>(promise: Promise<T>, messages: IToastMessages): Promise<T> => {
  const toastId = toast.loading(messages.loading);
  try {
    const result = await promise;
    toast.success(messages.success, { id: toastId });
    return result;
  } catch (err) {
    toast.error(getErrorMessage(err, messages.error), { id: toastId });
    throw err;
  }
};
```

### `src/components/DeleteConfirmation.tsx`

```tsx
import {
  AlertDialog,
  AlertDialogAction,
  AlertDialogCancel,
  AlertDialogContent,
  AlertDialogDescription,
  AlertDialogFooter,
  AlertDialogHeader,
  AlertDialogMedia,
  AlertDialogTitle,
  AlertDialogTrigger,
} from "@/components/ui/alert-dialog";
import { Trash2Icon } from "lucide-react";
import { useState, type MouseEvent, type ReactNode } from "react";

interface IProps {
  /** যে button চাপলে dialog খুলবে */
  children: ReactNode;
  /** Delete চাপলে কী হবে; Promise দিলে শেষ না হওয়া পর্যন্ত dialog খোলা থাকে */
  onConfirm: () => void | Promise<unknown>;
  title?: string;
  description?: ReactNode;
  confirmText?: string;
}

export function DeleteConfirmation({
  children,
  onConfirm,
  title = "Delete this item?",
  description = "This action can't be undone.",
  confirmText = "Delete",
}: IProps) {
  const [open, setOpen] = useState(false);
  const [isPending, setIsPending] = useState(false);

  const handleConfirm = async (event: MouseEvent<HTMLButtonElement>) => {
    event.preventDefault(); // Radix যেন এখনই dialog বন্ধ না করে
    setIsPending(true);
    try {
      await onConfirm();
      setOpen(false); // সফল হলে তবেই বন্ধ
    } catch {
      // ব্যর্থ: dialog খোলা থাকে; error-এর বার্তা onConfirm দেখিয়েছে
    } finally {
      setIsPending(false);
    }
  };

  return (
    <AlertDialog open={open} onOpenChange={(next) => !isPending && setOpen(next)}>
      <AlertDialogTrigger asChild>{children}</AlertDialogTrigger>

      <AlertDialogContent size="sm">
        <AlertDialogHeader>
          <AlertDialogMedia className="bg-destructive-soft text-destructive">
            <Trash2Icon />
          </AlertDialogMedia>
          <AlertDialogTitle>{title}</AlertDialogTitle>
          <AlertDialogDescription>{description}</AlertDialogDescription>
        </AlertDialogHeader>

        <AlertDialogFooter>
          <AlertDialogCancel disabled={isPending}>Cancel</AlertDialogCancel>
          <AlertDialogAction variant="destructive" onClick={handleConfirm} disabled={isPending}>
            {isPending ? "Deleting…" : confirmText}
          </AlertDialogAction>
        </AlertDialogFooter>
      </AlertDialogContent>
    </AlertDialog>
  );
}
```

### `src/pages/Admin/AddTourType.tsx`

```tsx
import { DeleteConfirmation } from "@/components/DeleteConfirmation";
import AddTourTypeForm from "@/components/modules/Admin/TourType/AddTourTypeForm";
import { Button } from "@/components/ui/button";
import { Skeleton } from "@/components/ui/skeleton";
import { useGetTourTypesQuery, useRemoveTourTypeMutation } from "@/redux/features/Tour/tour.api";
import type { ITourType } from "@/types";
import { runWithToast } from "@/utils/runWithToast";
import { Pencil, Tag, Trash2 } from "lucide-react";

export default function AddTourType() {
  const { data: tourTypes = [], isLoading, isError } = useGetTourTypesQuery();
  const [removeTourType] = useRemoveTourTypeMutation();

  const handleRemoveTourType = (item: ITourType) =>
    runWithToast(removeTourType(item._id).unwrap(), {
      loading: `Removing "${item.name}"…`,
      success: `"${item.name}" removed`,
      error: "Couldn't remove the tour type.",
    });

  const countText = isLoading
    ? "Loading…"
    : `${tourTypes.length} ${tourTypes.length === 1 ? "type" : "types"}`;

  return (
    <div className="flex w-full max-w-6xl flex-col gap-5">
      {/* Page heading */}
      <div>
        <h1 className="text-3xl font-bold tracking-tight text-foreground">Tour types</h1>
        <p className="text-muted-foreground">Used to filter tours and shown as a badge on each card.</p>
      </div>

      <div className="grid grid-cols-1 items-start gap-5 lg:grid-cols-[360px_1fr]">
        {/* ───── Left: Add form ───── */}
        <AddTourTypeForm />

        {/* ───── Right: Tour types list ───── */}
        <section className="overflow-hidden rounded-xl border border-border bg-card shadow-card">
          <div className="flex items-center justify-between gap-3 px-4.5 py-4">
            <div>
              <h2 className="text-base font-semibold text-foreground">All tour types</h2>
              <p className="text-[13.5px] text-muted-foreground">{countText}</p>
            </div>
          </div>

          <ul className="list-none">
            {isLoading &&
              Array.from({ length: 3 }).map((_, i) => (
                <li key={i} className="flex items-center gap-3.5 border-t border-border px-4.5 py-3.5">
                  <Skeleton className="size-10 rounded-lg" />
                  <Skeleton className="h-4 flex-1" />
                </li>
              ))}

            {isError && (
              <li className="border-t border-border px-4.5 py-8 text-center text-sm text-destructive">
                Couldn't load tour types. Please refresh the page.
              </li>
            )}

            {!isLoading && !isError && tourTypes.length === 0 && (
              <li className="border-t border-border px-4.5 py-8 text-center text-sm text-muted-foreground">
                No tour types yet — add your first one.
              </li>
            )}

            {tourTypes.map((item) => (
              <li key={item._id} className="flex items-center gap-3.5 border-t border-border px-4.5 py-3.5">
                <span className="flex size-10 shrink-0 items-center justify-center rounded-lg bg-accent text-accent-foreground">
                  <Tag className="size-4.5" />
                </span>

                <div className="min-w-0 flex-1">
                  <strong className="block truncate text-foreground">{item.name}</strong>
                </div>

                {/* Edit-এর endpoint এলে চালু হবে */}
                <Button
                  variant="outline"
                  size="icon"
                  type="button"
                  disabled
                  title="Coming soon"
                  aria-label={`Rename ${item.name}`}
                  className="size-10 rounded-lg disabled:opacity-40"
                >
                  <Pencil className="size-4.25" />
                </Button>

                <DeleteConfirmation
                  title={`Delete "${item.name}"?`}
                  description="This tour type will be removed permanently."
                  onConfirm={() => handleRemoveTourType(item)}
                >
                  <Button
                    variant="outline"
                    size="icon"
                    type="button"
                    aria-label={`Delete ${item.name}`}
                    className="size-10 rounded-lg hover:border-destructive hover:text-destructive"
                  >
                    <Trash2 className="size-4.25" />
                  </Button>
                </DeleteConfirmation>
              </li>
            ))}
          </ul>

          <p className="border-t border-border bg-muted px-4.5 py-3 text-[13.5px] text-muted-foreground">
            A type that's still used by tours can't be deleted. Reassign those tours first.
          </p>
        </section>
      </div>
    </div>
  );
}
```

💡 **`hover:border-destructive hover:text-destructive`:** 🗑️ button সাধারণ অবস্থায় শান্ত outline, mouse রাখলে লাল হয়, যাতে বোঝা যায় এটা মোছার button। (ঐচ্ছিক, দেখতে ভালো লাগলে রাখুন।)

### পরে অন্য জায়গায় ব্যবহার (উদাহরণ)

```tsx
// Division মোছা — একই component, শুধু লেখা আর কাজ আলাদা
<DeleteConfirmation
  title={`Delete "${division.name}"?`}
  onConfirm={() =>
    runWithToast(removeDivision(division._id).unwrap(), {
      loading: "Removing…",
      success: "Division removed",
      error: "Couldn't remove the division.",
    })
  }
>
  <Button variant="outline" size="icon" aria-label={`Delete ${division.name}`}>
    <Trash2 />
  </Button>
</DeleteConfirmation>
```

এটাই reusable-এর সুবিধা: dialog-এর কোড আর কখনো লিখতে হবে না।

---

# Check-list

**ধাপে ধাপে**
- [ ] 🗑️ চাপলে dialog খোলে; বাইরে click করলে বন্ধ হয় না; Cancel দিলে বন্ধ, কিছু মোছে না
- [ ] Console-এ button-এর ভেতরে button-এর warning নেই (`asChild`)
- [ ] Delete চাপলে tour type মুছে যায়, reload ছাড়াই তালিকা থেকে চলে যায়
- [ ] `onConfirm`-এ arrow function: page খুললেই কিছু মোছে না

**উন্নতির পরে**
- [ ] Dialog-এর শিরোনামে ঐ row-এর নাম ("Delete "Short tour"?"), আর "Delete chat?" কোথাও নেই
- [ ] Slow 4G-তে Delete চাপলে dialog খোলা থাকে, button-এ "Deleting…", দুটো button-ই disabled; শেষে নিজে বন্ধ হয়
- [ ] মোছা চলার সময় Escape চাপলে বন্ধ হয় না
- [ ] Backend বন্ধ বা মানা করলে: loading toast লাল error-এ বদলায়, dialog খোলা থাকে
- [ ] Toast-এ নামসহ: "Removing "Short tour"…" → ""Short tour" removed"
- [ ] Tag-এর নাম `addTourType`, `removeTourType`, `getTourTypes` তিন জায়গায় হুবহু একই
- [ ] Dark mode-এ dialog, লাল icon-এর ঘর (`bg-destructive-soft`) ঠিক দেখায়
