# Frontend Notes — ০১: Project Structure Intro

> এই নোটে দুটো আলাদা সিদ্ধান্ত নিয়ে আলোচনা:
> ১. **Project organization** — কোড কয়টা repo-তে থাকবে (Monorepo vs Polyrepo)
> ২. **Architectural pattern** — অ্যাপ্লিকেশন কীভাবে ভাগ হবে ও deploy হবে (Monolith vs Microservice)
>
> ⚠️ এই দুটো **আলাদা বিষয়**। Repo structure মানে কোড কোথায় রাখা হচ্ছে; architecture মানে অ্যাপ runtime-এ কীভাবে চলছে। একটা monolith polyrepo-তে থাকতে পারে, আবার microservice-গুলোও একটা monorepo-তে থাকতে পারে।

---

## ১. Project Organization

প্রজেক্ট শুরুর আগে একটা সিদ্ধান্ত নিতে হয় — **monorepo** করব নাকি **polyrepo**।

| | Monorepo | Polyrepo |
|---|---|---|
| গঠন | frontend + backend একই repo-তে (যেমন `client/`, `server/` ফোল্ডার) | `client`-এর জন্য একটা repo, `server`-এর জন্য আরেকটা repo |
| CI/CD | সাধারণত একটা pipeline (বা path অনুযায়ী আলাদা job) | প্রতিটা repo-র নিজস্ব pipeline |

### Polyrepo

**সুবিধা:**
- শুরুতে maintain করা সহজ। Frontend developer শুধু frontend repo নিয়ে, backend developer শুধু backend repo নিয়ে কাজ করে — ফলে প্রতিটা repo ছোট ও বোঝা সহজ থাকে।
- আলাদা access control, আলাদা release cycle রাখা যায়।

**অসুবিধা:**
- Frontend আর backend আলাদাভাবে deploy করতে হয়।
- দুই repo-র মধ্যে shared code (যেমন API types, validation schema) ভাগ করা ঝামেলার — আলাদা package বানাতে হয় বা কপি করতে হয়।
- API-তে breaking change হলে দুই repo-তে আলাদা PR লাগে, sync রাখা কঠিন।

#### CORS প্রসঙ্গে (সংশোধিত)

আলাদা deploy করলে frontend আর backend সাধারণত **আলাদা origin**-এ থাকে (যেমন `app.example.com` আর `api.example.com`)। তখন browser **CORS** error দেখায় — এটা খুবই common।

সঠিক ধারণা:
- **Origin** = protocol + domain + port। এর যেকোনো একটা আলাদা হলেই origin আলাদা।
- CORS কোনো bug না, এটা **browser-এর security policy**। Browser অন্য origin-এ request পাঠানোর পর দেখে, server response-এ `Access-Control-Allow-Origin` header দিয়ে এই origin-কে অনুমতি দিয়েছে কিনা। না দিলে browser response-টা JavaScript-কে দেয় না।
- CORS **repo structure-এর কারণে হয় না**, হয় **deployment-এর origin আলাদা হওয়ার কারণে**। Monorepo হলেও frontend আর backend আলাদা domain-এ deploy করলে একই সমস্যা হবে।
- **সমাধান:** backend-এ CORS configure করা (allowed origins বলে দেওয়া), অথবা একটা reverse proxy (যেমন Nginx) দিয়ে দুটোকে একই origin-এ serve করা (`example.com` → frontend, `example.com/api` → backend)।

### Monorepo

**সুবিধা:**
- সব কোড এক জায়গায়, তাই frontend আর backend-এর মধ্যে **shared code/types** সহজে ব্যবহার করা যায়।
- একটা commit/PR-এ frontend ও backend দুটোর change একসাথে করা যায় (atomic change)।
- একটা `docker-compose.yml` বা Dockerfile দিয়ে পুরো সিস্টেম একসাথে চালানো/deploy করা তুলনামূলক সহজ।
- সবকিছু co-related হওয়ায় আলাদা আলাদা repo handle করতে হয় না।

**অসুবিধা:**
- প্রজেক্ট বড় হলে repo ভারী হয়ে যায়, CI build time বাড়ে।
- বড় monorepo-র জন্য বিশেষ tooling লাগতে পারে (যেমন Turborepo, Nx, pnpm workspaces)।

> নোট: Monorepo মানেই এক deployment না। Monorepo-তেও frontend আর backend আলাদা জায়গায় deploy করা যায়।

---

## ২. Architectural Pattern

Architecture মূলত দুই রকম: **Monolith** আর **Microservice**।

### Microservice

অ্যাপ্লিকেশনকে ছোট ছোট **স্বাধীন service**-এ ভাগ করা হয়। প্রতিটা service-এর নিজস্ব codebase, নিজস্ব database, নিজস্ব deployment।

**উদাহরণ:** একটা e-commerce অ্যাপে —
- `payment-service` — আলাদা project, আলাদা codebase, আলাদা DB
- `user-service` — আলাদা project, আলাদা codebase, আলাদা DB

তারপর এগুলোকে connect করে frontend-এ data পাঠানো হয়।

