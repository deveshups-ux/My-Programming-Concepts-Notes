# 🏗️ SYSTEM DESIGN — Complete Notes (Concept + Code + Scenario)

> Restaurant analogy se poori kahani — Monolith se lekar Sharded Database tak

---

# PART 1 — Foundation: Scaling Concepts

## Scaling Kya Hai
> System ki capacity badhana taaki zyada users/traffic handle ho sake.

```
100 users → Server → No Issues ✅
1 million users → Server → CRASH ❌ (Scaling Needed)
```

## Vertical vs Horizontal Scaling

| | Vertical Scaling | Horizontal Scaling |
|---|---|---|
| Kya karte ho | Ek hi server ko **powerful** banate ho | **Naye servers add** karte ho (same size ke) |
| Before → After | 2 CPU/8GB → 8 CPU/64GB (ek hi server) | 1 server → 4 servers (har ek 2 CPU/8GB) |
| Problem | ❌ Expensive, ❌ Hardware ki limit | ✅ Real Modern Scaling — industry standard |

**Restaurant analogy**: Vertical = ek hi cook ko "superhuman" bana dena (impossible/expensive). Horizontal = zyada cooks hire karna (practical, scalable).

---

# PART 2 — Load Balancer

## Kya Hai
> Load balancer distributes incoming traffic across multiple backend servers.

```
users → Load Balancer → Server 1
                       → Server 2
                       → Server 3
```

**Round Robin (default algorithm)**:
```
Request 1 → Server1
Request 2 → Server2
Request 3 → Server3
Request 4 → wapas Server1 (cycle)
```

**Kyun chahiye**: Bina Load Balancer, saari requests ek hi server pe jaayengi → overload → crash. Baaki servers khaali baithe rahenge.

**Real Load Balancers**:
| Technology | Type |
|---|---|
| Nginx | Software |
| AWS ELB | Cloud service |

---

# PART 3 — Nginx (Deep Dive)

## Definition
> Nginx is a reverse proxy and load balancer used to manage traffic between users and backend servers.

## Do Alag Concepts — Poori Tarah Alag Karke Samjho

### 🔹 Reverse Proxy
Client ko pata nahi chalta backend me kya/kitna hai — client sirf Nginx se baat karta hai.
```
Client → Nginx → [Backend chupa hua hai]
```
**Restaurant analogy**: Reception desk — customer ko pata nahi andar konsa employee kaam karega, wo sirf reception se baat karta hai.

### 🔹 Load Balancer (Same Nginx, Doosra Kaam)
Multiple backends ke beech traffic baantna.
```
Client → Nginx → Backend1
              → Backend2
```

**NGINX Features**: ✅ reverse proxy, ✅ load balancing, ✅ SSL, ✅ routing

## `nginx.conf` — Poori File, Har Line Ka Reason

```nginx
events {}

http {
  upstream backend_servers {
    server backend1:3000;
    server backend2:3000;
  }

  server {
    listen 80;

    location / {
      proxy_pass http://backend_servers;
    }
  }
}
```

| Part | Kyun |
|---|---|
| `events {}` | Mandatory formality — connection handling settings, khaali chhod sakte ho |
| `http {}` | Web traffic ke rules yahan likhte hain |
| `upstream backend_servers {}` | Ek **naam diya gaya group** hai servers ka — jaise WhatsApp group banaya ho. Ismein jo bhi `server` lines hain, unke beech Nginx **Round Robin** se baantega (default, kuch alag se likhna nahi padta) |
| `server { listen 80; }` | Nginx khud kaha "khada" hai — `80` = HTTP ka default port, isliye browser me `http://localhost` se hi kaam ho jaata hai, `:80` likhna nahi padta |
| `location / { proxy_pass ... }` | Jo bhi request aaye, isko `backend_servers` group ko forward kar do |

### Load Balancing Algorithms
```nginx
upstream backend_servers {
  least_conn;              # jo server sabse kam busy hai, usko bhejo
  server backend1:3000;
  server backend2:3000;
}
```
| Algorithm | Kaam |
|---|---|
| Round Robin (default) | Baari-baari se |
| `least_conn` | Jo kam busy hai usko |
| `ip_hash` | Same client hamesha same server pe (session persistence) |

## Docker Ke Saath Nginx — Poora Setup

