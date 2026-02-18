# Power User Mentor's Guide
## Master OpenClaw: From Competent to Legendary

---

### Chapter 1: The Mental Model

#### How OpenClaw Actually Works Under the Hood

OpenClaw is a multi-channel AI gateway with extensible messaging integrations. At its core, it connects messaging platforms (Telegram, Discord, Slack, Signal, iMessage, WhatsApp, MS Teams, Matrix, and more) to AI model providers (Anthropic, OpenAI, Ollama, vLLM, Bedrock, and others) through a unified gateway architecture.

The project is built as a TypeScript ESM monorepo managed by pnpm (v10.23.0), requiring Node.js 22+, with Bun supported as an alternative runtime for development and testing. The binary entry point is `openclaw.mjs` at the repository root, which bootstraps into the Commander-based CLI defined in `/home/user/openclaw/src/cli/program/build-program.ts`.

#### The Message Lifecycle: From Channel Input to AI Response

1. **Ingestion**: A message arrives on a channel (e.g., Telegram via grammy, Discord via Carbon, Slack via Bolt). Each channel adapter normalizes it into a unified internal format.
2. **Routing**: The routing engine (`/home/user/openclaw/src/routing/resolve-route.ts`) resolves which agent should handle the message. It evaluates a tiered binding system -- peer match, parent peer inheritance, guild+roles, guild-only, team, account, and channel-wide -- before falling back to the default agent.
3. **Session Resolution**: A session key is built from the resolved agent, channel, account, and peer information. The `dmScope` configuration (main, per-peer, per-channel-peer, per-account-channel-peer) controls session isolation.
4. **Agent Processing**: The resolved agent processes the message through its configured AI provider, tools, skills, and memory context.
5. **Response Delivery**: The response is routed back through the originating channel. Only final replies are delivered to external surfaces; streaming/partial replies are restricted to internal UIs.

#### Architecture Overview

```
                    CLI (Commander)
                         |
              +----------+-----------+
              |                      |
         Core Commands          Sub-CLI Commands
     (setup, onboard,        (gateway, daemon, models,
      configure, config,      plugins, channels, nodes,
      doctor, agent,          sandbox, cron, skills,
      message, memory,        security, tui, browser,
      status, health,         acp, dns, webhooks,
      sessions, browser)      pairing, devices, logs,
                              update, completion, docs,
                              hooks, directory, system)
              |                      |
              +----------+-----------+
                         |
                    Gateway Server
                    /    |    \
               Channels  |  Agents
              /  |  \    |   / | \
           TG  DC  SL  Route  Provider(s)
           WA  SG  iM    |    Memory
           MT  MX  ZL    |    Skills
           IR  TW  NS    |    Hooks
           LN  GC  NC    |    Sandbox
           FB  VC  TL    |
                         |
                    Extensions (plugins)
```

**Key source directories:**
- `/home/user/openclaw/src/cli/` -- CLI wiring and command registration
- `/home/user/openclaw/src/commands/` -- Command implementations (onboard, doctor, status, agents, etc.)
- `/home/user/openclaw/src/gateway/` -- Gateway server, auth, chat handling, config reload
- `/home/user/openclaw/src/routing/` -- Message routing (bindings, session keys, agent resolution)
- `/home/user/openclaw/src/agents/` -- Agent runtime, auth profiles, bash tools, sandboxing
- `/home/user/openclaw/src/plugins/` -- Plugin loader, registry, discovery, hooks, manifest system
- `/home/user/openclaw/src/memory/` -- Embedding backends (OpenAI, Gemini, Voyage), vector search
- `/home/user/openclaw/src/config/` -- Config loading, agent directories, channel capabilities
- `/home/user/openclaw/extensions/` -- 36 extension packages (channels + utilities)

#### Key Design Patterns

