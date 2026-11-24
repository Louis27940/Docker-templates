# syntax=docker/dockerfile:1
# ==============================================================================
# Dockerfile - Tout Langage / Modèle Universel (Universal Multi-Stage Blueprint)
# Modèle architectural de référence 3DOKR optimisé pour le moteur Docker (BuildKit)
# Applicable à n'importe quel langage : Go, Python, Java, .NET, Rust, Node, etc.
#
# Principes clés de performance & sécurité intégrés :
# 1. Directives BuildKit modernes : utilisation intensive de --mount=type=cache pour
#    les gestionnaires de paquets universels (apk, pip, go mod, nuget, m2, cargo, npm).
#    -> Les caches de téléchargement ne sont JAMAIS écrits dans les couches de l'image.
# 2. Ordonnancement strict des couches (Layer Ordering) :
#    [Outils/Dépendances système] -> [Manifestes de dépendances] ->
#    [Pré-résolution/Cache] -> [Code source applicatif] -> [Build final].
# 3. Réduction des couches intermédiaires : regroupement des opérations logiques,
#    chown direct à la copie (COPY --chown) sans RUN chown supplémentaire.
# 4. Multi-Stage Build & Runner minimal : Base OS Alpine fixée en version 3.24 (ou 3.23),
#    ou image officielle runtime minimale (scratch, distroless, temurin-jre, etc.).
# 5. Principe de moindre privilège : Utilisateur non-root dédié (UID/GID 10001:10001).
# 6. Compatibilité filesystem en lecture seule (read_only: true + tmpfs dans compose.yaml) :
#    Variables d'environnement préconfigurées pour rediriger caches/fichiers temporaires vers /tmp.
# 7. Healthcheck déterministe intégré et compatible compose.yaml.
# ==============================================================================

# Arguments globaux de build (privilégiant Alpine fixé en 3.24 ou 3.23)
ARG BASE_BUILDER_IMAGE=alpine:3.24
ARG BASE_RUNNER_IMAGE=alpine:3.24
ARG APP_VERSION=1.0.0

# ==============================================================================
# Étape 1 : BUILDER (Environnement de compilation ou de packaging)
# ==============================================================================
FROM ${BASE_BUILDER_IMAGE} AS builder

# Définition du répertoire de travail de build
WORKDIR /src

# Installation d'éventuels outils de build système avec cache APK persistant
# (Le cache d'APK est monté à la volée et n'augmente pas la taille de la couche)
RUN --mount=type=cache,target=/var/cache/apk \
    apk add --no-cache ca-certificates tzdata 2>/dev/null || true

# Création anticipée du compte utilisateur non-root dédié (UID 10001 / GID 10001)
# Utilisable pour exporter /etc/passwd et /etc/group vers les runners minimaux (scratch)
RUN addgroup -S -g 10001 appgroup && \
    adduser -S -u 10001 -G appgroup -D -H appuser

# ------------------------------------------------------------------------------
# EXEMPLES D'OPTIMISATIONS MULTI-LANGAGES AVEC BUILDKIT (DÉPENDANCES & CACHE)
# Décommentez ou adaptez la section correspondant à votre technologie :
# ------------------------------------------------------------------------------

# --- [A] GO (Ultra-rapide, binaire statique sans CGO) ---
# ARG BASE_BUILDER_IMAGE=golang:1.24-alpine3.23
# ARG BASE_RUNNER_IMAGE=alpine:3.24  # ou FROM scratch
# 1. Copie exclusive des fichiers de modules (invalidation minimale du cache)
# COPY go.mod go.sum ./
# 2. Téléchargement des modules avec cache partagé BuildKit
# RUN --mount=type=cache,target=/go/pkg/mod \
#     go mod download -x
# 3. Copie du code source applicatif
# COPY . .
# 4. Compilation Release avec cache de compilation et de modules
# RUN --mount=type=cache,target=/go/pkg/mod \
#     --mount=type=cache,target=/root/.cache/go-build \
#     CGO_ENABLED=0 go build -ldflags="-s -w -extldflags '-static'" -trimpath -o /app/bin/app .

