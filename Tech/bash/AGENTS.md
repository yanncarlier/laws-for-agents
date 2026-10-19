# AGENTS.md

Development guidelines for AI coding agents working in this repository.
Humans are welcome to read it too — it is the single source of truth for how we build here.

> **Scope:** These rules apply to the whole repo. A nested `AGENTS.md` in a subdirectory overrides this file for that subtree.

---

## 1. Project overview

| Field | Value |
| --- | --- |
| Name | `<project-name>` |
| Shell | Bash `>=5.0` (not `sh`, not `zsh` — see §2 for why) |
| Target OS | `<linux / macOS / both>` |
| Linter | `shellcheck` |
| Formatter | `shfmt` |
| Test framework | `bats-core` (Bash Automated Testing System) |
| CI | `<github-actions / gitlab-ci / none>` |

Replace the `<...>` placeholders before relying on this file.

---

## 2. Setup

```bash
# verify Bash version — scripts assume 5.0+ (associative arrays, mapfile, nameref)
bash --version

# install dev tooling
# macOS:
brew install shellcheck shfmt bats-core
# Debian/Ubuntu:
sudo apt-get install -y shellcheck shfmt
# bats-core via git submodule if not packaged, see test/bats/README.md

# make hooks executable (git does not preserve the bit on clone in some setups)
chmod +x scripts/*.sh hooks/*.sh
```

