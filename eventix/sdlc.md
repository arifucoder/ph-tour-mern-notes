# Eventix — Requirements আর System Design Guide

> **Eventix** — একটা Event Management আর Ticketing platform। Organizer event বানায় আর টিকিট বিক্রি করে, user টিকিট কিনে QR code পায়। Event হতে পারে **venue-তে (in-person)**, **অনলাইনে (virtual)**, অথবা **দুটোই একসাথে (hybrid)** — venue-তে QR scan করে ঢোকে, অনলাইনে ticket দিয়ে meeting link পায়। Event **Free** (শুধু registration, টাকা লাগে না) অথবা **Premium** (টাকা দিয়ে টিকিট) হতে পারে।
>
> Stack: Node.js, Express 5, TypeScript, MongoDB + Mongoose, Zod, Redis, SSLCommerz, Cloudinary, Nodemailer — Phovia-র মতোই।
> Phovia repo: https://github.com/arifucoder/phovia-tour-server

---

## ০. SDLC — আমরা কোথায় আছি

**SDLC** (Software Development Life Cycle) হলো একটা software বানানোর ধাপগুলো। Code লেখা শুরু হয় চার নম্বর ধাপে — তার আগে **কী** বানাব আর **কীভাবে** বানাব, সেটা পরিষ্কার করা।

| # | ধাপ | প্রশ্ন | এই document-এ? |
|---|---|---|---|
| 1 | **Planning** | কী বানাব, কেন, কারা ব্যবহার করবে? | ✅ Part 1 |
| 2 | **Requirement Analysis** | কী কী feature, কী কী নিয়ম? | ✅ Part 2 |
| 3 | **System Design** | Database, relation, API, flow কেমন হবে? | ✅ Part 3–6 |
| 4 | Implementation | Code লেখা | Part 8-এর roadmap অনুযায়ী |
| 5 | Testing | ঠিকমতো কাজ করছে কিনা | Part 7-এ পরিকল্পনা |
| 6 | Deployment | Live করা | Phovia-র note 38 |
| 7 | Maintenance | Bug fix, নতুন feature | — |

> 💡 এই document আগে পুরোটা পড়ে বুঝে, তারপর code শুরু করো। মাঝপথে কোনো সিদ্ধান্ত বদলালে এখানেও update করো — এটাই তোমার project-এর "নকশা"।

---

# Part 1: Planning

## 1.1 Problem Statement

ছোট-বড় event (কনসার্ট, tech workshop, seminar, খেলা) আয়োজকরা এখন Facebook post আর bKash নম্বর দিয়ে টিকিট বিক্রি করে, Excel-এ হিসাব রাখে, gate-এ কাগজের list মিলিয়ে মানুষ ঢোকায়। এতে:
- কত টিকিট বিক্রি হলো, ঠিক হিসাব থাকে না — seat-এর চেয়ে বেশি বিক্রি হয়ে যায়
- নকল টিকিট দিয়ে ঢোকা যায়
- টাকা আসল কিনা যাচাই কঠিন

**Eventix:** অনলাইনে event খোঁজা, টিকিট কেনা, অনলাইন payment, QR টিকিট, আর gate-এ scan করে ঢোকা — সব এক জায়গায়।

## 1.2 Roles (কারা ব্যবহার করবে)

| Role | কে | কী করতে পারে |
|---|---|---|
| **USER** | সাধারণ মানুষ | Event দেখা, টিকিট কেনা, নিজের টিকিট দেখা, review দেওয়া, organizer হওয়ার আবেদন |
| **ORGANIZER** | Event আয়োজক | নিজের event আর ticket type বানানো, নিজের event-এর booking/attendee দেখা, QR scan করে check-in |
| **ADMIN** | Platform-এর কর্মী | Category/venue manage, organizer আবেদন approve/reject, সব event/booking/payment দেখা, user block |
| **SUPER_ADMIN** | Platform-এর মালিক | ADMIN-এর সব + admin বানানো; কেউ তাকে বদলাতে পারে না |

> Phovia-র `GUIDE` role-এর জায়গায় এখানে `ORGANIZER` — তবে এবার সে নিজের data নিজে manage করবে, তাই "কে কার data ছুঁতে পারবে" (ownership) নিয়মগুলো আরও বেশি।

## 1.3 Phovia থেকে কী আসছে, কী নতুন

| আবার করব (পাকা করা) | নতুন শিখব |
|---|---|
| Auth (JWT, cookie, Google, local), OTP, forgot/reset | **Organizer approval workflow** |
| Modular pattern, Zod, error handler, QueryBuilder | **Ticket Type** (একই event-এ General/VIP আলাদা দাম ও সংখ্যা) |
| Slug, Cloudinary upload | **Seat race condition** — atomic update দিয়ে oversell আটকানো |
| SSLCommerz + IPN + transaction | **Cron job** — ১৫ মিনিটে pay না করলে booking বাতিল, seat ফেরত |
| PDF invoice, Nodemailer, Redis | **QR code টিকিট + check-in** |
| Stats aggregation, deploy | **Review & rating**, organizer-এর নিজের dashboard |
| | **Event mode (Online / Offline / Hybrid)** — mode অনুযায়ী আলাদা validation (Zod `superRefine`), গোপন meeting link |
| | **Free / Premium** — একই booking API-তে দুই রকম flow (payment সহ ও ছাড়া), free টিকিট জমিয়ে রাখা আটকানো |
| | **Rate limiting, Testing, Swagger** |

---

# Part 2: Requirement Analysis

## 2.1 Functional Requirements (কী কী করতে পারবে)

### Auth & User
- **FR-1** Email + password দিয়ে register; register-এর পর email OTP দিয়ে verify।
- **FR-2** Email/password আর Google দিয়ে login; access + refresh token cookie-তে।
- **FR-3** Logout, change password, Google user-এর set password, forgot/reset password।
- **FR-4** নিজের profile দেখা ও update (name, phone, address, picture)।
- **FR-5** Admin সব user দেখতে পারবে, block/unblock করতে পারবে।

### Organizer
- **FR-6** USER "organizer হতে চাই" আবেদন করবে (organization-এর নাম, বিবরণ, phone)।
- **FR-7** Admin আবেদন approve করলে user-এর role `ORGANIZER` হবে; reject করলে কারণ জানাবে। দুই ক্ষেত্রেই email যাবে।

### Category & Venue
- **FR-8** Admin category (Music, Tech, Sports...) আর venue (নাম, শহর, ঠিকানা, ধারণক্ষমতা, ছবি) CRUD।

### Event
- **FR-9** Organizer event বানাবে (title, বিবরণ, ছবি, category, **event mode**, শুরু-শেষের সময়) — শুরুতে `DRAFT`। Event mode তিন রকম:
  - **OFFLINE** — শুধু venue-তে; venue লাগবে
  - **ONLINE** — শুধু অনলাইনে (Zoom, Google Meet, YouTube Live...); platform আর meeting link লাগবে, venue লাগবে না
  - **HYBRID** — দুটোই; venue আর meeting link দুটোই লাগবে
