# AGENTS.md

## Cursor Cloud specific instructions

### Project overview

NanoClaw is a lightweight personal AI assistant that runs Claude agents in isolated Docker containers. See `README.md` for full philosophy and `CLAUDE.md` for key files and skills.

### Development commands

Standard commands are documented in `package.json` scripts and `CLAUDE.md`. Key ones:

- `npm run dev` — Run with hot reload (tsx)
- `npm run build` — Compile TypeScript
- `npm run typecheck` — Type checking without emit
- `npm run format:check` / `npm run format:fix` — Prettier lint
- `npm test` — Run vitest unit tests (219 tests across 18 files)
- `./container/build.sh` — Rebuild the agent Docker container image

### Cloud VM caveats

- **Docker is required** for `npm run dev` to start. The process calls `docker info` at startup and exits immediately if Docker is unavailable. Ensure Docker daemon is running and the current user has socket access (`sudo chmod 666 /var/run/docker.sock` if needed).
- **No channels = immediate exit.** Without any messaging channel credentials configured (WhatsApp, Telegram, etc.), the process logs `FATAL: No channels connected` and exits with code 1. This is expected for a clean install.
- **The `.env` file** stores `ANTHROPIC_API_KEY` or `CLAUDE_CODE_OAUTH_TOKEN`. The credential proxy reads secrets from `.env` (never `process.env`) and injects them into container API calls.
- **Container image must be built** before agents can run: `./container/build.sh` (requires Docker).
- **Pre-commit hook** runs `npm run format:fix` automatically via Husky.
- **CI checks** (see `.github/workflows/ci.yml`): format:check, tsc --noEmit, vitest run.

### Docker setup in Cloud VM

Docker must be installed and configured for the nested Docker-in-Docker environment:

1. Install Docker engine packages
2. Install `fuse-overlayfs` and configure `/etc/docker/daemon.json` with `"storage-driver": "fuse-overlayfs"`
3. Switch to `iptables-legacy` via `update-alternatives`
4. Start dockerd: `sudo dockerd &>/tmp/dockerd.log &`
5. Fix socket permissions: `sudo chmod 666 /var/run/docker.sock`

### Hello world verification

To verify the full pipeline (credential proxy + container + Claude Agent SDK):

1. Ensure `ANTHROPIC_API_KEY` is in `.env`
2. Build the container: `./container/build.sh`
3. Start `npm run dev` — should show "Credential proxy started" with `authMode: "api-key"`
4. The process exits with "No channels connected" unless a messaging channel is configured (this is expected)
5. All 219 unit tests pass without Docker or API key: `npm test`

### Container image

The agent container image (`nanoclaw-agent:latest`, ~1.6GB) includes Node.js 22, Chromium, `agent-browser`, and `@anthropic-ai/claude-code`. It compiles agent-runner TypeScript on each boot from `/app/src` (mounted per-group for customization). Build time is ~60s.
