# Frontend Notes — ২৪: JavaScript Promise — Resolve, Reject আর "পরে জানাব"

এই project-এ আমরা প্রতিদিন promise ব্যবহার করেছি, নাম না জেনেই: `await login(data).unwrap()`, `try/catch`, `runWithToast`, `Promise.all`, `Promise.allSettled`। এই নোটে সেই ভিত্তিটা: **promise আসলে কী**, resolve আর reject কীভাবে কাজ করে, আর teacher-এর দেখানো "resolve/reject variable-এ রেখে পরে ডাকা" কৌশল।

প্রতিটা ধাপে 🎯 লক্ষ্য, 🛠️ কোড, 💡 ব্যাখ্যা, ✅ নিজে চালিয়ে দেখা। কোডগুলো browser-এর DevTools → **Console**-এ paste করে চালানো যায়।

---

## ভূমিকা: Promise এক কথায়

> **Promise = "এখনই ফল দিতে পারছি না, পরে দেব; দিতে পারলে জানাব, না পারলেও জানাব।"**

আপনার নিজের কথাতেই: *promise করছি এখন, কিন্তু কিছু সময় পরে গিয়ে সেটা resolve বা reject হবে।*

বাস্তবে এমন কাজ প্রচুর, যার ফল সাথে সাথে আসে না:
- Backend থেকে data আনা (Network-এ সময় লাগে)
- ছবি upload
- Timer (`setTimeout`)
- User-এর কোনো সিদ্ধান্তের অপেক্ষা (dialog-এ Delete চাপবে কিনা)

এগুলোর জন্য JavaScript থেমে বসে থাকে না; একটা **promise** ফেরত দেয়, আর বলে "ফল এলে জানাব"।

### Promise-এর তিন অবস্থা

```
              resolve(value)
           ┌──────────────────► fulfilled (সফল)  → .then চলে
pending ───┤
(অপেক্ষায়)  └──────────────────► rejected  (ব্যর্থ) → .catch চলে
              reject(reason)
```

| অবস্থা | মানে |
|---|---|
| **pending** | এখনো ফল আসেনি |
| **fulfilled** | সফল (`resolve` ডাকা হয়েছে) |
| **rejected** | ব্যর্থ (`reject` ডাকা হয়েছে, বা ভেতরে error হয়েছে) |

Fulfilled বা rejected, যেকোনো একটায় পৌঁছালে বলে **settled** (মীমাংসা হয়ে গেছে)। আর মনে রাখার সবচেয়ে জরুরি নিয়ম (ধাপ ৪):

> **একবার settled হলে আর কখনো বদলায় না।**

---

## ধাপ ১ — Resolve: সফল

🎯 **লক্ষ্য:** সবচেয়ে ছোট promise তৈরি আর তার ফল ধরা।

🛠️ **কোড:**

```js
new Promise((resolve, reject) => resolve())
  .then(() => console.log("resolved"))
  .catch(() => console.log("rejected"));

// Console: resolved
```

💡 **টুকরো টুকরো করে:**

```js
new Promise(               // ১. নতুন promise বানাও
  (resolve, reject) => {   // ২. এই function সাথে সাথে চলে (executor)
    resolve();             // ৩. "সফল" ঘোষণা
  }
)
  .then(() => ...)         // ৪. সফল হলে এটা
  .catch(() => ...);       // ৫. ব্যর্থ হলে এটা
```

- **Executor function:** `new Promise(...)`-এর ভেতরে যে function দিই, JavaScript সেটা **সাথে সাথে** চালায়, আর তাকে দুটো জিনিস দেয়: `resolve` আর `reject`। এই দুটো নিজেও function, আমরা বানাইনি, JavaScript দিয়েছে।
- **`resolve()` ডাকলাম**, তাই promise হলো fulfilled → `.then`-এর function চলল → "resolved"।
- `.catch` চলেনি, কারণ reject হয়নি।

আপনি ঠিকই বলেছেন: *resolve-কে call করছি বলেই "resolved"।*

---

## ধাপ ২ — Reject: ব্যর্থ

🛠️ **কোড:**

```js
new Promise((resolve, reject) => reject())
  .then(() => console.log("resolved"))
  .catch(() => console.log("rejected"));

// Console: rejected
```

