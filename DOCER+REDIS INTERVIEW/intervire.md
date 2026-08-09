# 🐳🔴 DOCKER + REDIS — Connection Guide + Interview Question Bank

---

# SECTION A — Docker + Redis: Kaun Kis Se, Kab, Kaise Connect Hota Hai

## A1. Sabse Simple Tareeka — Standalone Redis Container

Jab tumhe sirf Redis chahiye (bina Docker Compose ke), seedha run karo:

```bash
docker run -d --name my-redis -p 6379:6379 redis
```

**Breakdown:**
- `-d` → detached mode (background me chalega)
- `--name my-redis` → container ko naam diya
- `-p 6379:6379` → **host ka port : container ka port** — matlab tumhare laptop ka 6379 → container ke andar wale Redis ke 6379 se map ho gaya
- `redis` → Docker Hub se official Redis image download karke chalayega

Check karo:
```bash
docker ps                    # dekho container chal raha hai
docker exec -it my-redis redis-cli   # container ke andar jaake redis-cli chalao
```

**Iske baad tumhara Node.js app (jo Docker ke bahar, seedha tumhare laptop pe chal raha hai) connect karega:**
```js
const redis = new Redis("redis://localhost:6379");
```

**Kyun `localhost` chalega yahan?** Kyunki `-p 6379:6379` ne container ka port tumhare host machine pe **expose** kar diya hai — isliye bahar se `localhost:6379` pe hi Redis mil jayega.

---

## A2. Docker Compose — Jab Poori App (Node + Redis) Dono Container Me Ho

Ye **real-world setup** hai — jahan tumhara Node.js backend bhi Docker me hai, aur Redis bhi Docker me hai, dono saath chalte hain.

### `docker-compose.yml`

```yaml
version: "3.8"

services:
  app:
    build: .
    container_name: node-app
    ports:
      - "3000:3000"
    depends_on:
      - redis
    environment:
      - REDIS_URL=redis://redis:6379
    networks:
      - my-network

  redis:
    image: redis
    container_name: redis-container
    ports:
      - "6379:6379"
    networks:
      - my-network

networks:
  my-network:
    driver: bridge
```

### 🔑 Sabse Important Cheez Yahan Samjho

**`REDIS_URL=redis://redis:6379`** — dekho, yahan `localhost` nahi likha, **`redis`** likha hai (jo service ka naam hai upar `services:` ke andar).

**Kyun?** Jab dono containers (`app` aur `redis`) **same Docker network** (`my-network`) me hote hain, Docker apna internal **DNS** provide karta hai — jisme **service ka naam hi hostname ban jaata hai**. Matlab `app` container ke andar se `redis` likhne se Docker khud-ba-khud us naam ko `redis` container ke internal IP pe resolve kar deta hai.

**Agar galti se `localhost` likh do to kya hoga?**
- `app` container ke andar `localhost` ka matlab hoga **`app` container khud** — na ki `redis` container. Isliye connection **fail** ho jayega, kyunki `app` container ke andar koi Redis chal hi nahi raha.

**Yaad rakhne ka tareeka**: 
> "Docker Compose me service-to-service baat karne ke liye **service ka naam** use hota hai (`redis://redis:6379`), `localhost` sirf tab chalega jab tum apne khud ke laptop se (Docker ke bahar se) container se baat kar rahe ho."

### `Dockerfile` (Node app ke liye)

```dockerfile
FROM node:18

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .

EXPOSE 3000

CMD ["node", "index.js"]
```

### Node.js code me Redis connect karna (env variable se)

```js
import Redis from "ioredis";

// Docker Compose se milega REDIS_URL env variable
const redis = new Redis(process.env.REDIS_URL || "redis://localhost:6379");

export default redis;
```

**Kyun `process.env.REDIS_URL || "redis://localhost:6379"` aise likha?**
- Agar Docker me chal raha hai → `REDIS_URL` env variable milega Compose file se → `redis://redis:6379` use hoga
- Agar tum local pe (bina Docker ke) directly `node index.js` chalate ho → env variable nahi milega → fallback `localhost:6379` use hoga

Ye **ek hi code, dono jagah (local + Docker) chalne** ke liye best practice hai.