### Folder Structure
```
project/
├── docker-compose.yml
├── nginx/
│   └── nginx.conf
├── backend/
│   ├── Dockerfile
│   └── index.js
```

### `docker-compose.yml`
```yaml
version: "3.8"

services:
  nginx:
    image: nginx:latest
    ports:
      - "80:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - backend1
      - backend2
    networks:
      - app-network

  backend1:
    build: ./backend
    container_name: backend1
    networks:
      - app-network

  backend2:
    build: ./backend
    container_name: backend2
    networks:
      - app-network

networks:
  app-network:
    driver: bridge
```

**Kyun `image: nginx:latest`, `build` nahi**: Nginx **Docker Hub pe already ready-made** hai — apna Dockerfile likhne ki zaroorat nahi. (Rule: agar cheez pehle se duniya me bani hai — Nginx, Redis, MongoDB — to `image:` use karo. Apna khud ka code hai to `build:` use karo, kyunki Dockerfile chahiye hoga.)

### 🔑 Volume — Local Config Container Se Kaise Judti Hai

```yaml
volumes:
  - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
```

Format: `host_path : container_path : mode`

**Kya ho raha hai**: Nginx image ke andar pehle se ek **default** config file hoti hai (`/etc/nginx/nginx.conf`), jo generic hai, tumhare backends ke baare me kuch nahi jaanti. Volume mounting keh raha hai: *"jab Nginx container ke andar us file ko dhoondhe, usko meri local (laptop wali) config dikha do, original nahi."*

**Analogy**: Container ek kiraye ka ghar hai jisme pehle se ek almari (default config) hai — tum apni khud ki almari (apni config file) usi jagah rakh do.

`:ro` = **Read-Only**, safety ke liye — container ke andar se koi is file ko edit na kar sake.

**Kyun zaroori**: Bina volume ke, config change karne ke liye har baar **poori image rebuild** karni padegi. Volume se — bas local file edit karo, restart karo, naya config turant apply.

### `container_name` Kyun Zaroori Hai
```yaml
container_name: backend1
```
Nginx config me `server backend1:3000` likha tha — ye naam **match** hona chahiye. Agar naam na do, Docker random naam dega, Nginx config se match nahi hoga → **connection fail**.

### Poora Docker + Nginx Command Flow

```bash
# Start karo (build ke saath, agar code change hua ho)
docker-compose up --build

# Debug — container ke andar config verify karo
docker exec -it <nginx_container_name> cat /etc/nginx/nginx.conf

# Sab band karo
docker-compose down
```

## Load Testing — Autocannon

**Kyun chahiye**: Sirf ek request se pata nahi chalta Load Balancing ho rahi hai ya nahi — bahut saari requests ek saath bhejni padengi.

```bash
npm install -g autocannon
autocannon -c 10 -d 10 http://localhost
```
- `-c 10` → 10 connections (fake users) ek saath
- `-d 10` → 10 seconds tak test chalega

**Report padhna**:
| Stat | Matlab |
|---|---|
| Latency (Avg) | Response time — jitna kam utna better |
| Req/Sec | Per second capacity — jitna zyada utna better |
| Total errors | Fail hui requests — high ho to system overload |

**Verify karna Load Balancing ho rahi hai**: Backend code me `process.env.HOSTNAME` print karo, phir `docker-compose logs -f backend1` aur `backend2` dono dekho — requests baari-baari se dono me aani chahiye.

## `localhost:3000` vs `localhost` (Port 80) — Deep Reason

- Jab code khud `app.listen(3000)` karta hai, wahi port use hota hai — tumhe manually `:3000` likhna padta hai, kyunki ye "default" port nahi hai.
- `http://` ka **default port 80** hai — agar tum koi port na likho, browser **khud-ba-khud `:80` add** kar deta hai.
```
http://example.com  ==  http://example.com:80   (same cheez)
```
- Isliye jab humne Nginx ko `listen 80` diya, browser me sirf `localhost` likhne se hi Nginx tak pahunch gaye — koi zabardasti nahi hai `80` use karne ki, but ye **industry convention** hai taaki users ko port yaad na rakhna pade.
- **Backend servers (jo Nginx ke peeche hain)** ka port kuch bhi ho sakta hai — users unse kabhi seedha baat nahi karte.

---

# PART 4 — Monolith vs Microservices

## Monolith
> Ek single large application architecture. Frontend → One Backend → One Database. Sab modules ek hi project me.

