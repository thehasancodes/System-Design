# Docker Learning Notes — Progress So Far

> **Status:** Beginner track in progress  
> **Environment:** Windows + PowerShell + Docker Desktop (Linux containers)  
> **Current checkpoint:** Docker networking theory covered; practical networking / Compose lab is pending.

## Learning roadmap

Our planned path is:

1. Docker fundamentals and architecture
2. Run existing containers
3. Build a Node.js image with a Dockerfile
4. Image layers, build cache, and `.dockerignore`
5. Volumes and persistent data
6. Container networking
7. Docker Compose
8. Node.js API + PostgreSQL + Prisma
9. Production image practices and VPS deployment

This document covers what we've studied so far. Some practical steps are still pending because Docker Hub DNS resolution failed earlier and a later Compose command was run outside the project directory.

---

## 1. What problem does Docker solve?

Applications can fail on another computer because of different runtime versions, missing dependencies, or configuration differences.

Docker packages an application with its runtime and dependencies into an image so it can run in a more reproducible environment.

Docker does not make every environment perfectly identical: host OS, CPU architecture, networking, permissions, and configuration can still matter.

### Key terms

- **Image:** A reusable, mostly read-only template containing the application filesystem, runtime, and dependencies.
- **Container:** An isolated process environment created from an image. It has a writable container layer.
- **Dockerfile:** Instructions used to build an image.
- **Volume:** Persistent storage that can outlive a container.
- **Docker Compose:** A way to define and run multiple related services using a YAML file.

### Image vs. container

- Image = blueprint/package.
- Container = an instance created from that image.
- A container can be stopped and removed while the image remains available.

---

## 2. Docker architecture

Docker commonly uses a client-server architecture:

- **Docker CLI:** The command-line tool used by the developer.
- **Docker daemon (`dockerd`):** Manages images, containers, networks, and volumes.
- **Docker objects:** Images, containers, networks, and volumes.

Example:

```powershell
docker run nginx
```

The CLI asks the daemon to obtain the image if needed, create a container, and start it.

On Windows, Docker Desktop commonly runs Linux containers through a lightweight Linux environment using WSL 2.

---

## 3. Installation and verification

Docker Desktop was installed/launched for the initial practical lab.

Useful verification commands:

```powershell
docker --version
docker compose version
docker info
```

- `docker --version` checks the CLI version.
- `docker compose version` checks Compose availability.
- `docker info` displays engine information and helps confirm the daemon is reachable.

If `docker info` cannot connect to the daemon, make sure Docker Desktop is running.

---

## 4. First container: `hello-world`

Command:

```powershell
docker run hello-world
```

Docker downloads the image if needed, creates a container, prints a message, and exits.

Inspect images and containers:

```powershell
docker image ls
docker container ls
docker container ls -a
```

- `docker image ls` lists local images.
- `docker container ls` lists running containers.
- `docker container ls -a` lists all containers, including stopped ones.

---

## 5. Running Nginx

Command:

```powershell
docker run -d --name my-nginx -p 8080:80 nginx
```

Meaning:

- `docker run`: create and start a container.
- `-d`: run in the background.
- `--name my-nginx`: set the container name.
- `-p 8080:80`: publish host port 8080 to container port 80.
- `nginx`: image to use.

Open:

http://localhost:8080

Useful commands:

```powershell
docker ps
docker logs my-nginx
docker stop my-nginx
docker start my-nginx
docker rm -f my-nginx
```

`docker rm -f` removes the container, not the image.

### Port mapping rule

```text
-p HOST_PORT:CONTAINER_PORT
```

Example:

```powershell
docker run -d --name demo-web -p 8085:80 nginx
```

The browser connects to `localhost:8085`; Docker forwards traffic to port 80 in the container.

`EXPOSE` in a Dockerfile documents a port but does **not** publish it to the host.

---

## 6. First Node.js HTTP server

Project folder:

```powershell
mkdir docker-node-lab
cd docker-node-lab
npm init -y
```

Create `server.js`:

```javascript
const http = require("http");

const server = http.createServer((req, res) => {
  res.writeHead(200, {
    "Content-Type": "application/json"
  });

  res.end(JSON.stringify({
    message: "Hello from Docker!",
    status: "success"
  }));
});

server.listen(3000, "0.0.0.0", () => {
  console.log("Server running on port 3000");
});
```

