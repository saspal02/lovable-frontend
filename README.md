# Lovable — Frontend

Web client for the Lovable platform. An AI pair-programming workspace: chat with an agent
about a project, watch it edit files in a live editor, and deploy a preview.

Serves the UI only. It is a static SPA; all data comes from the platform's HTTP API.

- **Live site:** <https://lovable.in>
- **API gateway:** <https://api.lovable.in>
- **Backend:** [`saspal02/vibe-coding-platform`](https://github.com/saspal02/vibe-coding-platform)
  (api-gateway, account-service, workspace-service, intelligence-service, discovery-service)

## Stack

| | |
|---|---|
| Framework | React 18 + TypeScript |
| Build | Vite 5 |
| Routing | react-router-dom 6 (`BrowserRouter`, HTML5 history) |
| Data | TanStack Query |
| UI | shadcn/ui on Tailwind CSS 3 + Radix |
| Editor | CodeMirror 6 |
| Charts | Recharts |
| Tests | Vitest + Testing Library |

## Local development

```sh
npm install
npm run dev          # http://localhost:5173
```

The dev server binds `::` and disables the HMR error overlay so it works inside containers.

### Configuration

| Variable | Required | Default | Purpose |
|---|---|---|---|
| `VITE_API_URL` | no | `http://api.codingshuttle.in` | API gateway base URL. Read once in `src/lib/api.ts`. |

Vite inlines `VITE_*` variables **at build time**, so changing the API URL requires a rebuild —
not a runtime env var.

To target the platform's own gateway locally:

```sh
VITE_API_URL=http://api.lovable.in npm run dev
```

## Scripts

| Command | Description |
|---|---|
| `npm run dev` | Dev server with hot reload |
| `npm run build` | Production build to `dist/` |
| `npm run build:dev` | Development-mode build (enables the Lovable component tagger) |
| `npm run preview` | Serve the production build locally |
| `npm run lint` | ESLint across the project |
| `npm run test` | Vitest, single run |

## Docker

The image is a two-stage build: Node compiles the SPA, then the static output is copied into
nginx, which serves it with SPA fallback.

```sh
docker build -t saspal02/lovable-frontend:latest .
```

Point the build at a different gateway:

```sh
docker build \
  --build-arg VITE_API_URL=http://api.lovable.in \
  -t saspal02/lovable-frontend:latest .
```

`VITE_API_URL` defaults to `http://api.lovable.in`.

### Run

```sh
docker run --rm -p 8080:80 saspal02/lovable-frontend:latest
```

### What the image ships

- **Port 80** to match the `lovable-frontend` Service in `k8s/services/frontend.yaml`
- **SPA fallback** — `nginx.conf` rewrites unknown paths to `/index.html`, so deep links like
  `/projects/abc123` load correctly on refresh. This is required: the app uses HTML5 history routing
- **Caching** — `/assets/*` is content-hashed and served `immutable` for 1 year; `index.html` is
  served `no-cache` so new deploys are picked up immediately
- **gzip** for text, JS, CSS, SVG, and JSON
- **`HEALTHCHECK`** polling `/`, as the Kubernetes manifests define no probes

## Routes

| Path | View |
|---|---|
| `/` | Redirects to `/projects`, or `/login` when unauthenticated |
| `/login` | Sign in |
| `/signup` | Register |
| `/projects` | Project dashboard |
| `/projects/:projectId` | Workspace: chat, file tree, editor, preview |
| `*` | 404 |

## Deployment

The image is deployed to the `lovable-core` namespace in Kubernetes and fronted by the
`lovable-main-ingress` Ingress, which routes `lovable.in` and `www.lovable.in` to it and
`api.lovable.in` to `api-gateway`.

```sh
docker push docker.io/saspal02/lovable-frontend:latest
```

The Deployment uses `imagePullPolicy: Always`, so a re-pulled image rolls out on the next
restart of the pod.