```
users → Server1 (Auth + Product + Order sab code ek saath)
```

**Restaurant analogy**: Ek akela cook sab kuch karta hai — order leta hai, khana banata hai, bill banata hai.

## Microservices
> Application ko small independent services me divide karna.

```
Frontend → API Gateway → Auth Service
                        → Product Service
                        → Order Service
```

**Restaurant analogy**: Alag specialists — Waiter (order), Cook (khana), Cashier (bill) — apna-apna kaam.

## Monolith vs Microservices — Kab Kya

| | Monolith | Microservices |
|---|---|---|
| Shuruaat | ✅ Simple, fast | ❌ Complex setup |
| Scaling | Poori app scale karni padti | Sirf busy service scale karo |
| Deployment | Chhoti change → poori app redeploy | Sirf effected service deploy |
| Team | Chhoti team ke liye better | Badi team, alag services alag team |
| Debugging | Easy | Thoda mushkil (logs alag-alag jagah) |

**Practical advice**: Monolith se shuru karo, jab traffic/team badhe tab Microservices me todo.

---

# PART 5 — API Gateway

## Kya Hai
**Restaurant analogy**: Reception/Manager — customer ko pata nahi kitne specialists hain, sirf reception se baat karta hai. Reception decide karta hai kis specialist ko bhejna hai.

```
Customer → Reception (API Gateway)
              ↓ "order dena hai" → Waiter (Auth Service)
              ↓ "bill chahiye"    → Cashier (Order Service)
```

## Code Example
```js
const express = require("express");
const { createProxyMiddleware } = require("http-proxy-middleware");
const app = express();

app.use("/auth", createProxyMiddleware({ target: "http://auth-service:4001" }));
app.use("/products", createProxyMiddleware({ target: "http://product-service:4002" }));
app.use("/orders", createProxyMiddleware({ target: "http://order-service:4003" }));

app.listen(80, () => console.log("API Gateway running"));
```

## 🔑 API Gateway vs Nginx (Load Balancer) — Sabse Important Farak

| | API Gateway | Nginx (Load Balancer) |
|---|---|---|
| Kaam | **Alag-alag kaam** wali services ke beech route karna | **Same kaam** karne wale multiple copies ke beech baantna |
| Analogy | "Order dena hai → Waiter, bill chahiye → Cashier" | "3 Cook hain, koi bhi free ho usko do" |
| Example | `/auth` → Auth Service, `/products` → Product Service | Product Service ki 3 copies me se kisi ko |

**Dono saath use hote hain**: Gateway decide karta hai **WHICH SERVICE**, uske peeche Nginx decide karta hai **WHICH COPY** of that service (agar service horizontally scaled hai).

```
Customer → API Gateway ("kaunsi service?") → Product Service ke 3 copies
                                                    ↑
                                        Nginx Load Balancer ("kaunsi copy?")
```

---

# PART 6 — Poori Combined Architecture (Course Diagrams)

## Diagram 1 — Vertical Scaling Ki Problem
```
DNS: amazon.com → 1.2.3.4 (ek hi IP)
1 million users → ek hi server (18 CPU, 512GB RAM)
Result: crash, downtime, cost ↑
```
**DNS kya hai**: Ek "phonebook" jo naam (amazon.com) ko IP address me convert karta hai.

## Diagram 2 — Horizontal Scaling (Monolith Ki Multiple Copies)
```
DNS: amazon.com → 1.2.5.6 (Load Balancer ka IP)
Load Balancer → 3 servers (same Monolith app, har ek 2 CPU/4GB)
```

## Diagram 3 — Load Balancer + API Gateway (Poora Production Flow)
```
Users
  ↓
Load Balancer          ← "kaunsi Gateway copy khaali hai?"
  ↓
API Gateway (1 ya 2)   ← "kis Service (Auth/Product/Order) ko bhejna hai?"
  ↓
Auth Service / Product Service / Order Service
```

**Kyun 2 API Gateways**: API Gateway khud bhi ek server hai — saari requests usi se guzarti hain, isliye **wo khud bhi overload** ho sakta hai. Isliye Gateway ki bhi multiple copies banate hain, aur unke upar bhi ek Load Balancer.

