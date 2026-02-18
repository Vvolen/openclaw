# Universal Taxonomy of OpenClaw Power

## A Complete Reference to Every Customization Surface

---

### Introduction

OpenClaw is a multi-channel AI gateway with extensible messaging integrations, published as an open-source npm package (`openclaw`, version 2026.2.16, MIT license). It routes AI conversations across more than a dozen messaging channels -- Telegram, Discord, Slack, Signal, iMessage, WhatsApp, MS Teams, Matrix, Zalo, Voice, and more -- through a single gateway server that can be self-hosted via Docker, Fly.io, or bare metal.

The architecture philosophy is layered extensibility. The core runtime (`/home/user/openclaw/src/`) handles CLI wiring, agent management, routing, memory, and channel integration. A plugin system (`/home/user/openclaw/extensions/`) allows any channel or capability to be packaged as a standalone workspace. On top of this, a Claude Flow v3 orchestration layer (`.claude/` and `.claude-flow/`) adds multi-agent coordination, swarm topologies, self-learning memory, and proactive automation hooks.

This taxonomy matters because OpenClaw exposes an unusually deep customization surface -- spanning identity configuration, AI agent routing, plugin architecture, swarm orchestration, proactive hooks, memory systems, and deployment topology. A power user who understands these surfaces can tailor the platform to virtually any conversational AI use case.

---

### Section 1: Soul.md -- The Identity Layer

The identity layer is the foundational configuration that shapes how AI agents behave within the OpenClaw codebase. It lives in two symlinked files at the repository root:

- `/home/user/openclaw/AGENTS.md` -- the canonical "soul" document (234 lines)
- `/home/user/openclaw/CLAUDE.md` -- a symlink pointing to `AGENTS.md` (`ln -s AGENTS.md CLAUDE.md`)

This symlink pattern is a deliberate convention. The repository guidelines state: "When adding a new `AGENTS.md` anywhere in the repo, also add a `CLAUDE.md` symlink pointing to it." This ensures Claude Code and other AI tooling can discover the identity layer regardless of which filename convention they look for.

**Repository Guidelines Encoded in AGENTS.md:**

1. **Project Structure Rules**: Source in `src/`, tests colocated as `*.test.ts`, plugins in `extensions/*` as workspace packages, docs in `docs/` hosted on Mintlify at `docs.openclaw.ai`.

2. **Coding Standards**: TypeScript ESM with strict typing; no `any`. Formatting via Oxlint and Oxfmt. Files under approximately 500 LOC guideline. CLI progress uses `src/cli/progress.ts`; status tables use `src/terminal/table.ts`. Color palette lives in `src/terminal/palette.ts` -- no hardcoded colors.

3. **Naming Conventions**: "OpenClaw" for product/app/docs headings; `openclaw` for CLI command, package, paths, and config keys.

4. **Release Channels**:
   - `stable`: tagged releases (`vYYYY.M.D`), npm dist-tag `latest`
   - `beta`: prerelease tags (`vYYYY.M.D-beta.N`), npm dist-tag `beta`
   - `dev`: moving head on `main` (no tag)

5. **Commit Conventions**: Use `scripts/committer "<msg>" <file...>` instead of manual `git add`/`git commit`. Follow concise, action-oriented messages (e.g., `CLI: add verbose flag to send`).

6. **Multi-Agent Safety**: Never create/apply/drop git stashes. Never switch branches without explicit request. Scope commits to your changes only. Assume other agents may be working concurrently.

7. **Channel Awareness**: When refactoring shared logic, always consider all built-in channels (`src/telegram`, `src/discord`, `src/slack`, `src/signal`, `src/imessage`, `src/web`) plus extension channels (`extensions/*`).

8. **Tool Schema Guardrails**: Avoid `Type.Union` in tool input schemas; no `anyOf`/`oneOf`/`allOf`. Use `stringEnum`/`optionalStringEnum` for string lists. Keep top-level tool schema as `type: "object"`.

---

### Section 2: Skills and Memory -- The Knowledge Architecture

OpenClaw has a deeply layered knowledge system organized across multiple directories.

**Skills System (`.claude/skills/` -- 29 skill directories)**

Skills are specialized knowledge modules that agents can invoke. Each skill directory contains markdown files with detailed instructions, examples, and context. The 29 skill areas span:

