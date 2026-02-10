# Repository Guidelines

- Repo: https://github.com/openclaw/openclaw
- GitHub issues/comments/PR comments: use literal multiline strings or `-F - <<'EOF'` (or $'...') for real newlines; never embed "\\n".

## What Is OpenClaw

OpenClaw is a **personal AI assistant platform** that runs on your own devices. It acts as a multi-channel gateway connecting to messaging platforms (WhatsApp, Telegram, Slack, Discord, Google Chat, Signal, iMessage, Microsoft Teams, Matrix, Zalo, and more) and provides voice/chat capabilities on macOS, iOS, and Android. The embedded Pi agent runtime (Claude-based) powers tool use, skills, memory, and coding assistance.

- **Website:** https://openclaw.ai | **Docs:** https://docs.openclaw.ai | **Discord:** https://discord.gg/clawd
- **License:** MIT
- **Versioning:** Calendar-based (`YYYY.M.D`, e.g. `2026.2.9`)

## Technology Stack

| Layer | Technology |
|---|---|
| Language | TypeScript (ESM, strict mode) |
| Runtime | Node.js 22+ (Bun supported for scripts/dev) |
| Package manager | pnpm 10.x (workspaces) |
| Server | Express 5, WebSocket (ws) |
| AI runtime | Pi agent (`@mariozechner/pi-agent-core`) with Anthropic, OpenAI, Bedrock, Gemini providers |
| Database | SQLite + `sqlite-vec` (vector search) |
| Linter | Oxlint (Rust-based, type-aware) |
| Formatter | Oxfmt |
| Test framework | Vitest 4 (V8 coverage) |
| Build | tsdown + Rolldown |
| UI framework | Lit 3 (Web Components) + Vite |
| Native apps | Swift/SwiftUI (macOS, iOS), Kotlin/Gradle (Android) |
| Validation | Zod 4 |
| CI/CD | GitHub Actions, Docker, Fly.io |

## Project Structure & Module Organization

```
openclaw/
├── src/                    # Core TypeScript source
│   ├── agents/             # Pi agent runtime (models, tools, skills, sandbox)
│   ├── gateway/            # WebSocket gateway (auth, sessions, routing, webhooks, bridge)
│   ├── commands/           # CLI commands (gateway, agent, send, wizard, doctor, config)
│   ├── channels/           # Core channel abstractions & shared routing logic
│   ├── routing/            # Message routing engine
│   ├── sessions/           # Session management (main, group, activation/queue)
│   ├── providers/          # Model providers (Anthropic, OpenAI, etc.)
│   ├── media/              # Media pipeline (images, audio, video, transcription)
│   ├── memory/             # Memory & knowledge base (SQLite + vector)
│   ├── config/             # Configuration management
│   ├── plugin-sdk/         # Plugin SDK for extensions
│   ├── plugins/            # Plugin management
│   ├── security/           # Security checks
│   ├── telegram/           # Telegram channel (grammY)
│   ├── discord/            # Discord channel
│   ├── slack/              # Slack channel (Bolt)
│   ├── signal/             # Signal channel
│   ├── imessage/           # iMessage channel
│   ├── web/                # WhatsApp web provider (Baileys)
│   ├── browser/            # Browser automation (Playwright)
│   ├── tts/                # Text-to-speech (ElevenLabs, edge-tts)
│   ├── terminal/           # Terminal/PTY integration + table/palette utils
│   ├── tui/                # Terminal UI
│   ├── wizard/             # Setup wizard
│   ├── infra/              # Infrastructure (Tailscale, Firebase, AWS)
│   ├── cli/                # CLI wiring + progress utilities
│   ├── shared/             # Shared utilities
│   ├── types/              # TypeScript types
│   └── utils/              # General utilities
├── extensions/             # 34+ channel/feature plugins (workspace packages)
│   ├── discord/            # Discord extension
│   ├── telegram/           # Telegram extension
│   ├── slack/              # Slack extension
│   ├── signal/             # Signal extension
│   ├── matrix/             # Matrix extension
│   ├── msteams/            # Microsoft Teams extension
│   ├── googlechat/         # Google Chat extension
│   ├── whatsapp/           # WhatsApp extension
│   ├── zalo/               # Zalo extension
│   ├── voice-call/         # Voice call extension
│   ├── memory-core/        # Memory interface
│   ├── memory-lancedb/     # LanceDB vector store
│   ├── llm-task/           # LLM task execution
│   └── ...                 # mattermost, nextcloud-talk, nostr, twitch, etc.
├── skills/                 # 50+ bundled skills (gemini, notion, trello, github, etc.)
├── apps/                   # Native applications
│   ├── macos/              # macOS menu bar app (Swift/SwiftUI)
│   ├── ios/                # iOS app (Swift, alpha)
│   ├── android/            # Android app (Kotlin/Gradle)
│   └── shared/             # Shared Swift code (OpenClawKit)
├── ui/                     # Control UI (Lit Web Components + Vite)
├── docs/                   # Mintlify documentation site
├── scripts/                # Build, test, CI, release scripts
├── vendor/                 # Vendored code (a2ui Canvas library)
├── packages/               # Legacy compatibility shims (clawdbot, moltbot)
├── .github/                # GitHub Actions CI/CD workflows
├── openclaw.mjs            # CLI entry point
├── Dockerfile              # Production Docker image
├── docker-compose.yml      # Docker Compose (gateway + CLI)
└── fly.toml                # Fly.io deployment config
```