**Restaurant analogy**: Restaurant itna bada ho gaya ki ek reception desk kaafi nahi — 2 reception desks khol do, aur unke aage ek "line manager" (Load Balancer) khada karo jo batata hai "Desk 1 ya Desk 2 jao."

---

# PART 7 — Database Scaling (Replication + Sharding)

## Definition
> Database ko itna powerful banana ki wo zyada users, zyada requests aur zyada data handle kar sake.

## Vertical vs Horizontal (Database Level Pe Bhi Same Concept)
```
Vertical:   2 CPU/8GB  →  8 CPU/64GB (ek hi DB, powerful bana diya)
Horizontal: 2 CPU/8GB  →  3 DBs of 2 CPU/8GB each (multiple DBs)
```

Horizontal Scaling ke 2 Main Concepts: **Replication** aur **Sharding**

## 🔹 Replication

> Same data multiple DB servers pe copy hota hai.

```
        WRITE
App ────────────→ Primary DB
                       │
              ┌────────┴────────┐
              ▼                 ▼
          Secondary         Secondary
```

**Roles**:
| Server | Kaam |
|---|---|
| Primary | Writes |
| Secondary | Reads + Backup |

**Kyun**: Read traffic ko multiple replica nodes me distribute karna — jitne zyada replicas, utna zyada **read** load handle ho sakta hai.

```
Single DB     → 10K reads/sec handle
Replica 1+2   → 5K + 5K = combined zyada read capacity
```

**Restaurant analogy**: Ek hi recipe (data) ki photocopy 3 alag counters pe rakh do — jitne zyada log "menu dekhna" (read) chahte hain, utne alag counters se dikha sakte ho. Lekin **naya order likhna (write)** hamesha ek hi jagah (Primary/Head chef) se hota hai — warna sab counters ka data mismatch ho jayega.

## 🔹 Sharding

> Data ko divide karna multiple databases me.

**Kab chahiye**: Jab **data itself bohot huge** ho jaaye — ki single DB storage bhi handle nahi kar paa raha (Replication isko solve nahi karti, kyunki Replication to **same** data ko copy karti hai, poore data ko chhota nahi karti).

**Example**:
```
Users 1-1M   → Shard 1
Users 1M-2M  → Shard 2
Users 2M-3M  → Shard 3
```

```
         Router
           │
   ┌───────┼───────┐
   ▼       ▼       ▼
Shard1   Shard2   Shard3
```

**Restaurant analogy**: Ek hi godown (database) me itna saaman aa gaya ki jagah hi kam pad gayi — ab tum data ko **3 alag godowns** me baant dete ho (Godown A me sirf "A-M" customers ka data, Godown B me "N-Z" wala) — koi ek godown poora data nahi rakhta, sab mil ke poora data hain.

**Sharding Solves**: ✅ write scaling, ✅ huge data, ✅ storage scaling

## 🔑 Replication vs Sharding — Sabse Important Comparison Table

| Feature | Replication | Sharding |
|---|---|---|
| **Purpose** | Read scaling | Write/data scaling |
| **Data** | Same copy (har jagah pura data) | Divided (har jagah data ka hissa) |
| **Storage** | Repeated | Distributed |
| **Complexity** | Easier | Harder |

**Ek line me yaad rakho**:
> *"Replication increases availability and read performance. Sharding increases storage and write scalability."*

## Real Production Setup — Dono Ek Saath (Companies Dono Combine Karti Hain)

```
App
  │
Load Balancer
  │
mongos               ← ye router hai (MongoDB ka sharding router)
  │
  ├──────────────┐
  ▼              ▼
Shard 1       Shard 2
Replica Set   Replica Set
```

**Kya ho raha hai**: Data pehle **Sharding** se multiple shards me baanta gaya (write/storage scaling ke liye), aur **har ek Shard khud bhi ek Replica Set hai** (read scaling + backup ke liye). `mongos` ek **router** hai jo decide karta hai konsi query kis Shard ko jaani chahiye.

**Restaurant analogy (final combine)**: 3 alag godowns hain (Sharding — data baant diya), aur **har godown ki khud bhi 2-3 photocopies** hain alag jagah (Replication — har shard ka apna backup/read-copies) — taaki ek godown ka ek copy kharab ho bhi jaaye, poora data safe rahe aur reads bhi fast rahen.

---

# PART 8 — Docker Fundamentals (Dockerfile + Container Deep Concepts)