- **Lazy Command Registration**: Commands are registered as placeholders and only fully loaded when invoked (`/home/user/openclaw/src/cli/program/command-registry.ts`). This keeps CLI startup fast.
- **Dependency Injection**: Commands use `createDefaultDeps` for testability.
- **Plugin SDK with jiti Alias**: Extensions import `openclaw/plugin-sdk` which resolves at runtime via jiti aliasing, decoupling plugins from core internals.
- **WeakMap-based Caching**: The route resolver caches evaluated bindings per config object using WeakMap, preventing memory leaks while maintaining performance.

---

### Chapter 2: CLI Mastery

#### Every CLI Command and What It Does

OpenClaw's CLI is organized into **core commands** (always available) and **sub-CLI commands** (lazy-loaded). The full registry is defined in `/home/user/openclaw/src/cli/program/command-registry.ts` and `/home/user/openclaw/src/cli/program/register.subclis.ts`.

**Core Commands:**
| Command | Purpose |
|---------|---------|
| `setup` | Initial setup helpers |
| `onboard` | Interactive onboarding wizard (auth, channels, skills, hooks) |
| `configure` | Configuration wizard |
| `config` | Direct config get/set/list operations |
| `doctor` | Health checks and quick fixes for gateway and channels |
| `dashboard` | Open the Control UI with current token |
| `reset` | Reset local config/state (keeps CLI installed) |
| `uninstall` | Uninstall gateway service and local data |
| `message` | Send, read, and manage messages |
| `memory` | Memory search, store, and management |
| `agent` / `agents` | Agent runtime and isolated agent management |
| `status` | Gateway status (use `--all` for read-only, `--deep` for probes) |
| `health` | Gateway health endpoint |
| `sessions` | Session management |
| `browser` | Browser tools (inspect, debug, actions, manage, resize, state) |

**Sub-CLI Commands:**
| Command | Purpose |
|---------|---------|
| `gateway` | Gateway control (run, stop, status) |
| `daemon` | Gateway service (legacy alias for gateway) |
| `logs` | Gateway log access |
| `system` | System events, heartbeat, presence |
| `models` | Model configuration and listing |
| `approvals` | Exec approval management |
| `nodes` | Node commands (canvas, camera, screen, run) |
| `node` | Node control |
| `devices` | Device pairing and token management |
| `sandbox` | Sandbox tools and explanation |
| `tui` | Terminal UI |
| `cron` | Cron scheduler |
| `dns` | DNS helpers |
| `docs` | Documentation helpers |
| `hooks` | Hook tooling |
| `webhooks` | Webhook helpers |
| `pairing` | Channel pairing helpers |
| `plugins` | Plugin management (install, uninstall, list) |
| `channels` | Channel management and status |
| `directory` | Directory commands |
| `security` | Security helpers |
| `skills` | Skills management |
| `update` | CLI update helpers |
| `completion` | Shell completion script generation |
| `acp` | Agent Control Protocol tools |

#### Power Flags and Hidden Options

- `OPENCLAW_DISABLE_LAZY_SUBCOMMANDS=1` -- Force eager registration of all subcommands (useful for debugging or shell completion generation).
- `OPENCLAW_SKIP_CHANNELS=1` -- Skip channel initialization during gateway dev mode.
- `OPENCLAW_PROFILE=dev` -- Use the dev profile for TUI sessions.
- `CLAWDBOT_LIVE_TEST=1` or `LIVE=1` -- Enable live tests against real provider keys.

#### Dev Mode Tricks

```bash
# Run CLI in dev mode (uses Bun under the hood)
pnpm openclaw <command>

# Gateway dev mode (skips channel startup)
pnpm gateway:dev

# Gateway dev with state reset
pnpm gateway:dev:reset

# Gateway with file watching
pnpm gateway:watch

# TUI dev mode
pnpm tui:dev

# Run as RPC agent
pnpm openclaw:rpc
```

---

### Chapter 3: Channel Configuration Deep-Dive

#### Supported Channels

OpenClaw supports an impressive array of messaging channels. Core channels live in `src/` and extension channels live in `extensions/`.

