# Frontend Notes — ১৯: Division CRUD — Modal, ছবি Upload (FormData) আর Cloudinary থেকে ছবি মোছা

Tour বানানোর আগে দুটো জিনিস লাগে: tour type (১৭–১৮ নম্বর নোট) আর **division**। এই নোটে division-এর তিনটা কাজ:

| CRUD | কাজ | এই নোটে |
|---|---|---|
| **C**reate | নতুন division (ছবিসহ) | ✅ Modal + image uploader + FormData |
| **R**ead | সব division-এর তালিকা | ✅ Table (teacher repetitive বলে করেননি, আমরা করেছি) |
| **U**pdate | বদলানো | পরে |
| **D**elete | মোছা (Cloudinary-র ছবিসহ) | ✅ `DeleteConfirmation` + backend-এর ঠিক করা |

Tour type-এর সাথে বড় পার্থক্য: এখানে **ছবি** পাঠাতে হয়। আর ছবি JSON-এ পাঠানো যায় না, তাই আসছে **FormData**।

নোট দুই ভাগে:
1. **ধাপে ধাপে (ধাপ ১–১৪):** teacher-এর ক্রমে, আপনার কোড দিয়ে, সাথে আপনার প্রশ্নের উত্তর।
2. **উন্নতি আর চূড়ান্ত কোড।**

প্রতিটা ধাপে 🎯 লক্ষ্য, 🛠️ কাজ, 💡 কেন, ✅ যাচাই।

> নোট: আপনার project-এর version মেনে লেখা (shadcn `radix-nova`, Zod 4, React Hook Form 7, axios 1)। আপনার বর্তমান কোড পড়ে নিয়ে লেখা; পুরনো ধরনের কোড দিয়ে কিছু বদলানো হয়নি।

---

## ধাপের মানচিত্র

| ভাগ | ধাপ | শেষে কী হবে |
|---|---|---|
| **ক. JSON বনাম FormData** | ১ | Backend কী আকারে data চায়, বোঝা |
| **খ. Page, Modal, Image uploader** | ২–৫ | "Add division" চাপলে modal, ছবি বেছে নিলে preview |
| **গ. Submit** | ৬–১০ | ছবিসহ division backend-এ তৈরি হয় |
| **ঘ. তালিকা ও মোছা** | ১১–১২ | Table-এ আসল data, মোছা যায় |
| **ঙ. Backend: ছবি মোছা** | ১৩–১৪ | Division মুছলে Cloudinary-র ছবিও মোছে |
| **উন্নতি ও চূড়ান্ত কোড** | — | — |

### শেষে যে ফাইলগুলো থাকবে

```
src/
├── components/
│   ├── DeleteConfirmation.tsx                  ← itemName সহ (ধাপ ১২)
│   ├── modules/Admin/Division/
│   │   └── AddDivisionModal.tsx                ← ধাপ ৩, ৬–১০
│   └── ui/
│       ├── dialog.tsx                          ← shadcn
│       └── image-uploader.tsx                  ← reusable (ধাপ ৪–৫)
├── pages/Admin/
│   └── AddDivision.tsx                         ← table (ধাপ ২, ১১)
├── redux/features/division/
│   └── division.api.ts                         ← ধাপ ৯, ১১, ১২
└── types/
    └── division.type.ts                        ← ধাপ ১১
```

---

# ভাগ ক: JSON বনাম FormData

## ধাপ ১ — Postman-এ দেখা: backend কী চায়

🎯 **লক্ষ্য:** কোড লেখার আগে জানা, division তৈরির API কী আকারে data নেয়।

🛠️ **কাজ:** Postman-এ create division-এর request খুলে **Body** tab দেখুন। এতদিন আমরা দেখেছি **raw → JSON**। এখানে বেছে নেওয়া **form-data**, আর দুটো key:

| Key | Type (Postman-এ) | মান |
|---|---|---|
| `data` | Text | `{"name":"Sylhet","slug":"sylhet","description":"..."}` (একটা JSON string) |
| `file` | **File** | একটা ছবি |

### 💡 JSON আর FormData-র পার্থক্য

| | JSON (`raw → JSON`) | FormData (`form-data`) |
|---|---|---|
| কী পাঠানো যায় | শুধু লেখা: string, সংখ্যা, true/false, object, array | লেখা **আর ফাইল** (ছবি, PDF …) |
| Header | `Content-Type: application/json` | `Content-Type: multipart/form-data; boundary=...` |
| কখন | ছবি নেই এমন সব (login, tour type …) | ফাইল আছে এমন সব (division, tour, profile ছবি) |

**কেন ছবি JSON-এ যায় না:** JSON শুধু লেখা বোঝে। ছবি হলো **binary data**: লেখা না, ০ আর ১-এর লম্বা সারি (pixel-এর তথ্য)। JSON-এ ঢোকাতে হলে আগে ছবিকে লেখায় রূপান্তর করতে হয় (base64), তাতে মাপ প্রায় ৩৩% বেড়ে যায়, আর server-কে আবার উল্টো রূপান্তর করতে হয়। FormData ফাইলকে **যেমন আছে তেমনই** পাঠায়।

### 💡 Blob আর File

Teacher বলেছেন file type-কে বলে **Blob**:
- **Blob** (Binary Large Object) = browser-এ "একটা টুকরো binary data"। নাম নেই, শুধু data আর তার ধরন (`image/png`)।
- **File** = Blob + নাম + শেষ বদলের সময়। User `<input type="file">` দিয়ে যা বেছে নেয়, সেটা `File`। প্রতিটা File আসলে একটা Blob-ও।

তাই "file পাঠানো" মানে binary data-র Blob পাঠানো, আর FormData ঠিক এটা পাঠাতে পারে।

### 💡 Key-গুলো কেন এত জরুরি

Teacher জোর দিয়ে বলেছেন: **backend যে key চায়, frontend থেকে হুবহু সেই key-তেই পাঠাতে হবে।** কারণ backend-এ সম্ভবত এমন কোড আছে:

```ts
// backend (উদাহরণ)
router.post("/create", multerUpload.single("file"), ...)   // ← "file" নামের key থেকে ছবি নেয়
const payload = JSON.parse(req.body.data);                 // ← "data" নামের key থেকে লেখা নেয়
```

Frontend থেকে `"image"` নামে ছবি পাঠালে backend ছবিই পাবে না (multer অনেক সময় "Unexpected field" error দেয়); `"info"` নামে data পাঠালে `req.body.data` হবে `undefined`। বানান, বড়-ছোট হাতের অক্ষর সব মিলতে হবে।

✅ **যাচাই:** Postman-এ form-data দিয়ে একটা division তৈরি হয়; key দুটোর নাম লিখে রাখুন (`data`, `file`)।

---

# ভাগ খ: Page, Modal, Image uploader

## ধাপ ২ — Page-এর design আর route

🎯 **লক্ষ্য:** Division-এর table (এখনো নমুনা data) আর sidebar-এ link।

🛠️ **কাজ ১ — page:** আপনি Claude-এর সাহায্যে বানিয়েছেন: উপরে শিরোনাম আর "Add division" button, নিচে table (`#`, Division, Slug, Description, Actions)। প্রথমে উপরে একটা `divisions` array-তে ৮টা নমুনা division রেখে `map` করা।

💡 **Table কেন (tour type-এ list ছিল):** Tour type-এ শুধু একটা নাম (১৭ নম্বর নোট), তাই `<ul>` যথেষ্ট। Division-এ একাধিক column (ছবি, নাম, slug, বর্ণনা, কাজ), তাই `<table>`। আপনার design-এ সাধারণ HTML `<table>`, Tailwind দিয়ে সাজানো; shadcn-এর `Table`-ও একই HTML বানায়, শুধু class আগে থেকে দেওয়া থাকে। দুটোই ঠিক।

**Table-এর কিছু অংশ:**
- **`<th scope="col">`:** screen reader-কে বলে এই শিরোনাম পুরো column-এর।
- **`overflow-x-auto` + `min-w-[680px]`:** ছোট screen-এ table চেপে ভাঙে না, বরং পাশে scroll হয়।
- **`truncate` + `title={division.description}`:** লম্বা বর্ণনা এক লাইনে কেটে "…", mouse রাখলে পুরোটা দেখায়।

🛠️ **কাজ ২ — route** (১৪ নম্বর নোটের নিয়মে, ১৫ নম্বর নোটের মতো lazy):

```ts
// src/routes/adminSidebarItems.ts
const AddDivision = lazy(() => import("@/pages/Admin/AddDivision"));

{ title: "Divisions", url: "/admin/divisions", component: AddDivision },
```

> নোট: আপনার কোডে এখন `{ title: "Add Division", url: "/admin/add-division" }`। কিন্তু page-টা শুধু যোগ করার না, **সব division দেখা আর মোছারও** জায়গা (যোগ করা হয় modal-এ)। তাই menu-তে "Divisions" আর URL `/admin/divisions` বেশি মানানসই, ১৪ নম্বর নোটের প্রথম sidebar-এও এটাই ছিল। একই কারণে "Tour type"-কেও "Tour types" রাখার কথা ১৭ নম্বর নোটে বলা হয়েছে।

✅ **যাচাই:** Sidebar থেকে page খোলে, ৮টা নমুনা division-এর table।

---

## ধাপ ৩ — Modal (Dialog)