- **FR-9a** Meeting link **গোপন** থাকবে — event-এর public page-এ দেখাবে না। শুধু confirmed ticket-এর মালিক event শুরুর কিছুক্ষণ আগে থেকে দেখতে পাবে।
- **FR-10** Organizer event-এ এক বা একাধিক **ticket type** যোগ করবে (নাম, দাম, মোট সংখ্যা, প্রতি booking-এ সর্বোচ্চ কয়টা, বিক্রির শুরু-শেষ)।
- **FR-11** অন্তত একটা ticket type থাকলে organizer event `PUBLISHED` করতে পারবে — তখনই সবাই দেখতে পাবে।
- **FR-12** Public event list: search (title, বিবরণ), filter (category, **event mode**, venue, শহর, তারিখের range, দামের range), sort, pagination। ("শুধু অনলাইন event দেখাও" — ঢাকার বাইরের মানুষের জন্য কাজের।)
- **FR-13** Slug দিয়ে একটা event-এর বিস্তারিত (ticket type, কত বাকি, গড় rating সহ)।
- **FR-14** Organizer নিজের event cancel করতে পারবে → সব booking বাতিল, সব attendee-কে email।

### Booking & Payment
- **FR-14a** Event দুই ধরনের — organizer event বানানোর সময় বেছে নেবে:
  - **FREE** — টাকা লাগে না, শুধু registration (seminar, community meetup, free webinar)। Seat-এর সীমা তবুও থাকবে।
  - **PREMIUM** — টাকা দিয়ে টিকিট (কনসার্ট, paid workshop)। এক বা একাধিক দামের ticket type।
- **FR-15** **Premium:** User একটা ticket type থেকে N টা টিকিট book করবে → booking `PENDING`, seat **সাময়িক ধরে রাখা** হবে, SSLCommerz payment link।
- **FR-15a** **Free:** User register করলেই (payment ছাড়া) booking সাথে সাথে `CONFIRMED`, ticket তৈরি, ticket PDF (QR সহ) email-এ। কোনো payment record তৈরি হবে না।
- **FR-16** (Premium) ১৫ মিনিটে pay না করলে booking নিজে `EXPIRED`, seat ফেরত।
- **FR-17** Payment সফল (success route বা IPN) → booking `CONFIRMED`, N টা **ticket** তৈরি (প্রতিটার আলাদা QR), invoice PDF (QR সহ) email-এ।
- **FR-18** Payment fail/cancel → booking বাতিল, seat ফেরত; pending থাকা অবস্থায় আবার pay করার সুযোগ।
- **FR-19** User নিজে cancel করতে পারবে: Premium-এ শুধু `PENDING` booking (টাকা দেওয়ার পর refund-এর ঝামেলা, তাই না); Free-তে `CONFIRMED` booking-ও event শুরুর আগে পর্যন্ত — seat অন্যরা পাবে।
- **FR-20** User নিজের booking, payment, invoice দেখবে; organizer নিজের event-এর booking দেখবে; admin সব।

### Ticket & Check-in
- **FR-21** User নিজের সব ticket (QR সহ) দেখবে।
- **FR-22** **In-person:** Event-এর দিন organizer ticket code/QR scan করবে → ticket `USED`, সময় আর কে scan করেছে লেখা থাকবে। একই ticket দুবার ঢুকতে পারবে না।
- **FR-22a** **Online:** User "Join" চাপলে system ticket যাচাই করে meeting link দেবে আর ticket `USED` করবে (attendance-এর হিসাব থাকবে)। Online ticket আবার join করা যাবে (connection কেটে গেলে), কিন্তু শুধু ওই মালিকই।

### Review
- **FR-23** Event `COMPLETED` হলে, যে user-এর ওই event-এর ticket `USED` (মানে সত্যিই এসেছিল), সে একবার rating (১–৫) আর মন্তব্য দিতে পারবে।
- **FR-24** Event-এর পাশে গড় rating আর review সংখ্যা দেখাবে।

### Stats
- **FR-25** Admin dashboard: user, event, booking, payment, আয়ের stats।
- **FR-26** Organizer dashboard: নিজের event-গুলোর বিক্রি, আয়, check-in-এর হার।

## 2.2 Business Rules (নিয়ম — code-এ এগুলোই enforce হবে)

| # | নিয়ম |
|---|---|
| BR-1 | Unverified (`isVerified: false`) user booking করতে পারবে না |
| BR-2 | Booking-এর আগে profile-এ phone থাকতে হবে |
| BR-3 | শুধু `PUBLISHED` আর শুরু হয়নি এমন event-এ booking |
| BR-4 | Ticket type-এর বিক্রির সময় (`salesStart`–`salesEnd`)-এর বাইরে booking না |
| BR-5 | `quantity` ≤ `maxPerBooking` আর ≤ বাকি seat; **কখনো oversell না** |
| BR-6 | `PENDING` booking ১৫ মিনিট পর `EXPIRED`, seat ফেরত |
| BR-7 | Organizer শুধু **নিজের** event/ticket type বদলাতে, নিজের event-এ check-in করতে পারবে |
| BR-8 | `DRAFT` event মোছা যাবে; booking আছে এমন event মোছা যাবে না, শুধু cancel |
| BR-9 | কোনো টিকিট বিক্রি হওয়া ticket type-এর দাম বা মোট সংখ্যা কমানো যাবে না |
| BR-10 | Ticket শুধু `VALID` অবস্থায় check-in, তাও event-এর দিনে (শুরুর কয়েক ঘণ্টা আগে থেকে শেষ পর্যন্ত); **online ticket scan করে check-in না, join দিয়ে** |
| BR-11 | একজন user একটা event-এ একটাই review; শুধু attend করলে |
| BR-12 | Venue-এর `capacity`-র চেয়ে event-এর সব **IN_PERSON** ticket type-এর মোট সংখ্যা বেশি হবে না (online seat venue-র জায়গা নেয় না) |
| BR-12a | Mode অনুযায়ী বাধ্যতামূলক: OFFLINE → venue; ONLINE → onlineDetails; HYBRID → দুটোই। ভুল mode-এর data পাঠালে 400 |
| BR-12b | Ticket type-এর `accessType` event-এর mode-এর সাথে মিলতে হবে: OFFLINE event-এ শুধু IN_PERSON, ONLINE-এ শুধু ONLINE, HYBRID-এ দুটোই |
| BR-12c | Meeting link কোনো public API-তে যাবে না; শুধু join API দিয়ে, confirmed ONLINE ticket-এর মালিককে, শুরুর ৩০ মিনিট আগে থেকে শেষ পর্যন্ত |
| BR-12d | `PUBLISHED` event-এর mode বদলানো যাবে না (যারা in-person টিকিট কিনেছে, তাদের ক্ষতি) |
| BR-13 | ADMIN, SUPER_ADMIN-কে বদলাতে পারবে না (Phovia note 35) |
| BR-14 | PAID payment কখনো উল্টানো যাবে না; refund আলাদা প্রক্রিয়া |
| BR-15 | FREE event-এর সব ticket type-এর `price` = 0; PREMIUM event-এর সব ticket type-এর `price` > 0 (মিশ্র না) |
| BR-16 | FREE event-এ একজন user-এর **একটাই** সক্রিয় (`CONFIRMED`) booking, আর `maxPerBooking` ডিফল্ট ২ — কেউ বিনা পয়সায় সব seat নিয়ে বসে থাকতে না পারে |
| BR-17 | FREE booking-এ payment, invoice, ১৫ মিনিটের hold, expire cron — কিছুই লাগে না |
| BR-18 | একবার টিকিট বিক্রি/registration হয়ে গেলে event-এর `pricingType` বদলানো যাবে না |

