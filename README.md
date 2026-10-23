# My Claude Code loads 39.6k tokens before I type anything — here's where they go

I ran `/context all` in a fresh Claude Code session on my Mac (Claude Code 2.1.287, October 8, 2026) before sending a single prompt. The header said:

```
Tokens: 39.6k / 1m (4%)
```

So 39.6k tokens were already in context before I'd asked for anything. On a 1M-token window that's only 4%, which sounds harmless. But this is overhead that comes back every turn and every new session, and most of it was stuff I had added myself and forgotten about.

This post walks through what each bucket in that number is, how to measure your own, and what I'm trimming. One machine, one measurement — your setup will differ, and that's the point: run the command yourself.

## The breakdown

Here's what `/context all` showed, sorted by size:

| Category | Tokens | Counts toward the 39.6k? |
|---|---|---|
| Memory files (CLAUDE.md + `~/.claude/rules`) | 20.4k | yes |
| Skills | 9.9k | yes |
| System tools | 4k | yes |
| Custom agents | 3.3k | yes |
| System prompt | 1.9k | yes |
| MCP tools (deferred) | 10.4k | no |
| System tools (deferred) | 19.6k | no |
| Autocompact buffer | 33k | no (reserved) |
| Free space | 927.4k | — |

Two things surprised me.

First, **the biggest bucket wasn't skills or MCP. It was memory files: 20.4k tokens**, about 52% of the total, and more than skills and agents combined (13.2k).

Second, **33.6k of the 39.6k is under my control** (memory 20.4k + skills 9.9k + agents 3.3k). Only 5.9k is Claude Code's own baseline (system prompt 1.9k + system tools 4k).

### A note on the deferred rows

The table lists 30.0k tokens of "deferred" tools: 10.4k for MCP tools (29 tools, all from one server, chrome-devtools) and 19.6k for deferred system tools. These are **not** in the 39.6k. Deferred tools are only loaded when they're needed.

You can check this from the numbers: 39.6k used + 927.4k free + 33k autocompact buffer = 1,000k, exactly the 1M window. If the deferred 30k were counted, the header would read around 69.5k. (The five counted categories add up to 39.5k; the 0.1k gap is rounding.)

So if you remember the older complaints about MCP tool definitions eating context before you even start, check your own `/context` before blaming MCP. In my case, deferral took care of it.

## Context occupancy is not the same as billed cost

The 39.6k figure is how much of the context window is occupied, not what each turn costs. That prefix is resent with every request, but Claude Code uses Anthropic's prompt caching, so after the first request in a session, an unchanged prefix is normally read from cache instead of being processed as fresh input.

Per Anthropic's [prompt caching docs](https://docs.claude.com/en/docs/build-with-claude/prompt-caching), cache reads are billed at 0.1× the base input price on most models (0.05× on Claude Opus 5.5 and Sonnet 5.5). Writing the cache costs more than normal input: 1.25× for the default 5-minute lifetime, 2× for the 1-hour option. The cache expires after 5 minutes without use by default, and any change to the prefix (editing CLAUDE.md, adding a skill, changing tools) invalidates it.

So in a busy session the fixed overhead is far cheaper than the raw token count suggests. It still matters, though: it fills the context window, and it is paid at the higher write rate every time the cache is cold or invalidated. Cheaper is not free.

A before/after comparison from my own cleanup is coming. I'll add it here once I've re-run `/context all`.

## What each bucket actually is

**System prompt (1.9k) and system tools (4k).** Claude Code's own instructions and the tool definitions it needs on every turn. You don't control these; they're the baseline.

