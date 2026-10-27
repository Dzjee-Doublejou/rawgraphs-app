# Deploy RAWGraphs with Docker Compose and Coolify

This repository builds a client-side Vite app with `npm run build`.
`vite.config.js` sets the output directory to `build/`, which the
multi-stage Dockerfile copies into Nginx. The builder uses Node 22 (matching
the existing deployment workflows), the committed lockfile, and `.npmrc`
(`legacy-peer-deps=true`). The runtime serves HTTP on port 80.

No database, persistent server volume, API server, or environment variables
are required. User data is processed in the browser. Browser storage is tied
to the deployment's origin; export work before changing domains or clearing
browser storage. External data URLs, SPARQL endpoints, and custom chart CDNs
still require browser network access and suitable CORS permissions.

## Coolify

1. Merge the deployment PR, or select its branch to try it before merging.
2. In the target Coolify project/environment, add a **Git repository application**.
   Use `https://github.com/Dzjee-Doublejou/rawgraphs-app` and branch `master`
   (or the PR branch). Do not use an Empty Docker Compose service: this
   deployment needs the repository as its build context.
3. Choose the **Docker Compose** build strategy/build pack.
   Set Base Directory to `/` and Docker Compose Location to
   `/docker-compose.yml`. Save/load the Compose configuration.
4. Under Domains, select service `rawgraphs`, enter your hostname with HTTPS,
   and target internal port **80**, path `/`. Older Coolify versions use
   a domain value such as `https://rawgraphs.example.com:80`.
   Point DNS to the Coolify server and allow inbound ports 80/443.
5. Deploy. The build server needs outbound access to Docker Hub and the npm
   registry, plus enough memory for the Vite build. Nginx's image healthcheck
   probes `/healthz`; wait for the container to become healthy.
6. Open the domain, load a sample dataset, create a chart, and export SVG.
   Refresh the page and confirm the app still loads.

Coolify terminates TLS and supplies proxy networking. The Compose file uses
`expose`, with no host port binding or hard-coded container name.
Serve the app at a dedicated hostname's root; subpath hosting requires a
separate Vite base-path configuration and matching Nginx routing.

See [Coolify's Compose guide](https://coolify.io/docs/applications/builds/docker-compose)
and [domain configuration](https://coolify.io/docs/core/networking/domains).

## Local Docker smoke test

Docker Engine/Desktop with Compose v2 is required.

```sh
docker compose config --quiet
docker compose build --pull
docker compose run --rm --service-ports -p 127.0.0.1:8080:80 rawgraphs
```

In another terminal:

```sh
curl -fsS http://127.0.0.1:8080/healthz
curl -I http://127.0.0.1:8080/
curl -I http://127.0.0.1:8080/example-client-route
curl -I http://127.0.0.1:8080/assets/missing.js
```

Expect 200 for health, the homepage, and SPA fallback; 404 for a missing
bundle. Visit `http://localhost:8080` and test chart creation/export.
Stop the foreground container with Ctrl-C.

## Updates and troubleshooting

Redeploy after merging updates; dependencies are installed during image build,
not container startup. No server data migration is needed. Roll back by
redeploying a previous Git commit/image.

- **Build fails:** inspect the first npm/Vite error. Keep `.npmrc` in the
  build context and install development dependencies (Vite is one).
  Do not replace `npm ci` with an unlocked install.
- **Out of memory:** increase build-server memory or swap and retry.
- **No available server/502:** check runtime logs and health; ensure the
  domain targets `rawgraphs:80` and use Coolify-managed proxy networking.
- **Blank page:** check browser console and asset requests, then hard-refresh
  after a redeploy. HTML is revalidated; hashed bundles are cached.
- **External imports fail:** inspect the remote endpoint's HTTPS/CORS policy;
  Nginx does not proxy third-party data requests.
