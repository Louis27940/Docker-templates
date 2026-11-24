# syntax=docker/dockerfile:1
# ==============================================================================
# Dockerfile - Node.js (Production & Best Practices)
# Inspiré de l'architecture 3DOKR :
# - Multi-stage build (séparation dépendances / runtime)
# - Cache BuildKit persistant (--mount=type=cache,target=/root/.npm)
# - Exécution sous utilisateur non-root (node:node, UID 1000)
# - Ordonnancement optimal des couches (layers) pour réutilisation maximale du cache
# - Healthcheck intégré sans outil externe (node HTTP client natif)
# - Image minimale Alpine 3.23 fixée au patch (node:24.22.3-alpine3.23)
# ==============================================================================

# --- Étape 1 : Dépendances de production ---
FROM node:24.22.3-alpine3.23 AS dependencies

WORKDIR /app

# Définition de l'environnement de production pour npm
ENV NODE_ENV=production

# Copie exclusive des manifestes de paquets pour maximiser le cache Docker
COPY package*.json ./

# Installation propre et déterministe via cache BuildKit persistant
# - --mount=type=cache : stocke le cache hors de l'image (0 octet résiduel dans la couche)
# - --no-audit / --no-fund : réduit la latence réseau et le bruit dans les logs
# - rm -rf /tmp/* : garantit le nettoyage immédiat des fichiers d'extraction temporaires
RUN --mount=type=cache,target=/root/.npm \
    npm ci --omit=dev --no-audit --no-fund && \
    rm -rf /tmp/*

# --- Étape 2 : Image d'exécution finale (Runtime) ---
FROM node:24.22.3-alpine3.23 AS runner

# Variables d'environnement standard
ENV NODE_ENV=production \
    PORT=3000

WORKDIR /app

# 1. Copie des dépendances installées (change rarement, couche mise en cache)
COPY --from=dependencies --chown=node:node /app/node_modules ./node_modules

# 2. Copie du code applicatif et des manifestes (change fréquemment)
COPY --chown=node:node . .

# Bascule vers l'utilisateur non-root natif de Node.js (UID 1000)
USER node

# Port d'écoute du serveur
EXPOSE 3000

# Healthcheck natif sans nécessiter curl ou wget (évite des dépendances supplémentaires)
HEALTHCHECK --interval=30s --timeout=5s --start-period=5s --retries=3 \
    CMD node -e "require('http').get('http://127.0.0.1:' + (process.env.PORT || 3000) + '/health', (res) => process.exit(res.statusCode === 200 ? 0 : 1))"

# Commande de démarrage
CMD ["node", "server.js"]