Test locally:

```powershell
node server.js
```

Open http://localhost:3000 and stop the server with `Ctrl+C`.

### Why listen on `0.0.0.0`?

Inside a container, the server needs to listen on an interface reachable through the container's network. Listening only on the container's own loopback interface (`localhost` / `127.0.0.1`) would prevent ordinary access through the container network.

---

## 7. Dockerfile basics

Create a file named exactly `Dockerfile` in `docker-node-lab`:

```dockerfile
FROM node:22-alpine

WORKDIR /app

COPY server.js .

EXPOSE 3000

CMD ["node", "server.js"]
```

Instructions:

- `FROM node:22-alpine`: use a base image containing Node.js 22 on Alpine Linux.
- `WORKDIR /app`: set the working directory inside the image.
- `COPY server.js .`: copy the server file into `/app`.
- `EXPOSE 3000`: document the container's intended listening port.
- `CMD ["node", "server.js"]`: set the default command when a container starts.

### Build and run

Build:

```powershell
docker build -t docker-node-lab .
```

Run:

```powershell
docker run -d --name node-api -p 3000:3000 docker-node-lab
```

Open http://localhost:3000.

Inspect:

```powershell
docker images
docker ps
docker logs node-api
docker inspect node-api
```

Clean up:

```powershell
docker stop node-api
docker rm node-api
```

The image remains available after the container is removed.

### Important: build attempt failed because of DNS

The build failed while Docker tried to fetch metadata for `node:22-alpine`. The important error was:

```text
lookup registry-1.docker.io: no such host
```

This indicates Docker could not resolve Docker Hub's hostname at that time. It is a network/DNS/proxy issue, not a Dockerfile syntax error.

Suggested diagnostic commands:

```powershell
nslookup registry-1.docker.io
curl.exe -I https://registry-1.docker.io/v2/
docker pull node:22-alpine
```

An HTTP `401 Unauthorized` from the registry `/v2/` endpoint can indicate that the endpoint is reachable and asking for authentication; it is not necessarily a connectivity failure.

If Windows resolves the host but Docker still cannot pull, restart Docker Desktop, consider `wsl --shutdown` and reopen Docker Desktop, and inspect VPN, DNS, and proxy settings. Don't change network settings blindly.

**Status:** Base image download/build needs to be retried after fixing connectivity.

---

## 8. `RUN` vs. `CMD` vs. `ENTRYPOINT`

| Instruction | When it runs | Typical purpose |
|---|---|---|
| `RUN` | During image build | Install packages, create files, run build steps |
| `CMD` | When a container starts | Set the default command or arguments |
| `ENTRYPOINT` | When a container starts | Define the main executable |

Example:

```dockerfile
FROM node:22-alpine
WORKDIR /app
COPY server.js .
RUN echo "Image build completed"
CMD ["node", "server.js"]
```

The `RUN` command executes during the build. The `CMD` command specifies the default process to run when the container starts.

---

## 9. Image layers and build caching

Images are built from a sequence of filesystem changes and metadata. Many Dockerfile instructions create filesystem layers; not every instruction creates a filesystem layer.

Docker can reuse cached build steps when their inputs and relevant build conditions have not changed.

### Less efficient pattern

```dockerfile
FROM node:22-alpine
WORKDIR /app

COPY . .
RUN npm install

CMD ["node", "server.js"]
```

If application code changes, the `COPY . .` step can invalidate the cache for the following `npm install` step.

### Better dependency-cache pattern

```dockerfile
FROM node:22-alpine

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY server.js .

EXPOSE 3000

CMD ["node", "server.js"]
```

Copying package manifests before application code lets Docker reuse the dependency installation step when only `server.js` changes and dependency manifests remain unchanged.

For projects with a committed `package-lock.json`, `npm ci` is usually preferable for reproducible dependency installation.

Inspect image history:

```powershell
docker image history docker-node-lab
```

This displays build history and approximate layer sizes.

---

## 10. `.dockerignore`

Create `.dockerignore` in the project root:

```text
node_modules
.git
.env
.env.*
npm-debug.log*
coverage
dist
```

Purpose:

- `node_modules`: avoid sending local dependencies unnecessarily.
- `.git`: exclude repository history.
- `.env` and `.env.*`: reduce the chance of exposing secrets in the build context.
- `npm-debug.log*`: exclude debug logs.
- `coverage`: exclude coverage output.
- `dist`: exclude generated output when it is not needed as a build input.

