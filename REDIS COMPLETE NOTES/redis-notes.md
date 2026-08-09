# 🔴 REDIS — Poora Deep Dive (Theory + Full Working Code)

> Har topic me: **Kyun chahiye → Kaise kaam karta hai → Poora Code → Kaha/Kab use hota hai**

---

# PART 0 — Setup (Sabse Pehle Ye)

### Redis install + run karna (local machine pe)

```bash
# Mac
brew install redis
redis-server          # ye Redis ko start karega, port 6379 pe

# Windows → WSL use karo, ya Docker
docker run -d -p 6379:6379 redis

# Linux
sudo apt install redis-server
redis-server
```

Check karo chal raha hai ya nahi:
```bash
redis-cli ping
# Output: PONG   → matlab Redis chal raha hai
```

`redis-cli` ek terminal tool hai jisse tum seedha Redis se baat kar sakte ho, bina Node.js ke:
```bash
redis-cli
127.0.0.1:6379> SET name "Dev"
OK
127.0.0.1:6379> GET name
"Dev"
```

### Node.js project me Redis connect karna

```bash
npm init -y
npm install express ioredis
```

`config/redisClient.js`:
```js
import Redis from "ioredis";

const redis = new Redis("redis://localhost:6379");

redis.on("connect", () => {
  console.log("✅ Redis connected");
});

redis.on("error", (err) => {
  console.log("❌ Redis error:", err);
});

export default redis;
```

**Kyun aise likhte hain?** Ye ek **single connection** hai jo poori app me reuse hoga. Har jagah naya connection banane se resources waste honge — isliye ek jagah bana ke `export` kar dete hain, jahan bhi chahiye `import` kar lo.

---

# PART 1 — Redis Ki Neenv: GET, SET, DEL (Deeply)

### Kyun chahiye
Ye Redis ke **sabse basic building blocks** hain — har pattern (caching, OTP, session, rate-limit) inhi 3-4 commands pe bana hai. Ye clear na ho to aage kuch samajh nahi aayega.

### Terminal (redis-cli) me:
```bash
SET name "Deveshh"       # key = "name", value = "Deveshh" store hua
GET name                  # → "Deveshh"
DEL name                  # key delete
EXISTS name                # → 0 (nahi hai) ya 1 (hai)
EXPIRE name 60              # 60 sec baad "name" khud delete ho jayega
TTL name                    # kitna time bacha hai (seconds me), -1 = never expire, -2 = already expired/nahi hai
SET name "Dev" EX 60        # set + 60 sec expiry ek saath
INCR counter                 # number ko atomically +1
DECR counter                 # number ko atomically -1
```

### Node.js me (yehi commands, code se):

```js
import redis from "./config/redisClient.js";

async function basics() {
  // SET
  await redis.set("name", "Deveshh");

  // GET
  const name = await redis.get("name");
  console.log(name); // "Deveshh"

  // DEL
  await redis.del("name");

  // EXPIRE wala SET (OTP jaisa use-case)
  await redis.set("otp:9876543210", "482913", "EX", 300); // 5 min expiry

  // EXISTS
  const exists = await redis.exists("name"); // 0 ya 1

  // INCR (rate limiting me use hota hai)
  const count = await redis.incr("visits"); // pehli baar → 1, agli baar → 2, ...
}

basics();
```

### Kab/kaha use hote hain
| Command | Real Use |
|---|---|
| `SET` / `GET` | Cache me data store/read karna |
| `DEL` | Logout pe session hatana, OTP verify hone ke baad hatana |
| `EXPIRE` / `EX` | OTP, session, rate-limit window — sab jagah "temporary rakhna" |
| `INCR` | Rate limiting counter, view counter, like counter |

---

# PART 2 — JSON.stringify / JSON.parse (Object Store Karna)

### Kyun chahiye
Redis **sirf strings** samajhta hai. JavaScript object ya array seedha Redis me nahi ja sakta.

```js
const user = { name: "Dev", age: 21, city: "Lucknow" };

// ❌ GALAT — ye error dega ya "[object Object]" store hoga
await redis.set("user:1", user);

// ✅ SAHI — object ko text (JSON string) banao
await redis.set("user:1", JSON.stringify(user));
// Redis me actually store hua: '{"name":"Dev","age":21,"city":"Lucknow"}'

// ✅ Nikalte waqt wapas object banao
const data = await redis.get("user:1");
const userObj = JSON.parse(data);
console.log(userObj.name); // "Dev"
```

### Poora chhota example — User cache karna