**Core channels** (in `/home/user/openclaw/src/`): Telegram, Discord, Slack, Signal, iMessage, WhatsApp (web)

**Extension channels** (in `/home/user/openclaw/extensions/`): MS Teams, Matrix, Zalo, ZaloUser, IRC, Line, Feishu, Google Chat, Mattermost, Nextcloud Talk, Nostr, Tlon, Twitch, Voice Call, BlueBubbles, Discord (extension variant)

Full documentation for each channel is maintained at `/home/user/openclaw/docs/channels/`. Every channel has a dedicated markdown file.

#### Channel-Specific Configuration

Extension channels declare their identity in `package.json` under the `openclaw` key. For example, MS Teams (`/home/user/openclaw/extensions/msteams/package.json`):

```json
{
  "openclaw": {
    "extensions": ["./index.ts"],
    "channel": {
      "id": "msteams",
      "label": "Microsoft Teams",
      "selectionLabel": "Microsoft Teams (Bot Framework)",
      "docsPath": "/channels/msteams",
      "aliases": ["teams"],
      "order": 60
    },
    "install": {
      "npmSpec": "@openclaw/msteams",
      "localPath": "extensions/msteams",
      "defaultChoice": "npm"
    }
  }
}
```

#### Multi-Channel Routing Strategies

The routing engine in `/home/user/openclaw/src/routing/resolve-route.ts` supports a sophisticated tiered binding system for directing messages to specific agents:

1. **Peer binding** -- Route specific users to specific agents
2. **Parent peer binding** -- Thread inheritance from parent conversations
3. **Guild + Roles** -- Discord server + role-based routing
4. **Guild** -- Discord server-wide routing
5. **Team** -- Slack/Teams workspace routing
6. **Account** -- Per-account routing (wildcard `*` supported)
7. **Channel** -- Channel-wide default routing
8. **Default** -- Fallback to the default agent

Configure `session.dmScope` for session isolation granularity: `main` (all DMs share one session), `per-peer` (per-user sessions), `per-channel-peer` (per-user-per-channel), or `per-account-channel-peer` (maximum isolation).

---

### Chapter 4: Extension Development

#### Creating Your First Extension

Every extension is a standalone npm package under `/home/user/openclaw/extensions/`. The minimum structure:

```
extensions/my-channel/
  index.ts              # Entry point
  package.json          # With openclaw metadata
  openclaw.plugin.json  # Plugin manifest
  src/                  # Implementation
  CHANGELOG.md          # Required for releases
```

#### Extension package.json Requirements

Critical rules from `/home/user/openclaw/AGENTS.md`:

- Plugin-only dependencies go in the extension's `package.json`, not the root
- Put `openclaw` in `devDependencies` or `peerDependencies` (never `dependencies` with `workspace:*`)
- Runtime deps must be in `dependencies` (npm install runs `--omit=dev`)
- Import the plugin SDK via `openclaw/plugin-sdk` -- resolved at runtime through jiti aliasing

```json
{
  "name": "@openclaw/my-channel",
  "version": "2026.2.16",
  "type": "module",
  "dependencies": { /* runtime deps only */ },
  "devDependencies": {
    "openclaw": "workspace:*"
  },
  "openclaw": {
    "extensions": ["./index.ts"],
    "channel": {
      "id": "mychannel",
      "label": "My Channel",
      "docsPath": "/channels/mychannel",
      "order": 100
    }
  }
}
```

#### Plugin SDK and jiti Alias Resolution

The Vitest config (`/home/user/openclaw/vitest.config.ts`) reveals how the SDK alias works:

```typescript
alias: [
  { find: "openclaw/plugin-sdk/account-id", replacement: "src/plugin-sdk/account-id.ts" },
  { find: "openclaw/plugin-sdk", replacement: "src/plugin-sdk/index.ts" },
]
```

At build time, the SDK is compiled to `dist/plugin-sdk/` with full TypeScript declarations (`/home/user/openclaw/tsdown.config.ts`).

#### The Plugin Manifest

