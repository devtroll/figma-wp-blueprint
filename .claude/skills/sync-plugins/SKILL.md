---
name: sync-plugins
description: Merge upstream changes of companion plugins into the project - pull the locked base plugin, integrate the original repository of the editable template plugin into its project copy, commit the upstream merge, then re-read both and report what the project must change. Use for /sync-plugins, "pull the plugins", or before continuing work after the plugin owner delivered changes.
---

# sync-plugins - merge upstream plugin changes and review them

**Context reuse:** Reuse any required file already loaded in project instructions or the current session. Read it again only if absent, changed, or exact current bytes or line references are needed.

For projects whose output lives in plugins next to the theme (see "Companion
repositories" in `AGENTS.md`): a locked base plugin that only gets pulled, and an
editable template plugin that has its own repository in the project and an
original repository elsewhere. This skill brings both up to date and tells you
whether the project must react. It never pushes.

## Input

- The base plugin directory (read only, git repository with a remote).
- The template plugin directory of the project (git repository, may carry the
  customer's changes and an unfinished work branch).
- The original template repository (a git checkout elsewhere on the machine).

Take the paths from the arguments, otherwise from the delivery table in
`blueprint/context/project-overview.md`. When one is unknown, ask once.

Invoking `/sync-plugins` approves exactly one commit: the upstream merge commit on
the default branch of the project's template plugin (Step 3). Every other commit,
merge and push needs its own approval.

## Step 1 - preconditions (read only)

For each repository: current branch, `git status --short`, no merge or rebase in
progress. The base plugin must have no local changes; if it does, stop and report
them (it is locked). Note whether the template plugin sits on a work branch with
uncommitted changes.

## Step 2 - base plugin

`git pull --ff-only` in the base plugin. Never edit it. List the new commits
(`git log --oneline <old>..HEAD`).

## Step 3 - template plugin

1. `git fetch` in the original repository and find what is new there: the
   last integrated upstream commit is named in the body line `upstream: <sha>` of
   the latest `merge upstream` commit of the project copy; without such a line,
   compare the trees (`git diff --stat <candidate> HEAD`) until one upstream
   commit reproduces the project copy minus the customer's files. Nothing new:
   say so and skip to Step 4.
2. Build the patch: `git -C <original> diff --binary <last>..HEAD`.
3. If the project copy is on a work branch with uncommitted changes:
   `git stash push -u -m <name>`; then `git checkout <default>`.
4. `git apply --check`, `git apply`; a file the customer changed on the default
   branch (for example `CLAUDE.md`) is merged by hand. Run the plugin's checks.
5. Commit on the default branch: `feat: merge upstream <topic>` (conventional,
   lowercase, under 50 characters) with the body line `upstream: <sha>`. No
   attribution lines.
6. Back on the work branch: `git merge --ff-only <default>`, then `git stash pop`.
7. Resolve conflicts by hand, hunk by hunk: keep the customer's values and
   layout, take the upstream logic. Never use `git checkout --theirs/--ours`, never
   `git checkout <file>` or `git restore` to drop a side; they destroy other
   uncommitted work. Remove conflict markers with targeted edits, re-run the
   checks, recompile translations (`composer i18n:compile` or the plugin's command).

## Step 4 - re-read and review

Read what changed in both plugins, not only the file list:

- Changelogs, docs and hook lists of the base plugin since the old commit; new,
  renamed or removed options, filters, template variables, catalog keys, REST
  routes; template interface version.
- The upstream diff of the template plugin and how it meets the customer's
  changes (CSS variables, markup, JS selectors that now collide).
- The active spec, `blueprint/context/current-feature.md`, and the plugin's own
  `CLAUDE.md`: statements that are now wrong (option names, labels, "missing in
  the base plugin").
- A database option or setting the change makes necessary, and values that the
  project set by hand earlier (list them for the "Live site (manual)" section).

Check the result in the browser when the page is reachable (screenshot or a few
measured values), and run the plugin checks and the project's Verify command.

## Step 5 - report

One table per repository: old and new commit, new commits, and the commit you
made. Then list only what needs action: a required change (done or proposed with
the exact file), a conflict you resolved and how, an open question for the user,
and a wish for the base plugin owner as a precise specification. State plainly
when nothing is necessary. Update the spec's upstream-state note and the plugin's
`CLAUDE.md` history when they name a version.

## Rules

- Never push, and never modify the base plugin.
- Never discard uncommitted work. Stash, merge by hand, restore.
- Do not touch the customer's work branch beyond fast-forwarding it and
  restoring the stash.
- Do not fix things the review finds unless they are required by the merge;
  report them.