## 2.3 Non-Functional Requirements

| বিষয় | চাহিদা |
|---|---|
| Security | Password hash, httpOnly cookie, role + ownership check, payment validation, rate limit (login, OTP, booking) |
| Data integrity | Booking + seat + payment transaction-এ; oversell অসম্ভব |
| Validation | সব input Zod দিয়ে; সুন্দর error format (Phovia note 23) |
| Performance | List-এ pagination, দরকারি field-এ index, stats Redis-এ cache |
| Reliability | Invoice/email fail হলেও payment ঠিক থাকবে; IPN দিয়ে ব্যাকআপ |
| Maintainability | Modular structure, ESLint strict, TypeScript strict |
| Testing | Auth, booking, payment, check-in-এর API test |
| Documentation | Swagger-এ সব API |

## 2.4 Out of Scope (v1-এ করব না)

Seat map (নির্দিষ্ট আসন বাছাই), organizer-কে টাকা পাঠানো (payout), automatic refund, coupon/discount, multi-language, push notification, chat। পরে v2-তে ভাবা যাবে।

---

# Part 3: Data Model আর Relation

## 3.1 Entity Relationship (এক নজরে)

```
                         ┌──────────────┐
                         │   Category   │
                         └──────┬───────┘
                                │ 1
                                │
                                │ N
┌──────────┐ 1          N ┌─────┴───────┐ N          1 ┌──────────┐
│   User   │──────────────│    Event    │──────────────│  Venue   │
│(ORGANIZER)│  organizes  └─────┬───────┘              └──────────┘
└────┬─────┘                    │ 1
     │                          │
     │ 1                        │ N
     │                   ┌──────┴───────┐
     │                   │  TicketType  │  (General, VIP...)
     │                   └──────┬───────┘
     │                          │ 1
     │ 1                        │
     │           N       ┌──────┴───────┐ 1        1 ┌──────────┐
     ├───────────────────│   Booking    │────────────│ Payment  │
     │   books           └──────┬───────┘            └──────────┘
     │                          │ 1
     │                          │
     │                          │ N
     │ 1          N      ┌──────┴───────┐
     ├───────────────────│    Ticket    │  (প্রতিটা আলাদা QR)
     │   owns            └──────────────┘
     │
     │ 1          N      ┌──────────────┐ N        1
     ├───────────────────│    Review    │──────────── Event
     │   writes          └──────────────┘
     │
     │ 1          N      ┌──────────────────┐
     └───────────────────│ OrganizerRequest │
         applies          └──────────────────┘
```

> Event → Venue-এর রেখাটা **ঐচ্ছিক**: OFFLINE আর HYBRID event-এর venue থাকে, ONLINE event-এর থাকে না। ONLINE event-এর meeting link থাকে event-এর ভেতরেই (`onlineDetails`), আলাদা collection লাগে না।

## 3.2 Relation Table

| Relation | ধরন | কোথায় reference রাখব | কেন |
|---|---|---|---|
| User → Event (organizer) | 1 : N | `Event.organizer` | একজন organizer অনেক event বানায় |
| Category → Event | 1 : N | `Event.category` | |
| Venue → Event | **0..1** : N | `Event.venue` (**optional**) | একই venue-তে অনেক event; ONLINE event-এর venue থাকে না |
| Event → TicketType | 1 : N | `TicketType.event` | আলাদা collection — seat কমাতে atomic update লাগবে (3.4) |
| User → Booking | 1 : N | `Booking.user` | |
| Event → Booking | 1 : N | `Booking.event` | Organizer-এর "আমার event-এর booking" দ্রুত খুঁজতে |
| TicketType → Booking | 1 : N | `Booking.ticketType` | একটা booking = একটা ticket type থেকে N টা |
| Booking ↔ Payment | 1 : 1 | দুই দিকেই (`Booking.payment`, `Payment.booking` unique) | Phovia-র মতো two-way |
| Booking → Ticket | 1 : N | `Ticket.booking` | ৩টা কিনলে ৩টা ticket, প্রতিটা আলাদা QR |
| User → Ticket | 1 : N | `Ticket.user` | "আমার টিকিট" দ্রুত |
| Event → Ticket | 1 : N | `Ticket.event` | Check-in-এর সময় "এই ticket কি এই event-এর?" |
| User → Review / Event → Review | 1 : N / 1 : N | `Review.user`, `Review.event` (জোড়া unique) | |
| User → OrganizerRequest | 1 : N | `OrganizerRequest.user` | Reject হলে আবার আবেদন করা যায় |

### Reference নাকি Embed?

| Data | সিদ্ধান্ত | কারণ |
|---|---|---|
| Ticket type | **আলাদা collection** (reference) | প্রতিটার seat আলাদাভাবে atomic কমাতে হবে; event document বারবার বদলানো ঠিক না |
| Event images | Embed (`string[]`) | শুধু URL-এর list, আলাদা পরিচয় নেই |
| User auths | Embed | Phovia-র মতো |
| Booking-এ দাম | **কপি করে রাখা** (`unitPrice`, `totalAmount`) | পরে organizer দাম বদলালেও পুরোনো booking-এর হিসাব বদলাবে না |

## 3.3 Collection-গুলোর Field

### User

| Field | Type | নোট |
|---|---|---|
| name | string | required |
| email | string | required, unique |
| password | string? | Google user-এর থাকে না |
| phone | string? | Booking-এর আগে লাগবে |
| address | string? | |
| picture | string? | Cloudinary URL |
| role | `SUPER_ADMIN \| ADMIN \| ORGANIZER \| USER` | default `USER` |
| isActive | `ACTIVE \| INACTIVE \| BLOCKED` | default `ACTIVE` |
| isDeleted | boolean | default false |
| isVerified | boolean | default false (Google login-এ true) |
| auths | `{ provider: "google" \| "credentials", providerId }[]` | |
| createdAt, updatedAt | Date | timestamps |

### OrganizerRequest

| Field | Type | নোট |
|---|---|---|
| user | ObjectId → User | required |
| organizationName | string | required |
| description | string | |
| phone | string | required |
| status | `PENDING \| APPROVED \| REJECTED` | default `PENDING` |
| rejectionReason | string? | Reject হলে |
| reviewedBy | ObjectId → User? | কোন admin |
| reviewedAt | Date? | |

> একজন user-এর একসাথে একটাই `PENDING` আবেদন থাকতে পারবে।

### Category

| Field | Type | নোট |
|---|---|---|
| name | string | required, unique |
| slug | string | unique, auto (Phovia-র `generateUniqueSlug`) |
| description | string? | |
| icon | string? | Cloudinary URL |

### Venue

| Field | Type | নোট |
|---|---|---|
| name | string | required |
| slug | string | unique, auto |
| city | string | required — filter-এ লাগবে |
| address | string | required |
| capacity | number | required, > 0 |
| thumbnail | string? | |
| location | `{ type: "Point", coordinates: [lng, lat] }`? | ঐচ্ছিক — পরে "কাছাকাছি event" (2dsphere index) |

### Event

