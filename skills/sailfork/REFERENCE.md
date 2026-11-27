# sailfork CLI reference

Flags: `sailfork <command> --help`, `sailfork schema <command>` for its JSON output, `sailfork commands --json` for
every command. `sf` is the same binary; for `pr` commands `N` defaults to the current branch's pull request.

## Basics

- Every command: `-R OWNER/REPO` (env `SAILFORK_REPO`), `--host URL`, `--json[=a,b]`, `--jq EXPR`, `-r`,
  `--format table|tsv|json|ndjson`, `--summary`, `--no-color`; agent output stops at 64 KiB (`--max-bytes 0`).
- Exit codes: 0 ok, 1 failed, 2 usage, 3 signed out, 4 not found, 5 forbidden, 6 conflict, 7 network, 8 rejected
  input, 9 retry later, 10 precondition, 11 held for a person, 12 pending or timed out. `sandbox exec`, `caisson
  test` and `repo test` exit with the command's own code. Under `--json` a failure is one stderr object with `code`,
  `hint` and `retryable`.
- `sailfork api PATH` (`-X`, `-f`, `-F`, `--input`, `--paginate`) reaches any REST route without a command.
- `sailfork batch` runs one command per line (MCP `batch`: one tool call per item), eight at a time, results in
  order; not atomic, and a long call holds the batch.

## Refusals and what to do

Retrying the same call, or another route to the same change, never works for these.

| Answer | Meaning | Do |
| --- | --- | --- |
| exit 11, 202 `HELD FOR A PERSON` | A merge, preview or `ask` write waits for a person | Tell the user, `wait held ID`, run it again on 0 |
| 403 `held_declined` | A person declined it; holds 24 hours on every head | Tell the user |
| 403 `bypass_needs_person` | Merging past the queue or a code owner | Only a person in the browser |
| 403 `agent_policy` | The org set agents read-only or stopped, or the token is older than its limit | Tell the user |
| 402 `agent_change_cap` | The org's monthly agent change cap | Tell the user |
| 403 `insufficient_scope` | The token lacks the scope (over MCP: the person did not tick it) | Ask for a token or connection with it |
| "needs a browser session" | Approving held requests, agent controls, merging past the queue, transfers, account deletion | A person does it |
| 409 merge rules not met | Lists every unmet rule: approvals, threads, a required check, owners, a held write | Fix what it lists |
| exit 10 on `pr merge` | Local commits the pull request lacks | `git push`, or `--force` to merge without them |
| 409 `idempotency_in_progress` | The same request is still running | Wait a second, retry once |
| 429 `Retry-After` | Writes: 30 comments, issues, PRs or reviews a minute and 200 mentions, assignments or review asks an hour, per person; MCP 600 requests a minute; code search 30 a minute | Wait, retry once, then double the wait; never in parallel |
| 503 on a search | A query over 2 seconds | Narrow it |

## Context and coordination

- **Start on one item:** `brief N` (MCP `work_context`): body, comments, threads with their code and failed checks in
  one call. `issue view N` (MCP `issue_get`) adds relations and the timeline.
- **Issue context**, the handoff the next worker reads first: `issue context N` reads it; `--state`, `--decision`,
  `--tried`, `--verified`, `--file PATH`, `--next`, `--question` / `--answered`, `--branch`, `--visibility
  private|team|repo` write it, `--clear` starts over. Update it at each milestone, not only at the end.
- **What earlier work found:** `issue related --paths P --query Q`, ranked decisions, dead ends and proofs.
- **Work graph:** `/<owner>/<repo>/graph?focus=issue-7` (or `focus=pull-12`) draws parents, sub-issues, blockers, what
  it blocks and the stacked pull requests that close them; give a person the link. The same data: `issue view N`,
  `issue list --parent N`, `pr stack PR`.
- **Who else is changing this code** (open pull requests and live caissons of everyone):
  - `inflight touching NAME [--path P]`: changes to a symbol or its callers, before you start.
  - `inflight symbols --pull N|--caisson ID`: the definitions a change adds, removes or rewrites.
  - `inflight conflicts --pull N|--caisson ID`: changes that would conflict in text, file by file.
  - `inflight collisions --pull N|--caisson ID`: changes that still use a definition this one removes or re-signs;
    they merge cleanly and break the build.
  - `inflight risk --pull N`: a score from sensitive paths, symbol churn, collisions and checks.
  - `inflight affected --pull N`: the test and build steps the change reaches.
  - `caisson intend ID NAME...`: declare what you will change so the others see it.
