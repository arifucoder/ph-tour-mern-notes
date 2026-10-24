# Frontend Notes — ২৬: DevOps — VPS-এ Client আর Server Deploy (Ubuntu, Nginx, PM2)

এই নোটে Phovia Tour-এর client (React build) আর server (Express) কীভাবে VPS-এ বসানো হলো, ধাপে ধাপে। আপনার নোটের বাইরে প্রায় কিছু যোগ করা হয়নি, শুধু যেখানে ভুল বা অস্পষ্ট ছিল সেখানে চিহ্ন দেওয়া।

প্রতিটা ধাপে 🎯 লক্ষ্য, 🛠️ কাজ, 💡 কেন, ✅ যাচাই।

### ধাপের মানচিত্র

| ধাপ | বিষয় |
|---|---|
| ০ | কেন root দিয়ে না |
| ১ | `ubuntu` user বানানো |
| ২ | Node (nvm), PM2, Bun install |
| ৩ | MongoDB Atlas-এ VPS-এর IP দেওয়া |
| ৪ | Folder-এর গঠন |
| ৫ | GitHub থেকে clone |
| ৬ | Client build করে `/var/www/phovia`-তে copy |
| ৭ | Domain, Cloudflare SSL আর Nginx |
| ৮ | Server-এর `.env` update আর PM2 restart |

---

## ধাপ ০ — কেন `/root`-এ রেখে root user দিয়ে না

🎯 **লক্ষ্য:** আলাদা user কেন লাগে, বোঝা।

| কী | `/root`-এ, root দিয়ে চালালে | কেন |
|---|---|---|
| **Backend** | কাজ করবে, PM2 ঠিকঠাক চলবে | কিন্তু Express app-এ বা কোনো npm package-এ security hole থাকলে, app যে user হিসেবে চলছে attacker সেই অনুমতি পেয়ে যায়। Root হলে সে **পুরো server-এর মালিক**: সব file পড়া, মোছা, বদলানো, malware বসানো। আলাদা user হলে ক্ষতি ঐ user-এর folder-এই আটকে থাকে |
| **Frontend** | **কাজই করবে না** | Nginx চলে `www-data` user হিসেবে, সে `/root`-এর ভেতরে ঢুকতে পারে না → site খুললে **403 Forbidden** |

---

## ধাপ ১ — `ubuntu` user বানানো

🎯 **লক্ষ্য:** app চালানোর জন্য একটা সাধারণ (root না) user, যার দরকার হলে `sudo` আছে।

🛠️ **কাজ (root হিসেবে login থাকা অবস্থায়):**

```shell
# ubuntu নামে user আছে কিনা
id ubuntu

# না থাকলে বানাও, আর sudo group-এ দাও
adduser ubuntu
usermod -aG sudo ubuntu

# password আছে কিনা দেখো (P = আছে, L = locked, NP = নেই)
passwd -S ubuntu
# না থাকলে সেট করো
passwd ubuntu

# এরপর থেকে সব কাজ ubuntu user হিসেবে
su - ubuntu
```

💡 **`usermod -aG sudo ubuntu`:** `-a` = append (আগের group-গুলো রেখে), `-G sudo` = sudo group-এ যোগ। `-a` বাদ দিলে আগের সব group থেকে বেরিয়ে যায়।

> ➕ **সংযোজন:** নোটে `su - ubuntu` ছিল না, কিন্তু এটা জরুরি। পরের ধাপের nvm, Node, PM2, Bun **ubuntu user হিসেবেই** install করতে হবে। Root হিসেবে করলে সব `/root/.nvm`-এ বসে যায়, ubuntu user সেগুলো খুঁজেই পায় না।

### Apps folder-এর মালিকানা

```shell
mkdir -p /home/ubuntu/apps
sudo chown -R ubuntu:ubuntu /home/ubuntu/apps
```

💡 **`ubuntu:ubuntu`:** প্রথমটা user, দ্বিতীয়টা group, একসাথে দুটোকেই মালিক বানানো। `-R` = ভেতরের সব file আর folder-এও।

> ⚠️ **সংশোধন:** নোটে শুধু `chown` ছিল, কিন্তু folder না থাকলে `chown` "No such file or directory" দেয়। তাই আগে `mkdir -p`। আর root folder বানালে মালিক হয় root, তাই `chown` লাগে; ubuntu user নিজে বানালে এমনিতেই মালিক সে।