Each extension also has an `openclaw.plugin.json` (e.g., `/home/user/openclaw/extensions/msteams/openclaw.plugin.json`):

```json
{
  "id": "msteams",
  "channels": ["msteams"],
  "configSchema": { "type": "object", "properties": {} }
}
```

The plugin loader (`/home/user/openclaw/src/plugins/loader.ts`) discovers, validates, and registers extensions. The manifest registry (`/home/user/openclaw/src/plugins/manifest-registry.ts`) tracks installed plugins, and the hook runner (`/home/user/openclaw/src/plugins/hooks.ts`) wires plugin lifecycle hooks.

---

### Chapter 5: Claude Flow v3 Orchestration

#### Overview

Claude Flow v3 is configured in `/home/user/openclaw/.claude/settings.json` under the `claudeFlow` key. It provides multi-agent orchestration, memory systems, and automated workflows.

```json
{
  "claudeFlow": {
    "version": "3.0.0",
    "enabled": true,
    "swarm": { "topology": "hierarchical-mesh", "maxAgents": 15 },
    "memory": { "backend": "hybrid", "enableHNSW": true },
    "agentTeams": { "enabled": true, "teammateMode": "auto" }
  }
}
```

#### Swarm Topologies

Three topologies are available, defined in `/home/user/openclaw/.claude/agents/swarm/`:

1. **Hierarchical** (`/home/user/openclaw/.claude/agents/swarm/hierarchical-coordinator.md`) -- A "Queen" agent delegates to specialized workers (research, code, analyst, test). Best for well-defined tasks with clear decomposition.

2. **Mesh** (`/home/user/openclaw/.claude/agents/swarm/mesh-coordinator.md`) -- Peer-to-peer with broadcast communication. Best for collaborative tasks where any agent might contribute.

3. **Adaptive** (`/home/user/openclaw/.claude/agents/swarm/adaptive-coordinator.md`) -- Dynamically switches topology based on task complexity. Starts mesh, escalates to hierarchical when needed.

The default configured topology is `hierarchical-mesh` with a maximum of 15 agents.

#### Agent Templates

Nine agent templates are available at `/home/user/openclaw/.claude/agents/templates/`:

| Template | Purpose |
|----------|---------|
| `orchestrator-task.md` | Central coordination for task decomposition and execution planning |
| `coordinator-swarm-init.md` | Swarm initialization and topology setup |
| `implementer-sparc-coder.md` | SPARC-methodology coding agent |
| `github-pr-manager.md` | GitHub PR lifecycle management |
| `memory-coordinator.md` | Memory system coordination |
| `performance-analyzer.md` | Performance analysis and optimization |
| `automation-smart-agent.md` | Automated workflow agent |
| `sparc-coordinator.md` | SPARC development methodology coordinator |
| `migration-plan.md` | Migration planning agent |

#### Memory Systems

The memory backend is configured as `hybrid` with HNSW enabled:

- **HNSW** (Hierarchical Navigable Small World) -- Vector-based similarity search with sub-millisecond performance via `sqlite-vec` 
- **Learning Bridge** -- Connects short-term session memory to long-term storage
- **Memory Graph** -- Tracks relationships between memory entries
- **Agent Scopes** -- Isolates memory per agent

Embedding backends (in `/home/user/openclaw/src/memory/`) support OpenAI, Gemini, and Voyage with configurable chunk limits and model-specific input limits.

#### Skills (29 Available)

All 29 skills live at `/home/user/openclaw/.claude/skills/`, each containing a `SKILL.md`:

**AgentDB Skills** (5): `agentdb-advanced`, `agentdb-learning`, `agentdb-memory-patterns`, `agentdb-optimization`, `agentdb-vector-search`

**GitHub Skills** (5): `github-code-review`, `github-multi-repo`, `github-project-management`, `github-release-management`, `github-workflow-automation`