💡 `reject()` ডাকায় promise rejected → `.then` **পুরোটা বাদ পড়ে**, সরাসরি `.catch` চলে।

### 💡 মান পাঠানো

`resolve` আর `reject`-এ মান দেওয়া যায়, আর সেটা `.then`/`.catch`-এর function পায়:

```js
new Promise((resolve) => resolve("চা রেডি ☕"))
  .then((value) => console.log(value));        // চা রেডি ☕

new Promise((_, reject) => reject(new Error("দোকান বন্ধ")))
  .catch((err) => console.log(err.message));   // দোকান বন্ধ
```

> 💡 **Reject-এ string না, `Error` দেওয়া ভালো অভ্যাস:** `reject("দোকান বন্ধ")`-ও চলে, কিন্তু `new Error("…")` দিলে কোন লাইনে সমস্যা হয়েছে (stack trace) সেটাও থাকে, আর সব জায়গায় `err.message` দিয়ে পড়া যায়। Backend-এর `throw new AppError(...)` বা `throw new Error(...)`-এর একই কারণ।

### 💡 ভেতরে error হলেও reject

```js
new Promise(() => {
  throw new Error("ভেতরে কিছু ভেঙেছে");
}).catch((err) => console.log(err.message));   // ভেতরে কিছু ভেঙেছে
```

Executor-এর ভেতরে `throw` হলে JavaScript নিজেই সেটাকে `reject` ধরে নেয়।

✅ **নিজে চালিয়ে দেখুন:** Console-এ ধাপ ১ আর ২-এর কোড paste করুন।

---

## ধাপ ৩ — `resolve`/`reject` বাইরে রেখে পরে ডাকা

🎯 **লক্ষ্য:** Teacher-এর দেখানো কৌশল: promise এখন বানাও, কিন্তু সফল না ব্যর্থ সেটা **পরে, অন্য জায়গা থেকে** ঠিক করো।

🛠️ **কোড:**

```js
let savedResolve, savedReject;

const myPromise = new Promise((resolve, reject) => {
  savedResolve = resolve;   // resolve function-টা বাইরের variable-এ রেখে দিলাম
  savedReject = reject;     // reject-ও
});

myPromise
  .then((value) => console.log("Promise Resolved:", value))
  .catch((err) => console.log("Promise Rejected:", err));

// এখনো কিছুই print হয়নি — promise pending, কেউ সিদ্ধান্ত দেয়নি

setTimeout(() => {
  savedResolve("He he etai bastob");
}, 3000);

// ৩ সেকেন্ড পরে: Promise Resolved: He he etai bastob
```

💡 **কী হলো, ধাপে ধাপে:**
1. Executor সাথে সাথে চলল, কিন্তু resolve/reject **ডাকল না**, শুধু বাইরের variable-এ রেখে দিল। Promise **pending**।
2. `.then`/`.catch` লাগানো হলো; এরা অপেক্ষা করছে।
3. ৩ সেকেন্ড পরে কোডের সম্পূর্ণ অন্য জায়গা থেকে `savedResolve(...)` → promise fulfilled → `.then` চলল।

> ⚠️ **সংশোধন (ছোট কিন্তু জরুরি):** নোটে লেখা ছিল "**promise-টাকে** variable-এ store করে রাখতে পারি"। Promise variable-এ রাখা তো সবসময়ই যায় (`const myPromise = …`), এখানে নতুন যা হলো সেটা হলো **`resolve` আর `reject` function দুটোকে** বাইরে রাখা। Promise থাকল `myPromise`-এ (যার ওপর `.then` লাগাই), আর সিদ্ধান্তের "চাবি" দুটো থাকল `savedResolve`/`savedReject`-এ (যেটা ডেকে ফল ঠিক করি)। দুটো আলাদা জিনিস, দুটো আলাদা হাতে থাকতে পারে, এটাই এই কৌশলের শক্তি।

```
myPromise                     ← যে অপেক্ষা করছে, সে এটা পায় (.then/.catch লাগায়)
savedResolve / savedReject    ← যে সিদ্ধান্ত দেবে, সে এটা পায় (পরে ডাকে)
```

---

## ধাপ ৪ — একবার মীমাংসা মানে চিরকাল (আপনার উদাহরণে যা আসলে ঘটে)

