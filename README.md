# mediary-scout-image

CI/CD repository that builds and publishes the **media-track** (`mediary-scout`)
Docker image to GitHub Container Registry (GHCR).

This repo does **not** contain application source. The source lives in a git
submodule so it can be kept in sync with upstream automatically.

```
fancydirty/mediary-scout          (upstream / source of truth)
        │  sync-upstream.yml (daily)
        ▼
SageAutoman/mediary-scout        (your fork — a pure mirror of upstream)
        │  submodule `src/`
        ▼
SageAutoman/mediary-scout-image  (this repo)
   ├── src/                        ← submodule → SageAutoman/mediary-scout
   ├── docker-compose.yml          ← self-host stack; `web` uses the GHCR image
   ├── deploy/pansou.channels.env  ← PanSou 115/网盘源频道默认配置
   └── .github/workflows/
       ├── sync-upstream.yml       → mirror upstream→fork, bump submodule, push main
       └── build-and-push.yml      → on Git tag push: build multi-platform image → GHCR
```

## How it works

1. **sync-upstream.yml** runs on a daily schedule (and manually):
   - force-mirrors `fancydirty/mediary-scout` → `SageAutoman/mediary-scout`;
   - bumps the `src` submodule in this repo to the new fork HEAD and pushes to `main`.
2. Pushing a Git tag triggers **build-and-push.yml**, which:
   - checks out with submodules,
   - builds `src/Dockerfile` for `linux/amd64` + `linux/arm64` via Buildx/QEMU,
   - pushes to `ghcr.io/sageautoman/mediary-scout-image` (`latest`, Git tag, short-SHA).

Ordinary `main` pushes, including automated submodule bumps, do not build images.

## Published image

```
ghcr.io/sageautoman/mediary-scout-image:latest          # 最新构建
ghcr.io/sageautoman/mediary-scout-image:<git-tag>       # 对应发布 tag
ghcr.io/sageautoman/mediary-scout-image:sha-<commit>    # 锁定某次构建
```

The image is **public** — `docker pull` works without login.

## Self-hosting (full stack)

This repo ships a `docker-compose.yml` that pulls the **prebuilt** `web` image
(no local build) and wires up Postgres + PanSou. It's adapted from the app
repo's own `docker-compose.yml`; the only change is `web` uses the GHCR image
instead of building from source.

```bash
# 拉取最新镜像并起栈
docker compose pull
docker compose up -d
# 打开 http://<host>:3000 → 设置页扫码连 115 即用
# 自定义主机端口: WEB_PORT=8080 docker compose up -d
```

What's included:
- **postgres** — `postgres:16-alpine`, data in the `pgdata` volume.
- **pansou** — `ghcr.io/fish2018/pansou-web:latest`, seeded with good 115/网盘源
  channels from `deploy/pansou.channels.env` (copy it next to the compose file;
  without it PanSou returns zero 115 shares).
- **web** — the GHCR image above. Env vars (`MEDIA_TRACK_POSTGRES_URL`,
  `PANSOU_BASE_URL`, agent/storage adapters, …) are pre-set; tune in `.env`
  (optional, loaded if present) or the 设置 page.
- **cloudflared** — optional, off by default; `docker compose --profile tunnel up -d`
  for a Cloudflare Tunnel (no open ports). Put `TUNNEL_TOKEN` in `.env`.

> The image bakes `MEDIA_TRACK_ALLOWED_ORIGINS` and `GIT_SHA` (as `BUILD_COMMIT`)
> at build time. To verify the running container serves the expected commit:
> `docker compose exec web cat BUILD_COMMIT`.

Just the image, no stack:

```bash
docker run -p 3000:3000 ghcr.io/sageautoman/mediary-scout-image:latest
```

## Required GitHub secret

`sync-upstream.yml` needs a **Personal Access Token** stored as the **`PAT`**
secret: classic tokens need `repo` + `workflow`; fine-grained tokens need
Contents + Workflows write access on both `SageAutoman/mediary-scout` and
`SageAutoman/mediary-scout-image`. `GITHUB_TOKEN` cannot push to a different
repository, so a PAT is mandatory.

`build-and-push.yml` logs in to GHCR with the built-in **`GITHUB_TOKEN`**
(the repo's workflow-permission policy is set to `write`, so it carries
`packages: write`). Pushing with `GITHUB_TOKEN` links the package to this repo,
so it shows in the repo's **Packages** tab and its visibility can be changed
there. The `PAT` secret is only used by `sync-upstream.yml` to push to the fork.

## First-time setup (one time, on github.com)

1. Create the repo **SageAutoman/mediary-scout-image** (public or private).
2. Enable GHCR: Settings → Packages is on by default; the image repo
   (`ghcr.io/sageautoman/mediary-scout-image`) is auto-created on first push.
3. Add the **`PAT`** secret described above.
4. Push this local repo:
   ```bash
   git remote add origin https://github.com/SageAutoman/mediary-scout-image.git
   git push -u origin main
   ```
5. (Optional) Run **Sync Upstream** manually once from the Actions tab to
   prime the submodule bump.
6. Push a release tag to build and publish the image:
   ```bash
   git tag v1.0.0
   git push origin v1.0.0
   ```

## Submodule management

```bash
git submodule update --remote src   # pull latest fork/main into src/
git add src && git commit -m "bump src" && git push
```
