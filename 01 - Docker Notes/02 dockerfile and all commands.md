# 2. Dockerfile Line-by-Line + Saari Commands

## 2.1 Next.js AI Chatbot ke liye poora Dockerfile

```dockerfile
# Line 1: Base image select karo
FROM node:18-alpine

# Line 2: Container ke andar working folder banao
WORKDIR /app

# Line 3: Sirf package files pehle copy karo (caching ke liye)
COPY package.json package-lock.json ./

# Line 4: Dependencies install karo
RUN npm install

# Line 5: Baaki sara code copy karo
COPY . .

# Line 6: Next.js production build banao
RUN npm run build

# Line 7: Kaunse port pe app chalega, documentation ke liye batao
EXPOSE 3000

# Line 8: Container start hote hi ye command chale
CMD ["npm", "start"]
```

### Har line ka reason (yaad rakhne wali table)

| Line | Kya karta hai | Kyu |
|---|---|---|
| `FROM node:18-alpine` | Base OS + Node.js installed image | Next.js ko Node chahiye |
| `WORKDIR /app` | Container ke andar ek working folder | Files organized rahein |
| `COPY package.json ...` | Sirf dependency files pehle copy | **Layer caching trick** — code change ho, deps same rahe to `npm install` dobara nahi chalega |
| `RUN npm install` | node_modules install | Project ki libraries chahiye |
| `COPY . .` | Baaki poora code copy | Actual app code andar aata hai |
| `RUN npm run build` | Production build | Next.js ko run karne se pehle build chahiye |
| `EXPOSE 3000` | Port ka documentation/hint | Batata hai app kis port pe chalega (**actual mapping nahi karta**) |
| `CMD ["npm","start"]` | Container start hote hi ye chalega | Actual app run karta hai |

---

## 2.2 Alag-alag project types ke liye pattern (universal formula)

**Pattern hamesha same rehta hai, sirf 3 cheezein change hoti hain: base image,
install command, start command.**

### Plain Node/Express backend (bina build step ke)
```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package.json ./
RUN npm install
COPY . .
EXPOSE 5000
CMD ["node", "index.js"]      # seedha file run, build step nahi
```

### Python/Flask project
```dockerfile
FROM python:3.11-slim          # Node ki jagah Python base image
WORKDIR /app
COPY requirements.txt ./
RUN pip install -r requirements.txt   # npm install ki jagah pip
COPY . .
EXPOSE 5000
CMD ["python", "app.py"]
```

### React (static frontend) — Multi-stage build
```dockerfile
FROM node:18-alpine AS build
WORKDIR /app
COPY package.json ./
RUN npm install
COPY . .
RUN npm run build

FROM nginx:alpine               # final me sirf Nginx, Node nahi
COPY --from=build /app/build /usr/share/nginx/html
EXPOSE 80
```

**Multi-stage build kya hai:** ek "temporary" heavy image me build karo (build
tools + source code), phir **sirf final output** ko ek naye chhote image me copy
karo, pehli heavy image discard ho jaati hai. Final image chhoti/clean banti hai —
production me bahut use hota hai.

**Nginx kya hai:** Ek **web server** — static files (HTML/CSS/JS) fast serve karta
hai, reverse proxy/load balancing bhi karta hai. React build ke baad sirf static
files bachte hain, unhe chalane ke liye Node.js zarurat nahi — Nginx **halka aur
fast** hai isliye use hota hai. Next.js me Nginx use nahi karte kyuki Next.js
server-side rendering bhi karta hai, isliye usko Node runtime chahiye hi hota hai.

---

## 2.3 `.dockerignore` file — mat bhoolna

`.gitignore` jaisi hi hoti hai, Docker ke liye:

```
node_modules
.git
.env
.next
```

Agar ye file nahi banayi, to `COPY . .` karte waqt `node_modules`, `.git`, `.env`
sab image ke andar copy ho jayenge — image ka size bina wajah bada hoga, aur
`.env` jaisi sensitive file image ke andar chali jayegi (**security risk!**).

---

## 2.4 Port Mapping — deep explanation

Container apna **isolated box** hai — jaise ek **locked room**. Andar app port 3000
pe chal raha ho, tumhara laptop (host) us port ko **directly nahi dekh sakta**, jab
tak koi "darwaza" (port mapping) na khula ho.

```bash
docker run -p 3000:3000 my-chatbot-image
```

```
-p  HOST_PORT : CONTAINER_PORT
```

- **Left (Host Port)** → tumhare laptop ka port, jo browser me type karoge
- **Right (Container Port)** → container ke andar jis port pe app chal raha hai

