
# Docker Cheat Sheet

A quick-reference guide covering the core concepts and commands from the Docker course: containers vs VMs, images, layers, volumes, networking, Docker Compose, and how it all compares to Kubernetes and cloud deployments.

---

## 1. Containers vs Virtual Machines

| | **Virtual Machine** | **Container** |
|---|---|---|
| Isolation | Full OS per VM (hypervisor) | Process-level isolation (namespaces) |
| Boot time | Minutes | Seconds |
| Size | GBs (full OS) | MBs (shares host kernel) |
| Resource usage | Heavy | Lightweight |
| Portability | Lower | High |

- A **container is just a process** running on the host, isolated using Linux **namespaces** (PID, NET, MNT, UTS, IPC, USER) and resource-limited by **cgroups**.
- Containers share the host's kernel — there's no guest OS to boot.

---

## 2. Basic Container Commands

```bash
docker ps                  # List running containers
docker ps -a                # List all containers (including stopped)
docker container ls         # Same as `docker ps` (alias)
docker container ls -a      # Same as `docker ps -a`

docker start <container>    # Start a stopped container
docker stop <container>     # Stop a running container
docker restart <container>  # Restart a container
docker rm <container>       # Remove a container
docker rm -f <container>    # Force remove a running container

docker logs <container>     # View container logs
docker logs -f <container>  # Follow logs in real time
docker exec -it <container> bash   # Open a shell inside a running container
```

---

## 3. `docker run` — More Than Just "Running"

```bash
docker run <image>
docker run -d <image>              # Detached mode (background)
docker run -it <image> bash        # Interactive with terminal
docker run --name my-app <image>   # Custom container name
docker run -p 8080:80 <image>      # Port mapping (host:container)
docker run -e VAR=value <image>    # Environment variable
docker run --rm <image>            # Auto-remove container on exit
```

**What `docker run` actually does:**
1. Checks if the image exists locally.
2. If not, **pulls it from Docker Hub** (or the configured registry) — downloading only the **layers** it doesn't already have.
3. Creates a new container from the image.
4. Executes the container's default (or overridden) command.

---

## 4. Images & Layers

- Docker images are built in **layers**, each representing a change (a Dockerfile instruction).
- Layers are **cached and reused** across images — if two images share a base layer, Docker won't download/store it twice.

```bash
docker images               # List local images
docker history <image>      # Show the layers that make up an image
docker rmi <image>          # Remove an image
docker build -t name:tag .  # Build an image from a Dockerfile
docker pull <image>         # Download an image without running it
docker push <image>         # Push an image to a registry
```

**Dockerfile basics:**
```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY . .
RUN npm install
EXPOSE 3000
CMD ["node", "index.js"]
```

---

## 5. Volumes — Data Persistence

When containers are removed, their internal filesystem changes are lost. **Volumes** solve this by persisting data outside the container's writable layer.

```bash
docker volume create my-volume
docker volume ls
docker volume inspect my-volume
docker volume rm my-volume

# Mounting a named volume
docker run -v my-volume:/app/data <image>

# Bind mount (host path <-> container path)
docker run -v /host/path:/container/path <image>
```

| Type | Description |
|---|---|
| **Named volume** | Managed by Docker, stored in Docker's storage area |
| **Bind mount** | Maps a specific host directory/file into the container |
| **tmpfs mount** | Stored in host memory only (not persisted to disk) |

---

## 6. Networking

```bash
docker network ls                        # List networks
docker network create my-network         # Create a custom network
docker network inspect my-network        # Inspect a network
docker network connect my-network <c>    # Connect a container to a network
docker network rm my-network             # Remove a network
```

**Network drivers:**
- `bridge` (default) — isolated network on a single host, containers can talk to each other via container name (DNS).
- `host` — container shares the host's network stack directly.
- `none` — no networking.
- `overlay` — connects containers across multiple Docker hosts (used in Swarm/multi-host setups).

> Containers on the same custom bridge network can reach each other **by container name** — Docker provides built-in DNS resolution.

---

## 7. Docker Compose — Multi-Container Coordination

Compose lets you define and run multi-container applications with a single YAML file.

```yaml
version: "3.8"
services:
  app:
    build: .
    ports:
      - "3000:3000"
    depends_on:
      - db
    networks:
      - app-net

  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: secret
    volumes:
      - db-data:/var/lib/postgresql/data
    networks:
      - app-net

volumes:
  db-data:

networks:
  app-net:
  driver: bridge
```

```bash
docker compose up              # Start all services
docker compose up -d           # Start in detached mode
docker compose up --build      # Rebuild images before starting
docker compose down            # Stop and remove all containers, networks
docker compose down -v         # Also remove volumes
docker compose ps              # List running services
docker compose logs -f         # Follow logs of all services
```