🎯 **লক্ষ্য:** "Add division" চাপলে একটা popup form।

🛠️ **কাজ:**

```sh
bunx --bun shadcn@latest add dialog
```

> ⚠️ সংশোধন: নোটে `npx shadcn@latest add dialog` লেখা। আমাদের project-এ package manager bun, তাই সবসময় `bunx --bun shadcn@latest add …` (৩ নম্বর নোট)।

Modal আলাদা ফাইলে: `src/components/modules/Admin/Division/AddDivisionModal.tsx`। প্রথম রূপে শুধু design (সাধারণ input, কোনো form library না):

```tsx
<Dialog>
  <DialogTrigger asChild>
    <Button type="button">
      <Plus /> Add division
    </Button>
  </DialogTrigger>

  <DialogContent className="max-h-[90vh] gap-0 overflow-y-auto rounded-xl p-0 sm:max-w-120">
    <DialogHeader>
      <DialogTitle>Add division</DialogTitle>
      <DialogDescription>Each division gets its own public page at /divisions/slug.</DialogDescription>
    </DialogHeader>

    <form id="add-division">{/* Thumbnail, Name, Slug, Description */}</form>

    <DialogFooter>
      <DialogClose asChild>
        <Button variant="outline" type="button">Cancel</Button>
      </DialogClose>
      <Button type="submit" form="add-division">Save division</Button>
    </DialogFooter>
  </DialogContent>
</Dialog>
```

💡 **কিছু অংশ:**
- **Dialog, AlertDialog না:** এটা form, সিদ্ধান্ত না; বাইরে click করে বন্ধ করতে পারা স্বাভাবিক (১৮ নম্বর নোটের পার্থক্যের টেবিল)।
- **`DialogTrigger asChild`, `DialogClose asChild`:** নিজের `Button`-কে trigger/close বানানো, button-এর ভেতরে button না (১৮ নম্বর নোট)।
- **`max-h-[90vh] overflow-y-auto`:** form লম্বা হলে (ছবি + তিনটা field) ছোট screen-এ modal screen ছাড়িয়ে যায় না, ভেতরে scroll হয়।
- **`<form id="add-division">` + `<Button form="add-division">`:** Submit button `<form>`-এর **বাইরে** (footer-এ, যাতে footer আলাদা রঙে নিচে বসে)। HTML-এর `form` attribute দিয়ে বাইরের button-ও ঐ id-র form submit করতে পারে (১৩ নম্বর নোটের Verify page-এর মতো)।

Page-এ পুরনো "Add division" button-এর জায়গায় `<AddDivisionModal />` বসান।

✅ **যাচাই:** "Add division" চাপলে modal খোলে, Cancel বা বাইরে click করলে বন্ধ হয়।

---

## ধাপ ৪ — Reusable Image Uploader

🎯 **লক্ষ্য:** ছবি বেছে নেওয়ার সুন্দর একটা component, যেটা division, tour, profile সব জায়গায় চলবে।

Teacher Origin UI-র "single image uploader" ব্যবহার করেছেন; আপনি Claude-এর সাহায্যে নিজের বানিয়েছেন: `src/components/ui/image-uploader.tsx` (পুরো কোড আপনার নোটে আছে; এখানে মূল ধারণাগুলো)।

### 💡 মূল কৌশল ১: আসল `<input type="file">` লুকানো, `<label>` দেখানো

Browser-এর নিজের file input দেখতে কুৎসিত আর সাজানো যায় না। তাই:

```tsx
<input ref={inputRef} id={id} type="file" accept={accept} className="sr-only" onChange={...} />

<label htmlFor={id} className="... border-dashed ...">
  Click or drag an image here
</label>
```

- **`sr-only`:** চোখে দেখা যায় না, কিন্তু screen reader আর keyboard (Tab) এখনো পায়। (`hidden` দিলে keyboard দিয়েও যাওয়া যেত না।)
- **`<label htmlFor={id}>`:** label-এ click করলে browser নিজেই ঐ id-র input-এ click করে, file বেছে নেওয়ার জানালা খোলে। তাই সাজানো বাক্সটা আসলে label।
- **`inputRef.current?.click()`:** "Replace" button-এর মতো অন্য জায়গা থেকে জানালা খুলতে হাতে input-এ click করানো।

### 💡 মূল কৌশল ২: `value` + `onChange` (controlled, সাধারণ input-এর মতো)

```ts
type ImageUploaderProps = {
  value?: File | string | null;              // বর্তমান ছবি
  onChange: (file: File | undefined) => void; // নতুন ছবি বা মুছে ফেলা
  onBlur?: () => void;
  ...
};
```

Component নিজে ছবি মনে রাখে না, বাইরে থেকে `value` পায় আর বদলালে `onChange` দিয়ে জানায়। ঠিক সাধারণ `<Input>`-এর মতো। এজন্যই React Hook Form-এর `Controller` এটাকে সরাসরি চালাতে পারে (ধাপ ৭)।

**`value`-তে `File | string` কেন:** নতুন ছবি বাছলে `File`; কিন্তু পরে **Edit**-এর সময় database-এ আগে থেকে থাকা ছবি আসবে Cloudinary-র URL (`string`) হিসেবে। দুটোই preview দেখাতে পারে, তাই একই component add আর edit দুই জায়গায় চলবে। চমৎকার আগাম চিন্তা।

### 💡 মূল কৌশল ৩: Drag and drop

```tsx
onDragOver={(e) => { e.preventDefault(); setIsDragging(true); }}
onDragLeave={() => setIsDragging(false)}
onDrop={(e) => { e.preventDefault(); setIsDragging(false); handleFile(e.dataTransfer.files?.[0]); }}
```

- **`e.preventDefault()`:** না দিলে browser ফাইল ছাড়লে নিজেই নতুন tab-এ ছবিটা খুলে ফেলে।
- **`e.dataTransfer.files[0]`:** টেনে আনা ফাইল।
- **`isDragging`:** টেনে বাক্সের ওপর আনলে রং বদলায় আর লেখা হয় "Drop the image here"।

### 💡 Remove-এ `inputRef.current.value = ""` কেন

ধরুন `sylhet.jpg` বাছলেন, Remove করলেন, আবার **একই** `sylhet.jpg` বাছলেন। Input-এর মান না মুছলে browser ভাবে "কিছুই বদলায়নি", `onChange` চালায় না, ছবি আসে না। মান খালি করলে আবার একই ফাইল বাছা যায়।

✅ **যাচাই:** Modal-এ বাক্সে click → ফাইল বাছার জানালা; ছবি টেনে এনে ছাড়লেও কাজ করে (এখনো preview নাও আসতে পারে, পরের ধাপ)।

---

## ধাপ ৫ — Preview: `FileReader` বনাম `URL.createObjectURL`

🎯 **লক্ষ্য:** Upload-এর আগেই বাছা ছবিটা দেখানো।

`<img src>`-এ একটা URL লাগে, কিন্তু বাছা ছবিটা এখনো কোথাও upload হয়নি, শুধু user-এর computer-এ। তাই browser-কে দিয়ে ঐ ফাইলের একটা "URL" বানাতে হয়। Teacher দুটো উপায় দেখিয়েছেন:

### 💡 দুটো উপায়, ছোট করে

| | `FileReader.readAsDataURL` | `URL.createObjectURL` |
|---|---|---|
| কী বানায় | **Data URL**: পুরো ছবিকে **base64 লেখায়** রূপান্তর করে URL বানায় | **Blob URL**: ফাইলের দিকে ইশারা করা ছোট একটা ঠিকানা |
| দেখতে | `data:image/jpeg;base64,/9j/4AAQSkZJRg…` (লক্ষ লক্ষ অক্ষর) | `blob:http://localhost:3000/4f1c…` (ছোট) |
| গতি | ধীর: পুরো ফাইল পড়ে রূপান্তর করে, **async** (callback/event-এ ফল আসে) | তাৎক্ষণিক, **sync** (এক লাইনে ফল) |
| Memory | ছবির চেয়ে ~৩৩% বড় একটা লেখা memory-তে | প্রায় কিছুই না, ফাইলটাই ব্যবহার হয় |
| পরিষ্কার করা | লাগে না | কাজ শেষে **`URL.revokeObjectURL(url)`** দিয়ে ছাড়তে হয় |
| বয়স | পুরনো উপায় | নতুন, preview-এর জন্য বেশি ব্যবহৃত |

> ⚠️ সংশোধন: নোটে লেখা ছিল "`createObjectURL` base64 দিয়ে read করে"। আসলে উল্টো: **base64 বানায় `readAsDataURL`** (FileReader)। `createObjectURL` কিছুই পড়ে বা রূপান্তর করে না, শুধু browser-এর memory-তে থাকা ফাইলটার জন্য একটা অস্থায়ী ঠিকানা বানায়, তাই এত দ্রুত। আপনার নোটের "FileReader পুরনো" কথাটা ঠিক।

আপনার uploader `createObjectURL` ব্যবহার করে, যেটাই ভালো পছন্দ:

```ts
// File হলে blob URL বানাও, string (পুরনো ছবির URL) হলে সেটাই
const previewUrl = useMemo(() => {
  if (!value) return null;
  return value instanceof File ? URL.createObjectURL(value) : value;
}, [value]);

// ছবি বদলালে বা component সরে গেলে পুরনো blob URL ছেড়ে দাও
useEffect(() => {
  return () => {
    if (value instanceof File && previewUrl) URL.revokeObjectURL(previewUrl);
  };
}, [value, previewUrl]);
```