**v3 Implementation Skills** (8): `v3-cli-modernization`, `v3-core-implementation`, `v3-ddd-architecture`, `v3-integration-deep`, `v3-mcp-optimization`, `v3-memory-unification`, `v3-performance-optimization`, `v3-security-overhaul`, `v3-swarm-coordination`

**Other Skills** (11): `hooks-automation`, `pair-programming`, `skill-builder`, `sparc-methodology`, `stream-chain`, `swarm-advanced`, `swarm-orchestration`, `reasoningbank-agentdb`, `reasoningbank-intelligence`, `verification-quality`

#### Slash Commands

Slash commands at `/home/user/openclaw/.claude/commands/` are organized into categories:

- **analysis/** (7 commands) -- Compliance reports, bottleneck detection, performance reports, token efficiency
- **automation/** (6 commands) -- Auto-agent, self-healing, session memory, smart agents, workflow selection
- **github/** (19 commands) -- Code review, PR management, release management, multi-repo coordination, issue tracking
- **hooks/** (7 commands) -- Hook lifecycle management (pre-edit, post-edit, pre-task, post-task, session-end, setup)
- **monitoring/** (5 commands) -- Agent metrics, real-time views, swarm monitoring
- **optimization/** (5 commands) -- Topology optimization, cache management, parallel execution
- **sparc/** (34 commands) -- Full SPARC methodology suite (architect, coder, tester, debugger, reviewer, etc.)
- **Top-level** (3 commands) -- `claude-flow-help.md`, `claude-flow-memory.md`, `claude-flow-swarm.md`

#### Hook Automation

The hook system is configured in `/home/user/openclaw/.claude/settings.json` and dispatched by `/home/user/openclaw/.claude/helpers/hook-handler.cjs`:

| Hook Event | Action |
|------------|--------|
| `PreToolUse` (Bash) | Validates commands against dangerous patterns (e.g., `rm -rf /`) |
| `PostToolUse` (Write/Edit) | Records edit metrics and intelligence feedback |
| `UserPromptSubmit` | Routes tasks via intelligence module, logs routing confidence |
| `SessionStart` | Restores session state, loads intelligence graph (nodes + edges) |
| `SessionEnd` | Consolidates intelligence (entries, edges, PageRank), ends session |
| `Stop` | Syncs auto-memory |
| `PreCompact` (manual/auto) | Injects agent context guidance before compaction |
| `SubagentStart` | Status check |
| `TeammateIdle` | Triggers post-task intelligence feedback and auto-assignment |
| `TaskCompleted` | Trains patterns and notifies lead agent |

---

### Chapter 6: Advanced Configuration

#### .claude/settings.json Deep-Dive

The settings file (`/home/user/openclaw/.claude/settings.json`) controls:

**Permissions:**
```json
{
  "permissions": {
    "allow": ["Bash(npx @claude-flow*)", "Bash(node .claude/*)", "mcp__claude-flow__:*"],
    "deny": ["Read(./.env)", "Read(./.env.*)"]
  }
}
```

**Environment Variables:**
```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1",
    "CLAUDE_FLOW_V3_ENABLED": "true",
    "CLAUDE_FLOW_HOOKS_ENABLED": "true"
  }
}
```

**Daemon Workers** (10 background workers with scheduled intervals):
- `map`, `audit` (1h, critical), `optimize` (30m, high), `consolidate` (2h, low), `testgaps`, `ultralearn` (1h), `deepdive` (4h, triggered by complex changes), `document` (1h, triggered by ADR/API changes), `refactor`, `benchmark`

**Learning System:**
- Auto-training enabled with coordination, optimization, and prediction patterns
- Short-term retention: 24h; Long-term retention: 30d

**Model Preferences:**
- Default: `claude-opus-4-6`
- Routing: `claude-haiku-4-5-20251001`

#### AGENTS.md Customization

`/home/user/openclaw/AGENTS.md` is the primary repository guidelines file. A `CLAUDE.md` symlink should always accompany it. Key conventions:

- Product name: **OpenClaw** (headings/docs), `openclaw` (CLI/code/config)
- Files should stay under ~500 LOC
- Use `scripts/committer "<msg>" <files...>` for commits (not manual git add/commit)
- Commit messages: concise, action-oriented (e.g., `CLI: add verbose flag to send`)

#### Build System Tuning

**Build** (`/home/user/openclaw/tsdown.config.ts`): Uses tsdown (Rolldown-based) with 8 entry points compiled for Node platform. Outputs to `dist/` with separate `dist/plugin-sdk/` for the SDK.

**Type checking**: `pnpm tsgo` (uses `@typescript/native-preview` for fast checking), `pnpm build` for full build

**Lint/Format**: Oxlint (type-aware) + Oxfmt. Run `pnpm check` for the full suite: format check, type check, lint.

**Pre-commit Hook** (`/home/user/openclaw/git-hooks/pre-commit`): Filters staged files, runs oxlint with `--fix` on lint-eligible files, runs oxfmt with `--write` on format-eligible files, then re-stages.

---

### Chapter 7: Testing Like a Pro

#### Vitest Configuration

The test configuration (`/home/user/openclaw/vitest.config.ts`) uses:

- **Pool**: `forks` (not threads -- avoids env pollution)
- **Workers**: CI uses 2-3; local uses `Math.max(4, Math.min(16, cpus))` (cap at 16 -- more was already tried)
- **Timeout**: 120s per test, 180s hooks on Windows
- **Environment isolation**: `unstubEnvs: true`, `unstubGlobals: true` to prevent cross-test pollution
- **Setup**: `test/setup.ts`

#### Coverage Thresholds

```
Lines:      70%
Functions:  70%
Branches:   55%
Statements: 70%
```

Coverage only counts `src/**/*.ts` files actually exercised (not `all: true`), excluding extensions, apps, CLI wiring, gateway server surfaces, channel integrations, and interactive UIs.

#### Test Tiers

| Tier | Command | Description |
|------|---------|-------------|
| Unit | `pnpm test` | Parallel vitest run (custom parallel script) |
| Fast | `pnpm test:fast` | Unit config only, no extras |
| Coverage | `pnpm test:coverage` | Unit tests with V8 coverage |
| Watch | `pnpm test:watch` | Vitest in watch mode |
| E2E | `pnpm test:e2e` | End-to-end tests |
| Live | `pnpm test:live` | Real API keys (`OPENCLAW_LIVE_TEST=1`) |
| Docker | `pnpm test:docker:all` | Full Docker test suite (8 suites) |
| Install Smoke | `pnpm test:install:smoke` | Install-path smoke test |
| UI | `pnpm test:ui` | UI component tests |
| All | `pnpm test:all` | Lint + build + unit + e2e + live + docker |

#### Docker Test Suites

- `test:docker:live-models` -- Live model integration
- `test:docker:live-gateway` -- Live gateway tests
- `test:docker:onboard` -- Onboarding E2E
- `test:docker:gateway-network` -- Network isolation tests
- `test:docker:qr` -- QR code import tests
- `test:docker:doctor-switch` -- Doctor install/switch tests
- `test:docker:plugins` -- Plugin installation tests
- `test:docker:cleanup` -- Docker environment cleanup

---

### Chapter 8: Performance Optimization

#### Response Time Optimization

- The CLI uses **lazy command registration** -- only the invoked command's module is loaded, keeping startup under 100ms
- The routing engine uses **WeakMap-based binding caches** with a 2000-key eviction policy
- Gateway configuration supports hot reload (`/home/user/openclaw/src/gateway/config-reload.ts`)
- Chat abort support (`/home/user/openclaw/src/gateway/chat-abort.ts`) prevents wasted computation

#### Memory Management

- Embedding chunk limits are managed per-model (`/home/user/openclaw/src/memory/embedding-chunk-limits.ts`)
- Batch processing for embeddings supports OpenAI, Gemini, and Voyage backends
- Session compaction is documented at `/home/user/openclaw/docs/reference/session-management-compaction.md`

#### Scaling Strategies

- Multiple gateways: `/home/user/openclaw/docs/gateway/multiple-gateways.md`
- Daemon workers run on configurable intervals (30m to 4h) to prevent resource contention
- Agent concurrency defaults are configurable (`/home/user/openclaw/src/config/config.agent-concurrency-defaults.test.ts` documents the patterns)
- The gateway supports binding to loopback with configurable ports: `openclaw gateway run --bind loopback --port 18789 --force`

---

### Chapter 9: Security Hardening

#### Secret Management

- `.env` files are denied in Claude settings: `"deny": ["Read(./.env)", "Read(./.env.*)"]`
- Credentials stored at `~/.openclaw/credentials/`
- `.detect-secrets.cfg` (`/home/user/openclaw/.detect-secrets.cfg`) excludes lockfiles, dist/vendor directories, and known false positives (API key labels, schema references)
- Real phone numbers, videos, and live config values are never committed

#### Pre-Bash Hook Safety

The `pre-bash` hook in `/home/user/openclaw/.claude/helpers/hook-handler.cjs` blocks dangerous commands:
```javascript
var dangerous = ['rm -rf /', 'format c:', 'del /s /q c:\\', ':(){:|:&};:'];
```

#### Docker Sandbox Isolation

- Gateway sandboxing docs: `/home/user/openclaw/docs/gateway/sandboxing.md`
- Sandbox vs tool policy: `/home/user/openclaw/docs/gateway/sandbox-vs-tool-policy-vs-elevated.md`
- Agent sandboxing: `/home/user/openclaw/src/agents/sandbox.ts`, `/home/user/openclaw/src/agents/sandbox-paths.ts`
- CLI sandbox tools: `openclaw sandbox explain`, `openclaw sandbox` subcommands

#### Security Documentation

- Threat Model Atlas: `/home/user/openclaw/docs/security/THREAT-MODEL-ATLAS.md`
- Contributing Threat Model: `/home/user/openclaw/docs/security/CONTRIBUTING-THREAT-MODEL.md`
- Formal Verification: `/home/user/openclaw/docs/security/formal-verification.md`
- Gateway security: `/home/user/openclaw/docs/gateway/security/`
- Gateway authentication: `/home/user/openclaw/docs/gateway/authentication.md`
- Trusted proxy auth: `/home/user/openclaw/docs/gateway/trusted-proxy-auth.md`

#### Network Security

- Gateway auth with rate limiting: `/home/user/openclaw/src/gateway/auth-rate-limit.ts`
- Device auth: `/home/user/openclaw/src/gateway/device-auth.ts`
- Tailscale integration: `/home/user/openclaw/docs/gateway/tailscale.md`
- Network model documentation: `/home/user/openclaw/docs/gateway/network-model.md`

---

### Chapter 10: The Power User's Toolkit

#### 10 Workflows That 10x Your Productivity

1. **Quick Status Check**: `openclaw status --all` gives a read-only, pasteable overview; `openclaw status --deep` probes live services.

2. **Doctor Everything**: `openclaw doctor` detects legacy config, migration issues, workspace problems, sandbox staleness, and auth failures. Run it first whenever something feels wrong.

3. **Multi-Channel Gateway**: Run `openclaw gateway run --bind loopback --port 18789 --force` then verify with `openclaw channels status --probe`.

4. **Dev Loop**: `pnpm gateway:watch` auto-restarts on file changes. Pair with `pnpm tui:dev` for rapid iteration.

5. **Agent per Channel**: Use routing bindings to assign different agents to different channels -- a support agent on Slack, a creative agent on Discord, a terse agent on Telegram.

6. **Swarm a Complex Task**: Use the swarm-orchestration skill or `/claude-flow-swarm` slash command to spin up a hierarchical swarm with specialized worker agents.

7. **Memory-Powered Context**: `openclaw memory` CLI commands let you store, search, and manage persistent knowledge across sessions. The hybrid backend with HNSW provides sub-millisecond semantic search.

8. **Plugin Hot-Loading**: Install channel plugins with `openclaw plugins install @openclaw/msteams` without restarting the gateway.

9. **Cron-Scheduled Tasks**: Use `openclaw cron` to schedule recurring agent tasks (e.g., daily digest, weekly reports).

10. **Shell Completion**: `openclaw completion` generates shell-specific scripts for bash/zsh/fish autocompletion.

#### Custom Skill Creation

Skills are directories under `.claude/skills/` containing a `SKILL.md` file with YAML frontmatter:

```markdown
---
name: "My Custom Skill"
description: "What this skill does and when to use it"
---