### Run karna

```bash
docker-compose up --build
```

**Kya hota hai internally:**
1. Docker `redis` image se ek container banata hai
2. Docker `app` ke liye `Dockerfile` se image build karta hai, container banata hai
3. `depends_on: redis` — matlab Docker pehle `redis` container start karega, phir `app` container
4. Dono ek hi network (`my-network`) me aa jaate hain → ek dusre se naam se baat kar sakte hain

---

## A3. Data Persist Karna — Volumes (Important!)

**Problem**: Agar Redis container delete/restart ho gaya, to andar ka saara data (cache, sessions, etc.) **gayab** ho jaata hai — kyunki containers by default **ephemeral** (temporary) hote hain.

**Solution — Volume use karo:**

```yaml
services:
  redis:
    image: redis
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data          # Redis apna data /data folder me rakhta hai andar
    command: redis-server --appendonly yes   # persistence enable karna zaroori hai

volumes:
  redis-data:
```

**Kyun ye zaroori hai**: `redis-data:/data` ka matlab hai Docker ek **named volume** banayega jo container ke bahar (host machine pe) physically store hoga. Container delete ho bhi jaaye, volume bacha rahega — naye container me wahi volume mount karo to data wapas mil jayega.

`--appendonly yes` — Redis ko batana ki wo apne changes ek log file (AOF) me likhta rahe, taaki restart pe data recover ho sake.

---

## A4. Poora Real-World `docker-compose.yml` (Node + Redis + MongoDB)

```yaml
version: "3.8"

services:
  app:
    build: .
    ports:
      - "3000:3000"
    depends_on:
      - redis
      - mongo
    environment:
      - REDIS_URL=redis://redis:6379
      - MONGO_URL=mongodb://mongo:27017/mydb
    networks:
      - backend-net

  redis:
    image: redis
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    networks:
      - backend-net

  mongo:
    image: mongo
    ports:
      - "27017:27017"
    volumes:
      - mongo-data:/data/db
    networks:
      - backend-net

volumes:
  redis-data:
  mongo-data:

networks:
  backend-net:
    driver: bridge
```

**Poori picture**: 3 containers (`app`, `redis`, `mongo`) same network pe. `app` andar se `redis://redis:6379` aur `mongodb://mongo:27017` se dono se baat karta hai — sab service-name se, `localhost` kahin nahi.

---

## A5. Quick Reference — Kab Kya Use Karo

| Situation | Redis URL kya hoga |
|---|---|
| Node.js app tumhare laptop pe (bina Docker), Redis Docker container me | `redis://localhost:6379` (port mapped hai) |
| Dono (Node app + Redis) alag-alag Docker containers me, same Compose file | `redis://<service-name>:6379` (jaise `redis://redis:6379`) |
| Redis Cloud/managed service use kar rahe ho (production) | `redis://username:password@cloud-host:port` |
| Redis khud tumhare laptop pe (bina Docker) chal raha hai | `redis://localhost:6379` |

---

# SECTION B — DOCKER Interview Questions

## 🟢 Easy Level

**Q1. Docker kya hai aur ye kyun use hota hai?**
> **Answer kaise do**: Docker ek **containerization tool** hai jo application ko uske dependencies (libraries, runtime, config) ke saath ek **container** me pack kar deta hai. Isse "mere system pe chal raha tha" wali problem khatam ho jaati hai — kyunki container har jagah (dev, staging, production) same environment carry karta hai.

**Q2. Docker Image aur Docker Container me kya farak hai?**
> **Image** = ek **blueprint/template** (read-only) — jisme app code + dependencies + instructions hoti hain.
> **Container** = image ka **running instance** — jaise class (image) aur object (container) ka relation hota hai OOP me.

**Q3. `Dockerfile` kya hota hai?**
> Ek text file jisme **step-by-step instructions** likhi hoti hain ki image kaise banegi — base image kaunsi, kya install karna hai, code kaha copy karna hai, konsa command run karna hai container start hote hi.

**Q4. `docker run` aur `docker start` me kya farak hai?**
> `docker run` = naya container **banata hai** image se, aur usko start karta hai.
> `docker start` = pehle se bane **existing (stopped) container** ko phir se start karta hai.

