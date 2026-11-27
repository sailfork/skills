# Caisson reference

Flags: `sf cs <command> --help`; MCP tools carry their own schemas. This file holds what neither shows. Every
`caisson_*` tool that takes `id` also takes an optional `org`, found from the id when left out.

## Every command

`sf cs` is `sailfork caisson`; each MCP tool is `caisson_` plus the command name with `_` for `-`.

- **Open and find:** `open` (`--scope`, repeatable; `--base`, `--branch`, `--carry-edits` takes over a stopped
  caisson's uncommitted edits; MCP `widen_scope` widens the live caisson on `branch`), `list` (`--state`), `view ID
  [--log]` (MCP `caisson_status`: diff, commits, session log, held write), `locate ID` (which repository and branch
  an id is), `policy -R OWNER/REPO` (path rules and limits without opening one).
- **Read:** `cat`, `ls`, `find GLOB`, `grep PATTERN [--full]`, `tree`, `refs NAME`, `symbols QUERY [--kind]`,
  `ast-grep PATTERN --lang L`, `lsp ID status|definition|references|hover|diagnostics [path:line:col]`, `lsp-stop`
  (ends the language server sandbox, to restart one that failed), `diff` (everything against the base).
- **Change:** `edit` (an edits JSON), `replace --old A --new B` (repeat for several; `--regex`, `--all`), `insert
  --after N`, `write PATH` (`--create`, `--if-match ETAG`), `mv`, `delete` (alias `rm`), `restore PATH|.`, `apply
  --file F` or `apply --patch F` (same as `apply-patch`), `bash SCRIPT` (no build; `--dry-run`), `ast-rewrite` and
  `lsp-rename` (both a dry run until `--apply`), `put PATHS --to PREFIX` (local files up), `sync ID [DIR] --open
  zed|code` (mirror it into a local folder to read in an editor).
- **Coordinate:** `intend ID NAME...`, `fork ID`.
- **Run:** `test` (`--affected`, `--compare`, `--changes`, `--save`, `--output`, `--env`, `--sparse`), `preview`.
- **Finish:** `review` (`--full`), `review-submit`, `review-resolve ID --finding N --text WHY` (clears a finding;
  under `independent` review your own clearing never counts), `commit`, `pr`, `ship`, `stop ID --yes` (`--reason`,
  `--delete-branch`).

## Edit shapes

Each edit in `edits` is exactly one of, applied in order, each to the result of the one before:

- `{old, new, all?}`: exact text; must match once unless `all`.
- `{regex, new, all?}`: Rust regex, `^`/`$` at line ends; `new` may use `$1` or `${name}`.
- `{insert_after, text}` (MCP) / `{insertAfter, text}` (CLI JSON, REST): after that line, 0 = the top.

## Batch changes

For `caisson_batch`, `sf cs apply ID --file F`, and `changes` / `--changes F` on a test. Each change is `path` plus
exactly one of `edits`, `content`, `delete: true`, `move_to` / `moveTo` (optional `overwrite`), or `restore: true`.
`delete`, `move_to` and `restore` take folders; `restore` on `""` undoes everything, and on a held path withdraws it.
Changes apply in order, then all land or none do. CLI JSON is `{"changes": [...]}` or a bare array, with REST names:

```json
[{"path": "src/lib.rs", "edits": [{"old": "fn old_name(", "new": "fn new_name(", "all": true}]},
 {"path": "src/util.rs", "moveTo": "src/helpers.rs"},
 {"path": "docs/old.md", "delete": true}]
```

## Answers

| Answer | Meaning | Do |
| --- | --- | --- |
| 202 `HELD FOR A PERSON`, CLI exit 11 | `ask` path; nothing applied, caisson `waiting`; one held write at a time | Work on other paths, tell the user |
| 403 `REFUSED BY RULE` | `never` path | Another path or tell the user; retrying cannot work |
| 403 `REFUSED` | Outside the scope, or the token lacks it | Open a caisson with a wider scope |
| 403 `caisson_held_by` | Another key or person opened this caisson | Open your own |
| 403 `agent_policy`, 402 `agent_change_cap`, 429 | The org set agents read-only or stopped, or a too-old token; its monthly change cap (a batch over it is refused whole); its per-minute limit | Tell the user; for 429 wait `Retry-After` |
| 403 on a test, review or language server, or `--- sailfork:hidden ---` in a file | Your access is limited to some folders or regions; edits apply inside your regions only | Work within them; a missing file can be hidden, not absent |
| 404 | No such file | Find it by glob |
| 409 `edit N` / `change N` | Matched nothing, several times, or the destination exists; nothing applied | Re-read, resend the whole call |
| 409 `needs a person's approval` | A batch touched an `ask` path | Send that path alone |
| 409 merged or stopped | Caisson takes no more changes | Open a new one |
| 412 | `--if-match` stale, or a line-number insert or an LSP rename whose file moved under it | Re-read (`--etag`), retry |
| 409 on commit or ship naming a command | A `[check]` command failed; nothing was committed, the formatters' changes are saved | Fix the files, run it again |
| 409 on commit naming paths | Commits pushed to the branch changed the same paths; the edits are kept | `restore` a path (take theirs) or write it again |
| `bash`: changed while the script ran | A file moved under the script; nothing was saved | Run it again |
| 413 | A file over 64 MiB, or over 12 GiB of uncommitted edits in the caisson | Leave it out, or commit and continue |
| 422 | Bad range, pattern, regex, glob or shape | Fix the call |

## Sandboxes (`caisson test`, `repo test`)

- They run on Sailfork's Drydock hosts; nobody registers or sets up a host, and every account with a verified email
  can run them. `sailfork sandbox hosts --org ORG` shows each host's state and free room.
- `caisson test` runs the base plus your uncommitted edits in `/work` of your machine on the caisson, which Sailfork
  finds or starts; the next run reuses it, re-synced to the caisson first. It goes 10 minutes after its last command
  (or after an idle minute when another of your requests waits for room) and `/work` stays for the next one. A
  machine still waiting for room keeps its place: run the same command again. `repo test` runs a ref as committed, once.
- The repository's `.sailfork/workspace.toml` sets `image`, `cpus`, `memory_mb`, `disk_gb`, `timeout_seconds`, a
  `setup` command (at most 7 minutes, run only with the file's image) and `sparse` folders; a flag on the call wins,
  and a value your machine does not match replaces it. Its `[format]` and `[check]` tables (glob of changed paths =
  command, with `{files}` and `{crates}` filled in) run on every `commit`, `ship` and `issue ship`; `format = false`
  turns formatting off. Its `[preview]` table (`build`, `start`, `port`, `env`) is what `caisson preview` runs. A
  caisson cannot write `.sailfork/**`: change it through a reviewed pull request. Code over the
  sandbox limits ("the code is too large for a sandbox") needs `--sparse FOLDER` (MCP `sparse`, repeatable) or the
  `sparse` list: only those folders, the files at the top of the tree and of their parent folders go in `/work`.
- `--compare` (MCP `compare: true`) runs the command on the caisson's base commit and on its edits at once, in two
  sandboxes it destroys, so one call answers whether the edits broke or fixed it. The answer is a verdict line
  (`the edits fixed it`, `the edits broke it`, `same exit`), the output lines only the base and only the edits printed
  (stdout then stderr, 40 lines each), then each run's output tail. Base is the commit the caisson started from, so
  commits the caisson made since count as edits. CLI exit code is the edits run's. `--changes` apply first; `save`,
  `affected` and `outputs` are refused with it.
- Exit: the command's own code (MCP: `exit N`), 124 on timeout (`--timeout`, default the repository's, else 120 s, at
  most 3600), 127 not found, 137 out of memory (on some hosts only the process ends, on others the whole machine
  does and the next call starts another on the same `/work`), 3 when `--affected` found nothing to run (`NO TESTS
  RAN` over MCP, which is not a pass).
- `--save` saves what the command changed in `/work` back into the caisson when it exits 0 (`--save-on-failure` also
  after a nonzero exit, never after a timeout); ignored files and build output stay out. `--env KEY=VALUE` sets a
  variable for the command, `--on HOST` and `--label L` pick where it runs, `--dry-run` prints the request.
- A shell chain goes after `--` as one quoted string, or `--cmd '...'`; MCP `command` runs under `/bin/sh -c`, `argv`
  without a shell.
- With no `.sailfork/workspace.toml`, the image is `buildpack-deps:noble-scm`: git, curl, python3 without pip; no node,
  compilers or apt index (`apt-get update` first). Pick `--image` by stack: `python:3.12-slim`, `node:22`, `rust:1`,
  `golang:1.23`, or better, commit the file.
- `/work` is a git working tree on the caisson's branch (blobless history, a read-only token for this repository),
  except with `--sparse` or no network.
- One sandbox may use at most half of a shared host's vCPUs; a bigger request is refused or placed elsewhere.
- Waits up to `--wait` (default 60, at most 600) for room; a machine still waiting keeps its place for the next call.
  Answers from the host side: `no_host` (503, none has room: retry), `host_offline` (its host dropped: retried within
  the grace period), `quota_exceeded` (429 too many at once, 422 over the size or minutes the org allows),
  `image_not_allowed` and `egress_blocked` (an owner's `sandbox settings`: tell the user).
- `outputs` (MCP) / `--output PATH` (CLI, repeatable) brings back files the command wrote: paths under `/work` or
  absolute, `*`, `?` and `[..]` within one folder (no `**`). They are read after the command ends, whatever its exit
  code. MCP inlines each text file up to 16 KiB (64 KiB in all, at most 100 files) and gives binaries by size; the
  rest stays in your machine's `/work` for a later command. The CLI writes them under
  `--output-dir` (default: here) with their `/work` paths and prints each on stderr. Paths that matched nothing are
  named.
- Per-repo dependency cache (pip, uv, npm, pnpm, bun, Go, cargo) and a per-caisson `/work` volume on the host that
  keeps sources and build output (`target/`, `node_modules`) from one machine to the next: a second
  `cargo check -p` of one crate went from 55 s with nothing reused to 10 s with every crate Fresh. One sandbox holds
  the volume at a time; another on the same caisson (a parallel test, `--compare`'s base run) starts without it and
  builds from the warm image or from nothing. `--fresh-workspace` discards it; stopping the caisson drops it.
- With no `.sailfork/workspace.toml`, a machine is small (2 vCPU, 2 GB). A build that needs more passes `--cpus N
  --memory MB` once (the machine is replaced at that size, same `/work`), or the repository commits the size. Size it
  to the room `sailfork sandbox hosts --org ORG` shows free: a request bigger than that queues.
- In a `rust:1` sandbox whose image lacks the toolchain the repository pins, the first `cargo` call installs it with
  rustup. Let that call finish before starting a second one; two installs at once fail with `detected conflict`.

## Browser

- No image has a browser unless you pick one. For screenshots, a viewport check or a scripted click-through of a page
  your change serves, pass `--image mcr.microsoft.com/playwright:v1.65.1-noble` (MCP `image`) to `caisson test`: it
  ships Chromium and node. Do not install Chromium into the repository's image yourself.
- Start the app on `localhost` and drive it in the same command, so the server is still up: Playwright directly, or
  `browse shot URL out.png --full --width 390` and `browse run script.mjs` after its setup line. Bring results back
  with `outputs` (binaries come back by size; attach a screenshot to a pull request with `issue_attach_sandbox_file`).
  Setup, options and errors: https://sailfork.com/docs/sandbox-browser.
- The image is about 3.7 GB and pulled once per host. An org or host that restricts images refuses it until an owner
  allows `mcr.microsoft.com/playwright:*` (`sailfork sandbox settings set --allowed-images`); tell the user, do not
  work around it.
- A person who wants to look at the page needs a link instead: `sf cs preview ID` (MCP `caisson_preview`).

## Limits

Files up to 64 MiB. A read takes up to 100 files; a batch or patch touches up to 100,000. Grep answers from the search
index on big repositories, which leaves out vendored folders, generated files and files over 1 MiB (`--full`, MCP
`full`, reads every file); it skips binary files and files over 16 MiB and returns up to `limit` (default 200, at most
2000, context at most 20 lines). `sf cs grep` and `sf cs find` with no match exit 1. `cs intend` takes up to 50
symbols. A family of forks holds at most 24 caissons.
