# Git worktrees with a bare repo

## Overview

With a bare repo, every branch lives in its own folder and no checkout is "the" repo, so you start a task by creating a folder instead of switching branches. This guide sets up that layout and one command, `git wt`, with the subcommands `init`, `clone`, `add` and `remove` to manage it.

```text
~/projects/project-xxxx/
├── .bare/            # bare clone: all history and config
├── .git              # file containing: gitdir: ./.bare
├── main/             # worktree for the default branch
├── feature-login/    # worktree for branch feature/login
└── agent-fix-123/    # worktree for another task or agent
```

Why use it:

- No stashing when you jump between tasks: each folder keeps its own uncommitted changes.
- No rebuild on switching: each folder keeps its own `node_modules` and build output.
- Agents and humans can work on separate branches in parallel without touching each other's files.

The commands:

| Command | What it does |
| --- | --- |
| `git wt init [folder]` | Creates an empty repo in this layout (default folder: the current one) with `main` as the default branch |
| `git wt clone <url> [folder]` | Clones the repo in this layout and creates the default-branch worktree |
| `git wt add <branch> [base]` | Creates a new branch and worktree from `origin/<base>` (default: the remote's default branch) |
| `git wt remove <branch>` | Removes the worktree folder and removes the branch (only if merged) |

## One-time setup

You need git 2.48 or newer and bash. Check with `git --version`.

1. Create a folder for the commands and put it on your `PATH`:
   - Linux and macOS: create the folder, add the `export` line to `~/.zshrc` or `~/.bashrc`, then open a new terminal:

     ```bash
     mkdir -p ~/.local/bin
     export PATH="$HOME/.local/bin:$PATH"
     ```

   - Windows (Git for Windows): create the folder `%USERPROFILE%\.local\bin` and add it to your user `PATH` (Start menu, "Edit environment variables for your account"), then open a new Git Bash window. `chmod +x`, mentioned in the next section, is not needed.

## Install the commands

Git treats any executable named `git-<name>` on your `PATH` as the subcommand `git <name>`. The script lives in this repository, in [`bin/`](bin/):

| Script | Command |
| --- | --- |
| [`bin/git-wt`](bin/git-wt) | `git wt init`, `git wt clone`, `git wt add`, `git wt remove` |

Clone this repository, then copy or link the script into `~/.local/bin` and make it executable:

```bash
git clone <this-repo-url>
cp <this-repo>/bin/git-wt ~/.local/bin/
chmod +x ~/.local/bin/git-wt
```

Or symlink it, so updates from this repo show up immediately:

```bash
ln -s <this-repo>/bin/git-wt ~/.local/bin/git-wt
```

On Windows (Git for Windows), copying the file into `%USERPROFILE%\.local\bin` is enough; `chmod +x` is not needed.

### git wt init

Like `git init`, but in the bare layout: it uses the current folder (or creates `[folder]`), makes an empty `main` branch and its `main/` worktree, and sets the same config as `clone`, including the `origin` fetch refspec. It refuses to run if `.bare` or `.git` already exists, and removes both again if a step fails. The repo has no remote yet; connect it with `git remote add origin <url>`, then push from `main/` with `git push -u origin main`. `git wt add` needs `origin` and that first push.

### git wt clone

Clones a repo in the bare layout. The folder name defaults to the repo name; the default branch gets its own worktree and tracks `origin`. It also sets `worktree.useRelativePaths`, so worktrees link to `.bare` with relative paths and keep working if the project folder is mounted at a different path (container, VM, WSL).

### git wt add

Creates a branch and its worktree next to the others. A slash in the branch name becomes a dash in the folder name, so `feature/login` lives in `feature-login/`. Run it from the project root or from inside any worktree.

The base branch is detected automatically: a bare clone's own `HEAD` points at the remote's default branch, and the script reads it from there. Pass a second argument to branch from something else, for example `git wt add fix-x develop`. `--no-track` keeps the new branch from tracking its base; the first `git push` creates and tracks `origin/<branch>` (thanks to `push.autoSetupRemote`). If the remote's default branch is later renamed, point the bare repo at the new name once: `git --git-dir=.bare symbolic-ref HEAD refs/heads/<new-name>`.

### git wt remove

Removes the worktree folder and removes the branch. The final `git branch -d` is only a second safety net: the script first checks that the branch is merged (into its upstream, or into the default branch if it has none), so an unmerged branch leaves the folder untouched.

Both `add` and `remove` find the project root through the shared git directory (`.bare`), which is why they work from anywhere inside the project.

## Daily workflow

You never push from `.bare`. You commit and push from inside any worktree; all worktrees share the same history and the same `origin`.

1. Clone once, from `~/projects`. This does not change your current directory:

```bash
git wt clone git@github.com:beppler/devca.git
cd devca/main
```

2. Start a task from the project root or from any worktree:

```bash
# from main/; adjust to where you are
git wt add feature/login
cd ../feature-login
```

3. Work as usual. The first push creates the remote branch:

```bash
git add -A && git commit -m "Add login form"
git push
```

4. After the pull request is merged, clean up from anywhere in the project:

```bash
git wt remove feature/login
```

5. Keep `main/` current whenever you like:

```bash
cd main && git pull
```

Useful built-in commands:

| Command | Use |
| --- | --- |
| `git worktree list` | Show every worktree and its branch |
| `git worktree prune` | Forget worktrees whose folder was removed by hand |
| `git fetch origin` | Update `origin/*` for all worktrees at once |

A branch can be checked out in only one worktree at a time, so `git switch` inside a worktree only works for branches no other folder is using.

## Starting from an empty repository

If you cloned a brand-new GitHub repository with no commits, `git wt clone` stops after the clone: the remote has no branch yet, so the upstream step fails and no worktree is created. You are left with only `.bare` and `.git`. Make the first commit in an orphan worktree (git 2.42 or newer). Run this from the project folder, the one that contains `.bare`:

```bash
branch=$(git symbolic-ref --short HEAD)
git worktree add --orphan -b "$branch" "$branch"
cd "$branch"
echo "# my-project" > README.md
git add README.md
git commit -m "Initial commit"
git push
```

- The first line reads the default branch name from the bare repo's `HEAD` (`main` or `master`, depending on your git settings). The worktree folder gets the same name.
- An orphan worktree starts empty on a branch that does not exist yet; the first commit creates it.
- A plain `git push` creates `origin/<branch>` and sets the upstream, because `git wt clone` already set `push.autoSetupRemote` in this repo. For a repo cloned before that, use `git push -u origin <branch>`.
- After this push, `git wt add` works normally.

## Working with agents (Claude Code, opencode)

The most reliable way is to create the worktree yourself and start the agent inside it, one agent per worktree. Agents are scoped to the folder they start in, so an agent that starts in `main/` and then moves elsewhere hits permission prompts and path confusion.

1. `git wt add fix-login`
2. `cd fix-login && claude` (or `opencode`), ideally in its own terminal tab or tmux window.
3. Let the agent commit and push; review and merge through a pull request.
4. You run `git wt remove`. Do not give agents that command, because it removes branches.

### Rules to commit in AGENTS.md or CLAUDE.md

Commit these files to the repo so every worktree has them. A config file in the project root folder, outside any worktree, is not reliably picked up.

- Work only in the current folder; never touch other worktrees.
- Never remove branches or worktrees.
- Commit and push your own branch; do not merge into `main`.

### Known rough edges (as of November 2026)

- Launch tools from inside a worktree, never from the project root folder. That folder is not a worktree, and reports include `/ultrareview` failing, `--resume` showing no sessions and subagents defaulting to the bare repo root when Claude Code starts there.
- Claude Code's built-in `claude --worktree` assumes a normal (non-bare) repo and has failed in bare layouts. Its worktree isolation has also been reported to break existing worktrees in bare layouts by changing shared git config. Use `git wt add` instead.
- Compound commands like `cd <folder> && git ...` trigger an approval prompt in Claude Code about bare repository attacks.
- opencode fixed a bug where its project cache landed in a shared parent folder for bare-repo worktrees; use a recent version.
- Several of these issues are marked closed upstream. Check your tool's version if one bites you.

## Troubleshooting and renaming

| Symptom | Cause and fix |
| --- | --- |
| `fatal: not a git repository` | You are outside the project. `cd` into the project root or any worktree. |
| No `origin/*` branches after a bare clone | The fetch refspec is missing. Run `git config remote.origin.fetch "+refs/heads/*:refs/remotes/origin/*"` then `git fetch origin`. `git wt clone` does this for you. |
| `'main' is already checked out at ...` | A branch can live in one worktree only. Use another worktree, or create a new branch with `git wt add`. |
| `git push` says there is no upstream | Set `git config push.autoSetupRemote true` in that repo (git wt clone does this for you), or push once with `git push -u origin <branch>`. |
| `unknown option: --path-format` | Your git is older than 2.48. Upgrade git. |
| `git wt remove` refuses to remove the folder | It has uncommitted changes or untracked files. Commit or stash them (`git stash -u` includes untracked files), or force with `git worktree remove --force <folder>`. |
| `git wt remove` refuses to remove the branch | The branch is not merged. Merge it first, or remove it on purpose with `git branch -D <branch>`. |
| A worktree folder was removed by hand | Run `git worktree prune`. |

### Renaming a command

A command is just its filename. To rename `git wt` to `git worktree-tools`, run:

```bash
mv ~/.local/bin/git-wt ~/.local/bin/git-worktree-tools
```

## Cheat sheet

| Task | Command | Run from |
| --- | --- | --- |
| Clone a repo in this layout | `git wt clone <url> [folder]` | `~/projects` |
| Start a task | `git wt add <branch> [base]` | Project root or any worktree |
| Finish a task | `git wt remove <branch>` | Project root or any worktree |
| List worktrees | `git worktree list` | Anywhere in the project |
| Update remote branches | `git fetch origin` | Anywhere in the project |
| Forget removed folders | `git worktree prune` | Anywhere in the project |
| Push a new branch | `git push` | Inside that branch's worktree |

Rules of thumb: one worktree per task, launch agents inside the worktree, commit `AGENTS.md` or `CLAUDE.md`, and clean up with `git wt remove` yourself.