| Field | Type | নোট |
|---|---|---|
| title | string | required |
| slug | string | unique, auto |
| description | string | |
| images | string[] | Cloudinary |
| category | ObjectId → Category | required |
| eventMode | `OFFLINE \| ONLINE \| HYBRID` | required |
| pricingType | `FREE \| PREMIUM` | required — booking flow এটা দেখেই ঠিক হয় |
| venue | ObjectId → Venue? | OFFLINE / HYBRID হলে required, ONLINE হলে থাকবে না |
| onlineDetails | `{ platform, meetingLink, meetingPassword? }`? | ONLINE / HYBRID হলে required; schema-তে `select: false` — কোনো query-তে নিজে থেকে আসবে না |
| organizer | ObjectId → User | required — token থেকে, body থেকে না |
| startDateTime | Date | required, ভবিষ্যতে |
| endDateTime | Date | required, start-এর পরে |
| status | `DRAFT \| PUBLISHED \| CANCELLED \| COMPLETED` | default `DRAFT` |
| minPrice | number | ticket type থেকে হিসাব — "৳৫০০ থেকে শুরু", দাম দিয়ে filter |
| averageRating | number | default 0 — review থেকে হিসাব |
| reviewCount | number | default 0 |
| cancelReason | string? | |

> **`onlineDetails`-এ `select: false` কেন?** Password-এর মতোই — event list, event detail কোথাও ভুল করেও meeting link যেন না যায়। শুধু join API-তে `.select("+onlineDetails")` লিখে ইচ্ছা করে আনব। `platform` ("Zoom", "Google Meet") দেখানো যায় — চাইলে `platform` আলাদা field-এ রেখে শুধু `meetingLink`/`meetingPassword` লুকাও।

> `minPrice`, `averageRating`, `reviewCount` — হিসাব করে রাখা field (denormalized)। প্রতিবার list দেখানোর সময় aggregation না চালিয়ে আগেই রেখে দিই; ticket type বা review বদলালে update করি।

### TicketType

| Field | Type | নোট |
|---|---|---|
| event | ObjectId → Event | required |
| name | string | "General", "VIP", "Early Bird" — একই event-এ unique |
| price | number | FREE event-এ `0`, PREMIUM-এ `> 0` (BR-15) |
| totalQuantity | number | > 0 |
| availableQuantity | number | শুরুতে = totalQuantity; booking-এ কমে, expire/cancel-এ বাড়ে |
| accessType | `IN_PERSON \| ONLINE` | ONLINE event-এ ONLINE, OFFLINE-এ IN_PERSON, HYBRID-এ organizer বেছে নেবে (যেমন "Venue General ৳৫০০", "Online Pass ৳২০০") |
| maxPerBooking | number | default 5 |
| salesStart, salesEnd | Date | বিক্রির সময় |

### Booking

| Field | Type | নোট |
|---|---|---|
| user | ObjectId → User | token থেকে |
| event | ObjectId → Event | |
| ticketType | ObjectId → TicketType | |
| quantity | number | ≥ 1 |
| unitPrice | number | Booking-এর সময়ের দাম (কপি) |
| totalAmount | number | unitPrice × quantity |
| status | `PENDING \| CONFIRMED \| CANCELLED \| EXPIRED \| FAILED` | default `PENDING` |
| isFree | boolean | event-এর pricingType থেকে কপি — query আর stats সহজ |
| payment | ObjectId → Payment? | শুধু premium-এ; free-তে থাকে না |
| expiresAt | Date? | শুধু premium-এ: createdAt + ১৫ মিনিট |

### Payment

| Field | Type | নোট |
|---|---|---|
| booking | ObjectId → Booking | required, **unique** |
| transactionId | string | unique, `randomUUID` |
| amount | number | |
| status | `UNPAID \| PAID \| FAILED \| CANCELLED \| REFUNDED` | default `UNPAID` |
| paymentGatewayData | Mixed? | SSLCommerz validation response |
| invoiceUrl | string? | |
| paidAt | Date? | |

### Ticket

| Field | Type | নোট |
|---|---|---|
| booking | ObjectId → Booking | |
| event | ObjectId → Event | |
| ticketType | ObjectId → TicketType | |
| user | ObjectId → User | মালিক |
| ticketCode | string | unique, অনুমান-অযোগ্য (`EVX-` + random) — এটাই QR-এর ভেতরে |
| status | `VALID \| USED \| CANCELLED` | default `VALID` |
| checkedInAt | Date? | |
| checkedInBy | ObjectId → User? | কোন organizer scan করেছে (online join-এ নিজে) |
| accessType | `IN_PERSON \| ONLINE` | Ticket type থেকে কপি — check-in নাকি join, সেটা বুঝতে |

### Review

| Field | Type | নোট |
|---|---|---|
| user | ObjectId → User | |
| event | ObjectId → Event | |
| rating | number | ১–৫, পূর্ণসংখ্যা |
| comment | string? | |

> Index: `{ user: 1, event: 1 }` unique — একজন একটাই review।

## 3.4 Status-এর পরিবর্তন (State Machine)

কোন status থেকে কোন status-এ যাওয়া যায় — এর বাইরে কিছু হলে service error দেবে।

**Event**
```
DRAFT ──publish──→ PUBLISHED ──সময় শেষ (cron)──→ COMPLETED
  │                    │
  └──delete            └──cancel──→ CANCELLED
```

**Booking**
```
PENDING ──payment PAID──→ CONFIRMED
   │──payment fail─────→ FAILED      (seat ফেরত)
   │──user cancel──────→ CANCELLED   (seat ফেরত)
   └──১৫ মিনিট (cron)──→ EXPIRED     (seat ফেরত)

CONFIRMED ──event cancel──→ CANCELLED (payment → refund-এর জন্য চিহ্নিত)
```

**Booking (Free)** — PENDING ধাপটাই নেই:
```
(register) ──→ CONFIRMED ──user cancel (শুরুর আগে)──→ CANCELLED (seat ফেরত, ticket CANCELLED)
                   └──event cancel──→ CANCELLED
```

**Ticket**
```
VALID ──check-in──→ USED
  └──event/booking cancel──→ CANCELLED
```

**Payment**: Phovia-র মতো — `UNPAID → PAID / FAILED / CANCELLED`, `PAID → REFUNDED` (শুধু admin)।

## 3.5 Index Plan

| Collection | Index | কেন |
|---|---|---|
| Event | `slug` (unique), `{ status, startDateTime }`, `category`, `venue`, `organizer`, text: `title, description` | List, filter, search |
| TicketType | `{ event, name }` unique | |
| Booking | `{ user, createdAt }`, `{ event, status }`, `{ status, expiresAt }` | My bookings, organizer view, cron |
| Payment | `transactionId` unique, `booking` unique | |
| Ticket | `ticketCode` unique, `{ user }`, `{ event, status }` | Check-in, my tickets, attendee list |
| Review | `{ user, event }` unique, `{ event }` | |

---

# Part 4: API Design

**Base URL:** `/api/v1`

**Access-এর চিহ্ন:** 🌐 Public · 👤 Login করা যেকোনো role · 🎤 ORGANIZER · 🛡️ ADMIN + SUPER_ADMIN · ⭐ শুধু SUPER_ADMIN · 🔒 মালিক (ownership check)

## 4.1 Auth — `/auth`