### Key Workspace Packages

Defined in `pnpm-workspace.yaml`: root (`.`), `ui`, `packages/*`, `extensions/*`.

### Source Code Conventions

- Tests: colocated `*.test.ts` next to source files.
- Docs: `docs/` (Mintlify). Built output: `dist/`.
- Plugins/extensions: `extensions/*` (workspace packages). Keep plugin-only deps in the extension `package.json`; do not add them to root unless core uses them.
- Plugins: install runs `npm install --omit=dev` in plugin dir; runtime deps must live in `dependencies`. Avoid `workspace:*` in `dependencies` (npm install breaks); put `openclaw` in `devDependencies` or `peerDependencies` instead (runtime resolves `openclaw/plugin-sdk` via jiti alias).
- Installers served from `https://openclaw.ai/*`: live in the sibling repo `../openclaw.ai` (`public/install.sh`, `public/install-cli.sh`, `public/install.ps1`).
- Messaging channels: always consider **all** built-in + extension channels when refactoring shared logic (routing, allowlists, pairing, command gating, onboarding, docs).
  - Core channel docs: `docs/channels/`
  - Core channel code: `src/telegram`, `src/discord`, `src/slack`, `src/signal`, `src/imessage`, `src/web` (WhatsApp web), `src/channels`, `src/routing`
  - Extensions (channel plugins): `extensions/*` (e.g. `extensions/msteams`, `extensions/matrix`, `extensions/zalo`, `extensions/zalouser`, `extensions/voice-call`)
- When adding channels/extensions/apps/docs, update `.github/labeler.yml` and create matching GitHub labels (use existing channel/extension label colors).

## Architecture Overview

### Gateway Control Plane (`src/gateway/`)

WebSocket-based server handling sessions, presence, routing, webhooks. Serves the Control UI and provides a bridge for RPC communication. Native apps and Control UI connect here.

### Multi-Agent Routing (`src/routing/`, `src/agents/`)

Per-channel/account/peer routing to isolated agents. Each agent gets its own workspace + Pi agent session. Messages are routed based on channel, peer, and group context.

### Pi Agent Runtime (`src/agents/`)

Embedded Claude-based coding agent with tool streaming, block streaming, model failover, and auth profile rotation. Supports sandbox execution (Docker/local).

### Channels as Plugins (`extensions/`)

34+ channel implementations as workspace packages with shared routing logic, per-channel pairing, and allowlists.

### Skills System (`src/agents/skills*`, `skills/`)

50+ bundled skills, managed skills (ClawHub), and workspace-specific skills. Each agent gets its own skill snapshot.

### Session Model (`src/sessions/`)

`main` session for direct chats, group isolation, activation/queue modes, per-agent message history.

### Request Data Flow

