# 1. Docker Basics & Core Concepts

## 1.1 Docker ki zarurat kyu padi — "Works on my machine" problem

Socho tumne ek Node.js backend banaya. Tumhare laptop pe:
- Node version 18 hai
- MongoDB install hai
- `.env` file me sahi values hain

Sab perfectly chal raha hai. Tumne wahi project apne dost ko diya — uske system pe error
aa gaya, kyuki uske paas alag Node version tha, Mongo install nahi tha, env variables
missing the.

Isko bolte hain **"Works on my machine" problem** — industry me ye bahut common issue tha.

**Docker isko solve karta hai:** poora application — code + runtime + dependencies +
config — sab ek "box" (**container**) me pack kar deta hai. Ye box jahan bhi le jao
(kisi bhi machine, server, cloud), andar sab same rahega, isliye app hamesha same
tarike se chalega → **"Run Anywhere"**.

> **Analogy:** Shipping container. Truck ho, ship ho, train ho — sab same design ka
> container carry karte hain. Andar chahe jo ho, container ka structure fixed hota hai.

---

## 1.2 "Docker" naam kyu pada

**Docker** (real English word) ka matlab hota hai — wo insaan jo **port/dock** pe
cargo ko ships me load-unload karta hai.

Real container shipping me: chahe andar jo bhi cargo ho, container ka size/shape
**standard/fixed** hota hai — isliye kisi bhi ship/truck/train pe easily chadh jata hai.

Docker (software) ne yehi idea copy kiya: app ko ek standard container me "load" karke
kahin bhi bhej do, wahan chala do. Isi wajah se naam **Docker** pada, aur logo bhi ek
**whale** hai jo peeth pe containers carry kar raha hai.

---

## 1.3 Image vs Container — sabse important concept

| | Image | Container |
|---|---|---|
| Kya hai | **Frozen template/blueprint** (jaise photo) | **Running/live instance** (jaise photo ko "chalu" kar diya) |
| Change hoti hai? | Nahi, fixed rehti hai | Haan, chalte waqt state change ho sakti hai |
| Kaise banti hai | Dockerfile se `docker build` karke | Image ko `docker run` karke |

> **Analogy:** Image = Recipe (likha hua hai kya banana hai).
> Container = Actual Dish (bana hua khana, jo abhi chal raha/serve ho raha hai).

`docker run` command Image se ek Container start karti hai.

---

## 1.4 Image ke andar "Layers" ka concept

Image ek **cake** ki tarah layers me bani hoti hai — har layer ek ke upar ek:

```
Layer 5: Tumhara code (app.js, package.json)
Layer 4: npm install se aayi dependencies (node_modules)
Layer 3: Node.js runtime install
Layer 2: System tools/libraries
Layer 1: Base OS (Ubuntu / Alpine)
```

Dockerfile ki **har line ek naya layer** banati hai.

**Layers kyu useful hain — Caching:**
Agar tumne sirf apna code change kiya (dependencies same rakhi), Docker **sirf last
layer ko dobara banata hai**, baaki saari layers (OS, Node, dependencies) already
**cached** hoti hain, dobara banane ki zarurat nahi.

**Fayde:**
- Build fast hota hai (baar-baar sab install nahi karna padta)
- Space bachta hai (same base layer 2 images me share ho sakti hai)

**Isliye order matter karta hai Dockerfile me:**
```dockerfile
FROM node:18          # Layer 1
COPY package.json .   # Layer 2 — pehle sirf ye copy karo
RUN npm install        # Layer 3 — dependencies install (rarely change hoti)
COPY . .               # Layer 4 — baaki sara code (frequently change hota)
CMD ["node", "app.js"] # Layer 5
```
Dependencies-related lines **upar** rakho, code wali lines **niche** — taaki jab sirf
code change ho, `npm install` dobara na chale.

---

## 1.5 Base Image kya hai

**Base Image = wo starting/foundation image jispe tum apni khud ki image banate ho.**

```dockerfile
FROM node:18-alpine   # ← ye Base Image hai
```

Tumhe apna app chalane ke liye ek basic environment chahiye (OS + Node.js already
installed). Ye khud se banana (scratch se Linux + Node install karna) time-consuming
hai — isliye Docker Hub pe **ready-made base images** available hain.

> **Analogy:** Cake banana hai. Scratch se sab kuch banao (mushkil), ya **ready-made
> cake-mix packet** lo aur apna topping (apna code) add karo. Base Image = wo
> ready-made mix.

Common base images:

| Base Image | Kab use karte hain |
|---|---|
| `node:18` | Node.js app, full version (bada size) |
| `node:18-alpine` | Node.js app, lightweight (chhota size) |
| `python:3.11` | Python app |
| `ubuntu` | Generic Linux, custom setup chahiye ho tab |
| `nginx` | Web server / static files serve karne ke liye |
| `mongo` | MongoDB database |

Base images khud bhi kisi base pe bani hoti hain (chain):
```
alpine (bare minimum Linux)
   ↓
node:18-alpine (Linux + Node.js installed)
   ↓
tumhari image (Node + tumhara code)
```

---

## 1.6 "Bare minimum" / Alpine ka matlab

**Bare minimum** = sirf sabse zaroori/basic cheezein, koi extra cheez nahi.

**Alpine** ek super lightweight Linux distro hai — sirf itna hi hai jitna OS ko
basic chalne ke liye chahiye, baaki sab hataya gaya hai.

| Image | Approx size |
|---|---|
| `node:18` | ~900 MB |
| `node:18-alpine` | ~150 MB |

