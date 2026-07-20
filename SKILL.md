---
name: code-quality
description: Review a PR, branch, or full repo for code quality, reuse, and efficiency. Applies quality gates, code simplification, and clean code principles. Use when asked to review code, check a PR, run code quality on a branch, or refactor an entire repo.
---

# Code Quality Review

Review code changes on a branch, PR, or full repo for quality, reuse, and efficiency. Applies code style quality gates, code simplification, and clean code principles. Invoke on a branch, PR, or use `--repo` for full-repo refactoring.

## Constraints

- **Commit fixes but do NOT push** — user will push
- **Only touch files that are part of the branch's changes** — do not refactor unrelated code
- **Prefer smaller diffs** — do not clean up code adjacent to changes
- **If all changes are clean, say so** — do not invent work
- **No unnecessary comments, docstrings, or type annotations** on unchanged code

## Handle --help Flag

If the user passes `--help`, `-h`, or `help` as the argument, display this usage guide and DO NOT run the review:

```
/code-quality - Review code changes for quality, reuse, and efficiency

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

1. If the argument is a GitHub PR URL, extract the repo owner/name and PR number. Use `gh pr view <number> --json headRefName` to get the branch name.
2. If the argument is a branch name, use it directly.
3. If a repo path is provided, `cd` to it. Otherwise use the current working directory.
4. If `--repo` is passed, run the Full Repo Refactor mode (see below).
5. If `--deps`, `--testability`, or `--principle=<name>` is passed, run only that focused analysis mode (see Analysis Modes below).
6. **If no arguments are provided**, check the current branch:
   - If on a feature branch (not main/master/dev/develop), review the current branch vs the base branch. This is the default behavior.
   - If on main/master/dev/develop, present the user with options:
     - `--repo` — full repo refactor
     - Review the last commit
     - Review a specific branch (list recent branches)
     - Review a specific PR
   Do not proceed until the user picks an option.

## Step 1: Checkout and Identify Changes

```bash
git fetch origin <branch>
git checkout <branch>
git pull origin <branch>
git diff main...HEAD --name-only
```

Read all changed files. Skip non-code files (.changeset, .gitignore, README, etc.).

## Step 2: Dispatch Quality Gates

These code style rules apply to ALL repos. Scan the diff (`git diff main...HEAD`) for violations in **new code only** — pre-existing violations in untouched lines are out of scope.

### Code Style Gates

- **No for loops** — use `.forEach()`, `.filter()`, `.map()`, `.flatMap()`, `.some()`, `.every()` instead of `for (let i = ...)` or `for...of`
- **No else branches** — use early returns, guard clauses, `??`, and ternaries instead of `if/else`
- **Chain over intermediates** — prefer `.filter().forEach()` and `.flatMap()` chains over accumulator loops with push
- **Prefer immutable variables** — avoid reassignment; use multiple immutable declarations rather than one mutable variable that gets reassigned. Examples: `const` over `let` in TypeScript/JavaScript, `final` in Java, tuples or frozen dataclasses in Python, short-lived values over pointer reassignment in Go
- **No one-line (or two-line) functions** — a function whose body is a single statement, expression, or short composed call (a `.map().find()` chain, a trim-and-compare, a formatted string) is indirection, not abstraction. Inline it at the call site; duplicating that snippet across call sites is preferred over the extra function, **no matter how many times it repeats**. If the repeated thing is a literal value, extract a named constant, not a function. Does not apply to functions that are an exported API, an interface implementation, or a required callback signature.

### Test Gates

1. **Every test must be deterministic**
   - No `setTimeout` or timing-based assertions — await the actual promise or use fake timers
   - No loops in tests — loops hide which iteration failed, and an empty array silently passes. Assert on specific indices (`result[0]`, `result[1]`) or use `.every()` / `.forEach()` with a length guard. If you need to verify a collection, assert on the full array with `.toEqual()`
   - No mutable state (`callCount`, flags) with if/else in mocks — chain `mockResolvedValueOnce` / `mockReturnValueOnce` instead
   - Import and use the service's existing constants — never hardcode string/number values that already exist as named exports. Use the same constants the source code uses.

2. **Every test must be declarative**
   - Readable without mentally simulating state
   - Description matches exactly what's asserted
   - Arrange → Act → Assert structure, all three visible in the test body

3. **Every test must be able to fail**
   - Don't assert on a value you just set on the same object (tautology)
   - Don't assert a string constant equals its own hardcoded value
   - Don't just parse a valid object against a schema and assert `success === true` — exercise the real execution path and verify side effects

4. **Scope**
   - One behavior per test
   - Test the contract (inputs → outputs/side effects), not implementation
   - Cover meaningful edge cases: empty inputs, missing optional fields, error paths

These rules prevent silent-pass bugs (empty array loops, tautological assertions), flaky tests (timing-dependent), and hard-to-read tests (mutable mock state, missing arrange/act/assert).

- Use the project's native mocking framework

### How to Check

Run these searches against changed files only:

```bash
# Get list of changed files
git diff main...HEAD --name-only | grep -v '.changeset\|.gitignore'