- **Conflicts with the base:** `pr conflicts N` lists files, `--path P --merged > F` prints one with markers, `pr
  resolve N --file P=F` or `--take head|base` resolves in one merge commit; `pr update-branch N` when clean.
- **Notifications:** `inbox --unread --tab for-you|following [--follow]`, `inbox read` once handled; `events --types
  review,comment,ci,mention,merge,push,issue,pull,caisson --follow`, saved with `events subscription
  create|list|view|advance|delete`.

## Not obvious from `--help`

**Issues and notes**
- `issue claim --minutes` takes 5 to 1440; claiming again renews.
- `issue create --batch FILE` files many at once, all or none; `{{ref}}` placeholders link them.
- `issue close --reason done|wont-do|duplicate` (`--duplicate-of N`); `issue reopen` undoes it.
- `issue ask N --kind single|multi|text --text Q --options a --options b --recommended a` (2 to 10 options, `--other
  true` adds free text); the answer comes back in the issue's context (`asks`); `issue unask N ID` withdraws it.
- Notes: 400 characters each, 30 per repository, 4,000 in all; credential-shaped text is refused.

**Pull requests and merging**
- `pr create --onto` excludes `--base`; `pr merge --stack` excludes `--keep-branch` and `--match-head-commit` and
  stops at the first failure.
- `pr merge --method merge|squash|rebase|fast-forward` (`repo edit --merge-method` sets the default);
  `--match-head-commit SHA` refuses when the branch moved; `--cancel-auto` drops a pending `--auto`.
- Held for a person: anyone with write access decides a merge, an org member a preview, never an agent. An approval
  covers that request at that exact head for 24 hours (a moved head is held again unless the org set
  `mergeApprovalFollowsPull`). Personal repositories, browser sessions and a workflow's `GITHUB_TOKEN` are never
  held. `held list` is what waits on you, `held org --org ORG` the org's.
- Merge queue: batches tested at their tip; a red batch reruns its failed jobs once, then the queue finds the culprit
  and lands a revert pull request as whoever queued it. A stack queues as one unit, bottom first. Risk never changes
  the order.
- `--method rebase` refuses a branch with merge commits or conflicts; `fast-forward` refuses when the base moved.
  `--match-head-commit` takes 7 or more hex characters.
- Owners: `OWNERS` or `CODEOWNERS` (`.github/`, the root, then `docs/`) read from the base branch, so a pull request
  cannot change who approves it; a direct push to an owned path is refused unless the pusher owns it.
- Stacks: after the bottom merges, its branch is deleted and the next retargets; one that does not carry over cleanly
  needs `pr edit N --base B`. A push to any level, or closing one, cancels a pending stack merge.
- `pr revert N [--also M]` opens one revert pull request (`repo:write` is enough); 409 when later changes touch the
  same files. `undo` cannot reverse a merge the base moved past: use `pr revert`.
- `pr github send N` opens it on a push-mirrored GitHub repository; the body gets a footer (`mirror pulls edit`).
- `attachment upload PATH...` prints one `![name](url)` per file; `--sandbox ID` takes the files from a sandbox.

**Waiting**
- `wait issue N --state open|closed|triage|ready|in-progress|in-review|done|canceled|duplicate`, `wait caisson ID
  --state editing|waiting|review|merged|stopped`; every `wait` takes `--timeout` (default 10m) and `--interval` (5s).

**CI**
- `ci list --pr N|--branch B --status pass|fail|running --workflow W`; `ci view ID --exit-status`; `ci logs ID --job J
  --step S --tail N --grep T --follow`; `ci rerun ID --failed` or `--job J` (that job and what needs it).
- `ci run WORKFLOW --ref R -f key=value` starts a `workflow_dispatch` workflow (`--json` gives the id for `wait run`).
- `ci artifacts ID --download NAME -o PATH`; `ci caches --key PREFIX --ref R`, `ci caches delete ID|--all --yes`.
- `ci secret set NAME --value -`: `--environment NAME` limits it to jobs declaring it, on default-branch runs and tags
  on a default-branch commit; `--org ORG` or `--user` with `--scope all|private|selected` and `--select OWNER/NAME`
  share one. No `GITHUB_` prefix, 48 KiB per value, 100 per repository; names only in `ci secret list`.
