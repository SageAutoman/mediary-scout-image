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
   ├── src/                       ← submodule → SageAutoman/mediary-scout
   └── .github/workflows/
       ├── sync-upstream.yml      → mirror upstream→fork, bump submodule, push main
       └── build-and-push.yml     → on push to main: build multi-platform image → GHCR
```

## How it works

1. **sync-upstream.yml** runs on a daily schedule (and manually):
   - force-mirrors `fancydirty/mediary-scout` → `SageAutoman/mediary-scout`;
   - bumps the `src` submodule in this repo to the new fork HEAD and pushes to `main`.
2. The submodule-bump push to `main` triggers **build-and-push.yml**, which:
   - checks out with submodules,
   - builds `src/Dockerfile` for `linux/amd64` + `linux/arm64` via Buildx/QEMU,
   - pushes to `ghcr.io/sageautoman/mediary-scout-image` (`latest`, short-SHA, date tags).

## Published image

```
ghcr.io/sageautoman/mediary-scout-image:latest
```

Run it:

```bash
docker run -p 3000:3000 ghcr.io/sageautoman/mediary-scout-image:latest
```

(For a full self-host stack with Postgres + PanSou see the app repo's
`docker-compose.yml`.)

## Required GitHub secret

`sync-upstream.yml` needs a **repo-scoped Personal Access Token** (classic
`repo`, or fine-grained with write on both `SageAutoman/mediary-scout` and
`SageAutoman/mediary-scout-image`) stored as the **`PAT`** secret in this repo's
Settings → Secrets and variables → Actions. `GITHUB_TOKEN` cannot push to a
different repository, so a PAT is mandatory.

`build-and-push.yml` uses the built-in `GITHUB_TOKEN` (with `packages: write`)
and needs no extra secret.

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
   prime the submodule bump and trigger the first image build.

## Submodule management

```bash
git submodule update --remote src   # pull latest fork/main into src/
git add src && git commit -m "bump src" && git push
```