**Memory files (20.4k).** Everything Claude Code loads as standing instructions at launch: your user-level `~/.claude/CLAUDE.md`, the project `CLAUDE.md` files, and every rule file under `.claude/rules/` that has no `paths:` frontmatter. Per the [memory docs](https://code.claude.com/docs/en/memory), rules without `paths` load unconditionally "with the same priority as `.claude/CLAUDE.md`", and `@path` imports don't save anything because imported files also load at launch.

Mine broke down like this:

| File group | Files | Tokens |
|---|---|---|
| `~/.claude/CLAUDE.md` | 1 | 6k |
| `~/.claude/rules/common/` | 10 | 5,401 |
| `~/.claude/rules/web/` | 7 | 4,555 |
| `~/.claude/rules/zh/` | 11 | 3,049 |
| `~/.claude/rules/README.md` | 1 | 1.5k |

That's 29 rule files totalling roughly 14.5k tokens, on top of a 6k CLAUDE.md.

**Skills (9.9k).** Skill *bodies* only load when a skill is invoked. What sits in context all the time is the **skill listing**: each skill's name plus its description, so Claude knows what's available. My listing has 318 entries (306 user, 2 plugin, 10 built-in). Most are tiny: 198 of them are under 20 tokens each. The biggest were built-ins (`dataviz` at about 480, `update-config` at about 240), then a few of mine like `blueprint` (about 200) and `energy-procurement` (about 180).

One detail from the [skills docs](https://code.claude.com/docs/en/skills): the listing has a character budget that scales with 1% of the context window, and each entry's description is capped at 1,536 characters. When the listing overflows, Claude Code drops descriptions for the skills you use least. The Skills row in `/context` is the size *after* that budget is applied. So with hundreds of skills, the cost isn't only tokens — some skills may be listed by name only, and Claude has less to match your request against.

**Custom agents (3.3k).** Each subagent in `~/.claude/agents/` contributes its name and description so the main session knows when to delegate. I have 48, averaging about 70 tokens each; the largest were `opensource-sanitizer` (106), `chief-of-staff` (104) and `e2e-runner` (103). Small individually, but they add up, and unlike skills there's no budget truncation, so every description counts in full.

## How to measure your own

1. Start a fresh session in the directory you normally work in (project CLAUDE.md files and project rules depend on where you launch).
2. Before typing anything, run:

   ```
   /context all
   ```

   The plain `/context` gives the category grid; `all` expands the per-item tables for MCP tools, agents, memory files and skills.
3. Look at the header first (`Tokens: X / window`), then the Memory files table, which is where hidden weight tends to be.
4. Check which rows say "(deferred)" before drawing conclusions about MCP.

Two other built-in commands are worth knowing:

- **`/skill-doctor`** (Claude Code v2.1.252+) shows what each skill costs *and how often you actually use it*, and flags skills you've never invoked. Usage data is the thing a static count can't give you.
- **`/doctor prompt-audit`** (v2.1.283+) checks your CLAUDE.md files, rules, skills and agents for outdated or conflicting instructions and proposes edits without changing anything until you approve.

## What I'm trimming

### 1. Deduplicate the rules folders (biggest win)

Looking only at file names in my `rules/` directory:

- Six topics exist in **all three** folders (`common/`, `web/`, `zh/`): coding-style, testing, performance, patterns, hooks, security.
- Four more exist in both `common/` and `zh/`: git-workflow, code-review, development-workflow, agents.
- `zh/` mirrors every filename in `common/`.

I haven't diffed the contents yet, but that layout strongly suggests I'm loading the same guidance two or three times, partly in two languages. Plan: merge each topic into one file, keep one language, and delete the rest. The docs also warn that conflicting instructions can make Claude "pick one arbitrarily", so duplicates aren't just a token cost.

### 2. Move README files out of `rules/`

`rules/README.md` (1.5k) and `rules/zh/README.md` (778) are documentation for humans, but because every `.md` under `rules/` is discovered recursively, they load as memory in every session — about 2.3k tokens for nothing. Move them somewhere outside `rules/`.

### 3. Path-scope rules that only apply to some files

The `web/` rules (design-quality, hooks, performance and so on) only matter when Claude is touching front-end code. Add frontmatter so they load only then:

```markdown
---
paths:
  - "**/*.{ts,tsx,css,html}"
---
```

Path-scoped rules load when Claude reads or edits a matching file instead of at launch.

### 4. Shrink CLAUDE.md

6k tokens is a lot for a user-level file that applies to every project. The memory docs suggest keeping each CLAUDE.md under 200 lines, putting multi-step procedures into skills (whose bodies load on demand), and moving anything area-specific into path-scoped rules. Block-level HTML comments are stripped before injection, so notes for yourself can stay in the file at zero cost.

### 5. Move rarely used skills out, or demote them

With 300+ skills, nothing looks heavy on its own, which is exactly why they pile up. Options, from gentlest to strongest:

- In `/skills`, set low-priority entries to **name-only** (`skillOverrides`), so they stay invocable but list without a description.
- Set skills you only ever trigger by hand to `disable-model-invocation: true`, which removes the description from context entirely.
- Move skills you haven't touched in months out of `~/.claude/skills/` into an archive folder. Use `/skill-doctor`'s "never invoked" list as a starting point.

My listing also seems to contain a block of old slash commands showing up as skills (short names like `aside`, `build-fix`, `tdd`). Worth checking whether yours has duplicates between `commands/` and `skills/`.

### 6. Trim agent descriptions

An agent description only needs to tell the main session *when* to delegate. A sentence or two usually does it; anything longer belongs in the agent's body, which isn't loaded into the main session. Also delete agents for languages or stacks you no longer use — I have separate reviewers and build resolvers for C++, Go, Java, Kotlin, Rust and more.

I'll re-run `/context all` after the cleanup and post the before/after numbers rather than guessing at them now.

## Tools that help

You can do all of this with built-ins, and I'd start there:

- **`/context all`**: the authoritative count for your current session.
- **`/skill-doctor`**: per-skill cost plus usage frequency.
- **`/doctor prompt-audit`**: finds stale or conflicting instructions.

Free third-party options:

- **[ccusage](https://github.com/ryoppippi/ccusage)** reads your local Claude Code logs and shows what you've actually *spent* per day or session. Good for seeing whether a cleanup changed your real usage.
- **[ctxaudit](https://github.com/sanjeed5/ctxaudit)** and **[claude-context-lint](https://github.com/skibidiskib/claude-context-lint)** are CLIs that audit per-turn overhead (including CLAUDE.md and rules) from the terminal.

And, with a disclosure: **I'm the developer of [Vexilo Code](https://apps.apple.com/us/app/vexilo-code/id6800593544)**, a paid native macOS app I built for the skills-and-agents part of this problem. It reads `~/.claude` (read-only, on-device), estimates each skill's and agent's per-turn tokens, lists them heaviest first, and copies a cleanup prompt for Claude Code to walk through with you. Its numbers are rough estimates (characters ÷ 4), so when they disagree with `/context`, trust `/context`. It does **not** currently count CLAUDE.md or rules files — which, in my own setup, was the bigger bucket — so for that part the manual steps above are still the way. The catalog and search are free; the full app is a 7-day trial, then $19.99 one-time. Vexilo is independent and not affiliated with Anthropic.

## Takeaways

- Run `/context all` before your first prompt. The header number is your fixed cost.
- Check the **Memory files** table first. In my case CLAUDE.md plus rules was over half the total.
- Deferred MCP and system tools aren't in the starting total. Don't optimize what isn't loaded.
- Skills and agents are many small numbers. Use usage data (`/skill-doctor`) to decide what goes.
- Re-measure after every change.

---

*Measured on one Mac with Claude Code 2.1.287 on 2026-10-08. All figures come from that single `/context all` run; the raw output (with local paths removed) is summarized in the tables above. Claude and Claude Code are trademarks of Anthropic.*