🎯 **লক্ষ্য:** একই promise-এ বারবার resolve/reject ডাকলে কী হয়, বোঝা।

আপনার নোটের কোড একসাথে চালালে:

```js
let savedResolve, savedReject;
const myPromise = new Promise((resolve, reject) => {
  savedResolve = resolve;
  savedReject = reject;
});

savedResolve("Ami Resolve hoye gesi");          // ১

myPromise
  .then((value) => console.log("Promise Resolved", value))
  .catch((err) => console.log("Promise Rejected", err));

savedReject("Kono akata error hoye gese");      // ২

setTimeout(() => {
  savedResolve("He he etai bastob");            // ৩
}, 3000);
```

**Console-এ শুধু একটা লাইন:**

```
Promise Resolved Ami Resolve hoye gesi
```

💡 **কেন বাকি দুটো কিছুই করল না:**

| লাইন | promise-এর অবস্থা আগে | ফল |
|---|---|---|
| ১. `savedResolve("Ami Resolve…")` | pending | **fulfilled**, মান মনে রাখল |
| ২. `savedReject("Kono…")` | fulfilled (settled) | **উপেক্ষা** — কিছুই হয় না, কোনো error-ও না |
| ৩. ৩ সেকেন্ড পরে `savedResolve("He he…")` | fulfilled | **উপেক্ষা** |

> ⚠️ **সংশোধন:** নোটে তিনটা ডাক একসাথে লেখা আছে, দেখে মনে হতে পারে তিনটাই কাজ করবে, বা শেষেরটা ("He he etai bastob") দেখাবে। আসলে **প্রথম সিদ্ধান্তটাই চূড়ান্ত**। Promise একবার settled হলে তার অবস্থা আর মান কখনো বদলায় না, পরের সব resolve/reject নিঃশব্দে উপেক্ষিত। ৩ সেকেন্ড পরের লাইন দেখতে চাইলে প্রথম দুটো (`savedResolve(...)` আর `savedReject(...)`) মুছে চালান, অথবা প্রতিটার জন্য আলাদা নতুন promise বানান।

এটা আসলে promise-এর একটা **নিরাপত্তা**: কথা দিয়ে কথা ফিরিয়ে নেওয়া যায় না। Payment "সফল" বলার পরে কোনো ভুলে আবার "ব্যর্থ" বলে ফেলা অসম্ভব।

### 💡 আরও একটা জিনিস চোখে পড়ার মতো

লাইন ১-এ resolve **আগে** হয়ে গেছে, `.then` লাগানো হয়েছে **পরে**, তবু `.then` চলল। কারণ promise **মান মনে রাখে**। পরে যে-ই `.then` লাগাক, সাথে সাথে সেই মান পায়। এজন্যই RTK Query-র cache-এর মতো জিনিস সম্ভব: একবার আনা ফল অনেকে পরে নিতে পারে।

✅ **নিজে চালিয়ে দেখুন:** উপরের কোড Console-এ চালান, তারপর লাইন ১ আর ২ মুছে আবার চালান, ৩ সেকেন্ড পরে "He he etai bastob" আসবে।

---

## ধাপ ৫ — বাস্তব উদাহরণ: চায়ের দাওয়াত

আপনার উদাহরণটা একটু গুছিয়ে, কে কোন অংশ ধরে আছে পরিষ্কার করে:

> সকালে বন্ধু বলল: "বিকাল ৪টায় চা খেতে বের হবি?" আপনি বললেন: "এখন বলতে পারছি না, কাজ শেষ করে জানাব।"

```js
let yes, no;

// আপনি কথা দিলেন: "জানাব"
const teaPlan = new Promise((resolve, reject) => {
  yes = resolve;   // "হ্যাঁ" বলার চাবি আপনার পকেটে
  no = reject;     // "না" বলার চাবিও
});

// বন্ধু নিজের পরিকল্পনা করে রাখল, উত্তরের অপেক্ষায়
teaPlan
  .then((place) => console.log(`বন্ধু: চল, ${place}-এ দেখা হবে!`))
  .catch((reason) => console.log(`বন্ধু: ঠিক আছে, ${reason}। পরে একদিন।`));

// ...দুপুরে কাজ শেষ, মন চাইল
setTimeout(() => yes("টং দোকান"), 2000);
// অথবা: setTimeout(() => no("আজ অনেক কাজ"), 2000);
```

