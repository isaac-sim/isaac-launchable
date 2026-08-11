---
name: release-git-flow
description: Prepare and publish this repository's releases by selecting internal dev commits, squashing them into one commit on main, and creating an annotated release tag, using origin as the only remote. Use when checking release readiness, identifying unreleased commits, preparing a versioned release, creating a release tag, or publishing main and its tag to origin.
---

# Release Git Flow

Use only the `origin` remote. Prepare releases from internal `dev` work as one squash commit on `main`, with an annotated version tag on that commit.

## Preserve these invariants

- Keep internal development commits on `dev` and synchronize them with `origin/dev`.
- Keep release commits on `main` and synchronize them with `origin/main`.
- Represent each release on `main` as one squash commit.
- Create release tags only on `main` release commits.
- Push `main` and release tags only to `origin`.
- Never force-push, rewrite shared history, replace a tag, or resolve unexpected divergence without explicit user approval.

## Inspect and refresh

Require a clean worktree. Preserve unrelated user changes. Run:

```sh
git status --short --branch
git remote get-url origin
git branch --all --verbose --no-abbrev
git log --oneline --decorate --graph -20 --all
git fetch origin dev main --tags
```

Stop if local `dev` or `main` has unexpected divergence from its corresponding `origin` branch. Do not silently merge, rebase, reset, or force-push.

## Select the release

Obtain the version from the user; do not guess it. Prefer the existing `vMAJOR.MINOR[.PATCH]` tag style. Confirm that the version is absent locally and on `origin`:

```sh
RELEASE_VERSION=v1.6
git tag --list "$RELEASE_VERSION"
git ls-remote --tags origin "refs/tags/$RELEASE_VERSION" "refs/tags/$RELEASE_VERSION^{}"
```

Squashed release history may not share ancestry with internal development history. Compare patch equivalence and the net tree change instead of assuming `origin/main` is an ancestor of `origin/dev`:

```sh
git log --left-right --cherry-mark --oneline origin/main...origin/dev
git diff --stat origin/main..origin/dev
git diff --check origin/main..origin/dev
```

Identify the exact unreleased internal commits, order them oldest to newest, and obtain user confirmation when the intended set is ambiguous. Exclude commits whose patches are already represented on `origin/main`. Review the resulting files for secrets, internal-only material, generated files, and unintended changes.

## Build a clean release source

Create a temporary source branch from `origin/main`, then cherry-pick only the confirmed commits in oldest-to-newest order:

```sh
git switch --create "release/$RELEASE_VERSION-source" origin/main
git cherry-pick COMMIT_1 COMMIT_2
git log --reverse --oneline origin/main..HEAD
git diff --stat origin/main..HEAD
git diff --check origin/main..HEAD
```

Replace the example commit names with the confirmed commit IDs. Stop on conflicts and inspect them rather than resolving automatically. Run tests appropriate to the selected changes.

## Prepare main and its tag

Fast-forward local `main` from `origin/main`, squash the verified release source, and review the complete staged diff:

```sh
git switch main
git merge --ff-only origin/main
git merge --squash "release/$RELEASE_VERSION-source"
git diff --cached --stat
git diff --cached --check
```

Commit once and create an annotated tag:

```sh
git commit -m "Release $RELEASE_VERSION"
git tag -a "$RELEASE_VERSION" -m "Release $RELEASE_VERSION"
git show --stat --decorate --oneline HEAD
test "$(git rev-parse HEAD)" = "$(git rev-parse "$RELEASE_VERSION^{}")"
```

Stop here when the user asks only to prepare a release. State clearly that `main` and the tag are local.

## Publish when requested

Ask for confirmation immediately before publishing and show the exact destination. Push the branch before its tag:

```sh
git push origin main
git push origin "$RELEASE_VERSION"
```

Do not force-push or use `--tags`. Verify the resulting remote branch and tag after publishing.

## Report the outcome

State the selected internal commits, release commit, version tag, tests run, and whether anything was pushed. Call out unresolved divergence, conflicts, or validation gaps.
