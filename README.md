# Production Docker Templates (3DOKR Architecture)

This directory provides a collection of ready-to-use Docker templates designed to quickly bootstrap containerized applications while applying **all security, performance, and resilience standards established in the 3DOKR project**.

All base images are **consistently built on Alpine Linux (3.23 or 3.24)** and are **strictly pinned down to the patch version** (or minor release if the patch is unavailable).

---

## Table of Contents

1. [Directory Structure](#1-directory-structure)
2. [Best Practices Matrix](#2-best-practices-matrix)
3. [Templates and Image Versions](#3-templates-and-image-versions)
   - [Node.js (`./templates/node`)](#nodejs-template)
   - [TypeScript (`./templates/typescript`)](#typescript-template)
   - [Rust (`./templates/rust`)](#rust-template)
   - [Universal / Polyglot (`./templates/tout-langage`)](#universal--polyglot-template)
4. [Quick Start Guide](#4-quick-start-guide)
5. [Security & Production Checklist](#5-security--production-checklist)

---

## 1. Directory Structure

Each subdirectory is a standalone project template containing its `Dockerfile`, `.dockerignore`, `compose.yaml`, environment variables example (`.env.example`), and sample secrets (`secrets/`):

```text
templates/
├── README.md                          # General documentation and usage guide
│
├── node/                              # Node.js template (JavaScript / Express / Fastify / etc.)
│   ├── Dockerfile                     # Multi-stage, node:24.22.3-alpine3.23, non-root (node:1000)
│   ├── .dockerignore                  # Filters node_modules, tests, logs, secrets, VCS
│   ├── compose.yaml                   # Dual-tier networks, postgres:17.12-alpine3.23, resource quotas
│   ├── .env.example                   # Non-sensitive configuration variables
│   └── secrets/
│       └── db_password.example.txt    # Docker Compose secret placeholder
│
├── typescript/                        # TypeScript template (transpilation & devDependencies pruning)
│   ├── Dockerfile                     # Parallel multi-stage (deps -> build -> lean runner)
│   ├── .dockerignore                  # Filters dist, tsbuildinfo, node_modules, tests
│   ├── compose.yaml                   # postgres:17.12-alpine3.23, network isolation, native secrets
│   ├── .env.example
│   └── secrets/
│       └── db_password.example.txt
│
├── rust/                              # Rust template (high performance, ultra-compact)
│   ├── Dockerfile                     # Builder rust:1.85.0-alpine3.24 + Runner alpine:3.24
│   ├── .dockerignore                  # Filters target/, local cargo files (preserves Cargo.lock)
│   ├── compose.yaml                   # postgres:17.12-alpine3.23, no-new-privileges, cap_drop: ALL
│   ├── .env.example
│   └── secrets/
│       └── db_password.example.txt
│
└── tout-langage/                      # Universal polyglot template (architectural blueprint)
    ├── Dockerfile                     # Universal multi-stage (Alpine 3.24/3.23, Go, Python, Java, .NET)
    ├── .dockerignore                  # Comprehensive multi-language exclusion rules
    ├── compose.yaml                   # Reference stack (read_only, tmpfs, secrets, 2-tier network)
    ├── .env.example
    └── secrets/
        ├── db_password.example.txt
        └── app_secret.example.txt
```

---

## 2. Best Practices Matrix

| Best Practice | 3DOKR Baseline | Description & Implementation |
| :--- | :---: | :--- |
| **Alpine Images & Pinned Tags** | `redis`, `db`, `worker` | All images prioritize Alpine Linux (3.23 or 3.24) and are locked to exact patch versions (e.g., `postgres:17.12-alpine3.23`, `node:24.22.3-alpine3.23`, `rust:1.85.0-alpine3.24`). |
| **Multi-Stage Builds** | `worker`, `result` | Clear separation between build environments (SDKs, compilers, package caches) and minimal runtime images to reduce attack surface and memory footprints. |
| **Non-Root Execution** | `vote` (10001), `result` (node), `worker` (app) | No container runs as `root`. Strict ownership mapping using `COPY --chown=user:group` and late `USER` declarations. |
| **Native Secret Management** | `secrets/` & `/run/secrets/` | Sensitive credentials are **never** passed as environment variables (`ENV`). Docker Compose Secrets are mounted into memory under `/run/secrets/`. |
| **Dual-Tier Network Zoning** | `front-tier` / `back-tier` | Architecture split into two distinct networks: an ingress bridge (`front-tier`) and a strictly internal backend (`back-tier` with `internal: true`) that prohibits outbound Internet traffic from databases. |
| **Restricted Port Binding** | `127.0.0.1:PORT:PORT` | Published ports bind exclusively to the host loopback interface, preventing unintended public exposure without an ingress proxy. |
| **Deterministic Healthchecks** | `healthcheck` & `service_healthy` | Periodic lightweight native probes coupled with `depends_on: { condition: service_healthy }` to eliminate cold-start crash loops. |
| **Linux Capability Drops** | `cap_drop: [ALL]`, `no-new-privileges:true` | Complete removal of Linux kernel capabilities (`cap_drop: ALL`) and strict enforcement against privilege escalation (`security_opt: [no-new-privileges:true]`). |
| **Read-Only Root Filesystem** | `read_only: true` + `tmpfs` | Root filesystem mounted in read-only mode with dedicated temporary in-memory mounts (`tmpfs: /tmp:rw,noexec,nosuid`). |
| **Resource Quotas** | `deploy.resources.limits` | Explicit CPU and RAM quotas to protect host stability and prevent noisy-neighbor / denial-of-service issues. |
| **Strict Environment Validation** | `${VAR:?Required error}` | Compose parameter substitution syntax triggering immediate termination if a mandatory variable is missing in `.env`. |

---

## 3. Templates and Image Versions

### Node.js Template

- **Location**: `templates/node/`
- **Images Used**:
  - Application: `node:24.22.3-alpine3.23` (pinned to patch 24.22.3, Alpine 3.23)
  - Database: `postgres:17.12-alpine3.23` (pinned to patch 17.12, Alpine 3.23)
- **Technical Highlights**:
  - **Stage 1 (dependencies)**: Installs production dependencies via `npm ci --omit=dev --no-audit --no-fund` leveraging BuildKit cache mounts (`--mount=type=cache,target=/root/.npm`) and atomic cleanup of `/tmp/*`.
  - **Stage 2 (runner)**: Minimal runtime Alpine image without compilers or residual npm cache files.
  - **Optimal Layer Ordering**: Source code copies are decoupled from dependency installation, ensuring application code edits do not invalidate the cached `node_modules` layer.
  - **Non-Root User**: Runs as native unprivileged system user `node:node` (UID/GID `1000:1000`).
  - **Native Healthcheck**: HTTP request executed via Node.js native `http` module without external utilities (`curl` or `wget`).

### TypeScript Template

- **Location**: `templates/typescript/`
- **Images Used**:
  - Application: `node:24.22.3-alpine3.23` (`prod-deps`, `builder`, and `runner`)
  - Database: `postgres:17.12-alpine3.23`
- **Technical Highlights**:
  - **Stage 1 (prod-deps)**: Isolated production dependency installation (`npm ci --omit=dev`) with BuildKit cache mount (`--mount=type=cache,target=/root/.npm`). Runs in parallel with the build stage.
  - **Stage 2 (builder)**: Full dependency installation (`npm ci`) and compilation of `src/` into `dist/`. No slow `npm prune` operation is required because production dependencies are isolated in Stage 1.
  - **Stage 3 (runner)**: Copies only compiled JavaScript (`dist/`) and production `node_modules`. Neither `tsc`, TypeScript source files, nor `devDependencies` exist in the final production image. Source code modifications achieve instant rebuilds since the runtime dependency layer remains untouched.

### Rust Template

- **Location**: `templates/rust/`
- **Images Used**:
  - Builder: `rust:1.85.0-alpine3.24` (pinned to patch 1.85.0, Alpine 3.24)
  - Runner: `alpine:3.24` (pinned to minor release Alpine 3.24)
  - Database: `postgres:17.12-alpine3.23`
- **Technical Highlights**:
  - **Advanced BuildKit Caching**: Persistent cache mounts (`/usr/local/cargo/registry`, `/usr/local/cargo/git`, and `/app/target`) combined with a dummy `main.rs` compilation step to pre-fetch and pre-compile all crates without bloating Docker layers.
  - **Binary Stripping**: Strips debug symbols and relocation tables (`strip --strip-all`), reducing final binary size by 60% to 80%.
  - **Clean Artifact Extraction**: Dynamically resolves and extracts the compiled binary to `/app/bin/app`, avoiding unintended file copying into the runner.
  - **Ultra-Compact Runner**: Final image weighs approximately 15 MB on Alpine 3.24 with dedicated `appuser (10001:10001)` and consolidated layers.
  - **Native Healthcheck**: Implemented via Busybox `wget` built into Alpine.

### Universal / Polyglot Template

- **Location**: `templates/tout-langage/`
- **Default Images**:
  - Builder: `alpine:3.24` (or `alpine:3.23`)
  - Runner: `alpine:3.24` (or `alpine:3.23`)
  - Database: `postgres:17.12-alpine3.23`
- **Language Adaptations (with pinned patch tags on Alpine 3.23 / 3.24)**:
  - **Go**: `golang:1.24-alpine3.23` (builder) -> `alpine:3.24` or `FROM scratch` (runner).
  - **Python**: `python:3.12.16-alpine3.23` (builder & runner).
  - **.NET**: `mcr.microsoft.com/dotnet/sdk:10.0.404-alpine3.24` (builder) -> `mcr.microsoft.com/dotnet/runtime:10.0.15-alpine3.24-amd64` (runner).
  - **Java**: `eclipse-temurin:21-jdk-alpine3.23` (builder) -> `eclipse-temurin:21-jre-alpine3.23` (runner).
  - **Rust**: `rust:1.85.0-alpine3.24` (builder) -> `alpine:3.24` (runner).
- **Technical Highlights**:
  - Universal BuildKit cache mount syntax (`--mount=type=cache`) for common package managers (`apk`, `pip`, `go mod`, `nuget`, `m2`, `cargo`).
  - Strict layer sequencing: package manifests are copied and resolved before application source code.
  - Built-in support for read-only containers (`read_only: true` with `tmpfs: /tmp:rw,noexec,nosuid`).
  - Runtime environment variables tuned for read-only execution: `TMPDIR=/tmp`, `PYTHONDONTWRITEBYTECODE=1`, `DOTNET_EnableDiagnostics=0`.
  - Comprehensive `.dockerignore` filtering VCS, CI/CD configs, test caches, local build outputs, and sensitive files.

---

## 4. Quick Start Guide

To initialize a new service using one of the templates:

### 1. Copy the Chosen Template
```bash
# Example for a TypeScript service
cp -r templates/typescript ./my-service
cd ./my-service
```

### 2. Configure Environment Variables and Secrets
```bash
# 1. Create the environment file
cp .env.example .env

# 2. Create the secret file for the database password
mkdir -p secrets
cp secrets/db_password.example.txt secrets/db_password.txt
```

### 3. Start the Stack with Docker Compose
```bash
docker compose up -d --build
```

### 4. Verify Health Status
```bash
docker compose ps
```
All containers should transition to `Up (healthy)`.

---

## 5. Security & Production Checklist

Validate this checklist before promoting any containerized service to production:

- [x] **Pinned Alpine Images**: Verify that all images target Alpine 3.23/3.24 with explicit patch tags where available.
- [x] **Non-Root Execution**: Inspect container processes (`docker exec <id> id`) to confirm the UID is not `0`.
- [x] **Protected Secrets**: Ensure no credentials exist in plaintext inside `compose.yaml`, `.env`, or `docker inspect`.
- [x] **Restricted Ports**: Bind ports exclusively to `127.0.0.1:PORT:PORT` or manage access through an ingress controller.
- [x] **Isolated Database Network**: Ensure data tiers reside in an isolated network with `internal: true`.
- [x] **Restricted Capabilities**: Confirm `cap_drop: [ALL]` and `security_opt: [no-new-privileges:true]` are configured.
- [x] **Configured Healthchecks**: Use deterministic healthchecks and `condition: service_healthy` on all `depends_on`.
- [x] **Resource Limits**: Define CPU and memory ceilings under `deploy.resources.limits`.
