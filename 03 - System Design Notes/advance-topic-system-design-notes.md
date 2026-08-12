# 🚀 ADVANCED SYSTEM DESIGN — Topics Beyond The Course

> Ye topics course PDF me nahi the, lekin 1-2 saal experience ke baad interviews me poochte hain. Same style — concept + analogy + code + kab use karna hai.

---

# 1. CAP Theorem (Distributed Systems Ki Foundation)

## Kya Hai
Ek distributed system (jisme data multiple servers pe hai — jaise Replication/Sharding wale setup) **teeno cheezein ek saath 100% guarantee nahi kar sakta**:

| Letter | Matlab |
|---|---|
| **C — Consistency** | Har server se same time pe **same, latest** data milega |
| **A — Availability** | System hamesha **response dega** (chahe data thoda purana ho) |
| **P — Partition Tolerance** | Servers ke beech network toot jaaye (partition), tab bhi system **kaam karta rahe** |

**Real duniya me Partition Tolerance almost hamesha chahiye hoti hai** (network fail ho hi sakta hai) — isliye **asli choice hoti hai: Consistency vs Availability**.

## Restaurant Analogy

Socho tumhare 2 branches hain (Delhi aur Mumbai), dono ka menu-board sync hona chahiye. Achanak dono branches ke beech network toot gaya (Partition).

- **Consistency choose karo**: Jab tak dono branches sync na ho jaayein, **koi bhi order lena band** kar do (Availability qurbaan) — taaki galat/purana menu na dikhe
- **Availability choose karo**: Order lete raho dono branches me (chahe menu thoda mismatch ho jaaye) — turant customer ko "sorry" bolna nahi (Consistency qurbaan)

## Real Systems Kya Choose Karte Hain

| System | Choice |
|---|---|
| MongoDB (default settings) | **Availability** ki taraf — Secondary se stale read allow karta hai |
| Banking systems | **Consistency** ki taraf — galat balance dikhane se accha thoda downtime |
| Redis | Configurable — dono modes possible |

## Kab Interview Me Kaam Aata Hai
Jab tumse poocha jaaye "Replication design karte waqt Secondary se read allow karoge ya nahi" — iska jawab CAP theorem se aata hai: Availability chahiye to allow karo (stale data ka risk lo), Consistency chahiye to sirf Primary se read karwao (thoda slow/less available).

---

# 2. Consistent Hashing (Sharding Ko Deeply Samajhna)

## Problem — Normal Hashing Kyun Fail Karti Hai

Maan lo tum simple formula use karte ho:
```
shard_number = hash(user_id) % total_shards
```

Agar `total_shards = 3` hai, aur tum ek naya shard add karte ho (`total_shards = 4`), **almost saara data ka `% ` result badal jaata hai** — matlab **poora data reshuffle** karna padega naye shards me. Ye bahut expensive operation hai bade systems me.

## Solution — Consistent Hashing

Socho shards aur data keys dono ko ek **circular ring (0 se 360 degree)** pe map kar diya jaaye (hash function se).

```
        Shard A (at 0°)
       /              \
Shard C (240°)      Shard B (120°)
```

Har **data key** ring pe ek jagah padta hai, aur usko **clockwise sabse najdeek wale Shard** ko assign kar diya jaata hai.

**Jab naya Shard add karte ho**: Wo bhi ring pe kahin aa jaata hai — **sirf uske "clockwise pehle" wale data ko** move karna padta hai naye shard me, **baaki sab jagah same rehta hai.**

**Analogy**: Socho ek gol table pe log baithe hain (Shards), aur naye log (data) table ke around khade ho jaate hain — har naya banda apne **"clockwise sabse najdeek baithe insaan"** ko follow karta hai. Agar table pe ek naya banda (Shard) baith jaaye, sirf uske paas khade log hi apna "leader" badlenge, baaki sab same rehte hain.

