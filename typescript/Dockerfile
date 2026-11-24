# ==============================================================================
# Dockerfile - TypeScript (Production & Multi-Stage Optimisé)
# Inspiré de l'architecture 3DOKR :
# - Multi-stage optimisé : prod-deps (parallèle) -> builder -> runner ultra-léger
# - Cache BuildKit haute performance (--mount=type=cache,target=/root/.npm)
# - Séparation nette : aucun compilateur (tsc), devDependencies ou sources .ts dans le runner
# - Droits non-root stricts (node:node, UID 1000)
# - Image Alpine 3.23 fixée au patch (node:24.22.3-alpine3.23)
# - Healthcheck natif
# ==============================================================================

# --- Étape 1 : Dépendances de production uniquement (isolées et parallélisables) ---
FROM node:24.22.3-alpine3.23 AS prod-deps

WORKDIR /app

# Copie exclusive des manifestes pour maximiser la réutilisation du cache de couche
COPY package*.json ./

# Installation déterministe des dépendances de production avec cache BuildKit persistant
# Évite de télécharger les devDependencies et supprime le besoin de "npm prune"
RUN --mount=type=cache,target=/root/.npm \
    npm ci --omit=dev

# --- Étape 2 : Compilation TypeScript vers JavaScript (dist/) ---
FROM node:24.22.3-alpine3.23 AS builder

WORKDIR /app

# Copie des manifestes de paquets
COPY package*.json ./

# Installation complète (avec devDependencies nécessaires au compilateur tsc)
# Le cache BuildKit /root/.npm est réutilisé et partagé avec la première étape
RUN --mount=type=cache,target=/root/.npm \
    npm ci

# Copie de la configuration TypeScript et des fichiers sources
COPY tsconfig*.json ./
COPY src/ ./src/

# Compilation du code source TypeScript vers JavaScript (dans /app/dist)
RUN npm run build

# --- Étape 3 : Image d'exécution minimale pour la production ---
FROM node:24.22.3-alpine3.23 AS runner

# Variables d'environnement standard
ENV NODE_ENV=production \
    PORT=3000

WORKDIR /app

# Copie exclusive des modules de production depuis prod-deps (couche stable en cache)
COPY --from=prod-deps --chown=node:node /app/node_modules ./node_modules

# Copie des fichiers compilés JavaScript depuis le builder
COPY --from=builder --chown=node:node /app/dist ./dist

# Copie du package.json pour les métadonnées runtime (type module, nom, version)
COPY --from=prod-deps --chown=node:node /app/package.json ./package.json

# Sécurité : passage sous l'utilisateur non privilégié node (UID 1000)
USER node

# Port d'écoute du serveur
EXPOSE 3000

# Healthcheck HTTP natif sans binaire curl/wget
HEALTHCHECK --interval=30s --timeout=5s --start-period=5s --retries=3 \
    CMD node -e "require('http').get('http://127.0.0.1:' + (process.env.PORT || 3000) + '/health', (res) => process.exit(res.statusCode === 200 ? 0 : 1))"

# Démarrage de l'application compilée
CMD ["node", "dist/index.js"]