- **AgentDB skills** (5): `agentdb-advanced`, `agentdb-learning`, `agentdb-memory-patterns`, `agentdb-optimization`, `agentdb-vector-search`
- **GitHub skills** (5): `github-code-review`, `github-multi-repo`, `github-project-management`, `github-release-management`, `github-workflow-automation`
- **V3 architecture skills** (9): `v3-cli-modernization`, `v3-core-implementation`, `v3-ddd-architecture`, `v3-integration-deep`, `v3-mcp-optimization`, `v3-memory-unification`, `v3-performance-optimization`, `v3-security-overhaul`, `v3-swarm-coordination`
- **Methodology skills**: `sparc-methodology`, `pair-programming`, `skill-builder`, `hooks-automation`, `stream-chain`, `swarm-advanced`, `swarm-orchestration`, `verification-quality`, `reasoningbank-agentdb`, `reasoningbank-intelligence`

Additionally, a top-level `/home/user/openclaw/skills/` directory contains 51 installable skill packages (e.g., `discord`, `slack`, `github`, `obsidian`, `notion`, `spotify-player`, `voice-call`, `weather`, `peekaboo`, `coding-agent`, `canvas`, etc.).

**Agent Templates (`.claude/agents/` -- 84 agent definition files across 24 subdirectories)**

Agent templates define the identity, capabilities, and hooks for each type of AI agent. They use YAML frontmatter for metadata and markdown for behavioral instructions. Key categories:

- **Core agents** (5): `coder.md`, `planner.md`, `researcher.md`, `reviewer.md`, `tester.md` -- located in `.claude/agents/core/`
- **Swarm coordinators** (3): `hierarchical-coordinator.md`, `mesh-coordinator.md`, `adaptive-coordinator.md`
- **Hive-mind agents** (5): `queen-coordinator.md`, `collective-intelligence-coordinator.md`, `scout-explorer.md`, `swarm-memory-manager.md`, `worker-specialist.md`
- **Consensus agents** (7): `byzantine-coordinator.md`, `raft-manager.md`, `gossip-coordinator.md`, `crdt-synchronizer.md`, `quorum-manager.md`, `security-manager.md`, `performance-benchmarker.md`
- **V3 specialized** (5): `v3-queen-coordinator.md`, `v3-security-architect.md`, `v3-performance-engineer.md`, `v3-memory-specialist.md`, `v3-integration-architect.md`
- **SONA learning**: `sona-learning-optimizer.md`
- **Dual-mode** (Codex integration): `codex-coordinator.md`, `codex-worker.md`, `dual-orchestrator.md`
- **Templates**: `automation-smart-agent.md`, `coordinator-swarm-init.md`, `github-pr-manager.md`, `implementer-sparc-coder.md`, `memory-coordinator.md`, `orchestrator-task.md`, `performance-analyzer.md`, `sparc-coordinator.md`

Each agent template (e.g., `/home/user/openclaw/.claude/agents/core/coder.md`) includes YAML frontmatter defining `name`, `type`, `color`, `description`, `capabilities`, `priority`, and `hooks` (pre/post scripts), followed by markdown instructions for behavior.

**Slash Commands (`.claude/commands/` -- 85 command definitions across 9 categories)**

- **Analysis** (7): compliance reports, bottleneck detection, performance analysis, token efficiency
- **Automation** (7): auto-agent, self-healing, session memory, smart agents, workflow selection
- **GitHub** (19): code review, PR management, issue triage, release management, multi-repo swarms
- **Hooks** (7): pre-edit, post-edit, pre-task, post-task, session-end, setup, overview
- **Monitoring** (6): agent metrics, real-time views, swarm monitoring, status
- **Optimization** (6): auto-topology, cache management, parallel execution, topology optimization
- **SPARC** (33): the full SPARC methodology suite -- specification, pseudocode, architecture, refinement, coding, debugging, security review, devops, documentation, orchestration, and more

**Memory Architecture**

The memory system (`/home/user/openclaw/src/memory/`) is a production-grade embedding and search engine with:

- **SQLite-vec backend** (`sqlite-vec.ts`, `sqlite.ts`) for persistent vector storage
- **Hybrid search** (`hybrid.ts`) combining vector similarity with keyword matching
- **Multi-provider embeddings** (`embeddings-openai.ts`, `embeddings-gemini.ts`, `embeddings-voyage.ts`) with batching support
- **HNSW indexing** for fast approximate nearest-neighbor search (150x-12,500x speedup)
- **QMD query parsing** (`qmd-query-parser.ts`, `qmd-manager.ts`) for structured memory queries with scope filtering
- **Session file sync** (`sync-session-files.ts`, `session-files.ts`) for persisting memory across sessions

**Claude Flow v3 Memory Configuration** (from `/home/user/openclaw/.claude/settings.json`):

```json
"memory": {
  "backend": "hybrid",
  "enableHNSW": true,
  "learningBridge": { "enabled": true },
  "memoryGraph": { "enabled": true },
  "agentScopes": { "enabled": true }
}
```

The LearningBridge connects insights to the SONA/ReasoningBank neural pipeline. Confidence evolves at +0.03 per access and -0.005/hour decay. The MemoryGraph builds a PageRank-based knowledge graph with community detection. AgentMemoryScope maps three scope levels: `project` (git root), `local` (git root local), and `user` (`~/.claude/agent-memory/`).

---

### Section 3: Heartbeat -- Proactive Intelligence

The hooks system in OpenClaw runs proactive intelligence at every stage of the development lifecycle. Configuration lives in `/home/user/openclaw/.claude/settings.json`.

**Hook Lifecycle Events:**

| Event | Handler | Timeout | Purpose |
|-------|---------|---------|---------|
| `SessionStart` | `hook-handler.cjs session-restore` + `auto-memory-hook.mjs import` | 15s + 8s | Restore session state and import auto-memory |
| `SessionEnd` | `hook-handler.cjs session-end` | 10s | Persist session state |
| `UserPromptSubmit` | `hook-handler.cjs route` | 10s | Route tasks to optimal agent via intelligence module |
| `PreToolUse` (Bash) | `hook-handler.cjs pre-bash` | 5s | Validate commands, block dangerous operations |
| `PostToolUse` (Write/Edit) | `hook-handler.cjs post-edit` | 10s | Record edit outcomes for learning |
| `Stop` | `auto-memory-hook.mjs sync` | 10s | Sync insights back to MEMORY.md |
| `PreCompact` (manual/auto) | `hook-handler.cjs compact-*` + `session-end` | 5-6s | Save state before context compaction |
| `SubagentStart` | `hook-handler.cjs status` | 3s | Report status on subagent spawn |
| `TeammateIdle` | `hook-handler.cjs post-task` | 5s | Auto-assign work to idle teammates |
| `TaskCompleted` | `hook-handler.cjs post-task` | 5s | Train patterns, notify lead agent |

**The Hook Handler** (`/home/user/openclaw/.claude/helpers/hook-handler.cjs`) is the central dispatcher. It loads four helper modules:
- `router.cjs` -- task routing with confidence scoring
- `session.cjs` -- session persistence and metrics
- `memory.cjs` -- memory backend operations
- `intelligence.cjs` -- contextual intelligence for routing decisions

The `pre-bash` handler blocks dangerous commands (e.g., `rm -rf /`, fork bombs). The `route` handler analyzes prompts and produces agent recommendations with confidence percentages.

**Auto-Memory Bridge** (`/home/user/openclaw/.claude/helpers/auto-memory-hook.mjs`) implements ADR-048/049. On `SessionStart`, it imports auto-memory files from disk into the backend. On `Stop`, it syncs insights back. It uses a `JsonFileBackend` storing entries at `.claude-flow/data/auto-memory-store.json`.

**Background Daemon Workers** (configured in `.claude/settings.json`):

10 daemon workers run on configurable schedules:
- `audit` (1h, critical priority) -- security auditing
- `optimize` (30m, high priority) -- performance optimization
- `consolidate` (2h, low priority) -- memory consolidation
- `document` (1h, triggers: `adr-update`, `api-change`) -- auto-documentation
- `deepdive` (4h, triggers: `complex-change`) -- deep analysis
- `ultralearn` (1h) -- deep knowledge acquisition
- `map`, `testgaps`, `refactor`, `benchmark` -- codebase mapping, test gap analysis, refactoring suggestions, benchmarking

**Pre-Commit Hooks** (`/home/user/openclaw/git-hooks/pre-commit`): Runs Oxlint (`--type-aware --fix`) on staged lint-eligible files, then Oxfmt (`--write`) on staged format-eligible files, then re-stages the fixed files.