| Method | Route | Access | কাজ |
|---|---|---|---|
| POST | `/auth/login` | 🌐 | Email + password (passport-local) |
| POST | `/auth/refresh-token` | 🌐 (refresh cookie) | নতুন access token |
| POST | `/auth/logout` | 👤 | Cookie মোছা |
| POST | `/auth/change-password` | 👤 | পুরোনো password দিয়ে বদল |
| POST | `/auth/set-password` | 👤 | Google user-এর প্রথম password |
| POST | `/auth/forgot-password` | 🌐 | Reset link email (আলাদা secret) |
| POST | `/auth/reset-password` | 🌐 (reset token) | নতুন password |
| GET | `/auth/google` | 🌐 | Google login শুরু |
| GET | `/auth/google/callback` | 🌐 | Google ফেরত পাঠায় |

## 4.2 OTP — `/otp`

| Method | Route | Access | কাজ |
|---|---|---|---|
| POST | `/otp/send` | 🌐 | Email verify-এর OTP (Redis, cooldown) |
| POST | `/otp/verify` | 🌐 | OTP মেলানো (attempt limit) |

## 4.3 User — `/user`

| Method | Route | Access | কাজ |
|---|---|---|---|
| POST | `/user/register` | 🌐 | নতুন account |
| GET | `/user/me` | 👤 | নিজের profile |
| PATCH | `/user/me` | 👤 | নিজের profile update (multipart: `file` = picture) |
| GET | `/user` | 🛡️ | সব user (QueryBuilder: search, role, isActive) |
| GET | `/user/:id` | 🛡️ | একজন user |
| PATCH | `/user/:id` | 🛡️ | Role / isActive / isDeleted (ADMIN → SUPER_ADMIN না) |

> 💡 Phovia-তে নিজের update আর admin-এর update একই `PATCH /:id`-এ ছিল, ভেতরে অনেক if-else। এখানে আলাদা: `/me` (নিজের, শুধু profile field) আর `/:id` (admin, শুধু status/role)। প্রতিটার Zod schema আলাদা — code অনেক পরিষ্কার।

## 4.4 Organizer Request — `/organizer-requests`

| Method | Route | Access | কাজ |
|---|---|---|---|
| POST | `/organizer-requests` | 👤 USER | আবেদন (একটা PENDING থাকলে আর না) |
| GET | `/organizer-requests/me` | 👤 | নিজের আবেদনের অবস্থা |
| GET | `/organizer-requests` | 🛡️ | সব আবেদন (filter: status) |
| PATCH | `/organizer-requests/:id/approve` | 🛡️ | Approve → user.role = ORGANIZER (transaction), email |
| PATCH | `/organizer-requests/:id/reject` | 🛡️ | Reject + কারণ, email |

## 4.5 Category — `/categories`

| Method | Route | Access | কাজ |
|---|---|---|---|
| POST | `/categories` | 🛡️ | নতুন (multipart: `file` = icon) |
| GET | `/categories` | 🌐 | সব |
| GET | `/categories/:slug` | 🌐 | একটা |
| PATCH | `/categories/:id` | 🛡️ | Update |
| DELETE | `/categories/:id` | 🛡️ | মোছা — কোনো event থাকলে না |

## 4.6 Venue — `/venues`

| Method | Route | Access | কাজ |
|---|---|---|---|
| POST | `/venues` | 🛡️ | নতুন (multipart: `file`) |
| GET | `/venues` | 🌐 | সব (filter: city, search) |
| GET | `/venues/:slug` | 🌐 | একটা |
| PATCH | `/venues/:id` | 🛡️ | Update |
| DELETE | `/venues/:id` | 🛡️ | মোছা — event থাকলে না |

## 4.7 Event — `/events`

| Method | Route | Access | কাজ |
|---|---|---|---|
| POST | `/events` | 🎤 | নতুন event, `DRAFT` (multipart: `files`) — `eventMode` অনুযায়ী venue / onlineDetails |
| GET | `/events` | 🌐 | শুধু `PUBLISHED` — search, filter (category, `eventMode`, `pricingType`, venue, city, `startFrom`, `startTo`, `minPrice`, `maxPrice`), sort, pagination — **meeting link কখনো না** |
| GET | `/events/my-events` | 🎤 | নিজের সব event (সব status) |
| GET | `/events/admin` | 🛡️ | সব event, সব status |
| GET | `/events/:slug` | 🌐 | বিস্তারিত — ticket type, বাকি seat, rating |
| PATCH | `/events/:id` | 🎤🔒 / 🛡️ | Update (images-এ `deleteImages`, Phovia note 29) |
| PATCH | `/events/:id/publish` | 🎤🔒 | `DRAFT → PUBLISHED` (ticket type লাগবে) |
| PATCH | `/events/:id/cancel` | 🎤🔒 / 🛡️ | `→ CANCELLED`, সব booking/ticket বাতিল, email |
| DELETE | `/events/:id` | 🎤🔒 / 🛡️ | শুধু `DRAFT` (Cloudinary image-ও মোছা) |

> Route order: `/my-events`, `/admin` অবশ্যই `/:slug`-এর **আগে** (Phovia note 26)।

## 4.8 Ticket Type — `/events/:eventId/ticket-types` ও `/ticket-types`

| Method | Route | Access | কাজ |
|---|---|---|---|
| POST | `/events/:eventId/ticket-types` | 🎤🔒 | নতুন ticket type (venue capacity check) |
| GET | `/events/:eventId/ticket-types` | 🌐 | ওই event-এর সব |
| PATCH | `/ticket-types/:id` | 🎤🔒 | Update (বিক্রি শুরু হলে দাম/সংখ্যা কমানো না) |
| DELETE | `/ticket-types/:id` | 🎤🔒 | একটাও বিক্রি না হলে |

## 4.9 Booking — `/bookings`

| Method | Route | Access | কাজ |
|---|---|---|---|
| POST | `/bookings` | 👤 | `{ ticketType, quantity }` → **Premium:** seat ধরে রাখা + payment link · **Free:** সাথে সাথে CONFIRMED + ticket |
| GET | `/bookings/my-bookings` | 👤 | নিজের |
| GET | `/bookings/:id` | 👤🔒 / 🎤 (নিজের event) / 🛡️ | একটা |
| PATCH | `/bookings/:id/cancel` | 👤🔒 | Premium: শুধু `PENDING`; Free: `CONFIRMED`-ও (শুরুর আগে) — seat ফেরত |
| GET | `/events/:eventId/bookings` | 🎤🔒 / 🛡️ | Organizer-এর event-এর booking |
| GET | `/bookings` | 🛡️ | সব (filter: status, event, তারিখ) |

## 4.10 Payment — `/payments`

| Method | Route | Access | কাজ |
|---|---|---|---|
| POST | `/payments/init/:bookingId` | 👤🔒 | `PENDING` booking-এর জন্য নতুন link (মেয়াদ থাকলে) |
| POST | `/payments/success` | 🌐 (SSLCommerz) | Validate → CONFIRMED → ticket তৈরি → invoice → redirect |
| POST | `/payments/fail` | 🌐 (SSLCommerz) | FAILED, seat ফেরত → redirect |
| POST | `/payments/cancel` | 🌐 (SSLCommerz) | CANCELLED, seat ফেরত → redirect |
| POST | `/payments/ipn` | 🌐 (SSLCommerz server) | IPN — success-এর মতোই, idempotent |
| GET | `/payments/:id/invoice` | 👤🔒 / 🛡️ | Invoice URL |
| GET | `/payments` | 🛡️ | সব payment |
| PATCH | `/payments/:id/refund` | 🛡️ | `PAID → REFUNDED` (হাতে টাকা ফেরতের পর চিহ্নিত করা) |