| গল্পে | কোডে |
|---|---|
| আপনার "জানাব" কথা | `new Promise(...)` → `teaPlan` |
| আপনার পকেটের "হ্যাঁ"/"না" | `yes` / `no` (resolve/reject) |
| বন্ধুর "হ্যাঁ হলে…", "না হলে…" পরিকল্পনা | `.then(...)` / `.catch(...)` |
| দুপুর পর্যন্ত অপেক্ষা | pending |
| "হ্যাঁ, টং দোকানে" | `yes("টং দোকান")` → fulfilled |
| একবার "হ্যাঁ" বলার পরে আবার "না" | কাজ করে না (ধাপ ৪) |

> ⚠️ **সংশোধন (গল্পে কে কী):** নোটে লেখা ছিল "বন্ধু দুটো function invoke করে রাখল, আমি resolve বা reject চেপে দেব"। Function দুটো (`resolve`/`reject`) আসলে **promise তৈরির সময় JavaScript দেয়**, আর সেগুলো থাকে **যে সিদ্ধান্ত দেবে তার কাছে** (আপনি)। বন্ধু যা করে রাখে সেটা `.then`/`.catch`: "উত্তর এলে কী করব"। আর "invoke করে রাখা" না, **রেখে দেওয়া** (store); invoke মানে ডাকা, সেটা হয় পরে, যখন আপনি সিদ্ধান্ত নেন।

---

## ধাপ ৬ — `.then` সবসময় "একটু পরে" চলে

🎯 **লক্ষ্য:** Promise-এর ফল **কখন** চলে, বোঝা (অনেক bug-এর উৎস)।

🛠️ **অনুমান করুন Console-এ ক্রম কী হবে:**

```js
console.log("১");

Promise.resolve().then(() => console.log("২"));

console.log("৩");
```

**উত্তর:** `১`, `৩`, `২`

💡 **কেন:** Promise আগেই resolved, তবু `.then`-এর function **সাথে সাথে** চলে না। JavaScript আগে বর্তমান সব সাধারণ কোড শেষ করে (`১`, `৩`), তারপর promise-এর অপেক্ষমাণ কাজগুলো চালায় (`২`)। একে বলে **asynchronous** (অসমকালিক)।

এজন্যই:

```js
let data;
fetchSomething().then((result) => { data = result; });
console.log(data);   // undefined! .then এখনো চলেনি
```

ফল **ব্যবহার করতে হবে `.then`-এর ভেতরে** (বা `await`-এর পরে, ধাপ ৮)।

---

## ধাপ ৭ — Chaining আর `.finally`

🎯 **লক্ষ্য:** একটার পর একটা কাজ, আর সবশেষে যা-ই হোক এমন কাজ।

```js
new Promise((resolve) => resolve(2))
  .then((n) => n * 10)              // ২ → ২০ (return করা মান পরের .then পায়)
  .then((n) => {
    if (n > 15) throw new Error("বেশি বড়!");
    return n;
  })
  .then((n) => console.log("এটা চলবে না", n))   // উপরে throw হয়েছে, তাই বাদ
  .catch((err) => console.log("ধরা পড়ল:", err.message))   // ধরা পড়ল: বেশি বড়!
  .finally(() => console.log("সব শেষ, loading বন্ধ"));     // সফল/ব্যর্থ যা-ই হোক
```

💡 **নিয়মগুলো:**
- `.then` যা **return** করে, পরের `.then` সেটা পায়।
- যেকোনো `.then`-এ **error হলে** (throw), মাঝের সব `.then` বাদ দিয়ে সরাসরি পরের `.catch`-এ যায়। একটা `.catch` পুরো শিকলের সব error ধরে।
- **`.finally`** সফল বা ব্যর্থ, দুই ক্ষেত্রেই চলে। "Loading বন্ধ করো"-র মতো কাজের জন্য। (১৩ নম্বর নোটের logout-এ `finally { dispatch(resetApiState()) }` ঠিক এটাই।)