```js
async function getUser(userId) {
  // Pehle Redis me check karo
  const cached = await redis.get(`user:${userId}`);
  if (cached) {
    console.log("Cache se mila");
    return JSON.parse(cached);
  }

  // Nahi mila → DB se lao (yahan fake DB call)
  const userFromDB = await fakeDatabaseCall(userId);

  // Redis me save karo agli baar ke liye (JSON string bana ke)
  await redis.set(`user:${userId}`, JSON.stringify(userFromDB), "EX", 3600); // 1hr cache

  console.log("DB se mila, cache me save kiya");
  return userFromDB;
}
```

---

# PART 3 — API Caching (Cache-Aside Pattern) — Poora Working Example

### Kyun chahiye
Baar-baar DB query karna slow hota hai. Same data agar baar-baar maanga ja raha hai (jaise products list), to usko Redis me temporarily rakh do.

### Poora Flow
```
Request → Redis check → Mila (Cache Hit)? → Return fast ⚡
                       → Nahi mila (Cache Miss)? → DB se lao → Redis me save karo → Return
```

### Poora Express route (real project jaisa):

```js
import express from "express";
import redis from "./config/redisClient.js";
import Product from "./models/Product.js"; // Mongoose model (example)

const app = express();

app.get("/products", async (req, res) => {
  try {
    // Step 1: Redis me check karo
    const cachedProducts = await redis.get("all_products");

    if (cachedProducts) {
      console.log("⚡ Cache Hit — Redis se de diya");
      return res.json(JSON.parse(cachedProducts));
    }

    // Step 2: Cache Miss — DB se lao
    console.log("🐢 Cache Miss — MongoDB se laa rahe hain");
    const products = await Product.find();

    // Step 3: Redis me save karo (60 sec ke liye cache rakho)
    await redis.set("all_products", JSON.stringify(products), "EX", 60);

    // Step 4: Response bhejo
    res.json(products);

  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

app.listen(3000, () => console.log("Server running on 3000"));
```

**Line-by-line kyun:**
- `redis.get("all_products")` — pehle Redis me check, kyunki ye DB se 100x fast hai
- Agar mil gaya (`cachedProducts` truthy) — seedha wahi bhej do, DB ko touch mat karo
- Nahi mila — DB se query maaro (`Product.find()`)
- Result ko Redis me save kar do — **`EX 60`** matlab ye cache sirf 60 sec ke liye valid hai (kyunki products change ho sakte hain, hamesha ke liye cache nahi rakh sakte)

### Cache Invalidation (Important — jab data update ho)

Agar koi naya product add kare, to purana cache **galat (stale)** ho jayega. Isliye:

```js
app.post("/products", async (req, res) => {
  const newProduct = await Product.create(req.body);

  // Purana cache delete kar do, taaki next GET request fresh data laaye
  await redis.del("all_products");

  res.json(newProduct);
});
```

**Ye samajhna zaroori hai**: Caching ka sabse bada risk hai **stale data** (purana/galat data dikhana). Isliye jab bhi underlying data change ho, uska cache invalidate (delete) karna mat bhoolo.

---

# PART 4 — OTP Storage — Poora Working Example

### Poora flow with code (signup/login OTP verification jaisa real system)

```js
import express from "express";
import redis from "./config/redisClient.js";

const app = express();
app.use(express.json());

// Step 1: OTP generate + send
app.post("/send-otp", async (req, res) => {
  const { phone } = req.body;

  // 6-digit random OTP banao
  const otp = Math.floor(100000 + Math.random() * 900000);

  // Redis me save karo — 5 min (300 sec) ke liye
  await redis.set(`otp:${phone}`, otp, "EX", 300);

  // Yahan real app me SMS gateway se OTP bhejte (Twilio/MSG91 etc.)
  console.log(`OTP for ${phone}: ${otp}`); // for testing

  res.json({ message: "OTP sent successfully" });
});

// Step 2: OTP verify karo
app.post("/verify-otp", async (req, res) => {
  const { phone, otp } = req.body;

  const savedOtp = await redis.get(`otp:${phone}`);

  if (!savedOtp) {
    return res.status(400).json({ message: "OTP expired ya bheja hi nahi gaya" });
  }

  if (savedOtp !== String(otp)) {
    return res.status(400).json({ message: "Galat OTP" });
  }

  // Sahi hai — ab isko delete kar do (ek baar hi use ho sakta hai)
  await redis.del(`otp:${phone}`);

  res.json({ message: "OTP Verified! ✅" });
});

app.listen(3000);
```