✅ **যাচাই:** `whoami` → `ubuntu`; `ls -ld ~/apps` → `ubuntu ubuntu`।

---

## ধাপ ২ — Node (nvm), PM2, Bun install

🎯 **লক্ষ্য:** Linux server-এ Node install-এর সবচেয়ে ভালো উপায় **nvm**, ubuntu user-এর জন্য।

🛠️ **কাজ:**

```shell
# nvm install
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash

# terminal reload (নতুন PATH নেওয়ার জন্য)
source ~/.bashrc

# latest LTS version
nvm install --lts

# যাচাই
nvm --version
node -v
npm -v
which node        # /home/ubuntu/.nvm/... দেখালে ঠিক আছে
```

💡 **কেন nvm:** `apt install nodejs` অনেক সময় পুরনো version দেয়, আর global package-এর জন্য `sudo` লাগে। nvm Node বসায় user-এর নিজের folder-এ (`~/.nvm`), তাই `sudo` ছাড়াই `npm install -g`, আর পরে সহজে version বদলানো যায়।

### PM2

```shell
npm install -g pm2
which pm2         # /home/ubuntu/.nvm/... path দেখাবে
```

### Bun

```shell
# Bun install-এ unzip লাগে, server-এ সেটা নেই, তাই আগে:
sudo apt update
sudo apt install -y unzip

# তারপর Bun
curl -fsSL https://bun.sh/install | bash
source ~/.bashrc
bun --version
```

✅ **যাচাই:** `which node`, `which pm2` দুটোই `/home/ubuntu/...` দেখায় (`/root/...` বা `/usr/...` না)।

---

## ধাপ ৩ — MongoDB Atlas setup

🎯 **লক্ষ্য:** VPS থেকে database-এ connect করার অনুমতি দেওয়া।

🛠️ **কাজ:**
1. `cloud.mongodb.com`-এ একটা free cluster বানাও।
2. Database user বানিয়ে connection string (`mongodb+srv://...`) কপি করে রাখো, server-এর `.env`-এ লাগবে।
3. VPS-এর IP বের করো:

   ```shell
   curl ifconfig.me
   ```

4. **Atlas → Security → Database & Network Access → IP Access List**-এ এই IP add করো।

💡 **কেন:** Atlas নিরাপত্তার জন্য শুধু তালিকার IP থেকে connection নেয়। IP না দিলে server চালু হলেও database connect হয় না (timeout error)।

✅ **যাচাই:** server চালু করলে log-এ database connected দেখায়।

---

## ধাপ ৪ — Folder-এর গঠন

🎯 **লক্ষ্য:** কোন code কোথায় থাকবে, আগে ঠিক করা।

```
/home/ubuntu/apps/
├── phovia-tour-server/    ← pm2 এখান থেকে সরাসরি চালায়
└── phovia-tour-client/    ← শুধু build করার জায়গা

/var/www/phovia/           ← শুধু client-এর dist file (Nginx serve করে)
```

### 💡 Client-এর জন্য কেন এই পদ্ধতি সবচেয়ে ভালো

1. **নিরাপত্তা:** `/var/www/phovia` হলো public folder; Nginx এর ভেতরের সবকিছু internet-এ দেখাতে পারে। এখানে source code রাখলে ভুল config-এ `.env`, `.git` folder ইত্যাদি leak হয়ে যেতে পারে। শুধু `dist` রাখলে শুধু public হওয়ার মতো file-ই থাকে।
2. **Downtime নেই:** build চলার সময় (বা build fail করলে) live site অক্ষত থাকে। সফল build-এর পরেই একবারে copy হয়।
3. **Permission ঝামেলা নেই:** Nginx home folder পড়তে পারে না, কিন্তু `/var/www` পারে।

### 💡 Server code `~/apps`-এ রাখাই ঠিক ✅

Backend-এর ক্ষেত্রে পরিস্থিতি আলাদা:
- Server code pm2 চালায় ubuntu user হিসেবে, তাই নিজের home folder-এ থাকলে permission-এর কোনো সমস্যা নেই।
- Nginx এখানে file পড়ে না, শুধু **reverse proxy** করে (request পাঠিয়ে দেয় `localhost:5000`-এর মতো port-এ)। তাই Nginx-এর home folder access লাগে না।
- `.env` বা source code কোনোভাবেই public হয় না, কারণ কোনো public folder-এ নেই।

