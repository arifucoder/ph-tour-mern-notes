# Frontend Notes — ২০: Tour CRUD — বড় Form, Searchable Select, একাধিক ছবি আর Pagination

Tour হলো এই project-এর মূল জিনিস। এতদিনের সব কাজ (tour type, division, image upload, RTK Query, React Hook Form + Zod) এখানে এসে একসাথে মিলছে। Form-টা বড়: ৬টা ভাগে প্রায় ১৮টা field, তার মধ্যে dropdown, তারিখ, সংখ্যা, একাধিক ছবি, আর "যত খুশি item যোগ করো" ধরনের তালিকা।

এই অংশে আপনি teacher-কে হুবহু follow করেননি (তাঁর design আপনার পছন্দ হয়নি), নিজে Claude-এর সাহায্যে বানিয়েছেন, কিন্তু teacher-এর ব্যাখ্যাগুলো শুনেছেন। তাই এই নোটের কাঠামো:
- **ধাপগুলো আপনার বানানোর ক্রমে**, আপনার কোড দিয়ে।
- **Teacher-এর ধারণাগুলো** (reusable component নিয়ে তাঁর মত, Day Picker, `useFieldArray`) জায়গামতো আলাদা box-এ, আপনার পদ্ধতির সাথে তুলনা করে।
- শেষে **উন্নতি আর চূড়ান্ত কোড**।

প্রতিটা ধাপে 🎯 লক্ষ্য, 🛠️ কাজ, 💡 কেন, ✅ যাচাই।

> নোট: আপনার project-এর version মেনে লেখা (shadcn `radix-nova`, Zod 4, React Hook Form 7, RTK 2, React Router 8)। আপনার বর্তমান কোড পড়ে নিয়ে লেখা; পুরনো ধরনের কোড দিয়ে কিছু বদলানো হয়নি।

---

## ধাপের মানচিত্র

| ভাগ | ধাপ | শেষে কী হবে |
|---|---|---|
| **ক. প্রস্তুতি** | ১–৩ | Backend কী চায় জানা; Searchable Select বানানো |
| **খ. Form-এর design** | ৪–৫ | Add tour page আর Tours page (নমুনা data) |
| **গ. Form জীবন্ত করা** | ৬–১১ | Division/type API থেকে, তারিখ, সংখ্যা, তালিকা, একাধিক ছবি সব কাজ করে |
| **ঘ. পাঠানো** | ১২–১৩ | FormData দিয়ে tour তৈরি হয় |
| **ঙ. তালিকা, Pagination, মোছা** | ১৪–১৬ | Tours page-এ আসল data, পাতা বদলানো, মোছা (ছবিসহ) |
| **উন্নতি ও চূড়ান্ত কোড** | — | — |

### শেষে যে ফাইলগুলো থাকবে

```
src/
├── components/ui/
│   ├── popover.tsx, command.tsx            ← shadcn
│   ├── searchable-select.tsx               ← reusable (ধাপ ৩)
│   └── multi-image-uploader.tsx            ← reusable (ধাপ ১১)
├── pages/Admin/
│   ├── AddTour.tsx                         ← বড় form (ধাপ ৪, ৬–১৩)
│   └── Tours.tsx                           ← তালিকা + pagination (ধাপ ৫, ১৪–১৬)
├── redux/features/tour/
│   └── tour.api.ts                         ← addTour, getTours, removeTour
└── types/
    └── tour.type.ts                        ← ITour, IMeta
```

---

# ভাগ ক: প্রস্তুতি

## ধাপ ১ — Postman-এ দেখা: tour তৈরিতে কী লাগে

🎯 **লক্ষ্য:** Backend-এর create tour API কী চায়, আগে জানা।

🛠️ **কাজ:** Postman-এ `POST /tour/create` দেখলে (division-এর মতো form-data, ১৯ নম্বর নোট):

| Key | Type | মান |
|---|---|---|
| `data` | Text | Tour-এর সব তথ্য JSON string হিসেবে |
| `files` | File (**একাধিক**) | Tour-এর ছবিগুলো |

আর `data`-র ভেতরে দুটো জিনিস ছাড়া tour তৈরিই হয় না:

```json
{
  "title": "Sajek Valley 3-Day Escape",
  "division": "6650f1a2…",     ← division-এর _id
  "tourType": "6650e9b7…",     ← tour type-এর _id
  ...
}
```

💡 **কেন id, নাম না:** MongoDB-তে tour শুধু division-এর `_id` রাখে (reference)। পরে tour আনার সময় backend `populate` দিয়ে id-র জায়গায় পুরো division বসিয়ে দেয় (ধাপ ১৪)। তাই form-এ user নাম দেখে বাছবে ("Sylhet"), কিন্তু পাঠানো হবে id। ঠিক এই কাজটাই করবে Searchable Select।

💡 **Division-এর সাথে পার্থক্য:** সেখানে একটা ছবি, key `file` (`multer.single("file")`)। এখানে একাধিক ছবি, key **`files`** (`multer.array("files")`)। Key-র নাম backend-এর সাথে হুবহু মেলাতে হবে (১৯ নম্বর নোটের ধাপ ১)।

✅ **যাচাই:** Postman-এ একটা division id আর tour type id দিয়ে tour তৈরি হয়।

---

## ধাপ ২ — Popover আর Command install

🎯 **লক্ষ্য:** খোঁজা যায় এমন dropdown বানানোর component আনা।

🛠️ **কাজ:**

```sh
bunx --bun shadcn@latest add popover command
```

💡 **সাধারণ `Select` কেন না:** shadcn-এর `Select` (আগেই install করা) ৮টা division-এর জন্য চলে। কিন্তু পরে division বাড়লে বা tour type ৩০টা হলে scroll করে খোঁজা কষ্টের। তাই **লিখে খোঁজা যায়** এমন dropdown (combobox):

| Component | কাজ |
|---|---|
| `Popover` | একটা button-এর নিচে ভাসা box খোলে (dropdown-এর কাঠামো) |
| `Command` | খোঁজার box + তালিকা; লিখলে তালিকা নিজে ছেঁকে দেয় (ভেতরে `cmdk` library) |

দুটো মিলিয়ে shadcn-এর docs-এ "Combobox" নামে উদাহরণ আছে, সেটা থেকেই আমাদের component।

---

## ধাপ ৩ — Reusable `SearchableSelect`

🎯 **লক্ষ্য:** Division আর tour type দুই জায়গায় চলবে, এমন একটা খোঁজা যায় dropdown।

🛠️ **কাজ:**

```tsx
// src/components/ui/searchable-select.tsx
import { Button } from "@/components/ui/button";
import { Command, CommandEmpty, CommandGroup, CommandInput, CommandItem, CommandList } from "@/components/ui/command";
import { Popover, PopoverContent, PopoverTrigger } from "@/components/ui/popover";
import { cn } from "@/lib/utils";
import { Check, ChevronsUpDown } from "lucide-react";
import { useState } from "react";

export type TOption = { value: string; label: string };

type SearchableSelectProps = {
  id?: string;
  options: TOption[];
  /** controlled value (optional) — না দিলে component নিজে মনে রাখে */
  value?: string;
  onChange?: (value: string) => void;
  placeholder?: string;
  searchPlaceholder?: string;
  emptyText?: string;
  invalid?: boolean;
  disabled?: boolean;
  className?: string;
};

export default function SearchableSelect({
  id,
  options,
  value,
  onChange,
  placeholder = "Select an option",
  searchPlaceholder = "Search…",
  emptyText = "No results found.",
  invalid = false,
  disabled = false,
  className,
}: SearchableSelectProps) {
  const [open, setOpen] = useState(false);
  const [innerValue, setInnerValue] = useState("");

  const selectedValue = value ?? innerValue;
  const selected = options.find((option) => option.value === selectedValue);

  const handleSelect = (newValue: string) => {
    const next = newValue === selectedValue ? "" : newValue; // আবার click → খালি
    if (value === undefined) setInnerValue(next);
    onChange?.(next);
    setOpen(false);
  };

  return (
    // modal → Dialog-এর ভেতরে ব্যবহার করলেও তালিকা scroll হয়
    <Popover open={open} onOpenChange={setOpen} modal>
      <PopoverTrigger asChild>
        <Button
          id={id}
          type="button"
          variant="outline"
          role="combobox"
          aria-expanded={open}
          aria-invalid={invalid}
          disabled={disabled}
          className={cn(
            "h-11 w-full justify-between rounded-lg px-3.5 text-[15px] font-normal",
            !selected && "text-muted-foreground",
            className,
          )}
        >
          <span className="truncate">{selected ? selected.label : placeholder}</span>
          <ChevronsUpDown className="size-4 shrink-0 opacity-50" />
        </Button>
      </PopoverTrigger>

      <PopoverContent align="start" className="w-(--radix-popover-trigger-width) p-0">
        <Command>
          <CommandInput placeholder={searchPlaceholder} className="h-10" />
          <CommandList className="max-h-64">
            <CommandEmpty>{emptyText}</CommandEmpty>
            <CommandGroup>
              {options.map((option) => (
                <CommandItem
                  key={option.value}
                  value={option.label} // খোঁজা হবে নাম দিয়ে
                  onSelect={() => handleSelect(option.value)}
                  className="py-2"
                >
                  <span className="flex-1 truncate">{option.label}</span>
                  <Check
                    className={cn("size-4 text-primary", selectedValue === option.value ? "opacity-100" : "opacity-0")}
                  />
                </CommandItem>
              ))}
            </CommandGroup>
          </CommandList>
        </Command>
      </PopoverContent>
    </Popover>
  );
}
```

### 💡 মূল ধারণাগুলো

**১. `{ value, label }` — দেখাই এক, পাঠাই আরেক**

```ts
{ value: "6650f1a2…", label: "Sylhet" }
```

User দেখে `label` ("Sylhet"), form পায় `value` (id)। ধাপ ১-এর "নাম দেখে বাছো, id পাঠাও" ঠিক এভাবেই হয়। `TOption` type `export` করা, যাতে অন্য ফাইল এই আকারে option বানাতে পারে।

**২. Controlled আর uncontrolled, দুটোই**

```ts
const selectedValue = value ?? innerValue;
```

- বাইরে থেকে `value` দিলে (React Hook Form-এর `Controller`) সেটাই সত্য, component শুধু `onChange` দিয়ে জানায়। (**controlled**)
- `value` না দিলে নিজের `innerValue`-তে মনে রাখে, যাতে form ছাড়াও সাধারণভাবে ব্যবহার করা যায় (যেমন table-এর filter)। (**uncontrolled**)
- `if (value === undefined) setInnerValue(next)`: শুধু uncontrolled হলেই নিজে মনে রাখে।

**৩. আবার click করলে খালি**

`newValue === selectedValue ? "" : newValue`: বাছা item-এ আবার click করলে বাছাই উঠে যায়। Form-এ তখন মান `""`, আর Zod-এর `min(1)` "Please select a division" দেখায়।

**৪. খোঁজা `label` দিয়ে**

`cmdk` তালিকা ছাঁকে `CommandItem`-এর `value` দিয়ে। তাই সেখানে `option.label` দেওয়া, যাতে "syl" লিখলে "Sylhet" আসে (id দিয়ে খোঁজার মানে নেই)। Item বাছলে `handleSelect(option.value)`-এ আসল id যায়।

> নোট: দুটো item-এর `label` হুবহু এক হলে (যেমন দুটো "Sylhet") `cmdk` দুটোকে একই ভাবে, বাছাই গুলিয়ে যায়। Division আর tour type-এর নাম backend-এ unique হলে সমস্যা নেই।

**৫. `w-(--radix-popover-trigger-width)`**

Radix popover-এর ভেতরে একটা CSS variable দেয়: trigger button কত চওড়া। Tailwind v4-এ `w-(--নাম)` লিখে সেই variable দিয়ে প্রস্থ দেওয়া যায়। ফলে dropdown-এর তালিকা ঠিক button-এর সমান চওড়া হয়।

**৬. `role="combobox"`, `aria-expanded`**

Screen reader-কে বলে এটা সাধারণ button না, একটা বাছাইয়ের box, আর এখন খোলা না বন্ধ।

✅ **যাচাই:** কোনো page-এ সাময়িকভাবে `<SearchableSelect options={[{ value: "1", label: "Sylhet" }, { value: "2", label: "Dhaka" }]} />` বসিয়ে দেখুন: click করলে তালিকা, "dh" লিখলে শুধু Dhaka, বাছলে button-এ নাম আর পাশে ✓।

---

# ভাগ খ: Form-এর design

## ধাপ ৪ — Add tour page: ৬ ভাগের design

🎯 **লক্ষ্য:** বড় form-কে ভাগ করে সাজানো, যাতে চোখে ভারী না লাগে।

🛠️ **কাজ:** `src/pages/Admin/AddTour.tsx`-এ প্রথমে শুধু design (নমুনা options, কোনো form library না)। Form ৬টা ভাগে (card):

| ভাগ | Field |
|---|---|
| ১. Basic info | Title*, Division*, Tour type*, Description |
| ২. Images | একাধিক ছবি (প্রথমটা cover) |
| ৩. Location & dates | Location, Departure, Arrival, Start date, End date |
| ৪. Pricing & guests | Cost from (৳), Max guests, Min age |
| ৫. What's included | Included, Excluded, Amenities (তালিকা) |
| ৬. Tour plan | দিনভিত্তিক পরিকল্পনা (নম্বরসহ তালিকা) |