# Check for violations in the actual diff (new lines only)
git diff main...HEAD | grep '^+' | grep '\bfor\s*('  # for loops
git diff main...HEAD | grep '^+' | grep '\belse\b'   # else branches
git diff main...HEAD | grep '^+' | grep '\blet\b'    # let declarations
```

Report violations as a list. Distinguish between new code violations (must fix) and pre-existing violations in touched files (note but don't fix — prefer smaller diffs).

## Step 3: Code Simplification

If `/simplify` is available, invoke it. Otherwise, run the model's equivalent code simplification command (e.g., Gemini's built-in code review, Codex's refactor mode). As a last resort, perform the following three reviews manually:

1. **Code Reuse Review (do this first)** — see the theme below.
2. **Code Quality Review** — redundant state, parameter sprawl, copy-paste, leaky abstractions, stringly-typed code, unnecessary comments
3. **Efficiency Review** — redundant computations, missed concurrency, hot-path bloat, recurring no-op updates, memory concerns

Aggregate findings. Fix actionable issues directly. Skip false positives with a brief note.

### Reuse before addition — the central theme

The point is not merely to de-duplicate the newly-added lines. It is to verify that an existing helper, function, or code path does not already solve the same problem — or solve a similar problem similarly enough to serve with a small change. New code that reimplements something the codebase already does is a defect even when it duplicates nothing *within* the diff.

For each new block, function, or code path, find the closest existing code that already does this (search siblings in the same file, the same service, and shared helper/util modules). Then satisfy the requirement with the least new code, in this order of preference:

1. **Reuse the existing path unchanged** — call the existing function/helper as-is. Best outcome: zero new logic, nothing to keep in sync. (Example: a new "send this notification" path calling the existing notification-dispatch helper instead of hand-rolling the transport lookup and send.)
2. **Reuse it with a small addition** — one extra parameter, an optional flag, or a single new branch on the existing path, when that does not distort its responsibility.
3. **Extract a shared path** — when the new code and an existing block (often a sibling added in the same or an adjacent change) solve the same problem the same way, factor the common part into one helper both call. (Example: an automatic-retry block that repeats the manual-retry block's validate/execute/fallback sequence, extract one helper, call it from both.)

Two failure modes to catch specifically:
- **Reimplements an existing path** — a hand-rolled copy of what a shared wrapper/util already provides. Redirect to option 1 or 2.
- **Duplicates a sibling block** — a second near-copy of a block that already exists nearby. Redirect to option 3.

Prefer the option that removes the duplication while changing the fewest existing call sites. But do not create a helper just to satisfy this when the shared body is one or two lines — see the small-function threshold under DRY; inline duplication wins there.

## Step 4: Clean Code Principles

Apply all 8 clean code principles to changed files. Verify new code follows these principles:

### 1. Meaningful Names

Names reveal intent without requiring comments.

**Check for:** Single-letter variables (except loop counters), generic names (`data`, `info`, `temp`, `result`, `handler`), unclear abbreviations, booleans not phrased as questions.

```typescript
// Bad                          // Good
const d = new Date();           const createdAt = new Date();
const u = getUser();            const currentUser = getUser();
let flag = true;                let isEnabled = true;
```

### 2. No Side Effects

Functions either perform an action OR return data, not both (Command-Query Separation).

**Check for:** Functions returning values while modifying external state, mutating input parameters, hidden I/O in pure-looking functions.

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

Every piece of knowledge has a single, authoritative representation.

**Check for:** Duplicate code blocks (3+ similar lines), repeated magic numbers/strings, same validation logic in multiple places, scattered configuration values.

```typescript
// Bad - scattered                // Good - centralized
// file1.ts: MENU_WIDTH = 400    export const Layout = {
// file2.ts: menuWidth = 400       menu: { width: 400, height: 300 },
// file3.ts: width: 400,           editor: { width: 800 }
                                 } as const;
