# Docker + Deployment — Short Notes

## 1. Docker Basics
- **Docker** = app+dependencies ko ek "container" me pack karna, taaki "works on my machine" problem na ho. Naam port pe cargo load karne wale "docker" (person) se pada.
- **Image** = frozen template (recipe). **Container** = image ka running instance (dish). `docker run` image se container banata hai.
- **Layers**: Dockerfile ki har line = layer. Unchanged layers cache hoti hain → rebuild fast. Isliye deps pehle copy karo, code baad me.
- **Base Image**: starting image (`node:18-alpine`) jispe apna code daalte ho. Alpine = bare minimum Linux, chhota/fast.
- **VM vs Docker**: VM = poora OS (heavy, mins). Container = host kernel share karta hai (halka, secs).
- **Multiple containers kyu**: Traffic zyada → load balance; crash se bachna; zero-downtime update; microservices (app/db/redis alag). 2 use-cases: Portability (1 container kaafi) vs Scaling (same image, multiple containers).
- Random container naam auto milta hai; `--name` se khud do. `docker rm` = delete, `docker stop` = paused (naam reserved).
- Public images (Docker Hub) sabko dikhti hain; Private (AWS ECR jaisa) sirf company ko. Secrets kabhi image/code me hardcode mat karo.

## 2. Dockerfile + Commands
```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm install
COPY . .
RUN npm run build
EXPOSE 3000
CMD ["npm", "start"]
```
- Python: `FROM python:3.11-slim` + `pip install -r requirements.txt` + `CMD ["python","app.py"]`
- React static: multi-stage build → final stage `FROM nginx:alpine` (Nginx = lightweight web server/reverse proxy).
- `.dockerignore`: node_modules, .git, .env, .next
- **Port mapping**: `-p HOST:CONTAINER` (e.g. `-p 8080:3000`). `EXPOSE` = sirf documentation, actual mapping `-p` hi karta hai.
- Docker Engine (local build/run) ≠ Docker Hub (online storage, jaise GitHub). Push/pull hamesha terminal se.

**Commands cheat-sheet:**
```
docker build -t name .        docker images        docker rmi name
docker pull/push name         docker login
docker run -p H:C -d --name x -e K=V image
docker ps / docker ps -a       docker logs x
docker stop/start/restart/rm x    docker exec -it x sh
docker volume ls / rm name
docker network ls / create name
docker system prune -a
docker compose up/-d/down/build
```

## 3. Compose + Networking + Volumes
```yaml
services:
  app:
    build: .
    ports: ["3000:3000"]
    environment: [MONGO_URL=mongodb://mongo-db:27017/db]
    depends_on: [mongo-db, redis]
  mongo-db:
    image: mongo
    volumes: ["mongo-data:/data/db"]
  redis:
    image: redis
volumes:
  mongo-data:
```
- Service naam se hi DNS resolve hota hai (IP yaad nahi rakhna padta) — Compose apna private network + DNS deta hai.
- **Network types**: Bridge (default, normal projects) | Host (fast, no isolation) | None (fully isolated) | Overlay (multi-server, Kubernetes).
- **Volumes**: container temporary hai, data delete ho sakta hai bina volume ke. Named Volume (best, DB data) | Bind Mount (dev, live reload) | tmpfs (RAM only, temp). `docker volume rm` = data permanently gone.

## 4. Dockerize kisi bhi App ko — Steps
1. `.dockerignore` banao 2. `Dockerfile` likho 3. Env vars runtime pe pass karo (never hardcode) 4. DB/Redis decide karo (cloud vs local container) 5. `docker build -t name .` 6. `docker run -p .../docker compose up` se test karo 7. `--name` do 8. Docker Hub push ya GitHub pe daalo 9. Production: Vercel/Render/AWS pe deploy karo.
- Code change → **image rebuild karni padti hai** (image frozen hoti hai) → naya container chalao.
- Terminology: "Docker karna"→**Dockerize**, "naya Docker banana"→**Image build**, "Docker chalu karna"→**Container run**.

## 5. Server / Cloud / AWS / Deployment
- **Server** = koi bhi program jo ON hai aur request ka wait kar raha hai (role, fixed shape nahi). `server.listen(3000)` chalte hi laptop bhi server ban jata hai.
- Ek machine me theoretically ~65k ports/servers ho sakte hain; practically RAM/CPU limit karta hai.
- **Data Center**: bade buildings, racks me hazaron server machines (no screen/battery, 24/7 bijli+backup+cooling). Office building (log laptop leke baithte) ≠ Data Center (sirf machines, kam log).
- **AWS** = Amazon apna data-center infra "rent" pe deta hai. **Cloud** = kisi aur ke server internet se use karna, bina khud kharide.
- Alag companies ke servers (Vercel/Render/Mongo Atlas) internet ke common protocol (HTTP/TCP-IP) se baat karte hain — authentication (password) + IP whitelisting se security milti hai. Sab regions paas-paas rakho for speed.
- **Vercel free tier**: limited bandwidth/requests, non-commercial only; zyada traffic (jaise 1000 users) pe Pro plan (~$20/mo+) chahiye hota hai.
- **Scaling real me**: Kubernetes = container orchestration — auto-scale, auto-heal, load balancing, rolling updates, bina insaan ke. Roadmap: Docker → Redis → System Design → Cloud Deploy → Kubernetes.

## 6. Next.js Server vs Client Component
- **Server Component** (default): server pe chalta hai, DB seedha access, sirf HTML browser ko jata hai, fast. Interaction (click/type) handle nahi kar sakta.
- **Client Component** (`"use client"`): browser me chalta hai, `useState`/`onClick` yahi chalega, interaction ke liye zaroori.
- Rule: interaction chahiye → Client; sirf data dikhana → Server (default rakho).
```javascript
// Server: async function ChatHistory(){ const m=await db.getMessages(); return <List m={m}/> }
// Client: "use client"; function Input(){ const [v,setV]=useState(""); return <input onChange={e=>setV(e.target.value)}/> }
```
- Ek hi state se related UI (jaise bg-color + button text) control karo — alag states mat banao.