## Image vs Container
- **Image** = Recipe/Blueprint (read-only template)
- **Container** = Us recipe se banaya hua **running instance**

**Restaurant analogy**: Image = training manual "cook kaise banta hai". Container = us training se actual banaya hua ek cook.

**Ek Image se Multiple Containers**: Haan bilkul ban sakte hain — ek hi recipe se 3 alag cakes (containers) ban sakte hain, sab independent.
```bash
docker run --name nginx1 nginx
docker run --name nginx2 nginx
docker run --name nginx3 nginx
```

## Dockerfile Kab Zaroori Hai, Kab Nahi

| Situation | Dockerfile Chahiye? |
|---|---|
| Standard tool (Nginx, MongoDB, Redis) | ❌ Nahi — `image: nginx` seedha Docker Hub se le lo |
| Apna khud ka code (tumhara app) | ✅ Haan — Docker ko batana padega "kaise chalana hai" |

**Golden Rule**: *"Cheez duniya me pehle se bani hai (Nginx/Redis/Mongo) → Dockerfile mat likho. Tum khud kuch naya bana rahe ho → Dockerfile likhna padega."*

## Dockerfile Likhna — 6 Sawaal Khud Se Poocho

Kisi bhi service (Auth, Product, Order, koi bhi) ka Dockerfile banane ke liye, ye 6 sawaal poocho:

| # | Sawaal | Jawab Kaha Se Milta Hai | Instruction |
|---|---|---|---|
| 1 | Kis language/runtime pe bana hai? | `package.json` dekho | `FROM` |
| 2 | Code kaha rakhna hai andar? | Convention: `/app` | `WORKDIR` |
| 3 | Dependencies kaise install hongi? | `package.json` copy + `npm install` | `COPY` + `RUN` |
| 4 | Baaki code kab jaayega? | Dependencies ke baad | `COPY . .` |
| 5 | App kis port pe chalta hai? | Apne code me `app.listen(X)` dekho | `EXPOSE` |
| 6 | Start hote hi kya chale? | `package.json` ke `scripts.start` | `CMD` |

### Example — Auth Service ka Poora Dockerfile
```dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .

EXPOSE 4001

CMD ["node", "index.js"]
```

### 🔑 `COPY package*.json` Pehle, `COPY . .` Baad Me — Kyun (Layer Caching)

```dockerfile
# ❌ GALAT order
COPY . .
RUN npm install
```
Isse har chhoti si code change pe `npm install` **dobara** chalega (waste of time).

```dockerfile
# ✅ SAHI order
COPY package*.json ./
RUN npm install          # ye layer cache ho jaata hai
COPY . .                 # sirf ye layer badlega jab code change ho
```
Agar sirf code badla (dependencies nahi), Docker `npm install` wala layer **cache se reuse** karega — rebuild seconds me, minutes me nahi.

## Multi-Stage Build (Production Best Practice)
```dockerfile
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/.next ./.next
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./package.json
EXPOSE 3000
CMD ["npm", "start"]
```
**Kyun**: Build tools (TypeScript compiler, dev dependencies) final image me nahi chahiye — sirf final output. Image halki rehti hai.

---

# PART 9 — Poora Real Scenario, End-to-End (Next.js + TS + MongoDB App)

> Scenario: "BookMyEvent" — Event Booking App. Har step **kyun, kab, kaise** ke saath.

## Step 1 — Pehle Sirf App Banao
```bash
npx create-next-app@latest bookmyevent --typescript
npm install mongoose
```
Local pe `npm run dev`, port 3000. **Docker/Redis/Nginx ka naam nahi aaya abhi** — jab tak app hi kaam nahi karta, inko lagane ka fayda nahi.

## Step 2 — Redis Kab Aata Hai (Jab "Problem" Feel Ho)
- **Problem**: `/api/events` slow hai → **Cache-Aside Pattern** yaad aata hai
- **Problem**: OTP verification chahiye → **Redis OTP storage** (`SET ... EX 300`)
- **Problem**: Login/session maintain karna hai → **Redis session storage** (JWT + `session:userId`)

**Key insight**: Redis shuru se nahi lagate — jab specific problem aaye, tabhi specific pattern use karo.

## Step 3 — App Feature-Complete → Ab Dockerize Karo
Multi-stage `Dockerfile` (upar dekha hua).