- **`useMemo`:** `value` না বদলালে প্রতি render-এ নতুন blob URL বানায় না।
- **`revokeObjectURL` কেন:** Blob URL যতক্ষণ বেঁচে থাকে, browser ফাইলটা memory-তে ধরে রাখে। ১০টা ছবি বদলালে ১০টা ছবি memory-তে জমে থাকত (memory leak)। Cleanup-এ পুরনোটা ছেড়ে দেওয়া হয় (১৩ নম্বর নোটের `useEffect` cleanup-এর ধারণা)।

✅ **যাচাই:** ছবি বাছলেই বাক্সের জায়গায় ছবির preview, নিচে নাম, "Replace" আর "Remove"।

---

# ভাগ গ: Submit

## ধাপ ৬ — Zod schema: ফাইলও যাচাই

🎯 **লক্ষ্য:** নাম, বর্ণনা আর ছবি তিনটারই নিয়ম।

🛠️ **কাজ:**

```ts
const MAX_SIZE = 1 * 1024 * 1024; // 1 MB
const IMAGE_TYPES = ["image/png", "image/jpeg", "image/webp"];

const formSchema = z.object({
  name: z.string().trim().min(2, { error: "..." }).max(50, { error: "..." }),
  description: z.string().trim().max(300, { error: "..." }).optional(),
  thumbnail: z
    .instanceof(File)
    .refine((file) => IMAGE_TYPES.includes(file.type), { error: "Only PNG, JPG or WEBP allowed." })
    .refine((file) => file.size <= MAX_SIZE, { error: "Image must be 1 MB or smaller." })
    .optional(),
});
```

💡 **লাইনগুলোর মানে:**
- **`1 * 1024 * 1024`:** ফাইলের মাপ byte-এ। 1 KB = 1024 byte, 1 MB = 1024 KB। তাই 1 MB = 1,048,576 byte। এভাবে লিখলে পড়েই বোঝা যায় "১ MB"।
- **`z.instanceof(File)`:** মানটা সত্যিই একটা `File` কিনা।
- **`file.type`:** browser ফাইলের ধরন বলে দেয় (`"image/png"`)। `accept` দিয়ে জানালায় শুধু ছবি দেখালেও drag-and-drop-এ যেকোনো ফাইল আসতে পারে, তাই এখানে আবার যাচাই।
- **`file.size`:** byte-এ মাপ।
- **`.optional()`:** ছবি ছাড়াও division বানানো যাবে।

> ➕ সংযোজন (Zod 4-এর নতুন উপায়): Zod 4-এ ফাইলের জন্য আলাদা `z.file()` এসেছে, ধরন আর মাপের নিয়ম তৈরি:
> ```ts
> thumbnail: z
>   .file()
>   .mime(IMAGE_TYPES, { error: "Only PNG, JPG or WEBP allowed." })
>   .max(MAX_SIZE, { error: "Image must be 1 MB or smaller." })
>   .optional(),
> ```
> আপনার `instanceof` + `refine`-ও ঠিক চলছে; এটা শুধু ছোট আর পড়তে সহজ। বদলাতে চাইলে বদলান, না চাইলে যেমন আছে রাখুন।

> নোট: Frontend-এর নিয়ম (1 MB, ৩ ধরন) backend-এর multer-এর নিয়মের সাথে মিলিয়ে রাখবেন। Frontend 2 MB দিলে আর backend 1 MB নিলে, user সব ঠিক দেখেও backend-এর error পাবে।

---

## ধাপ ৭ — Field-গুলো `Controller` দিয়ে, আর slug

🎯 **লক্ষ্য:** React Hook Form-কে তিনটা field (ছবি, নাম, বর্ণনা) জানানো, আর নাম থেকে নিজে slug বানানো।

🛠️ **কাজ — ছবির field:**

```tsx
<Controller
  name="thumbnail"
  control={form.control}
  render={({ field, fieldState }) => (
    <Field data-invalid={fieldState.invalid} className="gap-1.5">
      <FieldLabel htmlFor="dv-thumbnail">Thumbnail</FieldLabel>
      <ImageUploader
        id="dv-thumbnail"
        value={field.value}
        onChange={field.onChange}
        onBlur={field.onBlur}
        invalid={fieldState.invalid}
      />
      {fieldState.invalid && <FieldError errors={[fieldState.error]} />}
    </Field>
  )}
/>
```

💡 **এখানে `{...field}` না, আলাদা আলাদা কেন:** `Input`-এ `{...field}` ছড়িয়ে দিলে চলে, কারণ `Input` সাধারণ `<input>`-এর সব prop চেনে। `ImageUploader`-এর নিজের prop আছে (`value`, `onChange`, `onBlur`); `field`-এর `name`, `ref` এগুলো সে নেয় না। তাই যা লাগে শুধু সেগুলো হাতে দেওয়া। ধাপ ৪-এর "controlled" নকশার কারণে এটা এত সহজে জুড়ে গেল।

### 💡 Slug: নাম থেকে নিজে

```ts
const makeSlug = (text: string) =>
  text
    .toLowerCase()                   // "Cox's Bazar" → "cox's bazar"
    .trim()                          // দুই পাশের space বাদ
    .replace(/[^a-z0-9\s-]/g, "")   // অক্ষর, সংখ্যা, space, - ছাড়া সব বাদ → "coxs bazar"
    .replace(/\s+/g, "-")            // এক বা একাধিক space → "-" → "coxs-bazar"
    .replace(/-+/g, "-");            // "--" → "-"

const name = form.watch("name");
// ...
<Input id="dv-slug" value={makeSlug(name)} readOnly />
```

**Slug কী:** URL-এ ব্যবহারের উপযোগী নাম: ছোট হাতের, space-এর বদলে `-`, কোনো বিশেষ চিহ্ন নেই। `/divisions/coxs-bazar` পড়তে সুন্দর, `/divisions/6650f1a2…` না।

**`form.watch("name")`:** নামের field-এ যা লেখা হচ্ছে সেটা প্রতি অক্ষরে দেখে, আর slug-এর box সাথে সাথে আপডেট হয়। Slug নিজে form-এর field না (`readOnly`, শুধু দেখানোর জন্য); পাঠানোর সময় আবার `makeSlug(data.name)` দিয়ে বানানো হয় (ধাপ ৮)।

> নোট (দুটো সতর্কতা):
> - `[^a-z0-9\s-]` শুধু ইংরেজি অক্ষর রাখে। কেউ বাংলায় নাম লিখলে ("সিলেট") slug হবে **ফাঁকা**। Division-এর নাম ইংরেজিতে রাখার নিয়ম হলে ঠিক আছে; নাহলে slug খালি হলে error দেখানো দরকার।
> - অনেক backend (এই course-এর backend-ও সম্ভবত) Mongoose-এর `pre("save")` hook-এ নিজেই নাম থেকে slug বানায়। Postman-এ slug না পাঠিয়ে দেখুন। Backend বানালে frontend-এর slug শুধু preview, আর দুই জায়গার নিয়ম এক রাখতে হবে, নাহলে user এক slug দেখবে আর সংরক্ষণ হবে আরেকটা।

---

## ধাপ ৮ — FormData বানানো

🎯 **লক্ষ্য:** Form-এর data-কে backend-এর চাওয়া আকারে (ধাপ ১) সাজানো।

🛠️ **কাজ:**

```ts
const onSubmit = async (data: FormValues) => {
  const formData = new FormData();

  formData.append(
    "data",
    JSON.stringify({ name: data.name, slug: makeSlug(data.name), description: data.description }),
  );

  if (data.thumbnail) formData.append("file", data.thumbnail);

  // ...পাঠানো (ধাপ ৯-১০)
};
```

### 💡 `append` আর `JSON.stringify` কেন (আপনার প্রশ্ন)

**`new FormData()`:** একটা খালি "খাম"। **`append(key, value)`:** খামে একটা জিনিস ঢোকানো, নাম দিয়ে। Postman-এর form-data-র প্রতিটা সারি = একটা `append`।

**FormData-র নিয়ম: মান হতে পারে শুধু দুই রকম, string অথবা Blob/File।** Object সরাসরি দেওয়া যায় না:

```ts
formData.append("data", { name: "Sylhet" });
// ❌ browser object-কে নিজে string বানায় → "[object Object]"  (backend কিছুই পায় না)

formData.append("data", JSON.stringify({ name: "Sylhet" }));
// ✅ '{"name":"Sylhet"}'  → backend JSON.parse(req.body.data) দিয়ে আবার object বানায়
```

তাই আপনার কথাই ঠিক: data object আকারে আছে, কিন্তু পাঠাতে হবে JSON (string) আকারে। আর ছবি (File) যেমন আছে তেমন `append` হয়, কারণ File তো Blob।

```
FormData (খাম)
├── "data" → '{"name":"Sylhet","slug":"sylhet","description":"..."}'   (string)
└── "file" → sylhet.jpg                                                 (File / Blob)
```

**`if (data.thumbnail)`:** ছবি না বাছলে `file` key-টাই পাঠানো হয় না (খালি key পাঠালে backend বিভ্রান্ত হতে পারে)।