- Workflows are GitHub Actions files. Never start a run: `release`, `pull_request_target`, `workflow_run`, `issues`,
  `issue_comment` (build releases on a tag push). Unsupported: `vars.*`, `container:` and `services:`, Windows and
  macOS runners, job-level `concurrency`, `gh` and `api.github.com`. `GITHUB_TOKEN` covers contents, issues, pull
  requests and actions only, and what it pushes starts no run.
- `ci rerun` replays the original workflow text and commit: a fixed workflow needs a new push or `ci run`. A job with
  `environment:` is not retried after a host loss unless `retry-on-host-loss: true`. Test results come from
  `**/junit*.xml` and `**/TEST-*.xml`; artifacts need `upload-artifact@v4`.
- No run at all: the owner's email is unverified (only they can fix it), or the push missed the workflow's filters.
  An outside contributor pushes only `contrib/<handle>/...`; those runs wait for a person with write access and get
  no secrets.

**Env files**
- End to end encrypted; the server cannot read them. `env init` starts the owner's device chain and a recovery key
  (`--recovery-file`); `env push|pull` sync untracked `.env` files key by key (`--theirs` / `--ours` on a clash); `env
  status`, `env log PATH`.
- An agent needs names, not values: `env keys`; `env run -- CMD` puts values in the process only; `env set KEY=-`.
- A new device waits for an owner: `env devices` shows its code, `env approve CODE`, `env grant HANDLE`, `env remove
  HANDLE|DEVICE` (re-encrypts without them); `env export DIR` writes what the `age` tool reads.

**Sandboxes (`sandbox`, `sb`)**
- `sandbox create`: `--image`, `--cpus` (2), `--memory MB` (2048), `--disk GB` (5), `--minutes` (1 to 1440, 60),
  `--network outbound|none`, `--env K=V`, `-R OWNER/REPO --ref R` or `--caisson ID` for `/work` (`--sparse FOLDER`),
  `--wait SECS` (0 fails at once; past it the sandbox stays queued), `--class borrow` (cheaper, can be taken back),
  `--dry-run`.
- `--stop files|checkpoint --keep PATH` (up to 16) keeps files, or memory too, past a stop or expiry; `sandbox start
  ID` (a checkpoint needs the same gVisor release); `sandbox stop ID --mode`; `sandbox delete ID --yes`.
- `sandbox exec ID -- CMD`: 124 on `--timeout` (120, at most 3600); one argument runs under a shell; `--cwd`, `--env`,
  `--stdin-file`. `sandbox cat ID PATH`, `sandbox put ID LOCAL REMOTE --mode 755` (16 MiB), `sandbox extend ID MIN`,
  `sandbox checkout ID --ref R` (keeps `node_modules` and `target/`).
- `sandbox service start ID --name N --command C [--cwd --env JSON]`: restarted on a crash (1 to 30 s backoff,
  `failed` after five quick exits); starting the name again restarts it. `sandbox ps ID`, `sandbox kill ID --pid
  P|--port N`.
- `sandbox expose ID PORT --minutes M` (1 to 480) prints a check token once (header `X-Sailfork-Preview-Check`); three
  links per sandbox; can be held for a person. `sandbox previews ID`, `sandbox unexpose ID PREVIEW`.
- Sailfork runs the hosts; any account with a verified email can use them. `sandbox hosts --org ORG` shows state and
  free room; `no_host` (503) means none has room now.
- `sandbox secret set NAME --value - --grant OWNER/REPO --org ORG`: an env var in sandboxes started from a granted
  repository (`-R` or `--caisson`); `--grant` replaces the list.
- Owners: `sandbox settings view|set|export` (`--max-per-member`, `--max-per-org`, `--max-vcpus`, `--max-memory-mb`,
  `--max-minutes`, `--allowed-images`, `--egress none|outbound|host`; with `-R`, `--queue-weight`,
  `--ci-max-running`); `sandbox log --org ORG` lists every command run.

**Repositories, mirrors, releases, account**
- `repo delete` schedules deletion in 14 days (`repo restore`). `repo purge` by a token deletes nothing: the
  repository stays hidden and restorable for at least 30 days (only a browser session purges at once). `repo
  commit-file PATH -m MSG --file F --branch B` writes one file to a branch with no caisson, under the push rules.
- Push rules refuse a `git push` with a named reason (`! [remote rejected] BRANCH (reason)`): secrets, unsigned commits
  where required, unverified author emails, blocked paths or extensions, file size. Caisson commits are server-signed.
- Search: branch heads only (up to 256), text files under 1 MiB, no vendored, lockfile or minified files, so a
  caisson's uncommitted edits are not in `search code`. A query is literal (a space is not AND), 256 characters;
  regex is Rust (no backreferences); issue and pull request search need `-R`.
