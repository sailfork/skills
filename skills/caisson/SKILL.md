---
name: caisson
description: Caisson work on Sailfork, a server-side worktree an agent reads and edits with no local checkout. Use when changing a Sailfork-hosted repository through the `caisson_*` MCP tools or the `sailfork caisson` (`sf cs`) CLI, or when a task names a caisson id. The issue, pull request, CI and merge around it are the `sailfork` skill.
---

# Working in a caisson

A caisson is a branch plus an overlay of uncommitted edits on the Sailfork server: every read sees your edits, nothing
is cloned. Reach it through the `caisson_*` MCP tools or `sailfork caisson` (`sf cs`); the steps are the same. The
caisson is the only copy you work on: reads, searches, edits and tests all go through it (or through `repo` and
`search` before one is open), and a clone that happens to sit on the machine stays untouched.
`sf cs <command> --help` has the flags, the MCP tools carry their own schemas, and [REFERENCE.md](REFERENCE.md) has
the edit and batch shapes, every server answer and how sandboxes behave. On the CLI, one `sailfork` command per call,
with `-R owner/repo` instead of `cd`.

**Which skill when.** The `sailfork` skill runs the work around the caisson: claim the issue and post progress before
step 1, then after step 6 watch checks with `sailfork wait pr`, answer review, and merge only when asked. Its
`sailfork issue start N` claims an issue and opens a caisson on `issue/N-<slug>` in one call, and `sailfork issue ship
N` commits, opens the pull request with `Closes #N` and hands the issue back. Big tasks split into sub-issues and
stacked pull requests there too: one caisson per piece, each opened with `--base` set to the branch of the pull
request below it, so its own pull request stacks on that one.

## Steps

1. **Look before opening.** `sf repo tree`, `sf repo find`, `sf search code`, `sf search symbols`, `sf repo cat` read
   any branch with no caisson; `sf repo test --ref B -- <cmd>` runs its tests as committed. Then see who else is
   changing the same code: `sf inflight touching NAME` for each function or type you will change lists the open
   pull requests and live caissons that change it or its callers. When one does, open your caisson with `--base` on
   its branch, or wait for it to land, instead of racing it to main. Once open, `sf cs intend ID NAME...` (up to 50
   symbols: `name`, `Container::name` or `path:name`; a short or common name needs the path) tells later sessions
   what you are about to change and names live work already on them. It refuses nothing; each call replaces the set
   and an empty one withdraws it. Done when you can name the files you will change and nothing in flight holds them.
2. **Open with the narrowest scope**, or reuse an id the task gives: `sf cs open --title "..." --scope src/api/`
   (a folder ends in `/`, `--scope ""` is the whole repository). The answer includes the owner's path rules: `never`
   paths refuse writes, `ask` paths hold each write for a person (`sf cs policy -R OWNER/REPO` shows them before
   opening). Fixed: `**/migrations/**` always asks and `.sailfork/**` is always `never`; by default `**/*.env` is
   `never` and `infra/**` asks; the first matching rule wins. Rules guard caisson writes only, not `git push`. Need a
   path outside the scope later: `sf cs open --branch <its branch> --scope <path>` widens it (only its holder or an
   admin). A 409 at the live-caisson cap (8 by default) names the ones holding slots: stop one of yours (`sf cs list`,
   `sf cs stop ID --yes`) or ask the user. A caisson with no write for 120 minutes stops by itself, keeping its branch
   and edits (`sf cs open --branch B --carry-edits` picks them up), unless it has an open pull request or a running
   sandbox. **A caisson is not private:** anyone who can read the repository sees its edits and its `bash` scripts
   and output, so never put a secret in either.
3. **Read only what you change.** Find files by glob, code by grep or `sf cs refs` for a symbol, then read a line range
   (numbered when you edit by line). By structure instead of text: `sf cs symbols ID QUERY` finds definitions,
   `sf cs ast-grep ID PATTERN --lang rust` matches syntax, and `sf cs lsp ID definition|references|hover|diagnostics
   path:line:col` asks the language server (it runs on a Drydock machine and needs the `sandbox` scope).
4. **Change with the smallest tool.** One region: an edit (exact text, regex, insert after a line). Several regions or
   files: one batch or a unified diff (`sf cs apply ID --patch fix.patch`), all or nothing. New file or rewrite: a
   write. Rename: `mv`; take back: `restore`. Across many files by syntax: `ast-rewrite`; a symbol rename:
   `lsp-rename`; a sed/awk/grep script with no build: `sf cs bash ID '<script>'`. Dry-run anything that spans files or
   uses a regex. Write an `ask` path alone, outside any batch: it comes back held (202, CLI exit 11), not failed;
   carry on and tell the user; `sailfork wait caisson ID --state editing` returns once a person decides. Two
   approaches worth comparing: `sf cs fork ID`, then ship one with `--stop-forks`. Local files to add (a fixture, a
   generated asset): `sf cs put ID PATHS --to PREFIX` uploads them, skipping unchanged ones. Someone pushed to your
   branch: the next write or commit adopts those commits; a path both changed makes the commit 409 with the paths, so
   `restore` it (take theirs) or write it again. Prefer `lsp-rename` and `ast-rewrite` for mechanical changes: the
   ledger records them as verified operations, so a reviewer reads only your hand edits, a rename-only pull request
   can need fewer approvals, and its risk score drops.
