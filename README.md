# Code Quality Skill

A code quality skill for AI coding agents. It reviews a PR, branch, or full repo for code style, test quality, clean code principles, and reuse, then reports the issues. It fixes them when you ask. It includes quality gates, test gates, grep patterns, a scored report, and modes for dependencies, testability, and full-repo refactoring. It works with any language and model.

## What It Does

1. **Code Style Gates**: flags `for` loops, `else` branches, nested ternaries, `let` over `const`, and accumulator patterns in new code
2. **Test Gates**: checks that tests are deterministic, declarative, able to fail, and scoped to one behavior
3. **Reuse**: checks that no existing code path already solves the problem
4. **Clean Code Principles**: applies 8 principles, from meaningful names to testable code
5. **Diff Reduction**: shrinks the diff without breaking a gate
6. **Lint & Tests**: runs the project's lint and test commands on affected files
7. **Report and Fix**: reports per-principle scores, gate results, and a verdict. When you ask, it applies fixes, re-verifies, and commits

## Install

### Claude Code

```bash
mkdir -p ~/.claude/skills/code-quality
cp SKILL.md ~/.claude/skills/code-quality/SKILL.md
```

### Other AI Coding Agents

`SKILL.md` is plain Markdown instructions. Point your agent at it, or paste it as a system prompt.

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

With no arguments on a feature branch, it reviews the branch against its base. On main/master/dev, it offers options.

## Analysis Modes

- **Default**: the full review
- **`--repo`**: full repo refactor, with logic and test changes in separate commits so each verifies the other
- **`--deps`**: circular dependencies, god modules, orphan modules, layer violations
- **`--testability`**: hard-coded instantiation, global state, non-deterministic calls
- **`--principle=<name>`**: deep analysis of one principle (e.g., `--principle=dry`)

## Code Style Gates

The gates apply to new code in the diff. The report notes violations on untouched lines but does not fix them.

- **No for loops**: use `.forEach()`, `.filter()`, `.map()`, `.flatMap()`, `.some()`, `.every()`
- **No else branches**: use early returns, guard clauses, `??`, ternaries
- **Chain over intermediates**: prefer `.filter().forEach()` and `.flatMap()` over accumulator loops
- **Prefer immutable variables**: `const` over `let` (JS/TS), `final` (Java), tuples or frozen dataclasses (Python)

## Test Gates

1. **Deterministic**: no timing assertions, no loops in tests (an empty array passes silently), no mutable mock state, and service constants instead of hardcoded values
2. **Declarative**: readable without simulating state, the description matches the assertion, arrange/act/assert visible
3. **Able to fail**: no tautological assertions, no schema-only checks that skip the real logic
4. **Scoped**: one behavior per test, the contract and not the implementation, meaningful edge cases

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

## License

MIT