**Q5. `docker ps` kya karta hai?**
> Currently **running containers** ki list dikhata hai. `docker ps -a` sabhi containers dikhata hai (stopped bhi).

**Q6. `EXPOSE` instruction Dockerfile me kya karta hai?**
> Ye **documentation purpose** ke liye batata hai ki container kis port pe listen karega. Actual port mapping `docker run -p` se hoti hai — `EXPOSE` khud port ko host pe open nahi karta.

**Q7. Docker Hub kya hai?**
> Ek **public registry** (jaise GitHub, but images ke liye) jahan se pre-built images (redis, mongo, node, etc.) download kar sakte ho, aur apni images upload bhi kar sakte ho.

---

## 🟡 Medium Level

**Q8. `COPY` aur `ADD` me kya farak hai Dockerfile me?**
> Dono files ko host se image me copy karte hain. `ADD` extra features deta hai — jaise URL se download karna, ya `.tar` file ko automatically extract karna. **Best practice**: jab tak extra feature na chahiye ho, `COPY` hi use karo (predictable behavior).

**Q9. Docker Volume kya hai aur kyun zaroori hai?**
> Containers by default **ephemeral** hote hain — delete hone pe andar ka data gayab ho jaata hai. **Volume** ek mechanism hai jisse data ko container ke bahar (host pe) persist kiya jaata hai, taaki container delete/restart ho, data safe rahe. Databases (Redis, MongoDB) ke liye critical hai.

**Q10. Docker Network kaise kaam karta hai, aur containers ek dusre se kaise baat karte hain?**
> Docker apna internal network banata hai (`bridge` default hota hai). Jab tum Docker Compose use karte ho, saare services ek hi custom network me aa jaate hain, aur Docker **internal DNS** provide karta hai — jisse container apne **service name** se ek dusre ko dhoondh (resolve) sakte hain, IP address yaad rakhne ki zaroorat nahi.

**Q11. `docker-compose.yml` me `depends_on` kya karta hai — kya ye guarantee deta hai ki dependent service "ready" hai?**
> `depends_on` sirf **start order** control karta hai (pehle `redis` start hoga, phir `app`) — ye **guarantee nahi deta** ki Redis andar se fully "ready to accept connections" hai. Isliye production me `healthcheck` use karte hain ye ensure karne ke liye.

**Q12. Multi-stage build kya hota hai, aur ye kyun use karte hain?**
> Ek Dockerfile me **multiple `FROM` stages** hote hain — pehla stage build/compile karta hai (heavy tools ke saath), doosra stage sirf final output ko copy karta hai ek **chhoti, clean image** me. Isse final image ka size bahut kam ho jaata hai (production ke liye bahut important).
```dockerfile
# Stage 1: Build
FROM node:18 AS builder
WORKDIR /app
COPY . .
RUN npm install && npm run build

# Stage 2: Production
FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
CMD ["node", "dist/index.js"]
```

**Q13. `docker-compose up` aur `docker-compose up -d` me farak?**
> `-d` (detached) — containers background me chalenge, terminal free rahega. Bina `-d` ke, logs terminal me live dikhte rahenge aur terminal us process se attached rahega.

**Q14. Environment variables Docker me kaise pass karte hain?**
> 3 tareeke: (1) Dockerfile me `ENV` instruction, (2) `docker run -e KEY=value`, (3) Compose file me `environment:` block ya `.env` file se.

---

## 🔴 Hard Level

**Q15. Docker container aur Virtual Machine (VM) me kya fundamental difference hai?**
> **VM** apna poora **guest OS** chalata hai (hypervisor ke upar) — heavy, slow boot, GBs me size.
> **Container** host OS ka **kernel share** karta hai, sirf application + dependencies alag rehte hain (namespaces + cgroups se isolated) — lightweight, seconds me start, MBs me size. Isliye Docker itna fast aur efficient hai VM ke comparison me.

