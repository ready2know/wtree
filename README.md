# 🌳 wtree

A single-file Bash helper for working with git worktrees in a bare-repo layout.

Instead of one checkout you switch back and forth in, you get a directory of sibling
worktrees — one per branch — that all share a single bare clone. `wtree` handles the
bookkeeping: creating them with the right upstream, listing their state, fast-forwarding
them, sweeping the dead ones, and dropping an editor or a coding agent into any of them.

```
$ wtree ls
🌳 /Users/me/work/acme

  NAME            BRANCH          STATE  SYNC
  ──────────────────────────────────────────────────────
* main            main            clean  up to date
  feature/login   feature/login   dirty  ↑2
  spike-caching   spike-caching   clean  ↓5
  old-experiment  old-experiment  clean  upstream gone

  * current   ↑ ahead   ↓ behind
```

## Install

`wtree` is one script with no dependencies beyond `git` and Bash. Put it on your `PATH`:

```bash
curl -o ~/.local/bin/wtree https://raw.githubusercontent.com/ready2know/wtree/main/wtree
chmod +x ~/.local/bin/wtree
```

Add the shell hook so `wtree cd` can actually move your shell:

```bash
eval "$(wtree shell-init)"   # in ~/.zshrc or ~/.bashrc
```

(A child process can't change its parent's directory, so `cd` is the one command that
needs a shell function wrapping it. Everything else works without the hook.)