- In flight: `touching` takes an exact name and scans the newest 100 pull requests and 100 caissons; a collision is a
  likely break, not a built one; `conflicts` with `checked: 0` is still working. Risk adds 25 per sensitive path
  (`auth`, `billing`, `migrations`, `payments`, `secrets`, `permissions`; `.sailfork/risk-paths` replaces them), takes
  40 off a verified rename, and is advice only. A push workflow with `# sailfork: integration` builds overlapping pull
  requests together and reports `breaks`.
- Provenance: `pr ledger N` and `commit ledger SHA` split a change into verified operations and hand-edit residue;
  contract breaks list removed routes, non-additive migrations and removed `pub` items. Every merge writes a signed
  attestation (`commit attestation SHA`; a reader with hidden paths gets an unsigned, redacted copy).
- Mirrors: a pull mirror makes the repository, its issues and pull requests read-only, and pauses itself when a fifth
  of the remote's refs vanish. A push mirror that takes pushes made on GitHub lands them here past branch protection,
  with CI and this repository's secrets: protect the same branches on both sides. Diverged refs are held, never
  overwritten.
- `repo access show`, `repo access rules set`, `repo access grants set|add|remove` limit people to folders or to
  `sailfork:region NAME` regions of a file.
- `mirror add OWNER/NAME` pushes to a GitHub copy; `--pull` follows a remote and makes this repository read-only
  (`--forge`, `--user`, `--token -`, `--branch`, `--include`, `--exclude`, `--prune`); `mirror pulls view|edit` sets
  the footer on pull requests sent to GitHub.
- `release create TAG FILES... --title --notes-file --draft --prerelease`, `release upload TAG FILES --clobber`.
- `key add --signing|--gpg`; `usage --org ORG --month YYYY-MM`; `undo --list` (kept 7 days), `undo ID --yes`.
- Notices: `inbox --follow` leaves out what your own agents caused and issues another session holds, and a notice
  names what happened, never someone's words. Events are kept 30 days; `events subscription advance` never moves
  back.
- `X-Sailfork-Agent` and the attribution labels are labels, not proof of identity; a plain `git push` is unmarked.
  With the org's "Track agent work" off there are no leases or agent marks.

## Scopes

`repo:read`, `repo:write`, `repo:admin`, `user`, `org:read`, `org:admin`, `sandbox`, `sandbox:expose`, `repo:merge`,
`repo:purge`. `auth login` grants all but `repo:admin`, `org:admin` and `repo:purge`. A merge needs `repo:merge`, a
sandbox or language server `sandbox`, a preview link `sandbox:expose`; `repo:write` never merges and `repo:admin`
never purges. An MCP connection has merge, sandbox and preview scopes only if the person ticked them, and never
`repo:purge`, `repo:admin` or `org:admin`. Approving a sign-in that asks for an admin or purge scope needs two-factor
on that account. A narrower token for an agent: `token create NAME --scopes repo:read,repo:write --repos acme/app
--expires 8h` (the secret prints once); a token can also be limited to branches (Settings), and then moves only
those.

## MCP toolsets

`sailfork mcp` serves `core` (the working loop) unless given `--toolsets core,caissons,issues,pulls,ci,repos,drydock,
context` or `all`; `--read-only` drops every tool that changes something. A tool's toolset is the first word of its
name (`sandbox_` is `drydock`, an unknown word `context`). Outside `core`:

- `drydock`: every `sandbox_*` tool.
- `caissons`: `caisson_preview`, `caisson_fork`, `caisson_intend`, `caisson_status`, `caisson_symbols`,
  `caisson_symbol_refs`, `caisson_lsp*`, `caisson_ast_grep`, `caisson_ast_rewrite`, `caisson_delete`.
- `pulls`: `pull_review`, `pull_revert`, `pull_conflicts`, `pull_resolve_conflicts`, `pull_stack`, `pull_ledger`,
  `pull_github`, `pull_github_send`.
- `ci`: `ci_rerun`, `ci_dispatch`, `ci_cancel`, `ci_artifacts`, `check_run_create`, `check_run_update`.
- `context`: `inbox`, `inbox_mark_read`, `events`, `event_subscription_*`, `held_mine`, `held_org`, `inflight_*`,
  `merge_queue`, `changelog`.
- `issues`, `repos`: `issue_react`, `issue_attach*`, `issue_comment_update|delete`, `issue_create_batch`,
  `repo_note_*`, `repo_purge`, `commit_ledger`, `commit_attestation`.