If a script must run under POSIX `sh` (e.g. an Alpine container's `/bin/sh` or a package pre/post-install script), say so explicitly at the top of that file and hold it to the POSIX subset in §11 instead of this document.

---

## 3. Commands

| Task | Command |
| --- | --- |
| Lint all scripts | `./scripts/lint.sh` (wraps `shellcheck **/*.sh`) |
| Format check | `shfmt -d .` |
| Format + write | `shfmt -w .` |
| Run test suite | `bats test/` |
| Run one test file | `bats test/deploy.bats` |
| Full pre-push gate | `./scripts/verify.sh` (lint → format check → test) |

`./scripts/verify.sh` must pass before any change is considered done. Every script in the repo must pass `shellcheck` with zero warnings (suppressions require the justification rule in §12).

---

## 4. Directory structure

```
.
├── bin/                 # user-facing entrypoints, no .sh extension, on PATH when installed
├── scripts/             # internal automation: build, deploy, lint, verify, ci helpers
├── lib/                 # sourced function libraries — never executed directly
│   ├── log.sh           # logging helpers
│   ├── check.sh         # dependency / environment checks
│   └── trap.sh          # shared cleanup/trap helpers
├── test/
│   ├── *.bats           # bats test files, one per script under test
│   └── fixtures/        # sample input files, mock binaries on PATH
├── hooks/               # git hooks, installed via scripts/install-hooks.sh
└── AGENTS.md
```

- Files in `lib/` are sourced, never executed: they must not have a shebang run directly in CI, and must not call `exit` (use `return`).
- Files in `bin/` and `scripts/` are executed, not sourced: each has a shebang and is independently runnable.
- A script that grows past ~150 lines or accumulates more than a couple of independent responsibilities should have its logic split into `lib/` functions with a thin entrypoint left in `bin/`/`scripts/`.

---

## 5. Script header (required in every executable script)

```bash
#!/usr/bin/env bash
#
# Usage: deploy.sh [-e ENV] [-n] TARGET
#
# Deploys TARGET to the environment given by -e (default: staging).
# -n performs a dry run without making changes.
#
# Requires: aws-cli >=2, jq
set -euo pipefail
IFS=$'\n\t'

# Resolve the script's own directory even if it's symlinked or sourced elsewhere.
SCRIPT_DIR="$(cd -- "$(dirname -- "${BASH_SOURCE[0]}")" &>/dev/null && pwd)"
```

- `#!/usr/bin/env bash`, never a hardcoded `#!/bin/bash` — the former respects `PATH` and works on systems where Bash 5 lives outside `/bin`.
- `set -euo pipefail` is mandatory at the top of every executable script:
  - `-e` — exit on any unhandled command failure.
  - `-u` — treat unset variables as an error, catching typos.
  - `-o pipefail` — a pipeline fails if any stage fails, not just the last.
- `IFS=$'\n\t'` prevents word-splitting surprises on spaces in filenames/arguments. Only omit it when a script deliberately relies on space-splitting, and say why in a comment.
- A short usage comment block at the top doubles as documentation and as the text an `-h`/`--help` flag should print.

---

## 6. Quoting and variables

- **Quote every variable expansion**: `"$var"`, `"${array[@]}"`, `"$(cmd)"`. Unquoted expansion is the single most common source of bugs and is a `shellcheck` error (SC2086) — never suppress it, fix it.
- Use `"${var}"` (braces) when concatenating against adjacent text (`"${name}.log"`) or inside more complex parameter expansions; plain `"$var"` is fine standalone.
- Always use `[[ ... ]]` for conditionals, never `[ ... ]` or `test`. `[[` doesn't word-split or glob-expand its operands and supports `&&`, `||`, `=~` directly.
- Use `(( ... ))` for arithmetic comparisons and assignment, not `[[ "$a" -gt "$b" ]]` or `expr`.
- Declare variables with the narrowest appropriate scope: `local` inside every function, `readonly`/`declare -r` for constants, `declare -A` for associative arrays.
- Name constants and exported environment variables `UPPER_SNAKE_CASE`; everything else `lower_snake_case`. No bare globals inside functions — every function-local variable is `local`.
- Prefer `${var:-default}`, `${var:?error message}`, and `${var:+alt}` over manual `if`-checks for defaults and required-variable validation.

```bash
# Good
local retries="${1:-3}"
readonly LOG_DIR="${LOG_DIR:?LOG_DIR must be set}"
[[ -n "${DRY_RUN:-}" ]] && echo "dry run"

# Bad — unquoted, uses [ ], uses expr
retries=$1
if [ -z $LOG_DIR ]; then echo "LOG_DIR must be set"; exit 1; fi
count=`expr $count + 1`
```

---

## 7. Functions

- Every reusable piece of logic is a function; scripts over ~30 lines should have a `main()` at the bottom, invoked as the final line: `main "$@"`.
- Every function starts with `local` declarations for its parameters — never read `$1`/`$2` scattered mid-body.
- Functions return status via `return 0`/`return 1` (checked with `if func; then` or `$?`), and return data via `stdout` captured with `$(...)`, never by leaking variables into the caller's scope.
- No function relies on global mutable state it didn't receive as a parameter, except for the handful of `readonly` constants declared at the top of the script.

```bash
log_error() {
  local message="$1"
  printf 'ERROR: %s\n' "$message" >&2
}

is_valid_env() {
  local env="$1"
  [[ "$env" =~ ^(dev|staging|prod)$ ]]
}
```

---

## 8. Control flow and command substitution

- Use `$(...)` for command substitution, never backticks — it nests cleanly and is unambiguous.
- Check command success with `if cmd; then` / `if ! cmd; then`, not by comparing `$?` after the fact, except where the exit code itself is data (then capture it immediately: `rc=$?`).
- Use `mapfile -t arr < <(cmd)` to read command output into an array — never `arr=($(cmd))`, which word-splits and glob-expands unpredictably.
- Never parse `ls` output. Use globs, `find -print0` piped to `xargs -0` or `while IFS= read -r -d ''`, or `mapfile`.
- Prefer a `case` statement over a long `if/elif` chain when matching against a fixed set of string values.
- For loops over lines, always `while IFS= read -r line; do ... done < file` — the `IFS=` and `-r` are both required to avoid stripping whitespace and mangling backslashes.

```bash
# Good
while IFS= read -r -d '' file; do
  process "$file"
done < <(find . -name '*.log' -print0)

# Bad — breaks on spaces, newlines, and glob characters in filenames
for file in $(find . -name '*.log'); do
  process $file
done
```

---

## 9. Error handling and cleanup

- Trap cleanup on exit, not just on the happy path: `trap cleanup EXIT` (and `INT TERM` when the script does long-running work), where `cleanup()` removes temp files, kills background jobs, and restores anything it changed.
- Create temp files/dirs with `mktemp`/`mktemp -d`, never a hardcoded path in `/tmp`.
- On a fatal condition, print a clear message to `stderr` and exit with a specific non-zero code — don't let the script die on an unrelated command with no context.
- Reserve exit codes deliberately and document them in the usage header when a caller might branch on them (e.g. `2` = bad usage, `3` = missing dependency, `4` = network failure).
- Validate all required external commands exist before using them: `command -v jq &>/dev/null || { echo "jq is required" >&2; exit 3; }`.

```bash
readonly TMP_DIR="$(mktemp -d)"
cleanup() {
  rm -rf -- "$TMP_DIR"
}
trap cleanup EXIT INT TERM
```

---

## 10. Input, arguments, and side effects

- Parse flags with `getopts` for POSIX-style short options, or a manual `while [[ $# -gt 0 ]]; case "$1" in ... esac` loop for long options (`getopts` alone doesn't support `--long-flags`).
- Validate argument count and values before acting on them; print usage and exit non-zero on invalid input rather than proceeding with a guess.
- A script that deletes, overwrites, or deploys something supports a `-n`/`--dry-run` flag that prints what it would do without doing it, and a destructive default action requires either an explicit flag or an interactive confirmation — never silently destructive by default.
- Never `eval` user-controlled or externally-sourced input. If dynamic execution is unavoidable, constrain it to a fixed allowlist of known-safe commands and say why in a comment.
- Don't hardcode secrets, tokens, or credentials in a script. Read them from environment variables or a secrets manager, and never `echo`/log their values.

---

## 11. Portability

- Default target is Bash 5+; use Bashisms freely (`[[`, arrays, `local`, process substitution, `mapfile`) rather than crippling scripts for `sh` compatibility that isn't needed.
- If a script genuinely must run under POSIX `sh`: no arrays, no `[[`, no `local` (or guard it), no process substitution, use `command -v` not `type`, and say `#!/bin/sh` plus a comment noting the POSIX constraint at the top.
- Don't assume GNU-only flag behaviour (`sed -i`, `date -d`, `readlink -f`) without checking `<Target OS>` in §1 — GNU and BSD (macOS) coreutils differ on these. When both must be supported, detect the platform once at the top of the script and branch, rather than sprinkling `uname` checks through the body.
- Don't assume a particular `PATH` or working directory. Resolve paths relative to `SCRIPT_DIR` (see §5), and `cd` back or use subshells `( cd dir && cmd )` instead of leaving the caller's shell in a different directory.

---

## 12. Shellcheck and suppressions

- `shellcheck` runs clean (no warnings) on every script in CI. A warning is either fixed or explicitly suppressed with a reason — never left unaddressed.
- A suppression is a last resort, placed on the line it applies to, with a one-line justification:
  ```bash
  # shellcheck disable=SC2086 # intentional word-splitting: $FLAGS holds multiple space-separated args
  some_command $FLAGS
  ```
- Suppressing SC2086 (unquoted variable) is almost never correct — treat any instance as a bug first, a suppression only after genuine, documented investigation.
- Run `shfmt -d .` and fix formatting rather than hand-aligning indentation; don't fight the formatter's output in review.

---

## 13. Testing

- Every script in `bin/` and `scripts/` has a corresponding `test/<name>.bats` file.
- Structure each test as Arrange / Act / Assert using `bats`' `run` helper, and assert on both `$status` and `$output`:
  ```bash
  @test "deploy.sh rejects an unknown environment" {
    run ./scripts/deploy.sh -e bogus target
    [ "$status" -eq 2 ]
    [[ "$output" == *"invalid environment"* ]]
  }
  ```
- Never let a test touch real infrastructure, the real filesystem outside a `mktemp` sandbox, or the network. Stub external commands by prepending a fixture directory of fake binaries to `PATH` in `setup()`, and restore it in `teardown()`.
- Test failure paths as thoroughly as the happy path: missing arguments, missing dependencies, non-zero exit from a wrapped command.
- Every bug fix ships with a regression test that fails before the fix.

---

## 14. Git and pull requests

- Branches: `feat/short-description`, `fix/short-description`, `chore/...`.
- Commits follow [Conventional Commits](https://www.conventionalcommits.org/): `fix(deploy): quote target path to handle spaces`.
- PR description states what changed, why, how it was verified (paste the `verify.sh` output), and calls out any behavioural change to a script's flags, exit codes, or defaults.
- Never force-push to `main`. Never amend a commit others may have pulled.
- Do not commit: generated log output, local `.env` files, editor config, anything written by a script's own runtime into `test/fixtures` or `/tmp`.

---

## 15. Rules for agents

**Do**
- Read the surrounding script before editing; match its existing patterns over the ones in this file when they conflict, and flag the conflict.
- Run `shellcheck` and `bats` on any script you touch, and report the actual output before claiming success.
- Make the smallest change that fully solves the problem.
- Add a `-n`/`--dry-run` path when adding any new destructive behaviour to an existing script.
- Ask before: changing a script's flags, exit codes, or default (non-dry-run) behaviour in a way that could break an existing caller or cron job.

**Don't**
- Don't write a Bashism into a script whose header declares POSIX `sh`, or vice versa introduce `sh`-only crippling into a Bash 5 script.
- Don't suppress a `shellcheck` warning without a one-line justification comment.
- Don't parse `ls`, use unquoted expansions, or use backticks — fix these on sight even outside the immediate task if they're trivial and in a line you're already touching; otherwise leave unrelated code alone and flag it.
- Don't add a new external binary dependency without checking it's already assumed elsewhere in the repo or documenting it in the script's usage header.
- Don't mark work complete with a failing `shellcheck`, a failing `bats` run, or an unformatted diff (`shfmt -d`).

---

## 16. Definition of done

1. `./scripts/verify.sh` passes locally (lint, format check, tests).
2. `shellcheck` reports zero warnings on every touched file, with any suppression justified.
3. New or changed behaviour has a `bats` test; bug fixes include a regression test.
4. Every variable expansion is quoted; `[[`, `$(...)`, and `local` are used throughout.
5. Destructive scripts support `-n`/`--dry-run` and fail safely on bad input.
6. The PR documents any change to a script's flags, exit codes, or defaults.