**Q16. Docker layers kaise kaam karte hain, aur caching kaise optimize karte hain?**
> Har Dockerfile instruction (`RUN`, `COPY`, etc.) ek naya **layer** banata hai, jo cache hota hai. Agar koi layer change nahi hua (jaise `package.json` same hai), Docker us layer ko **cache se reuse** karta hai — rebuild fast ho jaata hai.
> **Optimization trick**: `package.json` ko pehle `COPY` karo aur `npm install` chalao, **phir** baaki code copy karo — isse jab bhi sirf code change ho (dependencies nahi), `npm install` wala layer cache se hi mil jayega, dobara install nahi hoga.
```dockerfile
COPY package*.json ./
RUN npm install
COPY . .          # ye baad me, taaki upar wala cache rahe
```

**Q17. Docker me container ka data kaise `restart` ya `crash` hone ke baad bhi persist hota hai — samjhao internals ke saath?**
> Do tareeke: **Volumes** (Docker-managed storage, host filesystem pe rehta hai, container life-cycle se independent) aur **Bind Mounts** (host ka specific folder directly container ke andar map kar dena). Redis/DB jaise services ke liye Volumes best practice hain kyunki Docker khud manage karta hai, portable hota hai.

**Q18. Agar Redis container crash ho jaaye Docker Compose me, kya poori app crash ho jayegi? Kaise handle karoge?**
> Depends on code — agar Node app me Redis connection error properly handle nahi kiya, to app crash ho sakta hai. **Best practice**: retry logic lagao (`ioredis` khud retry karta hai by default), aur graceful degradation design karo (jaise agar cache fail ho, seedha DB se serve karo, poori app down mat karo). `restart: always` bhi Compose me lagate hain taaki Docker khud crashed container ko restart kar de.

**Q19. `docker-compose.yml` me `networks` explicitly define karna kyun zaroori/best practice hai, jab Compose khud-ba-khud ek default network bana deta hai?**
> Default network sab kuch ek saath rakh deta hai — koi isolation nahi. Explicit networks se tum **segment** kar sakte ho (jaise `frontend-net` aur `backend-net` alag), taaki security-sensitive services (jaise DB) sirf unhi services se baat kar paayein jinhe zaroorat hai, poori app se nahi.

---

# SECTION C — REDIS Interview Questions

## 🟢 Easy Level

**Q1. Redis kya hai?**
> Redis (**RE**mote **DI**ctionary **S**erver) ek **in-memory key-value data store** hai jo super-fast read/write ke liye use hota hai — mainly caching, session storage, aur real-time data ke liye.

**Q2. Redis itna fast kyun hai?**
> Kyunki data **RAM** me store hota hai, disk pe nahi (jaise traditional databases karte hain). RAM access disk access se **hazaaron guna fast** hota hai.

**Q3. Redis ka default port kya hai?**
> `6379`

**Q4. `SET` aur `GET` commands kya karte hain?**
> `SET key value` — ek key ke against value store karta hai. `GET key` — us key ki value return karta hai.

**Q5. Redis me data kaise expire karte hain?**
> `EXPIRE key seconds` command se, ya `SET key value EX seconds` se ek saath set + expiry.

**Q6. Redis single-threaded hai ya multi-threaded?**
> Core command execution **single-threaded** hai (ek time pe ek hi command process hota hai) — ye Redis ko simple aur race-condition-free banata hai. (Newer versions me kuch I/O operations multi-threaded hain, but core logic single-threaded hi hai.)

---

## 🟡 Medium Level

**Q7. Redis ke main data types kaunse hain?**
> - **String** — simple key-value (`SET`/`GET`)
> - **Hash** — object jaisa (fields ke andar fields) — `HSET`, `HGET`
> - **List** — ordered list (queue/stack jaisa) — `LPUSH`, `RPUSH`
> - **Set** — unique unordered values — `SADD`
> - **Sorted Set** — set + score (ranking/leaderboard ke liye) — `ZADD`

**Q8. Redis Caching me "Cache Invalidation" kya hota hai aur ye zaroori kyun hai?**
> Jab underlying data (DB me) change hota hai, lekin Redis me purana (stale) data reh jaata hai. **Cache Invalidation** matlab us purane cache ko manually delete karna jab data update ho, taaki agli request fresh data DB se le aur naya cache bane. Isko na karne se users ko **galat/purana data** dikhta reh sakta hai.