> ⚠️ **`.catch` না দিলে:** promise reject হলো কিন্তু কেউ `.catch` বা `try/catch` দিয়ে ধরল না, তখন Console-এ লাল **"Uncaught (in promise)"**। ১৮ নম্বর নোটে teacher-এর delete-এর কোডে আর ১৭ নম্বর নোটে add tour type-এ এটাই হচ্ছিল।

---

## ধাপ ৮ — `async`/`await`: একই জিনিস, পড়তে সহজ

🎯 **লক্ষ্য:** এই project-এ আমরা যেভাবে promise লিখেছি, তার সাথে জোড়া লাগানো।

`.then`/`.catch` আর `async`/`await` **একই promise**-এর দুই রকম লেখা:

```js
// .then / .catch
function bookTour() {
  createBooking(payload)
    .then((res) => {
      window.location.href = res.paymentUrl;
    })
    .catch((err) => {
      toast.error(err.message);
    });
}

// async / await — একই কাজ
async function bookTour() {
  try {
    const res = await createBooking(payload);   // promise resolve হওয়া পর্যন্ত এখানে অপেক্ষা
    window.location.href = res.paymentUrl;
  } catch (err) {                               // reject হলে এখানে
    toast.error(err.message);
  }
}
```

| `.then`/`.catch` | `async`/`await` |
|---|---|
| `.then((value) => …)` | `const value = await promise;` |
| `.catch((err) => …)` | `try { … } catch (err) { … }` |
| `.finally(() => …)` | `finally { … }` |

💡 **`await` মানে:** "এই promise মীমাংসা হওয়া পর্যন্ত **এই function-এর** বাকি অংশ থামিয়ে রাখো" (পুরো browser না, অন্য কাজ চলতে থাকে)। Resolve হলে মান ফেরত দেয়, reject হলে error **throw** করে, তাই `try/catch` দিয়ে ধরা যায়। `await` শুধু `async` function-এর ভেতরে লেখা যায়।

### 💡 এই project-এর কোডে promise কোথায় ছিল

| আমাদের কোড | Promise-এর ভাষায় |
|---|---|
| `await login(data).unwrap()` (১৩ নম্বর নোট) | RTK Query-র mutation একটা promise দেয়; `.unwrap()` সেটাকে এমন promise বানায় যেটা error হলে **reject** করে, তাই `catch` চলে |
| `runWithToast(promise, …)` (১৮ নম্বর নোট) | promise নিয়ে `await` করে: resolve হলে success toast, reject হলে error toast আর আবার `throw` |
| `DeleteConfirmation`-এর `await onConfirm()` (১৮) | delete-এর promise মীমাংসা হওয়া পর্যন্ত dialog খোলা; reject হলে খোলাই থাকে |
| `Promise.all([data, meta])` (২০) | অনেক promise একসাথে; **সবগুলো** resolve হলে resolve, **একটাও** reject হলে সাথে সাথে reject |
| `Promise.allSettled(...)` (২০) | সবগুলো মীমাংসা হওয়া পর্যন্ত অপেক্ষা, প্রত্যেকের আলাদা ফল (কারো reject-এ পুরোটা ব্যর্থ হয় না) |
| `lazy(() => import(...))` (১৫) | dynamic `import()` একটা promise দেয়, chunk download হলে resolve |

---

## ধাপ ৯ — "Resolve বাইরে রাখা" কৌশল আসলে কোথায় লাগে

🎯 **লক্ষ্য:** ধাপ ৩-এর কৌশল (একে বলে **deferred** pattern) বাস্তবে কেন দরকার।

সাধারণ promise-এ সিদ্ধান্ত হয় promise-এর **ভেতরেই** (executor-এ)। কিন্তু কখনো কখনো সিদ্ধান্ত আসে **সম্পূর্ণ অন্য জায়গা থেকে**, পরে:

**উদাহরণ ১ — User-এর সিদ্ধান্তের অপেক্ষা:**

```js
// "নিশ্চিত?" dialog খোলো, user কী চাপবে তার জন্য একটা promise ফেরত দাও
function askConfirm() {
  return new Promise((resolve) => {
    openDialog({
      onYes: () => resolve(true),    // user Yes চাপলে, অনেক পরে
      onNo: () => resolve(false),
    });
  });
}

const ok = await askConfirm();   // user চাপা পর্যন্ত এখানে অপেক্ষা
if (ok) deleteTour();
```