```

**Small-function threshold — do not over-DRY.** The size of the extracted body is a hard floor that overrides repetition count. A helper whose body would be one or two lines — a single composed call (`.map().find()`), a trim-and-compare, a formatted string — must NOT be introduced purely for reuse, no matter how many times the snippet repeats. Inlining it at each call site is clearer than the indirection and reads better in the diff. Repetition frequency does not lift this floor. Neither does "authoritative knowledge" when the shared thing is trivial logic: if what must stay consistent is a *value* (a magic number, a URL, a wire constant), extract a named constant, not a function. Extract a helper only when its body is 3 or more lines of real logic that would otherwise be genuinely duplicated. This is the "No one-line functions" gate applied to DRY: a two-line composed call repeated four times is still four cheap inline copies, not a missing abstraction.

### 4. Single Responsibility

Each module/class/file has only one reason to change.

**Check for:** Files handling multiple unrelated concerns, classes mixing data access + business logic + presentation, services spanning multiple domains.

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

Code is self-documenting. Comments explain *why*, never *what*.

**Check for:** Comments explaining what code does, commented-out code, TODO/FIXME without ticket references.

```typescript
// Bad: Loop through users and check if active
for (const user of users) { if (user.isActive) { } }

// Good: Filter active users - inactive migrated to legacy (JIRA-1234)
const activeUsers = users.filter(user => user.isActive);
```

**A comment earns its bytes only by carrying knowledge the code cannot.** Calibrate to a reader who is fluent at reading code and never skims — increasingly that reader is a model, not a human, which means every comment is ingested every time the file is in context. Precision matters *more*, not less: a good comment lands reliably, a redundant one taxes reliably. A comment is worth keeping only when it says something the code itself cannot:
- **why** a non-obvious approach was chosen,
- an **invariant** the code must preserve,
- a **gotcha / hazard** (rate limits, ordering, thread/actor safety, a value that must stay in sync elsewhere),
- a **cross-reference** ("mirrors X", "matches Y"),
- **domain knowledge** not evident from the names,
- the **provenance / freshness** of data (why an entry exists, "as of <date>"),
- a doc-comment that conveys a type/function's **purpose** when the name alone doesn't.

Keep navigation markers (e.g. `// MARK:`) — cheap and aid jumping. Remove the rest. Highest-yield removals in practice: boilerplate doc-comments that paraphrase the signature ("Returns a new X with the Y set"), field-group / section labels that just repeat the field names below them, unit annotations on already-named constants (`// 1 hour` on a `3600`), and arrange/act/assert narration in tests. Be conservative when unsure — over-keeping a borderline "why" costs a few tokens; over-deleting loses knowledge that cannot be recovered from the code.

**Prefer an enforced invariant over a described one.** A comment is advisory; a test or a type is load-bearing. When a comment states an invariant ("must stay deduplicated", "these two must match", "returns exactly N"), consider encoding it as an assertion or a type so a future change that breaks it fails loudly instead of silently contradicting the prose. Comment the *why*; let a test guard the *what*.

**Comment accuracy near the diff.** For every comment that is added, modified, or sits within ~5 lines of changed code, verify it still describes the code correctly after the update. Stale comments are worse than no comments — they actively mislead.

**Check for:**
- Comment refers to a parameter, return value, branch, or behavior that no longer exists or has changed.
- Comment describes the *old* algorithm/intent after a refactor.
- Comment mentions a function/file/ticket that was renamed, moved, or closed.
- Comment claims an invariant or precondition the new code no longer guarantees.
- Comment count/example is now wrong (e.g., "returns 3 fields" when it now returns 4).
- JSDoc/docstring `@param`, `@returns`, or type description doesn't match the current signature.

```typescript
// Before the change
// Returns the user's active subscriptions, sorted by created date
function getSubscriptions(userId: string): Subscription[] { ... }

// After adding a status filter param — comment is now stale (lies about behavior)
// Bad: comment unchanged
// Returns the user's active subscriptions, sorted by created date
function getSubscriptions(userId: string, status: Status): Subscription[] { ... }

// Good: comment updated or removed
function getSubscriptions(userId: string, status: Status): Subscription[] { ... }
```

**How to check:** for each hunk in `git diff main...HEAD`, read the surrounding context (the file at HEAD, ±5 lines around each changed range). Flag any comment whose claim no longer matches the current code. Fix by updating the comment to reflect the new behavior or, when the comment was just restating the code, deleting it.

**Code paragraphs.** Function bodies should read as a sequence of paragraphs, each representing one conceptual step (a guard, a normalization, a derivation, a side effect, a write). Paragraphs are separated by a single blank line, and non-trivial paragraphs are led by a brief 1–2 line comment that names the step. This complements minimal-comments: don't comment individual lines; *do* comment conceptual paragraphs where the topic sentence saves a reader from re-deriving intent.

Three rules:

1. **Paragraph boundaries are conceptual, not syntactic.** A `try/catch` wrapping an external call is one paragraph. The lines that build a request payload are one paragraph. Don't break paragraphs mid-step just to add whitespace.
2. **The comment is a topic sentence.** One line preferred, two if a tradeoff or non-obvious mechanism needs to fit. No blank line between the comment and its code; one blank line above the comment, separating from the prior paragraph.
3. **Skip the comment when the paragraph is self-evident from naming.** A guard whose names already explain themselves doesn't need a heading. The comment is for the steps where intent isn't already legible — typically when *why* the step exists is non-obvious from the code.