**Q9. Redis Persistence kya hai — RDB aur AOF me farak?**
> Redis by default sirf RAM me data rakhta hai — restart hote hi sab gayab. Persistence se disk pe bhi save hota hai:
> - **RDB (Snapshot)**: periodic intervals pe poora dataset ek file me snapshot leta hai — fast restart, but beech ka data (last snapshot ke baad) restart pe lost ho sakta hai.
> - **AOF (Append Only File)**: har write operation ko ek log file me likhta hai — zyada durable (kam data loss), lekin file size bada ho sakta hai, thoda slower.

**Q10. Redis me "Eviction Policy" kya hoti hai?**
> Jab Redis ki RAM full ho jaaye, to naya data store karne ke liye purana data hatana padta hai — kaunsa hataye, ye **eviction policy** decide karti hai. Common policies:
> - `noeviction` — naya data reject kar dega, error dega
> - `allkeys-lru` — sabse **kam recently used** key hatao
> - `volatile-lru` — sirf un keys me se LRU hatao jinme expiry set hai
> - `allkeys-random` — random key hatao

**Q11. Redis Pub/Sub kya hota hai?**
> Ek messaging pattern — ek client kisi "channel" pe message **publish** karta hai, doosre clients us channel ko **subscribe** karke real-time message receive karte hain. Chat apps, live notifications ke liye use hota hai. (Limitation: agar subscriber offline hai, message miss ho jaata hai — persistent nahi hota jaise queue me hota hai.)

**Q12. Redis me `EXPIRE` set karne ke baad agar tum `SET` se dobara same key pe value update karo, to expiry ka kya hota hai?**
> Plain `SET key newvalue` (bina EX ke) purani expiry **hata deta hai** — key permanent ho jaati hai. Agar expiry retain karni hai, to `SET key value EX seconds` dobara likhna padega, ya `KEEPTTL` option use karo (`SET key value KEEPTTL`).

---

## 🔴 Hard Level

**Q13. Redis me Race Condition kaise handle karte ho, jab multiple requests same key ko simultaneously modify kar rahi ho?**
> Redis commands khud **atomic** hote hain (jaise `INCR`), isliye simple counter operations me race condition nahi hoti. Complex multi-step operations ke liye:
> - **Transactions** (`MULTI`/`EXEC`) — multiple commands ko ek atomic block me group karna
> - **Lua Scripting** (`EVAL`) — poora logic ek script me likh do, Redis usko atomically execute karega
> - **Optimistic Locking** (`WATCH`) — key ko watch karo, agar beech me koi aur change kar de to transaction fail ho jayega, retry karo

**Q14. Redis Cluster kya hai aur ye scaling kaise achieve karta hai?**
> Jab ek Redis instance ki RAM/traffic capacity kam pad jaaye, **Redis Cluster** data ko multiple nodes me automatically **shard** (baant) kar deta hai — har node data ka ek hissa (hash slot range) handle karta hai. Isse horizontal scaling milti hai — single machine ki limit se bahar nikal sakte ho.

**Q15. Redis Sentinel kya hai?**
> High-availability ke liye — Sentinel process Redis master-replica setup ko **monitor** karta hai. Agar master node down ho jaaye, Sentinel automatically ek replica ko naya master bana deta hai (**automatic failover**) — bina manual intervention ke.

**Q16. Cache Stampede (Thundering Herd) problem kya hai, aur kaise solve karte ho?**
> Jab ek popular cache key **expire** ho jaati hai, aur us waqt **hazaaron requests ek saath** aati hain — sab ek saath cache miss dekhkar DB ko hit karte hain, jisse DB overload ho sakta hai. **Solutions:**
> - **Locking**: pehli request "lock" le le, DB se data laaye, baaki requests wait karein ya thoda purana data serve karein
> - **Probabilistic early expiration**: expiry se thoda pehle hi randomly refresh kar do (sab ek saath expire na ho)
> - **Stale-while-revalidate**: purana (thoda stale) data turant serve karo, background me fresh karo