## Kaha Use Hoti Hai
DynamoDB, Cassandra, aur bade distributed caching systems (jaise distributed Redis clusters) — jahan servers frequently add/remove hote hain bina poora data reshuffle kiye.

---

# 3. Message Queues (Kafka vs RabbitMQ vs BullMQ)

## Kyun Chahiye (Recap + Extend)

Humne BullMQ dekha tha (Redis-based) — email jaisi background tasks ke liye. Lekin bade systems me **teen alag tools** milte hain, alag use-cases ke liye:

| Tool | Best For | Kyun |
|---|---|---|
| **BullMQ** (Redis-based) | Chhoti-medium apps, simple background jobs | Simple setup, Redis already hoga to extra infra nahi chahiye |
| **RabbitMQ** | Complex routing, guaranteed delivery | Advanced message routing (exchanges, bindings), reliable acknowledgments |
| **Kafka** | Bahut high-throughput, event streaming, multiple consumers same data padhein | Lakhon events/second handle kar sakta hai, data ko "replay" bhi kar sakte ho (log-based) |

## Kafka Ka Core Concept (Naya, BullMQ Se Alag)

BullMQ me ek job **ek hi worker** consume karta hai aur khatam. Kafka me messages ek **"log"** (jaise ek diary) me store hote hain, aur **multiple consumers alag-alag speed pe** unhe padh sakte hain — kisi ek consumer ke padhne se message delete nahi hota.

```
Producer → Kafka Topic (log ki tarah, persist hota hai)
                ↓
      Consumer Group 1 (Analytics service — apni speed se padhta hai)
      Consumer Group 2 (Notification service — apni speed se padhta hai)
```

**Kab Kafka chahiye**: Jab **same event ko multiple services** ko independently process karna ho (jaise "Order Placed" event — Payment Service, Inventory Service, Notification Service, Analytics Service — sabko chahiye, alag-alag speed pe).

---

# 4. Idempotency (Payment/Order APIs Ke Liye Must-Know)

## Kya Hai
Ek operation **idempotent** hai agar usko **1 baar ya 100 baar chalao, result same hi rahe** (koi duplicate side-effect na ho).

## Problem Jo Ye Solve Karta Hai

Socho user "Pay ₹500" button dabata hai. Network slow hai, response nahi aaya, user **phir se dabaata hai**. Agar backend properly design nahi hai, **2 baar payment ho sakta hai!**

## Solution — Idempotency Key

```js
app.post("/payment", async (req, res) => {
  const idempotencyKey = req.headers["idempotency-key"]; // frontend generate karke bhejta hai

  // Pehle check karo ye request pehle bhi process hui thi kya
  const existing = await redis.get(`idempotency:${idempotencyKey}`);
  if (existing) {
    return res.json(JSON.parse(existing));  // purana result hi wapas bhej do, dobara process mat karo
  }

  // Naya payment process karo
  const result = await processPayment(req.body);

  // Result ko is key ke saath save kar do (24hr ke liye)
  await redis.set(`idempotency:${idempotencyKey}`, JSON.stringify(result), "EX", 86400);

  res.json(result);
});
```

**Kya ho raha hai**: Frontend har request ke saath ek **unique key** bhejta hai (ek UUID, jo button click pe generate hota hai, dobara click pe **same key** reuse hoti hai). Backend check karta hai — "ye key pehle process ho chuki hai?" Agar haan, **purana result hi wapas de do**, dobara payment mat karo.

**Kab chahiye**: Payment, Order placement, Email sending jaise APIs jaha **duplicate execution bahut mehenga/dangerous** ho sakta hai.

---

# 5. CDN (Content Delivery Network)

## Kya Hai
> Static content (images, CSS, JS, videos) ko duniya bhar ke **multiple locations** pe cache karke rakhna, taaki user ko **sabse najdeek wale server** se content mile — fast loading.