আর নিচে **sticky** action bar (Cancel + Save tour), যেটা scroll করলেও নিচে আটকে থাকে।

### 💡 একই ফাইলে ছোট helper component: `Section` আর `ListField`

```tsx
function Section({ title, description, children }: { title: string; description?: string; children: ReactNode }) {
  return (
    <section className="rounded-xl border border-border bg-card shadow-card">
      <div className="border-b border-border px-5.5 py-4">
        <h2 className="text-base font-bold text-foreground">{title}</h2>
        {description && <p className="text-[13px] text-muted-foreground">{description}</p>}
      </div>
      <div className="flex flex-col gap-4 p-5.5">{children}</div>
    </section>
  );
}
```

৬টা card-এর একই কাঠামো ৬ বার না লিখে একটা ছোট component। কিন্তু এটা `components/`-এ না, **এই ফাইলের ভেতরেই**, কারণ শুধু এই page-এ লাগে। `ListField`-ও তাই (ধাপ ১০-এ জীবন্ত হবে)।

### 💡 Sticky action bar

```tsx
<div className="sticky bottom-0 z-10 -mx-4 flex justify-end gap-2.5 border-t border-border bg-background/95 px-4 py-3.5 backdrop-blur">
```

- **`sticky bottom-0`:** form লম্বা, Save button একদম নিচে থাকলে প্রতিবার scroll করে নামতে হয়। Sticky দিলে button সবসময় screen-এর নিচে দেখা যায়, আর form শেষ হলে নিজের জায়গায় বসে যায়।
- **`bg-background/95 backdrop-blur`:** পেছনের field-এর ওপর দিয়ে গেলে একটু ঝাপসা আর প্রায়-অস্বচ্ছ, লেখা মিশে যায় না।
- **`-mx-4 px-4`:** পাশের padding-এর বাইরে পর্যন্ত টেনে পুরো চওড়া করা।

> 💬 **Teacher-এর কথা: এই field-গুলোর জন্য আবার reusable component বানানো উচিত কিনা?**
>
> Teacher-এর মত: **না।** কারণগুলো:
> - এই field-গুলো ইতিমধ্যে reusable component দিয়েই বানানো (`Input`, `Field`, `SearchableSelect`, `MultiImageUploader`)। তাদের ওপর আরেক স্তর reusable বানানো মানে অকারণ জটিলতা।
> - এই page বড় হলেও এর **একটাই কাজ**: tour যোগ করা। অন্য কোথাও এই form লাগবে না।
> - **Reusable component-এর সবচেয়ে বড় সমস্যা:** একটা জায়গার জন্য সামান্য বদল লাগলে মূল component বদলাতে হয়, আর তার প্রভাব বাকি ৫-৬ জায়গাতেও পড়ে, সেগুলোও পরীক্ষা করতে বা ঠিক করতে হয়। তখন সামলানো কঠিন হয়ে যায়।
>
> **কখন reusable বানাবেন:** একটা component দিয়ে যদি ৫-৬টা জায়গার কাজ হয়। আর বানালে মনে রাখবেন, component যেন **একটাই উদ্দেশ্য** (single purpose) পূরণ করে। `SearchableSelect` (শুধু খুঁজে বাছাই) আর `MultiImageUploader` (শুধু ছবি বাছাই) এই নিয়মের ভালো উদাহরণ; এগুলো tour, পরে booking-এর filter, profile সব জায়গায় লাগবে। আর `Section`/`ListField` এই page-এর ভেতরে, কারণ এখন শুধু এখানেই লাগে। পরে অন্য page-এ লাগলে তখন সরানো যাবে।

✅ **যাচাই:** `/admin/add-tour`-এ ৬টা card, নিচে আটকে থাকা Save bar; ছোট screen-এ সব এক column-এ।

---

## ধাপ ৫ — Tours page (নমুনা data) আর route

🎯 **লক্ষ্য:** সব tour-এর table, আর sidebar-এ দুটো link।

🛠️ **কাজ:** `src/pages/Admin/Tours.tsx`, division page-এর মতো table (১৯ নম্বর নোট), কিন্তু বেশি column: `#`, Tour (cover + নাম + location), Division, Type, Dates, Price from, Guests, Actions। প্রথমে নমুনা ৫টা tour।

```ts
// src/routes/adminSidebarItems.ts
const Tours = lazy(() => import("@/pages/Admin/Tours"));
const AddTour = lazy(() => import("@/pages/Admin/AddTour"));

{ title: "Tours", url: "/admin/tours", component: Tours },
{ title: "Add Tour", url: "/admin/add-tour", component: AddTour },
```

### 💡 তারিখ আর দাম সুন্দর করে দেখানো

```ts
const formatDate = (date?: string) =>
  date ? new Date(date).toLocaleDateString("en-GB", { day: "numeric", month: "short", year: "numeric" }) : "";
// "2026-11-12" → "12 Nov 2026"

const formatPrice = (amount?: number) => (amount === undefined ? "—" : `৳${amount.toLocaleString("en-IN")}`);
// 12500 → "৳12,500"   (en-IN: ভারতীয়/বাংলাদেশি ধরনে কমা, ১,২৫,০০০)
```

- **`toLocaleDateString("en-GB", …)`:** browser-এর নিজের তারিখ সাজানোর ক্ষমতা; বাড়তি library লাগে না (১৫ নম্বর নোট: ছোট কাজে বড় library না)।
- **`"en-IN"`:** লাখের হিসাবে কমা (১,২৫,০০০), আমাদের দেশের পরিচিত ধরন।
- **Start আর end একই দিন হলে** একটাই তারিখ দেখায় ("Old Dhaka Food Walk": এক দিনের)।

**`Add tour` button:** `<Button asChild><Link to="/admin/add-tour">` (link-কে button-এর চেহারা, ১০ নম্বর নোট)।

✅ **যাচাই:** Sidebar-এর "Tours"-এ table, "Add tour" button চাপলে form page।

---

# ভাগ গ: Form জীবন্ত করা

## ধাপ ৬ — Zod schema আর default values

🎯 **লক্ষ্য:** ১৮টা field-এর নিয়ম এক জায়গায়।

🛠️ **কাজ (মূল অংশ):**

```ts
const formSchema = z
  .object({
    title: z.string().trim().min(3, { error: "..." }).max(120, { error: "..." }),
    division: z.string().min(1, { error: "Please select a division." }),
    tourType: z.string().min(1, { error: "Please select a tour type." }),
    description: z.string().trim().optional(),
    images: z.array(z.instanceof(File).refine(/* ধরন */).refine(/* মাপ */)).max(10, { error: "..." }),
    location: z.string().trim().optional(),
    // departureLocation, arrivalLocation একই রকম
    startDate: z.string().optional(),
    endDate: z.string().optional(),
    costFrom: z.number({ error: "Enter a valid amount." }).min(0, { error: "..." }).optional(),
    maxGuest: z.number({ error: "..." }).int().min(1, { error: "At least 1 guest." }).optional(),
    minAge: z.number({ error: "..." }).int().min(0, { error: "..." }).optional(),
    included: z.array(z.string()),
    excluded: z.array(z.string()),
    amenities: z.array(z.string()),
    tourPlan: z.array(z.string()),
  })
  .refine((data) => !data.startDate || data.startDate >= getToday(), {
    error: "Start date can't be in the past.", path: ["startDate"],
  })
  .refine((data) => !data.endDate || data.endDate >= getToday(), {
    error: "End date can't be in the past.", path: ["endDate"],
  })
  .refine((data) => !data.startDate || !data.endDate || data.endDate >= data.startDate, {
    error: "End date can't be before start date.", path: ["endDate"],
  });
```

💡 **নতুন কিছু জিনিস:**
- **`z.array(z.instanceof(File)...)`:** একাধিক ছবি = File-এর array। ভেতরের নিয়ম (ধরন, মাপ) প্রতিটা ছবির ওপর আলাদা চলে, `.max(10)` পুরো array-র ওপর।
- **`z.number()`:** খরচ, অতিথি, বয়স সংখ্যা; input থেকে সংখ্যায় রূপান্তর ধাপ ৯-এ।
- **`.int()`:** অতিথি ২.৫ জন হয় না।
- **শেষের তিনটা `refine`:** একাধিক field মিলিয়ে নিয়ম (১২ নম্বর নোটের password মেলানোর মতো)। `path` দিয়ে কোন field-এর নিচে error দেখাবে।
- **`!data.startDate || …`:** তারিখ ঐচ্ছিক; খালি থাকলে নিয়মই চালাতে হবে না, সত্য ধরে নাও।

**Default values:** প্রতিটা field-এর শুরুর মান (১২ নম্বর নোটের নিয়ম: সব field-এ default)। লেখা `""`, তালিকা `[]`, সংখ্যা `undefined` (খালি box; `0` দিলে box-এ `0` লেখা থাকত)।

---

## ধাপ ৭ — Division আর Tour type API থেকে: `useMemo`

🎯 **লক্ষ্য:** নমুনা options বাদ দিয়ে database-এর আসল division আর tour type।

🛠️ **কাজ:**

```tsx
const { data: divisions = [], isLoading: divisionLoading } = useGetDivisionsQuery(undefined);
const { data: tourTypes = [], isLoading: tourTypeLoading } = useGetTourTypesQuery(undefined);

// API-র data → SearchableSelect-এর { value, label }
const divisionOptions = useMemo(
  () => divisions.map((item: IDivision) => ({ value: item._id, label: item.name })),
  [divisions],
);
```

```tsx
<Controller
  name="division"
  control={form.control}
  render={({ field, fieldState }) => (
    <Field data-invalid={fieldState.invalid} className="gap-1.5">
      <FieldLabel htmlFor="tour-division">Division *</FieldLabel>
      <SearchableSelect
        id="tour-division"
        options={divisionOptions}
        value={field.value}
        onChange={field.onChange}
        invalid={fieldState.invalid}
        disabled={divisionLoading}
        placeholder={divisionLoading ? "Loading divisions…" : "Select a division"}
      />
      {fieldState.invalid && <FieldError errors={[fieldState.error]} />}
    </Field>
  )}
/>
```

- **`isLoading: divisionLoading`:** দুটো query-র `isLoading` একই নামে নেওয়া যায় না, তাই নাম বদলে নেওয়া।
- লোড হওয়ার সময় dropdown বন্ধ আর লেখা "Loading divisions…"।
- **RTK Query-র cache:** Division page-এ একবার division আনা হয়ে থাকলে এখানে আবার request যায় না, cache থেকে সাথে সাথে আসে (১৩ নম্বর নোট)।

### 💡 `useMemo` কী (আপনার প্রশ্ন)

**সমস্যা:** component প্রতিবার render হলে তার ভেতরের সব কোড আবার চলে। এই form-এ title-এর প্রতিটা অক্ষর টাইপ করলে render হয় (slug দেখানোর জন্য `watch`, ধাপ ১২), অর্থাৎ প্রতি অক্ষরে `divisions.map(...)` আবার চলে আর একটা **নতুন** options array তৈরি হয়, যদিও division একটাও বদলায়নি।

**`useMemo` বলে:** "এই হিসাবটা করে ফলাফল মনে রাখো। পরের render-এ dependency (`[divisions]`) না বদলালে আবার হিসাব কোরো না, আগের ফলাফলটাই দাও।"

```ts
const divisionOptions = useMemo(
  () => divisions.map(...),   // ১. হিসাব
  [divisions],                // ২. শুধু এটা বদলালে আবার হিসাব
);
```

| | `useMemo` ছাড়া | `useMemo` সহ |
|---|---|---|
| Title-এ এক অক্ষর টাইপ | `map` আবার চলে, নতুন array | আগের array-টাই ফেরত |
| নতুন division যোগ হয়ে data বদলাল | নতুন array | নতুন array (dependency বদলেছে) |

রান্নার উদাহরণ: প্রতিবার কেউ রান্নাঘরে ঢুকলে নতুন করে মসলা বাটা না, একবার বেটে বয়ামে রাখা; শুধু মসলা ফুরালে (dependency বদলালে) আবার বাটা।

**সত্যি বলতে এখানে লাভ ছোট:** ৮টা division-এর `map` খুবই দ্রুত, `useMemo` না দিলেও কেউ টের পাবে না। `useMemo` সত্যিই কাজে লাগে যখন:
1. হিসাবটা **ভারী** (হাজার item ছাঁকা বা সাজানো)।
2. ফলাফলটা অন্য hook-এর dependency বা `memo` করা component-এ যায়, আর প্রতিবার নতুন array গেলে সেগুলো অকারণে আবার চলে।

এখানে রেখে দেওয়ায় ক্ষতি নেই, আর ধারণাটা শেখার জন্য ভালো উদাহরণ। নিয়ম হিসেবে মনে রাখুন: **আগে সাধারণভাবে লিখুন; ধীর হলে বা উপরের দুটোর একটা হলে তবেই `useMemo`।**

> নোট: আপনার কোডে লেখা `(divisions as IDivision[]).map(...)`। `as` লাগছে কারণ `getDivisions` endpoint-এ type নেই (১৯ নম্বর নোটের উন্নতি ৫)। Endpoint-এ type দিলে `as` লাগে না; আর না দিলে উপরের মতো `(item: IDivision)` লিখে দেওয়াও চলে।