**Q17. Redis me Memory Leak kaise ho sakta hai, aur kaise avoid karte ho?**
> Agar keys **bina expiry ke** (`EXPIRE` set kiye bina) continuously banti rahein (jaise per-user unique keys, per-session data), RAM dheere-dheere bharti jaati hai aur kabhi clean nahi hoti — ye ek memory leak jaisa hi hai. **Avoid karne ke liye**: har temporary/cache key pe hamesha `EXPIRE`/`EX` lagao, aur monitoring rakho (`INFO memory` command se RAM usage check karte raho).

**Q18. Redis Cache-Aside vs Write-Through vs Write-Behind caching strategies me farak?**
> - **Cache-Aside** (jo humne padha): App khud cache check karta hai, miss pe DB se laata hai aur cache me save karta hai. Simple, most common.
> - **Write-Through**: Har write DB **aur** cache dono me ek saath hoti hai (synchronously) — cache hamesha fresh rehta hai, but writes thodi slow.
> - **Write-Behind (Write-Back)**: Write pehle cache me hoti hai, DB me **baad me async** (background me) sync hota hai — writes bahut fast, but crash hone pe data loss ka risk.

---

# SECTION D — Combined / Scenario-Based Questions (Interview me Bahut Common)

**Q1. "Tumne apni app me Redis kaha-kaha use kiya, aur kyun?"**
> **Kaise answer do**: Apna khud ka project example do — "Maine ek e-commerce app banayi thi jisme product listing API baar-baar hit ho rahi thi, to maine usko 60 sec ke liye Redis me cache kiya jisse response time 200ms se 5ms ho gaya. Iske alawa OTP verification ke liye bhi Redis use kiya kyunki auto-expiry chahiye thi."
> *(Real project example dena interviewer ko sabse zyada convince karta hai — generic answer se bachna)*

**Q2. "Agar Redis down ho jaaye production me, tumhari app kya karegi?"**
> Ye tumhara **system design maturity** test karta hai. Best answer: "Maine fallback design kiya hota — agar Redis unavailable ho, seedha DB se serve karta (thoda slow but working), aur error ko log/monitor karta. Poori app crash nahi hoti sirf isliye ki cache layer down hai." *(Graceful degradation ka concept dikhana important hai)*

**Q3. "Docker Compose me tumne Redis ka data persist kaise kiya production ke liye?"**
> Volumes ka concept explain karo (Section A3 dekho) — `redis-data:/data` + `--appendonly yes` (AOF persistence). Bata sakte ho ki tumne is se restart ke baad bhi cache/session data safe rakha.

**Q4. "Tum Redis aur MongoDB dono use kar rahe ho ek hi project me — kaunsa data kaha jaata hai, decide kaise karte ho?"**
> **Rule of thumb**: Jo data **permanent/source-of-truth** hai aur complex relations/queries chahiye — DB (MongoDB/SQL) me. Jo data **temporary, fast-access, ya frequently-read** hai — Redis me. Jaise: User profile → MongoDB (permanent). Session token, OTP, rate-limit counter, cached API response → Redis (temporary + fast).

---

# SECTION E — Interview Me Kaise Answer Dena Hai (General Tips)

1. **Definition + Analogy + Code** — teeno cheez do, sirf definition mat ratt lo. Interviewer ko lagna chahiye tumne actually samjha hai, ratta nahi maara.
2. **"Maine apne project me..."** wale examples zaroor daalo jahan possible ho — real experience dikhana bahut zyada impress karta hai.
3. **Trade-offs bolna seekho** — jaise "AOF zyada durable hai but RDB fast hai" — ye dikhata hai tumhe **depth** hai, sirf surface-level pata nahi.
4. **Hard questions me "main soch ke bataunga" mat bolo** — jo pata hai wahi confidently bolo, jo nahi pata usme bhi apni **reasoning** dikhao ("Mera guess ye hoga ki... kyunki...") — interviewers reasoning process dekhna chahte hain, sirf answer nahi.
5. **System design context me hamesha "scale" aur "failure" ka case socho** — "agar traffic badh jaaye to?", "agar ye service down ho jaaye to?" — in dono sawalon ka jawab tayyar rakho har concept ke liye.

---

*Practice karne ka best tareeka: Ye saare questions khud se, bina answer dekhe, bolke (loud) answer karne ki koshish karo — jaha atko, wahi wapas padho. Fir doosre din phir se try karo bina dekhe.*