## Restaurant Analogy
Socho tumhara ek famous recipe (menu image) hai jo poori duniya me dikhani hai. Har baar tumhara **ek hi kitchen (origin server)** se photo bhejna slow hoga (dur wale country ke liye). Isliye tum us photo ki copies **local branches (CDN edge servers)** me rakh dete ho — jo bhi user jaha se aaye, usko **wahi paas ka branch** photo de deta hai.

```
User (India) → CDN Edge Server (Mumbai) → Fast! (image already cached)
User (US)     → CDN Edge Server (New York) → Fast!

(Agar CDN pe nahi mila, tab jaake Origin Server se laayega — aur us CDN node pe cache kar lega agli baar ke liye)
```

## Kaha Use Hota Hai
- Website images/CSS/JS (Cloudflare, AWS CloudFront)
- Video streaming (YouTube, Netflix)
- **Nginx bhi CDN jaisa kaam kar sakta hai** chhote scale pe (static file caching) — lekin CDN **geographically distributed** hota hai, Nginx ek hi jagah.

## Kab Interview Me Poochte Hain
"Tumhari website globally slow load ho rahi hai kuch countries me — kya karoge?" → **CDN** ka jawab hona chahiye, na ki sirf "server upgrade karo".

---

# 6. Circuit Breaker + Retry Patterns (Microservices Resilience)

## Problem
Microservices me Service A, Service B ko call karti hai. Agar Service B **slow ho gayi ya down ho gayi**, Service A bhi wait karte-karte block ho jaayegi — **failure cascade** ho sakta hai (ek service ki problem poore system ko down kar de).

## Circuit Breaker Pattern

Socho ek **electrical circuit breaker** (fuse) — jab bahut zyada current aata hai, wo **trip** ho jaata hai (band ho jaata hai) taaki poora ghar na jal jaaye.

```
State 1: CLOSED (normal)     → Requests Service B ko jaati hain normally
State 2: OPEN (trip ho gaya)  → Service B baar-baar fail ho rahi hai → ab requests Service B ko jaati hi nahi, turant fallback response milta hai
State 3: HALF-OPEN (test)      → Kuch der baad, ek test request bhejta hai dekhne — Service B theek hai kya
                                  → theek hai to CLOSED wapas, nahi to phir OPEN
```

### Simple Code Example (Concept Dikhane Ke Liye)

```js
class CircuitBreaker {
  constructor() {
    this.failureCount = 0;
    this.state = "CLOSED";
    this.threshold = 5; // 5 baar fail ho to trip
  }

  async call(fn) {
    if (this.state === "OPEN") {
      return { error: "Service unavailable, try later" }; // turant fallback, wait nahi
    }

    try {
      const result = await fn();
      this.failureCount = 0; // success pe reset
      return result;
    } catch (err) {
      this.failureCount++;
      if (this.failureCount >= this.threshold) {
        this.state = "OPEN";
        setTimeout(() => { this.state = "HALF-OPEN"; }, 30000); // 30 sec baad retry allow
      }
      throw err;
    }
  }
}
```

## Retry Pattern (Circuit Breaker Ke Saath Use Hota Hai)

Agar request fail ho, turant hi 2-3 baar retry karo (thodi delay ke saath — **exponential backoff**):
```
1st retry: 1 sec baad
2nd retry: 2 sec baad
3rd retry: 4 sec baad
(fir Circuit Breaker trip ho sakta hai agar sab fail ho jaayein)
```

**Kyun exponential backoff**: Agar turant-turant retry karoge, aur **wahi problem** hai (service overloaded), tumhare retries **aur zyada overload** kar denge — isliye har baar zyada wait karo.

---

# 7. Database Indexing (Deep — Query Slow Kyun Hoti Hai)

## Problem Bina Index Ke

```js
db.users.find({ email: "dev@example.com" })
```

