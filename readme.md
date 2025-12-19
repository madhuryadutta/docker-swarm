

## 🏗️ Architecture (High level)

![Image](https://abhinavcreed13.github.io/blog/docker-swarm-amazon-ec2-aws/docker3.png?utm_source=chatgpt.com)

![Image](https://miro.medium.com/v2/resize%3Afit%3A1400/1%2AZ8JHDF6mzcJstNEQknKEQQ.png?utm_source=chatgpt.com)

![Image](https://k21academy.com/wp-content/uploads/2021/08/Swarm_ArchitectureDiagram.png?utm_source=chatgpt.com)

```
Internet
   |
AWS ALB (HTTPS, SSL)
   |
Target Group (HTTP)
   |
-------------------------
| EC2 Node 1 (Manager) |
| EC2 Node 2 (Worker)  |
-------------------------
   |
Docker Swarm Overlay Network
   |
Containers / Services
```

---

## 1️⃣ AWS Infrastructure (Production setup)

### EC2

* **Instances**: 2 × EC2
* **Type**: `t3.medium` (minimum for prod)
* **OS**: Amazon Linux 2023 / Ubuntu 22.04
* **Disk**: 30–50 GB gp3
* **IAM Role**:

  * CloudWatch logs (optional)
  * ECR pull access (if using ECR)

### Security Groups

**ALB SG**

* Inbound:

  * 80 (HTTP)
  * 443 (HTTPS)
* Outbound: All

**EC2 SG**

* Inbound:

  * 80 / 8080 (from ALB SG only)
  * 2377 (Swarm management)
  * 7946 TCP/UDP (Swarm)
  * 4789 UDP (Overlay network)
* Outbound: All

⚠️ **Never expose 2377, 7946, 4789 to public internet**

---

## 2️⃣ Install Docker on both nodes

```bash
sudo yum update -y
sudo yum install docker -y
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker ec2-user
```

Logout & login again.

---

## 3️⃣ Initialize Docker Swarm

### On **Node 1 (Manager)**

```bash
docker swarm init --advertise-addr <PRIVATE_IP_NODE1>
```

You’ll get a **join token**.

### On **Node 2 (Worker)**

```bash
docker swarm join --token <TOKEN> <PRIVATE_IP_NODE1>:2377
```

Verify:

```bash
docker node ls
```

---

## 4️⃣ Create Overlay Network (mandatory)

```bash
docker network create \
  --driver overlay \
  --attachable \
  prod_net
```

---

## 5️⃣ Application Load Balancer (ALB)

### ALB Configuration

* Type: **Application Load Balancer**
* Scheme: Internet-facing
* Listener:

  * 80 → redirect to 443
  * 443 → forward to target group
* SSL:

  * Use **ACM certificate**
  * Cheapest & best option

### Target Group

* Type: **Instance**
* Protocol: HTTP
* Port: `80` (or `8080`)
* Health check:

  * Path: `/health`
  * Interval: 15s

📌 ALB will automatically load-balance between both nodes.

---

## 6️⃣ Docker Swarm Stack (Production-ready)

### `docker-stack.yml`

```yaml
version: "3.9"

services:
  app:
    image: nginx:alpine
    networks:
      - prod_net
    deploy:
      replicas: 4
      placement:
        max_replicas_per_node: 2
      restart_policy:
        condition: on-failure
      update_config:
        parallelism: 1
        delay: 10s
      resources:
        limits:
          cpus: "0.50"
          memory: 512M
    ports:
      - target: 80
        published: 80
        protocol: tcp
        mode: host

networks:
  prod_net:
    external: true
```

Deploy:

```bash
docker stack deploy -c docker-stack.yml prod
```

---

## 7️⃣ Why `mode: host` is important with ALB

ALB → EC2 **instance port**

* ALB cannot see Swarm routing mesh IPs
* `mode: host` binds container directly to EC2 port
* ALB health checks work correctly

✅ This is **recommended for AWS + ALB**

---

## 8️⃣ Zero-Downtime Deployment

```bash
docker service update \
  --image nginx:latest \
  prod_app
```

Swarm will:

* Update one container at a time
* Keep traffic alive
* Respect health checks

---

## 9️⃣ Logging & Monitoring (Production)

### Minimum

* Docker logs → CloudWatch Agent
* ALB access logs → S3
* EC2 metrics → CloudWatch

### Optional (Enterprise)

* Prometheus + Grafana
* Loki for logs

---

## 🔐 Security Best Practices

* Use **IAM roles**, not access keys
* Private subnets for EC2
* ALB in public subnet
* No SSH from public (use bastion or SSM)
* Rotate Swarm join tokens

---

## 1️⃣0️⃣ Scaling

### Scale containers

```bash
docker service scale prod_app=6
```

### Scale nodes

* Add new EC2
* Join Swarm
* ALB auto detects instance

---

## 1️⃣1️⃣ Backup & Recovery

* App data → **RDS / S3**
* Swarm manager backup:

```bash
tar czvf swarm-backup.tar.gz /var/lib/docker/swarm
```

---

## ⚖️ Is Docker Swarm + ALB Production-ready?

✅ **YES**, if:

* Small to medium SaaS
* Simple networking
* Team prefers simplicity

❌ Not ideal if:

* Need autoscaling pods
* Service mesh
* Advanced traffic routing
  → then Kubernetes (EKS) is better

---

## 🧠 Recommendation (based on your SaaS style questions)

For **MVP / early SaaS**:
✔ Docker Swarm + ALB
✔ Cheapest
✔ Simple
✔ Stable

For **enterprise scale later**:
➡️ Migrate to EKS

---

If you want, next I can:

* Add **Laravel + Swarm** real example
* Add **Caddy / Nginx inside Swarm**
* Compare **ALB vs NLB**
* Give **cost estimate (₹ / month)**

Just tell me 👍