**Status Line** (`.claude/settings.json`): A continuously refreshing status line (`node .claude/helpers/statusline.cjs`, 5-second refresh) provides real-time system state in the Claude Code interface.

---

### Section 4: Tools and Integrations -- The Capability Surface

**MCP Integration** (`/home/user/openclaw/.mcp.json`):

The Model Context Protocol server is configured as:
```json
{
  "mcpServers": {
    "claude-flow": {
      "command": "npx",
      "args": ["@claude-flow/cli@latest", "mcp", "start"],
      "env": {
        "CLAUDE_FLOW_MODE": "v3",
        "CLAUDE_FLOW_TOPOLOGY": "hierarchical-mesh",
        "CLAUDE_FLOW_MAX_AGENTS": "15",
        "CLAUDE_FLOW_MEMORY_BACKEND": "hybrid"
      }
    }
  }
}
```

This provides the MCP tool surface for Claude agents: memory store/search/list, agent spawn/list, swarm management, task assignment, and more.

**Extension/Plugin Architecture** (`/home/user/openclaw/extensions/` -- 36 extensions):

Extensions are workspace packages that follow the plugin SDK contract (`openclaw/plugin-sdk`). Each extension has its own `package.json` with runtime deps in `dependencies` (installed via `npm install --omit=dev`). The convention: put `openclaw` in `devDependencies` or `peerDependencies`, never in `dependencies`.

The 36 current extensions span:
- **Messaging channels**: `telegram`, `discord`, `slack`, `signal`, `imessage`, `whatsapp`, `msteams`, `matrix`, `zalo`, `zalouser`, `line`, `googlechat`, `feishu`, `nostr`, `irc`, `mattermost`, `nextcloud-talk`, `tlon`, `twitch`
- **Voice**: `voice-call`, `talk-voice`
- **Auth adapters**: `google-antigravity-auth`, `google-gemini-cli-auth`, `copilot-proxy`, `minimax-portal-auth`, `qwen-portal-auth`
- **Capabilities**: `memory-core`, `memory-lancedb`, `llm-task`, `lobster`, `open-prose`, `phone-control`, `diagnostics-otel`, `device-pair`, `thread-ownership`, `bluebubbles`

The plugin system in `src/plugins/` handles discovery (`discovery.ts`), loading (`loader.ts`), installation (`install.ts`), manifest validation (`manifest-registry.ts`, `schema-validator.ts`), hook wiring (`hooks.ts`, `wired-hooks-*.test.ts`), and slot-based UI extension (`slots.ts`).

**Channel System** (built-in in `src/`):

Core channels live alongside the extension channels: `src/telegram/`, `src/discord/`, `src/slack/`, `src/signal/`, `src/imessage/`, `src/web/` (WhatsApp web), `src/whatsapp/`, `src/channels/`, `src/line/`. The channel registry (`src/channels/registry.ts`) normalizes channel IDs across all surfaces. Channel configuration includes allowlists (`src/channels/allowlists/`), command gating (`src/channels/command-gating.ts`), mention gating, typing indicators, and sender identity resolution.

**CLI Commands and Options** (`/home/user/openclaw/src/cli/`):

The CLI is built on Commander.js and provides: `gateway` (run/dev/reset), `config`, `channels`, `message`, `agent`, `status`, `doctor`, `sandbox`, `browser`, `plugins`, `nodes`, `memory`, `models`, `hooks`, `cron`, `daemon`, `tui`, `pairing`, `security`, `skills`, `update`, `logs`, `dns`, `webhooks`, `completion`, and more.

**Build Tooling**:
- Package manager: pnpm 10.23.0 (with Bun also supported)
- Bundler: tsdown (rolldown-based)
- Type checking: TypeScript 5.9 with `@typescript/native-preview` (tsgo)
- Linting: Oxlint with type-aware rules
- Formatting: Oxfmt
- Testing: Vitest 4.x with V8 coverage
- Runtime: Node 22+ baseline

---

### Section 5: Multi-Agent Routing -- The Binding System

The routing architecture at `/home/user/openclaw/src/routing/` is the binding system that determines which AI agent handles which conversation.

**Route Resolution** (`/home/user/openclaw/src/routing/resolve-route.ts`):

