---
name: code-quality
description: Review a PR, branch, or full repo for code quality and reuse. Applies quality gates, reuse checks, and clean code principles. Use when asked to review code, check a PR, run code quality on a branch, or refactor an entire repo.
---

# Code Quality Review

Review a branch, a PR, or a full repo (`--repo`) with code style gates, reuse checks, and clean code principles.

## Constraints

- **Report by default.** Edit files or commit only when the user asks for fixes. `--repo` counts as that request. Do NOT push; the user pushes.
- **Touch only files in the branch's changes.** Do not refactor unrelated code.
- **Prefer smaller diffs.** Do not clean up code next to the changes.
- **If all changes are clean, say so.** Do not invent work.
- **Add no comments, docstrings, or type annotations** to unchanged code.

## Handle --help Flag

If the argument is `--help`, `-h`, or `help`, print this guide and DO NOT run the review:

```
/code-quality - Review code changes for quality and reuse

USAGE:
  /code-quality <PR-url>
  /code-quality <branch-name>
  /code-quality <branch-name> <repo-path>
  /code-quality --repo                    Full repo refactor (logic and tests in separate commits)
  /code-quality --deps                    Focus on module dependencies
  /code-quality --testability             Focus on unit testability
  /code-quality --principle=<name>        Focus on one clean code principle

ARGUMENTS:
  PR URL        Full GitHub PR URL (e.g., https://github.com/org/repo/pull/123)
  branch        Branch name to review (e.g., ABC-123-add-feature)
  repo-path     Path to the repo (defaults to current working directory)

CLEAN CODE PRINCIPLES:
  1. meaningful-names       Names reveal intent
  2. no-side-effects        Functions act OR return, not both
  3. dry                    No duplicated knowledge
  4. single-responsibility  One reason to change per module
  5. minimal-comments       Code is self-documenting
  6. consistent-formatting  Related code grouped together
  7. error-handling         Exceptions over null, proper async
  8. testable-code          Designed for unit testing

EXAMPLES:
  /code-quality https://github.com/org/repo/pull/510
  /code-quality feature-branch ~/Development/your-repo
  /code-quality feature/new-widget
  /code-quality --deps
  /code-quality --testability
```

## Parse Arguments

1. For a GitHub PR URL, extract the owner/repo and PR number. Get the branch with `gh pr view <number> --json headRefName`.
2. For a branch name, use it directly.
3. For a repo path, `cd` to it. Otherwise use the current directory.
4. For `--repo`, run Full Repo Refactor mode.
5. For `--deps`, `--testability`, or `--principle=<name>`, run only that mode (see Analysis Modes).
6. **With no arguments**, check the current branch:
   - On a feature branch (not main/master/dev/develop), review it against the base branch. This is the default.
   - On main/master/dev/develop, offer these options and wait:
     - `--repo`: full repo refactor
     - Review the last commit
     - Review a specific branch (list recent branches)
     - Review a specific PR

## Step 1: Checkout and Identify Changes

```bash
git fetch origin <branch>
git checkout <branch>
git pull origin <branch>
git diff main...HEAD --name-only
```

Read all changed files. Skip non-code files (.changeset, .gitignore, README, etc.).

## Step 2: Dispatch Quality Gates

These rules apply to ALL repos. Scan `git diff main...HEAD` for violations in **new code only**. Violations on untouched lines are out of scope.

### Code Style Gates

- **No for loops.** Use `.forEach()`, `.filter()`, `.map()`, `.flatMap()`, `.some()`, `.every()` instead of `for (let i = ...)` or `for...of`.
- **Stop iterating once the goal is met.** To find one item or test one condition, use `.find()`, `.findIndex()`, `.some()` or `.every()`. They stop at the first match. Do not `.forEach()` or `.filter()` the whole collection and take the first result.

  ```typescript
  // Bad: walks every item, then takes the first
  const firstExpired = items.filter((item) => item.isExpired)[0]

  // Good: stops at the first match
  const firstExpired = items.find((item) => item.isExpired)
  ```
