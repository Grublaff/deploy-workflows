# deploy-workflows

Reusable GitHub Actions workflow for building and publishing homelab sites that run on `home-server` behind Cloudflare Tunnel.

## What it does

Each site repo calls this workflow on push to `main`. It:

1. Best-effort runs `npm run ci` if your `package.json` defines it (non-blocking).
2. Builds a Docker image — static sites are baked into `caddy:2-alpine`; Dockerized apps use the repo's `Dockerfile`.
3. Smoke-tests the image with `docker run` + `curl` (5 retries, 2-second poll).
4. Trivy-scans for HIGH/CRITICAL vulnerabilities (skippable but recommended).
5. Pushes to `ghcr.io/<owner-lowercased>/<image-name>:latest` and `:sha-<short>`.

Watchtower on `home-server` polls GHCR every 5 minutes and restarts containers when a new `:latest` lands.

## Use it

Pin to a tagged version so the workflow can evolve without breaking your sites silently:

### Static site

Copy [`examples/static-site.yml`](examples/static-site.yml) to `.github/workflows/deploy.yml` in your repo. Edit `image-name`, `build-command`, and `build-output`.

### Dockerized app

Copy [`examples/docker-site.yml`](examples/docker-site.yml). Edit `image-name` and `port` (default 80).

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `image-name` | yes | — | Image name (becomes `ghcr.io/<owner-lc>/<image-name>`) |
| `site-type` | yes | — | `static` or `docker` |
| `build-command` | no | `""` | Static-only: build command. Leave empty if `dist/` is pre-built. |
| `build-output` | no | `dist` | Static-only: build output dir |
| `port` | no | `80` | Port for smoke test |
| `skip-trivy` | no | `false` | Skip vuln scan (do not use in production) |

## Rollback

In the `infra` repo, edit `sites/<name>/docker-compose.yml` to pin a `:sha-<short>` tag instead of `:latest`. Watchtower will not overwrite a non-`:latest` tag. To re-enable automatic updates, change back to `:latest`.

## Scheduled health check

The companion `site-health.yml` workflow probes each registered URL every 15 minutes and opens an issue in the `infra` repo on failure. Add new sites to the matrix as they come online; the `INFRA_ISSUE_TOKEN` repo secret must be a fine-grained PAT with `Issues: read+write` scoped to the `infra` repo only.
