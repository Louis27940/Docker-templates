# syntax=docker/dockerfile:1
# ==============================================================================
# Dockerfile - Rust (Production, Ultra-Fast & High-Performance)
# Inspiré de l'architecture 3DOKR :
# - Multi-stage build optimisé avec caches BuildKit (registry, git, target)
# - Compilation des crates externes découplée du code source applicatif
# - Binaire compilé en mode Release, strippé de ses symboles de débogage
# - Builder Rust 1.85.0 fixé au patch sur base Alpine 3.24
# - Image finale minimale Alpine 3.24 fixée à la mineure (~15 Mo)
# - Exécution sous utilisateur non-root dédié (appuser, UID/GID 10001)
# - Healthcheck natif (wget Alpine/Busybox)
# ==============================================================================

# --- Étape 1 : Builder (Compilation optimisée avec caches BuildKit) ---
FROM rust:1.85.0-alpine3.24 AS builder

# Outils système nécessaires à la compilation (musl, build-base avec strip)
RUN --mount=type=cache,target=/var/cache/apk \
    apk add musl-dev build-base

WORKDIR /app

# Création anticipée du compte utilisateur non-root dédié pour l'image finale
RUN addgroup -S -g 10001 appgroup && \
    adduser -S -u 10001 -G appgroup -s /sbin/nologin -D appuser

# ------------------------------------------------------------------------------
# 1. Mise en cache des dépendances externes Cargo
# ------------------------------------------------------------------------------
# Copie exclusive des manifestes de dépendances et éventuel script de build
COPY Cargo.toml Cargo.lock* build.rs* ./

# Pré-téléchargement et pré-compilation des dépendances via un squelette factice (dummy)
# - Cache registry : évite de re-télécharger les crates depuis crates.io
# - Cache git      : évite de re-cloner les dépendances Git
# - Cache target   : conserve les crates pré-compilées hors des couches de l'image
RUN --mount=type=cache,target=/usr/local/cargo/registry \
    --mount=type=cache,target=/usr/local/cargo/git \
    --mount=type=cache,target=/app/target \
    mkdir -p src && \
    echo "fn main() { println!(\"dummy\"); }" > src/main.rs && \
    cargo build --release && \
    rm -rf src

# ------------------------------------------------------------------------------
# 2. Compilation de l'application réelle
# ------------------------------------------------------------------------------
# Copie des véritables sources applicatives
COPY src/ ./src/

# Nom du binaire configurable en argument de build (détecté automatiquement par défaut)
ARG BINARY_NAME

# Compilation du code applicatif en mode release, extraction et strip du binaire :
# - touch src/main.rs garantit l'invalidation du binaire dummy dans le cache target
# - L'extraction dynamique gère le nom du package sans dépendance externe
# - Le strip élimine symboles de debug et tables de relocation pour alléger l'artefact
# - L'artefact final est extrait du cache mount vers /app/bin/app dans le layer Docker
RUN --mount=type=cache,target=/usr/local/cargo/registry \
    --mount=type=cache,target=/usr/local/cargo/git \
    --mount=type=cache,target=/app/target \
    touch src/main.rs && \
    cargo build --release && \
    BIN_NAME="${BINARY_NAME:-$(cargo metadata --no-deps --format-version 1 | sed -n 's/.*"packages":\[{"name":"\([^"]*\)".*/\1/p')}" && \
    if [ ! -f "target/release/${BIN_NAME}" ]; then \
        BIN_NAME=$(find target/release -maxdepth 1 -type f -perm /111 ! -name "*.so" ! -name "*.dylib" -exec basename {} \; | head -n 1); \
    fi && \
    strip --strip-all "target/release/${BIN_NAME}" && \
    mkdir -p /app/bin && \
    cp "target/release/${BIN_NAME}" /app/bin/app && \
    chmod 0755 /app/bin/app

# --- Étape 2 : Image d'exécution minimale (Runtime) ---
FROM alpine:3.24 AS runner

# Installation minimale des certificats TLS et tzdata, initialisation de /app
RUN apk --no-cache add ca-certificates tzdata && \
    mkdir -p /app && \
    chown 10001:10001 /app

# Import sécurisé des utilisateurs non-root créés dans le builder (UID/GID 10001)
COPY --from=builder /etc/passwd /etc/group /etc/

WORKDIR /app

# Copie exclusive du binaire compilé et strippé depuis le builder avec droits stricts
COPY --from=builder --chown=10001:10001 --chmod=0755 /app/bin/app /usr/local/bin/app

# Passage sous l'utilisateur non privilégié
USER 10001:10001

# Port d'écoute par défaut
EXPOSE 8080

# Healthcheck HTTP via wget (inclus nativement dans Alpine/Busybox)
HEALTHCHECK --interval=30s --timeout=5s --start-period=5s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://127.0.0.1:8080/health || exit 1

# Démarrage du binaire compilé
CMD ["/usr/local/bin/app"]