## 4.11 Ticket — `/tickets`

| Method | Route | Access | কাজ |
|---|---|---|---|
| GET | `/tickets/my-tickets` | 👤 | নিজের সব ticket (event সহ) |
| GET | `/tickets/:ticketCode` | 👤🔒 | একটা ticket + QR image (data URL) |
| POST | `/tickets/check-in` | 🎤 (নিজের event) / 🛡️ | **In-person:** `{ ticketCode }` → VALID হলে USED |
| POST | `/tickets/:ticketCode/join` | 👤🔒 | **Online:** সময় ঠিক থাকলে meeting link দেয়, প্রথমবার USED করে |
| GET | `/events/:eventId/attendees` | 🎤🔒 / 🛡️ | কে কে ticket কিনেছে, কে ঢুকেছে |

## 4.12 Review — `/reviews`

| Method | Route | Access | কাজ |
|---|---|---|---|
| POST | `/reviews` | 👤 | `{ event, rating, comment }` — attend করলে, একবার |
| GET | `/events/:eventId/reviews` | 🌐 | Event-এর সব review (pagination) |
| PATCH | `/reviews/:id` | 👤🔒 | নিজের review বদলানো |
| DELETE | `/reviews/:id` | 👤🔒 / 🛡️ | মোছা |

> Review যোগ/বদল/মোছা হলেই `Event.averageRating` আর `reviewCount` aggregation দিয়ে আবার হিসাব।

## 4.13 Stats — `/stats`