- **No else branches.** Use early returns, guard clauses, `??`, and single-level ternaries.
- **No nested ternaries.** A ternary inside a ternary branch is an `if/else if/else` chain. Replace it with early returns in a small function, or with a lookup when the branches map a key to a value.

  ```typescript
  // Bad
  const endReason = result.isAbandoned
    ? resolveAbandonedEndReason(result.snapshot)
    : result.isComplete
      ? END_REASONS.COMPLETED_SUCCESS
      : undefined

  // Good
  function resolveEndReason(result: RunResult): EndReason | undefined {
    if (result.isAbandoned) return resolveAbandonedEndReason(result.snapshot)
    if (result.isComplete) return END_REASONS.COMPLETED_SUCCESS
    return undefined
  }
  ```
- **No `Object.entries`.** Iterate `Object.keys()` and index by key, or use `Object.values()` when the key is unused. `[key, value]` tuples read as array positions, not names. When the value type allows `undefined`, use `?.` and `??` on the lookup.

  ```typescript
  // Bad
  const readyFields = Object.entries(getSections(config))
    .filter(([sectionName]) => isSectionReady(input, sectionName))
    .flatMap(([, section]) => section.fields)

  // Good
  const sections = getSections(config)
  const readyFields = Object.keys(sections)
    .filter((sectionName) => isSectionReady(input, sectionName))
    .flatMap((sectionName) => sections[sectionName]?.fields ?? [])
  ```
- **At most two levels of loops or iterators.** One iterator callback (`.map()`, `.filter()`, `.find()`, `.some()`, `.every()`, `.forEach()`, `.flatMap()`, `.reduce()`) inside another is the limit. A third level hides which collection drives the work and often recomputes the inner result on every pass. Hoist the inner level into a named `const`, or extract a function.

  ```typescript
  // Bad: three levels
  const ruledField = missingFields.find((field) =>
    sections.some((section) => section.rules.some((rule) => rule.field === field))
  )

  // Good: the inner levels run once
  const ruleFields = sections.flatMap((section) => section.rules.map((rule) => rule.field))
  const ruledField = missingFields.find((field) => ruleFields.includes(field))
  ```
- **No iterators inside ternaries.** A ternary picks a value. It does not run a loop in a branch. Return early, or let the ternary pick the collection and run the iterator once on the result.

  ```typescript
  // Bad
  const labels = isActive ? items.map((item) => item.label) : []

  // Good
  const visibleItems = isActive ? items : []
  const labels = visibleItems.map((item) => item.label)
  ```
- **No function calls inside ternaries.** The same rule applies to calls such as `formatLabel(item)`. Use an early return, or let the ternary pick the argument and call once outside it.

  ```typescript
  // Bad
  return isActive ? formatLabel(item) : ''

  // Good
  if (!isActive) return ''
  return formatLabel(item)
  ```
- **Chain over intermediates.** Prefer `.filter().forEach()` and `.flatMap()` chains over accumulator loops with push.
- **Prefer immutable variables.** Use several immutable declarations, not one reassigned variable: `const` over `let` in TypeScript/JavaScript, `final` in Java, tuples or frozen dataclasses in Python, short-lived values over pointer reassignment in Go.
- **No one-line (or two-line) functions.** A body of one statement or short composed call (a `.map().find()` chain, a trim-and-compare, a formatted string) is indirection. Inline it at each call site, **no matter how many times it repeats**. For a repeated literal value, extract a named constant, not a function. Exempt: exported APIs, interface implementations, and required callback signatures.

### Test Gates

1. **Every test is deterministic**
   - No `setTimeout` or timing-based assertions. Await the promise or use fake timers.
   - No loops in tests. A loop hides which iteration failed, and an empty array passes silently. Assert on specific indices (`result[0]`), use `.every()` with a length guard, or assert on the full array with `.toEqual()`.
   - No mutable state (`callCount`, flags) with if/else in mocks. Chain `mockResolvedValueOnce` / `mockReturnValueOnce`.
   - Import the constants the source code uses. Never hardcode a string or number that exists as a named export.

