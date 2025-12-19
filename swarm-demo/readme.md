## 📁 1️⃣ Folder Structure 

```bash
swarm-demo/
├── Dockerfile
├── docker-stack.yml
├── index.html
└── default.conf
```

---

## ▶️ 6️⃣ Commands to Run (IN ORDER)

### Step 1: Build image

```bash
docker build -t swarm-demo:latest .
```

---


### Step 2: Deploy stack

```bash
docker stack deploy -c docker-stack.yml prod
```

---

## 🌐 7️⃣ Open in Browser

Open:

```
http://<EC2_PUBLIC_IP>
```

Refresh multiple times — you will see:

```
Service: prod_app
Task Name: prod_app.2
Task ID: x8k2d93k
Node ID: 9df83k2
```

👉 That confirms **load balancing is working**.

---

## 🔍 8️⃣ Useful Debug Commands

```bash
docker service ps prod_app
docker service logs prod_app
docker ps
```