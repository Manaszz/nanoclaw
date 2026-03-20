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
