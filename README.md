# Code Quality Skill

A code quality skill for AI coding agents. Reviews PRs and branches for code style violations, clean code principles, reuse, and efficiency — then fixes issues directly. Includes quality gates, grep patterns, scored reporting, and analysis modes for dependencies and testability. Language and model agnostic.

## What It Does

1. **Code Style Gates** — checks for `for` loops, `else` branches, `let` over `const`, and accumulator patterns
2. **Clean Code Principles** — applies 8 principles: meaningful names, no side effects, DRY, single responsibility, minimal comments, consistent formatting, error handling, testable code
3. **Reuse & Efficiency** — runs parallel review for code reuse, quality violations, and performance issues
4. **Lint & Tests** — runs the project's lint and test commands
5. **Fixes Issues** — applies fixes directly, re-verifies, then commits
6. **Scored Report** — outputs a structured report with per-principle scores and a verdict

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
/code-quality --deps                    Focus on module dependencies
/code-quality --testability             Focus on unit testability
/code-quality --principle=<name>        Focus on one clean code principle
```

## Analysis Modes

- **Default** — full review: quality gates, clean code principles, lint, tests
- **`--deps`** — circular dependencies, god modules, orphan modules, layer violations
- **`--testability`** — hard-coded instantiation, global state, non-deterministic calls
- **`--principle=<name>`** — deep analysis on one principle (e.g., `--principle=dry`)

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
