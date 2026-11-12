# Hosting this fork

Fork-only files. Nothing here changes upstream code, so syncing the fork stays
conflict-free: everything lives in `deploy/` plus one workflow,
`.github/workflows/deploy.yml`.

## What gets deployed, and why not a static host

God's Eye View is not a browser-only app. The page calls same-origin `/api/*`
routes (flights, ships, satellites, cameras, weather, news, voice) that a Node
process answers; that process also holds every private key. A static host
(GitHub Pages, Cloudflare Pages, Netlify, Vercel static) serves the globe and
nothing else.

So the deployable is a container: `deploy/Dockerfile` builds the production
bundle and runs `vite preview`, which serves the bundle and the same `/api`
middleware. Compared with upstream's dev image it has no HMR, no source bind
mount and no in-app key-entry endpoints (upstream leaves those out of preview).

On every push to `main` the workflow builds that image, smoke-tests the running
container and publishes it to `ghcr.io/<owner>/gods-eye-view` (`:latest` and
`:sha-<commit>`). Pull requests get the build and the smoke test only.

## Choose how you reach it

Upstream treats the server as a key broker and refuses, with `403 Proxied
requests are not accepted`, any key-spending request that arrives carrying
reverse-proxy headers (`X-Forwarded-For`, `Via`, `CF-Ray`, ...). That is
deliberate (`SECURITY.md`, `src/localRequestGate.mjs`) and it decides what each
access method can do:

|                                                             | SSH port-forward | Private address (tailnet/LAN IP) | Tunnel or reverse proxy with login |
| ----------------------------------------------------------- | :--------------: | :------------------------------: | :--------------------------------: |
| Globe, flights, satellites, cameras, weather, launches      |       yes        |               yes                |                yes                 |
| Ships (AISStream), fires (FIRMS), OpenSky credentials       |       yes        |               yes                |                yes                 |
| Google 3D Tiles / Cesium ion (browser keys)                 |       yes        |               yes                |                yes                 |
| Typed chat, HUD summary (OpenAI)                            |       yes        |               yes                |           **no** (403)             |
| Voice (needs the microphone, so HTTPS or `localhost`)       |       yes        |                no                |           **no** (403)             |
| Google place search, TomTom traffic, News, Mapillary, mapped installations | yes |          yes                |           **no** (403)             |

**Recommended: keep it private and use an SSH port-forward.** Every feature
works, nothing is exposed, and the login is your SSH key:

```bash
ssh -N -L 4173:127.0.0.1:4173 you@your-host
```

Then open `http://localhost:4173`.

**If you want a URL you can open from anywhere,** use the `tunnel` profile
(Cloudflare Tunnel with a Cloudflare Access login in front). It needs no open
inbound port, but the rows marked **no** stay off. Getting them back means
changing upstream's proxy gate in this fork, which is a security trade-off to
make on purpose, not a config switch.

Never publish the port on a public interface without a login in front: anyone
who can reach it can spend the keys in `gev.env`.

## One-time setup

### GitHub

1. Actions tab: enable workflows (forks start with them disabled).
2. Optional, for photorealistic 3D and Street Level: add repository secrets
   `GOOGLE_MAPS_API_KEY`, `CESIUM_ION_TOKEN`, `MAPILLARY_CLIENT_TOKEN`
   (Settings, Secrets and variables, Actions). These three are compiled into
   the browser bundle by upstream's design, so every visitor can read them.
   Restrict them at the provider before adding them:
   - Google: HTTP-referrer restriction to your hostname, API restriction to
     Map Tiles API + Maps JavaScript API, a daily quota and a budget alert.
   - Cesium ion: a token with only `assets:read` and an allowed-URL list.
3. After the first run on `main`, open your profile's Packages tab and check the
   `gods-eye-view` package's visibility. Keep it private if you added browser
   keys; a pulled image contains them too.

Server-side keys are never given to GitHub. They exist only in `gev.env` on the
host.

### Host

Any machine with Docker: a small VPS, a home server, a NAS. About 1 GB of disk
for the image; the app idles well under 1 GB of RAM.

```bash
mkdir gods-eye-view && cd gods-eye-view
# copy deploy/compose.yaml and deploy/gev.env.example here
cp gev.env.example gev.env && chmod 600 gev.env   # then edit
docker login ghcr.io          # only while the package is private; a token with read:packages
docker compose up -d
```

To update after a push to `main`:

```bash
docker compose pull && docker compose up -d
```

To roll back, pin a published commit: `GEV_IMAGE=ghcr.io/<owner>/gods-eye-view:sha-<commit> docker compose up -d`.

### Cloudflare Tunnel (optional)

1. In Cloudflare Zero Trust, create an Access application for the hostname with
   a policy that allows only your identity. Do this first.
2. Create a tunnel, add a public hostname routed to `http://app:4173`, and copy
   the tunnel token into `tunnel.env` as `TUNNEL_TOKEN=...` (`chmod 600`).
3. Set `GEV_ALLOWED_HOSTS=<that hostname>` in `gev.env`; other names get `403`.
4. `docker compose --profile tunnel up -d`

## Keys and spend

`gev.env.example` lists the hosting subset; the root `.env.example` documents
every variable. Before adding a metered key (OpenAI, Google), set a hard usage
limit or quota at the provider. The app's `GEV_RATELIMIT_*` throttles are
in-memory request caps, not billing caps.

## Building by hand

```bash
docker build -f deploy/Dockerfile -t gods-eye-view .
# with browser keys, read from your shell environment:
docker build -f deploy/Dockerfile \
  --secret id=GOOGLE_MAPS_API_KEY,env=GOOGLE_MAPS_API_KEY \
  --secret id=CESIUM_ION_TOKEN,env=CESIUM_ION_TOKEN \
  -t gods-eye-view .
```

`deploy/Dockerfile.dockerignore` replaces the root `.dockerignore` for this
image. Its first block copies upstream's list; update it if upstream's changes.

## Keeping up with upstream

Use GitHub's "Sync fork" on `main`, or:

```bash
git remote add upstream https://github.com/bilawalsidhu/gods-eye-view.git
git fetch upstream && git merge upstream/main
```

Upstream's own `CI` workflow also runs in the fork. If a sync changes the Node
version in the root `Dockerfile`, update the `FROM` line in `deploy/Dockerfile`
to match.