✅ **যাচাই:** Division dropdown-এ database-এর আসল নাম; বেছে submit করলে (পরে) id যায়।

---

## ধাপ ৮ — তারিখ: অতীত বন্ধ, end ≥ start

🎯 **লক্ষ্য:** User যেন অতীতের তারিখ বা শুরুর আগের শেষ তারিখ বাছতে না পারে।

🛠️ **কাজ:**

```ts
// আজকের তারিখ "YYYY-MM-DD" আকারে, user-এর নিজের সময় অনুযায়ী
const getToday = () => {
  const d = new Date();
  const month = String(d.getMonth() + 1).padStart(2, "0");
  const day = String(d.getDate()).padStart(2, "0");
  return `${d.getFullYear()}-${month}-${day}`;
};
```

```tsx
const today = getToday();
const startDate = form.watch("startDate");

// Start date
<Input
  {...field}
  type="date"
  min={today}                                       // ক্যালেন্ডারে অতীত ধূসর
  onChange={(e) => {
    field.onChange(e);
    const end = form.getValues("endDate");
    if (end && end < e.target.value) form.setValue("endDate", "");  // শেষ তারিখ এখন আগে পড়ে গেলে মুছে দাও
  }}
/>

// End date
<Input {...field} type="date" min={startDate || today} />  // শুরুর আগের তারিখ ধূসর
```

### 💡 তিন স্তরে পাহারা

| স্তর | কী করে |
|---|---|
| ১. `min` attribute | Browser-এর ক্যালেন্ডারেই অতীতের দিন বাছা যায় না |
| ২. `onChange`-এ end মোছা | Start পরে সরালে, end আগে পড়ে গেলে নিজে মুছে যায় |
| ৩. Zod-এর `refine` | হাতে টাইপ করে বা অন্য উপায়ে ভুল তারিখ দিলেও submit-এ ধরা পড়ে |

### 💡 লেখা হিসেবে তারিখ তুলনা (`"2026-11-20" >= "2026-11-12"`)

`<input type="date">`-এর মান সবসময় `"YYYY-MM-DD"` লেখা। এই আকারে বড় থেকে ছোট (বছর → মাস → দিন), আর মাস-দিন সবসময় দুই অঙ্কের, তাই **লেখা হিসেবে তুলনা করলেই সঠিক ক্রম পাওয়া যায়**, `Date` বানাতে হয় না। এজন্যই `getToday`-তে `padStart(2, "0")`: `"2026-11-5"` আর `"2026-11-12"` তুলনা ভুল হতো (`"5" > "1"`), `"05"` দিলে ঠিক।

**কেন `new Date().toISOString().slice(0, 10)` না:** `toISOString` **UTC** সময় দেয়। বাংলাদেশ UTC+6, তাই সকাল ৬টার আগে এটা **গতকালের** তারিখ দিত, আর user সকালে আজকের tour বাছতে গিয়ে বিভ্রান্ত হতো। আপনার `getToday` user-এর নিজের সময় (`getFullYear`, `getMonth`, `getDate`) ব্যবহার করে, এটাই সঠিক।

> 💬 **Teacher-এর Day Picker বনাম আপনার `<input type="date">`**
>
> Teacher shadcn-এর `Calendar` (ভেতরে `react-day-picker`) + `Popover` দিয়ে তারিখ বাছাই বানিয়েছেন, আর পরের class-এ অতীতের তারিখ বন্ধ করার সমস্যা ঠিক করেছেন। আপনি browser-এর নিজের `type="date"` ব্যবহার করেছেন, আর `min` দিয়ে অতীত আগেই বন্ধ।
>
> | | shadcn Calendar (teacher) | `<input type="date">` (আপনি) |
> |---|---|---|
> | দেখতে | সব browser-এ একই, project-এর ডিজাইনে মেলানো | প্রতিটা browser/মোবাইলের নিজের ধরন |
> | মোবাইলে | নিজের popup | Phone-এর নিজের তারিখ বাছার চাকা/ক্যালেন্ডার (মোবাইলে খুব সুবিধার) |
> | বাড়তি কোড/library | `react-day-picker`, `date-fns` | কিছুই না |
> | মান | `Date` object (পাঠানোর আগে রূপান্তর লাগে) | সরাসরি `"YYYY-MM-DD"` |
> | তারিখের পরিসর (range), বিশেষ দিন বন্ধ | সহজ | সীমিত (শুধু `min`/`max`) |
>
> Admin panel-এ আপনার পদ্ধতি সহজ আর যথেষ্ট। Public site-এ booking-এর তারিখ বাছাইয়ের মতো জায়গায়, যেখানে দেখতে সুন্দর আর range লাগে, তখন teacher-এর Calendar বেশি মানানসই।

> নোট: `getToday` আপনার কোডে schema-র **নিচে** লেখা, অথচ schema-র `refine`-এর ভেতরে ব্যবহার হয়। এটা কাজ করে, কারণ `refine`-এর function তখনই চলে যখন form যাচাই হয় (ততক্ষণে `getToday` তৈরি হয়ে গেছে)। তবে পড়ার সুবিধার জন্য (আর ভুল করে schema-র বাইরে সরাসরি ডাকলে error এড়াতে) চূড়ান্ত কোডে `getToday` schema-র উপরে রাখা হয়েছে।

✅ **যাচাই:** Start-এর ক্যালেন্ডারে আজকের আগের দিন ধূসর; start ২০ তারিখ দিলে end-এ ২০-এর আগে ধূসর; end ২২ রেখে start ২৫ করলে end নিজে খালি হয়।

---

## ধাপ ৯ — সংখ্যার input

🎯 **লক্ষ্য:** খরচ, অতিথি, বয়স `string` না, আসল **সংখ্যা** হিসেবে form-এ যাওয়া।

### 💡 সমস্যা

`<input type="number">` দেখতে সংখ্যার, কিন্তু তার `e.target.value` সবসময় **লেখা**: `"5500"`। `{...field}` দিলে form-এ `"5500"` (string) যেত, আর `z.number()` বলত "Enter a valid amount"।

### 🛠️ কাজ

```tsx
{(
  [
    { name: "costFrom", id: "tour-cost", label: "Cost from (৳)", placeholder: "5500", min: 0 },
    { name: "maxGuest", id: "tour-max-guest", label: "Max guests", placeholder: "20", min: 1 },
    { name: "minAge", id: "tour-min-age", label: "Min age", placeholder: "12", min: 0 },
  ] as const
).map((item) => (
  <Controller
    key={item.name}
    name={item.name}
    control={form.control}
    render={({ field, fieldState }) => (
      <Input
        type="number"
        name={field.name}
        ref={field.ref}
        onBlur={field.onBlur}
        value={field.value ?? ""}
        onChange={(e) => field.onChange(e.target.value === "" ? undefined : e.target.valueAsNumber)}
        ...
      />
    )}
  />
))}
```

💡 **লাইনগুলোর মানে:**
- **`e.target.valueAsNumber`:** browser নিজেই লেখাকে সংখ্যায় রূপান্তর করে দেয় (`"5500"` → `5500`)।
- **খালি হলে `undefined`:** box খালি করলে `valueAsNumber` হয় `NaN` (সংখ্যা না)। তাই খালি (`""`) হলে `undefined`, যাতে `.optional()` মেনে নেয়।
- **`value={field.value ?? ""}`:** মান `undefined` হলে box-এ `""`; নাহলে React "uncontrolled থেকে controlled" warning দিত (১২ নম্বর নোট)।
- **`{...field}` না দিয়ে আলাদা আলাদা:** `field.onChange`-কে নিজের রূপান্তর-সহ function দিয়ে বদলাতে হচ্ছে, তাই বাকিগুলো (`name`, `ref`, `onBlur`) হাতে দেওয়া।
- **তিনটা field একটা array থেকে `map`:** তিনটার কাঠামো হুবহু এক, শুধু নাম আর লেখা আলাদা। ৩ বার একই ৩০ লাইন না লিখে data থেকে বানানো।
- **`as const`:** TypeScript জানে `item.name` ঠিক `"costFrom" | "maxGuest" | "minAge"`, যেকোনো string না; তাই `Controller`-এর `name`-এ দিলে type মেলে (১৪ নম্বর নোটের `as const`)।

✅ **যাচাই:** Cost-এ `5500` লিখে submit (console log, ধাপ ১২) → `costFrom: 5500` (উদ্ধৃতি ছাড়া, সংখ্যা); খালি রাখলে field-ই থাকে না।

---

## ধাপ ১০ — তালিকার field: `ListField`

🎯 **লক্ষ্য:** Included, Excluded, Amenities, Tour plan-এ যত খুশি item যোগ আর মোছা।

🛠️ **কাজ:**

```tsx
function ListField({ id, label, placeholder, value, onChange, numbered = false }: { ... }) {
  const [draft, setDraft] = useState("");   // box-এ এখন যা লেখা হচ্ছে

  const addItem = () => {
    const text = draft.trim();
    if (!text || value.includes(text)) return;   // খালি বা আগে থেকে আছে → যোগ নয়
    onChange([...value, text]);                  // পুরনো তালিকা + নতুন
    setDraft("");
  };

  const removeItem = (index: number) => onChange(value.filter((_, i) => i !== index));

  // Enter চাপলে item যোগ, পুরো form submit না
  const handleKeyDown = (e: KeyboardEvent<HTMLInputElement>) => {
    if (e.key === "Enter") {
      e.preventDefault();
      addItem();
    }
  };
  // ...input + "+" button + তালিকা (প্রতিটার পাশে ✕)
}
```

```tsx
<Controller
  name="included"
  control={form.control}
  render={({ field }) => (
    <ListField id="tour-included" label="Included" placeholder="e.g. AC bus ticket"
      value={field.value} onChange={field.onChange} />
  )}
/>
```

💡 **দুই রকম state:**
- **`draft` (component-এর নিজের, `useState`):** box-এ এখন যা টাইপ হচ্ছে। Form-এর এটা জানার দরকার নেই।
- **`value` (form-এর, `Controller` থেকে):** যোগ হয়ে যাওয়া item-গুলোর array। এটাই submit-এ যায়।

💡 **`[...value, text]` আর `filter`:** React-এ array সরাসরি বদলানো (`value.push(text)`) যায় না; বদলালে React টের পায় না। সবসময় **নতুন array** বানিয়ে দিতে হয়: যোগ করতে spread, মুছতে `filter`।

💡 **Enter-এ `preventDefault`:** form-এর ভেতরের input-এ Enter চাপলে browser পুরো form submit করে। এখানে Enter মানে "এই item যোগ করো", তাই browser-এর আচরণ বন্ধ।

💡 **`numbered`:** Tour plan-এ ক্রম জরুরি (Day 1, Day 2…), তাই পাশে ১, ২, ৩। Included-এ ক্রম জরুরি না, তাই নম্বর নেই।