Bina index ke, MongoDB **poori table (Collection Scan)** check karta hai — har document ka email match kar raha hai. Agar 1 million documents hain, **1 million checks** — bahut slow.

## Index Kya Hai
Ek **separate, sorted structure** (jaise ek book ka index page) jo directly bata deta hai "email `dev@example.com` wala document **kaha** hai" — bina poori collection scan kiye.

```js
db.users.createIndex({ email: 1 })  // 1 = ascending order
```

**Analogy**: Kitab me agar tumhe ek topic dhoondhna ho, **bina index ke** tumhe **har page palatna** padega. **Index ke saath**, tum index page dekho, wahi bata deta hai "page 245 pe hai" — seedha wahi khol lo.

## Trade-off — Index Free Nahi Hai

| Fayda | Nuksan |
|---|---|
| Reads bahut fast | Writes thodi slow (kyunki har insert/update pe index bhi update karna padta hai) |
| | Extra storage lagti hai index rakhne ke liye |

**Isliye**: Har field pe index mat laga do — sirf un fields pe jo **baar-baar `find`/`filter`/`sort`** me use hote hain.

## Compound Index (Advanced)

```js
db.orders.createIndex({ userId: 1, createdAt: -1 })
```

Ye ek saath **do fields** pe index banata hai — useful jab query dono field pe filter/sort kare (jaise "is user ke orders, latest se sabse purane"). **Order matter karta hai** compound index me — `{userId, createdAt}` alag hai `{createdAt, userId}` se performance-wise.

---

# 8. Rate Limiting Algorithms (Basic Se Aage)

Humne **Fixed Window** dekha tha (`INCR` + `EXPIRE`). Ye 3 aur important algorithms hain:

## 🔹 Token Bucket

Socho ek **bucket** hai jisme har second ek **token** dalta jaata hai (max capacity tak). Har request ek token **consume** karti hai. Bucket khaali ho gaya to request reject.

```
Bucket capacity: 10 tokens
Refill rate: 2 tokens/sec

Request aayi → 1 token liya (bucket me 9 bache)
Agar bucket khaali → request reject (429)
```

**Fayda Fixed Window Se**: **Burst traffic allow karta hai** (agar bucket full hai, ek saath 10 requests aa sakti hain), lekin overall rate control me rehta hai. Fixed Window me sharp boundary problem hoti hai (window ke end aur start me double requests aa sakti hain).

## 🔹 Sliding Window Log

Har request ka **exact timestamp** store karo, aur check karo "pichle 60 second me kitni requests thi" (fixed 60-sec block nahi, **rolling** window).

```js
// Redis Sorted Set use karke
await redis.zremrangebyscore(key, 0, now - 60000); // 60 sec se purani entries hatao
await redis.zadd(key, now, `${now}`);
const count = await redis.zcard(key);
```

**Fayda**: Bahut precise hai (koi boundary trick nahi), lekin **zyada memory** use karta hai (har request store karni padti hai).

## 🔹 Leaky Bucket

Requests ek **queue** me aati hain, aur **fixed rate** pe process hoti hain (jaise ek bucket me pani daalo, wo **fix speed se leak/nikalta** hai). Agar bahut fast aayein, queue full ho jaaye to reject.

**Kab kaunsa use karo**:
| Algorithm | Best For |
|---|---|
| Fixed Window (jo humne seekha) | Simple cases, thoda inaccurate chalega |
| Token Bucket | Burst traffic allow karna hai but overall limit chahiye |
| Sliding Window | Precise control chahiye, memory available hai |
| Leaky Bucket | Output rate **bilkul smooth/constant** chahiye |

---

# 9. Saga Pattern (Distributed Transactions — Microservices Me)

## Problem

Monolith me agar Order place karte waqt "Payment deduct karo + Inventory kam karo + Order create karo" — sab **ek hi database transaction** me hota hai (ACID — sab ya to sab ho, ya kuch na ho).