**Kyun Redis perfect hai iske liye:**
1. `EX 300` se OTP apne aap 5 min baad delete ho jaata hai — koi cron job/setTimeout manually likhne ki zaroorat nahi
2. `DEL` se verify hote hi turant hata diya — replay attack (same OTP dobara use) rok diya
3. Bahut fast — SMS aane se pehle hi Redis me ready hota hai check karne ke liye

---

# PART 5 — Sessions (Login with JWT + Redis) — Poora Working Example

### Kyun chahiye
Jab user login karta hai, backend ko yaad rakhna hota hai "ye user logged-in hai" — bina baar-baar password maange. Iske liye **JWT token** banate hain aur usko Redis me bhi track karte hain (taaki chaho to forcefully logout/invalidate kar sako — pure JWT me ye possible nahi hota).

```js
import express from "express";
import jwt from "jsonwebtoken";
import redis from "./config/redisClient.js";

const app = express();
app.use(express.json());

const JWT_SECRET = "supersecretkey"; // .env me rakhna real project me

// LOGIN
app.post("/login", async (req, res) => {
  const { email, password } = req.body;

  // (yahan real app me DB se user check hota, password bcrypt.compare hota)
  const user = { id: "123", email }; // fake user for example

  // JWT token banao
  const token = jwt.sign({ userId: user.id }, JWT_SECRET, { expiresIn: "1d" });

  // Redis me session store karo — 24 hours (86400 sec)
  await redis.set(`session:${user.id}`, token, "EX", 86400);

  // Cookie me bhej do
  res.cookie("token", token, { httpOnly: true });
  res.json({ message: "Login successful" });
});

// PROTECTED ROUTE (middleware jo check karega login hai ya nahi)
async function isLoggedIn(req, res, next) {
  const token = req.cookies.token;
  if (!token) return res.status(401).json({ message: "Login required" });

  try {
    const decoded = jwt.verify(token, JWT_SECRET);

    // Redis me bhi check karo — session valid hai ya logout ho chuka
    const sessionToken = await redis.get(`session:${decoded.userId}`);
    if (sessionToken !== token) {
      return res.status(401).json({ message: "Session expired ya logged out" });
    }

    req.userId = decoded.userId;
    next();
  } catch (err) {
    res.status(401).json({ message: "Invalid token" });
  }
}

app.get("/dashboard", isLoggedIn, (req, res) => {
  res.json({ message: `Welcome user ${req.userId}` });
});

// LOGOUT
app.post("/logout", async (req, res) => {
  const token = req.cookies.token;
  const decoded = jwt.verify(token, JWT_SECRET);

  // Redis se session delete → token turant invalid ho jayega, chahe JWT khud expire na hua ho
  await redis.del(`session:${decoded.userId}`);

  res.clearCookie("token");
  res.json({ message: "Logged out" });
});

app.listen(3000);
```

**Sabse important cheez samjho — JWT + Redis dono kyun?**
- Sirf **JWT** use karoge to problem: ek baar issue hone ke baad, jab tak expire na ho, token hamesha valid rahega — chaho tum "logout" bhi kar do, backend ko pata nahi chalega (JWT stateless hota hai).
- **Redis ke saath**: session ko Redis me bhi store kar liya. Logout pe Redis se delete kar do → agli request pe check hoga "Redis me hai ya nahi" → nahi mila to reject, chahe JWT khud abhi valid ho.

**Ye pattern kehlata hai "Session-backed JWT"** — best of both worlds: JWT ki speed (verify karne ke liye DB call nahi chahiye) + Redis ki control (forcefully invalidate kar sakte ho).

---

# PART 6 — Rate Limiting — Poora Working Example

### Kyun chahiye
Bina limit ke koi bhi ek user/IP tumhare server ko spam kar sakta hai (brute force login, API abuse, DDoS) — server slow/crash ho sakta hai.

### Poora Middleware (production-style code)

```js
import redis from "./config/redisClient.js";

async function rateLimiter(req, res, next) {
  const ip = req.ip;
  const key = `rate_limit:${ip}`;

  // Is IP ka counter +1 karo (atomically — safe hai concurrent requests me bhi)
  const requests = await redis.incr(key);

  // Agar pehli hi request hai, to 60 sec ka window shuru karo
  if (requests === 1) {
    await redis.expire(key, 60);
  }

  // Kitna time bacha hai window khatam hone me
  const ttl = await redis.ttl(key);

  // Limit check — 60 sec me max 5 requests allowed
  if (requests > 5) {
    return res.status(429).json({
      message: "Too Many Requests",
      retryAfter: `${ttl} seconds baad try karo`
    });
  }

  next(); // limit ke andar hai, aage jaane do
}

export default rateLimiter;
```