**Kyun ab, pehle nahi**: Docker ka kaam hai "jo bana hai usko pack karna" — jab tak app hi nahi bana, pack karne ko kuch nahi. Bahut early Docker lagana baar-baar rebuild karwata hai.

## Step 4 — App + Mongo + Redis Ek Saath — `docker-compose.yml`
```yaml
version: "3.8"

services:
  app:
    build: .
    ports:
      - "3000:3000"
    depends_on:
      - mongo
      - redis
    environment:
      - MONGO_URL=mongodb://mongo:27017/bookmyevent
      - REDIS_URL=redis://redis:6379
    networks:
      - app-network

  mongo:
    image: mongo
    volumes:
      - mongo-data:/data/db
    networks:
      - app-network

  redis:
    image: redis
    volumes:
      - redis-data:/data
    networks:
      - app-network

volumes:
  mongo-data:
  redis-data:

networks:
  app-network:
    driver: bridge
```

**Yaad karo concepts**: `mongodb://mongo:27017` — **service name** se connect (`localhost` nahi, kyunki same Docker network). `volumes` — data persist rahe restart ke baad bhi.

## Step 5 — Traffic Badha → Multiple App Instances + Nginx
```yaml
services:
  app1:
    build: .
    container_name: app1
    environment:
      - MONGO_URL=mongodb://mongo:27017/bookmyevent
      - REDIS_URL=redis://redis:6379
    networks: [app-network]

  app2:
    build: .
    container_name: app2
    environment:
      - MONGO_URL=mongodb://mongo:27017/bookmyevent
      - REDIS_URL=redis://redis:6379
    networks: [app-network]

  nginx:
    image: nginx:latest
    ports: ["80:80"]
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on: [app1, app2]
    networks: [app-network]

  mongo:
    image: mongo
    volumes: [mongo-data:/data/db]
    networks: [app-network]

  redis:
    image: redis
    volumes: [redis-data:/data]
    networks: [app-network]

volumes:
  mongo-data:
  redis-data:

networks:
  app-network:
    driver: bridge
```

**🔑 Bahut Important Observation**: `app1` aur `app2` **same** Redis aur **same** MongoDB use karte hain. **Kyun**: Agar har app instance ka apna alag Redis/Mongo hota, to user `app1` pe login kare aur agli request `app2` pe jaaye, session hi nahi milega! **Cache aur Database hamesha "shared/central" rehte hain — sirf application servers "multiply" hote hain.**

## Step 6 — Load Testing
```bash
autocannon -c 20 -d 10 http://localhost
docker-compose logs -f app1
docker-compose logs -f app2
```
Requests dono me baari-baari se aani chahiye — proof ki Load Balancing ho rahi hai.

## Step 7 — Deploy (Production Me Jaana)