---

## ধাপ ৫ — GitHub থেকে clone

🛠️ **কাজ:**

```shell
cd ~/apps
git clone https://github.com/arifucoder/phovia-tour-client.git
cd phovia-tour-client
```

> ➕ **সংযোজন:** `cd ~/apps` আগে, নাহলে যে folder-এ আছেন সেখানেই clone হয়, ধাপ ৪-এর গঠন ভেঙে যায়। Repo private হলে GitHub username আর password-এর জায়গায় **Personal Access Token** লাগে।

---

## ধাপ ৬ — Client build করে `/var/www/phovia`-তে copy

🎯 **লক্ষ্য:** production build বানিয়ে Nginx-এর folder-এ রাখা।

🛠️ **কাজ:**

```shell
# ১. .env বানাও (backend-এর live URL ইত্যাদি)
vim .env

# ২. package install আর build
bun install
bun run build

# ৩. deploy folder তৈরি (একবারই)
sudo mkdir -p /var/www/phovia
sudo chown -R ubuntu:ubuntu /var/www/phovia

# ৪. build copy
rsync -a --delete dist/ /var/www/phovia/
```

💡 **`.env` build-এর আগে কেন:** Vite-এর `VITE_...` মানগুলো build-এর সময় সরাসরি JS file-এর ভেতরে বসে যায়। তাই `.env` বদলালে **আবার build** করতে হয়, শুধু file বদলালে কিছু হয় না।

💡 **`rsync -a --delete dist/ /var/www/phovia/`:**
- `-a` = archive: সব file, folder, permission ঠিক রেখে copy।
- `--delete` = `/var/www/phovia`-তে এমন পুরনো file থাকলে যা নতুন `dist`-এ নেই, সেগুলো মুছে দাও (আগের build-এর পুরনো JS জমে থাকে না)।
- `dist/`-এর শেষের `/` জরুরি: মানে "dist-এর **ভেতরের** জিনিস"। `/` না দিলে `/var/www/phovia/dist/...` হয়ে যায়, site 403/404 দেয়।

💡 **`chown` কেন:** `sudo mkdir` folder-এর মালিক বানায় root, তখন ubuntu user `rsync` দিয়ে লিখতে পারে না। মালিক ubuntu বানালে প্রতিবার deploy-এ `sudo` লাগে না। Nginx (`www-data`) শুধু পড়ে, সেটা এমনিতেই পারে।

✅ **যাচাই:** `ls /var/www/phovia` → `index.html`, `assets/` (কোনো `dist` folder না)।

---

## ধাপ ৭ — Domain, Cloudflare SSL আর Nginx

🎯 **লক্ষ্য:** `live.arifucoder.dev.cv` খুললে client দেখানো, HTTPS সহ।

🛠️ **কাজ ১:** subdomain (বা যে domain-এ client রাখবেন) Cloudflare-এ add করে SSL-এর কাজ করা (DNS-এ VPS-এর IP-তে `A` record)।

🛠️ **কাজ ২:** `/etc/nginx/sites-available/live.arifucoder.dev.cv` file বানানো:

```shell
sudo vim /etc/nginx/sites-available/live.arifucoder.dev.cv
```

```nginx
server {
    listen 80;
    listen 443 ssl;
    server_name live.arifucoder.dev.cv;

    include /etc/nginx/snippets/cloudflare-ssl-arifucoder.dev.cv.conf;

    root /var/www/phovia;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

💡 **লাইনগুলোর মানে:**
- **`listen 80` / `listen 443 ssl`:** HTTP আর HTTPS দুই port-এই শোনো।
- **`server_name`:** কোন domain-এর request এই block সামলাবে।
- **`include ...cloudflare-ssl...conf`:** আগে বানানো SSL certificate-এর setting (Cloudflare-এর certificate), প্রতিটা subdomain-এ আবার না লিখে একটা file থেকে নেওয়া।
- **`root /var/www/phovia`:** file কোথা থেকে দেবে।
- **`try_files $uri $uri/ /index.html`:** **React Router-এর জন্য জরুরি।** আগে ঐ নামের file খোঁজো (`/assets/index.js`), না পেলে folder, তাও না পেলে `index.html` দাও। `/tours/123`-এর মতো route-এর কোনো আসল file নেই; এই লাইন না থাকলে সরাসরি ঐ link খুললে বা reload দিলে **404** আসত। `index.html` পেলে React Router নিজে ঠিক page দেখায়।

🛠️ **কাজ ৩:** চালু করা

```shell
sudo ln -s /etc/nginx/sites-available/live.arifucoder.dev.cv /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