- `docker compose up` starts **all containers at once**, respecting `depends_on` order and connecting them via a shared network automatically.
- `docker compose down` tears everything down with **one single command** — no need to stop/remove containers manually one by one.

---

## 8. Kubernetes vs Docker Swarm

Both are **container orchestrators** — they manage multi-container, multi-host deployments, scaling, and self-healing. Compose is single-host; these tools go multi-host/cluster.

| | **Docker Swarm** | **Kubernetes (K8s)** |
|---|---|---|
| Setup complexity | Simple, built into Docker CLI | Steeper learning curve |
| Scaling | Manual/basic auto-scaling | Powerful, fine-grained auto-scaling (HPA/VPA) |
| Networking | Simple overlay networking | Complex but flexible (CNI plugins) |
| Self-healing | Yes (basic) | Yes (advanced, more configurable) |
| Load balancing | Built-in | Built-in (Services, Ingress) |
| Config management | Docker Compose-like files | YAML manifests (Deployments, Pods, Services) |
| Ecosystem | Smaller | Massive (Helm, Operators, service mesh, etc.) |
| Industry adoption | Declining | De facto standard |

**When to use which:**
- **Swarm** — small teams, simpler deployments.
- **Kubernetes** — production-grade, large-scale systems, when you need advanced scheduling, auto-scaling, multi-cloud portability, and a rich ecosystem.

```bash
# Swarm basics
docker swarm init
docker service create --name web --replicas 3 -p 80:80 nginx
docker service scale web=5
docker stack deploy -c docker-compose.yml mystack
```

```bash
# Kubernetes basics
kubectl apply -f deployment.yaml
kubectl get pods
kubectl scale deployment web --replicas=5
kubectl get services
```

---

## 9. Horizontal vs Vertical Scaling

| | **Vertical Scaling (Scale Up)** | **Horizontal Scaling (Scale Out)** |
|---|---|---|
| Method | Add more CPU/RAM to an existing instance | Add more instances/containers |
| Limit | Hardware ceiling of a single machine | Practically limitless (add more nodes) |
| Downtime | Often requires restart | Can be done with zero downtime |
| Cost model | Fewer, more powerful machines | Many, smaller machines |
| Fault tolerance | Single point of failure | Distributed — more resilient |
| Container fit | Increase container resource limits (`--cpus`, `--memory`) | Increase replica count (`docker service scale`, `kubectl scale`) |

```bash
# Vertical example: limit resources per container
docker run --cpus="2" --memory="1g" <image>

# Horizontal example: replicate containers
docker service scale web=10
kubectl scale deployment web --replicas=10
```

> In containerized/cloud-native systems, **horizontal scaling** is generally preferred — it aligns with how orchestrators (Swarm/K8s) and cloud auto-scaling groups are designed to work.

---

## 10. Docker & Cloud Services

Containers are the standard unit of deployment across major cloud providers:

| Cloud | Container Services |
|---|---|
| **AWS** | ECS (Elastic Container Service), EKS (managed Kubernetes), Fargate (serverless containers) |
| **Azure** | AKS (Azure Kubernetes Service), Container Instances, App Service (containers) |
| **GCP** | GKE (Google Kubernetes Engine), Cloud Run (serverless containers) |

**Typical cloud-native workflow:**
1. Build image locally → `docker build`
2. Push to a registry → `docker push` (Docker Hub, ECR, ACR, GCR, GitHub Container Registry)
3. Deploy to a managed orchestrator (EKS/GKE/AKS) or serverless container platform (Fargate/Cloud Run)
4. Orchestrator handles **scaling** (horizontal), **load balancing**, **self-healing**, and **rolling updates**

**Why containers fit cloud environments well:**
- Consistent runtime across dev, staging, and production ("works on my machine" → works everywhere).
- Fast startup enables efficient **auto-scaling**.
- Immutable images simplify rollbacks and CI/CD pipelines.
- Orchestrators abstract away individual servers, enabling elastic, on-demand infrastructure.

---

## 📌 Quick Command Reference

```bash
# Images
docker build -t name:tag .
docker images
docker history <image>
docker rmi <image>

# Containers
docker run -d -p 8080:80 --name app <image>
docker ps -a
docker exec -it app bash
docker logs -f app
docker rm -f app

# Volumes
docker volume create data
docker run -v data:/app/data <image>

# Networks
docker network create my-net
docker run --network my-net <image>

# Compose
docker compose up -d
docker compose down -v

# Swarm
docker swarm init
docker service scale web=5

# Kubernetes
kubectl apply -f deployment.yaml
kubectl scale deployment web --replicas=5
```

---