2. **Every test is declarative**
   - Readable without mentally simulating state
   - The description matches exactly what the test asserts
   - Arrange, Act, Assert, all three visible in the test body

3. **Every test can fail**
   - Do not assert on a value you just set on the same object (tautology)
   - Do not assert that a string constant equals its own hardcoded value
   - Do not only parse a valid object against a schema and assert `success === true`. Run the real execution path and verify side effects.

4. **Scope**
   - One behavior per test
   - Test the contract (inputs to outputs and side effects), not the implementation
   - Cover meaningful edge cases: empty inputs, missing optional fields, error paths

Use the project's native mocking framework.

### How to Check

Run these searches against changed files only:

```bash
# Get list of changed files
git diff main...HEAD --name-only | grep -v '.changeset\|.gitignore'

# Check for violations in the actual diff (new lines only)
git diff main...HEAD | grep '^+' | grep '\bfor\s*('  # for loops
git diff main...HEAD | grep '^+' | grep '\belse\b'   # else branches
git diff main...HEAD | grep '^+' | grep '\blet\b'    # let declarations
git diff main...HEAD | grep '^+' | grep 'Object\.entries('  # Object.entries

# Iterator calls, for the two checks below. `?.map(` is excluded because `?` must be followed by a space.
ITER='\.(map|filter|find|findIndex|findLast|some|every|forEach|flatMap|reduce)\('
# Iterators inside a ternary: on the same line as `? `, or on a `?`/`:` branch line.
git diff main...HEAD | grep '^+' | grep -E "(^\+\s*[?:] |[^?]\? ).*$ITER"
# Function calls inside a ternary: same shape as above, any call. Expect some false positives from `?.` chains; read each hit.
git diff main...HEAD | grep '^+' | grep -E "(^\+\s*[?:] |[^?]\? ).*[A-Za-z_][A-Za-z0-9_]*\("
# Filter-then-first: walks the whole collection to take one item.
git diff main...HEAD | grep '^+' | grep -E '\.filter\(.*\)(\[0\]|\.at\(0\)|\.shift\(\))'
# Three iterator levels on one line. Multi-line nesting still needs a read of each new callback.
git diff main...HEAD | grep '^+' | grep -E "$ITER.*$ITER.*$ITER"

# Nested ternaries: a ternary branch line indented deeper than the branch line above it,
# or two ternaries on one line. Requiring a space after `?` skips `?.` and `??`.
git diff main...HEAD | awk '
  /^\+/ {
    line = substr($0, 2); match(line, /^[ \t]*/); indent = RLENGTH
    isBranch = (line ~ /^[ \t]*[?:] /)
    if (isBranch && prevBranch && indent > prevIndent) print
    if (line ~ /[^?]\? .* : .*[^?]\? /) print
    prevBranch = isBranch; prevIndent = indent; next
  }
  { prevBranch = 0 }'

# Comment syntax: block/JSDoc comments, and // without the trailing space.
# Expect only eslint-disable directives and empty-catch markers to survive the filter.
git diff main...HEAD | grep '^+' | grep '/\*' | grep -v 'eslint-disable'
git diff main...HEAD | grep '^+' | grep '//[^ /]' | grep -v 'https\?://'
```

List the violations. Separate new-code violations (fix) from pre-existing violations in touched files (note, do not fix).

## Step 3: Reuse before addition

Check that no existing code path already solves the problem, or solves it closely enough to serve with a small change. New code that reimplements what the codebase already does is a defect, even when it duplicates nothing *within* the diff. At the module level, add to an existing library or use a package the repo already depends on before you create a new library.

For each new code path, find the closest existing code (same file, same service, shared utils). Use the least new code, in this order:

1. **Reuse the existing path unchanged.** Zero new logic. (Example: a new "send this notification" path calls the existing notification-dispatch helper instead of its own transport lookup and send.)
2. **Reuse it with a small addition.** One extra parameter, an optional flag, or one new branch, when that does not distort the path's responsibility.
3. **Extract a shared path.** When the new code and an existing block solve the same problem the same way, move the common part into one helper both call. (Example: an automatic-retry block that repeats the manual-retry block's validate/execute/fallback sequence.)

A hand-rolled copy of a shared util goes to option 1 or 2. A near-copy of a nearby block goes to option 3. Prefer the option that changes the fewest call sites. Do not create a helper whose shared body is one or two lines (see the one-line function gate).

## Step 4: Clean Code Principles

Apply all 8 principles to the changed files.

**Readability over cleverness applies to all 8.** When a clever form (a dense one-liner, a short-circuit trick) and a plain form do the same thing, use the plain form.

### 1. Meaningful Names

Names reveal intent without comments.

**Check for:** Single-letter variables (except loop counters), generic names (`data`, `info`, `temp`, `result`, `handler`), unclear abbreviations, booleans not phrased as questions.

```typescript
// Bad                          // Good
const d = new Date();           const createdAt = new Date();
const u = getUser();            const currentUser = getUser();
let flag = true;                let isEnabled = true;
```

### 2. No Side Effects

A function performs an action OR returns data, not both (Command-Query Separation).

**Check for:** Functions that return values and modify external state, mutated input parameters, hidden I/O in pure-looking functions.

```typescript
// Bad - mutates input
function addItem(cart: Cart, item: Item): Cart {
  cart.items.push(item);
  return cart;
}

// Good - returns new instance
function addItem(cart: Cart, item: Item): Cart {
  return { ...cart, items: [...cart.items, item] };
}
```

### 3. DRY (Don't Repeat Yourself)

Each piece of knowledge has one authoritative representation.

**Check for:** Duplicate code blocks (3+ similar lines), repeated magic numbers/strings, the same validation in several places, scattered configuration values.

```typescript
// Bad - scattered                // Good - centralized
// file1.ts: MENU_WIDTH = 400    export const Layout = {
// file2.ts: menuWidth = 400       menu: { width: 400, height: 300 },
// file3.ts: width: 400,           editor: { width: 800 }
                                 } as const;
```

**Do not over-DRY.** The one-line function gate overrides repetition count. Never extract a one- or two-line body for reuse, however often it repeats. When the shared thing is a *value* (a magic number, a URL, a wire constant), extract a named constant. Extract a helper only for 3 or more lines of real, duplicated logic.

### 4. Single Responsibility

Each module, class, or file has one reason to change, and each function does one thing.

**Check for:** Files with unrelated concerns, classes that mix data access, business logic, and presentation, services that span domains, functions whose accurate name needs an "and" (`validateAndSave`), and functions that mix a computation with the I/O that consumes it. A function with several paragraphs still does one thing when they all serve one step.

```typescript
// Bad - mixed concerns
class UserService {
  getUser(id) { /* db */ }
  formatName(user) { /* presentation */ }
  sendEmail(to, body) { /* I/O */ }
}

// Good - separated
class UserRepository { getUser(id) { } }
class UserFormatter { formatName(user) { } }
class EmailService { send(to, body) { } }
```

### 5. Minimal Comments

Comments explain *why*, never *what*.

**Check for:** Comments that explain what the code does, commented-out code, TODO/FIXME without a ticket reference.

```typescript
// Bad: Loop through users and check if active
for (const user of users) { if (user.isActive) { } }

// Good: Filter active users - inactive migrated to legacy (JIRA-1234)
const activeUsers = users.filter(user => user.isActive);
```

**Keep a comment only when it carries knowledge the code cannot.** The reader is fluent and often a model that reads every comment each time the file is in context, so a redundant comment costs tokens every time. Keep comments that state **why** the code uses a non-obvious approach, an **invariant**, a **hazard** (rate limits, ordering, thread safety, a value kept in sync elsewhere), a **cross-reference** ("mirrors X"), **domain knowledge** the names do not show, the **provenance** of data ("as of <date>"), or the **purpose** of a type or function the name does not show.

Keep navigation markers (`// MARK:`). Remove the rest, most often doc-comments that paraphrase the signature, section labels that repeat the field names, unit notes on named constants (`// 1 hour` on `3600`), and arrange/act/assert narration. When unsure, keep it: an extra "why" costs a few tokens, and a deleted one is lost.

**Prefer an enforced invariant over a described one.** When a comment states an invariant ("these two must match", "returns exactly N"), consider an assertion or a type, so a change that breaks it fails loudly.

**Comment accuracy near the diff.** Every comment added, modified, or within ~5 lines of changed code must still describe the code. A stale comment misleads.

**Check for** a comment that names a changed or removed parameter, return value, branch, or behavior; describes the *old* algorithm; names a renamed, moved, or closed function, file, or ticket; claims an invariant the code no longer guarantees; gives a wrong count or example ("returns 3 fields" when it returns 4); or has a `@param`, `@returns`, or type that does not match the signature.

```typescript
// Before the change
// Returns the user's active subscriptions, sorted by created date
function getSubscriptions(userId: string): Subscription[] { ... }

// After adding a status filter param, the comment is stale
// Bad: comment unchanged
// Returns the user's active subscriptions, sorted by created date
function getSubscriptions(userId: string, status: Status): Subscription[] { ... }

// Good: comment updated or removed
function getSubscriptions(userId: string, status: Status): Subscription[] { ... }
```

**How to check:** read ±5 lines around each hunk in the file at HEAD. Update each stale comment, or delete it when it only restated the code.

**Code paragraphs.** A function body is a sequence of paragraphs, each one conceptual step (a guard, a normalization, a derivation, a write), separated by one blank line. A 1-2 line comment names each non-trivial step. Do not comment individual lines.

Three rules:

1. **Boundaries are conceptual, not syntactic.** A `try/catch` around an external call is one paragraph, and so is a payload build. Do not split a step.
2. **The comment is a topic sentence.** One line, or two for a tradeoff. A blank line above it, none below it.
3. **Skip the comment when the names explain the paragraph.** Comment only steps whose *why* the code does not show.

```typescript
// Good: the body reads as paragraphs, each headed by intent (or skipped when obvious)
async function syncUpdate(params: { ... }): Promise<void> {
  if (!params.targetId || !params.actor) return

  const normalizedFirst = normalize(params.firstValue)
  const normalizedSecond = normalize(params.secondValue)

  // Normalize the previous values too so the diff compares like-for-like;
  // otherwise upstream formatting noise would re-fire the update every turn.
  const normalizedPrevFirst = normalize(params.previousFirstValue)
  const normalizedPrevSecond = normalize(params.previousSecondValue)
  const firstChanged = normalizedFirst !== undefined && normalizedFirst !== normalizedPrevFirst
  const secondChanged = normalizedSecond !== undefined && normalizedSecond !== normalizedPrevSecond
  if (!firstChanged && !secondChanged) return

  const payload = {
    ...(firstChanged && { first: normalizedFirst }),
    ...(secondChanged && { second: normalizedSecond }),
  }

  try { ... } catch (error) { ... }
}
```

Only the "normalize previous" paragraph needs a comment.

**Check for:** a long body with no blank lines, blank lines *inside* one step, a non-obvious multi-line step with no comment, a comment with no blank line above it, and a topic comment that restates a trivial step.

**Comment syntax: `// ` line comments, never `/** */` blocks.** Use a double slash and one space on every comment, including above a function, even when the file uses JSDoc. A multi-line comment is consecutive `// ` lines.

```typescript
// Bad
/** Which storage backend the uploader writes to. */
/**
 * Conflicts use the full-version check, not the coarse label: a
 * metadata-only edit leaves the label alone but still rejects with 409.
 */
//No space after the slashes

// Good
// Which storage backend the uploader writes to.
// Conflicts use the full-version check, not the coarse label: a
// metadata-only edit leaves the label alone but still rejects with 409.
// Space after the slashes
```

Two exceptions, neither a comment: tooling directives (`/* eslint-disable ... */`) and empty-block markers inside a `catch` (`/* not json */`). Leave them.

**Guard stacks get a blank line and a reason each.** When 3 or more consecutive early-return guards reject for *different* reasons, separate them with blank lines and give each a one-line comment on why it exists. This overrides rule 3: the condition is readable, but *why it disqualifies the operation* is domain knowledge. The smell is a wall of 8 `if (...) return "..."` lines.

```typescript
// Good
// Without live provider state there is nothing to compare a switch against.
if (providerConfig.kind !== "ok") return "Live provider configuration is unavailable"

// The provider was changed outside this control, so the saved setting is no longer trustworthy.
if (hasDrift) return "Saved setting does not match live provider configuration"

// Selecting the new backend with no captured fallback would leave the switch irreversible.
if (!record.fallbackConfig && liveTarget === "new") {
  return "A validated fallback must be captured before selecting the new backend"
}
```

One or two lead-in guards at the top of a function (`if (!id) return`) stay bare and unspaced.

### 6. Consistent Formatting

Group related code and use one style.

**Check for:** Mixed naming conventions, inconsistent indentation, lines > 100-120 chars, inconsistent import ordering.

### 7. Error Handling

Throw exceptions, not null, and keep error handling apart from business logic.

**Check for:** Returning `null`/`undefined` for errors, empty catch blocks, catching generic `Error`, missing async error handling.

```typescript
// Bad - hides failure reason
function findUser(id: string): User | null {
  try { return db.find(id); }
  catch { return null; }
}

// Good - explicit error
function findUser(id: string): User {
  const user = db.find(id);
  if (!user) throw new UserNotFoundError(id);
  return user;
}
```

### 8. Testable Code

Put logic in small, exported, pure functions that tests call with no setup. Keep I/O in thin callers at the edges. "Small" means one job, not one or two lines. Split a function that is hard to test.

**Check for:** Logic in a non-exported function or a handler that a test can reach only through I/O, hard-coded dependencies (`new` internally), direct DB/API calls in logic, scattered `process.env` access, non-deterministic calls (`Date.now()`, `Math.random()`).

```typescript
// Bad - untestable
class OrderService {
  async createOrder(items: Item[]) {
    const db = new Database();
    await db.save({ id: uuid(), items, createdAt: Date.now() });
  }
}

// Good - injectable
class OrderService {
  constructor(private db: Database, private clock: Clock) {}
  async createOrder(items: Item[]) {
    await this.db.save({ id: this.idGen.generate(), items, createdAt: this.clock.now() });
  }
}
```

## Step 5: Diff Reduction Sweep

Shrink the diff to the smallest one that delivers the change. Steps 2-4 can all add churn.

**The gates win.** When the smaller form breaks a Step 2 gate, keep the larger form: splitting a nested ternary into two `const`s, or inlining a one-line helper, adds lines and stays. Never trade behavior or test coverage for a smaller diff.

Measure before and after:

```bash
git diff --shortstat main...HEAD      # all changes
git diff --shortstat -w main...HEAD   # ignoring whitespace; the gap is whitespace-only churn
git diff --color-moved=dimmed-zebra main...HEAD   # moved blocks render dimmed
```

Sweep in this order:

1. **Put moved code back.** Code moved for no reason doubles its lines in the diff.
2. **Revert whitespace and formatting on untouched lines.** Re-indents, re-wraps, blank-line shuffles, and quote-style flips. Exception: when CI enforces the formatter on whole modified files, keep its output in a separate commit.
3. **Revert incidental renames and reorders.** Unneeded renames of variables, parameters, or files; reordered object keys, imports, or `switch` cases.
4. **Do not re-indent to add a signal.** Wrapping a body in `try` or `if` re-indents every line. Prefer an early return, a guard, or a wrapper at the call site.
5. **Consolidate tests.**
   - Fold a new assertion into an existing test that already drives the path.
   - Delete a new test that an extended existing test now covers.
   - Merge two tests only when they assert the same behavior.
   - Reuse existing fixtures and builders, not near-copies.
6. **Drop dead additions.** Unused exports, parameters, types, and imports; debug logs; commented-out code; a widened type nothing uses.
7. **Split out unrelated changes.** Drive-by fixes and cleanups go to their own PR.
8. **Reuse before adding.** Remove new code that an existing helper covers (see Step 3).
9. **Leave generated files alone.** Regenerate lockfiles, snapshots, and codegen only when the change requires it.

Then re-run the Step 2 checks on the new diff and record both `--shortstat` lines for the report.

## Step 6: Lint

Find the lint commands in `package.json` (e.g., `lint`, `lint:eslint`, `lint:types`). Run them and report the errors.

## Step 7: Tests

Run the test files that changed or that test changed code. Stash and re-run to prove a failure is pre-existing, and report it as such.

## Step 8: Fix and Re-verify

If the user asked for fixes and steps 2-7 found issues:
1. Apply the fixes
2. Re-run lint on the fixed files
3. Re-run the affected tests
4. Confirm the fixes add no new issues

## Step 9: Report

```
## Code Quality Report: PR #<number> (<branch-name>)

### Verdict: <Clean | Clean with N fixes applied | N issues found>

### Fixes Applied
- <description of each fix and why>

### Code Style Gates
- For loops: <pass/fail, details>
- Else branches: <pass/fail, details>
- Const over let: <pass/fail, details>

### Clean Code Scores
| Principle | Score | Issues |
|-----------|-------|--------|
| Meaningful Names | X/10 | N |
| No Side Effects | X/10 | N |
| DRY | X/10 | N |
| Single Responsibility | X/10 | N |
| Minimal Comments | X/10 | N |
| Consistent Formatting | X/10 | N |
| Error Handling | X/10 | N |
| Testable Code | X/10 | N |

**Overall Score: X/10**

### Diff Reduction
- Before: <files changed, insertions, deletions>
- After: <files changed, insertions, deletions>
- <each reduction applied, and any kept because a gate required it>

### Lint & Tests
- ESLint: <clean | N errors>
- Tests: <N/N pass | failures>

### Review Notes (no action needed)
- <observations worth noting but not worth fixing>
```

## Analysis Modes

### Default: Full Review
All steps above.

### Full Repo Refactor (`--repo`)

**Never change logic and tests in the same commit.** Each verifies the other, so a commit that changes both verifies neither.

1. **Scan the repo.** Sort files into source and test.
2. **Run the tests first.** If the baseline fails, stop and report.
3. **Phase 1: refactor logic.** Apply the gates and principles to source files only. After each logical group of changes:
   - Run the full test suite
   - Commit only source files (e.g., "refactor: apply clean code principles to auth module")
   - If tests fail, revert and fix before you continue
4. **Phase 2: refactor tests.** Apply the test gates and principles to test files only. After each logical group:
   - Run the full test suite
   - Commit only test files (e.g., "test: refactor auth module tests for clarity")
   - If tests fail, fix the test, not the source.
5. **Repeat** the phases until the repo is clean.
6. **Final check.** Run the full test suite and lint.

Every commit passes the test suite on its own.

### Module Dependencies (`--deps`)
Check for: circular dependencies, god modules (imported everywhere), orphan modules, layer violations.

### Testability (`--testability`)
Check for: hard-coded instantiation, global state access, non-deterministic calls, missing interfaces.

### Single Principle (`--principle=<name>`)
Deep analysis of one principle, with line numbers and specific fixes.

## Grep Pattern Reference

```bash
# Meaningful Names - single letter vars
const [a-z] =
let [a-z] =

# Minimal Comments - TODOs without tickets
// TODO(?!.*[A-Z]+-\d+)
// FIXME(?!.*[A-Z]+-\d+)

# Error Handling
catch\s*\(\s*\w*\s*\)\s*\{\s*\}
return null
return undefined

# Testability - hard-coded deps
new\s+\w+(Client|Service|Repository)\(
Date\.now\(\)
Math\.random\(\)
process\.env\.\w+

# Dead Code (React migrations)
document\.getElementById
document\.querySelector
\.innerHTML
```

## Completion signal

When the review (and any commits) is done, the last thing you say is `meow` on its own line.