The `resolveAgentRoute()` function takes a `ResolveAgentRouteInput` containing: config, channel, accountId, peer (kind + id), parentPeer, guildId, teamId, and memberRoleIds. It evaluates bindings in a tiered priority order:

1. **`binding.peer`** -- Direct peer match (highest priority): matches specific user/group/channel IDs
2. **`binding.peer.parent`** -- Parent peer inheritance for threads
3. **`binding.guild+roles`** -- Guild + Discord role-based routing
4. **`binding.guild`** -- Guild-only match (no role constraint)
5. **`binding.team`** -- Team-based match (e.g., Slack workspace)
6. **`binding.account`** -- Account-specific match (non-wildcard)
7. **`binding.channel`** -- Channel-wide wildcard match
8. **`default`** -- Falls through to the default agent

**Session Key Architecture** (`/home/user/openclaw/src/routing/session-key.ts`):

Session keys use the format `agent:<agentId>:<scope>` with DM scoping options:
- `main` -- All DMs collapse to one session per agent
- `per-peer` -- Separate session per unique peer
- `per-channel-peer` -- Separate session per channel + peer combination
- `per-account-channel-peer` -- Full isolation by account + channel + peer

Identity linking (`identityLinks` config) allows cross-channel identity resolution so the same person on Telegram and Discord maps to one session.

**Agent Team Configuration** (from `.claude/settings.json`):

```json
"agentTeams": {
  "enabled": true,
  "teammateMode": "auto",
  "taskListEnabled": true,
  "mailboxEnabled": true,
  "coordination": {
    "autoAssignOnIdle": true,
    "trainPatternsOnComplete": true,
    "notifyLeadOnComplete": true,
    "sharedMemoryNamespace": "agent-teams"
  }
}
```

**Swarm Topologies** (from `.claude-flow/CAPABILITIES.md`):

| Topology | Description | Best For |
|----------|-------------|----------|
| `hierarchical` | Queen controls workers directly | Anti-drift, tight control |
| `mesh` | Fully connected peer network | Distributed tasks |
| `hierarchical-mesh` | V3 hybrid (current default) | 10+ agents |
| `ring` | Circular communication | Sequential workflows |
| `star` | Central coordinator | Simple coordination |
| `adaptive` | Dynamic based on load | Variable workloads |

The current configuration uses `hierarchical-mesh` with a maximum of 15 agents. Strategies include `balanced` (even distribution), `specialized` (clear roles, no overlap), and `adaptive` (dynamic routing).

**Hive-Mind Consensus** supports Byzantine Fault Tolerance (f < n/3 faulty nodes), Raft (leader-based, f < n/2), Gossip (eventually consistent), CRDT (conflict-free), and Quorum (configurable). Queen types include Strategic (long-term planning), Tactical (execution), and Adaptive (dynamic optimization).

---

### Section 6: Self-Correction and Resilience

**Testing Framework**:

Vitest with V8 coverage, configured in `/home/user/openclaw/vitest.config.ts`:
- Coverage thresholds: 70% lines, 70% functions, 55% branches, 70% statements
- Test isolation: `unstubEnvs: true`, `unstubGlobals: true`, `pool: "forks"`
- Workers: max 16 local, 2-3 in CI
- Test naming: `*.test.ts` (unit), `*.e2e.test.ts` (end-to-end), `*.live.test.ts` (live/integration)
- Multiple Vitest configs: `vitest.unit.config.ts`, `vitest.e2e.config.ts`, `vitest.live.config.ts`, `vitest.gateway.config.ts`, `vitest.extensions.config.ts`
- Extensive Docker-based test suite: `test:docker:live-models`, `test:docker:live-gateway`, `test:docker:onboard`, `test:docker:gateway-network`, `test:docker:plugins`, `test:docker:qr`, `test:docker:doctor-switch`

**Pre-Commit Quality Gates** (`/home/user/openclaw/git-hooks/pre-commit`):

Every commit runs through: (1) NUL-delimited staged file collection (security hardened against option-injection), (2) Oxlint `--type-aware --fix` on lint-eligible files, (3) Oxfmt `--write` on format-eligible files, (4) automatic re-staging of fixed files.

**Security Scanning** (`/home/user/openclaw/.claude/helpers/security-scanner.sh`):