`.dockerignore` reduces the build context but is not a complete security boundary. Never bake production credentials into images; use suitable secret handling for sensitive build-time data.

---

## 11. Volumes and persistence

### The problem

Files stored only in a container's writable layer disappear when that container is removed.

A Docker volume stores data separately so it can be reused by a replacement container.

### Three storage approaches

- **Container writable layer:** temporary data associated with the container.
- **Named volume:** Docker-managed persistent storage, useful for databases.
- **Bind mount:** maps a specific host file or directory into a container, useful for development source code.

A volume provides persistence, not a backup.

### Practical named-volume lab

Create a volume:

```powershell
docker volume create site-data
docker volume ls
docker volume inspect site-data
```

Start Nginx with the volume mounted:

```powershell
docker run -d --name volume-demo -p 8081:80 -v site-data:/usr/share/nginx/html nginx
```

Write a file into the volume:

```powershell
docker exec volume-demo sh -c "echo 'My persistent Docker data!' > /usr/share/nginx/html/index.html"
```

Open http://localhost:8081.

Remove the container:

```powershell
docker rm -f volume-demo
```

Recreate it with the same volume:

```powershell
docker run -d --name volume-demo -p 8081:80 -v site-data:/usr/share/nginx/html nginx
```

Refresh the browser. The message should remain because the data is stored in the named volume.

Cleanup after testing:

```powershell
docker rm -f volume-demo
docker volume rm site-data
```

**Warning:** removing the volume permanently removes its stored data. Only run the final command after the experiment is finished.

---

## 12. Docker networking

### The core idea

Containers can communicate over Docker networks. A user-defined bridge network provides name-based DNS resolution between connected containers.

Analogy:

- Your computer is outside the private network.
- Each container is like a room.
- A Docker network is the corridor between rooms.
- A published port is an entrance from the host into a container service.

### `localhost` means “this environment”

| Where `localhost` is used | What it refers to |
|---|---|
| Windows host | Windows host |
| Node.js container | That Node.js container |
| PostgreSQL container | That PostgreSQL container |

If the Node.js API and PostgreSQL run in separate containers on the same Compose network, the API should usually connect to PostgreSQL using its service name, for example:

```text
db:5432
```

Using `localhost:5432` inside the API container would point to the API container itself, not the separate database container.

### Port publishing

```powershell
docker run -d --name demo-web -p 8085:80 nginx
```

- Host port: `8085`
- Container port: `80`
- Browser URL: http://localhost:8085

Containers on the same user-defined bridge network can generally communicate directly using container/service names and listening ports. They do not need published ports for this internal communication.

Not publishing a port does not make a service universally inaccessible: connected containers and potentially the host may still reach it, depending on configuration.

### Network types introduced

- **Bridge:** common local networking for containers; user-defined bridges provide container-name DNS.
- **Host:** uses host networking where supported, with different isolation and port-publishing behavior.
- **None:** no ordinary external network connectivity.

### Manual networking lab (attempted; needs retry)

Create a network:

```powershell
docker network create lesson-network
```

Start two Nginx containers:

```powershell
docker run -d --name web-public --network lesson-network -p 8085:80 nginx
docker run -d --name web-private --network lesson-network nginx
```

The `web-public` container publishes host port 8085. The `web-private` container has no published port.

Test the public service at http://localhost:8085.

Inspect the network:

```powershell
docker network inspect lesson-network
```

### Error encountered

Command attempted:

```powershell
docker exec web-public wget -qO- http://web-private
```

Error:

```text
exec: "wget": executable file not found in $PATH
```

This does not establish that networking is broken. The Nginx image used for the container does not have `wget` available in its `PATH`.

A possible test is to run a temporary curl container on the same network:

```powershell
docker run --rm --network lesson-network curlimages/curl:latest http://web-private
```

This requires Docker Hub access if the curl image is not already available locally.

Clean up the manual lab when done:

```powershell
docker rm -f web-public web-private
docker network rm lesson-network
```

---

## 13. Docker Compose networking

### What Compose does

Docker Compose defines multiple services in one YAML file and runs them together. By default, Compose creates a shared network for the project's services.

Services can normally reach one another using their Compose service names.