# My Custom Skill

## What This Skill Does
[Description]

## Quick Start
[Usage examples]
```

Use the `skill-builder` skill (`/home/user/openclaw/.claude/skills/skill-builder/`) to scaffold new skills.

#### The Swabble Component

Swabble (`/home/user/openclaw/Swabble/`) is a Swift 6.2 wake-word hook daemon for macOS 26. It uses Speech.framework for on-device, local-only speech recognition with zero network usage:

- **Wake word**: Default `clawd` (aliases `claude`), with `--no-wake` bypass
- **CLI commands**: `serve` (mic loop), `transcribe` (offline), `test-hook`, `mic list|set`, `doctor`, `status`, `service install|uninstall`
- **SwabbleKit**: Shared wake-word gate utilities usable as a Swift package dependency for iOS/macOS apps

#### Monitoring and Observability

- **Statusline** (`/home/user/openclaw/.claude/helpers/statusline.cjs`): Real-time display refreshing every 5 seconds showing V3 progress, security status, swarm state, hook activity, and performance metrics
- **Gateway logs**: `openclaw logs` with follow/tail/filter options
- **macOS logs**: `scripts/clawlog.sh` queries unified logs for the OpenClaw subsystem
- **Diagnostics**: The `extensions/diagnostics-otel` extension provides OpenTelemetry integration
- **Agent metrics**: `/monitoring/agent-metrics` slash command

#### Community Resources

- **Documentation**: https://docs.openclaw.ai (Mintlify-hosted, with zh-CN translations)
- **GitHub**: https://github.com/openclaw/openclaw
- **API Docs**: Gateway exposes OpenAI-compatible HTTP API (`/home/user/openclaw/docs/gateway/openai-http-api.md`) and OpenResponses API (`/home/user/openclaw/docs/gateway/openresponses-http-api.md`)
- **Provider docs**: 27 AI providers documented at `/home/user/openclaw/docs/providers/` (Anthropic, OpenAI, Ollama, vLLM, Bedrock, LiteLLM, OpenRouter, and many more)
- **Release process**: `/home/user/openclaw/docs/reference/RELEASING.md`

---

### Appendix: Quick Reference Card

```
# Daily workflow
openclaw doctor                          # Health check
openclaw status --all                    # Overview
openclaw gateway run --force             # Start gateway
openclaw channels status --probe         # Verify channels

# Development
pnpm openclaw <command>                  # Run CLI in dev
pnpm gateway:dev                         # Gateway dev (no channels)
pnpm test                                # Run tests
pnpm check                               # Lint + typecheck + format
pnpm build                               # Full build

# Testing
pnpm test:fast                           # Quick unit tests
pnpm test:coverage                       # With coverage
pnpm test:e2e                            # End-to-end
OPENCLAW_LIVE_TEST=1 pnpm test:live      # Live provider tests
pnpm test:docker:all                     # Full Docker suite

# Extensions
openclaw plugins install <name>          # Install plugin
openclaw plugins list                    # List installed
openclaw channels status                 # Check channels

# Memory
openclaw memory search "query"           # Semantic search
openclaw memory store "key" "value"      # Store entry

# Config
openclaw config set gateway.mode local   # Set config
openclaw config get gateway.mode         # Get config
openclaw configure                       # Interactive wizard
```