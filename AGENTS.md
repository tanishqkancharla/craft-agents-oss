# AGENTS.md

## Cursor Cloud specific instructions

Craft Agents is a Bun-based TypeScript monorepo (`apps/*`, `packages/*`). The primary
product is the Electron desktop app; the same UI is also served as a browser Web UI by a
headless WebSocket/RPC server. See `README.md` and `CONTRIBUTING.md` for full docs and
`package.json` for the canonical script list.

### Runtime/tooling
- Use **Bun** (pinned to `1.3.10`, matching CI) as the package manager and runtime; `node`
  is also present and is required for some subprocesses (e.g. the WhatsApp worker).
- Python doc-tool tests use `python3` (3.12 available); CI additionally installs `uv`.
- Bun and uv are installed to `~/.bun/bin` and `~/.local/bin` and added to PATH via
  `~/.bashrc`; a fresh non-login shell may need `export PATH="$HOME/.bun/bin:$HOME/.local/bin:$PATH"`.
- If `pi-agent-server` fails to build with `Could not resolve: "jiti/static"`, do a clean
  reinstall (`rm -rf node_modules && bun install --frozen-lockfile`). Nested `jiti@2.7.0`
  under `@earendil-works/pi-coding-agent` can be missing after a partial install.

### Running services (dev)
- Headless server (core service): `CRAFT_SERVER_TOKEN=$(openssl rand -hex 32) bun run server:dev`.
  Listens on `ws://127.0.0.1:9100` and prints `CRAFT_SERVER_URL` / `CRAFT_SERVER_TOKEN` on
  startup. `server:dev` first builds the required subprocess bundles.
- **Server + Web UI (recommended for browser testing):**
  `CRAFT_SERVER_TOKEN=<token> bun run server:prod`. Serves RPC + the built Web UI on
  `http://localhost:9100` (login page asks only for the server token). Prefer this over
  `webui:dev` alone — Vite on `:5175` has no `/api/auth` proxy, so login fails there.
- CLI client: `bun run apps/cli/src/index.ts <cmd>` (reads `CRAFT_SERVER_URL` /
  `CRAFT_SERVER_TOKEN`). Self-contained smoke: `bun run apps/cli/src/index.ts run --api-key "$ANTHROPIC_API_KEY" "…"`.
  Integration suite: `bun run apps/cli/src/index.ts --validate-server` (auto-spawns a
  server; stop anything on 9100 / delete `~/.craft-agent/.server.lock` first, or pass
  `--url`/`--token` to reuse a running server).
- Electron desktop app: `bun run electron:dev` (needs a display/X server; headless VMs
  require Xvfb).

### Non-obvious gotchas
- **`ANTHROPIC_API_KEY` is required for agent chat.** With it set, CLI `run` and Web UI
  chat produce real model replies, and `--validate-server` typically reaches ~35/40
  (remaining failures are usually missing `.github/agents/automations.json` and webhook
  network, not the env). Without it, RPC/session/workspace ops still work but any
  streaming `send`/tool-use step fails ("No text_delta events").
- Only one server instance may hold `~/.craft-agent/.server.lock`. Parallel instances need
  a different `CRAFT_CONFIG_DIR`.
- Known pre-existing repo issues (reproduce on upstream `main`, unrelated to environment;
  do not "fix" as part of setup): `bun run typecheck:all` fails because
  `session-tools-core` extends a missing `tsconfig.base.json`; `bun run lint` aborts
  because `scripts/check-raw-sends.sh` is absent. CI-relevant checks that pass:
  `test:shared:all`, `test:doc-tools`, and `lint:i18n:parity`.
- Local app data lives in `~/.craft-agent/` (config, encrypted credentials, workspaces,
  sessions). Delete it to reset to a clean state.