Microservices me — Payment Service, Inventory Service, Order Service **alag databases** hain. **Ek single transaction possible nahi hai across services.**

## Saga Pattern — Solution

Poore operation ko **chhote steps** me todo, har step apna kaam kare, aur agar koi step **fail** ho jaaye, to **pichle steps ko "undo" (Compensating Transaction)** karo.

```
Step 1: Order Service    → Order create karo (status: PENDING)
Step 2: Payment Service  → Payment deduct karo
Step 3: Inventory Service → Stock kam karo

Agar Step 3 FAIL ho jaaye:
Compensate Step 2: Payment REFUND karo
Compensate Step 1: Order status CANCELLED karo
```

**Restaurant analogy**: Order liya, payment liya, lekin kitchen me pata chala **ingredient khatam hai**. Ab tumhe **payment wapas** karna padega, aur order **cancel** karna padega — ye "undo" steps hi Compensating Transactions hain.

## 2 Types

| Type | Kaise |
|---|---|
| **Choreography** | Har service khud agla event trigger karti hai (koi central controller nahi) — jaise Kafka events se |
| **Orchestration** | Ek central "Saga Orchestrator" service sabko control karti hai — "ab Payment karo", "ab Inventory update karo" |

---

# 10. Observability — Logging, Metrics, Tracing

## Kyun Chahiye
Jab system **10-20 microservices** ka ho, ek request kai services se hoke guzarti hai — agar kuch **slow/fail** ho, **kaise pata chalega kaha problem hai?**

## 3 Pillars

### 🔹 Logging
Har service apne kaam ka **text record** rakhti hai.
```js
console.log(`[Auth Service] User ${userId} logged in at ${new Date()}`);
```
**Problem bade scale pe**: Har service alag logs rakhti hai — dhoondhna mushkil. Isliye **centralized logging** (jaise ELK Stack — Elasticsearch, Logstash, Kibana) use karte hain, sab logs ek jagah collect hote hain.

### 🔹 Metrics
**Numbers** jo system ki health batate hain — jaise "average response time", "requests per second", "error rate". Tools: **Prometheus + Grafana** (graphs/dashboards banane ke liye).

### 🔹 Distributed Tracing
Ek request jab **multiple services** se guzarti hai, har service pe usko ek **same Trace ID** de dete hain — taaki tum poori journey track kar sako "request kaha slow hui".

```
Trace ID: abc123
  → API Gateway (2ms)
  → Auth Service (5ms)
  → Order Service (450ms)  ← YAHAN SLOW HAI, isko dekhna hai
  → Payment Service (10ms)
```

**Tools**: Jaeger, Zipkin.

**Kab interview me kaam aata hai**: "Production me ek request slow hai, poore Microservices architecture me — kaise debug karoge?" → **Distributed Tracing** hi jawab hai, sirf logs se pata nahi chalega multi-service system me.

---

# 🗺️ Advanced Topics — Kab Kaunsa Yaad Karna Hai

| Situation | Topic Yaad Karo |
|---|---|
| "Replication me stale data ka risk kaise handle karoge" | CAP Theorem |
| "Naya Shard add karna hai bina reshuffle ke" | Consistent Hashing |
| "Ek event multiple services ko chahiye" | Kafka |
| "Payment 2 baar ho sakta hai duplicate click se" | Idempotency |
| "Globally slow loading" | CDN |
| "Ek service down poore system ko block kar rahi" | Circuit Breaker |
| "Query slow hai" | Indexing |
| "Rate limiting me burst traffic allow karna hai" | Token Bucket |
| "Multi-service transaction, ek fail ho gaya" | Saga Pattern |
| "Multi-service request debug karni hai" | Distributed Tracing |

---

*Ye advanced topics course me nahi the — inhe "bonus depth" samjho jo tumhe peers se aage rakhega interviews me. Pehle basic notes (System_Design_Complete_Notes.md) pakka karo, phir ye topics padho.*