💡 **কেন:**
- **`ln -s`:** `sites-available` = সব config রাখার জায়গা; `sites-enabled` = যেগুলো চালু। Shortcut (symlink) বানালে চালু, মুছলে বন্ধ, আসল file রয়ে যায়।
- **`nginx -t`:** config-এ ভুল আছে কিনা পরীক্ষা। ভুল থাকলে reload করলে Nginx-এর **সব** site বন্ধ হতে পারে, তাই সবসময় আগে `-t`।
- **`reload`:** চলমান অবস্থায় নতুন config নেওয়া, site বন্ধ না করে।

✅ **যাচাই:**
- `nginx -t` → `syntax is ok`, `test is successful`।
- Browser-এ `https://live.arifucoder.dev.cv` → site আসে।
- কোনো ভেতরের page-এ (যেমন `/tours`) গিয়ে reload → 404 আসে না।

---

## ধাপ ৮ — Server-এর `.env` update আর PM2 restart

🎯 **লক্ষ্য:** client-এর নতুন live URL backend-কে জানানো।

🛠️ **কাজ:**

```shell
cd ~/apps/phovia-tour-server
vim .env
```

যা বদলাতে হবে (**অবশ্যই**):
- **Frontend URL** (CORS-এর জন্য, আর redirect-এর জন্য): `https://live.arifucoder.dev.cv`
- **SSLCommerz-এর frontend link**গুলো: success, fail, cancel URL, সবগুলো নতুন domain-এ

তারপর (TypeScript server হলে) build, আর PM2-কে নতুন `.env` জানানো:

```shell
bun run build
pm2 restart 0 --update-env
```

💡 **কেন সব জরুরি:**
- **CORS:** backend শুধু তালিকার origin থেকে cookie সহ request নেয়। পুরনো URL থাকলে live site-এ login কাজ করে না।
- **SSLCommerz link:** payment শেষে user-কে এই URL-এ ফেরত পাঠানো হয়। পুরনো থাকলে payment-এর পরে `localhost`-এ চলে যায়।
- **`--update-env`:** এটা ছাড়া PM2 আগের environment দিয়েই চালায়, `.env` বদলেও কিছু হয় না (২৫ নম্বর নোট)।

> ➕ **সংযোজন:** নোটে "build korte hobe" লেখা ছিল। Server যদি TypeScript থেকে `dist`-এ build হয়ে চলে, তাহলে code বদলালে build লাগে। শুধু `.env` বদলালে build লাগে না, কারণ server `.env` চালু হওয়ার সময় পড়ে, build-এর সময় না (client-এর উল্টো, ধাপ ৬)। শুধু `pm2 restart 0 --update-env` যথেষ্ট।

✅ **যাচাই:**
- `pm2 list` → app `online`; `pm2 logs 0` → কোনো error নেই।
- Live site-এ login → logout হয় না।
- একটা test payment → success page live domain-এ আসে।

---

# Check-list

- [ ] `ubuntu` user আছে, sudo group-এ, password সেট
- [ ] Node, PM2, Bun ubuntu user হিসেবে install (`which node` → `/home/ubuntu/.nvm/...`)
- [ ] Atlas-এর IP Access List-এ VPS-এর IP
- [ ] Code `~/apps`-এ, client-এর শুধু `dist` `/var/www/phovia`-তে
- [ ] Client-এর `.env` build-এর **আগে** ঠিক করা
- [ ] `rsync`-এ `dist/` (শেষে `/`)
- [ ] Nginx config-এ `try_files ... /index.html`; `nginx -t` পাস
- [ ] Server-এর `.env`-এ নতুন frontend URL আর SSLCommerz-এর সব link
- [ ] `pm2 restart 0 --update-env`
- [ ] Live site-এ login, reload আর payment পরীক্ষা