1. Domain kharido (`bookmyevent.com`)
2. Cloud server lo (AWS EC2, DigitalOcean, Railway)
3. DNS domain ko server IP pe point karo
4. Nginx me HTTPS/SSL add karo (Let's Encrypt free certificate)
5. MongoDB/Redis — managed services use karo production me (MongoDB Atlas, Redis Cloud) — reliable, backup khud handle
6. Environment variables production server pe set karo

### Production Nginx Config
```nginx
server {
  listen 80;
  server_name bookmyevent.com;
  return 301 https://$host$request_uri;   # HTTP → HTTPS redirect
}

server {
  listen 443 ssl;
  server_name bookmyevent.com;
  ssl_certificate     /etc/nginx/ssl/bookmyevent.crt;
  ssl_certificate_key /etc/nginx/ssl/bookmyevent.key;

  location / {
    proxy_pass http://app_servers;
  }
}
```
**Naya yahan**: `443` = HTTPS ka default port (jaise 80 HTTP ka). `server_name` — agar ek server pe multiple domains hon, ye batata hai konsa rule kiske liye hai.

## Step 8 — Amazon-Scale Ho Jaaye To? (Karodo Servers Wala Case)

Humara `upstream { server backend1, server backend2 }` wala tareeka sirf **2-10 servers** tak practical hai. Amazon jaisi scale ke liye:

```
DNS (Route 53) → nearest region ka IP (geo-based)
    ↓
Cloud Load Balancer (ALB) → dynamic Target Group (servers khud register/deregister hote hain)
    ↓
Auto Scaling Group → traffic dekh ke khud servers badhata/ghatata hai
    ↓
Kubernetes Cluster → andar microservices khud replicate/load-balance hote hain
```

**Health Check** (naya concept): Load Balancer har server ko "Are you alive?" (`GET /health`) bhejta rehta hai — jawab na de to "unhealthy" maan ke traffic band, koi manual intervention nahi.

**Kubernetes ka core idea**: Tum sirf "desired state" batate ho (`replicas: 20`), system **khud** us state ko maintain karta hai — self-healing, auto load-balanced. Chhote projects me zaroorat nahi, bade scale pe zaroori.

---

# 🗺️ POORI TIMELINE — Ek Nazar Me

```
1. App banao (features)                       → koi Docker/Redis/Nginx nahi
2. Problem aayi (slow API/OTP/session)          → Redis add karo (targeted)
3. App feature-complete                          → Dockerize karo (Dockerfile)
4. App + DB + Cache ek saath                      → docker-compose.yml (1 app instance)
5. Traffic badha, ek server kam pada               → Multiple app instances + Nginx
6. Load test karo                                    → autocannon se verify
7. Deploy karo                                        → domain, HTTPS, cloud DB
8. Aur bada ho gaya                                    → Auto Scaling / Kubernetes
```

**Database ke liye alag se**:
```
1. Ek DB kaafi hai abhi                    → kuch mat karo
2. Reads slow ho rahe (bahut log padh rahe)  → Replication (Primary + Secondaries)
3. Data itself bahut bada ho gaya              → Sharding (data ko baanto)
4. Dono zaroori ho gaye (bade scale pe)          → Sharding + har Shard khud Replica Set
```

---

# 🧠 SABSE ZAROORI MENTAL MODELS (Yaad Rakhne Ke Liye)

1. **"Har technology tabhi aati hai jab uski zaroorat feel ho"** — pehle se sab kuch mat thoko.
2. **API Gateway = "WHICH SERVICE"**, **Load Balancer = "WHICH COPY"** — dono alag decisions, alag layers.
3. **Replication = same data, multiple copies (READ scaling)**. **Sharding = data divided, multiple pieces (WRITE/storage scaling)**.
4. **Cache/DB hamesha shared/central rehta hai — sirf application servers multiply hote hain.**
5. **Image = recipe, Container = us recipe se bana instance** — ek image se multiple containers ban sakte hain.
6. **Standard tool (Nginx/Redis/Mongo) → `image:` use karo. Apna code → `build:` (Dockerfile chahiye).**
7. **Port 80/443 "default" hain HTTP/HTTPS ke liye** — isliye tumhe likhna nahi padta, baaki sab ports explicitly likhne padte hain.

---

# PART 10 — Common Mistakes (Jo Log Production Me Karte Hain)

| Mistake | Kya Hota Hai | Fix |
|---|---|---|
| **Replication Lag ignore karna** | Write ke turant baad Secondary se read karo → purana data mil sakta hai (async replication) | Critical reads Primary se karwao, ya "read-your-writes" pattern |
| **Galat Shard Key choose karna** | Data unevenly distribute — kuch shards overloaded, kuch khaali (**"Hot Shard"**) | Aisi key choose karo jo evenly spread ho (jaise `user_id`, `random hash`), koi highly-skewed field nahi |
| **Load Balancer khud single point of failure** | Sirf ek Nginx instance — wo crash to poora system down | Load Balancer ki bhi multiple instances rakho (Part 6 wala combined diagram) |
| **App servers me local session store karna** | User `app1` pe login kare, agli request `app2` pe jaaye — session hi nahi milta | Sessions **Redis** (shared/central) me rakho, kabhi app server ki apni memory me nahi |
| **Sabse pehle hi Microservices bana dena** | Chhoti team, chhota traffic — lekin complexity itni ki development slow ho jaata hai | Monolith se shuru karo, jab zaroorat pade tab todo |
| **Cache Invalidation bhool jaana** | DB update hua, lekin purana cache reh gaya — users ko stale data dikhta hai | Jab bhi underlying data change ho, corresponding cache key **DEL** karna mat bhoolo |
| **Health Check na lagana** | Crashed server ko bhi Load Balancer traffic bhejta rehta hai | Regular health checks (`GET /health`) lagao, unhealthy server ko auto-remove karo |

---

# PART 11 — Failure Scenarios (Interview Me Zaroor Poochte Hain)

| Agar Ye Ho Jaaye... | Kya Hota Hai | Kaise Design Karo |
|---|---|---|
| **Primary DB crash** | Writes ruk jaate hain jab tak naya Primary na bane | Automatic failover setup karo (Replica Set with election) — ek Secondary khud naya Primary ban jaaye |
| **Ek Shard crash** | Sirf **us range ka data** unavailable, baaki system chalta rehta hai | Har Shard ko khud bhi Replica Set banao — Shard ke andar bhi failover ho jaaye |
| **Redis (Cache) down** | Agar code me fallback nahi hai, poori app crash ho sakti hai | Graceful degradation design karo — cache fail ho to seedha DB se serve karo, error na do |
| **Ek App Server crash** | Load Balancer usko "unhealthy" maan ke traffic bhejna band kar de | Health checks + kam se kam 2 app instances hamesha chalao |
| **Nginx/API Gateway crash** | Agar sirf ek instance hai, poora system unreachable | Gateway ki bhi multiple instances + upar apna Load Balancer (Part 6 diagram) |
| **Ek Microservice down (jaise Payment Service)** | Agar Order Service seedha isse baat kar rahi hai, wo bhi block ho sakti hai | **Circuit Breaker Pattern** use karo — baar-baar fail ho to turant fallback do, poori chain block na ho |

---

# PART 12 — Traffic-Scale Progression (Kab Kya Chahiye)

| Users/Traffic | Kya Kaafi Hai |
|---|---|
| **~100-1,000 users** | Monolith, ek server, ek DB — kuch aur zaroorat nahi |
| **~1,000-10,000 users** | Redis Caching add karo (slow APIs ke liye), DB Replication (read scaling) |
| **~10,000-100,000 users** | Horizontal Scaling (multiple app instances) + Nginx Load Balancer |
| **~100,000-1M users** | Microservices consider karo (agar team bhi badi ho gayi ho), Sharding (agar data bahut bada ho gaya ho) |
| **1M+ users (Amazon-scale)** | Auto Scaling Groups, Cloud Load Balancer, Kubernetes, multi-region DNS |

**Golden Rule**: Is table ko "roadmap" ki tarah mat lo — **jab tak actual problem na aaye (slow response, crash, overload), agla step mat lo.** Bahut early complexity add karna bhi ek mistake hai (dekho Part 10).

---

# 📋 ONE-PAGE CHEAT SHEET (Last-Minute Revision)

```
SCALING
  Vertical   = ek server powerful banao        (❌ expensive, ❌ hardware limit)
  Horizontal = zyada servers add karo           (✅ modern approach)

LOAD BALANCER
  Traffic ko multiple SAME servers me baantta hai
  Default algorithm: Round Robin
  Nginx = reverse proxy + load balancer (software)

API GATEWAY vs LOAD BALANCER
  Gateway  → WHICH SERVICE   (/auth, /products alag services)
  Load Bal → WHICH COPY      (Product Service ki 3 copies me se)

MONOLITH vs MICROSERVICES
  Monolith      = sab ek jagah        (simple, chhoti team)
  Microservices = alag-alag services  (complex, badi team, independent scaling)

REPLICATION vs SHARDING
  Replication = SAME data, multiple copies   → READ scaling
  Sharding    = DIVIDED data, multiple pieces → WRITE/STORAGE scaling
  Production  = dono saath (Sharded + har Shard khud Replica Set)

DOCKER
  Image = recipe (read-only)      Container = running instance
  Standard tool (Nginx/Redis/Mongo) → image: (no Dockerfile)
  Apna code                          → build: (Dockerfile chahiye)
  Dockerfile 6 sawaal: FROM(runtime) → WORKDIR(/app) → COPY package.json+RUN install
                       → COPY . . → EXPOSE(port) → CMD(start command)

PORT DEFAULTS
  HTTP  = 80   (isliye "localhost" likhna kaafi hai)
  HTTPS = 443

KEY PRINCIPLE
  Cache/DB = shared/central hamesha rehta hai
  Sirf application servers "multiply" hote hain (horizontal scaling)
```

---

*Ye poora notes tumhare System Design lecture (PDF pages 65-88) + humari deep practical discussions ka combined version hai. Revision ke liye isko baar-baar padhna, aur jahan bhi naya doubt aaye, wapas isi structure me add karte jaana.*