### Use karna (kisi bhi route pe lagao):

```js
import express from "express";
import rateLimiter from "./middleware/rateLimiter.js";

const app = express();

// Sirf login route pe lagaya (brute-force se bachne ke liye)
app.post("/login", rateLimiter, (req, res) => {
  res.json({ message: "Login attempt processed" });
});

// Poori app pe bhi laga sakte ho
app.use(rateLimiter);

app.listen(3000);
```

**Deeply samjho har line:**
1. `redis.incr(key)` — Redis ka `INCR` command **atomic** hai, matlab agar 1000 requests ek hi millisecond me aayein, tab bhi counting bilkul sahi hogi (race condition nahi hogi — ye normal JavaScript variable `count++` se possible nahi hota multi-request scenario me)
2. `if (requests === 1) { expire(key, 60) }` — sirf **pehli** request pe hi timer set karo. Agar har request pe expire karte, to timer kabhi khatam hi nahi hota (rolling window ban jaata, jo galat hai)
3. `ttl(key)` — client ko batane ke liye "kitni der baad phir try karo"
4. `429` — ye standard HTTP status code hai specifically "Too Many Requests" ke liye — browsers/API clients isko samajh ke retry logic laga sakte hain

### Real-world variation — sliding window (advanced, bonus)

Upar wala "fixed window" hai (60 sec ka block fix). Zyada precise systems me **sliding window** use hota hai:

```js
// Sorted Set use karke sliding window rate limit
async function slidingWindowLimiter(ip) {
  const key = `sliding:${ip}`;
  const now = Date.now();
  const windowMs = 60000; // 1 min

  // Purane entries hatao jo window ke bahar hain
  await redis.zremrangebyscore(key, 0, now - windowMs);

  // Current request ko add karo
  await redis.zadd(key, now, `${now}`);

  // Window ke andar kitni requests hain count karo
  const count = await redis.zcard(key);

  await redis.expire(key, 60);

  return count <= 5; // true = allowed, false = blocked
}
```
*(Ye advanced hai — pehle basic wala clear karo, phir isko baad me explore karna)*

---

# PART 7 — Queues + BullMQ — Poora Working Project

### Kyun chahiye
Kuch kaam **slow** hote hain (email bhejna, image processing, PDF generate karna). Agar ye kaam request ke andar hi (synchronously) karoge, to user ko response milne me time lagega.

### Poora Setup

```bash
npm install bullmq ioredis
```

**Folder structure:**
```
project/
├── config/
│   └── redisClient.js
├── queues/
│   └── emailQueue.js
├── workers/
│   └── emailWorker.js
├── utils/
│   └── sendEmail.js
└── index.js
```

### `queues/emailQueue.js` — Queue banana

```js
import { Queue } from "bullmq";
import Redis from "ioredis";

// BullMQ ke liye special connection (maxRetriesPerRequest: null zaroori hai)
export const connection = new Redis("redis://localhost:6379", {
  maxRetriesPerRequest: null
});

const emailQueue = new Queue("emailQueue", { connection });

export default emailQueue;
```

### `utils/sendEmail.js` — Actual email bhejne wala function

```js
export default async function sendMail(email) {
  // Yahan real app me nodemailer/SendGrid use hota
  console.log(`📧 Sending email to ${email}...`);
  await new Promise((resolve) => setTimeout(resolve, 3000)); // fake 3 sec delay
  console.log(`✅ Email sent to ${email}`);
}
```

### `workers/emailWorker.js` — Background me jobs process karna

```js
import { Worker } from "bullmq";
import { connection } from "../queues/emailQueue.js";
import sendMail from "../utils/sendEmail.js";

const worker = new Worker("emailQueue", async (job) => {
  console.log(`🔵 Job Started: ${job.id}`);

  const { email } = job.data;
  await sendMail(email);

  console.log(`🟢 Job Completed: ${job.id}`);
}, { connection });

worker.on("failed", (job, err) => {
  console.log(`🔴 Job Failed: ${job.id}`, err.message);
});

console.log("👷 Worker is running and listening for jobs...");
```

**Ye worker ko alag se, separate terminal me chalate hain:**
```bash
node workers/emailWorker.js
```

### `index.js` — Main server (jaha job add hota hai)