The diagram below shows how a message flows through the system, from arrival to response delivery. Two entry paths converge on the same agent pipeline.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         ENTRY POINTS                                    │
│                                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────┐  ┌──────────────┐  │
│  │   Telegram    │  │   Discord    │  │   Slack    │  │  WhatsApp    │  │
│  │  (grammY)     │  │ (discord.js) │  │  (Bolt)    │  │  (Baileys)   │  │
│  └──────┬───────┘  └──────┬───────┘  └─────┬──────┘  └──────┬───────┘  │
│         │                 │                │                │          │
│         ▼                 ▼                ▼                ▼          │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │              Channel Message Handler                             │  │
│  │  src/{telegram,discord,slack,web}/bot-message*.ts                │  │
│  │  • Parse platform-specific update into MsgContext                │  │
│  │  • Resolve media (stickers, images, voice)                      │  │
│  │  • Debounce / media-group assembly                              │  │
│  └──────────────────────────┬───────────────────────────────────────┘  │
│                             │                                          │
│  ┌──────────────┐  ┌───────┴──────┐  ┌──────────────────────────────┐  │
│  │  macOS App   │  │  iOS / Android│  │  Control UI (Lit + Vite)    │  │
│  │  (SwiftUI)   │  │  (Swift/     │  │  ui/                         │  │
│  │              │  │   Kotlin)    │  │                              │  │
│  └──────┬───────┘  └──────┬───────┘  └──────────────┬───────────────┘  │
│         │                 │                          │                 │
│         ▼                 ▼                          ▼                 │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │              WebSocket Gateway                                   │  │
│  │  src/gateway/server/ws-connection/message-handler.ts             │  │
│  │  • Authenticate handshake (connect frame + token/pairing)       │  │
│  │  • Route to handleGatewayRequest() by method name               │  │
│  │  • chat.send → chat handler │ send → outbound │ agent → agent   │  │
│  └──────────────────────────┬───────────────────────────────────────┘  │
│                             │                                          │
└─────────────────────────────┼──────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      ROUTING LAYER                                      │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │  Agent Route Resolution                                          │  │
│  │  src/routing/resolve-route.ts :: resolveAgentRoute()             │  │
│  │  Priority: peer binding → thread parent → guild → account →     │  │
│  │            channel binding → default agent                       │  │
│  └──────────────────────────┬───────────────────────────────────────┘  │
│                             │                                          │
│  ┌──────────────────────────▼───────────────────────────────────────┐  │
│  │  Session Key Resolution                                          │  │
│  │  src/routing/ :: buildAgentSessionKey()                          │  │
│  │  Key = agentId + channel + accountId + peer + dmScope            │  │
│  └──────────────────────────┬───────────────────────────────────────┘  │
│                             │                                          │
│  ┌──────────────────────────▼───────────────────────────────────────┐  │
│  │  Session Lane Queue                                              │  │
│  │  src/agents/pi-embedded-runner/lanes.ts                          │  │
│  │  • One-at-a-time per session (prevents concurrent runs)         │  │
│  │  • Global lane for cross-session concurrency limits             │  │
│  └──────────────────────────┬───────────────────────────────────────┘  │
│                             │                                          │
└─────────────────────────────┼──────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    MESSAGE PROCESSING                                   │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │  Reply Orchestrator                                              │  │
│  │  src/auto-reply/reply/get-reply.ts :: getReplyFromConfig()       │  │
│  │  1. initSessionState() — load/create session transcript         │  │
│  │  2. Resolve directives (inline /think, /model commands)         │  │
│  │  3. Apply media understanding (vision, link extraction)         │  │
│  │  4. Build system prompt with group/channel context              │  │
│  │  5. Snapshot skills for agent                                   │  │
│  └──────────────────────────┬───────────────────────────────────────┘  │
│                             │                                          │
│  ┌──────────────────────────▼───────────────────────────────────────┐  │
│  │  Agent Runner                                                    │  │
│  │  src/auto-reply/reply/agent-runner.ts :: runReplyAgent()         │  │
│  │  • Manages typing indicators to channel                         │  │
│  │  • Calls runAgentTurnWithFallback() (provider failover)         │  │
│  └──────────────────────────┬───────────────────────────────────────┘  │
│                             │                                          │
│  ┌──────────────────────────▼───────────────────────────────────────┐  │
│  │  Pi Embedded Agent                                               │  │
│  │  src/agents/pi-embedded-runner/run.ts :: runEmbeddedPiAgent()    │  │
│  │  • Load SessionManager with transcript history                  │  │
│  │  • Call Pi SDK with messages + tool definitions                 │  │
│  │  • Stream tool calls → execute → return results                 │  │
│  │  • Auto-compact on context overflow                             │  │
│  │  • Auth profile rotation on rate limit                          │  │
│  │  • Returns ReplyPayload[] (text + media)                        │  │
│  └──────────────────────────┬───────────────────────────────────────┘  │
│                             │                                          │
│              ┌──────────────┼──────────────┐                           │
│              ▼              ▼              ▼                            │
│  ┌────────────────┐ ┌────────────┐ ┌────────────────┐                  │
│  │  Tool Calls    │ │  Skills    │ │  Model Provider │                  │
│  │  (bash, web,   │ │  (50+      │ │  (Anthropic,    │                  │
│  │   file, etc.)  │ │  bundled)  │ │  OpenAI, etc.)  │                  │
│  │  src/agents/   │ │  skills/   │ │  src/providers/ │                  │
│  │  tools/        │ │            │ │                 │                  │
│  └────────────────┘ └────────────┘ └────────────────┘                  │
│                                                                         │
└─────────────────────────────┬───────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    RESPONSE DELIVERY                                    │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │  Payload Formatter                                               │  │
│  │  src/auto-reply/reply/agent-runner-payloads.ts                   │  │
│  │  • Chunk text into channel-sized pieces (e.g. 4000 for TG)     │  │
│  │  • Format markdown per channel capabilities                     │  │
│  └───────────────┬──────────────────────────┬───────────────────────┘  │
│                  │                          │                          │
│     Channel Path │              WebSocket Path                        │
│                  ▼                          ▼                          │
│  ┌────────────────────────┐  ┌──────────────────────────────────────┐  │
│  │  Channel Delivery      │  │  WebSocket Broadcast                 │  │
│  │  src/{channel}/        │  │  src/gateway/server-chat.ts          │  │
│  │  bot/delivery.ts       │  │  • emitChatDelta() — streaming      │  │
│  │  • sendMessage()       │  │  • emitChatFinal() — complete       │  │
│  │  • sendPhoto()         │  │  • broadcast("chat", payload)       │  │
│  │  • Per-platform API    │  │  → All connected WS clients         │  │
│  └────────┬───────────────┘  └──────────────┬───────────────────────┘  │
│           │                                 │                          │
│           ▼                                 ▼                          │
│  ┌────────────────────┐          ┌──────────────────────┐              │
│  │  Telegram / Discord│          │  macOS / iOS /       │              │
│  │  Slack / WhatsApp  │          │  Android / Control UI│              │
│  │  (user sees reply) │          │  (user sees reply)   │              │
│  └────────────────────┘          └──────────────────────┘              │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │  Session Persistence                                             │  │
│  │  ~/.openclaw/agents/<agentId>/sessions/*.jsonl                   │  │
│  │  • Append user message + assistant response to transcript       │  │
│  │  • Update usage stats (tokens, model, provider)                 │  │
│  │  • Update session metadata                                      │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

#### WebSocket Protocol (native apps & Control UI)

```
Client                          Gateway
  │                                │
  │◄─── connect.challenge ─────────│  (nonce + timestamp)
  │                                │
  │──── req: connect ─────────────►│  (token/pairing auth)
  │◄─── res: ok ──────────────────│
  │                                │
  │──── req: chat.send ──────────►│  (sessionKey + message)
  │◄─── event: chat (delta) ──────│  (streaming chunks)
  │◄─── event: chat (delta) ──────│
  │◄─── event: chat (final) ──────│  (complete response)
  │                                │
  │──── req: chat.abort ─────────►│  (cancel running agent)
  │◄─── res: ok ──────────────────│
```

## Build, Test, and Development Commands

- Runtime baseline: Node **22+** (keep Node + Bun paths working).
- Install deps: `pnpm install`
- Pre-commit hooks: `prek install` (runs same checks as CI)
- Also supported: `bun install` (keep `pnpm-lock.yaml` + Bun patching in sync when touching deps/patches).
- Prefer Bun for TypeScript execution (scripts, dev, tests): `bun <file.ts>` / `bunx <tool>`.
- Run CLI in dev: `pnpm openclaw ...` (bun) or `pnpm dev`.
- Node remains supported for running built output (`dist/*`) and production installs.
- Mac packaging (dev): `scripts/package-mac-app.sh` defaults to current arch. Release checklist: `docs/platforms/mac/release.md`.

### Build

```bash
pnpm build              # Full build (Canvas bundle + tsdown + plugin-sdk DTS + copy steps)
pnpm ui:build           # Build Control UI (Lit + Vite)
pnpm protocol:gen       # Generate protocol schema
pnpm protocol:gen:swift # Generate Swift protocol models
```

### Check & Lint

```bash
pnpm check              # format:check + tsgo + lint (run before commits)
pnpm tsgo               # TypeScript type-check
pnpm lint               # Oxlint (type-aware)
pnpm lint:fix           # Auto-fix + format
pnpm format             # Oxfmt auto-format
pnpm check:loc          # Check files stay under 500 LOC guideline
```

### Test

```bash
pnpm test               # Parallel unit tests (Vitest, forks pool)
pnpm test:watch         # Watch mode
pnpm test:coverage      # V8 coverage (70% lines/functions/statements, 55% branches)
pnpm test:e2e           # End-to-end tests
pnpm test:live          # Live tests (real APIs, requires CLAWDBOT_LIVE_TEST=1)
pnpm test:docker:all    # Docker-based integration tests
```

### Development Servers

```bash
pnpm openclaw            # Run CLI (dev mode)
pnpm gateway:watch       # Watch + auto-rebuild gateway
pnpm gateway:dev         # Dev gateway (no channels)
pnpm ui:dev              # Control UI dev server (Vite HMR)
pnpm tui:dev             # Terminal UI development
```

### Native Apps

```bash
pnpm ios:open            # Generate + open iOS Xcode project
pnpm mac:open            # Open built macOS app
pnpm mac:package         # Package macOS app
pnpm android:run         # Build + install + launch Android app
```

## Testing Guidelines

- Framework: Vitest with V8 coverage thresholds (70% lines/functions/statements, 55% branches).
- Naming: match source names with `*.test.ts`; e2e in `*.e2e.test.ts`.
- Run `pnpm test` (or `pnpm test:coverage`) before pushing when you touch logic.
- Do not set test workers above 16; tried already.
- Live tests (real keys): `CLAWDBOT_LIVE_TEST=1 pnpm test:live` (OpenClaw-only) or `LIVE=1 pnpm test:live` (includes provider live tests). Docker: `pnpm test:docker:live-models`, `pnpm test:docker:live-gateway`. Onboarding Docker E2E: `pnpm test:docker:onboard`.
- Full kit + what's covered: `docs/testing.md`.
- Pure test additions/fixes generally do **not** need a changelog entry unless they alter user-facing behavior or the user asks for one.
- Mobile: before using a simulator, check for connected real devices (iOS + Android) and prefer them when available.

## CI/CD Pipelines

GitHub Actions workflows in `.github/workflows/`:

| Workflow | Purpose |
|---|---|
| `ci.yml` | Main CI: lint, format, type-check, test, build (Ubuntu, macOS, Windows) |
| `docker-release.yml` | Multi-arch Docker image publish (Docker Hub + GHCR) |
| `install-smoke.yml` | Install script smoke test |
| `formal-conformance.yml` | Protocol schema conformance |
| `auto-response.yml` | Auto-respond to issues |
| `labeler.yml` | Auto-label PRs by changed paths |
| `workflow-sanity.yml` | Workflow validation |

**Deployment targets:** Fly.io (`fly.toml`), Docker (`Dockerfile`, `docker-compose.yml`).

## Coding Style & Naming Conventions

- Language: TypeScript (ESM). Prefer strict typing; avoid `any` (Oxlint enforces `no-explicit-any`).
- Formatting/linting via Oxlint and Oxfmt; run `pnpm check` before commits.
- Add brief code comments for tricky or non-obvious logic.
- Keep files concise; extract helpers instead of "V2" copies. Use existing patterns for CLI options and dependency injection via `createDefaultDeps`.
- Aim to keep files under ~500 LOC; guideline only (not a hard guardrail). Split/refactor when it improves clarity or testability.
- Naming: use **OpenClaw** for product/app/docs headings; use `openclaw` for CLI command, package/binary, paths, and config keys.
- CLI progress: use `src/cli/progress.ts` (`osc-progress` + `@clack/prompts` spinner); don't hand-roll spinners/bars.
- Status output: keep tables + ANSI-safe wrapping (`src/terminal/table.ts`); `status --all` = read-only/pasteable, `status --deep` = probes.
- Lobster seam: use the shared CLI palette in `src/terminal/palette.ts` (no hardcoded colors); apply palette to onboarding/config prompts and other TTY UI output as needed.
- Control UI: uses Lit with **legacy** decorators (`experimentalDecorators: true`, `useDefineForClassFields: false`). Use `@state()` and `@property()`, not `accessor` fields.
- Tool schema guardrails (google-antigravity): avoid `Type.Union` in tool input schemas; no `anyOf`/`oneOf`/`allOf`. Use `stringEnum`/`optionalStringEnum` (Type.Unsafe enum) for string lists, and `Type.Optional(...)` instead of `... | null`. Keep top-level tool schema as `type: "object"` with `properties`.
- Tool schema guardrails: avoid raw `format` property names in tool schemas; some validators treat `format` as a reserved keyword and reject the schema.

## Commit & Pull Request Guidelines

**Full maintainer PR workflow:** `.agents/skills/PR_WORKFLOW.md` -- triage order, quality bar, rebase rules, commit/changelog conventions, co-contributor policy, and the 3-step skill pipeline (`review-pr` > `prepare-pr` > `merge-pr`).

- Create commits with `scripts/committer "<msg>" <file...>`; avoid manual `git add`/`git commit` so staging stays scoped.
- Follow concise, action-oriented commit messages (e.g., `CLI: add verbose flag to send`).
- Group related changes; avoid bundling unrelated refactors.
- Read this when submitting a PR: `docs/help/submitting-a-pr.md` ([Submitting a PR](https://docs.openclaw.ai/help/submitting-a-pr))
- Read this when submitting an issue: `docs/help/submitting-an-issue.md` ([Submitting an Issue](https://docs.openclaw.ai/help/submitting-an-issue))
- PR review flow: when given a PR link, review via `gh pr view`/`gh pr diff` and do **not** change branches.
- PR review calls: prefer a single `gh pr view --json ...` to batch metadata/comments; run `gh pr diff` only when needed.
- Before starting a review when a GH Issue/PR is pasted: run `git pull`; if there are local changes or unpushed commits, stop and alert the user before reviewing.
- Goal: merge PRs. Prefer **rebase** when commits are clean; **squash** when history is messy.
- PR merge flow: create a temp branch from `main`, merge the PR branch into it (prefer squash unless commit history is important; use rebase/merge when it is). Always try to merge the PR unless it's truly difficult, then use another approach. If we squash, add the PR author as a co-contributor. Apply fixes, add changelog entry (include PR # + thanks), run full gate before the final commit, commit, merge back to `main`, delete the temp branch, and end on `main`.
- If you review a PR and later do work on it, land via merge/squash (no direct-main commits) and always add the PR author as a co-contributor.
- When working on a PR: add a changelog entry with the PR number and thank the contributor.
- When working on an issue: reference the issue in the changelog entry.
- When merging a PR: leave a PR comment that explains exactly what we did and include the SHA hashes.
- When merging a PR from a new contributor: add their avatar to the README "Thanks to all clawtributors" thumbnail list.
- After merging a PR: run `bun scripts/update-clawtributors.ts` if the contributor is missing, then commit the regenerated README.

## Release Channels (Naming)

- stable: tagged releases only (e.g. `vYYYY.M.D`), npm dist-tag `latest`.
- beta: prerelease tags `vYYYY.M.D-beta.N`, npm dist-tag `beta` (may ship without macOS app).
- dev: moving head on `main` (no tag; git checkout main).

## Shorthand Commands

- `sync`: if working tree is dirty, commit all changes (pick a sensible Conventional Commit message), then `git pull --rebase`; if rebase conflicts and cannot resolve, stop; otherwise `git push`.

### PR Workflow (Review vs Land)

- **Review mode (PR link only):** read `gh pr view/diff`; **do not** switch branches; **do not** change code.
- **Landing mode:** create an integration branch from `main`, bring in PR commits (**prefer rebase** for linear history; **merge allowed** when complexity/conflicts make it safer), apply fixes, add changelog (+ thanks + PR #), run full gate **locally before committing** (`pnpm build && pnpm check && pnpm test`), commit, merge back to `main`, then `git switch main` (never stay on a topic branch after landing). Important: contributor needs to be in git graph after this!

## Docs Linking (Mintlify)
## Security & Configuration Tips

- Docs are hosted on Mintlify (docs.openclaw.ai).
- Internal doc links in `docs/**/*.md`: root-relative, no `.md`/`.mdx` (example: `[Config](/configuration)`).
- Section cross-references: use anchors on root-relative paths (example: `[Hooks](/configuration#hooks)`).
- Doc headings and anchors: avoid em dashes and apostrophes in headings because they break Mintlify anchor links.
- When Peter asks for links, reply with full `https://docs.openclaw.ai/...` URLs (not root-relative).
- When you touch docs, end the reply with the `https://docs.openclaw.ai/...` URLs you referenced.
- README (GitHub): keep absolute docs URLs (`https://docs.openclaw.ai/...`) so links work on GitHub.
- Docs content must be generic: no personal device names/hostnames/paths; use placeholders like `user@gateway-host` and "gateway host".

## Docs i18n (zh-CN)

- `docs/zh-CN/**` is generated; do not edit unless the user explicitly asks.
- Pipeline: update English docs -> adjust glossary (`docs/.i18n/glossary.zh-CN.json`) -> run `scripts/docs-i18n` -> apply targeted fixes only if instructed.
- Translation memory: `docs/.i18n/zh-CN.tm.jsonl` (generated).
- See `docs/.i18n/README.md`.
- The pipeline can be slow/inefficient; if it's dragging, ping @jospalmbier on Discord instead of hacking around it.

## Docker & Deployment

### Docker

- **Dockerfile:** Multi-stage build (Node 22-bookworm, Bun, pnpm, non-root `node` user).
- **docker-compose.yml:** Two services: `openclaw-gateway` (port 18789) and `openclaw-cli`.
- **Sandbox Dockerfiles:** `Dockerfile.sandbox` (lightweight) and `Dockerfile.sandbox-browser` (with browser support).
- **Setup script:** `docker-setup.sh` (Docker availability check, token generation, volume setup).

### Fly.io

- Config: `fly.toml` (app = "openclaw", region iad, 2x shared CPU, 2GB RAM, `NODE_ENV=production`).
- Update command: `fly ssh console -a flawd-bot -C "bash -lc 'cd /data/clawd/openclaw && git pull --rebase origin main'"` then `fly machines restart e825232f34d058 -a flawd-bot`.

## Environment & Configuration

- Config/workspace stored in `~/.openclaw/` and `~/.openclaw/workspace/`.
- Pi sessions live under `~/.openclaw/sessions/` by default; the base directory is not configurable.
- Web provider stores creds at `~/.openclaw/credentials/`; rerun `openclaw login` if logged out.
- Environment variables: see `~/.profile`.
- Key env vars: `OPENCLAW_GATEWAY_TOKEN`, `OPENCLAW_CONFIG_DIR`, `OPENCLAW_WORKSPACE_DIR`, `OPENCLAW_GATEWAY_PORT`, `OPENCLAW_BRIDGE_PORT`, `OPENCLAW_GATEWAY_BIND`.

## Security & Configuration Tips

- Never commit or publish real phone numbers, videos, or live configuration values. Use obviously fake placeholders in docs, tests, and examples.
- Release flow: always read `docs/reference/RELEASING.md` and `docs/platforms/mac/release.md` before any release work; do not ask routine questions once those docs answer them.
- Release guardrails: do not change version numbers without operator's explicit consent; always ask permission before running any npm publish/release step.
- Release signing/notary keys are managed outside the repo; follow internal release docs.
- Notary auth env vars (`APP_STORE_CONNECT_ISSUER_ID`, `APP_STORE_CONNECT_KEY_ID`, `APP_STORE_CONNECT_API_KEY_P8`) are expected in your environment (per internal release docs).

## Version Locations

When bumping versions, update all of these:

- `package.json` (CLI)
- `apps/android/app/build.gradle.kts` (versionName/versionCode)
- `apps/ios/Sources/Info.plist` + `apps/ios/Tests/Info.plist` (CFBundleShortVersionString/CFBundleVersion)
- `apps/macos/Sources/OpenClaw/Resources/Info.plist` (CFBundleShortVersionString/CFBundleVersion)
- `docs/install/updating.md` (pinned npm version)
- `docs/platforms/mac/release.md` (APP_VERSION/APP_BUILD examples)
- Peekaboo Xcode projects/Info.plists (MARKETING_VERSION/CURRENT_PROJECT_VERSION)

## Troubleshooting

- Rebrand/migration issues or legacy config/service warnings: run `openclaw doctor` (see `docs/gateway/doctor.md`).

## exe.dev VM ops (general)

- Access: stable path is `ssh exe.dev` then `ssh vm-name` (assume SSH key already set).
- SSH flaky: use exe.dev web terminal or Shelley (web agent); keep a tmux session for long ops.
- Update: `sudo npm i -g openclaw@latest` (global install needs root on `/usr/lib/node_modules`).
- Config: use `openclaw config set ...`; ensure `gateway.mode=local` is set.
- Discord: store raw token only (no `DISCORD_BOT_TOKEN=` prefix).
- Restart: stop old gateway and run:
  `pkill -9 -f openclaw-gateway || true; nohup openclaw gateway run --bind loopback --port 18789 --force > /tmp/openclaw-gateway.log 2>&1 &`
- Verify: `openclaw channels status --probe`, `ss -ltnp | rg 18789`, `tail -n 120 /tmp/openclaw-gateway.log`.

## Agent-Specific Notes

- Vocabulary: "makeup" = "mac app".
- Never edit `node_modules` (global/Homebrew/npm/git installs too). Updates overwrite. Skill notes go in `tools.md` or `AGENTS.md`.
- When adding a new `AGENTS.md` anywhere in the repo, also add a `CLAUDE.md` symlink pointing to it (example: `ln -s AGENTS.md CLAUDE.md`).
- When working on a GitHub Issue or PR, print the full URL at the end of the task.
- When answering questions, respond with high-confidence answers only: verify in code; do not guess.
- Never update the Carbon dependency.
- Any dependency with `pnpm.patchedDependencies` must use an exact version (no `^`/`~`).
- Patching dependencies (pnpm patches, overrides, or vendored changes) requires explicit approval; do not do this by default.
- Gateway currently runs only as the menubar app; there is no separate LaunchAgent/helper label installed. Restart via the OpenClaw Mac app or `scripts/restart-mac.sh`; to verify/kill use `launchctl print gui/$UID | grep openclaw` rather than assuming a fixed label. **When debugging on macOS, start/stop the gateway via the app, not ad-hoc tmux sessions; kill any temporary tunnels before handoff.**
- macOS logs: use `./scripts/clawlog.sh` to query unified logs for the OpenClaw subsystem; it supports follow/tail/category filters and expects passwordless sudo for `/usr/bin/log`.
- If shared guardrails are available locally, review them; otherwise follow this repo's guidance.
- SwiftUI state management (iOS/macOS): prefer the `Observation` framework (`@Observable`, `@Bindable`) over `ObservableObject`/`@StateObject`; don't introduce new `ObservableObject` unless required for compatibility, and migrate existing usages when touching related code.
- Connection providers: when adding a new connection, update every UI surface and docs (macOS app, web UI, mobile if applicable, onboarding/overview docs) and add matching status + configuration forms so provider lists and settings stay in sync.
- **Restart apps:** "restart iOS/Android apps" means rebuild (recompile/install) and relaunch, not just kill/launch.
- **Device checks:** before testing, verify connected real devices (iOS/Android) before reaching for simulators/emulators.
- iOS Team ID lookup: `security find-identity -p codesigning -v` -> use Apple Development (...) TEAMID. Fallback: `defaults read com.apple.dt.Xcode IDEProvisioningTeamIdentifiers`.
- A2UI bundle hash: `src/canvas-host/a2ui/.bundle.hash` is auto-generated; ignore unexpected changes, and only regenerate via `pnpm canvas:a2ui:bundle` (or `scripts/bundle-a2ui.sh`) when needed. Commit the hash as a separate commit.
- Do not rebuild the macOS app over SSH; rebuilds must be run directly on the Mac.
- Never send streaming/partial replies to external messaging surfaces (WhatsApp, Telegram); only final replies should be delivered there. Streaming/tool events may still go to internal UIs/control channel.
- Voice wake forwarding tips:
  - Command template should stay `openclaw-mac agent --message "${text}" --thinking low`; `VoiceWakeForwarder` already shell-escapes `${text}`. Don't add extra quotes.
  - launchd PATH is minimal; ensure the app's launch agent PATH includes standard system paths plus your pnpm bin (typically `$HOME/Library/pnpm`) so `pnpm`/`openclaw` binaries resolve when invoked via `openclaw-mac`.
- For manual `openclaw message send` messages that include `!`, use the heredoc pattern noted below to avoid the Bash tool's escaping.
- When asked to open a "session" file, open the Pi session logs under `~/.openclaw/agents/<agentId>/sessions/*.jsonl` (use the `agent=<id>` value in the Runtime line of the system prompt; newest unless a specific ID is given), not the default `sessions.json`. If logs are needed from another machine, SSH via Tailscale and read the same path there.
- Bug investigations: read source code of relevant npm dependencies and all related local code before concluding; aim for high-confidence root cause.

## Multi-Agent Safety

- Do **not** create/apply/drop `git stash` entries unless explicitly requested (this includes `git pull --rebase --autostash`). Assume other agents may be working; keep unrelated WIP untouched and avoid cross-cutting state changes.
- When the user says "push", you may `git pull --rebase` to integrate latest changes (never discard other agents' work). When the user says "commit", scope to your changes only. When the user says "commit all", commit everything in grouped chunks.
- Do **not** create/remove/modify `git worktree` checkouts (or edit `.worktrees/*`) unless explicitly requested.
- Do **not** switch branches / check out a different branch unless explicitly requested.
- Running multiple agents is OK as long as each agent has its own session.
- When you see unrecognized files, keep going; focus on your changes and commit only those.
- Focus reports on your edits; avoid guard-rail disclaimers unless truly blocked; when multiple agents touch the same file, continue if safe; end with a brief "other files present" note only if relevant.

## Lint/Format Churn

- If staged+unstaged diffs are formatting-only, auto-resolve without asking.
- If commit/push already requested, auto-stage and include formatting-only follow-ups in the same commit (or a tiny follow-up commit if needed), no extra confirmation.
- Only ask when changes are semantic (logic/data/behavior).

## NPM + 1Password (publish/verify)

- Use the 1password skill; all `op` commands must run inside a fresh tmux session.
- Sign in: `eval "$(op signin --account my.1password.com)"` (app unlocked + integration on).
- OTP: `op read 'op://Private/Npmjs/one-time password?attribute=otp'`.
- Publish: `npm publish --access public --otp="<otp>"` (run from the package dir).
- Verify without local npmrc side effects: `npm view <pkg> version --userconfig "$(mktemp)"`.
- Kill the tmux session after publish.