CLI only: `env`, `mirror`, `release`, `org`, `migrate`, `token`, `key`, `usage`, `auth`, `ci secret`, `ci caches`,
`sandbox settings`, `sandbox secret`, `repo edit|fork|access|restore`, `branch`.

## CLI and MCP names

An MCP tool takes the same fields as its command's flags.

| MCP tool | CLI |
| --- | --- |
| `work_context` | `brief N` |
| `issue_next`, `issue_get`, `issue_list`, `issue_create`, `issue_update`, `issue_close` | `issue next`, `issue view`, `issue list`, `issue create`, `issue edit`, `issue close` |
| `issue_claim`, `issue_release`, `issue_start`, `issue_ship` | `issue claim`, `issue release`, `issue start`, `issue ship` |
| `issue_comment`, `issue_context`, `issue_related` | `issue comment`, `issue context`, `issue related` |
| `issue_ask`, `issue_answer`, `issue_unask` | `issue ask`, `issue answer`, `issue unask` |
| `label_list`, `label_create`, `label_update`, `label_delete` | `label list`, `label create`, `label edit`, `label delete` |
| `milestone_list`, `milestone_create`, `milestone_update`, `milestone_delete` | `milestone list`, `milestone create`, `milestone edit`, `milestone delete` |
| `pull_list`, `pull_get`, `pull_diff`, `pull_checks` | `pr list`, `pr view`, `pr diff`, `pr checks` |
| `pull_create`, `pull_update`, `pull_comment`, `pull_feedback`, `pull_resolve`, `pull_merge` | `pr create`, `pr edit`, `pr comment`, `pr feedback`, `pr thread resolve`, `pr merge` |
| `ci_runs`, `ci_failures`, `ci_logs` | `ci list`, `ci failures`, `ci logs` |
| `search_code`, `search_symbols`, `symbol_refs`, `search_issues` | `search code`, `search symbols`, `repo refs`, `search issues` / `search prs` |
| `repo_list`, `repo_get`, `repo_create`, `repo_tree`, `repo_glob`, `repo_read`, `repo_log` | `repo list`, `repo view`, `repo create`, `repo tree`, `repo find`, `repo cat`, `repo log` |
| `commit_get`, `commit_pulls` | `commit view`, `commit prs` |
| `github_pull_settings`, `github_pull_settings_update` | `mirror pulls view`, `mirror pulls edit` |
| `submit_feedback`, `doctor`, `undo`, `wait`, `batch` | `feedback`, `doctor`, `undo`, `wait`, `batch --file` |

## Every other command

- `auth login|logout|status|list|switch|token|setup-git`, `whoami`; `key list|add|delete`; `token list|create|delete`.
- `org list|view|members`, `org invites list|revoke|accept|decline`.
- `migrate repos|start|status|retry|bring|cancel|switch`; `migrate installations|orgs` list the GitHub App
  installations Sailfork can read and the accounts and organizations under each.
- `repo create|list|view|clone|edit|fork|forks|delete|restore|purge`, `repo tree`, `repo find GLOB`, `repo compare`;
  `branch list|delete`; `commit view|prs|ledger SHA`.
- `pr checkout N` (a local branch), `pr draft|ready`, `pr close|reopen`, `pr threads [--unresolved]`, `pr thread
  resolve|unresolve`, `pr ledger N` (the operations and checks behind each commit), `pr github view|send N`,
  `queue status`, `queue leave N`.
- `issue react N REACTION`, `issue subscribe|unsubscribe N`, `issue reopen N`; `note list|add|edit|delete`; `label
  create|edit|delete`; `milestone create|edit|close|reopen|delete`.
- `ci watch ID` (each job as it changes), `ci secret list|set|delete`; `attachment upload|download`; `release
  list|view|create|upload|edit|delete`; `env unset KEY`.
- `sandbox list [--all --kind]`, `sandbox view ID`, `sandbox settings view|set|export`, `sandbox secret
  list|set|rm`, `sandbox service start|list|logs|stop`.
- `mirror list|add|edit|pause|resume|sync|remove`, `mirror pulls view|edit`.
- `skill install [--project]` writes these skills into a project or agent folder; `setup pi`, `hook
  session-start|session-end` wire Sailfork into an agent; `upgrade [--check|--rollback]`, `uninstall`, `completion
  SHELL`, `help TOPIC` (`exit-codes`, `environment`, `formatting`), `changelog`, `browse`.
