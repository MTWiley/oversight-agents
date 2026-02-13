# Container Best Practices

Reference checklist for the `devops-reviewer` agent when evaluating Dockerfiles, Docker Compose files, Kubernetes manifests, and container supply chain practices. This file complements the inline criteria in `review-devops.md` with deeper guidance, severity classifications, common mistakes, and example correct configurations.

---

## 1. Dockerfile Quality

**Category**: Dockerfile Quality

A well-written Dockerfile produces small, fast, cacheable, and reproducible images. A poorly written one produces bloated images that are slow to build, slow to deploy, and contain unnecessary attack surface.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| Multi-stage builds separate build dependencies from runtime | MEDIUM | Build tools, compilers, and dev dependencies should not ship in production images |
| Base image is minimal and appropriate (alpine, distroless, slim) | MEDIUM | Smaller images have fewer vulnerabilities and faster pull times |
| Base image version is pinned (not `:latest`) | MEDIUM | `:latest` is mutable; builds become non-reproducible |
| Base image is pinned to a digest for critical production images | LOW | Digest pinning is maximum reproducibility; version tags are acceptable for most cases |
| `RUN` instructions are combined where appropriate to reduce layer count | LOW | Excess layers increase image size due to intermediate filesystem state |
| `.dockerignore` is present and excludes `.git`, `node_modules`, build artifacts, tests, docs | MEDIUM | Missing `.dockerignore` copies unnecessary files, bloating the image and leaking information |
| `COPY` instructions are ordered for cache efficiency (dependencies before source) | LOW | Copying source code before dependencies invalidates the cache on every code change |
| `HEALTHCHECK` instruction is defined | MEDIUM | Without `HEALTHCHECK`, Docker and orchestrators cannot detect unhealthy containers |
| `EXPOSE` documents the correct ports | LOW | Documentation value; does not affect security but aids discoverability |
| `WORKDIR` is set explicitly (not relying on image default) | LOW | Explicit working directory prevents ambiguity |
| Build args are not used for secrets | HIGH | Build args are visible in image history via `docker history` |

### Common Mistakes

**Mistake**: Single-stage build that ships build tools in production.

```dockerfile
# BAD - Go compiler and all build tools in production image
FROM golang:1.22
WORKDIR /app
COPY . .
RUN go build -o /app/server
CMD ["/app/server"]
```

```dockerfile
# GOOD - multi-stage build
FROM golang:1.22 AS build
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /server

FROM gcr.io/distroless/static-debian12
COPY --from=build /server /server
CMD ["/server"]
```

**Mistake**: Using `:latest` or no tag.

```dockerfile
# BAD
FROM node:latest
FROM python

# GOOD
FROM node:20.11.0-alpine3.19
FROM python:3.12.1-slim-bookworm
```

**Mistake**: Poor layer caching due to `COPY . .` before dependency installation.

```dockerfile
# BAD - every source change invalidates the npm install cache
FROM node:20.11.0-alpine3.19
WORKDIR /app
COPY . .
RUN npm ci
CMD ["node", "server.js"]

# GOOD - dependencies cached unless package files change
FROM node:20.11.0-alpine3.19
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
CMD ["node", "server.js"]
```

**Mistake**: Missing `.dockerignore`.

```
# .dockerignore
.git
.gitignore
node_modules
npm-debug.log
Dockerfile
docker-compose*.yml
.env
.env.*
*.md
tests/
docs/
.vscode/
.idea/
coverage/
```

**Mistake**: Passing secrets as build arguments.

```dockerfile
# BAD - secret visible in docker history
ARG NPM_TOKEN
RUN echo "//registry.npmjs.org/:_authToken=${NPM_TOKEN}" > .npmrc
RUN npm ci
RUN rm .npmrc

# GOOD - use BuildKit secrets (never stored in image layers)
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc npm ci
```

Build command: `docker build --secret id=npmrc,src=$HOME/.npmrc .`

### Example Correct Configuration

**Production Node.js Dockerfile**:

```dockerfile
FROM node:20.11.0-alpine3.19 AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --production=false
COPY . .
RUN npm run build
RUN npm prune --production

FROM node:20.11.0-alpine3.19
RUN apk add --no-cache tini
WORKDIR /app
COPY --from=build /app/node_modules ./node_modules
COPY --from=build /app/dist ./dist
COPY --from=build /app/package.json ./

USER node
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1
ENTRYPOINT ["tini", "--"]
CMD ["node", "dist/server.js"]
```

