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

### Running services (dev)
- Headless server (core service): `CRAFT_SERVER_TOKEN=$(openssl rand -hex 32) bun run server:dev`.
  Listens on `ws://127.0.0.1:9100` and prints `CRAFT_SERVER_URL` / `CRAFT_SERVER_TOKEN` on
  startup. `server:dev` first builds the required subprocess bundles.
- Web UI (browser client, dev/hot-reload): `bun run webui:dev` (Vite on
  `http://localhost:5175/`). It is only a client — it needs a running server; on the login
  page enter the server URL and token printed above.
- CLI client: `bun run apps/cli/src/index.ts <cmd>` (reads `CRAFT_SERVER_URL` /
  `CRAFT_SERVER_TOKEN`). `--validate-server --url ws://127.0.0.1:9100 --token <t>` runs a
  40-step end-to-end integration test against a running server.
- Electron desktop app: `bun run electron:dev` (needs a display/X server; headless VMs
  require Xvfb).

### Non-obvious gotchas
- **No LLM API key = agent chat cannot run.** Without `ANTHROPIC_API_KEY` (or another
  provider key), the server starts and all RPC/session/workspace/source/label/skill
  operations work, but any step that streams a model response (`send`, tool use) fails.
  In `--validate-server` this shows up as ~15 expected failures ("No text_delta events") —
  the ~25 non-LLM steps still pass. This is a missing-credential condition, not a broken env.
- `bun run --validate-server` **without** `--url` auto-spawns its own server on port 9100;
  it fails with "Server process exited before printing CRAFT_SERVER_URL" if another server
  already holds 9100. Either stop the other server or pass `--url`/`--token` to reuse it.
- Known pre-existing repo issues (reproduce on `main` CI, unrelated to environment; do not
  "fix" as part of setup): `bun run typecheck:all` fails because `session-tools-core`
  extends a missing `tsconfig.base.json`; `bun run lint` aborts because
  `scripts/check-raw-sends.sh` is absent; `lint:shared`/`lint:ui` report a few code lint
  errors. The CI-relevant checks that pass are `test:shared:all`, `test:doc-tools`, and
  `lint:i18n:parity`.
- Local app data lives in `~/.craft-agent/` (config, encrypted credentials, workspaces,
  sessions). Delete it to reset to a clean state.
