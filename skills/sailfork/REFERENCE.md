# sailfork CLI reference

Flags: `sailfork <command> --help`, or `sailfork commands --json` for every command. `sf` is the same binary. For `pr`
commands `N` defaults to the current branch's open pull request.

Every command: `-R OWNER/REPO` (env `SAILFORK_REPO`), `--host URL`, `--json[=a,b]`, `--jq EXPR`, `--summary`, and a
64 KiB output cap for agents (`--max-bytes 0` lifts it).

Exit codes: 0 ok, 1 failed, 2 usage, 3 signed out, 4 not found, 5 forbidden, 6 conflict, 7 network, 8 rejected
input, 9 retry later, 10 precondition, 11 waiting on a person, 12 pending or timed out. Under `--json` a failure is
one stderr object with `code`, `hint` and `retryable`.

Not obvious from `--help`:

- `issue claim --minutes` takes 5 to 1440; claiming again renews.
- Notes: 400 characters each, 30 per repository; credential-shaped text is refused.
- `pr create --onto` excludes `--base`; `pr merge --stack` excludes `--keep-branch` and `--match-head-commit` and
  stops at the first failure.
- `attachment upload PATH...` prints one `![name](url)` per file to paste into a body.