Runs every 30 minutes (or on demand). Scans `src/` for hardcoded passwords, API keys, secrets, tokens, and private keys using regex pattern matching. Results are stored in `.claude-flow/security/scan-results.json`. The `.claude/settings.json` configures automatic security scanning:
```json
"security": {
  "autoScan": true,
  "scanOnEdit": true,
  "cveCheck": true,
  "threatModel": true
}
```

The settings also deny reading `.env` and `.env.*` files: `"deny": ["Read(./.env)", "Read(./.env.*)"]`.

**Docker Sandboxing**: Three Dockerfiles for different sandboxing levels:
- `/home/user/openclaw/Dockerfile` -- Production image (Node 22 bookworm, non-root user, Bun for builds)
- `/home/user/openclaw/Dockerfile.sandbox` -- Sandbox environment
- `/home/user/openclaw/Dockerfile.sandbox-browser` -- Browser-capable sandbox
- `/home/user/openclaw/Dockerfile.sandbox-common` -- Shared sandbox base

The production Dockerfile includes security hardening: runs as non-root `node` user (uid 1000), binds to loopback by default.

**CI/CD Pipeline** (`/home/user/openclaw/.github/workflows/ci.yml`):

The CI workflow runs on push to `main` and all PRs. It includes: (1) docs-scope detection to skip heavy jobs on docs-only changes, (2) changed-scope detection to skip unrelated platform tests, (3) lint/format checks, (4) type checking, (5) unit tests, (6) e2e tests, (7) platform-specific builds (Node, macOS, Android). PR concurrency is managed with `cancel-in-progress` for the same PR number.

**Error Handling Patterns**: The codebase uses structured error types (`src/infra/errors.ts`), retry policies (`src/infra/retry-policy.ts`, `src/infra/retry.ts`), backoff strategies (`src/infra/backoff.ts`), unhandled rejection handling (`src/infra/unhandled-rejections.ts`), and runtime guards (`src/infra/runtime-guard.ts`).

---

### Section 7: Top 10 High-Leverage Power Customizations

These are the 10 most impactful customizations a power user can make to OpenClaw, ranked by leverage (effort-to-impact ratio).

**1. Agent Bindings for Per-Channel/Per-User Routing**

- **What it does**: Routes different channels, users, guilds, or roles to different AI agents with different personalities, models, and system prompts.
- **How to configure**: Add entries to the `bindings` array in the OpenClaw config. Each binding specifies `match` (channel, accountId, peer, guildId, roles) and `agentId`.
- **Expected impact**: Extremely high. A single config change lets you run a customer-support agent on Telegram, a coding assistant on Discord, and a creative writing agent on Slack -- all through one gateway.

**2. Custom Agent Definitions**

- **What it does**: Creates new agent identities with unique system prompts, model preferences, tool capabilities, and behavioral hooks.
- **How to configure**: Add a new `.md` file in `.claude/agents/custom/` or any agents subdirectory with YAML frontmatter (name, type, capabilities, priority, hooks) and markdown instructions.
- **Expected impact**: High. Each agent becomes a purpose-built persona. The 84 existing templates provide a starting point for virtually any use case.

**3. Plugin/Extension Development**

- **What it does**: Adds entirely new messaging channels, AI providers, or capability modules.
- **How to configure**: Create a new directory under `extensions/<name>/` with its own `package.json`. Implement the plugin SDK interface (`openclaw/plugin-sdk`). Runtime deps go in `dependencies`; `openclaw` goes in `devDependencies`.
- **Expected impact**: High. The 36 existing extensions demonstrate the pattern. Adding a new channel takes the AI gateway to any platform the user base lives on.

**4. DM Session Scoping and Identity Links**

- **What it does**: Controls whether DM conversations are isolated per user, shared across channels, or unified by identity.
- **How to configure**: Set `session.dmScope` to `main`, `per-peer`, `per-channel-peer`, or `per-account-channel-peer`. Configure `session.identityLinks` to map cross-channel user identities.
- **Expected impact**: High. Determines whether the AI "remembers" a user across channels. Critical for personalized multi-channel experiences.

**5. Swarm Topology Selection**

- **What it does**: Changes how multiple AI agents coordinate -- hierarchical control, peer mesh, hybrid, ring, star, or adaptive.
- **How to configure**: In `.claude/settings.json` under `claudeFlow.swarm.topology`, set to one of: `hierarchical`, `mesh`, `hierarchical-mesh`, `ring`, `star`, `adaptive`. Adjust `maxAgents` (current: 15).
- **Expected impact**: Medium-high. For complex tasks, the topology determines quality, speed, and reliability. `hierarchical-mesh` (current default) is recommended for 10+ agents.