**Agar ports alag rakhne ho** (jaise 3000 already busy hai laptop pe):
```bash
docker run -p 8080:3000 my-chatbot-image
```
Browser me `localhost:8080` khologe → andar container ke port 3000 pe forward hoga.

> **Analogy:** Hotel building (host) me Room 3000 (container) hai. Bahar se log
> seedha room number nahi jaante, unhe Reception (host port) pe aana padta hai, jo
> sahi room tak bhej deta hai.

### `EXPOSE` vs `-p` — bahut important fark

| | Kaam |
|---|---|
| `EXPOSE 3000` (Dockerfile me) | Sirf **documentation/hint** — batata hai app kis port pe chalta hai. Actually port open **nahi** karta |
| `-p HOST:CONTAINER` (docker run me) | **Actually** host aur container ke beech connection banata hai |

Bahar se access karne ke liye `docker run` karte waqt `-p` flag **zaroor** dena
padega, warna EXPOSE hone ke bawajood browser se access nahi hoga.

---

## 2.5 Docker Engine vs Docker Hub — do alag cheezein

| | Docker (App/Engine) | Docker Hub (Website) |
|---|---|---|
| Kya hai | Software jo local machine pe chalta hai | Cloud storage/website — Images ka "GitHub" |
| Kaam | Build, run, manage (local) | Store, share, download (online) |
| Kaise use karte ho | Terminal / Docker Desktop | Browser (dekhne) + Terminal (push/pull) |

Actual push/pull/build/run **hamesha terminal (ya Docker Desktop app) se hi hota
hai**, website sirf **browse/dekhne** ke liye hai — jaise GitHub.com pe code dekh
sakte ho par edit VS Code/terminal se karte ho.

**Poora flow:**
```
1. Terminal → docker build -t my-chatbot .        (image banao)
2. Terminal → docker login                        (Docker Hub me login)
3. Terminal → docker push username/my-chatbot      (upload karo)
4. Website  → check karo image upload ho gayi
5. Dost ka terminal → docker pull username/my-chatbot   (download kare)
```

---

## 2.6 Saari zaroori Docker Commands — ek jagah (cheat-sheet)

### Image se related
```bash
docker build -t my-image-name .        # current folder ke Dockerfile se image banao
docker images                          # saari local images dekho
docker rmi my-image-name               # image delete karo
docker pull mongo                      # Docker Hub se image download karo
docker push username/my-image          # apni image Docker Hub pe upload karo
docker login                           # Docker Hub me login karo (push se pehle)
```

### Container se related
```bash
docker run my-image                          # image se container chalao
docker run -p 3000:3000 my-image              # port mapping ke sath chalao
docker run -d my-image                        # background me chalao (detached)
docker run --name chatbot-app my-image        # custom naam do
docker run -e KEY=value my-image              # env variable pass karo runtime pe

docker ps                              # sirf RUNNING containers dikhao
docker ps -a                           # RUNNING + STOPPED, sab dikhao
docker stop chatbot-app                # container band karo (delete nahi)
docker start chatbot-app               # stopped container wapas chalao
docker restart chatbot-app             # restart karo
docker rm chatbot-app                  # container permanently delete karo
docker logs chatbot-app                # container ke logs dekho (debugging ke liye)
docker exec -it chatbot-app sh         # running container ke andar terminal kholo
```

### Volumes se related
```bash
docker volume ls                       # saare volumes dekho
docker volume rm mongo-data            # volume delete karo (data permanently gayab!)
```

### Networking se related
```bash
docker network ls                      # saare networks dekho
docker network create my-network       # naya network banao
```

### Cleanup / Maintenance
```bash
docker system prune -a                 # saari unused images/containers clean karo
                                        # (space bachane ke liye, but careful — permanent delete)
```

### Docker Compose (detail file 03 me)
```bash
docker compose up                      # saari services start karo (foreground)
docker compose up -d                   # background me start karo
docker compose down                    # saari services band + remove karo
docker compose build                   # images rebuild karo
```

---

## 2.7 Quick decision table — "kab kaunsi command"

| Zarurat | Command |
|---|---|
| Naya image banana hai | `docker build -t name .` |
| Image chalani hai (test karni hai) | `docker run -p host:container name` |
| Kaunse containers chal rahe hain dekhna | `docker ps` |
| Purane/stopped containers bhi dekhne hain | `docker ps -a` |
| Container ke andar kya ho raha hai (errors) | `docker logs <name>` |
| Container band karna hai (data safe rakhte hue) | `docker stop <name>` |
| Container hamesha ke liye hatana hai | `docker rm <name>` |
| Docker Hub pe apni image daalni hai | `docker push username/name` |
| Kisi ki ready image chahiye | `docker pull name` |
| Poora system clean karna hai (space) | `docker system prune -a` |