**সুবিধা:**
- **Fault isolation:** ধরুন hacker DDoS attack দিয়ে `user-service` down করে দিল। `payment-service` যেহেতু আলাদা, সেটা চালু থাকতে পারে। Payment data payment service-এর নিজস্ব DB-তে থাকবে; পরে user service restore হলে দুটো sync করে নেওয়া যাবে।
  > ⚠️ সংশোধন: এটা তখনই কাজ করে যখন payment service, user service-এর ওপর সরাসরি (synchronously) নির্ভর না করে। Payment করতে যদি user authentication বা user data লাগে, তাহলে user service down থাকলে payment-ও ব্যর্থ হবে। এজন্য microservice-এ message queue / event-based (asynchronous) communication ব্যবহার করা হয়।
- **Independent scaling:** যে service-এ load বেশি, শুধু সেটাকে scale করা যায় (যেমন sale-এর সময় শুধু `payment-service`)।
- **নতুন feature যোগ করা সহজ:** যেমন subscription feature লাগলে `subscription-service` নামে নতুন service বানিয়ে ecosystem-এ যোগ করা যায়।
- প্রতিটা service-এ আলাদা technology ব্যবহার করা যায়, আলাদা team আলাদা service নিয়ে কাজ করতে পারে।

**অসুবিধা:**
- সময়ের সাথে সিস্টেম অনেক **complex** হয়ে যায়।
- Frontend জানে না কোন data কোন service থেকে আনতে হবে। এটা সমাধানে সাধারণত একটা **API Gateway** রাখা হয় — frontend শুধু gateway-কে call করে, gateway সঠিক service-এ পাঠায়।
- Service-গুলো একে অপরের ওপর নির্ভরশীল (subscription → payment → user)। এদের মধ্যে communication-এর জন্য **messaging system** (যেমন RabbitMQ, Kafka) রাখতে হয়।
- আলাদা DB হওয়ায় data consistency রাখা কঠিন (distributed transaction)।
- Monitoring, logging, debugging, deployment — সবকিছুই বেশি কঠিন।

**পরামর্শ:** যতদিন সম্ভব monolith দিয়ে এগোনো ভালো। যখন দেখবেন monolith-এ আর scale করা যাচ্ছে না, architecture-এ আটকে যাচ্ছেন — তখন microservice-এ যাবেন।

**ইতিহাস (সংশোধিত):** Microservice-এর প্রথম দিকের বড় adopter-দের মধ্যে Amazon, Netflix আর Uber উল্লেখযোগ্য। Netflix ২০০৮-এর একটা বড় database outage-এর পর monolith থেকে ধীরে ধীরে microservice-এ যায়; Uber-ও শুরুতে monolith ছিল। অর্থাৎ এরা কেউই শুরু থেকে microservice দিয়ে শুরু করেনি — scale-এর প্রয়োজনে পরে migrate করেছে। (Netflix-কে "প্রথম adopter" বলা পুরোপুরি ঠিক না; Amazon এর আগেই service-oriented architecture-এ যাওয়া শুরু করেছিল।)

### Monolith

পুরো backend একটা **single unit** হিসেবে তৈরি ও deploy হয়।

- **Single codebase** (backend-এর জন্য)
- **Single database** — payment, user, সবকিছুর জন্য একটাই DB
- **Single deployment** — পুরো backend একসাথে deploy হয়

Microservice-এ payment-এর জন্য আলাদা DB, user-এর জন্য আলাদা DB; কিন্তু monolith-এ সবকিছুর জন্য একটাই DB।

**সুবিধা:**
- বানানো, বোঝা, debug ও test করা সহজ।
- Service-এর মধ্যে network call নেই, তাই দ্রুত ও কম ঝামেলার।
- Data consistency রাখা সহজ (একটাই DB, সাধারণ transaction)।

**অসুবিধা:**
- Scale হতে হতে একসময় codebase বিশাল হয়ে যায়; deployment ও maintain কঠিন হয়ে পড়ে।
- ছোট একটা change-এর জন্যও পুরো অ্যাপ redeploy করতে হয়।
- একটা অংশের bug পুরো অ্যাপকে down করে দিতে পারে।
- Scale করতে হলে পুরো অ্যাপটাই scale করতে হয়, শুধু দরকারি অংশ না।

> 💡 মাঝামাঝি একটা পথ হলো **Modular Monolith** — একটাই deployment, কিন্তু কোডের ভেতরে module-গুলো (user, payment, …) পরিষ্কারভাবে আলাদা রাখা। পরে দরকার হলে কোনো module-কে আলাদা service বানানো সহজ হয়।

---

## ৩. আমাদের প্রজেক্টে কী করছি

- **Architecture:** Monolith — একটা backend, একটা database, একটা frontend। Multiple service নেই, multiple DB নেই, multiple frontend নেই।
- **Repo:** Polyrepo — frontend আলাদা repo, backend আলাদা repo।
- **Deployment:** প্রতিটা repo-র নিজস্ব CI/CD pipeline থাকবে, অথবা deployment-এর সময় দুটো repo একসাথে এনে (যেমন `docker-compose` দিয়ে) একটা pipeline-এ deploy করা হবে।

> ⚠️ সংশোধন: Frontend আর backend আলাদা codebase হলেও এটা **monolith-ই থাকে**। "Monolith" কথাটা মূলত **backend**-এর গঠন বোঝায় — backend একটা single deployable unit কিনা। Frontend একটা আলাদা SPA (React ইত্যাদি) হওয়া মানে microservice না। তাই "single codebase" বলতে এখানে বোঝানো হচ্ছে **backend-এর single codebase**।