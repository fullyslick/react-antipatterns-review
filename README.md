# React Anti-Patterns Review

An [Agent Skill](https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview) that reviews React / JSX / TSX code for known anti-patterns and hands you a prioritized fix plan. It is **observe-only**: it never edits your source files.

Point it at a component, a pasted snippet, a diff or a PR, and you get a file-by-file walkthrough of numbered findings (each tied to a catalog ID, with the reasoning and a small before → after snippet), a short summary, and a checklist-style fix plan ordered by priority.

## Why observe-only?

People usually want to know *what's wrong and what to do about it* before touching the code: a PR they're reviewing, a teammate's component, a file they can't change yet. So the skill explains and plans, and stops. If you want the fixes applied, ask for that as a separate step.

## What it checks

The catalog (`references/catalog.md`, v2) covers 29 anti-patterns, each with a stable ID, a default severity, why it matters, how to fix it, and a list of legitimate uses so it doesn't flag false positives.

| Group | IDs | Examples |
|---|---|---|
| State & Effects | A1–A12 | Derived state, event logic in Effects, chained Effects, fetching without cleanup, prop copied into state |
| Memoization | B1–B4 | `useMemo`/`useCallback` that feed nothing, memoizing cheap operations |
| Component structure | C1 | Component defined inside another component |
| Hooks & state shape | D1–D9 | Missing deps, conditional hooks, stale closures, direct mutation, several booleans for one status |
| Rendering, lists & context | E1–E3 | `key={index}`, `&&` with a number, mega-contexts and unstable provider values |

It also adapts advice to your project when it can see it: React version (e.g. `useEffectEvent` only on 19.2+), whether React Compiler is on (memoization findings are skipped), React Native vs web, and which data-fetching library you already use.

## Installation

### Option 1 — `npx skills` (recommended)

Uses the open [skills CLI](https://github.com/vercel-labs/skills). It detects which agents you have (Claude Code, Cursor, Codex, OpenCode, and more) and puts the skill in the right folder. Requires Node.js.

```bash
# Install into the current project (interactive: pick your agents)
npx skills add fullyslick/react-antipatterns-review

# Install globally, for every project
npx skills add fullyslick/react-antipatterns-review -g

# Non-interactive, Claude Code only, global
npx skills add fullyslick/react-antipatterns-review -g -a claude-code -y
```

Keep it up to date later with:

```bash
npx skills update
```

### Option 2 — `git clone` (no Node.js needed)

Clone the repo straight into your agent's skills folder. The folder name should match the skill name.

**Claude Code, global (all projects):**

```bash
git clone https://github.com/fullyslick/react-antipatterns-review.git \
  ~/.claude/skills/react-antipatterns-review
```

**Claude Code, single project:**

```bash
git clone https://github.com/fullyslick/react-antipatterns-review.git \
  .claude/skills/react-antipatterns-review
```

**Other agents** (Cursor, Codex, Gemini CLI, Copilot) that read from `.agents/skills/`:

```bash
git clone https://github.com/fullyslick/react-antipatterns-review.git \
  .agents/skills/react-antipatterns-review
```

To update, run `git pull` inside that folder.

On Windows (PowerShell), replace `~` with `$HOME`.

### Option 3 — Claude.ai / Claude Desktop

1. Download the repo as a ZIP (**Code → Download ZIP**) or zip the folder yourself, keeping `SKILL.md` at the top level of the folder.
2. In Claude, go to **Settings → Capabilities → Skills** and upload the ZIP.

### Check it worked

The installed folder should look like this:

```
react-antipatterns-review/
├── SKILL.md
└── references/
    ├── catalog.md
    └── report-format.md
```

Restart your agent (or start a new session) so it picks up the skill.

## Usage

You don't need to name the skill. Any React review or hook question triggers it:

```
Review src/components/TodoList.tsx for anti-patterns
```

```
Why does this component re-render so much? [paste code]
```

```
Check the React changes in this PR before I merge
```

Before reviewing, it asks one question: show the review and plan in the chat (default), or also save them as a Markdown file? Skip the question by saying it up front, e.g. *"review this and save it to a file"* or *"just show me"*.

Saved files are named `react-review-<scope>-<YYYY-MM-DD>.md` and placed in your project root. Existing files are never overwritten.

Accepted input: files on disk or uploaded, pasted snippets, diffs and PRs, or a source file open in Chrome.

## Example output

```markdown
# React anti-pattern review

Scope: TodoList.tsx · 3 findings (1 High · 1 Medium · 1 Low) · catalog v2
Assumptions: React 18, no compiler

## src/components/TodoList.tsx

### F1 [HIGH] Rows keyed by index — E1 (line 42)
**What:** `todos.map((t, i) => <TodoRow key={i} ... />)`
**Why it's flagged:** rows can be deleted and each holds an input's state, so deleting row 1 shifts row 2's text into it.
**Fix:** `key={t.id}`

...

## Fix plan

### [ ] 1. Key rows by `todo.id` — fixes F1
- **Where:** TodoList
- **Effort:** S
- **Check it worked:** delete the first row; the second row's input keeps its text
```

## Repository structure

| File | Purpose |
|---|---|
| `SKILL.md` | Entry point: workflow, scan checklist, rating rules |
| `references/catalog.md` | Every anti-pattern with ID, severity, fix and legitimate uses |
| `references/report-format.md` | Report template, fix-plan template, file delivery rules |

## Contributing

Issues and pull requests are welcome. If you add a catalog entry, give it a new ID. Existing IDs are never renumbered, so older reviews keep pointing at the right entry. Add its default severity to the severity index at the top of `catalog.md`.

## Sources

The catalog draws on:

- [You Might Not Need an Effect](https://react.dev/learn/you-might-not-need-an-effect) (react.dev)
- [Separating Events from Effects](https://react.dev/learn/separating-events-from-effects) (react.dev)
- [Updating Objects in State](https://react.dev/learn/updating-objects-in-state) (react.dev)
- [How to useMemo and useCallback](https://www.developerway.com/posts/how-to-use-memo-use-callback) (developerway.com)