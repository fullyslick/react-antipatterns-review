---
name: react-antipatterns-review
description: "Review React/JSX/TSX code for anti-patterns (Effects, state, hooks, memo, keys, mutation) and output a prioritized fix plan without editing code. Use for any React code review, audit, re-render or hook question, even if 'anti-pattern' isn't said."
---

# React Anti-Pattern Review (observe-only)

Review React code against a catalog of known anti-patterns, write a per-file report, and finish
with a prioritized fix plan. The source code is never modified.

The read-only stance is the point of this skill. People run it to learn what's wrong and what to do
about it, often on code they can't or don't want to change yet: a PR they're reviewing, a file open
in the browser, a teammate's snippet. So show small "before → after" snippets that illustrate each
fix, but don't rewrite their components or save corrected copies. If they want the changes applied,
that's a separate step they ask for.

## Files in this skill

- `references/catalog.md` — every anti-pattern with an ID (A1–A12, B1–B4, C1, D1–D9, E1–E3),
  the default severity index, why it matters, the fix, and a "Legitimate uses" list. Read it before
  reviewing, every time. Each finding cites an ID and its reasoning has to match the catalog, which
  is what makes a flag justified rather than just labeled. The "Legitimate uses" section is how you
  avoid false positives.
- `references/report-format.md` — the report template, the fix plan template, and how to save the
  Markdown file. Read it before writing the output.
- `evals/evals.json` — test prompts for checking the skill's behavior. Not needed during a review.

## Workflow

### 1. Settle scope and output in one message

Every review ends with a fix plan, so don't ask whether to make one. What you do ask, once and
before reviewing, is where the output goes:

> Should I just show the review and fix plan here (default), or also save them as a Markdown file?

Skip the question when the user already said ("save it", "just show me", "write it to review.md"),
or when the environment can't write files (then show it in the conversation). If the request is also
vague about *what* to review ("check my component" with nothing attached), ask both in the same
message so the user answers once. If they reply with code but don't answer the output question, use
the default. Asking twice would be more friction than the choice is worth.

In a project folder, a quick look for an obvious target is fine (the file open in the editor, the
only component in the folder). If you'd have to pick among several candidates, ask instead:
reviewing a file the user didn't mean wastes their time and looks like a real answer.

Accepted input: files (in the session, uploaded, or on disk), a pasted snippet, a diff or PR
(review the changed code; note older patterns only if worth it), or a page open in Chrome (read
the raw file text, not the rendered page). Review exactly the scope given.

### 2. Read the catalog and the project context

Read `references/catalog.md`, then the code. When `package.json`, framework config, or imports are
visible, note the things that change the advice:

- **React version** — `useEffectEvent` is only a valid fix on React 19.2+; otherwise use the ref pattern.
- **React Compiler** — if it's on, memoization findings (B1–B4, D1, E3's unstable-value case) don't
  apply, because the compiler memoizes automatically.
- **React Native vs web** — E2 crashes in React Native.
- **Existing data library** — point fetch fixes at the one the project already uses.

If you can't see this context, state your assumption in the report header (e.g. "assuming React 18,
no compiler") so the reader can correct it.

### 3. Scan each component

Walk each file top to bottom with this checklist, then read the full catalog entry for anything that
matches:

| Look at | Ask | IDs |
|---|---|---|
| Components declared in a component body | Is any rendered as `<X />`? | C1 |
| Hooks | Inside `if`/loop/`&&`, or after an early return? | D4 |
| `useState` | Derived from other state/props? Copied from a prop? An object copied out of a list? Never rendered? | A1, A2, A12, D6 |
| Several `useState` | Booleans for one status? Coupled transitions? | D9, D5 |
| State updates | In-place change (`.push`, `.sort`, `obj.x =`), or `setX(x)` after editing `x`? | D8 |
| `useEffect` | Needed at all, or a derivation / event handler? Chained? Unguarded one-time init? | A1–A8 |
| `useEffect` that starts something | Does a cleanup undo exactly that? | A9, A10, A11 |
| Dependency arrays | Every read value listed? | D3 |
| Timers, `await`, listeners | Reading state/props that may be stale by then? | D7 |
| `useMemo` / `useCallback` / `memo` | Does the reference feed a real comparison? | B1–B4, D1 |
| Lists | Stable id keys, not index or random? | E1 |
| JSX conditionals | Numeric left side of `&&`? | E2 |
| Context providers | Catch-all context? Fresh `value` object each render? | E3 |
| CSS-in-JS | Static styles computed in render? | D2 |

C1, D7, D8, and A2-style desync (a copied object going stale) look like working code, which is why
reviewers miss them. Check for them on purpose rather than waiting for them to stand out.

### 4. Confirm, then rate

Check each candidate against "Legitimate uses." If it's there, drop it, or note in one line why a
close call is fine. Start severity from the catalog's index and adjust for the concrete case: raise
it for hot paths, money/auth, network writes, React Native, or user-visible desync; lower it for
trivial, tiny, static, or primitive cases. Say why when you deviate.

Report each root cause once. When several IDs describe the same lines (a fetch Effect that is both
D3 and A10; three status booleans that are D9 and partly D6), write one finding that cites all of
them. Listing the same code under separate findings inflates the count and splits the fix across
plan steps.

Precision matters more than volume. A short list of correct, well-argued findings is useful; a long
list padded with guesses teaches the reader to ignore the review. When unsure, leave it out or label
it "worth a look, not certain."

For data fetching (A10) and network writes moved out of Effects (A4), the fix isn't just "add a
cleanup" or "move it to the handler." That removes the anti-pattern but leaves the code half-built.
Use the catalog's "Completing a data-fetching or mutation fix" checklist and name what's missing in
*this* code: AbortController, error handling, loading/error/empty UI, debouncing. Then point at a
reusable hook or the project's data library.

### 5. Write the report and plan, then deliver

Follow `references/report-format.md`: a file-by-file walkthrough in reading order with numbered
findings (F1, F2, …), a short summary, then the fix plan ordered by priority. Show it in the
conversation by default, or save the Markdown file if the user chose that.

## Staying in your lane

- The only file this skill may create is the review/plan Markdown file, and only when the user chose
  it. Source files are never touched, and that includes adding review comments to them. Don't
  offer inline annotations: they're edits, and they leave review noise in code the user may be
  about to commit.
- End after the plan. The output choice was settled in step 1, so don't close with a menu of
  follow-ups (save a plan file? annotate the code?). One line saying they can ask you to apply the
  plan is enough.
- If the user asks you to "just fix it," confirm that means leaving review mode, then help. By
  default the skill reports, plans, and stops.
- Something bad that isn't in the catalog (a 300-line component mixing fetching and UI, an
  accessibility problem) can be mentioned, labeled as outside the catalog so the user knows it's your
  judgment. It gets a plan step only if it's High.