---

## 2. Container Security

**Category**: Container Security

Containers are not a security boundary by default. Running as root, with full capabilities, and a writable filesystem gives an attacker inside the container nearly the same power as an attacker on the host.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| Container runs as a non-root user (`USER` instruction present and not `USER root`) | HIGH | Root in container is root on host in many configurations |
| If root is required during build, `USER` switches to non-root before `CMD`/`ENTRYPOINT` | HIGH | Build steps may need root; runtime must not |
| Capabilities are dropped (`--cap-drop ALL`) and only required ones added back | MEDIUM | Default capabilities include more than most apps need |
| Filesystem is read-only where possible | MEDIUM | Prevents an attacker from modifying application files |
| No secrets baked into the image (no `ENV SECRET=...`, no credentials in files) | CRITICAL | Image layers are extractable; secrets persist in history |
| `COPY` is used instead of `ADD` (unless extracting a tar archive) | LOW | `ADD` has implicit URL download and tar extraction that can be surprising |
| Image scanning is part of the CI pipeline (Trivy, Grype, Snyk) | HIGH | Base images and dependencies may contain known CVEs |
| Base image is from a trusted source (official images, verified publishers) | MEDIUM | Untrusted base images may contain malware |
| No `--privileged` flag in Docker run commands or compose files | CRITICAL | Privileged containers have full host access |
| No sensitive host paths mounted (Docker socket, `/etc/`, host PID namespace) | CRITICAL | Docker socket mount gives full control of the host |

### Common Mistakes

**Mistake**: No `USER` instruction -- container runs as root.

```dockerfile
# BAD - runs as root by default
FROM python:3.12-slim
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt
CMD ["python", "app.py"]

# GOOD - runs as non-root user
FROM python:3.12-slim
RUN groupadd -r appuser && useradd -r -g appuser -d /app -s /sbin/nologin appuser
WORKDIR /app
COPY --chown=appuser:appuser . .
RUN pip install -r requirements.txt
USER appuser
CMD ["python", "app.py"]
```

**Mistake**: Secrets in environment variables or files baked into the image.

```dockerfile
# BAD - secret persists in image layers
ENV DATABASE_URL=postgres://admin:s3cret@db:5432/prod
COPY credentials.json /app/credentials.json
```

Secrets must be injected at runtime via environment variables, mounted secrets, or a secrets manager.

**Mistake**: Using `ADD` when `COPY` would suffice.

```dockerfile
# BAD - ADD can fetch URLs and auto-extract archives unexpectedly
ADD https://example.com/config.tar.gz /app/

# GOOD - explicit and predictable
COPY config/ /app/config/

# ADD is acceptable when extracting a local tar archive
ADD rootfs.tar.gz /
```

**Mistake**: Mounting the Docker socket.

```yaml
# BAD - container can control the host via Docker API
volumes:
  - /var/run/docker.sock:/var/run/docker.sock
```

If Docker-in-Docker is genuinely needed, use `docker:dind` with proper isolation. For CI, prefer rootless Docker or Kaniko for image builds.

### Example Correct Configuration

**Secure Docker run**:

```bash
docker run \
  --read-only \
  --cap-drop ALL \
  --cap-add NET_BIND_SERVICE \
  --security-opt no-new-privileges:true \
  --tmpfs /tmp:rw,noexec,nosuid \
  --user 1000:1000 \
  myapp:v1.2.3
```

---

## 3. Docker Compose

**Category**: Docker Compose

Docker Compose files define multi-container environments. Missing restart policies, resource limits, and health checks lead to environments that crash and stay crashed, consume unbounded resources, or start services before their dependencies are ready.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| Restart policies are defined (`restart: unless-stopped` or `restart: on-failure`) | MEDIUM | Without a restart policy, crashed containers stay down |
| Resource limits are set (`mem_limit`/`deploy.resources.limits`) | MEDIUM | Unbounded resource usage can take down the host |
| Networks are explicitly defined (not using default bridge) | LOW | Explicit networks improve isolation and documentation |
| Service dependencies use `depends_on` with `condition: service_healthy` | MEDIUM | `depends_on` without a condition only waits for container start, not readiness |
| Volumes are used for persistent data | MEDIUM | Container filesystem is ephemeral; data is lost on recreation |
| Named volumes are preferred over bind mounts for data persistence | LOW | Named volumes are managed by Docker and more portable |
| No `privileged: true` on services | CRITICAL | Full host access |
| Environment variables for secrets use `.env` files that are gitignored, or external secret management | HIGH | Secrets in `docker-compose.yml` are committed to version control |
| Image tags are pinned (not `:latest`) | MEDIUM | Reproducibility |
| Logging driver is configured for production use | LOW | Default `json-file` driver can fill the disk without log rotation |