> নোট: FormData `console.log` করলে খালি `FormData {}` দেখায়, ভেতরে কী আছে বোঝা যায় না। দেখতে চাইলে: `console.log([...formData.entries()])`। আর আসল পরীক্ষা: Network tab → request → **Payload**, সেখানে `data` আর `file` দুটো আলাদা দেখায়।

---

## ধাপ ৯ — `division.api.ts`

🎯 **লক্ষ্য:** FormData পাঠানোর API।

🛠️ **কাজ:**

```ts
// src/redux/features/Division/division.api.ts
import { baseApi } from "@/redux/baseApi";

export const divisionApi = baseApi.injectEndpoints({
  endpoints: (builder) => ({
    addDivision: builder.mutation({
      query: (divisionData: FormData) => ({
        url: "/division/create",
        method: "POST",
        data: divisionData,
      }),
      invalidatesTags: ["DIVISION"],
    }),
  }),
});
```

আর `baseApi`-র `tagTypes`-এ `"DIVISION"` (আপনার কোডে আছে ✅)।

### 💡 Content-Type নিজে দেবেন না

আপনার কোডের comment-টা খুব জরুরি: **`Content-Type` হাতে বসাবেন না।** FormData পাঠালে header হয় এমন:

```
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary7MA4YWxkTrZu0gW
```

`boundary` হলো একটা এলোমেলো দাগ, যেটা দিয়ে body-র ভেতরে `data` আর `file` আলাদা করা থাকে। এটা browser নিজে বানায়, আর axios FormData দেখলে browser-কেই header বসাতে দেয়। হাতে `"multipart/form-data"` লিখে দিলে boundary থাকে না, backend body ভাগই করতে পারে না।

### 💡 আপনার প্রশ্ন: folder-এর নাম বড় হাতের না ছোট হাতের?

এখন আপনার `redux/features/`-এ: `auth` (ছোট), `Tour` (বড়), `Division` (বড়)। **ছোট হাতের ভালো: `auth`, `tour`, `division`।**

নিয়মটা এভাবে মনে রাখুন:

| কী | নামের ধরন | উদাহরণ |
|---|---|---|
| React component-এর ফাইল, আর component-এর দলের folder | **PascalCase** (বড় হাতে শুরু) | `AddDivisionModal.tsx`, `modules/Admin/Division/` |
| বাকি সব folder আর ফাইল (component না) | **ছোট হাতের** (প্রয়োজনে `-` বা `.`) | `redux/features/division/`, `division.api.ts`, `utils/`, `hooks/` |

`redux/features/division` কোনো component না, ভেতরের ফাইলও ছোট হাতের (`division.api.ts`), তাই folder-ও ছোট হাতের। আর `components/modules/Admin/Division/` বড় হাতের থাকবে, কারণ ভেতরে component আর এটা page-এর নাম (Authentication-এর মতো)।

**নাম বদলানোর সময় সাবধান:** Mac-এর file system `Division` আর `division`-কে **একই** নাম ধরে, তাই সরাসরি rename অনেক সময় কাজ করে না বা git বুঝতে পারে না। দুই ধাপে করুন: `Division` → `division-tmp` → `division`। অথবা git থাকলে `git mv Division division-tmp && git mv division-tmp division`। তারপর editor-এ সব import (`@/redux/features/Division/...`) খুঁজে ছোট হাতের করে দিন। Linux server (deploy) বড়-ছোট হাতের পার্থক্য ধরে, তাই import না মিললে সেখানে build ভাঙবে।

---

## ধাপ ১০ — Modal-এর অবস্থা: খোলা/বন্ধ, loading, reset

🎯 **লক্ষ্য:** Upload চলার সময় সব বন্ধ, সফল হলে modal বন্ধ আর form খালি।

🛠️ **কাজ (আপনার কোডের মূল অংশ):**

```tsx
const [open, setOpen] = useState(false);
const [addDivision, { isLoading }] = useAddDivisionMutation();

const onSubmit = async (data: FormValues) => {
  // ...formData (ধাপ ৮)
  const toastId = toast.loading("Uploading division…");
  try {
    await addDivision(formData).unwrap();
    toast.success("Division added", { id: toastId });
    form.reset();
    setOpen(false);
  } catch (err) {
    const error = err as TApiError;
    toast.error(error.data?.message || "Failed to add division", { id: toastId });
  }
};

// upload চলার সময় বন্ধ করতে দেব না; বন্ধ হলে form খালি
const handleOpenChange = (value: boolean) => {
  if (isLoading) return;
  setOpen(value);
  if (!value) form.reset();
};

<Dialog open={open} onOpenChange={handleOpenChange}>
```

💡 **লাইনগুলোর মানে:**
- **`open` / `onOpenChange` (controlled):** Dialog নিজে খোলা-বন্ধ সামলালে সফল হওয়ার পরে আমরা বন্ধ করতে পারতাম না। তাই অবস্থা নিজের হাতে (১৮ নম্বর নোটের উন্নতি ৩-এর মতো)।
- **`if (isLoading) return;`:** upload চলার সময় Escape, বাইরে click বা Cancel, কিছুতেই বন্ধ হয় না। নাহলে user ভাবত বাতিল হয়েছে, অথচ ছবি upload চলছে।
- **`if (!value) form.reset();`:** অর্ধেক লিখে Cancel করলে, পরের বার খুললে আগের লেখা থাকে না।
- **সফল হলে `form.reset()` + `setOpen(false)`:** modal বন্ধ, পরের বার খালি form। আর `invalidatesTags` তালিকা নিজে আপডেট করে।

### 💡 `<fieldset disabled>` — এক লাইনে সব field বন্ধ

```tsx
<form ...>
  <fieldset disabled={isLoading} className="flex flex-col gap-4 disabled:opacity-60">
    {/* সব field */}
  </fieldset>
</form>
```

HTML-এর `<fieldset>`-এ `disabled` দিলে **ভেতরের সব** input, textarea, button একসাথে disabled হয়। প্রতিটা field-এ আলাদা `disabled={isLoading}` লিখতে হয় না। `disabled:opacity-60` দিয়ে পুরো অংশ একটু ফিকে, বোঝা যায় কাজ চলছে।

✅ **যাচাই:**
- ছবি, নাম, বর্ণনা দিয়ে Save → "Uploading division…" → "Division added", modal বন্ধ।
- Network → `create` → Payload-এ `data` (JSON লেখা) আর `file` (binary)।
- Slow 4G-তে upload চলার সময় field-গুলো ফিকে আর বন্ধ, Escape চাপলে modal বন্ধ হয় না।
- 1 MB-এর বড় ছবি বা PDF দিলে ছবির নিচে লাল error, request যায় না।

---

# ভাগ ঘ: তালিকা ও মোছা

## ধাপ ১১ — আসল data দিয়ে table

🎯 **লক্ষ্য:** নমুনা array-র বদলে backend-এর division। (Teacher repetitive বলে এটা দেখাননি, আপনি করেছেন।)

🛠️ **কাজ ১ — type:**

```ts
// src/types/division.type.ts
export interface IDivision {
  _id: string;
  name: string;
  slug: string;
  description?: string;
  thumbnail?: string;   // Cloudinary-র URL
  createdAt?: string;
  updatedAt?: string;
}
```

```ts
// src/types/index.ts
export type { IDivision } from "./division.type";
```

🛠️ **কাজ ২ — endpoint:**

```ts
getDivisions: builder.query({
  query: () => ({ url: "/division", method: "GET" }),
  providesTags: ["DIVISION"],
  transformResponse: (response) => response.data,
}),
```

🛠️ **কাজ ৩ — page:** আপনার page-এ table-এর **চারটা অবস্থা**, ১৭ নম্বর নোটের নিয়মে:

| অবস্থা | আপনার কোডে |
|---|---|
| লোড হচ্ছে | ৫টা skeleton row, প্রতিটা column-এর আকারে |
| Error | লাল icon, "Couldn't load divisions", আর **"Try again"** button (`refetch()`) |
| খালি | "No divisions yet" |
| Data | আসল row; ছবি থাকলে `<img>`, না থাকলে icon; বর্ণনা না থাকলে "—" |

💡 **কিছু ভালো জিনিস যা আপনি যোগ করেছেন:**
- **`refetch`:** RTK Query-র hook দেয়; error হলে user নিজে আবার চেষ্টা করতে পারে, page reload ছাড়া।
- **`isFetching && !isLoading` → `opacity-60`:** প্রথমবার না, কিন্তু **আবার** আনার সময় (যেমন নতুন division যোগ বা মোছার পরে) table একটু ফিকে হয়, বোঝা যায় আপডেট হচ্ছে। (`isLoading` শুধু প্রথমবার, `isFetching` যেকোনো সময়, ১৭ নম্বর নোটের টেবিল।)
- **`colSpan={COLS}`:** Error বা খালি বার্তা একটা ঘরে, কিন্তু পুরো table-এর প্রস্থ জুড়ে। Column-এর সংখ্যা (`COLS = 5`) এক জায়গায় রাখা, table-এর column বদলালে শুধু এটা বদলালেই হয়।
- **`loading="lazy"`:** তালিকার ছবি screen-এর কাছে এলে তবেই আসে (১৫ নম্বর নোট)।
- **"Tours" column বাদ:** নমুনা data-য় ছিল, backend পাঠায় না, তাই সরিয়েছেন।