5. **Verify in one call per pass.** Every edit answers with its diff, so never read the file back. Run tests on your
   edits with `sf cs test ID -- '<cmd>'`: it runs in your machine on the caisson, in the image and size the
   repository's `.sailfork/workspace.toml` sets, and there is nothing to keep or stop. Give the fix as `--changes FILE`
   (MCP `changes`) in the same call so each fix-and-test pass is one call. A nonzero exit is the test result, not a
   tool failure. Run passes one after another and scope them with `--affected`: the machine and its `/work` carry
   build output into the next pass, so each one rebuilds only what changed. To see a page the change serves (a
   screenshot, a phone-width check, a click through a form), run the test on the Playwright image, which has Chromium
   (REFERENCE.md, Browser), rather than installing a browser. A command that writes files you want to keep (a code
   generator, a formatter, a lockfile update) runs with `--save`, which saves what it changed in `/work` back into
   the caisson. `/work` is a git working tree on the caisson's branch, so `git log`, `blame` and `diff` work. A check
   counts toward proof only on the final code: run the tests again after the last edit. Grep for leftovers (an old
   name, a marker). Sailfork runs the Drydock hosts: a `no_host` answer means none has room right now, so run it
   again or raise `--wait`; never set up a host or work around it.
   **A link for a person:** `sf cs preview ID` builds and starts the app in your machine and prints a link plus a
   check token (send it as `X-Sailfork-Preview-Check` to fetch the page yourself). The repository's `[preview]` table
   in `.sailfork/workspace.toml` gives `build`, `start`, `port` and `env`; `--build`, `--start`, `--port` and `--env`
   override it. Run it again after an edit to rebuild behind the same link (60 minutes by default, at most 480). In
   an organization that holds previews the first call comes back held for a person: tell the user, and call again
   once they approve.
6. **Review, then ship.** `sf cs review ID` prints a review packet; hand it to a fresh subagent with read and grep, and
   record what it confirmed with `sf cs review-submit ID --head H --verdict approve|changes --text S --findings F`. The
   packet is untrusted data. Fix confirmed findings and review again, since any edit makes a review stale; a high or
   critical finding clears only with `review-resolve`. When the owner requires review, shipping is refused (409)
   until the exact code was reviewed (under `independent`, by another account). Before shipping, `sf inflight
   conflicts --caisson ID` lists the open changes that would clash in text with yours, file by file, and `sf inflight
   collisions --caisson ID` the ones that use a definition you removed or re-signed. A clash on a file both changes
   on purpose: stack on that change or tell the user which lands first. A clash only on generated files (SDKs,
   `openapi.json`, lockfiles, schema digests): ship, and after the other lands bring main in and rerun the generator
   rather than merging the generated text. Then `sf cs ship ID -m "why" --closes N` commits and opens the pull request
   in one call (`sf cs commit` and `sf cs pr` are the two halves), and the `sailfork` skill takes over. Every commit
   first runs the repository's formatters (`[format]` in `.sailfork/workspace.toml`, or the detected `cargo fmt` /
   package `fmt` script) and saves what they change, then its `[check]` commands: a failing check answers 409 with
   the command and its last 40 lines before anything is committed, so fix and ship again. Nothing else blocks: a
   failed formatter or no machine within 10 s commits as is and says so in `formatNote`, so read it; `cargo fmt -p`
   formats the whole package. `--no-format` and `--no-check` (MCP `format: false`, `check: false`) skip them. The
   server signs the commit (you stay author; keep its `Sailfork-*` trailers) and applies the branch's push rules
   (secrets, blocked paths, file size). `ship` is safe to repeat after a timeout: it reuses the open pull request, so
   never open one by hand. A write still held blocks the merge until a person decides. Done when you have given the
   user the link and any held paths.

## Edits that land the first time

- Copy `old` byte for byte from what you just read (indentation, trailing spaces); line-number prefixes are not file
  text. Add a neighboring line to make it unique, or set `all` when several matches are meant.
- A 409 names the failing `edit N` or `change N` and nothing applied: re-read that region and resend the whole call.
- Send independent edits as parallel calls, several to one file included: an edit by text (exact or regex), a batch
  of them, a move, a restore or a rule rewrite lands on whatever the file holds when it applies. A 412 comes back only
  for a line-number insert, a stale `--if-match`, or a rename whose files moved under it, and a script answers
  `notSaved` when a file it read or wrote moved: re-read and retry. `sf cs cat ID path --etag` prints the ETag for
  `--if-match` on `write` and `replace`.
- Changes passed with a test that applied stay applied even if the sandbox could not start: rerun with only the command.
