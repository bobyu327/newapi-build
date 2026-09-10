# newapi-build

CI that builds our patched [New API](https://github.com/QuantumNous/new-api) image
and pushes it to `ghcr.io/bobyu327/new-api`.

Runs on GitHub Actions so the resource-constrained Seoul box (2 vCPU / 2 GB) never
has to compile it.

## What it does

1. Shallow-clones upstream `QuantumNous/new-api` at a pinned tag (default `v1.0.0-rc.34`).
2. Applies the patches in `patches/` (mailbox format, `git am`), in order.
3. Builds with upstream's own multi-stage `Dockerfile` (bun web build + Go build).
4. Pushes `ghcr.io/bobyu327/new-api:<tag>-schemaclean` and `:latest`.

## Patches

| file | commit | why |
|---|---|---|
| `0001-…headers…` | `459e1349` | reset runtime header-override between cross-channel retries (OC Go `x-opencode-session` lost on fallback) |
| `0002-…toolconv…schema…` | `163f1519` | strip `pattern` / `$schema` from tool JSON-Schema at the three toolconv attach points — DeepSeek V4.1 Flash / OpenCode "Console Go" / GPT-5.6-luna hard-400 on Claude Code v2.1.267's `Artifact` tool `pattern` (negative lookahead + `\p{Cc}`) |

## Use it

- **Rebuild**: Actions → *build* → *Run workflow* (optionally set a different `base_tag`).
- **Deploy on Seoul**:
  ```bash
  docker pull ghcr.io/bobyu327/new-api:v1.0.0-rc.34-schemaclean
  docker stop new-api && docker rm new-api
  docker run -d --name new-api --restart unless-stopped -p 3000:3000 \
    -v /opt/new-api/data:/data -e TZ=Asia/Shanghai \
    ghcr.io/bobyu327/new-api:v1.0.0-rc.34-schemaclean
  ```

## Bumping upstream

Bump `base_tag`, re-run. If a patch no longer applies cleanly, rebase it against the
new tag locally (`/opt/new-api` branch `newapi-rc34-schemaclean`), regenerate with
`git format-patch`, replace the file here. Mind upstream DB-migration jumps
(rc.28–30 had migration bugs — the pool doc notes this).