**Fayde:** fast download, fast deploy, kam security risk (jitna kam software, utni kam
vulnerabilities).

**Nuksaan:** kabhi-kabhi Alpine me kuch libraries missing hoti hain jo bade distros
me hoti hain, error aa sakta hai.

---

## 1.7 Docker vs Virtual Machine (VM) — confusion clear

Bahut log sochte hain Docker = Mini VM. **Ye galat hai.**

| | Virtual Machine | Docker Container |
|---|---|---|
| Kya carry karta hai | Poora alag OS (heavy, GBs) | Sirf app + dependencies (halka, MBs) |
| Boot time | Minutes | Seconds |
| Resource use | Bahut zyada RAM/CPU | Bahut kam |

VM me poora naya OS install karna padta hai. Container **host ka hi OS kernel share**
karta hai — isliye bahut halka aur fast hai.

---

## 1.8 Multiple containers ki zarurat kyu padti hai

Chhote/personal project ke liye **1 container kaafi hai**. Multiple containers ki
zarurat tab padti hai jab app **live/production** me ho:

1. **Traffic zyada ho** — ek container limited CPU/RAM use karta hai, itna traffic
   handle nahi kar payega. Solution: same image se 5-10 containers, traffic **Load
   Balancer** se baanto.

2. **Ek container crash ho jaye** — agar sirf 1 container hai aur wo crash ho gaya,
   poori site down. Multiple containers hon to baaki chalte rehte hain
   (**High Availability**).

3. **Zero downtime deployment** — naya version deploy karte waqt, ek-ek container
   update karo (**Rolling Update**), baaki live rehte hain, user ko downtime feel
   nahi hota.

4. **Microservices (alag images se alag containers)** — Next.js app, MongoDB, Redis
   — teeno alag kaam karte hain, isliye alag containers me rakhte hain, taaki ek me
   issue ho to dusre pe asar na pade.

> **Analogy:** Chai ki dukaan — shuru me 1 banda kaafi hai. Famous ho gayi, 50
> customers aa gaye → 3-4 employees (same training/recipe = same "image") rakho,
> alag counters pe kaam karein. Ek bimar pade (crash) to baaki chalte rahein.

**Important clarification:** Ye "scaling seekhna" nahi hai, ye sirf **scaling ka
concept/reason** samajhna hai. Actual scaling (Load Balancer set up karna, auto-scale
karna) System Design aur Kubernetes wale topics me aata hai — Docker basics sirf
foundation hai.

### 2 alag use-cases hain multiple containers ke — mix mat karo

| Use case | Kitne containers | Kyu |
|---|---|---|
| **Portability** (dost ko app dena, kahin bhi deploy karna) | 1 hi kaafi | Sirf consistency chahiye ("mere system pe bhi wahi chale") |
| **Scaling** (real traffic handle karna) | Multiple (same image se) | Load baantna, crash se bachna |

Same **Image** dono jagah use hoti hai — bas use-case alag hai.

---

## 1.9 Container ka random naam kyu hota hai

`docker run` bina `--name` diye chalao to Docker khud random naam deta hai
(jaise `happy_einstein`, `zen_curie`) — kyuki har container ka **unique naam/ID**
hona zaroori hai (identify/stop/restart karne ke liye).

Naam = ek **adjective** + ek **scientist ka naam**, random combine kiya jata hai —
taaki lambi random ID (jaise `a3f5e8d9c2b1`) yaad rakhne se aasan ho.

**Khud naam dena recommended hai:**
```bash
docker run --name chatbot-app -p 5000:3000 my-chatbot-image
```

**Agar naam pehle se kisi container ne le rakha ho** (running ya stopped, dono
count hote hain):
```
Error: Conflict. The container name "/chatbot-app" is already in use...
```

**Solve karne ke 3 tarike:**
```bash
docker rm chatbot-app              # purana delete karo, phir naam reuse karo
docker start chatbot-app           # agar sirf "stopped" hai, use hi start kardo
docker run --name chatbot-app-v2   # naya alag naam de do
```

**Stopped vs Deleted — fark samjho:**
- `docker stop <name>` → container band hua, **exist karta hai**, naam reserved hai
- `docker rm <name>` → container **permanently delete**, naam free ho gaya

Check karne ke liye: `docker ps -a` (running + stopped, dono dikhayega)

---

## 1.10 Public vs Private Images/Containers

| Type | Example | Kya koi bhi dekh sakta hai? |
|---|---|---|
| **Public** (Docker Hub) | `node`, `mongo`, `nginx`, `amazon/aws-cli` | ✅ Haan |
| **Private** (company ka internal registry) | Amazon ka internal backend | ❌ Nahi, sirf employees |

Companies apna **business logic / proprietary code** kabhi public nahi karti —
usko apne **Private Registry** me rakhti hain (Amazon ka apna hai: **AWS ECR —
Elastic Container Registry**).

**Apne project ke liye kab private rakhna chahiye:**

| Situation | Rakho |
|---|---|
| API keys/secrets involved | 🔒 Private |
| Sirf demo/learning project | 🌍 Public bhi chalega |
| Real users ka data handle ho raha | 🔒 Private |
| Sirf 1-2 logo ko dena hai | 🔒 Private repo (Docker Hub/GitHub) |
| Portfolio/showcase karna hai | 🌍 Public |

⚠️ **Golden rule:** Chahe public ho ya private — **kabhi API keys/secrets Dockerfile
ya code me hardcode mat karo.** Hamesha environment variables se runtime pe pass karo.