### Common Mistakes

**Mistake**: `depends_on` without health check condition.

```yaml
# BAD - web starts as soon as db container starts, not when db is ready
services:
  db:
    image: postgres:16
  web:
    depends_on:
      - db

# GOOD - web waits for db to be healthy
services:
  db:
    image: postgres:16
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
  web:
    depends_on:
      db:
        condition: service_healthy
```

**Mistake**: No resource limits.

```yaml
# BAD - no limits, a memory leak takes down the host
services:
  app:
    image: myapp:1.0

# GOOD - resource limits set
services:
  app:
    image: myapp:1.0
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: "1.0"
        reservations:
          memory: 256M
          cpus: "0.25"
```

**Mistake**: Secrets hardcoded in compose file.

```yaml
# BAD - secrets in version control
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: my-secret-password

# GOOD - secrets from .env file (gitignored) or Docker secrets
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    # Or using Docker secrets:
    secrets:
      - db_password
secrets:
  db_password:
    file: ./secrets/db_password.txt  # Not committed to git
```

### Example Correct Configuration

**Production-grade Docker Compose**:

```yaml
services:
  app:
    image: myapp:1.2.3
    restart: unless-stopped
    read_only: true
    tmpfs:
      - /tmp
    security_opt:
      - no-new-privileges:true
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: "1.0"
    environment:
      - DATABASE_URL=${DATABASE_URL}
      - LOG_LEVEL=info
    depends_on:
      db:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 10s
    networks:
      - frontend
      - backend
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

  db:
    image: postgres:16.1
    restart: unless-stopped
    deploy:
      resources:
        limits:
          memory: 1G
    volumes:
      - db-data:/var/lib/postgresql/data
    environment:
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - backend

volumes:
  db-data:

networks:
  frontend:
  backend:
```

---

## 4. Kubernetes Manifests

**Category**: Kubernetes Config

Kubernetes provides powerful security and reliability primitives, but they are all opt-in. An unconfigured Kubernetes deployment runs as root, with no resource limits, no health checks, and no network restrictions -- which is worse than bare Docker in many ways because it is in a shared cluster.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| Resource requests AND limits are set for CPU and memory | HIGH | Missing limits allow a single pod to starve the node |
| Readiness probe is defined | HIGH | Without readiness, traffic is sent to pods that are not ready |
| Liveness probe is defined | MEDIUM | Without liveness, a deadlocked pod stays in rotation |
| Startup probe is defined for slow-starting applications | LOW | Prevents liveness probe from killing a container that is still starting |
| `runAsNonRoot: true` in security context | HIGH | Root in container is a security risk |
| `readOnlyRootFilesystem: true` in security context | MEDIUM | Prevents attackers from modifying container filesystem |
| `allowPrivilegeEscalation: false` in security context | HIGH | Prevents `setuid` binaries from gaining root |
| `capabilities.drop: ["ALL"]` in security context | MEDIUM | Default Linux capabilities include more than most apps need |
| `NetworkPolicy` resources are defined | MEDIUM | Default Kubernetes networking allows all pod-to-pod communication |
| `PodDisruptionBudget` is defined for critical workloads | MEDIUM | Without PDB, cluster operations can take down all replicas |
| Labels are consistent and sufficient (app, version, component, part-of) | LOW | Labels enable service discovery, monitoring, and management |
| `imagePullPolicy` is set appropriately (`Always` for mutable tags, `IfNotPresent` for immutable) | LOW | Wrong pull policy causes either stale images or unnecessary pulls |
| Image references use a registry and are not just a bare name | MEDIUM | Bare names default to Docker Hub; explicit registry prevents confusion |
| Replicas are greater than 1 for production workloads | MEDIUM | Single replica means zero availability during pod disruption |
| Pod Security Standards (or PodSecurityPolicy for older clusters) are applied | MEDIUM | Cluster-level enforcement prevents misconfigurations |
| No `hostNetwork`, `hostPID`, or `hostIPC` without justification | HIGH | Host namespace access breaks container isolation |
| Secrets are not in plain YAML; use SealedSecrets, SOPS, or external secret operators | HIGH | Plain Kubernetes secrets are base64-encoded, not encrypted |
| Service accounts are not using the default account with auto-mounted token | MEDIUM | Default service account may have more permissions than needed |

