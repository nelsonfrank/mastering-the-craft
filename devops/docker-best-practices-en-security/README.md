# **Comprehensive Guide to Container & Deployment Security**  
## **Focus: Production-Grade NestJS Applications**

This document consolidates everything from foundational security principles to complete, ready-to-use code for Dockerfiles, Docker Compose, Traefik reverse proxy, pnpm optimizations, and more.


---

### 1. Introduction: Why Container & Deployment Security Matters

Containers (Docker) and orchestration (Docker Compose / Kubernetes) are the backbone of modern applications. However, they introduce unique risks:
- Supply-chain attacks via vulnerable base images
- Secrets leaking into layers
- Privilege escalation from root containers
- Compromised CI/CD pipelines
- Lateral movement inside Docker networks

**Goal**: Shift-left security + runtime hardening while keeping builds fast and images small.

**Core Pillars Covered**:
- Secure Dockerfiles
- Image scanning
- Secrets handling
- CI/CD pipeline protection
- Secure Docker Compose + Reverse Proxy
- Performance optimizations (pnpm, multi-stage, layer caching)

---

### 2. Secure Dockerfiles

**Key Best Practices**:
- Always use **multi-stage builds**
- Start with minimal trusted base images (Alpine + digest pinning)
- Never run as root → create non-root user
- Copy dependency files first for maximum caching
- Use `.dockerignore` aggressively
- Drop capabilities, read-only filesystem, and healthchecks
- No secrets in Dockerfile or image layers
- Prefer `COPY` over `ADD`, chain `RUN` commands, clean caches

#### Final Recommended Secure NestJS Dockerfile (pnpm version)

```dockerfile
# =============================================
# STAGE 1: Builder
# =============================================
FROM node:22.11-alpine3.21 AS builder

WORKDIR /app

# Enable corepack for pnpm
RUN corepack enable && corepack prepare pnpm@latest --activate

COPY package.json pnpm-lock.yaml ./
RUN pnpm install --frozen-lockfile --ignore-scripts

COPY . .
RUN pnpm run build

# =============================================
# STAGE 2: Runtime (Minimal & Hardened)
# =============================================
FROM node:22.11-alpine3.21 AS runtime

RUN corepack enable && corepack prepare pnpm@latest --activate

# Create non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -u 1001 -S -G nodejs nestjs

WORKDIR /app

# Production dependencies only
COPY package.json pnpm-lock.yaml ./
RUN pnpm install --frozen-lockfile --prod --ignore-scripts

# Copy built app
COPY --from=builder --chown=nestjs:nodejs /app/dist ./dist

# Permissions
RUN chown -R nestjs:nodejs /app && chmod -R 550 /app

USER 1001

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1

LABEL org.opencontainers.image.title="Secure NestJS App" \
      org.opencontainers.image.version="1.0.0" \
      org.opencontainers.image.source="https://github.com/yourorg/your-app"

CMD ["node", "dist/main.js"]
```

**Recommended `.dockerignore`**:
```
.git
.github
.gitignore
README.md
Dockerfile
.dockerignore
node_modules
dist
coverage
test
tests
*.log
.env
.env.*
pnpm-store
```

**Secure Build Command**:
```bash
docker build --pull --no-cache -t my-nestjs-app:1.0.0 .
```

---

### 3. Image Scanning

**Why?** Even secure Dockerfiles inherit vulnerabilities from base images and dependencies.

**Best Tools (2026)**:
- **Trivy** (recommended) – scans OS, dependencies, secrets, misconfigs
- **Grype** – excellent SBOM and alternative DB coverage

**Practice**:
- Run on every CI build: `trivy image --exit-code 1 --severity CRITICAL,HIGH myapp:latest`
- Scan registries continuously
- Generate SBOMs and enforce policies (no critical vulns, signed images)

---

### 4. Secrets Handling

**Golden Rule**: Never bake secrets into images or Dockerfiles.

**Best Approaches**:
- Use runtime injection only
- In Docker Compose → `.env` files (development) or Docker secrets (production)
- In Kubernetes → External Secrets Operator + Vault / AWS Secrets Manager / etc.
- Never use `ENV` for sensitive data; mount as read-only volumes
- Scan with Trivy / gitleaks / trufflehog in CI

---

### 5. CI/CD Pipeline Protection

**High-value target** — protect it like production.

**Key Practices**:
- Ephemeral build agents
- Least-privilege IAM roles + MFA
- Scan Dockerfile (Hadolint), dependencies, and images in every pipeline
- Sign images with cosign
- Require manual approval for production
- Store secrets in dedicated managers (never in repo)