### Example Compose networking lab

Create a project directory:

```powershell
mkdir docker-compose-network-lab
cd docker-compose-network-lab
```

Create `compose.yaml`:

```yaml
services:
  web:
    image: nginx
    ports:
      - "8085:80"

  private:
    image: nginx
```

Start the services:

```powershell
docker compose up -d
docker compose ps
docker network ls
```

Open http://localhost:8085.

Test Nginx configuration:

```powershell
docker compose exec web nginx -t
```

Note: `nginx -t` checks Nginx configuration; it does not test connectivity to the other container.

Test service-name networking from a temporary curl container. First find the Compose network name with `docker network ls`. It will usually resemble `docker-compose-network-lab_default`.

```powershell
docker run --rm --network docker-compose-network-lab_default curlimages/curl:latest http://private
```

Replace the network name with the actual name if it differs.

### Compose error encountered

The following command was run:

```powershell
docker compose exec web nginx -t
```

Error:

```text
no configuration file provided: not found
```

This means Compose did not find a configuration file in the current directory or supplied path. Run Compose commands from the directory containing `compose.yaml` (or explicitly provide the file with `-f`).

Useful checks:

```powershell
pwd
ls
```

Then navigate into the project directory containing `compose.yaml` and retry.

Stop and remove the Compose services and default network:

```powershell
docker compose down
```

Named volumes are normally preserved unless explicitly removed.

**Status:** Compose networking lab is pending. The directory/configuration issue must be resolved, and Docker Hub connectivity may also need fixing before images can be pulled.

---

## 14. PostgreSQL + API architecture preview

The planned architecture is a Node.js API and PostgreSQL database running as separate Compose services.

Illustrative configuration:

```yaml
services:
  api:
    build: .
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: "postgresql://app:secret@db:5432/appdb"

  db:
    image: postgres:17
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: appdb
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

This is only a preview, not yet a complete runnable project. The API needs a suitable Dockerfile, dependencies, and database connection logic. For real deployments, use secure credentials and readiness checks.

Key lessons:

- `db` is the database's service hostname on the Compose network.
- `5432` is PostgreSQL's internal listening port.
- No published database port is required for API-to-database communication.
- `pgdata` preserves database files beyond the lifetime of a container.

---

## 15. Current progress and next steps

### Studied

- [x] Docker's purpose and core vocabulary
- [x] Image vs. container
- [x] Docker CLI and daemon architecture
- [x] `hello-world`
- [x] Running and inspecting Nginx
- [x] Port mapping syntax
- [x] Writing a simple Node.js HTTP server
- [x] Dockerfile instructions: `FROM`, `WORKDIR`, `COPY`, `EXPOSE`, `CMD`
- [x] Difference between `RUN`, `CMD`, and `ENTRYPOINT`
- [x] Image layers and build caching
- [x] `.dockerignore`
- [x] Volumes and persistence (theory and lab instructions)
- [x] Docker networking concepts and service-name DNS
- [x] Compose networking concepts

### Pending / needs verification

- [ ] Fix Docker Hub DNS/connectivity and successfully build the Node.js image
- [ ] Complete the named-volume persistence lab
- [ ] Complete the manual networking lab
- [ ] Locate the Compose project directory and run the Compose networking lab
- [ ] Verify container-to-container communication using curl
- [ ] Build a complete Node.js API + PostgreSQL Compose project
- [ ] Learn environment variables, health checks, and production image practices
- [ ] Later: deploy the project to a VPS

## Quick revision

1. **Image:** reusable template.
2. **Container:** isolated running process environment created from an image.
3. **Dockerfile:** instructions for building an image.
4. **Volume:** persistent data independent of an individual container.
5. **Network:** enables containers to communicate.
6. **Port mapping:** `-p HOST_PORT:CONTAINER_PORT`.
7. **Compose service DNS:** use the service name, such as `db`, from another service on the same Compose network.
8. **`localhost`:** refers to the environment in which the client process is running.
9. **Build cache:** speeds up rebuilds when inputs to earlier build steps remain unchanged.
10. **`.dockerignore`:** excludes files from the build context.

### Resume instruction

When resuming, start by troubleshooting Docker Hub DNS resolution and finding the directory containing `compose.yaml`. Then complete the networking lab before moving on to a fully working Node.js API + PostgreSQL project with Docker Compose.