Optional: `jq`, for [hooks](#hooks) and the `herdr-agent` command, plus `herdr` itself for the latter.

## Layout

`wtree init` clones into `.bare` and points a `.git` file at it, so the root directory is
the repo and every subdirectory is a worktree:

```
acme/
├── .bare/            the bare clone — all git objects live here
├── .git              file containing "gitdir: ./.bare"
├── .wtree/           config.json for hooks, plus any files they copy around
├── main/             worktree on branch main
├── feature/login/    worktree on branch feature/login
└── spike-caching/    worktree on branch spike-caching
```

Worktree names map straight to directory names and branch names. `feature/login` becomes
a nested directory and a slashed branch. Absolute paths are also accepted anywhere a
worktree name is expected.

## Getting started

```bash
mkdir acme && cd acme
wtree init acme-corp/backend      # a full git URL works too
wtree add main                    # check out the default branch
wtree add feature/login           # tracks origin/feature/login if it exists,
                                  # otherwise branches off main
wtree cd feature/login
```

`init` refuses to run in a non-empty directory (the `wtree` script itself is allowed to
sit there), which keeps the layout above intact. It also drops a `.wtree/config.json` with
both [hooks](#hooks) present and empty, so setting worktrees up is a matter of filling in a
list rather than knowing a file you have to create first.

## Commands

| Command | What it does |
| --- | --- |
| `wtree init <owner>/<repo>` | Bare-clone into the current empty directory, wire up remote tracking and scaffold `.wtree/config.json` |
| `wtree add <worktree>` | Create a worktree. Checks out `origin/<worktree>` if it exists, otherwise branches off main/master, running the `pre-add` and `post-add` [hooks](#hooks) around it |
| `wtree remove <worktree>` | Remove a worktree and delete its local branch |
| `wtree ls` \| `list` | Table of worktrees with branch, dirty state and ahead/behind counts |
| `wtree sync [<worktree>]` | Fetch, then fast-forward one worktree — or all of them |
| `wtree sweep` \| `clean` | Remove worktrees whose branch is merged or whose upstream is gone |
| `wtree path [<worktree>]` | Print a worktree's path, or the repo root |
| `wtree cd <worktree>` | Jump to a worktree (needs the shell hook) |
| `wtree editor [<worktree>]` | Open a worktree in your IDE |
| `wtree run <worktree> <cmd...>` | Run a command with the worktree as its working directory |
| `wtree agent <worktree> [args...]` | Start a coding agent inside a worktree |
| `wtree herdr-agent <worktree>` | Start a named herdr agent on a worktree |
| `wtree prune` | Prune stale worktree metadata |
| `wtree shell-init` | Print the shell hook |
| `wtree help` | Full usage |

### add

```
wtree add <worktree> [-c <branch>] [-p] [--no-hooks]
  -c, --custom-branch <branch>   Base the new branch on <branch> instead of main/master
  -p, --print-path               Print only the worktree path on stdout
      --no-hooks                 Skip both hooks for this run
```

Three cases, in order: if the branch already exists locally it is checked out (and wired
up to `origin/<branch>` if it wasn't tracking anything — `clone --bare` copies remote
heads in without tracking info, which would otherwise leave `ls`/`sync`/`sweep` blind);
if only a remote branch exists it is tracked; otherwise a new branch is created off the
default branch with `--no-track`.

`-p` keeps stdout to just the path, so it composes:

```bash
cd "$(wtree add feature/login -p)"
```

Hook output goes to stderr under `-p`, so the pipe above still gets a bare path even when
the hook is running `npm ci`.

### remove

```
wtree remove <worktree> [-f]
  -f, --force   Remove even if dirty, and delete an unmerged branch
```

Without `-f`, a dirty worktree or an unmerged branch is kept and reported rather than
thrown away.

### ls

Shows every worktree with its branch, whether the working tree is dirty, and how it sits
against its upstream:

| SYNC | Meaning |
| --- | --- |
| `up to date` | Level with upstream |
| `↑n` / `↓n` | Ahead / behind by n commits |
| `↑n ↓m` | Diverged |
| `no upstream` | Local-only branch |
| `upstream gone` | The remote branch it tracked was deleted |
| `detached` | Detached HEAD |
| `missing` | Directory is gone — run `wtree prune` |

Extra arguments are passed straight to `git worktree list`, so `wtree ls --porcelain`
still gives you the raw thing.

### sync

```
wtree sync [<worktree>] [-r]
  -r, --rebase   Rebase local commits instead of skipping diverged branches
```

Fetches with `--prune`, then fast-forwards each worktree. Only clean fast-forwards happen
by default — a diverged branch is reported and skipped, never merged. With `-r` diverged
branches are rebased instead, and a rebase that hits conflicts is aborted rather than
left half-applied.

### sweep

```
wtree sweep [-n] [-y] [-f]
  -n, --dry-run   Only show what would go
  -y, --yes       Skip the confirmation
  -f, --force     Include dirty worktrees and unmerged commits
```

Removes worktrees whose branch is merged into the default branch, or whose upstream was
deleted. It asks before doing anything (and refuses in a non-interactive shell unless you
pass `-y`). Never touches the default branch, the worktree you're standing in, or — unless
forced — anything dirty or holding commits that aren't in the default branch. Anything
skipped is listed with the reason.

### run

```bash
wtree run main npm install
wtree run main "npm ci && npm test"    # a single quoted arg goes through the shell
```

The command replaces the script via `exec`, so its exit code and terminal are yours. The
`▶️` header goes to stderr, leaving stdout exactly what the command produced.

### agent

```
wtree agent <worktree> [--model <m>] [--effort <e>] [args...]
```

Runs your coding agent with the worktree as its working directory. `--model` and
`--effort` must come before the worktree name; everything after it is handed to the agent
untouched. Defaults are applied only if you didn't pass the same flag yourself.

### herdr-agent

```
wtree herdr-agent <worktree> [-n <name>] [-k <kind>] [--down|--right] [--focus]
```

Starts a named agent on a worktree inside herdr, the terminal multiplexer.
With `--down`/`--right` it splits the current pane and starts the agent there. Without a
direction it looks for an idle, agent-free pane in the current tab, and falls back to
taking over the current pane. Requires `herdr` and `jq`, and only works from inside a
herdr pane.

## Hooks

A fresh worktree is rarely ready to work in. It has no `node_modules`, no gitignored config
files, no local certificates — the things a `git clone` never carries. Hooks are the list of
commands that close that gap, run for you every time `wtree add` creates a worktree.

They live in `<repo-root>/.wtree/config.json`, next to `.bare`, so every worktree in the
setup shares one copy. `wtree init` writes it for you with both hooks empty:

```json
{
  "hooks": {
    "pre-add": {
      "actions": [
        { "type": "bash", "command": "test -f .wtree/secrets/.env" }
      ]
    },
    "post-add": {
      "actions": [
        { "type": "bash", "command": "cp ../.wtree/secrets/.env ./.env" },
        { "type": "bash", "command": "pnpm install" },
        { "type": "bash", "command": "cd api && npm ci" }
      ]
    }
  }
}
```

`.wtree/` is a good home for the files the hook hands out, too — certificates, `.env`
templates, anything that has to reach a new worktree but must never be committed.

### The two events

| Hook | Runs | Working directory | On failure |
| --- | --- | --- | --- |
| `pre-add` | Before `wtree add` does anything at all | The repo root | The worktree is **not** created, and `post-add` never runs |
| `post-add` | Once the worktree exists and is on its branch | The new worktree | Reported, never rolled back |

`pre-add` is a gate. Use it for the checks that make a worktree pointless if they fail — a
missing credential, a service that isn't up, a stale cache to warm first. Because it runs
before the fetch and before `git worktree add`, a `no` costs nothing. Both hooks still run
every action before deciding, so one broken precondition can't hide a second one behind it.

### How actions run

Every action runs in **its own subshell**, rooted at the directory in the table above. A `cd`
inside one action does not leak into the next, so `cd api && npm ci` above starts back at the
worktree root, not in `api/`. That also means `../.wtree/...` reliably points at the config
directory from a `post-add` action.

Each action gets the context in its environment:

| Variable | Example |
| --- | --- |
| `WTREE_EVENT` | `pre-add` or `post-add` |
| `WTREE_NAME` | `feature/login` — as typed on the command line |
| `WTREE_BRANCH` | `feature/login` |
| `WTREE_PATH` | `/Users/me/work/acme/feature/login` |
| `WTREE_ROOT` | `/Users/me/work/acme` |

`pre-add` runs before the worktree exists, so `WTREE_PATH` is where it is *about* to appear —
which is exactly what a check like "is that directory already in the way?" needs.

Actions run in order, each one timed, with a running total at the end. A failing action is
reported and the rest still run — a broken `npm ci` shouldn't cost you the config files that
would have been copied afterwards. Failures are collected and listed at the end, and
`wtree add` exits non-zero if there were any:

```
🪝 post-add - running 3 action(s) in /Users/me/work/acme/feature/login
  [1/3] cp ../.wtree/secrets/.env ./.env
        ✔ <1s
  [2/3] pnpm install
        ✘ exit 1 (4s)
  [3/3] cd api && npm ci
        ✔ 1m 12s
⚠️  post-add: 1 of 3 action(s) failed, 1m 16s total:
     [2] exit 1 after 4s - pnpm install
```

A clean run ends on one line instead:

```
✅ post-add: 3 action(s) in 2m 41s
```

For `post-add` the worktree itself is never rolled back — it exists, it is on the right
branch, and you can finish the setup by hand. For `pre-add` the same list is printed and then
`add` stops:

```
❌ Error: pre-add hook failed - worktree not created.
Fix the actions above, or re-run with --no-hooks to skip them.
```

### Notes

- `"type"` currently accepts only `"bash"`, and defaults to it when omitted. Any other value
  is reported as a failed action rather than silently skipped.
- Hooks need `jq`. If `.wtree/config.json` exists and `jq` doesn't, `wtree add` says so and
  carries on without running anything — a missing tool is not a reason to refuse you a
  worktree, so this never gates `pre-add` the way a failing action does.
- Invalid JSON is reported with jq's own parse error, and no actions run.
- An unrecognised key under `"hooks"` gets a warning — a hook nobody runs otherwise looks
  exactly like a hook that passed.
- An empty `"actions": []` is silent, so the scaffold `init` writes costs you nothing until
  you fill it in.
- `wtree add <worktree> --no-hooks` skips both hooks for one run.
- Durations come from bash's `$SECONDS`, so they are whole seconds; anything faster than a
  second reads as `<1s`. Enough to tell `npm ci` from a `cp`, which is the point.

## Environment variables

| Variable | Default | Used by |
| --- | --- | --- |
| `WTREE_EDITOR` | `webstorm` | `editor` — may carry flags, e.g. `WTREE_EDITOR="code -n"` |
| `WTREE_AGENT` | `claude` | `agent` — may carry flags, e.g. `WTREE_AGENT="claude --dangerously-skip-permissions"` |
| `WTREE_MODEL` | `opus` | `agent`, `herdr-agent` — set to `""` to pass no `--model` flag |
| `WTREE_EFFORT` | `xhigh` | `agent`, `herdr-agent` — set to `""` to pass no `--effort` flag |