### Common Mistakes

**Mistake**: No resource requests or limits.

```yaml
# BAD - no resource management
containers:
  - name: app
    image: myapp:1.0

# GOOD - explicit resource management
containers:
  - name: app
    image: myapp:1.0
    resources:
      requests:
        memory: "128Mi"
        cpu: "100m"
      limits:
        memory: "256Mi"
        cpu: "500m"
```

**Mistake**: No probes defined.

```yaml
# BAD - Kubernetes has no visibility into application health

# GOOD - readiness, liveness, and startup probes
containers:
  - name: app
    image: myapp:1.0
    readinessProbe:
      httpGet:
        path: /health/ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 10
    livenessProbe:
      httpGet:
        path: /health/live
        port: 8080
      initialDelaySeconds: 15
      periodSeconds: 20
    startupProbe:
      httpGet:
        path: /health/live
        port: 8080
      failureThreshold: 30
      periodSeconds: 10
```

**Mistake**: No security context.

```yaml
# BAD - runs as root with full capabilities

# GOOD - restricted security context
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  runAsGroup: 1000
  fsGroup: 1000
containers:
  - name: app
    image: myapp:1.0
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
          - ALL
    volumeMounts:
      - name: tmp
        mountPath: /tmp
volumes:
  - name: tmp
    emptyDir: {}
```

**Mistake**: No NetworkPolicy -- all pods can communicate with all other pods.

```yaml
# GOOD - restrict ingress to app pods from specific sources
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: app-netpol
spec:
  podSelector:
    matchLabels:
      app: myapp
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - port: 8080
          protocol: TCP
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: database
      ports:
        - port: 5432
          protocol: TCP
    - to:  # Allow DNS
        - namespaceSelector: {}
      ports:
        - port: 53
          protocol: UDP
        - port: 53
          protocol: TCP
```

**Mistake**: Secrets in plain YAML.

```yaml
# BAD - base64 is encoding, not encryption
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
data:
  password: cGFzc3dvcmQxMjM=  # "password123" - trivially decodable
```

Use SealedSecrets, SOPS, External Secrets Operator, or Vault to manage Kubernetes secrets securely.

### Example Correct Configuration

**Production Kubernetes Deployment**:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  labels:
    app.kubernetes.io/name: myapp
    app.kubernetes.io/version: "1.2.3"
    app.kubernetes.io/component: api
    app.kubernetes.io/part-of: myplatform
    app.kubernetes.io/managed-by: helm