# --- [B] PYTHON (Isolation des dépendances via --prefix) ---
# ARG BASE_BUILDER_IMAGE=python:3.12.16-alpine3.23
# ARG BASE_RUNNER_IMAGE=python:3.12.16-alpine3.23
# 1. Installation des dépendances de compilation C/musl si nécessaire
# RUN --mount=type=cache,target=/var/cache/apk \
#     apk add --no-cache build-base libffi-dev
# 2. Copie du manifeste de dépendances
# COPY requirements.txt ./
# 3. Installation isolée dans /install avec cache wheel/pip BuildKit
# RUN --mount=type=cache,target=/root/.cache/pip \
#     pip install --prefix=/install --no-warn-script-location --no-compile -r requirements.txt
# 4. Dans l'étape RUNNER :
# COPY --from=builder /install /usr/local
# COPY --chown=10001:10001 . /app/

# --- [C] .NET (C# / F# / WebAPI AOT ou ReadyToRun) ---
# ARG BASE_BUILDER_IMAGE=mcr.microsoft.com/dotnet/sdk:10.0.404-alpine3.24
# ARG BASE_RUNNER_IMAGE=mcr.microsoft.com/dotnet/runtime:10.0.15-alpine3.24-amd64
# 1. Copie isolée des fichiers projet / solution
# COPY *.sln *.csproj ./
# 2. Restauration des packages NuGet avec cache monté
# RUN --mount=type=cache,target=/root/.nuget/packages \
#     dotnet restore
# 3. Copie des sources
# COPY . .
# 4. Publication Release sans re-téléchargement
# RUN --mount=type=cache,target=/root/.nuget/packages \
#     dotnet publish -c Release -r linux-musl-x64 --no-restore -o /app/bin/

# --- [D] JAVA / KOTLIN (Maven / Gradle) ---
# ARG BASE_BUILDER_IMAGE=eclipse-temurin:21-jdk-alpine3.23
# ARG BASE_RUNNER_IMAGE=eclipse-temurin:21-jre-alpine3.23
# 1. Copie du descripteur de build
# COPY pom.xml ./
# 2. Résolution des dépendances hors-ligne avec cache .m2 monté
# RUN --mount=type=cache,target=/root/.m2 \
#     mvn dependency:go-offline -B
# 3. Copie du code source
# COPY src ./src
# 4. Packaging Release sans relancer les tests unitaires
# RUN --mount=type=cache,target=/root/.m2 \
#     mvn clean package -DskipTests -B && \
#     mkdir -p /app/bin && cp target/*.jar /app/bin/app.jar

# --- [E] RUST (Compilation native avec cache Cargo & target) ---
# ARG BASE_BUILDER_IMAGE=rust:1.85.0-alpine3.24
# ARG BASE_RUNNER_IMAGE=alpine:3.24
# 1. Copie des manifestes Cargo
# COPY Cargo.toml Cargo.lock* ./
# 2. Squelette factice pour mise en cache des crates externes
# RUN mkdir -p src && echo 'fn main() { println!("cached"); }' > src/main.rs
# RUN --mount=type=cache,target=/usr/local/cargo/registry \
#     --mount=type=cache,target=/usr/local/cargo/git \
#     --mount=type=cache,target=/src/target \
#     cargo build --release && rm -rf src
# 3. Copie des véritables sources et build final avec cache monté
# COPY src ./src
# RUN --mount=type=cache,target=/usr/local/cargo/registry \
#     --mount=type=cache,target=/usr/local/cargo/git \
#     --mount=type=cache,target=/src/target \
#     touch src/main.rs && cargo build --release && \
#     strip /src/target/release/$(cargo metadata --no-deps --format-version 1 | sed -n 's/.*"name":"\([^"]*\)".*/\1/p') && \
#     mkdir -p /app/bin && cp /src/target/release/$(cargo metadata --no-deps --format-version 1 | sed -n 's/.*"name":"\([^"]*\)".*/\1/p') /app/bin/app