**6. Hook Lifecycle Customization**

- **What it does**: Injects custom logic at every stage: session start/end, tool use, prompt submission, compaction, task completion, and teammate idle events.
- **How to configure**: Modify the `hooks` section in `.claude/settings.json`. Each hook specifies a `type` ("command"), `command` (script path), and `timeout`. Add new hooks by creating scripts in `.claude/helpers/` and referencing them.
- **Expected impact**: Medium-high. Hooks are the "heartbeat" -- they enable auto-memory, routing intelligence, dangerous-command blocking, edit tracking, and session persistence without modifying core code.

**7. Daemon Worker Schedules**

- **What it does**: Configures background workers for continuous optimization -- security auditing, performance tuning, memory consolidation, documentation generation, test gap analysis.
- **How to configure**: Under `claudeFlow.daemon.schedules` in `.claude/settings.json`, adjust `interval` and `priority` for each of the 10 workers (`audit`, `optimize`, `consolidate`, `document`, `deepdive`, `ultralearn`, `map`, `testgaps`, `refactor`, `benchmark`). Add triggers like `adr-update` or `complex-change`.
- **Expected impact**: Medium. Continuous background intelligence compounds over time. A well-tuned audit schedule catches security issues before they ship.

**8. Memory Backend and Learning Configuration**

- **What it does**: Tunes the hybrid memory system -- HNSW indexing speed, PageRank graph parameters, confidence decay rates, consolidation thresholds, and agent memory scoping.
- **How to configure**: Edit `.claude-flow/config.yaml` for runtime memory settings: `learningBridge.confidenceDecayRate` (default: 0.005), `learningBridge.accessBoostAmount` (default: 0.03), `memoryGraph.pageRankDamping` (default: 0.85), `memoryGraph.maxNodes` (default: 5000). Agent scopes default to `project`.
- **Expected impact**: Medium. Tuning memory directly affects how well the system learns from past interactions and how quickly patterns surface in future work.

**9. Installable Skills Packages**

- **What it does**: Extends agent capabilities with pre-built skill packages -- from messaging integrations (Discord, Slack) to productivity tools (Obsidian, Notion, Trello) to media (Spotify, voice-call, video-frames).
- **How to configure**: Skills live in `/home/user/openclaw/skills/` (51 available). Install via the `openclaw skills` CLI. Each skill package provides tool definitions, auth flows, and behavioral instructions.
- **Expected impact**: Medium. Each skill instantly gives agents new capabilities. The `coding-agent`, `github`, and `canvas` skills are particularly high-value for development workflows.

**10. Deployment Topology (Docker, Fly.io, Bare Metal)**

- **What it does**: Determines where and how the gateway runs -- local Docker with compose, cloud Fly.io with persistent volumes, or direct Node.js.
- **How to configure**: Docker: use `docker-compose.yml` with env vars (`OPENCLAW_GATEWAY_TOKEN`, `OPENCLAW_CONFIG_DIR`, `OPENCLAW_WORKSPACE_DIR`). Fly.io: use `fly.toml` (current: `shared-cpu-2x`, 2GB RAM, `iad` region, persistent volume at `/data`). Bare metal: `node dist/index.js gateway --bind lan --port 18789`.
- **Expected impact**: Medium. Self-hosting gives full control over data residency, latency, and cost. Fly.io provides zero-ops deployment with `auto_start_machines` and persistent state.

---

### Summary

OpenClaw's customization surface spans seven distinct layers: identity configuration (AGENTS.md), knowledge architecture (29 skills, 84 agent templates, 85 commands, 51 installable skill packages), proactive hooks (10 lifecycle events, 10 daemon workers), tools and integrations (36 extensions, MCP server, multi-provider memory), multi-agent routing (tiered binding resolution, 6 swarm topologies, 5 consensus mechanisms), self-correction (Vitest coverage gates, pre-commit hooks, security scanning, Docker sandboxing), and deployment topology (Docker, Fly.io, bare metal). Each layer is independently configurable, and the layers compose to support use cases ranging from a single-user CLI assistant to a 15-agent swarm serving conversations across a dozen messaging platforms simultaneously.