✅ **যাচাই:** Table-এ database-এর আসল division আর তাদের ছবি; নতুন যোগ করলে reload ছাড়াই আসে।

---

## ধাপ ১২ — মোছা

🎯 **লক্ষ্য:** Division মোছা, ১৮ নম্বর নোটের reusable `DeleteConfirmation` দিয়ে।

🛠️ **কাজ ১ — endpoint:**

```ts
removeDivision: builder.mutation({
  query: (divisionId: string) => ({
    url: `/division/${divisionId}`,
    method: "DELETE",
  }),
  invalidatesTags: ["DIVISION"],
}),
```

🛠️ **কাজ ২ — `DeleteConfirmation`-এ `itemName`:** আপনি component আপডেট করেছেন, শুধু নাম দিলেই শিরোনাম বানিয়ে নেয়:

```tsx
interface IProps {
  children: ReactNode;
  onConfirm: () => void;
  itemName?: string;   // "Sylhet" → শিরোনাম: Delete "Sylhet"?
  title?: string;
  description?: string;
  confirmText?: string;
  cancelText?: string;
}

const dialogTitle = title ?? (itemName ? `Delete "${itemName}"?` : "Are you sure?");
```

💡 **`??` (nullish coalescing):** বাম পাশ `undefined` বা `null` হলে ডান পাশ। পড়লে: "`title` দেওয়া থাকলে সেটাই; না থাকলে, `itemName` থাকলে `Delete "নাম"?`; সেটাও না থাকলে `Are you sure?`"। তিন স্তরের default, সবচেয়ে নির্দিষ্টটা আগে।

🛠️ **কাজ ৩ — page-এ:**

```tsx
<DeleteConfirmation
  itemName={division.name}
  description="This removes the division and its public page. This can't be undone."
  onConfirm={() => handleRemoveDivision(division._id)}
>
  <Button variant="outline" size="icon" aria-label={`Delete ${division.name}`} ...>
    <Trash2 className="size-4" />
  </Button>
</DeleteConfirmation>
```

(`onConfirm`-এ arrow function কেন, ১৮ নম্বর নোটের ধাপ ৬।)

✅ **যাচাই (এবং সমস্যা):** Division মুছে যায়, table থেকে চলে যায়। কিন্তু Cloudinary-র dashboard খুললে দেখা যায় **ছবিটা এখনো আছে**। পরের ভাগে।

---

# ভাগ ঙ: Backend — Cloudinary থেকে ছবি মোছা

> এই অংশ **backend project**-এর কোড, frontend-এর না।

## ধাপ ১৩ — Service-এ ছবি মোছার call ছিলই না

🎯 **লক্ষ্য:** Division মুছলে তার ছবিও Cloudinary থেকে মোছা।

### 💡 সমস্যা কেন জরুরি

Database থেকে division মুছল, কিন্তু Cloudinary-তে ছবি রয়ে গেল। এই ছবি আর কোনো কিছুর সাথে জোড়া নেই (**orphan file**), কেউ আর ব্যবহার করবে না, কিন্তু জায়গা নেয়। Cloudinary-র free plan-এ জায়গা সীমিত; সময়ের সাথে এমন ছবি জমে জমে plan শেষ হয়ে যায়।

### 🛠️ কাজ

`division.service.ts`-এ দেখা গেল delete-এর সময় ছবি মোছার function ডাকাই হয়নি। এখন:

```ts
const deleteDivision = async (id: string) => {
  const division = await Division.findById(id);   // ১. আগে খুঁজে নাও (ছবির URL লাগবে)
  if (!division) {
    throw new Error("Division not found.");
  }

  await Division.findByIdAndDelete(id);           // ২. database থেকে মোছো

  if (division.thumbnail) {
    await deleteImageFromCLoudinary(division.thumbnail);  // ৩. ছবি থাকলে Cloudinary থেকেও
  }

  return null;
};
```

💡 **কেন আগে `findById`:** সরাসরি `findByIdAndDelete` করলেও মুছত, কিন্তু তখন ছবির URL জানা থাকত না। তাই আগে খুঁজে `division.thumbnail` হাতে রাখা, তারপর মোছা।

## ধাপ ১৪ — URL থেকে `public_id` বের করা (regex)

🎯 **লক্ষ্য:** Cloudinary-কে বলা কোন ছবি মুছবে।

### 💡 সমস্যা

Database-এ রাখা আছে ছবির **পুরো URL**:

```
https://res.cloudinary.com/demo/image/upload/v1753126572/divisions/sylhet-ab12.jpg
```

কিন্তু Cloudinary-র মোছার function (`cloudinary.uploader.destroy`) URL চায় না, চায় **`public_id`**: ছবির নিজস্ব নাম, folder সহ, extension ছাড়া:

```
divisions/sylhet-ab12
```

### 🛠️ কাজ

```ts
export const deleteImageFromCLoudinary = async (url: string) => {
  try {
    const regex = /\/upload\/(?:v\d+\/)?(.+?)\.[a-z0-9]+$/i;
    const match = url.match(regex);

    if (!match?.[1]) {
      console.warn("Could not extract public_id from:", url);
      return;
    }

    const public_id = match[1];
    const result = await cloudinary.uploader.destroy(public_id);
    console.log(`Cloudinary destroy ${public_id}:`, result); // { result: "ok" } বা { result: "not found" }
  } catch (error: any) {
    throw new AppError(401, "Cloudinary image deletion failed", error.message);
  }
};
```

### 💡 Regex টুকরো টুকরো করে

```
/\/upload\/(?:v\d+\/)?(.+?)\.[a-z0-9]+$/i
```

| টুকরো | মানে | URL-এর যে অংশ মেলে |
|---|---|---|
| `\/upload\/` | হুবহু `/upload/` | `/upload/` |
| `(?:v\d+\/)?` | `v` + সংখ্যা + `/`, **থাকলে** (`?`); `(?:…)` মানে দল, কিন্তু আলাদা করে রাখার দরকার নেই | `v1753126572/` |
| `(.+?)` | **যা রাখতে চাই**: যেকোনো অক্ষর, যত কম সম্ভব (`?` = lazy) | `divisions/sylhet-ab12` |
| `\.[a-z0-9]+` | একটা `.` আর তারপর extension | `.jpg` |
| `$` | এখানেই URL শেষ | — |
| `i` (শেষে) | বড়-ছোট হাতের পার্থক্য ধরবে না (`.JPG`-ও চলবে) | — |

`match[1]` = প্রথম `( )`-এর ভেতরে যা মিলেছে = `divisions/sylhet-ab12`। ঠিক `public_id`।

**আগের regex-এ কী সমস্যা ছিল (Claude-এর মতে):** সাধারণত পুরনো regex-গুলো এই দুটোর একটা ভুল করে: version অংশ (`v1753…/`) ধরে নেয় না, ফলে `public_id`-এর সামনে `v1753…/` থেকে যায়; অথবা folder-এর `/` বাদ দিয়ে শুধু শেষ নামটা নেয়। দুই ক্ষেত্রেই Cloudinary বলে `{ result: "not found" }`, আর কোনো error হয় না বলে কেউ টেরও পায় না। নতুন regex version থাকলে বাদ দেয় আর folder সহ পুরো path রাখে।

> নোট: এই regex সাধারণ upload URL-এর জন্য ঠিক। URL-এ transformation থাকলে (যেমন `/upload/w_300,c_fill/v123/…`) সেটাকেও `public_id`-এর অংশ ভেবে ভুল করবে। আমরা database-এ সরাসরি upload-এর URL রাখি, তাই এখন সমস্যা নেই। সবচেয়ে নিরাপদ উপায় উন্নতি ৬-এ।

✅ **যাচাই:** ছবিসহ একটা division মুছুন → backend-এর console-এ `Cloudinary destroy divisions/…: { result: 'ok' }` → Cloudinary-র Media Library থেকে ছবি চলে গেছে।

---

# কী বাকি: সমস্যা চেনা

| # | কোথায় | সমস্যা |
|---|---|---|
| ১ | Uploader-এর লেখা বনাম schema | Uploader-এ লেখা "**max 2 MB**" (default `hint`), কিন্তু schema নেয় **1 MB** পর্যন্ত। 1.5 MB-এর ছবি দিলে user লেখা দেখে ঠিক ভাবে, তারপর error পায় |
| ২ | `redux/features/` | `auth` ছোট, `Tour`/`Division` বড় হাতের (ধাপ ৯) |
| ৩ | Error-এর message | `error.data?.message` শুধু; Zod-এর field error (`errorSources`) বাদ পড়ে। `getErrorMessage` (১৭ নম্বর নোট) ব্যবহার হচ্ছে না |
| ৪ | `console.log(res)` | Submit-এর পরে; আর `lib/axios.ts`-এর interceptor-এও `console.log("Axios", config)` এখনো আছে (১৩ নম্বর নোটে সরাতে বলা) |
| ৫ | `DeleteConfirmation` | `itemName` যোগ হয়েছে ভালো, কিন্তু ১৮ নম্বর নোটের উন্নতি (মোছা শেষ না হওয়া পর্যন্ত dialog খোলা, "Deleting…") এই version-এ নেই; Delete চাপামাত্র বন্ধ হয় |
| ৬ | Endpoint-এ type | `getDivisions`-এ type নেই, তাই page-এ `const divisions: IDivision[] = data ?? []` দিয়ে হাতে type বসাতে হচ্ছে |
| ৭ | Edit button | কিছু করে না, কিন্তু চালু দেখায় |
| ৮ | Backend: ছবি মোছা ব্যর্থ হলে | Division **আগেই** database থেকে মুছে গেছে, তারপর Cloudinary ব্যর্থ হলে পুরো request error দেয়। User দেখে "Failed to delete", অথচ division আসলে মুছে গেছে |
| ৯ | Backend: status code | Cloudinary ব্যর্থ হলে `401`। কিন্তু 401 মানে "login নেই" (১৬ নম্বর নোট), এখানে সমস্যা server-এর, তাই `500` হওয়া উচিত। আর "not found"-এ সাধারণ `Error` (status ছাড়া), `AppError(404)` হওয়া উচিত |
| ১০ | ছোটখাটো | `bg-destructive/10`-এর বদলে `bg-destructive-soft` token; শিরোনামে `text-foreground`; uploader-এ `[...].join(" ")`-এর বদলে `cn()`; `bg-gradient-to-t` (Tailwind v3-এর নাম) |