| Method | Route | Access | কাজ |
|---|---|---|---|
| GET | `/stats/overview` | 🛡️ | মোট user, event, booking, আয় — এক নজরে |
| GET | `/stats/users` | 🛡️ | Role, status, শেষ ৭/৩০ দিনে নতুন |
| GET | `/stats/events` | 🛡️ | Category/শহর/**event mode** অনুযায়ী, সবচেয়ে বেশি বিক্রি |
| GET | `/stats/bookings` | 🛡️ | Status অনুযায়ী, দিন অনুযায়ী trend |
| GET | `/stats/payments` | 🛡️ | মোট আয়, মাস অনুযায়ী আয় (শুধু premium; free registration আলাদা গোনা) |
| GET | `/stats/organizer` | 🎤 | শুধু নিজের: কত বিক্রি, আয়, check-in-এর হার, event অনুযায়ী |

**মোট: ৭৪টা endpoint।** দেখে ভয় পেও না — Part 8-এর roadmap ধরে ধাপে ধাপে হবে, আর অর্ধেকের বেশি তোমার Phovia-তে আগে বানানো জিনিসের মতোই।

---

# Part 5: গুরুত্বপূর্ণ Flow

## 5.1 Booking — Oversell ছাড়া (সবচেয়ে জরুরি)

**সমস্যা:** VIP-এর শেষ ২টা seat বাকি। একই মুহূর্তে ৩ জন ২টা করে book করল। সাধারণভাবে লিখলে:

```
৩ জনই পড়ল: availableQuantity = 2   → ৩ জনই ভাবল "২টা আছে, ঠিক আছে"
৩ জনই কমাল                           → ৬টা বিক্রি, অথচ seat ছিল ২টা ❌
```

**সমাধান:** check আর কমানো **একটা** atomic database operation-এ:

```ts
const updated = await TicketType.findOneAndUpdate(
	{ _id: ticketTypeId, availableQuantity: { $gte: quantity } }, // যথেষ্ট আছে কিনা
	{ $inc: { availableQuantity: -quantity } },                    // সাথে সাথে কমাও
	{ new: true, session },
);
if (!updated) throw new AppError(409, "Not enough tickets available.");
```

MongoDB একটা document-এ একসাথে একজনকেই update করতে দেয়। প্রথমজন কমিয়ে `0` করার পর বাকি দুজনের শর্ত (`$gte: 2`) আর মেলে না → `null` → "Not enough tickets"। ✅

**পুরো flow:**
```
POST /bookings { ticketType, quantity }
  ├─ user verified + phone আছে? (BR-1, BR-2)
  ├─ event PUBLISHED, শুরু হয়নি? sales window-এর ভেতরে? quantity ≤ maxPerBooking?
  ├─ transaction শুরু
  │    ├─ seat atomic কমাও (উপরের code)
  │    ├─ Booking PENDING (unitPrice কপি, expiresAt = এখন + ১৫ মিনিট)
  │    ├─ Payment UNPAID
  │    └─ SSLCommerz init → paymentUrl
  ├─ commit
  └─ { paymentUrl, booking }
```

## 5.1a Free Event-এ Booking (Registration)

একই `POST /bookings` — service event-এর `pricingType` দেখে পথ বেছে নেয়:

```
POST /bookings { ticketType, quantity }
  ├─ একই check: verified, event PUBLISHED, sales window, quantity ≤ maxPerBooking
  ├─ pricingType?
  │    ├─ PREMIUM → উপরের 5.1-এর flow (PENDING + payment)
  │    └─ FREE ↓
  ├─ এই event-এ এই user-এর CONFIRMED booking আগেই আছে?  হ্যাঁ → 409 "Already registered" (BR-16)
  ├─ transaction
  │    ├─ seat atomic কমাও (5.1-এর একই code — free হলেও seat সীমিত!)
  │    ├─ Booking CONFIRMED (unitPrice 0, totalAmount 0, isFree true, payment নেই)
  │    └─ quantity টা Ticket তৈরি
  ├─ commit
  ├─ commit-এর পরে: ticket PDF (QR সহ) + "Registration confirmed" email
  └─ { booking, tickets }
```

```ts
// booking.service.ts — মূল কাঠামো
const createBooking = async (payload: TCreateBookingPayload, userId: string) => {
	const ticketType = await TicketType.findById(payload.ticketType).populate<{ event: IEvent }>("event");
	// ... সাধারণ check

	return ticketType.event.pricingType === PRICING_TYPE.FREE
		? createFreeBooking(ticketType, payload.quantity, userId)
		: createPremiumBooking(ticketType, payload.quantity, userId);
};
```

> 💡 Free আর premium আলাদা function-এ, কিন্তু seat কমানো, ticket তৈরি, PDF বানানো — এই অংশগুলো দুজনেই একই helper (`reserveSeats`, `issueTickets`, `sendTicketsEmail`) ব্যবহার করবে। Premium-এ `issueTickets` চলে payment সফলের পর, free-তে সাথে সাথে।

> **Free হলেও seat-এর সীমা কেন?** Venue-তে ১০০টা চেয়ার থাকলে ১০০ জনই ঢুকতে পারবে, টাকা লাগুক বা না লাগুক। Online-এও Zoom-এর মতো platform-এ সর্বোচ্চ সংখ্যা থাকে।

## 5.2 Expire — Cron Job (শুধু Premium)

```
প্রতি মিনিটে (node-cron):
  Booking খোঁজো: status PENDING আর expiresAt < এখন
  প্রতিটার জন্য transaction-এ:
     ├─ Booking → EXPIRED   (শর্ত: এখনো PENDING — এর মধ্যে pay হয়ে গেলে কিছু করো না)
     ├─ Payment → CANCELLED
     └─ TicketType.availableQuantity += quantity
```

> ⚠️ Race: ঠিক ১৫ মিনিটের মাথায় user pay করল আর cron-ও চলল। তাই দুই জায়গাতেই update-এর শর্তে status রাখতে হবে (`status: PENDING`) — যে আগে পৌঁছাবে, সে জিতবে। Payment validate হলো কিন্তু booking ততক্ষণে EXPIRED হলে? → seat আবার পাওয়া গেলে confirm, না পেলে payment refund-এর জন্য চিহ্নিত। এটা নিজে ভেবে বের করার মতো ভালো চ্যালেঞ্জ।

## 5.3 Payment সফল → Ticket

```
success route বা IPN (Phovia note 28, 37)
  ├─ SSLCommerz validate (status + tran_id + amount)
  ├─ আগেই PAID? → কিছু করো না (idempotent)
  ├─ transaction
  │    ├─ Payment PAID, paidAt
  │    ├─ Booking CONFIRMED (শর্ত: PENDING)
  │    └─ quantity টা Ticket তৈরি (প্রতিটার আলাদা ticketCode)
  ├─ commit
  └─ commit-এর পরে (fail হলেও payment ঠিক থাকবে — note 34):
       ├─ প্রতিটা ticket-এর QR (qrcode package)
       ├─ Invoice PDF — প্রতিটা ticket-এর QR সহ
       └─ Email: PDF attachment
```

## 5.4 Check-in

```
Organizer-এর phone-এ scanner → QR থেকে ticketCode পড়ে
POST /tickets/check-in { ticketCode }
  ├─ ticket আছে?                        না → 404 "Invalid ticket"
  ├─ এই event-এর organizer কি সে?        না → 403
  ├─ event-এর দিন কিনা?                   না → 400
  ├─ atomic: { ticketCode, status: VALID } → USED, checkedInAt, checkedInBy
  │      পায়নি → আগেই USED → 409 "Already checked in at 7:42 PM"
  └─ 200 { attendee-এর নাম, ticket type } → scanner-এ সবুজ ✅
```

> একই atomic কৌশল: দুটো gate-এ একই ticket একসাথে scan করলেও একজনই ঢুকবে। ONLINE ticket এখানে scan করলে → 400 "This is an online ticket"।

## 5.4a Online Event-এ Join

```
User "My Tickets" → online ticket-এ "Join" চাপল
POST /tickets/:ticketCode/join
  ├─ ticket আছে, আর সে-ই মালিক?                 না → 404 / 403
  ├─ accessType ONLINE?                          না → 400 "Use the QR at the venue"
  ├─ ticket VALID বা USED? (CANCELLED না)         না → 403
  ├─ সময়: শুরুর ৩০ মিনিট আগে থেকে শেষ পর্যন্ত?    না → 400 "Join opens at 7:30 PM"
  ├─ Event.findById(...).select("+onlineDetails")   ← শুধু এখানেই link আনা হয়
  ├─ প্রথমবার হলে: VALID → USED, checkedInAt (attendance)
  └─ 200 { platform, meetingLink, meetingPassword }
```

> **কেন link সরাসরি email-এ না?** Email forward করলেই যে কেউ ঢুকে যেত, আর একটা link-এ একশো মানুষ। Join API-তে প্রতিবার login + ticket-এর মালিকানা যাচাই হয়, আর কে কখন join করল তার হিসাব থাকে। (পরে চাইলে প্রতি user-এর আলাদা link — Zoom registrant API — v2-তে।)

## 5.4b Event Mode-এর Validation (Zod)

তিন mode-এ তিন রকম field বাধ্যতামূলক — এটা Zod-এর `superRefine` দিয়ে এক জায়গায়:

```ts
const onlineDetailsSchema = z.object({
	platform: z.string().min(1),
	meetingLink: z.url({ error: "Meeting link must be a valid URL" }),
	meetingPassword: z.string().optional(),
});

export const createEventZodSchema = z
	.object({
		title: z.string().min(1),
		eventMode: z.enum(EVENT_MODE),
		venue: z.string().optional(),
		onlineDetails: onlineDetailsSchema.optional(),
		// ... বাকি field
	})
	.superRefine((data, ctx) => {
		const needsVenue = data.eventMode !== EVENT_MODE.ONLINE;
		const needsOnline = data.eventMode !== EVENT_MODE.OFFLINE;

		if (needsVenue && !data.venue) {
			ctx.addIssue({ code: "custom", path: ["venue"], message: "Venue is required for offline and hybrid events" });
		}
		if (!needsVenue && data.venue) {
			ctx.addIssue({ code: "custom", path: ["venue"], message: "Online events cannot have a venue" });
		}
		if (needsOnline && !data.onlineDetails) {
			ctx.addIssue({ code: "custom", path: ["onlineDetails"], message: "Online details are required for online and hybrid events" });
		}
	});
```

| eventMode | venue | onlineDetails |
|---|---|---|
| OFFLINE | ✅ লাগবে | ❌ দেওয়া যাবে না (বা উপেক্ষা) |
| ONLINE | ❌ দেওয়া যাবে না | ✅ লাগবে |
| HYBRID | ✅ লাগবে | ✅ লাগবে |

> Service-এও একই নিয়ম আরেকবার: venue id দিলে সেটা DB-তে আছে কিনা, আর update-এর সময় mode বদলালে (শুধু DRAFT-এ) পুরোনো venue/link-এর কী হবে।

## 5.5 Organizer Approval

```
USER → POST /organizer-requests → PENDING
ADMIN → PATCH /:id/approve
   transaction: request APPROVED + user.role = ORGANIZER
   commit-এর পরে email
   ⚠️ user-এর পুরোনো access token-এ এখনো role USER — নতুন role পেতে আবার login (বা refresh-token)
```

---

# Part 6: Project Setup

## 6.1 Folder Structure

```
eventix-server/
├── src/
│   ├── server.ts
│   ├── app.ts
│   └── app/
│       ├── config/          env, passport, cloudinary, multer, redis
│       ├── errorHelpers/    AppError, handle*Error
│       ├── interfaces/      error.types, index.d.ts
│       ├── middlewares/     checkAuth, validateRequest, globalErrorHandler, notFound, rateLimiter
│       ├── jobs/            expireBookings.job.ts, completeEvents.job.ts   ← নতুন
│       ├── routes/          index.ts
│       ├── utils/           catchAsync, sendResponse, jwt, setCookie, QueryBuilder,
│       │                    slug, sendEmail, invoice, qrcode, templates/
│       └── modules/
│           ├── auth/  user/  otp/  organizerRequest/
│           ├── category/  venue/  event/  ticketType/
│           ├── booking/  payment/  sslCommerz/  ticket/
│           ├── review/  stats/
├── tests/                   ← নতুন
├── tsup.config.ts  vercel.json  eslint.config.mjs  tsconfig.json
└── .env.example             ← secret ছাড়া নমুনা, GitHub-এ যাবে
```

## 6.2 Environment Variables (`.env.example`)

```bash
PORT=5000
NODE_ENV=development
DB_URL=
FRONTEND_URL=http://localhost:5173

JWT_ACCESS_SECRET=
JWT_ACCESS_EXPIRES=1d
JWT_REFRESH_SECRET=
JWT_REFRESH_EXPIRES=30d
JWT_RESET_SECRET=
BCRYPT_SALT_ROUND=10

SUPER_ADMIN_EMAIL=
SUPER_ADMIN_PASSWORD=

GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
GOOGLE_CALLBACK_URL=http://localhost:5000/api/v1/auth/google/callback
EXPRESS_SESSION_SECRET=

CLOUDINARY_CLOUD_NAME=
CLOUDINARY_API_KEY=
CLOUDINARY_API_SECRET=

SMTP_HOST=smtp.gmail.com
SMTP_PORT=465
SMTP_USER=
SMTP_PASS=
SMTP_FROM=

REDIS_HOST=
REDIS_PORT=
REDIS_USERNAME=default
REDIS_PASSWORD=

SSL_STORE_ID=
SSL_STORE_PASS=
SSL_PAYMENT_API=https://sandbox.sslcommerz.com/gwprocess/v4/api.php
SSL_VALIDATION_API=https://sandbox.sslcommerz.com/validator/api/validationserverAPI.php
SSL_SUCCESS_BACKEND_URL=http://localhost:5000/api/v1/payments/success
SSL_FAIL_BACKEND_URL=http://localhost:5000/api/v1/payments/fail
SSL_CANCEL_BACKEND_URL=http://localhost:5000/api/v1/payments/cancel
SSL_IPN_URL=http://localhost:5000/api/v1/payments/ipn
SSL_SUCCESS_FRONTEND_URL=http://localhost:5173/payment/success
SSL_FAIL_FRONTEND_URL=http://localhost:5173/payment/fail
SSL_CANCEL_FRONTEND_URL=http://localhost:5173/payment/cancel

BOOKING_HOLD_MINUTES=15
```

## 6.3 Package

Phovia-র সব package ([note 39](https://github.com/arifucoder/ph-tour-mern-notes/blob/main/server/39-packages-and-scripts.md)), তার সাথে নতুন:

| Package | কাজ |
|---|---|
| `qrcode` + `@types/qrcode` `-D` | Ticket code থেকে QR image |
| `node-cron` | প্রতি মিনিটে expire job (পরে চাইলে BullMQ) |
| `express-rate-limit` | Login, OTP, booking-এ অতিরিক্ত request আটকানো |
| `helmet` | নিরাপত্তার HTTP header এক লাইনে |
| `swagger-ui-express` + `swagger-jsdoc` | API documentation page |
| `vitest`, `supertest` `-D` | API test |
| `mongodb-memory-server` `-D` | Test-এর জন্য আলাদা in-memory MongoDB (replica set সহ — transaction test-এর জন্য) |

---

# Part 7: Testing Plan

প্রতিটা module শেষে অন্তত এই test-গুলো:

| Module | জরুরি test case |
|---|---|
| Auth | ভুল password → 401; blocked user login → 403; refresh token কাজ করে |
| Event | অন্য organizer-এর event update → 403; ticket type ছাড়া publish → 400; DRAFT event public list-এ নেই |
| Booking | Seat-এর বেশি → 409; **১০টা booking একসাথে (`Promise.all`), seat ৫টা → ঠিক ৫টা সফল**; unverified user → 403 |
| Free booking | সাথে সাথে CONFIRMED, কোনো Payment document নেই; একই event-এ দ্বিতীয়বার → 409; free event-এ price > 0-এর ticket type → 400 |
| Expire job | ১৫ মিনিট পার → EXPIRED, seat ফেরত |
| Payment | ভুয়া val_id → PAID হয় না; একই IPN দুবার → একবারই ticket তৈরি |
| Check-in | একই ticket দুবার → দ্বিতীয়বার 409; অন্য event-এর organizer → 403; online ticket scan → 400 |
| Event mode | ONLINE-এ venue দিলে → 400; HYBRID-এ link ছাড়া → 400; public list/detail-এর response-এ `meetingLink` নেই |
| Online join | অন্যের ticket → 403; শুরুর ১ ঘণ্টা আগে → 400; সময়মতো → link আসে, ticket USED |
| Review | Attend না করে → 403; দুবার → 409 |

---

# Part 8: Development Roadmap

প্রতিটা milestone শেষে: lint + typecheck + test + Postman collection update + note লেখা।

| # | Milestone | কী বানাবে | Phovia-র কোন note কাজে লাগবে |
|---|---|---|---|
| **M1** | Foundation | Project setup, env, error handler, catchAsync, sendResponse, Zod, notFound, ESLint, tsup | 01–11, 23 |
| **M2** | Auth & User | Register, login, JWT, cookie, refresh, logout, change/set/forgot/reset password, Google, `/me`, admin user management, super admin seed | 12–22, 30–32, 35 |
| **M3** | OTP | Redis, OTP send/verify, rate limit | 33 |
| **M4** | Organizer Request | Apply, approve/reject, role বদল, email | নতুন |
| **M5** | Category & Venue | CRUD, slug, Cloudinary | 24, 26, 29 |
| **M6** | Event & Ticket Type | CRUD, ownership, publish, **event mode + `superRefine`**, `accessType`, QueryBuilder (mode/দাম/তারিখ filter), image update | 24–27, 29 |
| **M7** | Booking | **Atomic seat**, transaction, concurrency test, **free (সরাসরি confirm) বনাম premium (payment) flow** | 28 |
| **M8** | Payment | SSLCommerz init, success/fail/cancel, IPN, validate | 28, 37 |
| **M9** | Ticket & Invoice | Ticket তৈরি, QR, invoice PDF, email, my-tickets | 34 |
| **M10** | Jobs | Booking expire, event complete — cron | নতুন |
| **M11** | Check-in & Join | In-person scan API, **online join API** (গোপন link), attendee list | নতুন |
| **M12** | Review | Review CRUD, rating aggregation | 36 |
| **M13** | Stats | Admin + organizer dashboard | 36 |
| **M14** | Hardening | Helmet, rate limit সব জায়গায়, Swagger, বাকি test | নতুন |
| **M15** | Deploy | VPS (PM2 + Nginx) — cron-এর জন্য VPS ভালো, Vercel serverless-এ cron আলাদাভাবে চলে | 38 |

> 💡 **M7 আর M8 সবচেয়ে কঠিন আর সবচেয়ে বেশি শেখার।** তাড়াহুড়ো কোরো না — concurrency test (১০টা একসাথে) pass না করা পর্যন্ত পরের milestone-এ যেও না।

---

# Part 9: শুরু করার আগে Checklist

- [ ] এই document পুরোটা পড়েছি, প্রতিটা relation বুঝেছি
- [ ] কাগজে বা [dbdiagram.io](https://dbdiagram.io)-তে ERD নিজে একবার এঁকেছি
- [ ] GitHub-এ `eventix-server` repo, `.gitignore`-এ `.env`, `node_modules`, `dist`
- [ ] MongoDB Atlas cluster (transaction-এর জন্য replica set)
- [ ] Redis Cloud, Cloudinary, SSLCommerz sandbox, Gmail App Password — সব নতুন credential
- [ ] Postman-এ `Eventix` collection, environment variable-এ `baseUrl` আর token
- [ ] M1 শুরু 🚀