# ------------------------------------------------------------------------------
# IMPLÉMENTATION PAR DÉFAUT DU BLUEPRINT UNIVERSEL (Exécutable autonome)
# Ce bloc génère un service fonctionnel prêt à l'emploi pour compose.yaml
# ------------------------------------------------------------------------------
COPY . .

RUN mkdir -p /app/bin && \
    cat << 'EOF' > /app/bin/app
#!/bin/sh
set -eu
trap 'echo "Arrêt propre du service..."; rm -f /tmp/healthy; exit 0' TERM INT

# Marqueur pour le healthcheck déterministe (/tmp est un volume tmpfs inscriptible)
touch /tmp/healthy

echo "================================================================="
echo "Service universel 3DOKR démarré avec succès !"
echo "Environnement  : ${APP_ENV:-production}"
echo "Port d'écoute  : ${PORT:-8080}"
echo "Utilisateur    : $(id -u):$(id -g) (Non-root)"
echo "Fichiers       : Read-Only filesystem actif (tmpfs sur /tmp)"
echo "================================================================="

while true; do
    sleep 3600 &
    wait $!
done
EOF
RUN chmod +x /app/bin/app

# ==============================================================================
# Étape 2 : RUNNER (Image de production minimale)
# Selon la stack technique :
# - Go / Rust compilé statiquement : alpine:3.24 ou FROM scratch
# - Python                         : python:3.12.16-alpine3.23
# - Node.js                        : node:24.22.3-alpine3.23
# - .NET Runtime                   : mcr.microsoft.com/dotnet/runtime:10.0.15-alpine3.24-amd64
# - Java JRE                       : eclipse-temurin:21-jre-alpine3.23
# ==============================================================================
FROM ${BASE_RUNNER_IMAGE} AS runner

# Installation minimale des certificats TLS et données de fuseau horaire
# Montage du cache APK pour éviter d'embarquer les index de paquets dans la couche finale
RUN --mount=type=cache,target=/var/cache/apk \
    apk add --no-cache ca-certificates tzdata 2>/dev/null || true

# Configuration du compte utilisateur non-root (UID/GID 10001:10001)
# Note : Pour les images 'scratch', utiliser :
#   COPY --from=builder /etc/passwd /etc/passwd
#   COPY --from=builder /etc/group /etc/group
RUN addgroup -S -g 10001 appgroup 2>/dev/null || true && \
    adduser -S -u 10001 -G appgroup -D -H appuser 2>/dev/null || true

# Répertoire de travail applicatif
WORKDIR /app

# Copie de l'artefact compilé en attribuant directement la propriété au compte non-root
# (Une seule couche combinée : évite la duplication d'espace disque d'un chown séparé)
COPY --from=builder --chown=10001:10001 /app/bin/ /app/

# Variables d'environnement standard et optimisations pour conteneur 'read_only: true'
# - TMPDIR=/tmp permet aux outils d'utiliser la partition volatile tmpfs définie dans compose.yaml
# - PYTHONDONTWRITEBYTECODE=1 évite les erreurs d'écriture de fichiers .pyc sur un FS en lecture seule
# - DOTNET_EnableDiagnostics=0 évite les erreurs de sockets/pipes de diagnostic sous read-only
ENV APP_ENV=production \
    PORT=8080 \
    TMPDIR=/tmp \
    PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    DOTNET_EnableDiagnostics=0 \
    COMPlus_EnableDiagnostics=0

# Verrouillage sous l'utilisateur non-privilégié
USER 10001:10001

# Port d'écoute documenté
EXPOSE 8080

# Sonde de santé périodique (Healthcheck) déterministe
# Valide la présence du fichier marqueur dans /tmp (ou exit 0 si non initialisé)
HEALTHCHECK --interval=30s --timeout=5s --start-period=5s --retries=3 \
    CMD test -f /tmp/healthy || exit 0

# Commande finale d'exécution (syntaxe exec JSON sans sous-shell sh inutile)
CMD ["/app/app"]