```typescript
// Good — function body reads as paragraphs, each headed by intent (or skipped when obvious)
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

The "normalize current" paragraph needs no comment — names tell the story. The "normalize previous" paragraph gets one because *why* it exists isn't visible from the code. Payload-build and try/catch are self-evident.

**Check for:**
- Long function bodies with no blank lines — flag as one undifferentiated wall of code; suggest paragraph breaks at conceptual boundaries.
- Blank lines *inside* what is conceptually one paragraph (mid-step gaps that disrupt flow).
- Multi-line conceptual steps that perform a non-obvious mechanism but have no leading comment.
- Comments that should have a blank line above them (separating from the prior paragraph) but don't.
- Single-line trivial steps that have unnecessary topic comments restating what the code already says.

### 6. Consistent Formatting

Related code grouped together with consistent style.

**Check for:** Mixed naming conventions, inconsistent indentation, lines > 100-120 chars, inconsistent import ordering.

### 7. Error Handling

Use exceptions properly, avoid returning null, separate error handling from business logic.

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

Code designed for unit testing in isolation.

**Check for:** Hard-coded dependencies (`new` internally), direct DB/API calls in logic, scattered `process.env` access, non-deterministic calls (`Date.now()`, `Math.random()`).

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

## Step 5: Lint

Check the project's `package.json` for lint commands (e.g., `lint`, `lint:eslint`, `lint:types`). Run whatever the project uses. Fix any lint errors found.

## Step 6: Tests

Run the project's test runner against affected test files.

Run only test files that are part of the branch's changes or directly test changed code. If a test failure is pre-existing (verify by stashing changes and re-running), note it as pre-existing infrastructure issue.

## Step 7: Fix and Re-verify

If any issues were found in steps 2-6:
1. Apply fixes directly to the code
2. Re-run lint on fixed files
3. Re-run tests on affected test files
4. Confirm fixes don't introduce new issues

## Step 8: Report

Provide a structured summary:

```
## Code Quality Report — PR #<number> (<branch-name>)

### Verdict: <Clean | Clean with N fixes applied | N issues found>

### Fixes Applied
- <description of each fix and why>

### Code Style Gates
- For loops: <pass/fail — details>
- Else branches: <pass/fail — details>
- Const over let: <pass/fail — details>

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

### Lint & Tests
- ESLint: <clean | N errors>
- Tests: <N/N pass | failures>

### Review Notes (no action needed)
- <observations worth noting but not worth fixing>
```

## Analysis Modes

### Default: Full Review
All steps above — quality gates, code simplification, clean code principles, lint, tests.

### Full Repo Refactor (`--repo`)

Refactor the entire repo for code quality. Logic and tests are kept in **separate commits** so they can verify each other.

**Critical rule: never change logic and tests in the same commit.** Tests confirm logic changes. Logic confirms test changes. If you change both at once, neither can verify the other.

**Workflow:**

1. **Scan the full repo** — identify all source files and test files. Categorize them.
2. **Run tests first** — establish a passing baseline. If tests don't pass before you start, stop and report.
3. **Phase 1: Refactor logic** — apply all quality gates and clean code principles to source files only. Do NOT touch test files. After each logical grouping of changes:
   - Run the full test suite to confirm nothing broke
   - Commit only the source file changes (e.g., "refactor: apply clean code principles to auth module")
   - If tests fail, revert and fix before continuing
4. **Phase 2: Refactor tests** — apply test gates and clean code principles to test files only. Do NOT touch source files. After each logical grouping of changes:
   - Run the full test suite to confirm tests still pass
   - Commit only the test file changes (e.g., "test: refactor auth module tests for clarity")
   - If tests fail, the test refactor introduced a bug — fix the test, not the source
5. **Repeat** — continue alternating phases until the repo is clean
6. **Final verification** — run the full test suite and lint one last time

**Commit discipline:**
- Logic commits contain ONLY source files
- Test commits contain ONLY test files
- Never mix them — this is the whole point
- Each commit should pass the test suite independently

### Module Dependencies (`--deps`)
Check for: circular dependencies, god modules (imported everywhere), orphan modules, layer violations.

### Testability (`--testability`)
Check for: hard-coded instantiation, global state access, non-deterministic calls, missing interfaces.

### Single Principle (`--principle=<name>`)
Deep analysis on one clean code principle with line numbers and specific fix suggestions.

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

When you're done with the entire review (and any commits), the very last thing you should do is say `meow` on its own line.

