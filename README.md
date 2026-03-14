# ForgeAI — Agentic Coding Skill for Claude Code

ForgeAI is a `/forge` skill for Claude Code CLI that acts as a **god-agent orchestrator**. Given a task description, it decomposes the work into a dependency-aware blueprint, spawns parallel coder and watcher subagents per function, runs free deterministic checks (tsc + eslint + unit tests) between each pair, and merges approved code into your project.

**~85–90% fewer tokens** than a naive single-context session. **Parallel execution** across independent functions. **Zero new infrastructure** — it runs entirely on Claude Code's existing Agent tool, worktree isolation, and model selection.

---

## How It Works

```
/forge "build a REST API with login, signup, and profile endpoints"
         │
         ▼
  God-Agent (Opus) scans codebase → generates forge-blueprint.md
         │  (you approve)
         ▼
  For each batch of independent functions (run in parallel):
  ├── Coder agent (Haiku/Sonnet/Opus by complexity)
  │     isolation: worktree — writes implementation + unit test
  │
  ├── Deterministic checks — FREE, no LLM:
  │     tsc --noEmit  →  eslint  →  vitest run
  │     FAIL → retry coder once → FAIL again → escalate
  │
  └── Watcher agent (Sonnet)
        reviews logic, edge cases, type safety, security
        PASS → merge into project
        FAIL → retry coder once → FAIL again → escalate
         │
         ▼
  Final report: N done, N skipped, N escalated
```

---

## Installation

### Requirements