spec:
  replicas: 3
  selector:
    matchLabels:
      app.kubernetes.io/name: myapp
  template:
    metadata:
      labels:
        app.kubernetes.io/name: myapp
        app.kubernetes.io/version: "1.2.3"
    spec:
      serviceAccountName: myapp
      automountServiceAccountToken: false
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 1000
        fsGroup: 1000
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: myapp
          image: registry.example.com/myapp:1.2.3
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8080
              name: http
              protocol: TCP
          env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: myapp-db
                  key: password
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "500m"
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop:
                - ALL
          readinessProbe:
            httpGet:
              path: /health/ready
              port: http
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /health/live
              port: http
            initialDelaySeconds: 15
            periodSeconds: 20
          volumeMounts:
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: tmp
          emptyDir:
            sizeLimit: 100Mi
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app.kubernetes.io/name: myapp
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app.kubernetes.io/name: myapp
```

---

## 5. Registry and Supply Chain

**Category**: Container Security

The container supply chain -- from base image selection to production registry -- is a high-value attack vector. Compromised images, unsigned artifacts, and unscanned dependencies are all paths to running malicious code in production.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| A private registry is used for application images (ECR, GCR, ACR, Harbor, Artifactory) | MEDIUM | Public registries expose internal images; private registries provide access control |
| Image signing is implemented (cosign, Notary) | MEDIUM | Unsigned images cannot be verified as legitimate |
| Admission controllers enforce image signing or registry source (Kyverno, OPA Gatekeeper, Connaisseur) | MEDIUM | Enforcement prevents deploying unsigned or untrusted images |
| Vulnerability scanning runs on images before deployment | HIGH | Known CVEs in images are a primary attack vector |
| Vulnerability scanning runs on a schedule for deployed images | MEDIUM | New CVEs are discovered after images are deployed |
| Tag immutability is enabled on the registry | MEDIUM | Mutable tags allow an attacker to replace a known-good image |
| Base images are from trusted sources (official, verified publisher, internal golden images) | MEDIUM | Untrusted base images are the foundation of every layer above them |
| Image pull secrets are properly configured in Kubernetes | MEDIUM | Misconfigured pull secrets prevent deployments and may expose credentials |
| Old and vulnerable images are cleaned up from the registry | LOW | Stale images with known CVEs should not be available for deployment |
| SBOM (Software Bill of Materials) is generated and stored | LOW | Supply chain transparency; increasingly required by regulation |

### Common Mistakes

**Mistake**: Using Docker Hub public images without verification for production infrastructure.

Why it is a problem: Docker Hub public images can be submitted by anyone. While official images are curated, community images may contain malware, cryptominers, or vulnerable software. For production, prefer verified publisher images, official images, or maintain internal golden images.

**Mistake**: No image signing -- any image pushed to the registry is deployable.

Why it is a problem: Without signing, a compromised CI pipeline credential or registry account can push a malicious image that looks identical to a legitimate one. Signing creates a verifiable chain from source to deployment.

**Mistake**: Registry allows tag overwriting.

Why it is a problem: If an attacker can overwrite `myapp:v1.2.3` in the registry, every new deployment or pod restart pulls the compromised image. Tag immutability ensures that once a tag is pushed, it cannot be changed.

```bash
# AWS ECR - enable tag immutability
aws ecr put-image-tag-mutability \
  --repository-name myapp \
  --image-tag-mutability IMMUTABLE
```

**Mistake**: No admission control on what images can be deployed.

Why it is a problem: Without admission control, anyone with `kubectl` access can deploy an image from any registry, including untrusted public registries or personal accounts.

### Fix Patterns

**Image signing and verification workflow**:

```yaml
# CI: Sign after build
- name: Sign image with cosign
  run: cosign sign --yes ${REGISTRY}/myapp:${GITHUB_SHA}

# Kubernetes: Enforce signature verification with Kyverno
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-image-signature
spec:
  validationFailureAction: Enforce
  rules:
    - name: verify-cosign
      match:
        any:
          - resources:
              kinds:
                - Pod
      verifyImages:
        - imageReferences:
            - "registry.example.com/*"
          attestors:
            - count: 1
              entries:
                - keyless:
                    url: https://fulcio.sigstore.dev
                    rekor:
                      url: https://rekor.sigstore.dev
```

**Registry vulnerability scanning (AWS ECR)**:

```hcl
resource "aws_ecr_repository" "app" {
  name                 = "myapp"
  image_tag_mutability = "IMMUTABLE"

  image_scanning_configuration {
    scan_on_push = true
  }

  encryption_configuration {
    encryption_type = "KMS"
    kms_key         = aws_kms_key.ecr.arn
  }
}
```

---

## Quick Reference: Severity Summary

| Severity | Container Findings |
|----------|--------------------|
| CRITICAL | Secrets baked into image layers or environment; `privileged: true`; Docker socket mounted; host PID/network namespace without justification |
| HIGH | Container runs as root (no `USER` instruction); no vulnerability scanning in CI; `allowPrivilegeEscalation` not set to false; no resource requests/limits in Kubernetes; no readiness probe; host namespace access; plain Kubernetes secrets in YAML; build args used for secrets |
| MEDIUM | No multi-stage build; unpinned base image; missing `.dockerignore`; no `HEALTHCHECK`; no restart policy in Compose; no resource limits in Compose; `depends_on` without health condition; no `NetworkPolicy`; no `PodDisruptionBudget`; no admission control; mutable registry tags; single replica for production; capabilities not dropped; read-only filesystem not set |
| LOW | `COPY` ordering not optimal for cache; `ADD` used where `COPY` suffices; no explicit networks in Compose; missing Kubernetes labels; base image not pinned to digest; no startup probe; no log rotation; no SBOM generation |
| INFO | Good multi-stage build pattern; effective use of distroless base; well-configured security context; comprehensive probe configuration; proper topology spread constraints |