**উদাহরণ ২ — Token নতুন হওয়া পর্যন্ত অন্য request-দের অপেক্ষা (axios interceptor):**
১৩ নম্বর নোটের ধাপ ৭-এ দেখেছি, access token-এর মেয়াদ শেষ হলে interceptor নতুন token আনে। কিন্তু একই সময়ে ৫টা request ব্যর্থ হলে ৫ বার refresh করা ভুল। তখন:
1. প্রথম request refresh শুরু করে।
2. বাকি ৪টা প্রতিটা একটা নতুন promise বানায়, আর তার `resolve`/`reject` একটা **তালিকায় (queue) রেখে দেয়**, অপেক্ষা করে।
3. Refresh সফল হলে তালিকার সবার `resolve()` ডাকা হয় → ৪টাই আবার চেষ্টা করে। ব্যর্থ হলে সবার `reject()`।

এটা ঠিক ধাপ ৩-এর কৌশল: promise এখন, সিদ্ধান্ত পরে অন্য জায়গা থেকে। Teacher সম্ভবত সামনে এই কাজের জন্যই promise-এর এই অংশটা দেখালেন।

> নোট: নতুন JavaScript-এ এই কাজের জন্য তৈরি একটা উপায়ও আছে:
> ```js
> const { promise, resolve, reject } = Promise.withResolvers();
> ```
> ধাপ ৩-এর `let savedResolve…` লেখার ছোট রূপ, একই কাজ। আধুনিক সব browser-এ চলে।

---

# সারসংক্ষেপ

| বিষয় | মনে রাখার কথা |
|---|---|
| Promise | "এখন না, পরে ফল দেব" |
| তিন অবস্থা | pending → fulfilled (resolve) বা rejected (reject) |
| **একবারই** | প্রথম resolve/reject-ই চূড়ান্ত, পরেরগুলো উপেক্ষিত |
| মান মনে রাখে | পরে লাগানো `.then`-ও ফল পায় |
| সবসময় পরে চলে | `.then` বর্তমান কোড শেষ হওয়ার পরে চলে |
| Error | `throw` = reject; না ধরলে "Uncaught (in promise)" |
| `.then`/`.catch`/`.finally` | = `await` / `try…catch` / `finally` |
| Deferred | `resolve`/`reject` বাইরে রেখে পরে অন্য জায়গা থেকে সিদ্ধান্ত |

---

# নিজেকে যাচাই (Console-এ চালিয়ে মিলিয়ে নিন)

**প্রশ্ন ১:** কী print হবে?
```js
const p = new Promise((resolve, reject) => {
  reject("না");
  resolve("হ্যাঁ");
});
p.then(console.log).catch(console.log);
```
<details><summary>উত্তর</summary>

`না` — প্রথম সিদ্ধান্ত reject, পরের resolve উপেক্ষিত।
</details>

**প্রশ্ন ২:** ক্রম কী?
```js
console.log("A");
setTimeout(() => console.log("B"), 0);
Promise.resolve().then(() => console.log("C"));
console.log("D");
```
<details><summary>উত্তর</summary>

`A`, `D`, `C`, `B` — সাধারণ কোড আগে (A, D), তারপর promise-এর কাজ (C), তারপর timer (B)। Promise-এর অপেক্ষমাণ কাজ timer-এর আগে চলে, ০ millisecond-এর timer হলেও।
</details>

**প্রশ্ন ৩:** `.catch` চলবে?
```js
Promise.resolve(5)
  .then((n) => JSON.parse("{bad json"))
  .catch(() => console.log("caught"));
```
<details><summary>উত্তর</summary>

হ্যাঁ, `caught` — `.then`-এর ভেতরে error (ভুল JSON) মানে reject, তাই `.catch` ধরে।
</details>

**প্রশ্ন ৪:** কেন এটা `undefined` দেখায়, আর কীভাবে ঠিক করবেন?
```js
let user;
fetch("/api/user/me").then((r) => r.json()).then((d) => (user = d));
console.log(user);
```
<details><summary>উত্তর</summary>

`.then` পরে চলে (ধাপ ৬), `console.log` আগে। ঠিক করতে: `console.log` শেষ `.then`-এর ভেতরে, অথবা `async` function-এ `const user = await (await fetch(...)).json();`।
</details>