---

# উন্নতি

## উন্নতি ১ — Uploader-এর লেখা আর schema-র মাপ এক জায়গা থেকে

মাপ একটা constant-এ রেখে দুই জায়গায় ব্যবহার, যাতে কখনো আলাদা না হয়:

```ts
const MAX_SIZE_MB = 1;
const MAX_SIZE = MAX_SIZE_MB * 1024 * 1024;

// schema-তে
.refine((file) => file.size <= MAX_SIZE, { error: `Image must be ${MAX_SIZE_MB} MB or smaller.` })

// uploader-এ
<ImageUploader hint={`PNG, JPG or WEBP · max ${MAX_SIZE_MB} MB`} ... />
```

## উন্নতি ২ — Folder ছোট হাতের

`redux/features/Tour` → `tour`, `Division` → `division` (ধাপ ৯-এর দুই ধাপের rename)। চূড়ান্ত কোডে import `@/redux/features/division/division.api`।

## উন্নতি ৩ — একই helper সব জায়গায়

১৮ নম্বর নোটের `runWithToast` (loading → success/error toast, `getErrorMessage` দিয়ে)। Add আর delete দুটোতেই এক লাইন। আর `console.log(res)` বাদ।

## উন্নতি ৪ — `DeleteConfirmation`: আপনার `itemName` + ১৮ নম্বর নোটের pending

দুটোর ভালো অংশ মিলিয়ে চূড়ান্ত component নিচে। `onConfirm` Promise দিলে মোছা শেষ না হওয়া পর্যন্ত dialog খোলা, "Deleting…"।

## উন্নতি ৫ — Endpoint-এ type

```ts
addDivision: builder.mutation<IResponse<IDivision>, FormData>(...)
getDivisions: builder.query<IDivision[], void>(...)   // transformResponse: (response: IResponse<IDivision[]>) => response.data
removeDivision: builder.mutation<IResponse<null>, string>(...)
```

এখন page-এ `const { data: divisions = [] } = useGetDivisionsQuery()`, হাতে type লাগে না।

## উন্নতি ৬ — Backend: ছবি মোছা ব্যর্থ হলেও request সফল, আর ঠিক status code

**নীতি:** user-এর আসল কাজ ছিল division মোছা, সেটা হয়ে গেছে। ছবি পরিষ্কার করা পেছনের কাজ (cleanup); সেটা ব্যর্থ হলে log করে রাখো, কিন্তু user-কে "ব্যর্থ" বোলো না।

```ts
// backend: division.service.ts
const deleteDivision = async (id: string) => {
  const division = await Division.findById(id);
  if (!division) {
    throw new AppError(404, "Division not found");
  }

  await Division.findByIdAndDelete(id);

  // Cleanup: ব্যর্থ হলেও division মোছা সফল; শুধু log
  if (division.thumbnail) {
    try {
      await deleteImageFromCLoudinary(division.thumbnail);
    } catch (error) {
      console.error("Cloudinary cleanup failed for division", id, error);
    }
  }

  return null;
};
```

```ts
// backend: deleteImageFromCLoudinary-এর catch-এ
throw new AppError(500, "Cloudinary image deletion failed", error.message);   // 401 না
```

(`AppError` আর status লেখার ধরন আপনার backend-এ যেভাবে আছে, যেমন `httpStatus.NOT_FOUND`, সেভাবেই লিখবেন।)

> ➕ সংযোজন — regex ছাড়াই, সবচেয়ে নিরাপদ: ছবি upload-এর সময় Cloudinary নিজেই `public_id` ফেরত দেয় (`result.public_id`)। Division-এর model-এ URL-এর পাশাপাশি `thumbnailPublicId` field রেখে সেটাও সংরক্ষণ করলে, মোছার সময় সরাসরি `cloudinary.uploader.destroy(division.thumbnailPublicId)`; URL ভাঙার কোনো regex লাগে না, transformation-এর সমস্যাও নেই। এখনকার regex কাজ করছে, তাই এটা পরে, model বদলানোর সুযোগ পেলে।

## উন্নতি ৭ — ছোটখাটো

- Uploader: class জোড়া লাগাতে `[...].join(" ")`-এর বদলে project-এর `cn()` (একই কাজ, আর Tailwind-এর পরস্পরবিরোধী class নিজে ঠিক করে, যেমন `border-border` আর `border-destructive` একসাথে থাকলে শেষেরটা রাখে)।
- Uploader: `bg-gradient-to-t` → `bg-linear-to-t` (Tailwind v4-এর নাম, ১১ নম্বর নোট)। ছবির ওপরের `text-white` ঠিক আছে (নিয়মের ব্যতিক্রম: ছবির ওপরের লেখা)।
- Table-এর ছবিতে `width`/`height` (১৫ নম্বর নোট: layout shift বন্ধ)।
- Error-এর icon-এর ঘরে `bg-destructive/10` → `bg-destructive-soft`।
- Edit button: endpoint আসা পর্যন্ত `disabled`, title "Coming soon" (১৭ নম্বর নোটের মতো)।
- Menu: "Add Division" → "Divisions", URL `/admin/divisions` (ধাপ ২)।

---

# চূড়ান্ত কোড

> ১৭ আর ১৮ নম্বর নোটের `getErrorMessage` আর `runWithToast` ধরে নেওয়া হয়েছে। Folder ছোট হাতের (উন্নতি ২) ধরে import লেখা; এখনো না বদলালে `Division` লিখবেন।

### `src/redux/features/division/division.api.ts`

```ts
import { baseApi } from "@/redux/baseApi";
import type { IDivision, IResponse } from "@/types";

export const divisionApi = baseApi.injectEndpoints({
  endpoints: (builder) => ({
    addDivision: builder.mutation<IResponse<IDivision>, FormData>({
      query: (divisionData) => ({
        url: "/division/create",
        method: "POST",
        data: divisionData,
        // Content-Type দেবেন না: browser নিজে multipart + boundary বসায়
      }),
      invalidatesTags: ["DIVISION"],
    }),

    getDivisions: builder.query<IDivision[], void>({
      query: () => ({
        url: "/division",
        method: "GET",
      }),
      providesTags: ["DIVISION"],
      transformResponse: (response: IResponse<IDivision[]>) => response.data,
    }),

    removeDivision: builder.mutation<IResponse<null>, string>({
      query: (divisionId) => ({
        url: `/division/${divisionId}`,
        method: "DELETE",
      }),
      invalidatesTags: ["DIVISION"],
    }),
  }),
});

export const { useAddDivisionMutation, useGetDivisionsQuery, useRemoveDivisionMutation } = divisionApi;
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
  /** "Sylhet" দিলে শিরোনাম হয়: Delete "Sylhet"? */
  itemName?: string;
  title?: string;
  description?: ReactNode;
  confirmText?: string;
  cancelText?: string;
}

export function DeleteConfirmation({
  children,
  onConfirm,
  itemName,
  title,
  description = "This action can't be undone. It will be permanently deleted.",
  confirmText = "Delete",
  cancelText = "Cancel",
}: IProps) {
  const [open, setOpen] = useState(false);
  const [isPending, setIsPending] = useState(false);

  const dialogTitle = title ?? (itemName ? `Delete "${itemName}"?` : "Are you sure?");

  const handleConfirm = async (event: MouseEvent<HTMLButtonElement>) => {
    event.preventDefault(); // Radix যেন এখনই বন্ধ না করে
    setIsPending(true);
    try {
      await onConfirm();
      setOpen(false);
    } catch {
      // ব্যর্থ: dialog খোলা থাকে; বার্তা onConfirm দেখিয়েছে
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
          <AlertDialogTitle>{dialogTitle}</AlertDialogTitle>
          <AlertDialogDescription>{description}</AlertDialogDescription>
        </AlertDialogHeader>

        <AlertDialogFooter>
          <AlertDialogCancel disabled={isPending}>{cancelText}</AlertDialogCancel>
          <AlertDialogAction variant="destructive" onClick={handleConfirm} disabled={isPending}>
            {isPending ? "Deleting…" : confirmText}
          </AlertDialogAction>
        </AlertDialogFooter>
      </AlertDialogContent>
    </AlertDialog>
  );
}
```