**Example GitHub Actions snippet** (add to your workflow):
```yaml
- name: Build & scan
  run: |
    docker build -t myapp:${{ github.sha }} .
    trivy image --exit-code 1 --severity CRITICAL myapp:${{ github.sha }}
```

---

### 6. Secure Docker Compose Setup

**Full Production-Ready `docker-compose.yml`** (NestJS + Postgres + Traefik)

```yaml
version: "3.9"

services:
  traefik:
    image: traefik:v3.2
    restart: unless-stopped
    ports: ["80:80", "443:443"]
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - traefik_cert:/letsencrypt
    command:
      - --api.dashboard=true
      - --providers.docker=true
      - --providers.docker.exposedbydefault=false
      - --entrypoints.web.address=:80
      - --entrypoints.websecure.address=:443
      - --certificatesresolvers.letsencrypt.acme.email=your-email@example.com
      - --certificatesresolvers.letsencrypt.acme.storage=/letsencrypt/acme.json
      - --certificatesresolvers.letsencrypt.acme.httpchallenge=true
    networks: [proxy-network]

  nestjs-app:
    build: .
    image: my-nestjs-app:${TAG:-latest}
    restart: unless-stopped
    user: "1001:1001"
    read_only: true
    tmpfs: /tmp:rw,noexec,nosuid,size=100m
    cap_drop: [ALL]
    security_opt: [no-new-privileges:true]
    environment:
      - NODE_ENV=production
    env_file: .env.production
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.app.rule=Host(`api.yourdomain.com`)"
      - "traefik.http.routers.app.entrypoints=websecure"
      - "traefik.http.routers.app.tls.certresolver=letsencrypt"
      - "traefik.http.services.app.loadbalancer.server.port=3000"
    networks: [proxy-network]
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:3000/health"]
      interval: 30s
      timeout: 5s
      retries: 3

  postgres:
    image: postgres:16.4-alpine3.21
    restart: unless-stopped
    user: "999:999"
    environment:
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=${POSTGRES_DB}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks: [internal-network]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]

networks:
  proxy-network:
  internal-network:
    internal: true

volumes:
  traefik_cert:
  postgres_data:
```

**`.env.production`** (never commit):
```env
TAG=1.0.0
POSTGRES_USER=appuser
POSTGRES_PASSWORD=SuperStrongPassword123!ChangeMeInProd
POSTGRES_DB=nestjs_db
```

---

### 7. How Reverse Proxy Works in Docker (Traefik)

**Purpose**: Single entry point for HTTPS, TLS termination, routing, rate limiting, and security headers.

**Flow**:
Internet → Traefik (ports 80/443) → Docker labels on containers → NestJS (internal port 3000)

**Benefits**:
- No public ports on NestJS container
- Automatic Let's Encrypt
- Rate limiting, compression, WAF middlewares
- Isolated networks

---

### 8. Why pnpm? (Performance + Security Win)

**Comparison Summary**:

| Feature                  | npm          | pnpm                  |
|--------------------------|--------------|-----------------------|
| Image size               | Larger       | 30-50% smaller        |
| Install speed            | Good         | 2-3× faster           |
| Security (dependency isolation) | Standard | Strict node_modules   |
| Determinism              | Good         | Excellent             |

**Conclusion**: Use pnpm for smaller attack surface, faster CI/CD, and more secure dependency resolution.

---

### 9. Overall Best Practices Checklist (Performant + Secure)

**Dockerfile**:
- Multi-stage + pnpm
- Non-root user + read-only + cap_drop ALL
- Layer caching order
- `.dockerignore`
- Healthcheck + labels

**Docker Compose**:
- Separate networks (proxy vs internal)
- Resource limits
- Traefik in front
- Docker secrets in production
- Healthchecks everywhere

**General**:
- Scan every build (Trivy)
- Sign images
- Rotate secrets
- Rebuild frequently with security patches

---

### 10. Final Recommendations & Next Steps

**Immediate Actions**:
1. Replace your current Dockerfile with the pnpm version above.
2. Use the full `docker-compose.yml` with Traefik.
3. Add Trivy scan to your CI pipeline.
4. Switch to Docker secrets / external secret manager for production.

**Advanced Enhancements** (available on request):
- Kubernetes manifests with securityContext
- GitHub Actions full pipeline
- Redis + rate limiting service
- Multi-arch builds
- SLSA / image signing pipeline

You now have a **complete, production-ready, secure, and performant** NestJS container setup.

Save this document and use it as your team’s reference.  

If you want any section expanded, converted to a PDF template, or a new variation (e.g., with Redis, monorepo, or Kubernetes), just say the word!