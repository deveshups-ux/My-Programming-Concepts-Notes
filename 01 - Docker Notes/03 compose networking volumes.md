# 3. Docker Compose + Networking + Volumes

## 3.1 Docker Compose ki zarurat kyu padi

Bina Compose ke, multiple containers manually chalane padte:

```bash
docker network create chat-network
docker run -d --name mongo-db --network chat-network -v mongo-data:/data/db mongo
docker run -d --name chatbot-app --network chat-network -p 3000:3000 my-chatbot-image
```

Lambi, alag-alag commands, yaad rakhna mushkil — especially 5+ services ho to.

**Docker Compose ek `docker-compose.yml` file me sab likh deta hai, ek hi command
(`docker compose up`) se sab chal jata hai.**

---

## 3.2 Compose file — poora example (Next.js + MongoDB + Redis)

```yaml
version: '3.8'

services:
  chatbot-app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - MONGO_URL=mongodb://mongo-db:27017/chatdb
      - OPENAI_API_KEY=${OPENAI_API_KEY}
    depends_on:
      - mongo-db
      - redis

  mongo-db:
    image: mongo
    volumes:
      - mongo-data:/data/db

  redis:
    image: redis
    ports:
      - "6379:6379"

volumes:
  mongo-data:
```

### Har part ka matlab

| Key | Matlab |
|---|---|
| `services:` | Kaunse-kaunse containers chalne hain, ek list |
| `chatbot-app:` | Service ka naam (khud rakhte ho) |
| `build: .` | "Isi folder ke Dockerfile se image banao" |
| `image: mongo` | Khud build nahi karna, Docker Hub se ready-made image utha lo |
| `ports:` | Wahi Port Mapping jo `docker run -p` me hoti hai |
| `environment:` | Env variables (API keys, DB URL) yahan daalo |
| `depends_on:` | "Pehle isko start hone do, uske baad mujhe start karna" — order matter karta hai |
| `volumes:` (service ke andar) | Data ko permanent jagah save karna |
| `volumes:` (bottom, top-level) | Named volume ko define karna zaroori hai |

**⚠️ YAML me Indentation (spacing) hi sab kuch hai** — galat spacing se poori file
error degi. Consistent **2 spaces** use karo, **tabs mat use karo**.

### Chalane ke commands
```bash
docker compose up          # foreground me chalao
docker compose up -d       # background me chalao (terminal free rahega)
docker compose down        # sab band + remove karo
```

---

## 3.3 Docker Networking — deep dive

### Kyu chahiye — basic problem

Container by default **isolated box** hota hai. 2 containers (jaise Next.js app +
MongoDB) ko **aapas me baat karni ho**, to unhe network chahiye:
1. Containers **aapas me baat kar sakein**
2. Containers **bahar ki duniya (browser/internet) se baat kar sakein**

> **Analogy:** 2 ghar (containers) hain jinke beech sadak (network) nahi hai — log
> (data) ek se dusre tak ja hi nahi sakte.

### Container ka apna IP address hota hai

Har container start hote hi ek **internal IP** milta hai (jaise `172.18.0.2`).
**Problem:** container restart ho to IP **change** ho sakta hai — agar code me IP
hardcode kiya, wo break ho jayega.

**Solution (jo Compose deta hai):** Docker automatically saari services ko ek
**common private network** me daal deta hai, aur ek **internal DNS** chalata hai jo
**service ka naam → us waqt ka sahi IP** automatically resolve kar deta hai.

```yaml
MONGO_URL=mongodb://mongo-db:27017/chatdb
```
Yahan `mongo-db` seedha **naam** hai, IP nahi — Docker khud IP dhoondh ke connect
kar deta hai, chahe IP kitni bhi baar change ho.

### 4 Network Types

| Type | Kab use karte hain | Isolation | Speed |
|---|---|---|---|
| **Bridge** (default) | Normal projects, ek hi laptop/server ke containers connect karne | ✅ Achi | Normal |
| **Host** | High performance chahiye, port mapping avoid karna ho | ❌ Kam | Fastest |
| **None** | Pure processing, koi network access nahi chahiye (security) | ✅✅ Sabse zyada | N/A |
| **Overlay** | Alag-alag physical servers ke containers connect karne (Kubernetes/Swarm) | ✅ Achi | Cross-server |

**Bridge Network detail:** Docker ek virtual switch (`docker0`) banata hai laptop
ke andar, har container ko is switch se connect karke private IP deta hai. Compose
use karne pe **custom bridge network** banta hai jisme service-name se DNS resolve
hoti hai.

**Host Network:** Container apna alag network hi nahi banata, seedha host ka
network use karta hai — port mapping (`-p`) ki zarurat nahi, but isolation khatam.

**None Network:** Container ko koi network hi nahi milta — bilkul isolated.

**Overlay Network:** Jab containers **ek laptop pe nahi, alag-alag physical
servers pe** chal rahe hon (large-scale production, Kubernetes). Ek "private
tunnel" banata hai 2 alag machines ke containers ke beech, jaise wo ek hi jagah ho.

> **Tumhare chatbot project ke liye** — sirf **Bridge** chahiye (Compose ye
> automatically deta hai). Baaki teeno abhi ke liye sirf "jaanna" wali cheez hain.

---

## 3.4 Docker Volumes — deep dive

### Problem — Container temporary hota hai

Container ke andar create hui files (jaise database ka data) — agar container
**crash ho jaye ya `docker rm` se delete ho jaye**, wo data **permanently gayab**!

> Real example: MongoDB container ko bina volume ke chalao, container delete ho
> gaya → poora database data gayab.

### Solution — Volume kya karta hai

**Volume** ek **permanent storage jagah** hai jo container ke **bahar, host machine
pe** rehti hai — Docker manage karta hai. Container is volume ko "mount" karta hai
(jaise external hard-disk plug karna). Container delete ho jaye → **data volume me
safe rehta hai**. Naya container banao, same volume attach karo → **purana data
wapas mil jaata hai**.

> **Analogy:** Container = hotel room (checkout pe sab clean ho jata hai). Volume =
> ghar ka locker (bahar rakha hai, room chahe kitni baar badlo, locker wahi ka wahi).

### Compose me syntax
```yaml
services:
  mongo-db:
    image: mongo
    volumes:
      - mongo-data:/data/db     # host-volume-naam : container-ke-andar-ka-path

volumes:
  mongo-data:                    # yahan define karna zaroori hai
```

### 3 Types of Storage

| Type | Kaam | Kab use karo |
|---|---|---|
| **Named Volume** | Docker khud manage karta hai kaha store hoga | Database ka data permanently save karna (best/recommended) |
| **Bind Mount** | Tum khud host ka path specify karte ho (`./folder:/data`) | Development ke waqt — code change turant container me reflect ho (live reload) |
| **tmpfs** | Sirf RAM me, container band hote hi gayab | Bahut temporary/sensitive data (jaise session tokens), rarely use hota hai |

### Volume commands
```bash
docker volume ls                # saare volumes dekho
docker volume rm mongo-data     # volume delete karo — data PERMANENTLY gayab
```
⚠️ **Fark yaad rakho:** `docker rm` (container delete) → data safe (volume me hai).
`docker volume rm` → data hamesha ke liye gayab.
