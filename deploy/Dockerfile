# syntax=docker/dockerfile:1
#
# Hosting image (fork-only; the root Dockerfile is upstream's local-dev image).
#
# The root image runs `vite` in dev mode against a bind-mounted checkout. This
# one bakes the production bundle and runs `vite preview`, which serves that
# bundle AND the same /api provider middleware, without HMR, the source tree or
# the Provider Settings key-entry endpoints (those are dev-only by design).
#
# Build from the repository root:
#   docker build -f deploy/Dockerfile -t gods-eye-view .
#
# Same Node as the root Dockerfile and CI, pinned by digest.
FROM node:24.22.2-slim@sha256:c86ecead9e55a5210ac1727de38a4ca840bf03dcaf34dc9749addbf383b1920c

# Puppeteer is only used by the QA scripts; skip its ~150MB Chromium download.
ENV PUPPETEER_SKIP_DOWNLOAD=1

WORKDIR /app
RUN chown node:node /app
USER node

COPY --chown=node:node package.json package-lock.json ./
# The download cache is ~115MB the running app never reads.
RUN npm ci && npm cache clean --force

COPY --chown=node:node . .

# The three keys below are the ones upstream injects into the browser bundle on
# purpose (build/vite.js `define`; see SECURITY.md). They are compiled into
# dist/ here, so anyone who can load the page or pull this image can read them:
# restrict each one at its provider. Passing them as BuildKit secrets only keeps
# them out of `docker history`. All are optional; with none the app starts on
# keyless Esri imagery. Server-side keys are NEVER build inputs; they are
# runtime environment only (see deploy/gev.env.example).
RUN --mount=type=secret,id=GOOGLE_MAPS_API_KEY,env=GOOGLE_MAPS_API_KEY,uid=1000 \
    --mount=type=secret,id=CESIUM_ION_TOKEN,env=CESIUM_ION_TOKEN,uid=1000 \
    --mount=type=secret,id=MAPILLARY_CLIENT_TOKEN,env=MAPILLARY_CLIENT_TOKEN,uid=1000 \
    npm run build \
    && mkdir -p .gev-cache .gev-logs

# Runtime state (downloaded map/satellite data, logs). Mount volumes here to
# keep it across container replacements.
VOLUME ["/app/.gev-cache", "/app/.gev-logs"]

# Bind every interface inside the container. This does not widen the Host
# allowlist: IP addresses and localhost are accepted, any other name only via
# GEV_ALLOWED_HOSTS (build/allowedHosts.js).
ENV HOST=0.0.0.0
ENV PORT=4173
ENV NODE_ENV=production
EXPOSE 4173

HEALTHCHECK --interval=30s --timeout=10s --start-period=20s --retries=3 \
  CMD ["node", "-e", "fetch('http://127.0.0.1:'+(Number.parseInt(process.env.PORT,10)||4173)+'/').then((r)=>process.exit(r.ok?0:1),()=>process.exit(1))"]

# Exec form via sh so PORT is honoured and node is PID 1's direct child that
# receives SIGTERM (exec replaces the shell).
CMD ["sh", "-c", "exec node node_modules/vite/bin/vite.js preview --host \"$HOST\" --port \"$PORT\" --strictPort"]
