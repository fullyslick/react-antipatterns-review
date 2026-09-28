---
name: react-antipatterns-review-code
description: >-
  Review React/JSX/TSX code for anti-patterns in Claude Code — redundant/derived state, unnecessary
  Effects, useless useMemo/useCallback, components defined inside components, Rules-of-Hooks
  violations, missing dependencies, and related smells. This skill DIAGNOSES: it reports findings,
  can drop non-behavioral review comments inline at the exact lines, and can write a remediation
  plan — but it never changes the code's logic. Use it whenever the user asks you to review, check,
  audit, annotate, or "look at" React code for anti-patterns, unnecessary re-renders, effect
  misuse, or memoization — across a file, folder, or diff. Trigger even without "anti-pattern":
  "why does this re-render so much?", "annotate the problems in src/components", or "what's wrong
  with this useEffect?" all count. Also trigger when they ask for a plan to fix
  React issues without touching the code yet. Do NOT use for non-React code, and do NOT rewrite the
  logic — this skill stops at comments and a plan; applying fixes is a separate opt-in.
---

# React Anti-Pattern Review — Claude Code (diagnose, annotate, plan)

Review React code against a catalog of well-known anti-patterns. This variant runs where you have
the filesystem and tools, so it can do three things — but all three are **diagnostic, not
corrective**:

1. **Report** findings in a per-file walkthrough (always).
2. **Annotate** the code with inline review comments at the exact lines (when the user wants it).
3. **Plan** the remediation as an ordered, checkable list (when the user wants it).
   **The one hard rule: you do not change the code's behavior.** You may *add* comment lines. You must
   not edit logic, rename things, reorder statements, delete code, or drop in a "corrected" version.
   Adding a comment is the only mutation allowed, and even that is opt-in. The user explicitly wants to
   see and decide, not have the code rewritten under them.

## The catalog

`references/catalog.md` is the full list of anti-patterns with IDs (A1–A10, B1–B4, C1, D1–D6),
default severities, the "why," and the fix. **Read it before reviewing.** Every finding, every
inline comment, and every plan step must cite a catalog ID, and the reasoning must match what the
catalog says. It also has a "Legitimate uses — do NOT flag these" section — check it so you don't
annotate correct memoization or a correct `key` remount as a problem. False positives are worse here
than in a pure report, because a wrong inline comment lands in the user's source tree.

## Workflow

1. **Read the catalog**, then read the files in scope. The user will point you at what to check (a
   file, a folder, a diff, changed files). Review exactly that scope; use `git diff`/`git status`
   when they say "what I changed." Don't sprawl into unrelated files unless asked.
2. **Scan each file top to bottom**, running every check (A1–A10, B1–B4, C1, D1–D6).
3. **Confirm each candidate is real** against "Legitimate uses." Drop the false positives.
4. **Assign severity** (High/Medium/Low) from the catalog default, adjusted for the concrete case;
   explain deviations.
5. **Deliver the written report** (always) in the per-file walkthrough format below.
6. **Then offer** the two optional actions — inline annotations and/or a remediation plan — unless
   the user already asked for them, in which case just do it. If the user's request clearly implies
   one ("annotate the problems", "write me a fix plan"), do that one directly.
   **Make fetch and mutation fixes complete.** When a finding is about data fetching (A10) or a
   network write done through a "trigger" Effect (the A4 POST case), the fix is not just "add a cleanup
   flag" or "move the call into the handler." Follow the catalog's "Completing a data-fetching or
   mutation fix" checklist and name, for *this* code, which pieces are missing — cancellation via
   `AbortController` (not merely an ignore flag), error state, loading state, debouncing a
   keystroke-driven fetch — and point at a reusable `useFetch`-style hook or a data library (TanStack
   Query / SWR / RTK Query, or `axios`). This applies in the report, in any inline comment on those
   lines, and in the remediation plan step for the finding. A one-line "add cleanup" is incomplete.

Bias toward precision. A few well-justified findings beat a long, shaky list — especially when
they'll become comments in the codebase.

## 1) The written report — per-file walkthrough

Go **file by file, and within each file top to bottom in reading order** (by line number). It's a
walkthrough the reader follows in their own editor, not a severity-sorted list.

```
# React anti-pattern review
 
Scope: <what you reviewed> · <N> findings (<x> High · <y> Medium · <z> Low)
 
## <path/to/File.tsx>
 
### [HIGH] <short title> — <catalog ID> (line <N> / `functionName`)
**What:** <the specific code flagged, quoted or tightly described>
**Why it's flagged:** <the concrete problem in THIS code — the catalog's reasoning, made specific>
**Fix:** <the change to make, tiny before → after snippet where it helps>
 
## Summary
<2–4 sentences: the top themes, anything recurring across files, what to fix first. No new findings.>
```