(১৮ নম্বর নোটের tour type page-এ `title={...}` দিয়ে ব্যবহার করা হয়েছিল, সেটাও এই version-এ আগের মতোই চলবে; চাইলে সেখানেও `itemName={item.name}` দিয়ে ছোট করা যায়।)

### `src/components/modules/Admin/Division/AddDivisionModal.tsx`

```tsx
import { Button } from "@/components/ui/button";
import {
  Dialog,
  DialogClose,
  DialogContent,
  DialogDescription,
  DialogFooter,
  DialogHeader,
  DialogTitle,
  DialogTrigger,
} from "@/components/ui/dialog";
import { Field, FieldDescription, FieldError, FieldLabel } from "@/components/ui/field";
import ImageUploader from "@/components/ui/image-uploader";
import { Input } from "@/components/ui/input";
import { Textarea } from "@/components/ui/textarea";
import { useAddDivisionMutation } from "@/redux/features/division/division.api";
import { runWithToast } from "@/utils/runWithToast";
import { zodResolver } from "@hookform/resolvers/zod";
import { Loader2, Plus } from "lucide-react";
import { useState } from "react";
import { Controller, useForm } from "react-hook-form";
import * as z from "zod";

const MAX_SIZE_MB = 1;
const MAX_SIZE = MAX_SIZE_MB * 1024 * 1024;
const IMAGE_TYPES = ["image/png", "image/jpeg", "image/webp"];

const formSchema = z.object({
  name: z
    .string()
    .trim()
    .min(2, { error: "Name must be at least 2 characters long." })
    .max(50, { error: "Name cannot exceed 50 characters." }),
  description: z.string().trim().max(300, { error: "Description cannot exceed 300 characters." }).optional(),
  thumbnail: z
    .instanceof(File)
    .refine((file) => IMAGE_TYPES.includes(file.type), { error: "Only PNG, JPG or WEBP allowed." })
    .refine((file) => file.size <= MAX_SIZE, { error: `Image must be ${MAX_SIZE_MB} MB or smaller.` })
    .optional(),
});

type FormValues = z.infer<typeof formSchema>;

// "Cox's Bazar" → "coxs-bazar" (শুধু ইংরেজি অক্ষর, সংখ্যা আর - রাখে)
const makeSlug = (text: string) =>
  text
    .toLowerCase()
    .trim()
    .replace(/[^a-z0-9\s-]/g, "")
    .replace(/\s+/g, "-")
    .replace(/-+/g, "-");

export default function AddDivisionModal() {
  const [open, setOpen] = useState(false);
  const [addDivision, { isLoading }] = useAddDivisionMutation();

  const form = useForm<FormValues>({
    resolver: zodResolver(formSchema),
    defaultValues: { name: "", description: "", thumbnail: undefined },
  });

  const name = form.watch("name");

  const onSubmit = async (data: FormValues) => {
    const formData = new FormData();
    // FormData-তে object যায় না, তাই JSON string; key-র নাম backend-এর সাথে হুবহু
    formData.append(
      "data",
      JSON.stringify({ name: data.name, slug: makeSlug(data.name), description: data.description }),
    );
    if (data.thumbnail) formData.append("file", data.thumbnail);

    try {
      await runWithToast(addDivision(formData).unwrap(), {
        loading: "Uploading division…",
        success: `"${data.name}" added`,
        error: "Couldn't add the division.",
      });
      form.reset();
      setOpen(false);
    } catch {
      // Error toast দেখানো হয়েছে; modal খোলা থাকে যাতে ঠিক করে আবার চেষ্টা করা যায়
    }
  };

  // Upload চলার সময় বন্ধ করতে দেব না; বন্ধ হলে form খালি
  const handleOpenChange = (value: boolean) => {
    if (isLoading) return;
    setOpen(value);
    if (!value) form.reset();
  };

  return (
    <Dialog open={open} onOpenChange={handleOpenChange}>
      <DialogTrigger asChild>
        <Button type="button" className="h-11 gap-2 rounded-lg px-4 text-[14.5px] font-semibold">
          <Plus className="size-4.5" strokeWidth={2.4} />
          Add division
        </Button>
      </DialogTrigger>

      <DialogContent className="max-h-[90vh] gap-0 overflow-x-hidden overflow-y-auto rounded-xl p-0 sm:max-w-120">
        <DialogHeader className="border-b border-border px-5.5 py-4.5 text-left">
          <DialogTitle className="text-lg font-bold">Add division</DialogTitle>
          <DialogDescription className="text-[13.5px]">
            Each division gets its own public page at /divisions/slug.
          </DialogDescription>
        </DialogHeader>

        <form
          id="add-division"
          className="flex flex-col gap-4 px-5.5 py-5"
          onSubmit={form.handleSubmit(onSubmit)}
          noValidate
        >
          <fieldset disabled={isLoading} className="flex flex-col gap-4 disabled:opacity-60">
            {/* Thumbnail */}
            <Controller
              name="thumbnail"
              control={form.control}
              render={({ field, fieldState }) => (
                <Field data-invalid={fieldState.invalid} className="gap-1.5">
                  <FieldLabel htmlFor="dv-thumbnail" className="text-sm font-semibold">
                    Thumbnail
                  </FieldLabel>
                  <ImageUploader
                    id="dv-thumbnail"
                    value={field.value}
                    onChange={field.onChange}
                    onBlur={field.onBlur}
                    invalid={fieldState.invalid}
                    hint={`PNG, JPG or WEBP · max ${MAX_SIZE_MB} MB`}
                  />
                  <FieldDescription className="text-[13px]">
                    Shown on the home page and the division banner. 1600 × 900 px recommended.
                  </FieldDescription>
                  {fieldState.invalid && <FieldError errors={[fieldState.error]} />}
                </Field>
              )}
            />

            {/* Name */}
            <Controller
              name="name"
              control={form.control}
              render={({ field, fieldState }) => (
                <Field data-invalid={fieldState.invalid} className="gap-1.5">
                  <FieldLabel htmlFor="dv-name" className="text-sm font-semibold">
                    Name
                  </FieldLabel>
                  <Input
                    {...field}
                    id="dv-name"
                    placeholder="e.g. Sylhet"
                    autoComplete="off"
                    aria-invalid={fieldState.invalid}
                    className="h-11 rounded-lg px-3.5 text-[15px]"
                  />
                  {fieldState.invalid && <FieldError errors={[fieldState.error]} />}
                </Field>
              )}
            />

            {/* Slug (নাম থেকে নিজে, শুধু দেখানোর জন্য) */}
            <Field className="gap-1.5">
              <FieldLabel htmlFor="dv-slug" className="text-sm font-semibold">
                Slug
              </FieldLabel>
              <Input
                id="dv-slug"
                value={makeSlug(name)}
                placeholder="sylhet"
                readOnly
                className="h-11 rounded-lg bg-muted px-3.5 text-[15px] text-muted-foreground"
              />
              <FieldDescription className="text-[13px]">Generated automatically from the name.</FieldDescription>
            </Field>

            {/* Description */}
            <Controller
              name="description"
              control={form.control}
              render={({ field, fieldState }) => (
                <Field data-invalid={fieldState.invalid} className="gap-1.5">
                  <FieldLabel htmlFor="dv-desc" className="text-sm font-semibold">
                    Description
                  </FieldLabel>
                  <Textarea
                    {...field}
                    id="dv-desc"
                    placeholder="A short line about what makes this division special…"
                    aria-invalid={fieldState.invalid}
                    className="min-h-25 resize-y rounded-lg px-3.5 py-2.5 text-[15px]"
                  />
                  {fieldState.invalid && <FieldError errors={[fieldState.error]} />}
                </Field>
              )}
            />
          </fieldset>
        </form>

        <DialogFooter className="border-t border-border bg-muted px-5.5 py-4">
          <DialogClose asChild>
            <Button variant="outline" type="button" disabled={isLoading} className="h-11 rounded-lg px-4 font-semibold">
              Cancel
            </Button>
          </DialogClose>
          <Button
            type="submit"
            form="add-division"
            disabled={isLoading}
            className="h-11 min-w-36 gap-2 rounded-lg px-4 font-semibold"
          >
            {isLoading ? (
              <>
                <Loader2 className="size-4 animate-spin" />
                Uploading…
              </>
            ) : (
              "Save division"
            )}
          </Button>
        </DialogFooter>
      </DialogContent>
    </Dialog>
  );
}
```

### `src/pages/Admin/AddDivision.tsx`

