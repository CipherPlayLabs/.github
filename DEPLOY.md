# Deploying a CipherPlay repo

Every CipherPlayLabs repo deploys the same way: **push to `main`, and the box
pulls, builds, publishes, and gets a TLS cert.** No per-project deploy scripts.

Two things make a repo deployable.

## 1. A manifest, at `.cipherplay/deploy.env`

```sh
DOMAIN=example.tylerw.ai   # required. the public hostname
TYPE=static                # required. static | docker
```

| Key | Applies to | Meaning |
|---|---|---|
| `DOMAIN` | all | Public hostname. TLS is issued for it automatically. |
| `TYPE` | all | `static` (files → nginx webroot) or `docker` (compose → nginx reverse proxy). |
| `ALIASES` | all | Extra names on the cert, space-separated. e.g. `www.example.com` |
| `TRIGGER` | all | `push` or `manual`. Documents intent; the actual gate is your caller workflow. |
| `HEALTH_PATH` | all | Path the post-deploy probe expects `200` from. Default `/`. |
| `BUILD` | static | `auto` (default; runs `npm run build` if present), `none`, or a literal command. |
| `PUBLISH_DIR` | static | Dir holding `index.html`. Auto-detects `dist/ build/ out/ site/ public/` if unset. |
| `SLUG` | static | Webroot name under `/var/www/`. Defaults to the lowercased repo name. |
| `PORT` | docker | Loopback port nginx proxies to. **Required** for `docker`. |
| `COMPOSE_FILE` | docker | Default `docker-compose.yml`. |

Only `PUBLISH_DIR` is exposed to the web. Everything else in the repo — `docs/`,
`brand/`, `.env` — stays off the public internet.

## 2. A caller workflow, at `.github/workflows/deploy.yml`

```yaml
name: Deploy
on:
  push:
    branches: [main]      # delete this line for manual-only deploys
  workflow_dispatch:
jobs:
  deploy:
    uses: CipherPlayLabs/.github/.github/workflows/deploy.yml@main
    secrets: inherit
```

That's it. `secrets: inherit` picks up the org-level `DEPLOY_SSH_HOST` and
`DEPLOY_SSH_KEY`; you never paste a key into a repo.

## Certificates

TLS is issued automatically **only once DNS already points at the box**
(`176.9.219.106`). The deployer checks the A record first:

- **A record correct** → certbot issues, nginx switches to HTTPS with redirect.
- **No A record, or pointing elsewhere** → cert is **skipped with a warning**, and
  the site stays up on plain HTTP. Add the DNS record and re-run; nothing is lost.

This is deliberate. Asking Let's Encrypt to validate a name that doesn't resolve
here burns a failure against a rate limit that is painful to hit.

Renewals are handled by the existing system certbot timer. Nothing to do.

## How it works on the box

```
GitHub Actions ──ssh(deploy@box, forced command)──► cpdeploy <Repo> <ref>
                                                        │  runs UNPRIVILEGED
                                                        │  git pull → npm build
                                                        ▼
                                                   sudo cpdeploy-priv
                                                      publish | site-static
                                                      site-proxy | cert | compose
```

- Repos live at **`/srv/<ExactRepoName>`** — one place, matching the GitHub name.
- Static output is mirrored to `/var/www/<slug>`.
- The box holds **no GitHub credentials**. Actions hands it a short-lived
  `GITHUB_TOKEN` over stdin, scoped to that one repo, and it's scrubbed after the
  clone.
- The deploy key is pinned to a **forced command**: it can run `cpdeploy` and
  nothing else. No shell, no port forwarding.
- `deploy` is an unprivileged user. Its only root power is `cpdeploy-priv`, which
  re-validates every argument it is given.

### Security note: `TYPE=docker` is root-equivalent

Building a container runs the repo's `Dockerfile`, and Docker builds as root.
**Anyone who can push to a repo deployed with `TYPE=docker` can get root on the
box.** The sudo allowlist does not contain this — it can't; it's inherent to
"build a container from a repo."

`TYPE=static` does **not** have this property: builds run as the unprivileged
`deploy` user, and only the finished files cross into root.

So: keep `main` protected on any `docker` repo, and treat push access to those
repos as equivalent to handing out server access. If that's not acceptable for a
given project, build the image in CI and deploy a pinned digest instead.

## Manual deploy, from the box

```sh
sudo -u deploy cpdeploy TheBlackIris main
```

Requires the box's `cipherplay-org-deploy` public key to be registered on GitHub
(read-only deploy key on the repo, or on a machine user with org read access).
The Actions path does **not** need this.