Same rules as the report: reading order within a file (severity is in the tag), cite a catalog ID
on every finding, make the "why" about *their* code, keep fix snippets minimal, and say so
explicitly when a file is clean.

## 2) Inline annotations (opt-in, comments only)

When the user wants the findings marked in the code, add a single-line review comment on the line
**above** each flagged line. Use a consistent, greppable marker so the user can find and remove them
all in one pass. **Never** put the comment inside a string/JSX text where it would render, and never
alter the line being flagged.

Comment format (match the file's comment syntax — `//` for JS/TS/JSX, `{/* */}` inside JSX):

```
// REACT-ANTIPATTERN [HIGH][C1] Component defined inside a component: remounts on every render,
// wiping local state and focus. Fix: hoist `Input` to module scope, pass values via props.
const Input = () => <input />;
```

Inside JSX where `//` isn't valid:

```jsx
{/* REACT-ANTIPATTERN [LOW][B2] useMemo feeds a DOM element — memo compares against nothing; remove it. */}
<button onClick={onClick}>Click me</button>
```

Rules for annotations:

- **Comments only.** Adding these comment lines is the *only* edit you make. The flagged code stays
  byte-for-byte the same. Do not "while I'm here" fix anything.
- **One marker per finding**, on the line above the code it refers to. Keep it to 1–3 lines: severity,
  catalog ID, the specific problem, and the one-line fix direction.
- **Always the `REACT-ANTIPATTERN` prefix** so `grep -rn "REACT-ANTIPATTERN"` finds every comment and
  the user can strip them with a single find-and-replace when done.
- **Put the removal instruction in the written report**, not only in your chat reply — the report is
  the artifact the user keeps, so end it with a short "Removing these annotations" line giving the
  exact `grep -rn "REACT-ANTIPATTERN"` command. Restating it in chat is fine, but the report must be
  self-contained.
- If annotating many files, summarize what you touched (files + count of comments added). You added
  only comments — reassure the user their logic is untouched, and they can `git diff` to confirm.
## 3) Remediation plan (opt-in)

When the user wants a plan (or asks how to address the findings), write an ordered, checkable plan —
a `react-antipattern-remediation-plan.md` file, or inline if they prefer. This plans the work; it
does **not** perform it.

Structure the plan by priority, not by file, so the user tackles the dangerous things first:

```
# React anti-pattern remediation plan
 
## Order of attack
1. High-severity correctness issues (do first)
2. Medium-severity structural fixes
3. Low-severity cleanup (batch these)
 
## Steps
 
### [ ] Step 1 — [HIGH][C1] Hoist `Input` out of `Component` (src/Form.tsx:42)
- Problem: nested component remounts every render (state/focus loss).
- Change: move `Input` to module scope; pass `value`/`onChange` as props.
- Risk / blast radius: low — mechanical move; verify focus behavior after.
- Verify: type in the field; focus should persist across parent re-renders.
 
### [ ] Step 2 — [HIGH][A10] Add cleanup to the results fetch (src/Search.tsx:31)
- Problem: no ignore flag → out-of-order responses overwrite fresh data.
- Change: add `let ignore = false` + cleanup, per catalog A10.
- Verify: type fast; results should match the latest query.
 
...
 
## Grouped quick wins
- [ ] Remove useless memoization: B1/B2/B4 sites in <files> — delete the useMemo/useCallback.
```

Plan rules:

- **Order by severity/risk**, High first. Group the Low-severity cleanups so they can be done in one
  sweep.
- Each step: the catalog ID + location, the problem in one line, the concrete change, its risk/blast
  radius, and how to verify it's fixed. The verify step matters — it's how the user (or a later
  session) confirms the fix without regressing behavior.
- Checkboxes (`[ ]`) so progress is trackable.
- The plan describes the fixes; it does not apply them. Applying fixes is a separate step the user
  explicitly asks for later. If they do ask you to apply the plan afterward, that's fine — but this
  skill's job ends at the plan.
## Staying in your lane

- **Never change behavior.** Adding review comments (opt-in) is the only file mutation. No logic
  edits, no refactors, no renames, no reordering, no deleting, no full-file "fixed version."
- Keep everything anchored to the catalog. Something bad that isn't in the catalog can be mentioned,
  but label it as outside the catalog (your judgment), not as a documented pattern.
- If the user says "go ahead and fix these," confirm they want to leave diagnose-mode, then help —
  but by default this skill reports, optionally annotates with comments, optionally plans, and stops.