```tsx
import { DeleteConfirmation } from "@/components/DeleteConfirmation";
import AddDivisionModal from "@/components/modules/Admin/Division/AddDivisionModal";
import { Button } from "@/components/ui/button";
import { Skeleton } from "@/components/ui/skeleton";
import { useGetDivisionsQuery, useRemoveDivisionMutation } from "@/redux/features/division/division.api";
import type { IDivision } from "@/types";
import { runWithToast } from "@/utils/runWithToast";
import { AlertCircle, ImageIcon, MapPinned, Pencil, RefreshCw, Trash2 } from "lucide-react";

const COLS = 5;

export default function AddDivision() {
  const { data: divisions = [], isLoading, isError, isFetching, refetch } = useGetDivisionsQuery();
  const [removeDivision] = useRemoveDivisionMutation();

  const handleRemoveDivision = (division: IDivision) =>
    runWithToast(removeDivision(division._id).unwrap(), {
      loading: `Deleting "${division.name}"…`,
      success: `"${division.name}" deleted`,
      error: "Couldn't delete the division.",
    });

  return (
    <div className="flex w-full flex-col gap-5">
      {/* Header + Add button */}
      <div className="flex flex-wrap items-end justify-between gap-3.5">
        <div>
          <h1 className="text-3xl font-bold tracking-tight text-foreground">Divisions</h1>
          <p className="text-muted-foreground">
            {isLoading ? "Loading…" : `${divisions.length} divisions`} · each has a public page at /divisions/slug
          </p>
        </div>
        <AddDivisionModal />
      </div>

      {/* Table */}
      <section className="overflow-hidden rounded-xl border border-border bg-card shadow-card">
        <div className="overflow-x-auto">
          <table className="w-full min-w-[680px] border-collapse text-left text-sm">
            <thead className="bg-muted">
              <tr className="text-[13px] font-semibold text-muted-foreground">
                <th scope="col" className="w-14 px-4.5 py-3">#</th>
                <th scope="col" className="px-4.5 py-3">Division</th>
                <th scope="col" className="px-4.5 py-3">Slug</th>
                <th scope="col" className="px-4.5 py-3">Description</th>
                <th scope="col" className="px-4.5 py-3 text-right">Actions</th>
              </tr>
            </thead>

            <tbody className={isFetching && !isLoading ? "opacity-60 transition-opacity" : ""}>
              {/* ── Loading ── */}
              {isLoading &&
                Array.from({ length: 5 }).map((_, i) => (
                  <tr key={i} className="border-t border-border">
                    <td className="px-4.5 py-3"><Skeleton className="h-4 w-5" /></td>
                    <td className="px-4.5 py-3">
                      <div className="flex items-center gap-3">
                        <Skeleton className="h-11 w-16 rounded-lg" />
                        <Skeleton className="h-4 w-28" />
                      </div>
                    </td>
                    <td className="px-4.5 py-3"><Skeleton className="h-4 w-24" /></td>
                    <td className="px-4.5 py-3"><Skeleton className="h-4 w-56" /></td>
                    <td className="px-4.5 py-3">
                      <div className="flex justify-end gap-2">
                        <Skeleton className="size-9 rounded-lg" />
                        <Skeleton className="size-9 rounded-lg" />
                      </div>
                    </td>
                  </tr>
                ))}

              {/* ── Error ── */}
              {isError && (
                <tr className="border-t border-border">
                  <td colSpan={COLS} className="px-4.5 py-14">
                    <div className="flex flex-col items-center gap-3 text-center">
                      <span className="flex size-12 items-center justify-center rounded-full bg-destructive-soft text-destructive">
                        <AlertCircle className="size-5" />
                      </span>
                      <div>
                        <p className="font-semibold text-foreground">Couldn't load divisions</p>
                        <p className="text-[13.5px] text-muted-foreground">Check your connection and try again.</p>
                      </div>
                      <Button variant="outline" onClick={() => refetch()} className="h-10 gap-2 rounded-lg">
                        <RefreshCw className="size-4" />
                        Try again
                      </Button>
                    </div>
                  </td>
                </tr>
              )}

              {/* ── Empty ── */}
              {!isLoading && !isError && divisions.length === 0 && (
                <tr className="border-t border-border">
                  <td colSpan={COLS} className="px-4.5 py-14">
                    <div className="flex flex-col items-center gap-3 text-center">
                      <span className="flex size-12 items-center justify-center rounded-full bg-accent text-accent-foreground">
                        <MapPinned className="size-5" />
                      </span>
                      <div>
                        <p className="font-semibold text-foreground">No divisions yet</p>
                        <p className="text-[13.5px] text-muted-foreground">Add your first division to start grouping tours.</p>
                      </div>
                    </div>
                  </td>
                </tr>
              )}

              {/* ── Data ── */}
              {!isLoading &&
                !isError &&
                divisions.map((division, index) => (
                  <tr key={division._id} className="border-t border-border transition-colors hover:bg-muted/60">
                    <td className="px-4.5 py-3 text-muted-foreground">{index + 1}</td>

                    {/* Thumbnail + name */}
                    <td className="px-4.5 py-3">
                      <div className="flex items-center gap-3">
                        <div className="flex h-11 w-16 shrink-0 items-center justify-center overflow-hidden rounded-lg bg-accent text-accent-foreground">
                          {division.thumbnail ? (
                            <img
                              src={division.thumbnail}
                              alt={division.name}
                              width={64}
                              height={44}
                              loading="lazy"
                              className="size-full object-cover"
                            />
                          ) : (
                            <ImageIcon className="size-4.5" />
                          )}
                        </div>
                        <span className="font-semibold text-foreground">{division.name}</span>
                      </div>
                    </td>

                    <td className="px-4.5 py-3 text-muted-foreground">/{division.slug}</td>

                    <td className="max-w-xs px-4.5 py-3">
                      <p className="truncate text-muted-foreground" title={division.description}>
                        {division.description || "—"}
                      </p>
                    </td>

                    <td className="px-4.5 py-3">
                      <div className="flex justify-end gap-2">
                        {/* Edit-এর endpoint এলে চালু হবে */}
                        <Button
                          variant="outline"
                          size="icon"
                          type="button"
                          disabled
                          title="Coming soon"
                          aria-label={`Edit ${division.name}`}
                          className="size-9 rounded-lg disabled:opacity-40"
                        >
                          <Pencil className="size-4" />
                        </Button>

                        <DeleteConfirmation
                          itemName={division.name}
                          description="This removes the division and its public page. This can't be undone."
                          onConfirm={() => handleRemoveDivision(division)}
                        >
                          <Button
                            variant="outline"
                            size="icon"
                            type="button"
                            aria-label={`Delete ${division.name}`}
                            className="size-9 rounded-lg text-destructive hover:text-destructive"
                          >
                            <Trash2 className="size-4" />
                          </Button>
                        </DeleteConfirmation>
                      </div>
                    </td>
                  </tr>
                ))}
            </tbody>
          </table>
        </div>

        {/* Footer */}
        {!isLoading && !isError && divisions.length > 0 && (
          <div className="flex items-center justify-between gap-3 border-t border-border bg-muted px-4.5 py-3 text-[13.5px] text-muted-foreground">
            <span>Showing {divisions.length} divisions</span>
          </div>
        )}
      </section>
    </div>
  );
}
```

### `src/components/ui/image-uploader.tsx` — শুধু বদলানো লাইন

```tsx
import { cn } from "@/lib/utils";

// Preview-এর নিচের অন্ধকার পট্টি
<div className="absolute inset-x-0 bottom-0 flex items-center justify-between gap-2 bg-linear-to-t from-black/60 to-transparent p-2.5 pt-8">

// Drop zone-এর class
className={cn(
  "flex h-44 cursor-pointer flex-col items-center justify-center gap-2 rounded-xl border-2 border-dashed text-center transition-colors",
  isDragging ? "border-primary bg-accent" : "border-border bg-muted hover:border-primary hover:bg-accent",
  invalid && "border-destructive",
)}
```

বাকি সব আপনার কোডের মতোই।

### `src/routes/adminSidebarItems.ts` — শুধু item

```ts
{ title: "Divisions", url: "/admin/divisions", component: AddDivision },
```

### Backend — `division.service.ts` আর Cloudinary helper

উন্নতি ৬-এর কোড।

---

# Check-list

**Create**
- [ ] "Add division" → modal; ছবি click বা drag করে বাছা যায়, সাথে সাথে preview
- [ ] Replace আর Remove কাজ করে; Remove-এর পরে একই ছবি আবার বাছা যায়
- [ ] 1 MB-এর বড় ছবি বা PDF → ছবির নিচে লাল error; uploader-এর লেখাতেও "max 1 MB"
- [ ] নাম লিখলে slug সাথে সাথে আপডেট
- [ ] Network → `create` → Payload-এ `data` (JSON লেখা) আর `file`; Request Headers-এ `multipart/form-data; boundary=…`
- [ ] Upload চলার সময় field ফিকে আর বন্ধ, Escape/Cancel-এ modal বন্ধ হয় না
- [ ] সফল হলে modal বন্ধ, পরের বার খালি form, table-এ নতুন row (reload ছাড়া)
- [ ] Backend error হলে লাল toast, modal খোলা থাকে

**Read**
- [ ] Slow 4G-তে skeleton row; backend বন্ধ থাকলে error + "Try again" কাজ করে; খালি হলে "No divisions yet"

**Delete**
- [ ] Dialog-এ `Delete "Sylhet"?`; Delete চাপলে "Deleting…", শেষে নিজে বন্ধ
- [ ] Cloudinary-র Media Library থেকেও ছবি চলে যায়; backend console-এ `{ result: 'ok' }`
- [ ] Cloudinary ব্যর্থ হলেও (যেমন ভুল credentials) division মুছে যায় আর user-কে "failed" দেখায় না; backend console-এ log

**নাম আর গঠন**
- [ ] `redux/features/`-এ সব folder ছোট হাতের (`auth`, `tour`, `division`), import-গুলোও
- [ ] Menu-তে "Divisions", URL `/admin/divisions`
- [ ] `console.log(res)` আর axios interceptor-এর `console.log` সরানো