```js
import express from "express";
import emailQueue from "./queues/emailQueue.js";

const app = express();
app.use(express.json());

app.post("/signup", async (req, res) => {
  const { email } = req.body;

  // (yahan real app me user DB me save hota)

  // Email bhejne ka kaam queue me daal do — turant return karo
  await emailQueue.add("send-email", { email });

  res.json({ message: "Signup successful! Welcome email will arrive shortly." });
});

app.listen(3000, () => console.log("Server running on port 3000"));
```

### Kaise run karo (2 terminals chahiye)

```bash
# Terminal 1 — main server
node index.js

# Terminal 2 — background worker (alag process)
node workers/emailWorker.js
```

**Kya hoga jab `/signup` call hoga:**
1. `index.js` — turant "Signup successful" response de dega (email ka wait nahi karega)
2. Job Redis me chala gaya `emailQueue` me
3. `emailWorker.js` (jo terminal 2 me chal raha hai, hamesha sunta rehta hai) — job ko utha lega
4. "Job Started" print hoga → 3 sec baad email "sent" hoga → "Job Completed" print hoga
5. **User ne bilkul wait nahi kiya**, sab background me hua

### Job/Worker/Queue — final analogy
- **Queue** = restaurant ki order-slip line (FIFO — pehli order pehle banegi)
- **Job** = ek order slip ("2 Pizza banao", data ke saath)
- **Worker** = chef jo slips uthata jaata hai, banata jaata hai

---

# PART 8 — Sab Kuch Ek Real Project Me Combine (End-to-End)

Poora `index.js` jisme sab patterns ek saath dikhte hain — isse dekhoge to poori picture clear ho jayegi:

```js
import express from "express";
import cookieParser from "cookie-parser";
import redis from "./config/redisClient.js";
import rateLimiter from "./middleware/rateLimiter.js";
import emailQueue from "./queues/emailQueue.js";
import jwt from "jsonwebtoken";

const app = express();
app.use(express.json());
app.use(cookieParser());

const JWT_SECRET = "secret123";

// ---- 1. SIGNUP (Queue use hota hai) ----
app.post("/signup", rateLimiter, async (req, res) => {
  const { email } = req.body;
  await emailQueue.add("send-email", { email });
  res.json({ message: "Signup successful" });
});

// ---- 2. SEND OTP (OTP storage use hota hai) ----
app.post("/send-otp", rateLimiter, async (req, res) => {
  const { phone } = req.body;
  const otp = Math.floor(100000 + Math.random() * 900000);
  await redis.set(`otp:${phone}`, otp, "EX", 300);
  res.json({ message: "OTP sent" });
});

// ---- 3. LOGIN (Session storage use hota hai) ----
app.post("/login", rateLimiter, async (req, res) => {
  const { userId } = req.body;
  const token = jwt.sign({ userId }, JWT_SECRET, { expiresIn: "1d" });
  await redis.set(`session:${userId}`, token, "EX", 86400);
  res.cookie("token", token);
  res.json({ message: "Logged in" });
});

// ---- 4. PRODUCTS (Caching use hota hai) ----
app.get("/products", async (req, res) => {
  const cached = await redis.get("all_products");
  if (cached) return res.json(JSON.parse(cached));

  const products = [{ name: "iPhone" }, { name: "Samsung TV" }]; // fake DB data
  await redis.set("all_products", JSON.stringify(products), "EX", 60);
  res.json(products);
});

app.listen(3000, () => console.log("🚀 Server running on port 3000"));
```

**Ek hi app me — Rate Limiting, OTP, Sessions, Caching, aur Queue (BullMQ) sab Redis pe hi based hain.** Ye poore backend system ka backbone hai.

---

# PART 9 — Quick Reference Table (Revision ke liye)

| Pattern | Redis Commands | Expiry Zaroori? | Kyun |
|---|---|---|---|
| Caching | `GET`, `SET` | Haan (data stale na ho) | Speed, DB load kam |
| OTP | `SET EX`, `GET`, `DEL` | Haan (security) | Temporary, auto-cleanup |
| Sessions | `SET EX`, `GET`, `DEL` | Haan (auto logout) | Login state track karna |
| Rate Limiting | `INCR`, `EXPIRE`, `TTL` | Haan (window reset) | Abuse/spam rokna |
| Queue (BullMQ) | Internally Redis lists/streams use karta hai | N/A | Background processing |

---

*Ab sab kuch code ke saath hai — is file ko khud se ek folder banake run karke try karo. Jahan error aaye ya samajh na aaye, wahi pooch lena.*