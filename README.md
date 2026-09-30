# Claude Code Playbook

I'm Alec Foster, Chief Agent Officer & Responsible AI Lead at the Marketing + Media Alliance (MMA). This is the Claude Code setup I run every day, written for experienced users and for people configuring Claude Code for a team: the architecture, the settings and the reasons behind them, the multi-model routing, and 34 lessons that each cost something to learn. It is extracted from my private setup notes with everything organization-specific removed. Facts are dated: every version-sensitive claim was checked against Claude Code 2.1.286 and codex-cli 0.159.2 on 2026-09-30, and anything marked with an earlier date was verified then and has not changed since.

Related public repos:

- **[cross-model-agent-delegation](https://github.com/alectivism/cross-model-agent-delegation):** the Claude Code and Codex delegation kit described in the multi-model section, with the wrapper scripts.
- **[MAVEN](https://github.com/alectivism/maven-template):** my AI chief-of-staff template for Claude Code.
- **[organization-ai-skills](https://github.com/alectivism/organization-ai-skills):** a skill pack for organizations rolling Claude out to staff.

More at [alecfoster.com](https://www.alecfoster.com).

## Contents

1. [Operating principles](#operating-principles)
2. [Three-layer architecture: personal repo, team template, org plugin](#three-layer-architecture-personal-repo-team-template-org-plugin)
3. [Configuration layout and key settings](#configuration-layout-and-key-settings)
4. [MCP strategy: connectors, local servers, and context cost](#mcp-strategy-connectors-local-servers-and-context-cost)
5. [Adaptive MCP profiles per project](#adaptive-mcp-profiles-per-project)
6. [Multi-model orchestration: Codex, Gemini, and quota routing](#multi-model-orchestration-codex-gemini-and-quota-routing)
7. [Skills: loading, descriptions, and retirement](#skills-loading-descriptions-and-retirement)
8. [Subagents: model pinning, return formats, and nesting](#subagents-model-pinning-return-formats-and-nesting)
9. [Rules, skills, and hooks as enforcement layers](#rules-skills-and-hooks-as-enforcement-layers)
10. [Permissions, secrets, and the security model](#permissions-secrets-and-the-security-model)
11. [Statusline for rate limits and context](#statusline-for-rate-limits-and-context)
12. [Remote control, notifications, and agent teams](#remote-control-notifications-and-agent-teams)
13. [Memory and session continuity](#memory-and-session-continuity)
14. [Verification and adversarial review](#verification-and-adversarial-review)
15. [Lessons learned](#lessons-learned)
16. [License](#license)

---

## Operating principles

I treat Claude Code as an operating layer that sits between me and every tool I use: email, calendar, task tracking, web research, drafting, code, and automation. The goal is one interface and no context switching. Five principles shape everything below:

- **One interface:** email, chat, tasks, documents, research, and code are all reachable from the same terminal session through MCP servers and CLIs.
- **Context persists:** Claude Code's auto-memory (a `MEMORY.md` index plus one file per fact) carries preferences and project facts between sessions, and a task tracker carries active work. I re-explain nothing.
- **Proactive surfacing:** a daily-briefing skill pulls new mail, unanswered sent mail, the calendar, and open commitments into one ranked list before I ask.
- **Parallel by default:** independent tasks go to subagents in one message, and the agent that researches a piece of content also writes it.
- **Several model families:** Claude orchestrates; GPT (through the Codex CLI) and Gemini handle cross-family review, large-file synthesis, and Google Search grounding.

## Three-layer architecture: personal repo, team template, org plugin

The setup serves three audiences, and each gets its own layer:

```
Personal assistant repo (full integrations, private)
  |
  |-- sync, strip private detail --> Team template (public, for power users)
  |
  |-- sync, strip private detail --> Org plugin marketplace (every seat, via the Claude admin console)
```

- **Personal assistant repo:** my working instance. About 22 local MCP servers plus about 40 claude.ai connectors, auto-memory, multi-model CLI access, and auto-mode permissions with a curated allowlist. It is a git repo with a `CLAUDE.md`, a `.claude/rules/` folder of auto-loaded rules, project skills, and slash commands.
- **Team template:** a public GitHub template (mine is [MAVEN](https://github.com/alectivism/maven-template)) that a colleague clones to get org context, brand rules, and a set of skills, with a setup script. It bundles no integrations; each person adds their own.
- **Org plugin marketplace:** a private plugin repo pushed to every seat through the Claude organization admin console. It holds skills with org-specific substance that staff cannot easily reproduce, plus org-defined subagents so everyone delegates to the same pinned targets. Generic drafting skills stay out of it; people build those for themselves with the skill-creator.

**Two-branch context model:** the personal layer holds operational detail (file-share paths, channel IDs, integration configs). The outer layers get the same org facts with that detail removed, because staff do not have the same tools and a path they cannot open wastes their context and confuses the model. Changes flow personal-first, then outward, scrubbed on the way.

## Configuration layout and key settings

Configuration lives at two levels.

### User level: `~/.claude/`

```
~/.claude/
  CLAUDE.md              # Communication style, safety rules, delegation policy
  settings.json          # Plugins, permissions, hooks, effort, statusline, env flags
  settings.local.json    # Machine-local allowlist and skill overrides (not synced)
  remote-settings.json   # Permission layer for sessions driven from web or mobile
  keybindings.json       # Custom key bindings
  agents/                # Shared subagents, each pinning model and effort
  hooks/                 # PreToolUse guards (blocked tools, git push, attribution, large reads)
  scripts/
    mcp-profile.sh       # SessionStart hook: detect project type, toggle MCPs and plugins
    mcp-profiles.conf    # Directory-to-profile overrides
    quota-probe.sh       # Prints the QUOTA routing line each turn
    statusline.sh        # Two-line statusline
  mcp-servers/           # Custom MCP server code (thin API wrappers)
  skills/                # User-level skills available in every project
  projects/              # Per-project auto-memory directories
```

MCP server definitions live in `~/.claude.json`, which Claude Code writes when you run `claude mcp add`.

### Project level: `<repo>/.claude/`

```
<repo>/.claude/
  rules/                 # Auto-loaded every session: small, focused files
  skills/                # On-demand project skills
  commands/              # Slash commands
  agents/                # Project-scoped subagents
```

Rules files hold short always-on guardrails (naming, voice, routing). Long reference material goes in skills, which load only when relevant.

### Key settings in `settings.json`

```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1",
    "CLAUDE_AUTOCOMPACT_PCT_OVERRIDE": "70",
    "CLAUDE_CODE_AUTO_COMPACT_WINDOW": "1000000",
    "CLAUDE_CODE_SUPPRESS_SESSION_ATTRIBUTION": "1"
  },
  "permissions": { "defaultMode": "auto", "allow": ["..."], "deny": ["..."] },
  "effortLevel": "medium",
  "tui": "fullscreen",
  "theme": "auto",
  "teammateMode": "auto",
  "remoteControlAtStartup": true,
  "agentPushNotifEnabled": true,
  "awaySummaryEnabled": false,
  "autoDreamEnabled": true,
  "statusLine": { "type": "command", "command": "~/.claude/scripts/statusline.sh" }
}
```

What each one buys:

- **`permissions.defaultMode: auto`:** runs everything on the curated `allow` list without a prompt, hard-blocks the `deny` list, and lets the auto-mode classifier decide the rest. See [Permissions](#permissions-secrets-and-the-security-model).
- **`effortLevel: medium`:** I ran `xhigh`, then `high`, and reverted to `medium` on 2026-08-03. High effort on Opus 5 made ordinary work worse: it over-planned bounded tasks, added scope nobody asked for, and revised correct first answers. Raise effort per task with `/effort`; agents that need depth pin `effort: high` in their own frontmatter, which survives the global default.
- **Model:** set with `/model`; there is no `model` key in my `settings.json`. On 2.1.286 the aliases `opus`, `fable`, `sonnet`, and `haiku` resolve to Opus 5.5 (`claude-opus-5-5`, my orchestrator), Fable 5.1 (`claude-fable-5-1`), Sonnet 5.5 (`claude-sonnet-5-5`), and Haiku 4.5 (`claude-haiku-4-5-20251001`). No subagent runs on Fable.
- **`CLAUDE_CODE_AUTO_COMPACT_WINDOW=1000000`:** compaction math tracks the 1M-token window instead of the 200K default.
- **`CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=70`:** compaction starts at 70% of the window, leaving headroom before the summary runs.
- **`CLAUDE_CODE_SUPPRESS_SESSION_ATTRIBUTION=1`:** no Claude attribution lines in commits or PRs. A `git-attribution-guard.py` hook backs it up, because the setting has been reported to revert silently after updates.
- **`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` and `teammateMode: auto`:** named, addressable teammate agents that render in split panes when the terminal supports it. See [Agent teams](#agent-teams-and-terminal-orchestration).
- **`tui: fullscreen`:** works around a rendering glitch where scrolling duplicated text indefinitely.
- **`remoteControlAtStartup` and `agentPushNotifEnabled`:** every session is steerable from web and mobile from its first prompt, and I get a push when a background agent finishes or blocks.
- **`autoDreamEnabled`:** background consolidation of auto-memory between sessions.

## MCP strategy: connectors, local servers, and context cost

Two kinds of MCP server reach a Claude Code session, and they behave differently.

- **claude.ai connectors (remote):** configured once in claude.ai and available on web, mobile, desktop, and Claude Code with no local install. Good defaults for mainstream SaaS (mail, chat, docs, meeting transcripts, search).
- **Local servers (`claude mcp add`):** run on your machine, can be scoped per project, and can do things a connector will not. My clearest case: the hosted Microsoft 365 connector strips `style=`, `<span>`, and `<img>` from mail HTML, so a styled email signature is impossible through it. A local Graph server (Softeria's `ms365`) keeps all three, so drafting and sending go through the local server and the connector handles search.

Rules I follow:

- **Pick one route per job and deny the other:** when a connector and a local server both offer a write tool for the same system, put the connector's write tools in `permissions.deny` so the model cannot choose the weaker path.
- **Never run the same server twice:** a hand-written `mcpServers` entry plus the same package installed as a managed connector produced one healthy instance and one crash-looping one. Prefer the managed copy; diagnose duplicates by server-name casing in the logs.
- **Route GPT through the subscription CLI:** an API-key GPT MCP wrapper billed per token while the same models sat on a flat ChatGPT subscription through Codex. The wrapper is unregistered and a PreToolUse hook blocks any `mcp__openai__*` call.
- **Keep names lowercase and hyphenated:** every non-alphanumeric character in a server name becomes `_` in tool names, so `Notion (Personal)` surfaces as `mcp__Notion__Personal___*`.

### Tool-schema cost and cold starts

Every enabled server costs context. On 2.1.286, MCP tool schemas are deferred: the session sees tool names, and full JSON schemas load on demand through the `ToolSearch` tool. That makes a large roster affordable, but the name list itself still grows with each server, and plugins still load their commands, agents, and skill descriptions into every session. Two consequences:

- **Prune plugins aggressively:** I keep 6 marketplace plugins enabled (feature-dev, code-review, code-simplifier, commit-commands, claude-md-management, claude-code-setup) and toggle the rest per project. LSP plugins (TypeScript, Pyright, and others) stay off by default and turn on only in coding profiles.
- **Strip MCP from headless worker processes:** a Codex run inheriting my MCP and plugin config took minutes to cold-start. The fix is `--ignore-user-config` (see the [Codex section](#codex-cli-through-a-wrapper)); the obvious-looking `-c 'mcp_servers={}'` does nothing.

### Two accounts on one service: the URL collision

Claude Code identifies a remote MCP server by its exact URL string. If a local server points at the same URL as a claude.ai connector, both are treated as one server and the local copy wins: the connector goes inactive in that project. I verified this on 2026-07-30 with two Notion accounts, one through the connector and one through a local server, both on `https://mcp.notion.com/mcp`.

Any distinguishing query parameter makes the strings differ, and both stay live:

```bash
cd ~/personal-project
claude mcp add --transport http notion-personal 'https://mcp.notion.com/mcp?account=personal'
```

```
claude.ai Notion: https://mcp.notion.com/mcp - ✔ Connected
notion-personal:  https://mcp.notion.com/mcp?account=personal (HTTP) - ✔ Connected
```

Notion ignores the unknown parameter. If a provider validates the canonical URL and rejects it, two other ways to break the match:

- **mcp-remote bridge:** `claude mcp add notion-personal --env MCP_REMOTE_CONFIG_DIR=$HOME/.mcp-auth/notion-personal -- npx -y mcp-remote https://mcp.notion.com/mcp` registers a stdio command with no `url` field, and each account gets its own token directory.
- **Provider token over stdio:** `@notionhq/notion-mcp-server` with an integration token created inside the target workspace. One token maps to exactly one account; keep the token in your secrets store, never in `~/.claude.json`.

Three details that bite:

- **Scope the second account per project:** `claude mcp add` run inside a project writes local scope, so the personal server loads only there.
- **The browser picks the identity:** OAuth consent authorizes whichever account is signed in. Run it in a private window signed in as the account you want, then read a page only that account can see to confirm.
- **A legacy endpoint exists:** `https://mcp.notion.com/sse` also answers, as a fallback with a different URL.

## Adaptive MCP profiles per project

A SessionStart hook (`mcp-profile.sh`) detects what kind of project I opened and switches the tool roster to match, so a coding session does not carry mail tools and a writing session does not carry LSPs.

How it works:

1. On session start the script checks `mcp-profiles.conf` for an explicit directory-to-profile mapping (prefix match).
2. If none matches, it detects the type from file signatures: `package.json` plus `tsconfig.json`, `pyproject.toml`, `firebase.json`, `Cargo.toml`, `go.mod`.
3. It toggles user-scope MCP servers from per-profile `MCP_ON` / `MCP_OFF` lists, and plugins (context7, playwright, frontend-design, LSPs, security-guidance) the same way.
4. It injects a system message naming the active profile, so the model knows which tools to expect.

| Profile | Trigger | Tools on |
|---|---|---|
| **org** | Work repos (explicit mapping) | Mail and calendar, web search, scraping, Gemini, task tracker, meeting transcripts, browser automation. Coding plugins off. |
| **coding** | Code file signatures | Web search, scraping, Gemini, browser automation; context7, frontend-design, TypeScript and Python LSPs, security-guidance. |
| **content** | Content repos | Web search, scraping, Gemini, workflow automation, page fetch, social scrapers, semantic search. |
| **none** | Notes vaults | Every toggled server off. |
| **default** | Everything else | Web search, Gemini, and browser automation only. |

Example `mcp-profiles.conf`:

```
~/assistant=org
~/org-plugin=org
~/side-project=coding
~/content=content
~/notes=none
```

The limit: the script toggles user-scope servers only. A server added with local scope inside a project (like the second Notion account above) is outside its reach, which is the point of local scope.

## Multi-model orchestration: Codex, Gemini, and quota routing

Claude Code orchestrates four model families: Claude (Team plan, Premium seat), GPT through the Codex CLI (ChatGPT Pro subscription), Gemini through `gemini -p` (free API tier, Flash only) and Google's Antigravity CLI `agy` (Google AI Pro subscription), and Grok through an MCP wrapper (API-billed). Subscriptions are rate-limited, so routing follows a quota signal described below.

Three tiers of access:

- **MCP tools, inline:** Gemini chat with Google Search grounding, Gemini image and video generation (API-billed), Grok for X context. Structured, low overhead.
- **CLI workers, file-heavy:** a CLI model reads files itself, reasons independently, and returns only a summary. When `gemini -p` reads 15 files and returns 200 words, Claude ingests 200 words.
- **Combined workshopping:** the same prompt through several models, then a synthesis of where they diverge. Useful for strategy calls, voice comparisons in drafting, and tiebreaks.

```bash
# Gemini CLI, Flash on the free API tier. @file refs go INSIDE the prompt
# and must sit within the current workspace.
gemini -p "summarize @doc1.md and @doc2.md into 5 bullets"

# Antigravity CLI: Gemini 3.1 Pro and other models on the AI Pro subscription.
# Tight weekly quota, so use it for second opinions.
agy --model "Gemini 3.1 Pro (High)" -p "<prompt>"

# Codex CLI: always through the wrapper, which owns every flag.
codex-run.sh review "<full-context prompt>"
# The last stdout line is OUT=<file>. Read that file. Piping Codex stdout through
# head or tail once silently truncated a real finding.
```

**Gemini free-tier limits:** Google deprecated the free Code Assist login that `gemini -p` used, and on 2026-07-01 it began erroring. The fix was setting `security.auth.selectedType` to `gemini-api-key` in `~/.gemini/settings.json`. The free API tier is Flash-only and trains on prompts, with human review possible, so confidential data stays out of it unless billing is on.

### Codex CLI through a wrapper

The model classifies the task into one enum; a script owns every flag. `codex-run.sh <class>` maps each class to a tier and effort, and a resolver picks the newest model of each family from Codex's live catalog (`~/.codex/models_cache.json`) at call time. Full scripts are in the [delegation kit](https://github.com/alectivism/cross-model-agent-delegation).

| Class | Tier / effort | Model on 2026-09-30 |
|---|---|---|
| `review` | frontier / medium | GPT-6 Astra (`gpt-6-astra`), about 5x Sol's usage per call |
| `hardest` | frontier / high | GPT-6 Astra |
| `implement`, `explore`, `ingest`, `prose` | standard / medium | GPT-6.1 Sol (`gpt-6.1-sol`) |
| `commit` | fast / low | GPT-6 Luna (`gpt-6-luna`); I override it to Sol with `CODEX_FAST_FAMILY=sol` |

Details that each cost a failure to learn:

- **Resolve by family name:** resolving tiers by catalog rank swapped frontier and standard on 2026-09-22, when the catalog listed Sol above Astra.
- **Escalate on evidence:** `--escalate` lifts a class one rung. Medium is the default; going past high needs a specific observed failure.
- **Refuse the priority service tier:** it bills 2.5x credits for 2x speed on Astra. The wrapper pins `service_tier="default"` and exits with an error on `--priority`. Answers are pinned to low verbosity.
- **`--ignore-user-config` is the cold-start fix:** TOML table overrides merge, so `-c 'mcp_servers={}'` changes nothing, and `codex mcp list -c 'mcp_servers={}'` still lists every server (found 2026-07-15). `--ignore-user-config` drops all MCP servers and plugins while auth survives through `CODEX_HOME`. The side effect: Codex workers see only the prompt plus files under `-C`, so the prompt must carry full context.
- **Block the raw path:** a PreToolUse hook (`codex-guard.sh`) denies any Bash call to `codex exec` that bypasses the wrapper. The escape hatch is a `CODEX_RAW=1` prefix plus a stated reason.

### Quota-aware routing

A `UserPromptSubmit` hook injects one line per turn, and the wrapper prints a fresh one after every Codex call:

```
QUOTA claude 5h 44%/4h00m 7d 57%/15h50m H=0.70 TIGHT | codex 33%/3d22h H=1.19 OPEN age=0m | prefer=codex
```

- **`H` is a pace ratio:** remaining quota divided by the remaining fraction of the window, capped at 3. Below 1 means running hot. A platform's `H` is its minimum across buckets, so Claude's usually comes from the 5-hour bucket.
- **States:** `OPEN`; `TIGHT` (under 25% left in some bucket, or `H` below 0.7); `CLOSED` (under 10% left); `UNKNOWN`.
- **`prefer=codex`:** divertible subagent work (summarization, review, bulk edits, repo search) moves to Codex or `gemini -p`, and the remaining Claude subagents run on Sonnet or Haiku. **`prefer=claude`** keeps Codex for `review` and `hardest` only. A platform must lead by 1.25x to flip `prefer`, which stops it flapping turn to turn.
- **Pinned work ignores quota:** anything in my voice, anything needing live MCP, and anything needing the conversation's context stays on Claude, because Codex workers have none of those.
- **Blind spot:** Claude Code exposes only the `five_hour` and `seven_day` buckets, and the per-model weekly limit that `/usage` shows is invisible to the CLI, so Claude's `H` is optimistic. Past 70% on 7d, check `/usage` before a large fan-out.

### Cross-family review

Consequential checks (adversarial reviews, second opinions, research that feeds a decision) run twice with the same brief: one Claude agent and one Codex run (`review` for critique, `explore --web` for research). Where they agree, confidence goes up. Where they disagree, the disagreement is the finding, because two models from one family tend to agree on the same mistakes.

## Skills: loading, descriptions, and retirement

Skills are folders with a `SKILL.md` that load into context only when relevant. Rules load every session; skills cost almost nothing until used. That split decides where knowledge lives: org context, brand guidelines, and program detail go in skills, and only the short guardrails (naming, voice, security) stay in always-on rules.

### How skills load

A skill enters context when:

1. A slash command invokes it by name.
2. Its description matches the current task, and the model calls the `Skill` tool.
3. A plugin that ships it is enabled.

At session start the model sees each skill's name and description. That listing is the whole trigger surface, which makes the description the most important line in the skill.

### Writing descriptions that trigger correctly

- **Lead with the job in plain verbs:** "Draft, reply to, and send Outlook email as the user" beats a paragraph of background.
- **Name the trigger phrases people type:** "Use for 'hand this off' or 'write this down'."
- **Name the boundary and the sibling:** "For summaries use summarizer; not for making changes." Overlapping skills with no boundary line trigger unpredictably.
- **Keep mechanism out of the description:** scripts, flags, and paths go in the body, which loads only after the skill fires.

### Retiring a skill

Skills are discovered by directory presence under a `skills/` path, so there are three levels of off:

1. **Name-only:** `skillOverrides` in `.claude/settings.local.json` (managed from the `/skills` dialog) keeps the name in the listing but suppresses the description. It costs a few tokens and stops description-match triggering, while `/name` still works.
2. **Full disable:** move the directory out of the discovered path, for example into `.claude/_disabled-skills/`. It vanishes from the listing; moving it back restores it.
3. **Delete:** only when it is dead. Git keeps the history.

Whole plugins toggle through `enabledPlugins` in `settings.json`. In a 2026-07-12 audit I cut 9 project skills, and on 2026-09-15 I merged two overlapping writing-style skills into one skill plus a linter script.

## Subagents: model pinning, return formats, and nesting

### When to spawn one

Spawn when the task would flood the main context with intermediate noise:

- 3 or more web searches.
- Content over about 500 words that needs research first.
- 3 or more integration API calls.
- A genuinely independent subtask where only the result matters.

Handle it directly for 1 or 2 lookups, edits to existing content, and anything with several decision points for the user. The saving is compression: a researcher that runs 5 searches and returns a 300-word brief costs the parent 300 words.

**Never split research from writing:** the agent that gathers the material also produces the output. Handing research notes to a separate writer is a telephone game, and the nuance dies in the handoff.

### Model tiers

Route by the judgment the task needs. The `model:` and `effort:` frontmatter fields pin each custom agent.

- **haiku (Haiku 4.5):** high-token, low-judgment work where any competent reader would produce the same output and the decision rule fits in the prompt as an explicit threshold. Briefs must be mechanical: exact calls, numeric thresholds, a hard call budget, raw values next to computed ones, and an instruction to return results through `SendMessage` (Haiku otherwise ends its turn silently).
- **sonnet (Sonnet 5.5):** the default for delegation. Review, search, docs lookup, summarization, classification, multi-step retrieval.
- **opus (Opus 5.5):** research that feeds a decision, docs, tests, critiques, red-teaming, architecture calls, high-stakes prose. Opus 5.5 at low effort now does most judgment work, because effort is a stronger cost lever than tier.

### The built-in-agent trap

Custom agents (in `~/.claude/agents/`, a project's `.claude/agents/`, or a plugin) pin their model in frontmatter. The built-in types (`general-purpose`, `Explore`, `Plan`, `claude-code-guide`, and `fork`) carry no pin and silently inherit the orchestrator's model. Dispatch `Explore` to grep for a symbol from an Opus session and the grep runs on Opus. Always pass `model` explicitly on a built-in dispatch. My CLAUDE.md states this rule, but a rule is compliance; it holds only when the orchestrator remembers.

### Agent anatomy

```yaml
---
name: researcher
description: Gather source-attributed findings across web and local files. Use for research needing three or more searches.
model: opus
effort: low
tools: Read, Grep, Glob, WebSearch, WebFetch
---

Role and context paragraph.

## Process
1. Step one
2. Step two

## Output format
The exact structure to return.

## Rules
- Constraint one
```

Design choices:

- **Tool restrictions:** each agent gets only what it needs. The bug investigator has no edit tools; the reviewer is read-only.
- **Return format baked in:** the worker's final message is re-read by the parent on every later turn, so it must be tight. Ask for evidence, sources, and reasoning in a short brief, and forbid raw dumps. A subagent sees none of the parent conversation, so the brief must carry full context.
- **Rules section:** task guardrails, such as "do NOT edit files" for an investigator or banned-word lists for a content writer.

My shared set is 13 agents: researcher, fact-verifier, red-teamer (opus / high), meeting-prep, content-reviewer, test-writer, doc-writer, bug-investigator (opus / medium), summarizer and a task tracker manager (sonnet / low), and bulk-worker, pr-preparer, dependency-auditor (haiku / low). The unmarked ones run opus / low.

### Nesting and parallelism

- **Nesting limit:** subagents can spawn their own subagents, up to 5 levels below the main session. Use it to keep intermediate output out of the parent: a researcher fans out per-source readers, a reviewer dispatches one verifier per finding. One level covers most work.
- **Run independent work in one message:** several Agent calls in a single message run concurrently.
- **Build nothing on partial results:** when agents are out for research or review, the deliverable waits until every one has reported. I learned this by building one deck three times, once per late-arriving result.
- **Known failure:** an agent told it may delegate sometimes nests an Agent call and returns empty. Tell research agents to research directly, and resume them if they come back blank.

## Rules, skills, and hooks as enforcement layers

Four layers, from soft to hard:

1. **Rules** — `CLAUDE.md` and `.claude/rules/*.md`, always in context. Hold the policy in a sentence.
2. **Skills** — loaded on demand. Hold the mechanism: long procedures, scripts, reference tables.
3. **Definitions** — agent frontmatter and Codex TOML roles. Pin model and effort mechanically.
4. **Hooks** — shell scripts on lifecycle events. Block the bypass path outright.

A model reading a rule is compliance; a script owning the behavior is determinism. Anything that has failed twice under a rule gets a hook.

Hooks I run:

| Event | Script | What it does |
|---|---|---|
| SessionStart | `mcp-env.sh` then `mcp-profile.sh` | Unsets `ANTHROPIC_API_KEY` and `ANTHROPIC_AUTH_TOKEN`, then applies the MCP profile |
| SessionStart, UserPromptSubmit | `quota-probe.sh --line` | Prints the QUOTA line; refreshes a stale Codex reading in the background without a model call |
| PreToolUse | `block-openai-mcp.py` | Denies any `mcp__openai__*` call |
| PreToolUse | `codex-guard.sh` | Denies raw `codex exec`; points to the wrapper |
| PreToolUse | `git-push-guard.py` | Blocks `git push <remote> <branch>` when HEAD is on a different branch |
| PreToolUse | `git-attribution-guard.py` | Blocks commits or PRs carrying Claude attribution or a foreign author |
| PreToolUse | `precise-read.py` | Blocks an unscoped Read of a text file over 64 KB and offers grep-then-range, a summarizer agent, or successive ranges |
| PreToolUse | writing-style `lint.py` | Lints any Bash command that copies text to the clipboard |
| PostToolUse | writing-style `lint.py --hook` | Lints written prose for banned words and em dashes |

Notes on three of them:

- **The API-key unset matters:** a stray `ANTHROPIC_API_KEY` in the environment makes Claude Code bill the API instead of the subscription, silently.
- **The push guard exists because of a 12-day miss:** in August 2026 a push run from the wrong branch reported success while the target branch stayed unpublished for 12 days.
- **The clipboard lint scans the whole command:** a heredoc that merely mentions the clipboard tool gets linted too, which is a false positive I accept.

**CLAUDE.md holds the rule; the skill holds the mechanism:** a 2026-08-03 audit of my global CLAUDE.md cut about a quarter of it with no change in behavior, mostly mechanism detail duplicated from skills, dated provenance notes, and history about retired systems. Provenance belongs in git.

## Permissions, secrets, and the security model

### Auto mode plus an allowlist

I moved off blanket `bypassPermissions` to `defaultMode: auto` with a curated allowlist of about 97 tool patterns (git, gh, npm, and the read tools of my MCP servers). Anything on `allow` runs without a prompt, anything on `deny` is blocked, and the auto-mode classifier decides the rest. Day-to-day friction matches bypass, and a novel command still gets a decision.

```json
{
  "permissions": {
    "defaultMode": "auto",
    "allow": ["Bash(git status*)", "Bash(gh pr view*)", "mcp__ms365__list-mail-messages", "..."],
    "deny": ["mcp__claude_ai_<Connector>__<write_tool>", "..."]
  }
}
```

- **Deny duplicates of a better path:** my local `deny` list holds a hosted connector's file-search and write tools, because a local server handles that system with fewer limits.
- **Destructive commands are gated three ways:** a CLAUDE.md rule (state what the command does and what could be lost, then wait for approval), the auto-mode classifier, and the git hooks.
- **`settings.local.json`:** a smaller machine-local allowlist that does not sync.
- **Untrusted repos:** a `.claude/` directory in a cloned repo can carry permissions and hooks. For team deployments use the standard permission modes and review `.claude/` before trusting a repo.

### Remote sessions get their own layer

`~/.claude/remote-settings.json` applies to sessions driven from web or mobile:

```json
{
  "channelsEnabled": true,
  "permissions": {
    "defaultMode": "auto",
    "ask": ["Bash(git push --force *)", "Bash(git push -f *)", "Bash(git push --mirror*)"],
    "deny": ["Bash(rm -rf /)", "Bash(rm *~/.ssh*)", "Bash(rm *.git/*)", "Bash(sudo rm *)",
             "Bash(chmod *777*)", "Read(./.env)", "Read(~/.ssh/**)", "..."]
  }
}
```

About 27 hard blocks: `rm` against `/`, `~`, `~/.ssh`, `~/.config`, `~/.claude`, `~/Library`, cloud-storage folders, and `.git`; `sudo rm` and `sudo chmod`; world-writable `chmod`; and reads of `.env`, credentials, `~/.ssh`, and `~/.aws`. A session steered from a phone carries guardrails that do not depend on me reading every step.

### Secrets live in a password manager

No `.env` files. Secrets live in a password manager that mounts them as an in-memory named pipe (no plaintext on disk). The shell loads them from a login-keychain cache of that mount, and a background job re-syncs the cache at login and every 30 minutes, with a command for an immediate sync. MCP servers read their keys from the shell environment. Rotating a key means editing it in the password manager's GUI; nothing in any repo changes.

Safety rules in my global CLAUDE.md:

1. **Confirm before external actions:** sending mail, posting to chat, deleting or overwriting files, publishing, and calendar changes. State exactly what will happen and wait for approval of that specific action.
2. **Secrets:** never hardcode, echo, print, or commit a credential. If one appears in a conversation or file, flag it and recommend rotation.
3. **Destructive commands:** state what the command does and what data could be lost, then wait for approval.
4. **Safe alternatives first:** `git stash` over `git reset --hard`, move to trash over `rm -rf`.

## Statusline for rate limits and context

A custom script replaces the default statusline. It is the most-glanced-at part of the setup:

```
~/assistant 🔀main ✎28 | Opus 5.5 medium | 🧠 222k | 5h 🟢 53% · 1h47m | 7d 🟢 6% · 6d17h
💬 audit the best-practices doc, it's outdated…
```

- **Directory and git:** `$HOME` shown as `~`; branch truncated at 24 characters; uncommitted-file count; ahead and behind counts, each shown only when nonzero.
- **Model and effort merged:** effort is color-coded (low gray, medium blue, high green, xhigh orange, max red).
- **Context:** tokens in use, in thousands.
- **Rate limits:** real 5-hour and weekly usage with time to reset. Gauge: 🟢 under 70%, 🟡 70 to 90%, 🔴 90% and up.
- **Line 2:** the start of the last message I typed, dimmed, so each block in the scrollback is labeled by its task.

**The numbers come straight from stdin:** Claude Code 2.1.x passes `rate_limits.five_hour` and `rate_limits.seven_day` (each with `used_percentage` and `resets_at` in Unix epoch seconds) plus `context_window` token counts. These are the numbers `/usage` shows, so the script needs no cost model or calibration. The earlier version estimated usage with `ccusage` and hand-tuned divisors that drifted.

**Staleness:** `rate_limits` refreshes only on an API response, so an idle session's snapshot freezes. When a window lapses during idle time the script prints `now` instead of a misleading `0m`.

**The remote-control badge is core UI:** the `/rc` indicator is drawn by Claude Code, and its state is absent from the statusline's stdin, so the script cannot recolor or hide it. When it wrapped onto a third line, the fix was compressing line 1: merge model and effort, drop the context percentage, strip "(1M context)" from the model name, and truncate long branch names.

## Remote control, notifications, and agent teams

### Remote control and push notifications

`remoteControlAtStartup: true` arms remote control on every session start, so a session begun at the desk can be steered from claude.ai web or the mobile app later. `agentPushNotifEnabled: true` sends a push when a long-running or background agent finishes or needs input. Together: start a long task, walk away, get pinged when it finishes or blocks, and resolve it from the phone. The leash is useful only when armed automatically; a manual toggle never gets flipped. I keep `awaySummaryEnabled: false` because the push and the statusline already cover what the away digest would say.

### Agent teams and terminal orchestration

Three layers let Claude Code workers collaborate, and they differ in whether workers can talk to each other:

| Layer | What it is | Workers message each other? |
|---|---|---|
| **Subagents** (Agent tool) | Helpers spawned inside one session | No. They report only to the caller. |
| **Agent teams** (`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`) | A lead plus independent Claude instances sharing a task list and a mailbox | Yes, by name through `SendMessage`. |
| **Terminal orchestration** (cmux, tmux) | Separately launched agent processes, including Codex or Gemini | Yes, through terminal primitives such as `send` and `read-screen`. |

Agent teams, as of 2.1.286:

- **Spawn by describing the team:** "Spawn three teammates to review PR #142: one on security, one on performance, one on test coverage. Have them challenge each other's findings." There has been no setup step since 2.1.178.
- **Steer:** in-process mode lists teammates in a panel below the prompt; select one, press Enter to open its transcript and type to it. `x` stops it; Ctrl+T toggles the task list.
- **Models:** teammates inherit the lead's effort level but not its `/model`. Say "use Sonnet for each teammate" or set the default teammate model in `/config`.
- **Guardrails:** require plan approval before teammates edit, and enforce quality with the `TeammateIdle`, `TaskCreated`, and `TaskCompleted` hooks.
- **Cost:** each teammate is a full Claude instance, so tokens scale linearly with team size. Start with 3 to 5, for research, parallel review, or debugging competing hypotheses. Sequential or same-file work belongs in one session.
- **Limits:** one team per session; teammates cannot spawn teammates; `/resume` and `/rewind` do not restore in-process teammates.
- **Display:** `teammateMode` is `in-process` by default (since 2.1.179), `auto` for split panes inside tmux or iTerm2, or `tmux` / `iterm2` to force a backend. Split panes are unsupported in Ghostty, VS Code's terminal, and Windows Terminal. The experimental flag name has churned, so check the current docs before relying on it.

I run [cmux](https://github.com/manaflow-ai/cmux), a Ghostty-based macOS terminal for parallel agents. Because Ghostty lacks split-pane support, `cmux claude-teams` installs a shim `tmux` binary, sets `TMUX` and `TMUX_PANE`, and execs `claude --teammate-mode auto`, so teammates render as cmux panes while Claude Code believes it is talking to tmux. For mixed-vendor swarms, cmux's socket CLI lets agents coordinate directly: `cmux send --surface <id> "message"` types into a peer's prompt, `cmux read-screen --surface <id>` watches a peer without interrupting it, and `cmux hooks setup` wires permission requests and notifications into the cmux UI. On real tmux, add `set -g focus-events on` to `~/.tmux.conf` so Claude Code knows which pane has focus.

## Memory and session continuity

### Auto-memory

Claude Code's auto-memory stores facts per project under `~/.claude/projects/<project-slug>/memory/`:

- **`MEMORY.md`:** an index loaded every session, kept under 200 lines.
- **One file per fact:** YAML frontmatter (name, description, type) plus the fact.
- **Four types:** user (role, preferences), feedback (corrections and confirmed approaches), project (ongoing work and decisions), reference (pointers to external resources).
- **Exclusions:** code patterns (read the code), git history (use `git log`), fixes (they are in the code), and anything already in a CLAUDE.md.
- **Point in time:** memories go stale, so verify against current state before acting on one.

My index covers 105 memory files as of 2026-09-30, and about 45 of them are feedback rules. `autoDreamEnabled` consolidates them between sessions.

### Why I retired end-of-session rituals

I used to run `/start`, `/end`, and `/update` commands that wrote state files. I retired them on 2026-07-09. A manual `/end` fails exactly when a session dies uncleanly (compaction, a rate limit, an accidental close), and that is when a record matters most. Continuity now runs on auto-memory, a task tracker for active work, and a write-first decision log.

### HANDOFF.md: an append-only log written before the work

For work that spans sessions, a `handoff` skill maintains `HANDOFF.md` at the project root. Entries are written **before** the work:

- Before a decision that constrains later work.
- Before a destructive or hard-to-reverse operation.
- Before a long or expensive step.
- When a fact took real effort to establish.
- When an approach fails. A recorded dead end is worth as much as a recorded decision.

Structure: a rewritten top block (Goal, Current state, Next steps, Open questions) over an append-only log. Each entry carries **Decision / Why / Evidence / Reversible**, with evidence labeled observed, reported, or assumed. Failures get **Tried / Result / Do not retry unless**. Entries are never edited or pruned, including wrong ones: a wrong decision followed by its correction is the record that stops the mistake repeating. Anything derivable from the code, the git log, or CLAUDE.md stays out.

## Verification and adversarial review

The highest-impact practice here, and the one Anthropic's Claude Code team recommends adopting if you adopt nothing else: **give the model a way to observe the result, then make it use that way.** If Claude can close the feedback loop, it iterates until the output is right. If it cannot, it guesses and reports the guess with the same confidence.

The rule in my global CLAUDE.md:

> Give yourself a way to observe the result, then use it. If a check exists (test suite, bash command, browser, simulator, log, live app), run it and report what you actually saw. A claim that something works is not evidence that it does.
>
> This applies to delegated work too. A subagent or a Codex run reporting "tests pass" is a claim. Re-run the check yourself before acting on it, and say which you did: observed or reported.

The observed-versus-reported label is the operative part. Collapsing the two is how a bad assumption survives three sessions.

| Work | How to observe it |
|---|---|
| Frontend | Claude in Chrome (`/chrome`) or Playwright MCP. Would an engineer build a website well without a browser? |
| Code | Test suite, `py_compile`, type-check, a timed dry run, diff inspection |
| Hardware and GUI automation | A physical check on the device. Files cannot prove a button works, so name what still needs a human press. |
| Documents and decks | Render the PDF, open the file, take a screenshot |
| Data and numbers | Re-query the source; report raw values next to computed ones |

### Adversarial review in three tiers

Self-review fails because a model that just wrote something defends its own choices. The fix is a fresh context with an explicitly adversarial framing, ideally on a different model family. Cheapest first:

1. **`red-teamer` agent** (Opus 5.5, effort high): one critic for plans, architecture, and decisions, run before code exists, when flaws are cheap.
2. **`war-council` skill:** named personas critique in parallel, and the orchestrator reconciles their findings. For consequential calls on spend, hiring, positioning, or program direction.
3. **Cross-model review** (`codex-run.sh review`): GPT-6 Astra reviewing Claude's diff, told to refute it. For consequential calls, run tiers 1 and 3 in parallel on the same brief.

**Make the critic terminate:** "Find the problems" is unbounded: a critic will always find some, including invented ones, and never signals completion. Since 2026-08-03 both `red-teamer` and `war-council` instead state what must be true for the plan to be correct (facts that must hold, people who must act, dependencies that must exist, numbers that must land in a range) and mark each condition supported, unsupported, or unknown. That is a bounded checklist with a stopping rule, and a condition nobody could challenge counts as a green light.

Prompts worth reusing:

- "Grill me on these changes and don't make a PR until I pass your test."
- "Prove to me this works." It diffs behavior between `main` and the branch instead of asserting success.
- "Knowing everything you know now, scrap this and implement the elegant solution."

## Lessons learned

### Architecture

1. **Context is the scarce resource.** Adaptive profiles, on-demand skills, subagents, and CLI offloading all exist to protect the context window. Large tool results, exploratory research, and whole-file reads are the biggest consumers.
2. **Split instructions into small rules files.** One large CLAUDE.md becomes hard to maintain; five focused files under `.claude/rules/` each stay reviewable.
3. **Domain knowledge belongs in skills.** Rules load every session and skills load on demand, so org context, brand guidelines, and program detail go in skills; naming, voice, and security stay in rules.
4. **Org deployment needs the two-branch context model.** Shipping internal paths and channel IDs to people who cannot open them wastes their context and misleads the model.

### Multi-model

5. **Several model families beat one.** Gemini brings Google Search grounding and a 1M-token window for large-file synthesis; GPT through Codex catches different bugs in review; Claude has the deepest tool integration.
6. **CLI offloading is context economics.** When Gemini reads 15 files and returns 200 words, Claude ingests 200 words. Model quality is a separate question.
7. **Subscriptions are rate-limited.** Codex and `gemini -p` cost nothing per call, but every platform has windows that bind, Claude's 5-hour bucket most often. The QUOTA line turns that into a routing signal.

### Workflow

8. **Never split research from writing.** A separate writer working from a researcher's notes loses the nuance the researcher saw.
9. **Spawn subagents to compress.** Protecting the main context from noise is the reason; speed is a side benefit.
10. **Continuity must survive an unclean exit.** Auto-memory, a task tracker, and `HANDOFF.md` replaced an `/end`-plus-state-files design on 2026-07-09, because that design lost everything whenever a session died before `/end`.

### MCP and integration

11. **Use the cheapest search that fits.** Native WebSearch and WebFetch for single lookups, a bulk search MCP for breadth and non-US queries, Jina Reader as the default page fetch, and credit-metered tools (Firecrawl at 1 credit per page, Perplexity per query) only for JS rendering, anti-bot, or synthesis.
12. **Some sites block every generic fetcher.** As of 2026-07-01, Reddit returned a 403 challenge to Firecrawl, Jina, WebFetch, and `curl` of the `.json` endpoints from my machine, and Claude's WebSearch excludes reddit.com. A dedicated scraper (an Apify actor) or scripted browser works. LinkedIn needs a dedicated MCP for the same reason.
13. **Pin MCP versions when latest breaks.** A Slack MCP release (v1.2.3) required a `users:read` scope my token lacked; pinning v1.1.28 fixed it. Read the changelog before upgrading.
14. **Custom MCPs are easy to build and easy to overpay for.** My Dynalist and Grok servers are thin wrappers over existing APIs. My GPT wrapper had the same shape and billed per token while the same models sat on a flat subscription.

### Subagent economics

15. **Pin the model on every subagent.** Sonnet follows structured prompts (output formats, brand rules, tool preferences) as reliably as Opus. Built-in agent types inherit the orchestrator's model, so pass `model` on every built-in dispatch.
16. **Design for context economy.** A researcher that runs 5 searches and returns 300 words saves the parent the 5 result pages; that is the whole economic case.
17. **A haiku tier pays off when the brief is mechanical.** Bulk edits, PR descriptions, and dependency scans run fine on Haiku 4.5 with explicit thresholds and a call budget. Task tracker CRUD moved back up to Sonnet.

### Scraping and data access

18. **Server-side and local IPs get different answers.** Reddit's `.json` endpoints still returned full comment trees to server-side workers on other IPs in July 2026 while 403-ing my laptop, so the same code can pass in CI and fail at the desk.
19. **Know each scraper's limits.** Firecrawl handles Cloudflare and JS rendering and is blocked on Reddit and LinkedIn; route those to dedicated tools up front.
20. **A database makes a good staging buffer.** When an automation platform hands data to Claude, a database both sides can reach by API (I use a Notion database) gives a schema and a status field (pending, triaged, drafted, published) that track the pipeline.

### Security

21. **Auto mode plus an allowlist beats blanket bypass.** Same low friction for common patterns, and novel commands still get a decision. Team deployments should use standard modes, since a `.claude/` folder in an untrusted repo can carry permissions.
22. **Confirm-before-send stays on in every mode.** A CLAUDE.md rule requires approval before any email, chat post, task change, or publish, and `remote-settings.json` adds hard filesystem and secrets blocks for phone-driven sessions.

### Operations and visibility

23. **A rate-limit statusline prevents budget surprises.** Reading the real 5h and 7d percentages from stdin replaced a cost proxy that needed recalibration. When 7d goes yellow, ease off high effort and parallel agents.
24. **Remote control must be armed automatically.** With `remoteControlAtStartup: true` every session is reachable from the phone from its first prompt.

### Claude Code versus Claude Desktop

25. **Never run one MCP server twice.** A manual filesystem server (logged as `[filesystem]`, healthy) beside the bundled directory connector (logged as `[Filesystem]`, "process exiting early") left one instance crash-looping. Removing the manual entry fixed it.
26. **`-32601 Method not found` on `resources/list` or `prompts/list` is normal.** A tools-only server declares `capabilities: { tools: {} }` and rejects that polling correctly. The real failure signatures are `transport closed unexpectedly` and `process exiting early`.
27. **Code and Desktop need different MCPs.** Claude Code has native file tools and Bash, so a filesystem MCP is redundant and `osascript` runs through the shell. Filesystem and macOS-control MCPs earn their place only in Claude Desktop.
28. **Pick one macOS-control tier and mind its privileges.** A single-tool AppleScript server needs only per-app Automation permission. A GUI-driving server with accessibility clicking, screenshots, and a shell runs unsandboxed with full Accessibility access, so scope it in its config and weigh it against any sensitive data synced to the machine.

### Effort, verification, and community advice

29. **Effort is a dial.** I ran `xhigh` for months on the theory that reasoning was free on a subscription. Subscription reasoning is rate-limited, and on Opus 5 high effort made ordinary work worse. `medium` is the global default, raised per task.
30. **Instructions written for an older model can hurt a newer one.** Guardrails that stopped earlier failure modes read to Opus 5 as constraints to satisfy, and satisfying them costs turns. Cutting a quarter of my global CLAUDE.md on 2026-08-03 changed no behavior.
31. **"Find the problems" never terminates.** "State what must be true, then check those things" turns an open-ended hunt into a checklist with a stopping rule.
32. **Handoffs are claims.** Re-run the check a subagent or Codex run reports and label the result observed or reported. It costs one command and catches confident summaries of code that was never written.
33. **Write the decision log before the work.** A summary written at the end is lost when the session has no end.
34. **Audit community advice against what you already run.** Of ten well-regarded recommendations I reviewed on 2026-08-03, four were already in place, three were counterproductive for this setup, and three were worth adopting. The most-cited one, a plugin for Claude orchestrating Codex, burned 9M Claude tokens plus 1.2M Codex tokens on one feature in its own author's write-up. Check the numbers in a post before adopting its pattern, and distrust benchmark tables with no cited source.

## License

Text © 2026 Alec Foster, licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). See [LICENSE](LICENSE).