> 💬 **Teacher-এর `useFieldArray` বনাম আপনার `ListField`**
>
> Teacher React Hook Form-এর **`useFieldArray`** দেখিয়েছেন (https://react-hook-form.com/docs/usefieldarray): চলতে চলতে নতুন input যোগ/মোছা (dynamic repeater field), যেমন amenities-এর জন্য "+ Add" চাপলে একটা নতুন খালি input আসে, প্রতিটায় সরাসরি লেখা যায়।
>
> ```tsx
> // useFieldArray-এর ধারণা (teacher)
> const { fields, append, remove } = useFieldArray({ control: form.control, name: "amenities" });
>
> {fields.map((field, index) => (
>   <div key={field.id}>
>     <Input {...form.register(`amenities.${index}.value`)} />
>     <Button type="button" onClick={() => remove(index)}>✕</Button>
>   </div>
> ))}
> <Button type="button" onClick={() => append({ value: "" })}>+ Add</Button>
> ```
>
> | | `useFieldArray` (teacher) | `ListField` (আপনি) |
> |---|---|---|
> | দেখতে | প্রতিটা item একটা আলাদা input, সব খোলা | একটা input, যোগ হলে "chip"-এর মতো তালিকা |
> | যোগ করা item বদলানো | সরাসরি input-এ লিখে | মুছে আবার যোগ করতে হয় |
> | Data-র আকার | **object-এর array**: `[{ value: "Wi-Fi" }]` (useFieldArray-এর নিয়ম; প্রতিটা item-এর একটা `id` লাগে) | সরাসরি **string-এর array**: `["Wi-Fi"]` |
> | পাঠানোর আগে | `map((a) => a.value)` দিয়ে string-এর array বানাতে হয় | যেমন আছে তেমন |
> | প্রতিটা item-এ একাধিক field | ✅ খুব ভালো (যেমন tour plan-এ day + title + description) | ❌ শুধু এক লাইনের লেখা |
> | প্রতিটা item-এর আলাদা validation | ✅ | সীমিত |
>
> **কোনটা কখন:** item যদি শুধু এক লাইনের লেখা হয় (included, amenities), আপনার `ListField` সহজ আর data-ও সরাসরি backend-এর আকারে। প্রতিটা item-এ **একাধিক অংশ** থাকলে (যেমন পরে tour plan-কে `{ day: 1, title, description }` বানালে) `useFieldArray` অনেক ভালো। দুটোই জেনে রাখুন।

✅ **যাচাই:** Included-এ "AC bus" লিখে Enter → তালিকায় আসে, form submit হয় না; একই লেখা আবার দিলে যোগ হয় না; ✕ দিলে মুছে যায়।

---

## ধাপ ১১ — একাধিক ছবি: `MultiImageUploader`

🎯 **লক্ষ্য:** ১০টা পর্যন্ত ছবি বাছা, preview, মোছা; প্রথমটা cover।

🛠️ **কাজ:**

```tsx
// src/components/ui/multi-image-uploader.tsx
import { cn } from "@/lib/utils";
import { ImagePlus, X } from "lucide-react";
import { useEffect, useMemo, useRef, useState } from "react";

type MultiImageUploaderProps = {
  id?: string;
  value?: File[];
  onChange: (files: File[]) => void;
  onBlur?: () => void;
  maxFiles?: number;
  accept?: string;
  hint?: string;
  invalid?: boolean;
};

export default function MultiImageUploader({
  id = "multi-image-uploader",
  value = [],
  onChange,
  onBlur,
  maxFiles = 10,
  accept = "image/png,image/jpeg,image/webp",
  hint = "PNG, JPG or WEBP · max 2 MB each",
  invalid = false,
}: MultiImageUploaderProps) {
  const inputRef = useRef<HTMLInputElement>(null);
  const [isDragging, setIsDragging] = useState(false);

  // প্রতিটা File → preview URL
  const previews = useMemo(() => value.map((file) => URL.createObjectURL(file)), [value]);

  // পুরনো preview URL ছেড়ে দেওয়া (memory)
  useEffect(() => {
    return () => previews.forEach((url) => URL.revokeObjectURL(url));
  }, [previews]);

  const addFiles = (fileList?: FileList | null) => {
    if (!fileList?.length) return;
    const next = [...value, ...Array.from(fileList)].slice(0, maxFiles);
    onChange(next);
    if (inputRef.current) inputRef.current.value = ""; // একই ফাইল আবার বাছা যায়
  };

  const removeFile = (index: number) => {
    onChange(value.filter((_, i) => i !== index));
  };

  const isFull = value.length >= maxFiles;

  return (
    <div className="flex flex-col gap-3">
      <input
        ref={inputRef}
        id={id}
        type="file"
        accept={accept}
        multiple
        className="sr-only"
        onBlur={onBlur}
        disabled={isFull}
        onChange={(e) => addFiles(e.target.files)}
      />

      <label
        htmlFor={id}
        onDragOver={(e) => {
          e.preventDefault();
          if (!isFull) setIsDragging(true);
        }}
        onDragLeave={() => setIsDragging(false)}
        onDrop={(e) => {
          e.preventDefault();
          setIsDragging(false);
          if (!isFull) addFiles(e.dataTransfer.files);
        }}
        className={cn(
          "flex h-40 flex-col items-center justify-center gap-2 rounded-xl border-2 border-dashed text-center transition-colors",
          isFull
            ? "cursor-not-allowed border-border bg-muted opacity-60"
            : isDragging
              ? "cursor-pointer border-primary bg-accent"
              : "cursor-pointer border-border bg-muted hover:border-primary hover:bg-accent",
          invalid && "border-destructive",
        )}
      >
        <span className="flex size-11 items-center justify-center rounded-full bg-accent text-accent-foreground">
          <ImagePlus className="size-5" />
        </span>
        <span className="text-sm font-semibold text-foreground">
          {isFull ? `Maximum ${maxFiles} images` : isDragging ? "Drop images here" : "Click or drag images here"}
        </span>
        <span className="text-[12.5px] text-muted-foreground">
          {hint} · {value.length}/{maxFiles}
        </span>
      </label>

      {value.length > 0 && (
        <div className="grid grid-cols-2 gap-3 sm:grid-cols-4">
          {value.map((file, i) => (
            <div
              key={`${file.name}-${file.lastModified}-${i}`}
              className="relative aspect-4/3 overflow-hidden rounded-lg border border-border bg-accent"
            >
              <img src={previews[i]} alt={file.name} className="size-full object-cover" />
              {i === 0 && (
                <span className="absolute top-2 left-2 rounded-md bg-primary px-2 py-0.5 text-[11px] font-semibold text-primary-foreground">
                  Cover
                </span>
              )}
              <button
                type="button"
                onClick={() => removeFile(i)}
                aria-label={`Remove ${file.name}`}
                className="absolute top-2 right-2 flex size-7 items-center justify-center rounded-full bg-black/60 text-white hover:bg-black/80"
              >
                <X className="size-3.5" />
              </button>
              <span className="absolute inset-x-0 bottom-0 truncate bg-linear-to-t from-black/60 to-transparent px-2 pt-4 pb-1.5 text-[11.5px] text-white">
                {file.name}
              </span>
            </div>
          ))}
        </div>
      )}
    </div>
  );
}
```

(আপনার কোডে class জোড়া লাগানো `[...].join(" ")` দিয়ে; এখানে project-এর `cn()`, ১৯ নম্বর নোটের উন্নতি ৭। বাকি সব আপনার কোড।)

### 💡 Single uploader (১৯ নম্বর নোট) থেকে কী আলাদা

| | `ImageUploader` | `MultiImageUploader` |
|---|---|---|
| `value` | একটা `File` | `File[]` |
| `<input>` | সাধারণ | **`multiple`**: একসাথে কয়েকটা বাছা যায় |
| যোগ করা | আগেরটা বদলে যায় | **আগেরগুলোর সাথে যোগ হয়**: `[...value, ...নতুন]` |
| সীমা | — | `maxFiles`; ভরে গেলে বাক্স বন্ধ আর "Maximum 10 images" |
| Preview | একটা blob URL | প্রতিটা ফাইলের আলাদা blob URL |

💡 **`Array.from(fileList)`:** `e.target.files` আসলে `FileList`, দেখতে array-র মতো কিন্তু array না (`map`, spread পুরোপুরি চলে না)। `Array.from` দিয়ে আসল array।

💡 **`.slice(0, maxFiles)`:** আগে ৮টা থাকলে আর ৫টা বাছলে মোট ১৩, `slice` প্রথম ১০টা রাখে।

💡 **Cover:** `i === 0` হলে "Cover" badge। Backend সাধারণত `images[0]`-কে প্রধান ছবি ধরে, Tours table-এও `tour.images[0]` দেখানো হয়।

💡 **Preview-এর memory:** `useMemo` প্রতিবার ফাইলের তালিকা বদলালে **সব** ফাইলের নতুন blob URL বানায়, আর `useEffect`-এর cleanup **আগের সব** URL ছেড়ে দেয় (১৯ নম্বর নোটের `revokeObjectURL`)। তাই যত বার যোগ-মোছা হোক, memory-তে শুধু এখনকার ছবিগুলোর URL থাকে।

💡 **Form-এ:**

```tsx
<MultiImageUploader id="tour-images" value={field.value} onChange={field.onChange} onBlur={field.onBlur}
  invalid={fieldState.invalid} />
{fieldState.error && (
  <FieldError>
    {fieldState.error.message ?? fieldState.error.root?.message ?? "Some images are invalid."}
  </FieldError>
)}
```

Array-র error দুই জায়গায় আসতে পারে: পুরো array-র নিয়ম ভাঙলে (১০টার বেশি) `error.message`-এ; কিন্তু **ভেতরের** কোনো একটা ছবি ভুল হলে (৩ নম্বরটা 2 MB-এর বেশি) error থাকে `error[2].message`-এ, উপরের `message` খালি। তাই কিছু না পেলে সাধারণ "Some images are invalid."।

> নোট: শেষের লেখাটা কোন ছবি ভুল সেটা বলে না। চাইলে সব ছবির error খুঁজে দেখানো যায়: `Object.values(fieldState.error).find((e) => e?.message)?.message`। অথবা uploader-এ ছবি যোগের সময়ই বড়/ভুল ধরনের ফাইল বাদ দিয়ে toast দেখানো যায় (উন্নতিতে বিস্তারিত না, ইচ্ছা হলে পরে)।

✅ **যাচাই:** একসাথে ৩টা ছবি বাছুন → ৩টা preview, প্রথমটায় "Cover"; আবার ২টা বাছুন → মোট ৫; মাঝেরটা ✕ → ৪; ১০টা হলে বাক্স ফিকে আর বন্ধ।

---

# ভাগ ঘ: পাঠানো

## ধাপ ১২ — প্রথমে console-এ দেখা

🎯 **লক্ষ্য:** API-তে পাঠানোর আগে নিশ্চিত হওয়া, data ঠিক আকারে আছে।

🛠️ **কাজ (আপনার পরীক্ষার version):**

```ts
const onSubmit = (data: FormValues) => {
  const { images, ...rest } = data;

  // খালি লেখা / খালি তালিকা বাদ, যাতে backend শুধু ভরা field পায়
  const tourData = Object.fromEntries(
    Object.entries(rest).filter(([, v]) => v !== "" && v !== undefined && !(Array.isArray(v) && v.length === 0)),
  );

  const formData = new FormData();
  formData.append("data", JSON.stringify(tourData));
  images.forEach((file) => formData.append("files", file));   // multerUpload.array("files")

  console.log("FormData entries:", [...formData.entries()]);
};
```

### 💡 লাইন ধরে ধরে

**১. `const { images, ...rest } = data;`** (destructuring + rest)
`data` থেকে `images` আলাদা করে বের করা, আর **বাকি সব** (`...rest`) একটা নতুন object-এ। কারণ ছবি যাবে `files`-এ, বাকি সব `data`-তে।

**২. খালি field বাদ দেওয়া: `Object.entries` → `filter` → `Object.fromEntries`**

```ts
{ title: "Sajek", location: "", included: [], costFrom: 5500, minAge: undefined }
   ↓ Object.entries  (object → [key, value]-এর array)
[["title","Sajek"], ["location",""], ["included",[]], ["costFrom",5500], ["minAge",undefined]]
   ↓ filter  ("", undefined, খালি array বাদ)
[["title","Sajek"], ["costFrom",5500]]
   ↓ Object.fromEntries  (আবার object)
{ title: "Sajek", costFrom: 5500 }
```

**কেন বাদ:** Backend-এ `location: ""` গেলে database-এ খালি লেখা সংরক্ষণ হয়, অথবা backend-এর validation (যেমন `min(1)`) error দেয়। না পাঠালে backend field-টা "দেওয়া হয়নি" ধরে (optional)। ১৩ নম্বর নোটের register-এ phone/address-এর একই সমস্যা, সেখানে spread দিয়ে, এখানে ১৮টা field বলে সাধারণ উপায়ে।

`filter(([, v]) => …)`: `[key, value]`-এর প্রথমটা (key) লাগছে না, তাই কমার আগে ফাঁকা।

**৩. একাধিক ছবি একই key-তে বারবার**

```ts
images.forEach((file) => formData.append("files", file));
```

FormData-তে একই key-তে বারবার `append` করলে আগেরটা মুছে যায় না, **সবগুলো** থাকে। Backend-এর `multer.array("files")` সব `files` একসাথে array হিসেবে পায়। (Division-এ `single("file")`, এখানে `array("files")`; নাম আর ধরন দুটোই মিলতে হবে।)

**৪. Slug পাঠানো হচ্ছে না:** প্রথম version-এ `tourData`-তে `slug: makeSlug(title)` ছিল, শেষ version-এ বাদ। Backend নিজে title থেকে slug বানালে frontend-এর পাঠানোর দরকার নেই, title-এর নিচের "Slug: /tours/…" শুধু preview (১৯ নম্বর নোটের slug-এর সতর্কতা)। Backend না বানালে আবার যোগ করবেন।

✅ **যাচাই:** Console-এ `[["data", "{...}"], ["files", File], ["files", File]]`; `data`-র JSON-এ খালি field নেই, সংখ্যা উদ্ধৃতি ছাড়া।

---

## ধাপ ১৩ — `addTour` API আর আসল submit

🎯 **লক্ষ্য:** Tour তৈরি, তারপর Tours page-এ ফেরা।

🛠️ **কাজ ১ — endpoint:**

```ts
// src/redux/features/Tour/tour.api.ts
addTour: builder.mutation({
  query: (tourData: FormData) => ({
    url: "/tour/create",
    method: "POST",
    data: tourData,
  }),
  invalidatesTags: ["TOUR"],
}),
```

(Content-Type হাতে দেবেন না, ১৯ নম্বর নোটের ধাপ ৯।)

🛠️ **কাজ ২ — submit (আপনার কোড):**

```tsx
const [addTour, { isLoading }] = useAddTourMutation();
const navigate = useNavigate();

const onSubmit = async (data: FormValues) => {
  // ...tourData, formData (ধাপ ১২)
  const toastId = toast.loading("Creating tour…");
  try {
    await addTour(formData).unwrap();
    toast.success("Tour created successfully", { id: toastId });
    form.reset();
    navigate("/admin/tours");
  } catch (err) {
    const error = err as TApiError;
    const message = error.data?.errorSources?.[0]?.message || error.data?.message || "Failed to create tour.";
    toast.error(message, { id: toastId });
  }
};
```

আর পুরো form `<fieldset disabled={isLoading}>`-এ (১৯ নম্বর নোট), Save button-এ spinner আর "Saving…"।

### 💡 Cancel link আলাদা করে বন্ধ কেন

```tsx
<Button asChild variant="outline" aria-disabled={isLoading}>
  <Link to="/admin/tours" className={isLoading ? "pointer-events-none opacity-50" : ""}>Cancel</Link>
</Button>
```

`<fieldset disabled>` শুধু input, button, select, textarea বন্ধ করে, **link (`<a>`) না**। Link-এ HTML-এর `disabled` বলে কিছুই নেই। তাই:
- **`pointer-events-none`:** mouse click কাজ করে না।
- **`aria-disabled`:** screen reader-কে জানায় বন্ধ।
- **`opacity-50`:** দেখতে ফিকে।

না দিলে upload চলার মাঝে Cancel চাপলে page ছেড়ে চলে যেত, অথচ tour তৈরি হচ্ছিলই, user ভাবত বাতিল হয়েছে।

✅ **যাচাই:** সব ভরে Save → "Creating tour…" → "Tour created successfully" → Tours page; Network → `create` → Payload-এ `data` আর একাধিক `files`।

---

# ভাগ ঙ: তালিকা, Pagination, মোছা

## ধাপ ১৪ — Backend: `populate` আর `meta`

🎯 **লক্ষ্য:** Tour-এর তালিকায় division আর tour type-এর **নাম** পাওয়া, আর pagination-এর তথ্য।

> এই ধাপ **backend project**-এর কোড।

```ts
// backend: tour.service.ts
const getAllTours = async (query: Record<string, string>) => {
  const queryBuilder = new QueryBuilder(
    Tour.find().populate("division", "name slug").populate("tourType", "name"),
    query,
  );

  const tours = queryBuilder.search(tourSearchableFields).filter().sort().fields().paginate();

  const [data, meta] = await Promise.all([tours.build(), queryBuilder.getMeta()]);

  return { data, meta };
};
```

💡 **লাইনগুলোর মানে:**
- **`.populate("division", "name slug")`:** database-এ tour রাখে শুধু `division: "6650f1…"` (id)। `populate` সেই id দিয়ে Division collection থেকে খুঁজে বসিয়ে দেয়: `division: { _id, name: "Sylhet", slug: "sylhet" }`। দ্বিতীয় parameter-এ শুধু যে field লাগবে (`"name slug"`), পুরো division না, response ছোট থাকে।
- **`QueryBuilder`** (backend course-এ বানানো): URL-এর `?page=2&limit=10&searchTerm=sajek` পড়ে খোঁজা, ছাঁকা, সাজানো, পাতা ভাগ করা।
- **`Promise.all([…])`:** এক পাতার data আনা আর মোট সংখ্যা গোনা (meta) **একসাথে** চালানো, একটার পর আরেকটা না। দুটো আলাদা database কাজ, একসাথে চললে মোট সময় কম।
- **`meta`:** `{ page, limit, total, totalPage }`, frontend-এর pagination এখান থেকেই।

> নোট: Division মুছে ফেলা হলে, সেই division-এর tour-এ `populate` কিছু খুঁজে পায় না, তাই `division: null` আসে। এজন্যই `ITour`-এ `division: {...} | null`, আর table-এ `tour.division?.name ?? "—"` (আপনার কোডে আছে ✅)।

---

## ধাপ ১৫ — Type, `getTours` আর Pagination

🎯 **লক্ষ্য:** Tours page-এ আসল data, ১০টা করে পাতায়।

🛠️ **কাজ ১ — type:**

```ts
// src/types/tour.type.ts (ITourType-এর নিচে)
export interface IMeta {
  page: number;
  limit: number;
  total: number;
  totalPage: number;
}

export interface ITour {
  _id: string;
  title: string;
  slug: string;
  description?: string;
  images?: string[];
  location?: string;
  costFrom?: number;
  startDate?: string;
  endDate?: string;
  departureLocation?: string;
  arrivalLocation?: string;
  included?: string[];
  excluded?: string[];
  amenities?: string[];
  tourPlan?: string[];
  maxGuest?: number;
  minAge?: number;
  division: { _id: string; name: string; slug?: string } | null;   // populate হওয়া
  tourType: { _id: string; name: string } | null;
  createdAt?: string;
  updatedAt?: string;
}
```

```ts
// src/types/index.ts
export type { IMeta, ITour, ITourType } from "./tour.type";
```

🛠️ **কাজ ২ — endpoint:**

```ts
type TToursParams = { page?: number; limit?: number; searchTerm?: string };

getTours: builder.query<{ data: ITour[]; meta: IMeta }, TToursParams | void>({
  query: (params) => ({
    url: "/tour",
    method: "GET",
    params: params ?? undefined,
  }),
  providesTags: ["TOUR"],
  transformResponse: (response: IResponse<ITour[]> & { meta: IMeta }) => ({
    data: response.data,
    meta: response.meta,
  }),
}),
```

💡 **লাইনগুলোর মানে:**
- **`params`:** axios-এর `params`, object থেকে URL-এর query string বানায়: `{ page: 2, limit: 10 }` → `/tour?page=2&limit=10`। নিজে string জোড়া লাগাতে হয় না (`axiosBaseQuery`-তে `params` আগে থেকেই রাখা, ১৩ নম্বর নোট)।
- **`TToursParams | void`:** argument দেওয়াও যায় (`useGetToursQuery({ page: 2 })`), না দিলেও চলে (`useGetToursQuery()`)।
- **`transformResponse`-এ `data` আর `meta` দুটোই:** ১৭ নম্বর নোটে সতর্ক করা হয়েছিল, শুধু `response.data` দিলে pagination-এর `meta` হারিয়ে যেত। আপনি দুটোই রেখেছেন ✅।
- **`IResponse<ITour[]> & { meta: IMeta }`:** `&` মানে "দুটো type মিলিয়ে": সাধারণ response-এর সব field **আর** `meta`।

🛠️ **কাজ ৩ — page-এ pagination (আপনার কোড):**

```tsx
const LIMIT = 10;

const [page, setPage] = useState(1);
const { data, isLoading, isError, isFetching, refetch } = useGetToursQuery({ page, limit: LIMIT });

const tours = data?.data ?? [];
const total = data?.meta?.total ?? 0;
const totalPage = data?.meta?.totalPage ?? 1;

const from = total === 0 ? 0 : (page - 1) * LIMIT + 1;   // এই পাতার প্রথম tour-এর নম্বর
const to = Math.min(page * LIMIT, total);                 // শেষ tour-এর নম্বর
```

```tsx
<span>Showing {from}–{to} of {total}</span>

<Button onClick={() => setPage((p) => p - 1)} disabled={page === 1 || isFetching} aria-label="Previous page">
  <ChevronLeft />
</Button>
<span>Page {page} of {totalPage}</span>
<Button onClick={() => setPage((p) => p + 1)} disabled={page >= totalPage || isFetching} aria-label="Next page">
  <ChevronRight />
</Button>
```

💡 **কীভাবে কাজ করে:**
1. `page` একটা state। "Next" চাপলে `page` হয় ২।
2. Hook-এর argument বদলাল (`{ page: 2, limit: 10 }`), RTK Query নিজে থেকে `/tour?page=2&limit=10` আনে।
3. **প্রতিটা পাতা আলাদা cache-এ:** পাতা ১-এ ফিরে গেলে আবার request লাগে না, সাথে সাথে আসে (৬০ সেকেন্ড পর্যন্ত)।
4. **নতুন পাতা আসার সময় পুরনোটা দেখায়:** argument বদলালে RTK Query-র `data` নতুনটা আসা পর্যন্ত **আগের পাতার** data রেখে দেয় (তখন `isFetching` সত্য)। আপনার `isFetching && !isLoading` → `opacity-60` সেই সময় table ফিকে করে, আর button-গুলো বন্ধ থাকে যাতে দ্রুত পরপর চাপা না যায়।
5. **ক্রমিক নম্বর:** `(page - 1) * LIMIT + index + 1`: পাতা ২-এর প্রথমটা ১১, ১ না।

**`from` আর `to`-এর হিসাব:** মোট ২৩টা tour, পাতা ৩ → from = (৩−১)×১০+১ = ২১, to = min(৩০, ২৩) = ২৩ → "Showing 21–23 of 23"।

✅ **যাচাই:** ১০টার বেশি tour থাকলে নিচে "Page 1 of 3"; Next-এ নতুন পাতা, আসার সময় table ফিকে; প্রথম পাতায় Previous বন্ধ, শেষে Next বন্ধ।

---

## ধাপ ১৬ — মোছা (ছবিসহ)

🎯 **লক্ষ্য:** Tour মোছা, Cloudinary-র সব ছবিসহ।

🛠️ **কাজ ১ — backend:**

```ts
// backend: tour.service.ts
const deleteTour = async (id: string) => {
  const tour = await Tour.findByIdAndDelete(id);
  if (!tour) {
    throw new AppError(404, "Tour not found.");
  }

  if (tour.images?.length) {
    await Promise.all(tour.images.map((url) => deleteImageFromCLoudinary(url)));
  }

  return null;
};
```

💡 **`findByIdAndDelete`-এর ফলাফল:** মুছে ফেলা document-টাই ফেরত দেয়, তাই আলাদা `findById` ছাড়াই `tour.images` পাওয়া যায় (১৯ নম্বর নোটে division-এ আগে খুঁজে তারপর মোছা হয়েছিল; দুটোই চলে)। না পেলে `null` → 404।

💡 **`Promise.all(… .map(...))`:** ৫টা ছবি একটার পর একটা না মুছে **একসাথে** মোছা।

🛠️ **কাজ ২ — frontend endpoint আর page:**

```ts
removeTour: builder.mutation({
  query: (tourId: string) => ({ url: `/tour/${tourId}`, method: "DELETE" }),
  invalidatesTags: ["TOUR"],
}),
```

```tsx
const handleRemoveTour = async (id: string) => {
  const toastId = toast.loading("Deleting tour…");
  try {
    await removeTour(id).unwrap();
    toast.success("Tour deleted", { id: toastId });

    // এই পাতার শেষ tour মুছলে এক পাতা পেছনে
    if (tours.length === 1 && page > 1) setPage((p) => p - 1);
  } catch (err) {
    // ...error toast
  }
};
```

💡 **শেষ tour মুছলে পেছনের পাতায়:** পাতা ৩-এ শুধু ১টা tour (২১ নম্বর) আছে। সেটা মুছলে মোট ২০টা, পাতা ৩ আর নেই, খালি table দেখাত ("No tours yet", অথচ tour আছে!)। তাই এই পাতায় মোছার আগে ১টাই থাকলে আর প্রথম পাতা না হলে, এক পাতা পেছনে।

✅ **যাচাই:** একটা tour মুছুন → table থেকে চলে যায়, Cloudinary থেকেও সব ছবি; শেষ পাতার একমাত্র tour মুছলে আগের পাতায় চলে আসে।

---

# কী বাকি: সমস্যা চেনা

| # | কোথায় | সমস্যা |
|---|---|---|
| ১ | `tour.api.ts`-এর tag | Tour আর tour type দুটোই `"TOUR"` tag। **ঠিক ১৭ নম্বর নোটে যেটা আগাম বলা হয়েছিল, এখন ঘটছে:** একটা tour type যোগ বা মুছলে tour-এর তালিকা অকারণে আবার আসে, আর tour যোগ/মুছলে tour type-এর তালিকাও। |
| ২ | AddTour-এর `<form>` | `noValidate` নেই। তারিখে `min` আর সংখ্যায় `min` আছে, তাই কেউ হাতে অতীতের তারিখ টাইপ করলে **browser নিজের ইংরেজি popup** দেখায় আর submit আটকে দেয়, আমাদের Zod-এর সুন্দর message আসে না (১২ নম্বর নোট)। |
| ৩ | সংখ্যার input | Mouse cursor box-এর ওপর রেখে scroll করলে সংখ্যা নিজে বদলে যায় (browser-এর আচরণ)। লম্বা form scroll করতে গিয়ে ভুল করে দাম ৫৫০০ থেকে ৫৪৯৭ হয়ে যেতে পারে। |
| ৪ | Error/toast | Add আর delete দুটোতেই একই ৮ লাইনের toast নকশা; `getErrorMessage`/`runWithToast` (১৭–১৮ নম্বর নোট) ব্যবহার হচ্ছে না। Delete-এর error-এ `errorSources` দেখা হয় না। |
| ৫ | `console.log(res)` | Submit-এর পরে রয়ে গেছে। |
| ৬ | `form.reset()` তারপর `navigate` | Page ছেড়েই চলে যাচ্ছি, form এমনিতেই মুছে যাবে, তাই `reset` অপ্রয়োজনীয় (ক্ষতিও নেই)। |
| ৭ | Delete dialog | `handleRemoveTour` নিজে error ধরে ফেলে, তাই ১৯ নম্বর নোটের মিলিত `DeleteConfirmation`-এও dialog error-এর পরে বন্ধ হয়ে যায়। |
| ৮ | Backend: `Promise.all` দিয়ে ছবি মোছা | Tour **আগেই** মুছে গেছে, তারপর একটা ছবি মুছতে ব্যর্থ হলে `Promise.all` সাথে সাথে পুরো request error দেয়। User "Failed" দেখে, অথচ tour মুছে গেছে (১৯ নম্বর নোটের উন্নতি ৬-এর একই সমস্যা)। |
| ৯ | Tours page-এর Edit button | কিছু করে না, চালু দেখায়। |
| ১০ | ছোটখাটো | Error-এর ঘরে `bg-destructive/10` → `bg-destructive-soft`; শিরোনামে `text-foreground`; table-এর ছবিতে `width`/`height`; `redux/features/Tour` বড় হাতের (১৯ নম্বর নোট); sidebar-এ "Tour type"/"Add Division" নাম (১৭, ১৯ নম্বর নোট)। |

---

# উন্নতি

## উন্নতি ১ — আলাদা tag: `TOUR` আর `TOUR_TYPE`

```ts
// baseApi.ts
tagTypes: ["USER", "TOUR", "TOUR_TYPE", "DIVISION"],
```

`getTours`, `addTour`, `removeTour` → `"TOUR"`; `getTourTypes`, `addTourType`, `removeTourType` → `"TOUR_TYPE"`। এখন প্রতিটা শুধু নিজের তালিকা আপডেট করে।

## উন্নতি ২ — `noValidate`

```tsx
<form id="add-tour" onSubmit={form.handleSubmit(onSubmit)} noValidate>
```

`min` attribute তবু থাকছে (ক্যালেন্ডারে অতীত ধূসর রাখার জন্য); শুধু browser-এর popup বন্ধ, যাচাই করবে Zod।

## উন্নতি ৩ — সংখ্যার box-এ scroll-এ মান বদলানো বন্ধ

```tsx
<Input type="number" onWheel={(e) => e.currentTarget.blur()} ... />
```

Scroll শুরু হলে box থেকে focus সরিয়ে দেয়; focus না থাকলে browser scroll-এ মান বদলায় না, page স্বাভাবিকভাবে scroll হয়।

## উন্নতি ৪ — `runWithToast` (১৮ নম্বর নোট)

Add tour:

```ts
try {
  await runWithToast(addTour(formData).unwrap(), {
    loading: "Creating tour…",
    success: `"${data.title}" created`,
    error: "Couldn't create the tour.",
  });
  navigate("/admin/tours");
} catch {
  // Error toast দেখানো হয়েছে; form-এ থাকি, ঠিক করে আবার চেষ্টা করা যায়
}
```

Delete:

```ts
const handleRemoveTour = async (tour: ITour) => {
  await runWithToast(removeTour(tour._id).unwrap(), {
    loading: `Deleting "${tour.title}"…`,
    success: `"${tour.title}" deleted`,
    error: "Couldn't delete the tour.",
  });
  if (tours.length === 1 && page > 1) setPage((p) => p - 1);
};
```

Delete-এ ব্যর্থ হলে `runWithToast` error আবার ছুড়ে দেয়, তাই `setPage`-এ পৌঁছায় না, আর ১৯ নম্বর নোটের `DeleteConfirmation` dialog খোলা রাখে।

## উন্নতি ৫ — Backend: ছবি মোছা ব্যর্থ হলেও request সফল

```ts
// backend: tour.service.ts
const deleteTour = async (id: string) => {
  const tour = await Tour.findByIdAndDelete(id);
  if (!tour) {
    throw new AppError(404, "Tour not found.");
  }

  // Cleanup: কোনো ছবি মুছতে ব্যর্থ হলেও tour মোছা সফল; শুধু log
  if (tour.images?.length) {
    const results = await Promise.allSettled(tour.images.map((url) => deleteImageFromCLoudinary(url)));
    results.forEach((result, i) => {
      if (result.status === "rejected") {
        console.error("Cloudinary cleanup failed:", tour.images?.[i], result.reason);
      }
    });
  }

  return null;
};
```

**`Promise.all` বনাম `Promise.allSettled`:**

| | `Promise.all` | `Promise.allSettled` |
|---|---|---|
| একটা ব্যর্থ হলে | সাথে সাথে পুরোটা ব্যর্থ (বাকিদের ফল জানা যায় না) | সবাই শেষ হওয়া পর্যন্ত অপেক্ষা, প্রত্যেকের ফল আলাদা (`fulfilled`/`rejected`) |
| কখন | সবগুলো সফল হওয়াই জরুরি (ধাপ ১৪-এর data + meta) | "যতটা পারো করো", ব্যর্থগুলো জানাও (cleanup) |

## উন্নতি ৬ — ছোটখাটো

- `getToday` schema-র উপরে (ধাপ ৮-এর নোট)।
- `console.log(res)` আর `form.reset()` (navigate-এর আগে) বাদ।
- Tours page: Edit `disabled` + "Coming soon"; error-এর ঘরে `bg-destructive-soft`; শিরোনামে `text-foreground`; table-এর ছবিতে `width`/`height`।
- `useMemo`-তে `as` না দিয়ে `(item: IDivision)`।
- Feature folder ছোট হাতের (`tour`, `division`), ১৯ নম্বর নোটের উন্নতি ২।

> নোট (ঐচ্ছিক ভাবনা): এখন ছবি ছাড়াও tour তৈরি হয়। Tour-এর card-এ cover ছবি জরুরি হলে schema-তে `images: z.array(...).min(1, { error: "Add at least one image." }).max(10, ...)` দিন। Backend কী চায় দেখে ঠিক করবেন।

---

# চূড়ান্ত কোড

> ১৭–১৯ নম্বর নোটের `getErrorMessage`, `runWithToast`, মিলিত `DeleteConfirmation` ধরে নেওয়া। Folder ছোট হাতের ধরে import লেখা (`@/redux/features/tour/...`); এখনো না বদলালে `Tour`/`Division` লিখবেন। `SearchableSelect` (ধাপ ৩) আর `MultiImageUploader` (ধাপ ১১) উপরেই সম্পূর্ণ।

### `src/redux/baseApi.ts`

```ts
import { createApi } from "@reduxjs/toolkit/query/react";
import axiosBaseQuery from "./axiosBaseQuery";

export const baseApi = createApi({
  reducerPath: "baseApi",
  baseQuery: axiosBaseQuery(),
  tagTypes: ["USER", "TOUR", "TOUR_TYPE", "DIVISION"],
  endpoints: () => ({}),
});
```

### `src/redux/features/tour/tour.api.ts`

```ts
import { baseApi } from "@/redux/baseApi";
import type { IAddTourType, IMeta, IResponse, ITour, ITourType } from "@/types";

type TToursParams = { page?: number; limit?: number; searchTerm?: string };

export const tourApi = baseApi.injectEndpoints({
  endpoints: (builder) => ({
    /* ───── Tour ───── */
    getTours: builder.query<{ data: ITour[]; meta: IMeta }, TToursParams | void>({
      query: (params) => ({
        url: "/tour",
        method: "GET",
        params: params ?? undefined,
      }),
      providesTags: ["TOUR"],
      transformResponse: (response: IResponse<ITour[]> & { meta: IMeta }) => ({
        data: response.data,
        meta: response.meta,
      }),
    }),

    addTour: builder.mutation<IResponse<ITour>, FormData>({
      query: (tourData) => ({
        url: "/tour/create",
        method: "POST",
        data: tourData,
      }),
      invalidatesTags: ["TOUR"],
    }),

    removeTour: builder.mutation<IResponse<null>, string>({
      query: (tourId) => ({
        url: `/tour/${tourId}`,
        method: "DELETE",
      }),
      invalidatesTags: ["TOUR"],
    }),

    /* ───── Tour type ───── */
    getTourTypes: builder.query<ITourType[], void>({
      query: () => ({
        url: "/tour/tour-types",
        method: "GET",
      }),
      providesTags: ["TOUR_TYPE"],
      transformResponse: (response: IResponse<ITourType[]>) => response.data,
    }),

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
  }),
});

export const {
  useGetToursQuery,
  useAddTourMutation,
  useRemoveTourMutation,
  useGetTourTypesQuery,
  useAddTourTypeMutation,
  useRemoveTourTypeMutation,
} = tourApi;
```

(`IAddTourType` ১৭ নম্বর নোটের `tour.type.ts`-এ; না থাকলে `{ name: string }` লিখুন।)

### `src/pages/Admin/AddTour.tsx`

```tsx
import { Button } from "@/components/ui/button";
import { Field, FieldDescription, FieldError, FieldLabel } from "@/components/ui/field";
import { Input } from "@/components/ui/input";
import MultiImageUploader from "@/components/ui/multi-image-uploader";
import SearchableSelect from "@/components/ui/searchable-select";
import { Textarea } from "@/components/ui/textarea";
import { useGetDivisionsQuery } from "@/redux/features/division/division.api";
import { useAddTourMutation, useGetTourTypesQuery } from "@/redux/features/tour/tour.api";
import type { IDivision, ITourType } from "@/types";
import { runWithToast } from "@/utils/runWithToast";
import { zodResolver } from "@hookform/resolvers/zod";
import { ArrowLeft, Loader2, Plus, X } from "lucide-react";
import { useMemo, useState, type KeyboardEvent, type ReactNode } from "react";
import { Controller, useForm } from "react-hook-form";
import { Link, useNavigate } from "react-router";
import * as z from "zod";

const inputCls = "h-11 rounded-lg px-3.5 text-[15px]";

/* ───────── helpers ───────── */

// আজকের তারিখ "YYYY-MM-DD", user-এর নিজের সময়ে (toISOString UTC দিত)
const getToday = () => {
  const d = new Date();
  const month = String(d.getMonth() + 1).padStart(2, "0");
  const day = String(d.getDate()).padStart(2, "0");
  return `${d.getFullYear()}-${month}-${day}`;
};

const makeSlug = (text: string) =>
  text
    .toLowerCase()
    .trim()
    .replace(/[^a-z0-9\s-]/g, "")
    .replace(/\s+/g, "-")
    .replace(/-+/g, "-");

/* ───────── schema ───────── */

const MAX_SIZE = 2 * 1024 * 1024; // 2 MB
const IMAGE_TYPES = ["image/png", "image/jpeg", "image/webp"];

const formSchema = z
  .object({
    title: z
      .string()
      .trim()
      .min(3, { error: "Title must be at least 3 characters long." })
      .max(120, { error: "Title cannot exceed 120 characters." }),
    division: z.string().min(1, { error: "Please select a division." }),
    tourType: z.string().min(1, { error: "Please select a tour type." }),
    description: z.string().trim().optional(),
    images: z
      .array(
        z
          .instanceof(File)
          .refine((file) => IMAGE_TYPES.includes(file.type), { error: "Only PNG, JPG or WEBP allowed." })
          .refine((file) => file.size <= MAX_SIZE, { error: "Each image must be 2 MB or smaller." }),
      )
      .max(10, { error: "You can upload up to 10 images." }),
    location: z.string().trim().optional(),
    departureLocation: z.string().trim().optional(),
    arrivalLocation: z.string().trim().optional(),
    startDate: z.string().optional(),
    endDate: z.string().optional(),
    costFrom: z.number({ error: "Enter a valid amount." }).min(0, { error: "Cost can't be negative." }).optional(),
    maxGuest: z.number({ error: "Enter a valid number." }).int().min(1, { error: "At least 1 guest." }).optional(),
    minAge: z.number({ error: "Enter a valid number." }).int().min(0, { error: "Age can't be negative." }).optional(),
    included: z.array(z.string()),
    excluded: z.array(z.string()),
    amenities: z.array(z.string()),
    tourPlan: z.array(z.string()),
  })
  .refine((data) => !data.startDate || data.startDate >= getToday(), {
    error: "Start date can't be in the past.",
    path: ["startDate"],
  })
  .refine((data) => !data.endDate || data.endDate >= getToday(), {
    error: "End date can't be in the past.",
    path: ["endDate"],
  })
  .refine((data) => !data.startDate || !data.endDate || data.endDate >= data.startDate, {
    error: "End date can't be before start date.",
    path: ["endDate"],
  });

type FormValues = z.infer<typeof formSchema>;

/* ───────── small helpers (শুধু এই page-এ) ───────── */

function Section({ title, description, children }: { title: string; description?: string; children: ReactNode }) {
  return (
    <section className="rounded-xl border border-border bg-card shadow-card">
      <div className="border-b border-border px-5.5 py-4">
        <h2 className="text-base font-bold text-foreground">{title}</h2>
        {description && <p className="text-[13px] text-muted-foreground">{description}</p>}
      </div>
      <div className="flex flex-col gap-4 p-5.5">{children}</div>
    </section>
  );
}

function ListField({
  id,
  label,
  placeholder,
  value,
  onChange,
  numbered = false,
}: {
  id: string;
  label: string;
  placeholder: string;
  value: string[];
  onChange: (items: string[]) => void;
  numbered?: boolean;
}) {
  const [draft, setDraft] = useState("");

  const addItem = () => {
    const text = draft.trim();
    if (!text || value.includes(text)) return;
    onChange([...value, text]);
    setDraft("");
  };

  const removeItem = (index: number) => onChange(value.filter((_, i) => i !== index));

  // Enter → item যোগ (পুরো form submit না)
  const handleKeyDown = (e: KeyboardEvent<HTMLInputElement>) => {
    if (e.key === "Enter") {
      e.preventDefault();
      addItem();
    }
  };

  return (
    <Field className="gap-1.5">
      <FieldLabel htmlFor={id} className="text-sm font-semibold">
        {label}
      </FieldLabel>
      <div className="flex gap-2">
        <Input
          id={id}
          value={draft}
          onChange={(e) => setDraft(e.target.value)}
          onKeyDown={handleKeyDown}
          placeholder={placeholder}
          autoComplete="off"
          className={inputCls}
        />
        <Button
          type="button"
          variant="outline"
          size="icon"
          onClick={addItem}
          aria-label={`Add to ${label}`}
          className="size-11 rounded-lg"
        >
          <Plus className="size-4.5" />
        </Button>
      </div>
      {value.length > 0 && (
        <ul className="mt-1 flex flex-col gap-1.5">
          {value.map((item, i) => (
            <li
              key={item}
              className="flex items-center gap-2.5 rounded-lg border border-border bg-muted px-3 py-2 text-[14px]"
            >
              {numbered && (
                <span className="flex size-6 shrink-0 items-center justify-center rounded-full bg-accent text-[12px] font-semibold text-accent-foreground">
                  {i + 1}
                </span>
              )}
              <span className="flex-1">{item}</span>
              <button
                type="button"
                onClick={() => removeItem(i)}
                aria-label={`Remove ${item}`}
                className="flex size-7 items-center justify-center rounded-md text-muted-foreground hover:bg-background hover:text-destructive"
              >
                <X className="size-4" />
              </button>
            </li>
          ))}
        </ul>
      )}
    </Field>
  );
}

/* ───────── page ───────── */

export default function AddTour() {
  const { data: divisions = [], isLoading: divisionLoading } = useGetDivisionsQuery(undefined);
  const { data: tourTypes = [], isLoading: tourTypeLoading } = useGetTourTypesQuery(undefined);
  const [addTour, { isLoading }] = useAddTourMutation();
  const navigate = useNavigate();

  // API-র data → SearchableSelect-এর { value, label }
  const divisionOptions = useMemo(
    () => divisions.map((item: IDivision) => ({ value: item._id, label: item.name })),
    [divisions],
  );
  const tourTypeOptions = useMemo(
    () => tourTypes.map((item: ITourType) => ({ value: item._id, label: item.name })),
    [tourTypes],
  );

  const form = useForm<FormValues>({
    resolver: zodResolver(formSchema),
    defaultValues: {
      title: "",
      division: "",
      tourType: "",
      description: "",
      images: [],
      location: "",
      departureLocation: "",
      arrivalLocation: "",
      startDate: "",
      endDate: "",
      costFrom: undefined,
      maxGuest: undefined,
      minAge: undefined,
      included: [],
      excluded: [],
      amenities: [],
      tourPlan: [],
    },
  });

  const title = form.watch("title");
  const startDate = form.watch("startDate");
  const today = getToday();

  const onSubmit = async (data: FormValues) => {
    const { images, ...rest } = data;

    // খালি লেখা / খালি তালিকা বাদ, যাতে backend শুধু ভরা field পায়
    const tourData = Object.fromEntries(
      Object.entries(rest).filter(([, v]) => v !== "" && v !== undefined && !(Array.isArray(v) && v.length === 0)),
    );

    const formData = new FormData();
    formData.append("data", JSON.stringify(tourData));
    images.forEach((file) => formData.append("files", file)); // backend: multerUpload.array("files")

    try {
      await runWithToast(addTour(formData).unwrap(), {
        loading: "Creating tour…",
        success: `"${data.title}" created`,
        error: "Couldn't create the tour.",
      });
      navigate("/admin/tours");
    } catch {
      // Error toast দেখানো হয়েছে; form-এ থাকি
    }
  };

  return (
    <div className="mx-auto flex w-full max-w-4xl flex-col gap-5">
      {/* Page header */}
      <div className="flex flex-col gap-3">
        <Link
          to="/admin/tours"
          className="inline-flex w-fit items-center gap-1.5 text-sm font-medium text-muted-foreground hover:text-foreground"
        >
          <ArrowLeft className="size-4" />
          Back to tours
        </Link>
        <div>
          <h1 className="text-3xl font-bold tracking-tight text-foreground">Add tour</h1>
          <p className="text-muted-foreground">Fill in the details below. Fields marked * are required.</p>
        </div>
      </div>

      <form id="add-tour" className="flex flex-col gap-5" onSubmit={form.handleSubmit(onSubmit)} noValidate>
        <fieldset disabled={isLoading} className="flex flex-col gap-5 disabled:opacity-60">
          {/* ── 1. Basic info ── */}
          <Section title="Basic info">
            <Controller
              name="title"
              control={form.control}
              render={({ field, fieldState }) => (
                <Field data-invalid={fieldState.invalid} className="gap-1.5">
                  <FieldLabel htmlFor="tour-title" className="text-sm font-semibold">
                    Title *
                  </FieldLabel>
                  <Input
                    {...field}
                    id="tour-title"
                    placeholder="e.g. Sajek Valley 3-Day Escape"
                    aria-invalid={fieldState.invalid}
                    className={inputCls}
                  />
                  <FieldDescription className="text-[13px]">
                    Slug: /tours/{makeSlug(title) || "your-tour-title"}
                  </FieldDescription>
                  {fieldState.invalid && <FieldError errors={[fieldState.error]} />}
                </Field>
              )}
            />

            <div className="grid grid-cols-1 gap-4 sm:grid-cols-2">
              <Controller
                name="division"
                control={form.control}
                render={({ field, fieldState }) => (
                  <Field data-invalid={fieldState.invalid} className="gap-1.5">
                    <FieldLabel htmlFor="tour-division" className="text-sm font-semibold">
                      Division *
                    </FieldLabel>
                    <SearchableSelect
                      id="tour-division"
                      options={divisionOptions}
                      value={field.value}
                      onChange={field.onChange}
                      invalid={fieldState.invalid}
                      disabled={divisionLoading}
                      placeholder={divisionLoading ? "Loading divisions…" : "Select a division"}
                      searchPlaceholder="Search division…"
                      emptyText="No division found."
                    />
                    {fieldState.invalid && <FieldError errors={[fieldState.error]} />}
                  </Field>
                )}
              />

              <Controller
                name="tourType"
                control={form.control}
                render={({ field, fieldState }) => (
                  <Field data-invalid={fieldState.invalid} className="gap-1.5">
                    <FieldLabel htmlFor="tour-type" className="text-sm font-semibold">
                      Tour type *
                    </FieldLabel>
                    <SearchableSelect
                      id="tour-type"
                      options={tourTypeOptions}
                      value={field.value}
                      onChange={field.onChange}
                      invalid={fieldState.invalid}
                      disabled={tourTypeLoading}
                      placeholder={tourTypeLoading ? "Loading tour types…" : "Select a tour type"}
                      searchPlaceholder="Search tour type…"
                      emptyText="No tour type found."
                    />
                    {fieldState.invalid && <FieldError errors={[fieldState.error]} />}
                  </Field>
                )}
              />
            </div>

            <Controller
              name="description"
              control={form.control}
              render={({ field }) => (
                <Field className="gap-1.5">
                  <FieldLabel htmlFor="tour-desc" className="text-sm font-semibold">
                    Description
                  </FieldLabel>
                  <Textarea
                    {...field}
                    id="tour-desc"
                    placeholder="What makes this tour special…"
                    className="min-h-32 resize-y rounded-lg px-3.5 py-2.5 text-[15px]"
                  />
                </Field>
              )}
            />
          </Section>

          {/* ── 2. Images ── */}
          <Section title="Images" description="First image is used as the cover.">
            <Controller
              name="images"
              control={form.control}
              render={({ field, fieldState }) => (
                <Field data-invalid={fieldState.invalid} className="gap-1.5">
                  <FieldLabel htmlFor="tour-images" className="sr-only">
                    Images
                  </FieldLabel>
                  <MultiImageUploader
                    id="tour-images"
                    value={field.value}
                    onChange={field.onChange}
                    onBlur={field.onBlur}
                    invalid={fieldState.invalid}
                  />
                  {fieldState.error && (
                    <FieldError>
                      {fieldState.error.message ?? fieldState.error.root?.message ?? "Some images are invalid."}
                    </FieldError>
                  )}
                </Field>
              )}
            />
          </Section>

          {/* ── 3. Location & dates ── */}
          <Section title="Location & dates">
            <div className="grid grid-cols-1 gap-4 sm:grid-cols-2">
              <Controller
                name="location"
                control={form.control}
                render={({ field }) => (
                  <Field className="gap-1.5 sm:col-span-2">
                    <FieldLabel htmlFor="tour-location" className="text-sm font-semibold">
                      Location
                    </FieldLabel>
                    <Input {...field} id="tour-location" placeholder="e.g. Sajek, Rangamati" className={inputCls} />
                  </Field>
                )}
              />
              <Controller
                name="departureLocation"
                control={form.control}
                render={({ field }) => (
                  <Field className="gap-1.5">
                    <FieldLabel htmlFor="tour-departure" className="text-sm font-semibold">
                      Departure location
                    </FieldLabel>
                    <Input {...field} id="tour-departure" placeholder="e.g. Dhaka (Arambagh)" className={inputCls} />
                  </Field>
                )}
              />
              <Controller
                name="arrivalLocation"
                control={form.control}
                render={({ field }) => (
                  <Field className="gap-1.5">
                    <FieldLabel htmlFor="tour-arrival" className="text-sm font-semibold">
                      Arrival location
                    </FieldLabel>
                    <Input {...field} id="tour-arrival" placeholder="e.g. Sajek Valley" className={inputCls} />
                  </Field>
                )}
              />
              <Controller
                name="startDate"
                control={form.control}
                render={({ field, fieldState }) => (
                  <Field data-invalid={fieldState.invalid} className="gap-1.5">
                    <FieldLabel htmlFor="tour-start" className="text-sm font-semibold">
                      Start date
                    </FieldLabel>
                    <Input
                      {...field}
                      id="tour-start"
                      type="date"
                      min={today}
                      onChange={(e) => {
                        field.onChange(e);
                        // শেষ তারিখ এখন শুরুর আগে পড়ে গেলে মুছে দাও
                        const end = form.getValues("endDate");
                        if (end && end < e.target.value) form.setValue("endDate", "");
                      }}
                      aria-invalid={fieldState.invalid}
                      className={inputCls}
                    />
                    {fieldState.invalid && <FieldError errors={[fieldState.error]} />}
                  </Field>
                )}
              />
              <Controller
                name="endDate"
                control={form.control}
                render={({ field, fieldState }) => (
                  <Field data-invalid={fieldState.invalid} className="gap-1.5">
                    <FieldLabel htmlFor="tour-end" className="text-sm font-semibold">
                      End date
                    </FieldLabel>
                    <Input
                      {...field}
                      id="tour-end"
                      type="date"
                      min={startDate || today}
                      aria-invalid={fieldState.invalid}
                      className={inputCls}
                    />
                    {fieldState.invalid && <FieldError errors={[fieldState.error]} />}
                  </Field>
                )}
              />
            </div>
          </Section>

          {/* ── 4. Pricing & guests ── */}
          <Section title="Pricing & guests">
            <div className="grid grid-cols-1 gap-4 sm:grid-cols-3">
              {(
                [
                  { name: "costFrom", id: "tour-cost", label: "Cost from (৳)", placeholder: "5500", min: 0 },
                  { name: "maxGuest", id: "tour-max-guest", label: "Max guests", placeholder: "20", min: 1 },
                  { name: "minAge", id: "tour-min-age", label: "Min age", placeholder: "12", min: 0 },
                ] as const
              ).map((item) => (
                <Controller
                  key={item.name}
                  name={item.name}
                  control={form.control}
                  render={({ field, fieldState }) => (
                    <Field data-invalid={fieldState.invalid} className="gap-1.5">
                      <FieldLabel htmlFor={item.id} className="text-sm font-semibold">
                        {item.label}
                      </FieldLabel>
                      <Input
                        id={item.id}
                        type="number"
                        min={item.min}
                        placeholder={item.placeholder}
                        name={field.name}
                        ref={field.ref}
                        onBlur={field.onBlur}
                        value={field.value ?? ""}
                        // লেখা → সংখ্যা (খালি → undefined)
                        onChange={(e) => field.onChange(e.target.value === "" ? undefined : e.target.valueAsNumber)}
                        // mouse scroll-এ মান বদলানো বন্ধ
                        onWheel={(e) => e.currentTarget.blur()}
                        aria-invalid={fieldState.invalid}
                        className={inputCls}
                      />
                      {fieldState.invalid && <FieldError errors={[fieldState.error]} />}
                    </Field>
                  )}
                />
              ))}
            </div>
          </Section>

          {/* ── 5. What's included ── */}
          <Section title="What's included" description="Type and press Enter (or +) to add.">
            <div className="grid grid-cols-1 gap-5 sm:grid-cols-2">
              <Controller
                name="included"
                control={form.control}
                render={({ field }) => (
                  <ListField
                    id="tour-included"
                    label="Included"
                    placeholder="e.g. AC bus ticket"
                    value={field.value}
                    onChange={field.onChange}
                  />
                )}
              />
              <Controller
                name="excluded"
                control={form.control}
                render={({ field }) => (
                  <ListField
                    id="tour-excluded"
                    label="Excluded"
                    placeholder="e.g. Personal expenses"
                    value={field.value}
                    onChange={field.onChange}
                  />
                )}
              />
              <Controller
                name="amenities"
                control={form.control}
                render={({ field }) => (
                  <ListField
                    id="tour-amenities"
                    label="Amenities"
                    placeholder="e.g. Wi-Fi"
                    value={field.value}
                    onChange={field.onChange}
                  />
                )}
              />
            </div>
          </Section>

          {/* ── 6. Tour plan ── */}
          <Section title="Tour plan" description="Add each day or step in order.">
            <Controller
              name="tourPlan"
              control={form.control}
              render={({ field }) => (
                <ListField
                  id="tour-plan"
                  label="Plan"
                  placeholder="e.g. Day 1: Dhaka → Khagrachari, then jeep to Sajek"
                  value={field.value}
                  onChange={field.onChange}
                  numbered
                />
              )}
            />
          </Section>

          {/* ── Sticky action bar ── */}
          <div className="sticky bottom-0 z-10 -mx-4 flex justify-end gap-2.5 border-t border-border bg-background/95 px-4 py-3.5 backdrop-blur">
            {/* fieldset link বন্ধ করে না, তাই আলাদা করে */}
            <Button asChild variant="outline" className="h-11 rounded-lg px-4 font-semibold" aria-disabled={isLoading}>
              <Link to="/admin/tours" className={isLoading ? "pointer-events-none opacity-50" : ""}>
                Cancel
              </Link>
            </Button>
            <Button type="submit" disabled={isLoading} className="h-11 min-w-32 gap-2 rounded-lg px-5 font-semibold">
              {isLoading ? (
                <>
                  <Loader2 className="size-4 animate-spin" />
                  Saving…
                </>
              ) : (
                "Save tour"
              )}
            </Button>
          </div>
        </fieldset>
      </form>
    </div>
  );
}
```

### `src/pages/Admin/Tours.tsx`

```tsx
import { DeleteConfirmation } from "@/components/DeleteConfirmation";
import { Button } from "@/components/ui/button";
import { Skeleton } from "@/components/ui/skeleton";
import { useGetToursQuery, useRemoveTourMutation } from "@/redux/features/tour/tour.api";
import type { ITour } from "@/types";
import { runWithToast } from "@/utils/runWithToast";
import {
  AlertCircle,
  CalendarDays,
  ChevronLeft,
  ChevronRight,
  ImageIcon,
  MapPin,
  MapPinned,
  Pencil,
  Plus,
  RefreshCw,
  Trash2,
  Users,
} from "lucide-react";
import { useState } from "react";
import { Link } from "react-router";

const LIMIT = 10;
const COLS = 8;

const formatDate = (date?: string) =>
  date ? new Date(date).toLocaleDateString("en-GB", { day: "numeric", month: "short", year: "numeric" }) : "";

const formatDateRange = (start?: string, end?: string) => {
  if (!start && !end) return "—";
  if (!end || start?.slice(0, 10) === end.slice(0, 10)) return formatDate(start ?? end);
  if (!start) return formatDate(end);
  return `${formatDate(start)} – ${formatDate(end)}`;
};

const formatPrice = (amount?: number) => (amount === undefined ? "—" : `৳${amount.toLocaleString("en-IN")}`);

export default function Tours() {
  const [page, setPage] = useState(1);

  const { data, isLoading, isError, isFetching, refetch } = useGetToursQuery({ page, limit: LIMIT });
  const [removeTour] = useRemoveTourMutation();

  const tours = data?.data ?? [];
  const total = data?.meta?.total ?? 0;
  const totalPage = data?.meta?.totalPage ?? 1;

  const handleRemoveTour = async (tour: ITour) => {
    await runWithToast(removeTour(tour._id).unwrap(), {
      loading: `Deleting "${tour.title}"…`,
      success: `"${tour.title}" deleted`,
      error: "Couldn't delete the tour.",
    });
    // এই পাতার শেষ tour মুছলে এক পাতা পেছনে
    if (tours.length === 1 && page > 1) setPage((p) => p - 1);
  };

  const from = total === 0 ? 0 : (page - 1) * LIMIT + 1;
  const to = Math.min(page * LIMIT, total);

  return (
    <div className="flex w-full flex-col gap-5">
      {/* Header + Add button */}
      <div className="flex flex-wrap items-end justify-between gap-3.5">
        <div>
          <h1 className="text-3xl font-bold tracking-tight text-foreground">Tours</h1>
          <p className="text-muted-foreground">
            {isLoading ? "Loading…" : `${total} tours`} · each has a public page at /tours/slug
          </p>
        </div>
        <Button asChild className="h-11 gap-2 rounded-lg px-4 text-[14.5px] font-semibold">
          <Link to="/admin/add-tour">
            <Plus className="size-4.5" strokeWidth={2.4} />
            Add tour
          </Link>
        </Button>
      </div>

      {/* Table */}
      <section className="overflow-hidden rounded-xl border border-border bg-card shadow-card">
        <div className="overflow-x-auto">
          <table className="w-full min-w-[960px] border-collapse text-left text-sm">
            <thead className="bg-muted">
              <tr className="text-[13px] font-semibold text-muted-foreground">
                <th scope="col" className="w-14 px-4.5 py-3">#</th>
                <th scope="col" className="px-4.5 py-3">Tour</th>
                <th scope="col" className="px-4.5 py-3">Division</th>
                <th scope="col" className="px-4.5 py-3">Type</th>
                <th scope="col" className="px-4.5 py-3">Dates</th>
                <th scope="col" className="px-4.5 py-3 text-right">Price from</th>
                <th scope="col" className="px-4.5 py-3 text-center">Guests</th>
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
                        <Skeleton className="h-12 w-18 rounded-lg" />
                        <div className="flex flex-col gap-1.5">
                          <Skeleton className="h-4 w-44" />
                          <Skeleton className="h-3 w-28" />
                        </div>
                      </div>
                    </td>
                    <td className="px-4.5 py-3"><Skeleton className="h-4 w-20" /></td>
                    <td className="px-4.5 py-3"><Skeleton className="h-5 w-20 rounded-full" /></td>
                    <td className="px-4.5 py-3"><Skeleton className="h-4 w-36" /></td>
                    <td className="px-4.5 py-3"><Skeleton className="ml-auto h-4 w-16" /></td>
                    <td className="px-4.5 py-3"><Skeleton className="mx-auto h-4 w-8" /></td>
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
                        <p className="font-semibold text-foreground">Couldn't load tours</p>
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
              {!isLoading && !isError && tours.length === 0 && (
                <tr className="border-t border-border">
                  <td colSpan={COLS} className="px-4.5 py-14">
                    <div className="flex flex-col items-center gap-3 text-center">
                      <span className="flex size-12 items-center justify-center rounded-full bg-accent text-accent-foreground">
                        <MapPinned className="size-5" />
                      </span>
                      <div>
                        <p className="font-semibold text-foreground">No tours yet</p>
                        <p className="text-[13.5px] text-muted-foreground">Create your first tour to start taking bookings.</p>
                      </div>
                      <Button asChild className="h-10 gap-2 rounded-lg">
                        <Link to="/admin/add-tour">
                          <Plus className="size-4" />
                          Add tour
                        </Link>
                      </Button>
                    </div>
                  </td>
                </tr>
              )}

              {/* ── Data ── */}
              {!isLoading &&
                !isError &&
                tours.map((tour, index) => (
                  <tr key={tour._id} className="border-t border-border transition-colors hover:bg-muted/60">
                    <td className="px-4.5 py-3 text-muted-foreground">{(page - 1) * LIMIT + index + 1}</td>

                    {/* Cover + title + location */}
                    <td className="px-4.5 py-3">
                      <div className="flex items-center gap-3">
                        <div className="flex h-12 w-18 shrink-0 items-center justify-center overflow-hidden rounded-lg bg-accent text-accent-foreground">
                          {tour.images?.[0] ? (
                            <img
                              src={tour.images[0]}
                              alt={tour.title}
                              width={72}
                              height={48}
                              loading="lazy"
                              className="size-full object-cover"
                            />
                          ) : (
                            <ImageIcon className="size-4.5" />
                          )}
                        </div>
                        <div className="min-w-0">
                          <p className="max-w-64 truncate font-semibold text-foreground" title={tour.title}>
                            {tour.title}
                          </p>
                          {tour.location && (
                            <p className="flex items-center gap-1 text-[13px] text-muted-foreground">
                              <MapPin className="size-3.5 shrink-0" />
                              <span className="max-w-56 truncate">{tour.location}</span>
                            </p>
                          )}
                        </div>
                      </div>
                    </td>

                    <td className="px-4.5 py-3 text-foreground">{tour.division?.name ?? "—"}</td>

                    <td className="px-4.5 py-3">
                      {tour.tourType ? (
                        <span className="inline-flex items-center rounded-full bg-accent px-2.5 py-0.5 text-[12.5px] font-semibold whitespace-nowrap text-accent-foreground">
                          {tour.tourType.name}
                        </span>
                      ) : (
                        <span className="text-muted-foreground">—</span>
                      )}
                    </td>

                    <td className="px-4.5 py-3">
                      <div className="flex items-center gap-1.5 whitespace-nowrap text-muted-foreground">
                        <CalendarDays className="size-3.5 shrink-0" />
                        {formatDateRange(tour.startDate, tour.endDate)}
                      </div>
                    </td>

                    <td className="px-4.5 py-3 text-right font-semibold whitespace-nowrap text-foreground">
                      {formatPrice(tour.costFrom)}
                    </td>

                    <td className="px-4.5 py-3">
                      <div className="flex items-center justify-center gap-1.5 text-muted-foreground">
                        <Users className="size-3.5" />
                        {tour.maxGuest ?? "—"}
                      </div>
                    </td>

                    <td className="px-4.5 py-3">
                      <div className="flex justify-end gap-2">
                        {/* Edit-এর page এলে চালু হবে */}
                        <Button
                          variant="outline"
                          size="icon"
                          type="button"
                          disabled
                          title="Coming soon"
                          aria-label={`Edit ${tour.title}`}
                          className="size-9 rounded-lg disabled:opacity-40"
                        >
                          <Pencil className="size-4" />
                        </Button>

                        <DeleteConfirmation
                          itemName={tour.title}
                          description="This removes the tour, its images and its public page. This can't be undone."
                          onConfirm={() => handleRemoveTour(tour)}
                        >
                          <Button
                            variant="outline"
                            size="icon"
                            type="button"
                            aria-label={`Delete ${tour.title}`}
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

        {/* Footer + pagination */}
        {!isLoading && !isError && total > 0 && (
          <div className="flex flex-wrap items-center justify-between gap-3 border-t border-border bg-muted px-4.5 py-3 text-[13.5px] text-muted-foreground">
            <span>
              Showing {from}–{to} of {total}
            </span>

            {totalPage > 1 && (
              <div className="flex items-center gap-2">
                <Button
                  variant="outline"
                  size="icon"
                  onClick={() => setPage((p) => p - 1)}
                  disabled={page === 1 || isFetching}
                  aria-label="Previous page"
                  className="size-9 rounded-lg"
                >
                  <ChevronLeft className="size-4" />
                </Button>
                <span className="min-w-20 text-center font-medium text-foreground">
                  Page {page} of {totalPage}
                </span>
                <Button
                  variant="outline"
                  size="icon"
                  onClick={() => setPage((p) => p + 1)}
                  disabled={page >= totalPage || isFetching}
                  aria-label="Next page"
                  className="size-9 rounded-lg"
                >
                  <ChevronRight className="size-4" />
                </Button>
              </div>
            )}
          </div>
        )}
      </section>
    </div>
  );
}
```

### Backend — `tour.service.ts`

`getAllTours` (ধাপ ১৪) যেমন আছে; `deleteTour` উন্নতি ৫-এর `Promise.allSettled` version।

---

# Check-list

**Form**
- [ ] Division আর tour type dropdown-এ database-এর আসল নাম; লিখে খোঁজা যায়; বাছা item-এ আবার click করলে খালি হয় আর "Please select…" দেখায়
- [ ] Title লিখলে নিচে slug-এর preview
- [ ] ক্যালেন্ডারে অতীত ধূসর; end-এ start-এর আগের দিন ধূসর; start পরে সরালে আগের end নিজে মুছে যায়
- [ ] হাতে অতীতের তারিখ টাইপ করলে browser-এর popup না, Zod-এর লাল message (`noValidate`)
- [ ] সংখ্যার box-এ mouse রেখে scroll করলে মান বদলায় না, page scroll হয়
- [ ] Included-এ লিখে Enter → item যোগ, form submit হয় না; একই item দুবার যোগ হয় না
- [ ] একসাথে কয়েকটা ছবি বাছা যায়, প্রথমটায় "Cover", ১০টা হলে বাক্স বন্ধ; 2 MB-এর বড় ছবিতে error

**পাঠানো**
- [ ] Network → `create` → Payload-এ `data` (খালি field ছাড়া JSON, সংখ্যা উদ্ধৃতি ছাড়া) আর একাধিক `files`
- [ ] Save চলার সময় সব field আর Cancel বন্ধ, button-এ "Saving…"
- [ ] সফল হলে toast-এ tour-এর নাম, Tours page-এ যায়, নতুন tour তালিকায়
- [ ] Backend error হলে লাল toast, form-এ থাকে, লেখা হারায় না

**তালিকা, pagination, মোছা**
- [ ] Table-এ division আর type-এর নাম (id না); division মুছে ফেলা tour-এ "—"
- [ ] ১০টার বেশি হলে "Page 1 of N"; পাতা বদলানোর সময় table ফিকে আর button বন্ধ; ক্রমিক নম্বর পাতা ২-এ ১১ থেকে
- [ ] মুছলে dialog-এ "Deleting…", Cloudinary থেকে সব ছবি যায়; শেষ পাতার একমাত্র tour মুছলে আগের পাতায়
- [ ] Tour type যোগ করলে Network-এ শুধু `tour-types` আবার আসে, `tour` না (আলাদা tag)
- [ ] Cloudinary ব্যর্থ হলেও tour মুছে যায় আর user "failed" দেখে না
