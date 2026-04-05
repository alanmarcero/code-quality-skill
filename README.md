# Code Quality Skill

A code quality skill for AI coding agents. Reviews PRs, branches, or full repos for code style violations, test quality, clean code principles, reuse, and efficiency — then fixes issues directly. Includes quality gates, test gates, grep patterns, scored reporting, and analysis modes for dependencies, testability, and full-repo refactoring. Language and model agnostic.

## What It Does

1. **Code Style Gates** — checks for `for` loops, `else` branches, mutable variables (`let` over `const`), and accumulator patterns in new code
2. **Test Gates** — checks test quality: determinism (no timing, no empty-array loops, no mutable mock state), declarative structure (arrange/act/assert), ability to fail (no tautologies), proper scoping, and use of service constants over hardcoded values
3. **Clean Code Principles** — applies 8 principles: meaningful names, no side effects, DRY, single responsibility, minimal comments, consistent formatting, error handling, testable code
4. **Code Simplification** — runs `/simplify` or the model's equivalent for reuse, quality, and efficiency review
5. **Lint & Tests** — runs the project's lint and test commands on affected files
6. **Fixes Issues** — applies fixes directly, re-verifies, then commits
7. **Scored Report** — outputs a structured report with per-principle scores, gate results, and a verdict

## Install

### Claude Code

Copy the skill to your global skills directory:

```bash
mkdir -p ~/.claude/skills/code-quality
cp SKILL.md ~/.claude/skills/code-quality/SKILL.md
```

### Other AI Coding Agents

The skill is a markdown file that any AI agent can follow as instructions. Point your agent at `SKILL.md` or paste its contents as a system prompt.

## Usage

```
/code-quality <PR-url>
/code-quality <branch-name>
/code-quality <branch-name> <repo-path>
/code-quality --repo                    Full repo refactor (logic and tests in separate commits)
/code-quality --deps                    Focus on module dependencies
/code-quality --testability             Focus on unit testability
/code-quality --principle=<name>        Focus on one clean code principle
```

When invoked with no arguments on a feature branch, it reviews the current branch against its base. On main/master/dev, it presents options.

## Analysis Modes

- **Default** — full review: code style gates, test gates, clean code principles, code simplification, lint, tests
- **`--repo`** — full repo refactor with logic and test changes in separate commits so each verifies the other
- **`--deps`** — circular dependencies, god modules, orphan modules, layer violations
- **`--testability`** — hard-coded instantiation, global state, non-deterministic calls
- **`--principle=<name>`** — deep analysis on one principle (e.g., `--principle=dry`)

## Code Style Gates

Applied to new code in the diff only. Pre-existing violations in untouched lines are noted but not fixed.

- **No for loops** — use `.forEach()`, `.filter()`, `.map()`, `.flatMap()`, `.some()`, `.every()`
- **No else branches** — use early returns, guard clauses, `??`, ternaries
- **Chain over intermediates** — prefer `.filter().forEach()` and `.flatMap()` over accumulator loops
- **Prefer immutable variables** — `const` over `let` (JS/TS), `final` (Java), tuples/frozen dataclasses (Python), etc.

## Test Gates

Applied to test files in the diff. Prevents silent-pass bugs, flaky tests, and hard-to-read tests.

1. **Deterministic** — no `setTimeout`/timing assertions, no loops in tests (hide which iteration failed, empty arrays silently pass — assert on specific indices or full array with `.toEqual()`), no mutable state with if/else in mocks (use `mockResolvedValueOnce` chains), use service constants not hardcoded values
2. **Declarative** — readable without simulating state, description matches assertion, visible arrange/act/assert
3. **Able to fail** — no tautological assertions, no asserting on values you just set, no schema-only validation without exercising real logic
4. **Scoped** — one behavior per test, test contracts not implementation, cover meaningful edge cases

## Clean Code Principles

| # | Principle | What It Checks |
|---|-----------|----------------|
| 1 | Meaningful Names | Single-letter vars, generic names, unclear abbreviations |
| 2 | No Side Effects | Functions that return AND mutate, hidden I/O |
| 3 | DRY | Duplicate blocks, repeated magic values, scattered config |
| 4 | Single Responsibility | Mixed concerns, god classes, multi-domain services |
| 5 | Minimal Comments | Comments explaining "what", commented-out code, TODOs without tickets |
| 6 | Consistent Formatting | Mixed conventions, inconsistent indentation, import ordering |
| 7 | Error Handling | Returning null for errors, empty catch blocks, missing async handling |
| 8 | Testable Code | Hard-coded deps, direct DB calls in logic, scattered env access |

## Full Repo Refactor (`--repo`)

Refactors the entire repo with logic and test changes in **separate commits** so each can verify the other. Never changes logic and tests in the same commit. Establishes a passing test baseline first, then alternates between source-only and test-only refactoring phases.

## License

MIT
