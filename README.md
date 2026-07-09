# Codex Skill for Claude Code

Give Claude Code a "second opinion" by letting it consult OpenAI's Codex CLI for independent verification.

## What It Does

When Claude Code creates a plan, Codex automatically reviews it before you approve. Two AIs checking each other's work catches more edge cases.

**Automatic review on:**
- Every plan Claude creates (via hook)
- Architecture decisions
- Implementation approaches

**Manual use for:**
- Researching unfamiliar APIs or libraries
- Verifying complex code patterns
- Getting alternative perspectives

## Prerequisites

Install [Codex CLI](https://github.com/openai/codex):

```bash
npm install -g @openai/codex
```

Configure your OpenAI API key.

## Supported Models

The 5.6 family is the current generation (ladder: `sol` > `terra` > `luna`). Reasoning-effort levels for 5.6 are `none`, `low`, `medium`, `high`, `xhigh`, `max`.

| Model | Context | Price /1M (in/out) | Best for |
|-------|---------|--------------------|----------|
| `gpt-5.6-sol` | 1.05M (128k out) | $5 / $30 | Flagship — deepest analysis, architecture, long-horizon agentic work (use with `high`/`xhigh`/`max` reasoning). Bare `gpt-5.6` aliases here |
| `gpt-5.6-terra` | 1.05M (128k out) | $2.50 / $15 | **Default** — most reviews and verification; ~GPT-5.5-flagship quality at roughly half the cost |
| `gpt-5.6-luna` | 1.05M (128k out) | $1 / $6 | Fast and affordable — quick fact checks, trivial queries |
| `gpt-5.5` | 400k | — | Prior frontier model, still selectable via `-m` |

## Installation

### Option A: Install as Claude Code plugin (recommended)

```bash
claude plugin add cathrynlavery/codex-skill
```

This auto-registers both the `/codex` skill and the automatic plan review hook. No manual configuration needed.

### Option B: Manual installation

#### 1. Install the skill

```bash
git clone https://github.com/cathrynlavery/codex-skill.git
mkdir -p ~/.claude/skills/codex
cp codex-skill/skills/codex/SKILL.md ~/.claude/skills/codex/
```

#### 2. Set up automatic plan review

Copy the hook script:

```bash
mkdir -p ~/.claude/hooks
cp codex-skill/hooks/plan-review.sh ~/.claude/hooks/
chmod +x ~/.claude/hooks/plan-review.sh
```

Add to your `~/.claude/settings.json`:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "ExitPlanMode",
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/plan-review.sh",
            "timeout": 120
          }
        ]
      }
    ]
  }
}
```

## How It Works

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Claude    │────>│ ExitPlanMode│────>│   Codex     │
│ creates plan│     │   (hook)    │     │  reviews    │
└─────────────┘     └─────────────┘     └─────────────┘
                                               │
                                               v
                                        ┌─────────────┐
                                        │ You approve │
                                        │ with context│
                                        └─────────────┘
```

The hook intercepts `ExitPlanMode` and reads the plan from `tool_response.plan` (the field where Claude Code stores the plan content). It passes the plan to Codex for review and displays the result before you approve.

Codex reviews for:
- Potential issues or risks
- Missing steps
- Better alternatives
- Edge cases not addressed

## Manual Usage

Invoke directly:

```
/codex
```

Or ask Claude:

> "Can you verify this approach with Codex?"
> "Get a second opinion on this architecture"

The skill uses `gpt-5.6-terra` by default — capable enough for most reviews and faster than the flagship. For the hardest questions (novel architecture, deep analysis), it escalates to `gpt-5.6-sol` with high reasoning effort. For trivial fact checks, it can drop to `gpt-5.6-luna`.

## Example Output

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
CODEX SECOND OPINION ON PLAN
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
LGTM - Plan covers the main implementation steps.

Minor suggestions:
- Consider adding error handling for the API timeout case
- Step 3 could be split into separate DB migration and code changes
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

## Troubleshooting

**Hook doesn't fire:** Make sure Codex CLI is installed (`codex --version`) and on your PATH. The hook exits silently on errors to avoid blocking your workflow.

**No plan content found:** The hook reads from `tool_response.plan` (primary) with fallbacks to `tool_response.filePath` and filesystem search. If you're seeing issues, check that you're using a current version of Claude Code.

## License

MIT
