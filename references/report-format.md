# Report, fix plan, and delivery format

Read this before writing the output. The report explains each problem where it sits in the code; the
plan turns the findings into an ordered to-do list.

## Contents

- Report — per-file walkthrough
- Fix plan
- Delivering the output

## Report — per-file walkthrough

Go file by file, and within each file top to bottom in reading order. It's a walkthrough, not a
severity-sorted list, so the reader can follow along in their own code; priority lives in the plan.
Number findings across the whole report (F1, F2, …) so plan steps can refer to them.

```
# React anti-pattern review

Scope: <what you reviewed> · <N> findings (<x> High · <y> Medium · <z> Low) · catalog v2
Assumptions: <React version / compiler / platform, if not visible in the code>

## <path/to/File.tsx>

### F1 [HIGH] <short title> — <catalog ID> (<anchor>)
**What:** <the specific code, quoted or tightly described>
**Why it's flagged:** <the concrete problem in THIS code, using the catalog's reasoning>
**Fix:** <the change, with a tiny before → after snippet where it helps>

### F2 [MEDIUM] ...

## <path/to/OtherFile.tsx>
...

## Outside the catalog / worth a look   (omit if empty)
- <one line each: an uncertain catalog match, or a real problem the catalog doesn't cover>

## Summary
<2–4 sentences: the highest-severity themes, patterns that recur across files, and the one or two
things to fix first. No new findings here.>
```

Guidelines:

- **Severity tag in the heading** so readers can scan for serious ones while the order stays sequential.
- **Anchors:** `line 12`, `lines 12–19`, or `around line 12` when you can count lines; otherwise
  the function or expression, e.g. `` `toggle` `` or `` `const sorted = todos.sort(...)` ``. Use one
  form, never a half-filled one like "around line, `toggle`".
- **Uncertain matches and outside-catalog notes** go in their own section before the Summary, not
  inside the Summary, which only restates the through-line of the findings.
- **The "Why" is about their code.** "`filtered` is set only inside the Effect that reads `items` and
  `query`, so it's derived state (A2)" beats "storing derived data in state is bad."
- **Fix snippets stay small** — the minimal before/after that shows the idea.
- **Clean files are reported as clean:** "`Foo.tsx` — no anti-patterns found." Inventing findings to
  fill space erodes trust in the real ones.

## Fix plan

Always included when there are findings; skip it and say the code is clean when there are none. The
plan doesn't repeat fix snippets or contain rewritten components — each step points back to findings.

**Ordering:**

1. **Crashes and wrong behavior first** — High findings, especially C1, D4, D8, D7, A10/A11, E1.
2. **Fixes that delete code before fixes that tweak it.** Deriving state (A1/A2/A12) or moving logic
   into handlers (A4/A7) often removes an Effect entirely, which can make a D3 or A11 finding on that
   Effect disappear. Say so when one step makes another unnecessary.
3. **Group steps that touch the same code**, so the user edits each place once.
4. **Low findings last**, in an "Optional clean-up" group.

```
## Fix plan

### [ ] 1. <action in imperative form> — fixes F<n>[, F<m>]
- **Where:** <file / component>
- **Effort:** S | M | L
- **Check it worked:** <one concrete check, e.g. "delete the first row; the second row's input keeps its text">
- **Note:** <only if needed: depends on step X / makes step Y unnecessary>

### [ ] 2. ...

### Optional clean-up (Low)
- [ ] <one line per step, with finding refs>
```

The `[ ]` boxes let the user tick steps off when the plan is saved as a file; keep them in the
conversation version too so both look the same.

Effort: **S** = a few lines in one place; **M** = several places, or a new small hook or state shape;
**L** = a new library, a context split across files, or a component restructure.

## Delivering the output

**Default — in the conversation.** Show the report and plan in the chat/terminal. Write no file.

**Markdown file — only when the user chose it.** Write one file with the full report followed by the
plan (plan steps reference findings, so they belong together).

- Name: `react-review-<scope>-<YYYY-MM-DD>.md`, where `<scope>` is the file or folder reviewed
  (e.g. `react-review-TodoList-2026-09-28.md`).
- Location: the user's working folder or project root (in Cowork, the folder they selected). If the
  name is taken, add `-2`, `-3`, … rather than overwriting.
- Afterwards, give the path and a 2–3 line summary (finding counts and the first plan step) rather
  than repeating the whole report.