- **Claude Code CLI** — install from [claude.ai/code](https://claude.ai/code) or `npm install -g @anthropic-ai/claude-code`
- A project with TypeScript (other stacks work but conventions are TS-first)

### Step 1 — Copy the skill files

```bash
# Create the skills and rules directories if they don't exist
mkdir -p ~/.claude/skills/forge
mkdir -p ~/.claude/rules

# Clone or download this repo, then copy:
cp forge/SKILL.md          ~/.claude/skills/forge/SKILL.md
cp forge/forge-blueprint.md ~/.claude/skills/forge/forge-blueprint.md
cp forge/forge-coder.md    ~/.claude/skills/forge/forge-coder.md
cp forge/forge-watcher.md  ~/.claude/skills/forge/forge-watcher.md
cp forge/forge.config.yaml ~/.claude/skills/forge/forge.config.yaml

# Install the conventions rule (auto-injected into all coder prompts)
cp rules/forge-conventions.md ~/.claude/rules/forge-conventions.md
```

### Step 2 — Verify installation

Start Claude Code in any project directory:

```bash
cd your-project
claude
```

Type `/` and look for `forge` in the skill list. You should see:

```
forge  ForgeAI agentic coder — decomposes a task into a dependency DAG...
```

### Step 3 — First run in a project

On the first `/forge` invocation in any project, ForgeAI automatically copies `forge.config.yaml` to the project root. Review and edit it if you need to override models or checks.

---

## File Structure

```
~/.claude/
├── skills/
│   └── forge/
│       ├── SKILL.md            ← main skill (the god-agent) — invoked by /forge
│       ├── forge-blueprint.md  ← blueprint YAML template + generation rules
│       ├── forge-coder.md      ← coder subagent prompt (function-slice-only context)
│       ├── forge-watcher.md    ← watcher subagent prompt (PASS/FAIL structured output)
│       └── forge.config.yaml   ← default config template (copied to project root)
└── rules/
    └── forge-conventions.md    ← stack-aware conventions injected into every coder prompt
```

> **Why a directory instead of a single file?**
> Claude Code loads skills from `~/.claude/skills/{name}/SKILL.md`. The other files in the directory are internal templates that the god-agent reads at runtime — they are not themselves invocable skills.

---

## Configuration

`forge.config.yaml` is copied to your **project root** on first run.

### Understanding the model tier system

**Important constraint**: Claude Code's Agent tool only accepts three `model` values — `haiku`, `sonnet`, `opus`. You cannot write `gpt-4o` or an OpenRouter model ID directly in `forge.config.yaml`. Instead, you configure the *backend* that resolves those tiers (see Provider Setup below).

```yaml
models:
  god_agent:    opus    # blueprint generation + orchestration
  coder_simple: haiku   # pure functions, <20 lines, no I/O
  coder_medium: sonnet  # DB calls, validation, 20–60 lines
  coder_complex: opus   # auth, pipelines, security-critical
  watcher:      sonnet  # logic review — all functions

limits:
  max_parallel_agents: 8       # lower if hitting rate limits
  max_retries_per_function: 1
  token_budget: 100000

checks:
  typecheck: tsc --noEmit
  lint: npx eslint --ext .ts,.tsx
  test: npx vitest run
  run_typecheck: true
  run_lint: true
  run_test: true
```

### Complexity → Tier mapping

| Complexity | Tier | Anthropic default | Cost |
|---|---|---|---|
| `simple` | `haiku` | claude-haiku-4-5 | Lowest |
| `medium` | `sonnet` | claude-sonnet-4-6 | Mid |
| `complex` | `opus` | claude-opus-4-6 | Highest |

The god-agent assigns complexity conservatively (when uncertain, goes one tier higher).

---

## Provider Setup

The tier names (`haiku`, `sonnet`, `opus`) route to actual models via **environment variables** set in your shell before starting Claude Code. The `forge.config.yaml` model fields stay the same regardless of provider.

### TL;DR — Just use OpenRouter (2 env vars, done)

OpenRouter is the easiest way to use any model (Claude, GPT-4o, Gemini, Llama, etc.) without any proxy or extra tooling:

```bash
# Add to ~/.zshrc or ~/.bashrc
export ANTHROPIC_BASE_URL=https://openrouter.ai/api/v1
export ANTHROPIC_API_KEY=sk-or-...   # your OpenRouter key (openrouter.ai)
```

That's it. `haiku/sonnet/opus` tiers route through OpenRouter, which defaults to equivalent Claude models. To pin specific non-Claude models per tier:

```bash
export FORGE_MODEL_HAIKU=google/gemini-2.5-flash     # cheapest/fastest
export FORGE_MODEL_SONNET=openai/gpt-4o
export FORGE_MODEL_OPUS=openai/gpt-4o
export FORGE_MODEL_GOD=anthropic/claude-opus-4-6     # god-agent (keeps Opus)
```

OpenRouter model IDs use `provider/model-name` format — browse the full list at openrouter.ai/models. Top up with a few dollars and you have access to every major model from one key.

> Everything below (LiteLLM, Ollama) is only needed for offline/air-gapped setups or if you specifically want to avoid cloud providers entirely.

---

### Anthropic (default — nothing extra needed)

```bash
export ANTHROPIC_API_KEY=sk-ant-...
# Start claude — tiers resolve to claude-haiku/sonnet/opus automatically
```

### OpenRouter (200+ models — Claude, GPT-4o, Gemini, Llama, etc.)

OpenRouter exposes a Claude-compatible API, so Claude Code talks to it directly:

```bash
export ANTHROPIC_BASE_URL=https://openrouter.ai/api/v1
export ANTHROPIC_API_KEY=sk-or-...    # your OpenRouter key (not your Anthropic key)
```

With this set, `haiku/sonnet/opus` tiers resolve to their Claude equivalents served via OpenRouter. To pin **specific non-Claude models** per tier, set these in your shell (ForgeAI reads them when building Agent prompts):

```bash
# Route haiku tier → Gemini Flash (cheapest, fastest)
export FORGE_MODEL_HAIKU=google/gemini-2.5-flash

# Route sonnet tier → GPT-4o
export FORGE_MODEL_SONNET=openai/gpt-4o

# Route opus tier → GPT-4o (or claude-opus-4-6 via OpenRouter)
export FORGE_MODEL_OPUS=openai/gpt-4o

# God-agent specifically
export FORGE_MODEL_GOD=anthropic/claude-opus-4-6
```

> OpenRouter model IDs use `provider/model-name` format. Browse at openrouter.ai/models.

**Example: cheapest possible ForgeAI run**
```bash
export ANTHROPIC_BASE_URL=https://openrouter.ai/api/v1
export ANTHROPIC_API_KEY=sk-or-...
export FORGE_MODEL_HAIKU=google/gemini-2.5-flash
export FORGE_MODEL_SONNET=google/gemini-2.5-flash
export FORGE_MODEL_OPUS=google/gemini-2.5-pro
export FORGE_MODEL_GOD=google/gemini-2.5-pro
```

**Example: OpenAI-only run**
```bash
export ANTHROPIC_BASE_URL=https://openrouter.ai/api/v1
export ANTHROPIC_API_KEY=sk-or-...
export FORGE_MODEL_HAIKU=openai/gpt-4o-mini
export FORGE_MODEL_SONNET=openai/gpt-4o
export FORGE_MODEL_OPUS=openai/gpt-4o
export FORGE_MODEL_GOD=openai/gpt-4o
```

### OpenAI directly (via LiteLLM proxy)

OpenAI's API is not Anthropic-compatible natively, so you need a local proxy:

```bash
# Install LiteLLM
pip install litellm[proxy]

# Start proxy — translates Anthropic API calls → OpenAI API calls
litellm --model gpt-4o --port 4000
```

Then point Claude Code at it:

```bash
export ANTHROPIC_BASE_URL=http://localhost:4000
export ANTHROPIC_API_KEY=sk-...          # your OpenAI key
export FORGE_MODEL_HAIKU=gpt-4o-mini
export FORGE_MODEL_SONNET=gpt-4o
export FORGE_MODEL_OPUS=gpt-4o
export FORGE_MODEL_GOD=gpt-4o
```

For mixed model routing (different models per tier):
```bash
litellm --config litellm.config.yaml --port 4000
```

```yaml
# litellm.config.yaml
model_list:
  - model_name: haiku
    litellm_params:
      model: gpt-4o-mini
      api_key: sk-...
  - model_name: sonnet
    litellm_params:
      model: gpt-4o
      api_key: sk-...
  - model_name: opus
    litellm_params:
      model: gpt-4o
      api_key: sk-...
```

### Local models via Ollama + LiteLLM

```bash
# Pull models
ollama pull qwen2.5-coder:7b
ollama pull qwen2.5-coder:32b

# Start LiteLLM proxy pointing to Ollama
litellm --model ollama/qwen2.5-coder:32b --port 4000
```

```bash
export ANTHROPIC_BASE_URL=http://localhost:4000
export ANTHROPIC_API_KEY=anything       # LiteLLM ignores this for local models
export FORGE_MODEL_HAIKU=ollama/qwen2.5-coder:7b
export FORGE_MODEL_SONNET=ollama/qwen2.5-coder:32b
export FORGE_MODEL_OPUS=ollama/qwen2.5-coder:32b
export FORGE_MODEL_GOD=ollama/qwen2.5-coder:32b
```

> Local models work for medium/simple functions but often struggle with the god-agent's blueprint synthesis. Consider keeping the god-agent on a cloud model even when running coders locally.

### Persisting provider config

Add your chosen env vars to `~/.zshrc` (or `~/.bashrc`) to make them permanent:

```bash
# ~/.zshrc — example: OpenRouter with mixed models
export ANTHROPIC_BASE_URL=https://openrouter.ai/api/v1
export ANTHROPIC_API_KEY=sk-or-...
export FORGE_MODEL_HAIKU=google/gemini-2.5-flash
export FORGE_MODEL_SONNET=openai/gpt-4o
export FORGE_MODEL_OPUS=anthropic/claude-opus-4-6
export FORGE_MODEL_GOD=anthropic/claude-opus-4-6
```

Or use a `.env` file in your project root and load it before starting Claude Code:

```bash
# .env (never commit this file)
ANTHROPIC_BASE_URL=https://openrouter.ai/api/v1
ANTHROPIC_API_KEY=sk-or-...
FORGE_MODEL_HAIKU=google/gemini-2.5-flash
FORGE_MODEL_SONNET=openai/gpt-4o
FORGE_MODEL_OPUS=anthropic/claude-opus-4-6

# Load and start
source .env && claude
```

---

## Usage

### Basic

```
/forge "build a REST API with login, signup, and profile endpoints"
```

### With a TypeScript project that has existing code

ForgeAI scans your codebase first with Glob/Grep. Existing types and utilities are reused — the blueprint will mark them `status: skipped` and exclude them from the build order.

### Approving the blueprint

After scanning, ForgeAI prints a summary and waits for you:

```
ForgeAI Blueprint — REST API
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Files to create/modify: 6
Functions to implement: 12
Execution batches:      3
Estimated tokens:       ~18,000–24,000

Batch 1 (parallel): src/utils/hash.ts, src/types/user.ts, src/types/auth.ts
Batch 2 (parallel): src/db/users.ts, src/db/sessions.ts
Batch 3 (sequential): src/auth/login.ts, src/auth/signup.ts, src/auth/profile.ts

Complexity breakdown:
  simple  (Haiku):  4 functions
  medium  (Sonnet): 6 functions
  complex (Opus):   2 functions

Type 'proceed' to start, or describe changes to the blueprint.
```

Type `proceed` to start. Or describe what you want changed: `"make login and signup complex tier"`.

### Inspecting the blueprint

The full dependency-aware blueprint is written to `forge-blueprint.md` in your project root. You can edit it directly before typing `proceed`.

### Handling escalations

If a function is escalated (failed tsc/lint/test twice, or watcher issued FAIL twice), it appears in the final report with the exact error or watcher feedback. Fix it manually or re-run `/forge` targeting just that function.

---

## Testing Against a Normal Claude Session

Use this checklist to verify ForgeAI actually improves on a standard session:

### Test 1 — Token comparison

**Baseline (normal session):**
```
# Open a new Claude Code session, no /forge
claude
> "Build a REST API with login, signup, profile — TypeScript, Supabase"
# Note the token count from the session summary
```

**ForgeAI session:**
```
claude
/forge "Build a REST API with login, signup, profile — TypeScript, Supabase"
# Note the token count from the session summary
```

Expected result: ForgeAI uses 80–90% fewer tokens. The difference is large on projects with 10+ functions because each coder agent receives only its function slice, not the full codebase.

### Test 2 — Parallel execution

Add a timing wrapper:
```bash
time claude --print '/forge "build 5 independent utility functions"'
```

vs.

```bash
time claude --print '"build 5 independent utility functions in TypeScript"'
```

Expected result: ForgeAI is faster for independent functions because they run in parallel worktrees. The baseline runs sequentially in one context.

### Test 3 — Code quality gate

Run the full test suite after both sessions:

```bash
# After normal session
npm run test && tsc --noEmit

# After ForgeAI session
npm run test && tsc --noEmit
```

Expected result: ForgeAI output passes more often on the first try because every function went through `tsc + eslint + test → watcher review` before being merged.

### Test 4 — Blueprint determinism

Run the same `/forge` prompt twice on an empty project. Compare the two `forge-blueprint.md` files:

```bash
diff forge-blueprint-run1.md forge-blueprint-run2.md
```

The dependency DAG and batch groupings should be identical. Function descriptions may vary slightly but the structure should be stable.

### Test 5 — Escalation handling

Intentionally break a function by editing `forge-blueprint.md` before proceeding — add an impossible constraint to the description. Confirm the final report lists it as `escalated` with the watcher's REQUIRED FIXES block, rather than silently failing or hanging.

---

## Gemini CLI

ForgeAI runs identically on Gemini CLI. Swap invocation and model names:

```
@forge "build a REST API with login, signup, profile"
```

In `forge.config.yaml`, replace model names:
```yaml
models:
  god_agent: gemini-2.5-pro
  coder_simple: gemini-2.5-flash
  coder_medium: gemini-2.5-flash
  coder_complex: gemini-2.5-pro
  watcher: gemini-2.5-flash
```

The skill files themselves (`SKILL.md`, `forge-coder.md`, `forge-watcher.md`, `forge-blueprint.md`) need no changes — they are model-agnostic prompt templates.

For Gemini CLI, place the files at:
```
~/.gemini/skills/forge/SKILL.md
~/.gemini/skills/forge/forge-*.md
~/.gemini/skills/forge/forge.config.yaml
~/.gemini/rules/forge-conventions.md
```

---

## Token Budget

For a 24-function project:

| Component | Tokens | Notes |
|---|---|---|
| God-agent (blueprint gen) | 5,000–10,000 | One call, structured YAML output |
| Coder per function | 200–500 | Function spec + dep interfaces + conventions only |
| Deterministic checks | 0 | tsc + eslint + tests — no LLM |
| Watcher per function | 300–800 | Spec + implementation + test only |
| 20% retry rate | +100–400 each | Error output + retry prompt |
| **Total (24 functions)** | **~15,000–25,000** | vs MetaGPT/naive approach: 200K+ |

The key discipline: **coders never see the full codebase**. They receive only their function slice, the interfaces of their direct deps, and the project conventions block.

---

## Troubleshooting

**`/forge` not showing in the skill list**
Confirm the directory structure: `~/.claude/skills/forge/SKILL.md` must exist. The file must have frontmatter with `name: forge`.

**Blueprint generation produces circular deps**
The god-agent will flag `circular_dependency: true` in `forge-blueprint.md` metadata. Restructure the task description to split the circular pair into independent interfaces.

**All functions escalated after tsc check**
Your project's `tsconfig.json` strict settings may conflict with the generated code. Edit `forge.config.yaml` to set `run_typecheck: false` temporarily, or add a `tsconfig.forge.json` with relaxed settings for generated code.

**Watcher always issues FAIL**
Check that `forge-watcher.md` is at `~/.claude/skills/forge/forge-watcher.md`. If the god-agent can't read it, it falls back to a generic prompt that may produce inconsistent verdicts.

**Context window errors on large projects**
Reduce `limits.max_parallel_agents` in `forge.config.yaml` to 4 or lower. Each parallel agent spawns a separate context, but the god-agent itself accumulates results.
