# CLAUDE.md

Personal fork of [obsidian-tasks](https://github.com/obsidian-tasks-group/obsidian-tasks),
built as **Tasks (JB)** (`obsidian-tasks-jb`) and installed via BRAT alongside the official plugin.

## Branch model

- `main` mirrors upstream `obsidian-tasks-group/obsidian-tasks:main`. Sessions never commit to or push it;
  the owner keeps it current with GitHub's **Sync fork** button if wanted. Nothing depends on it.
- `custom` (the default branch) holds all personal changes, kept as a small stack of commits rebased onto an
  **upstream release tag** (e.g. `8.6.0`), not upstream `main`. See "upstream sync" below.
- Keep the plugin-identity change (`manifest.json` id/name/version) as its own commit so rebases stay trivial.
- Develop on `custom` (or a short-lived branch merged into it). CI (`verify.yml`) runs on pushes and PRs to `custom`.

## Version scheme

`<upstream-version>-jb.<n>`, e.g. `8.6.0-jb.1`.

- `<upstream-version>` is the upstream release tag `custom` is based on.
- Bump `n` for every release. Reset to `1` when rebasing onto a new upstream version.
- Only `manifest.json` `version` matters for releases. Do not run upstream's `release.sh`.

## Release process

1. Bump `version` in `manifest.json` on `custom`, commit, push.
2. In GitHub: Actions → **Release Obsidian Plugin (custom)** → **Run workflow** on branch `custom`.
3. The workflow builds, tests, creates tag `<version>` (no `v` prefix) and publishes a Release with
   `main.js`, `manifest.json` and `styles.css` as separate assets. It fails if the tag already exists
   or if run from any branch other than `custom`.
4. BRAT picks up the new release on its next update check.

Cloud sessions cannot push tags; the workflow creates the tag, so never try to tag locally.

## Scope

Changes on `custom` are **UI-only**: rendering, styling, layout, settings UI.
They must **not** alter task syntax, serialization, parsing, query language, or any behavior that changes
what gets written to Markdown files. Both plugins may read the same vault; the files must stay
100% compatible with official Tasks.

If a requested change would cross that line, stop and say so before implementing.

## Testing discipline

- **Red-green TDD for any change to behavior or rendered output.**
  1. Write the test first.
  2. Run it and confirm it **fails for the expected reason**. Show the red run output, not just the green one.
  3. Implement until it passes.
  4. Refactor with tests green.
- **Prefer extending existing tests** (e.g. the approval tests in `tests/Renderer/`) over creating new test files.
  Approval tests: update `.approved.*` files only after inspecting the `.received.*` diff and explaining it.
- **CSS-only changes are exempt from TDD.** Instead, describe the visual change and exactly how to check it in
  Obsidian (which view, which query or task line, light and dark theme, desktop and mobile if relevant).
- **Never edit or delete an existing test to make it pass** without explicitly saying which test and why.
- **Run the full suite before every push:** `yarn test` (and `yarn run lint:no-fix` for TS/Svelte changes).

## Working agreements

- **Scope:** keep each change focused on what was asked. Don't bundle unrelated fixes or refactors;
  mention them as suggestions instead.
- **Plan first:** for anything beyond a small, obvious tweak (new UI element, changes across several files,
  anything touching settings), propose the approach and wait for an explicit OK before coding.
- **Corrections:** when corrected on a convention or preference, propose a CLAUDE.md update so the next
  session gets it right.

### Procedure: "check upstream" / "upstream sync"

Run when asked to "check upstream" or "do an upstream sync". Upstream tags have no `v` prefix (`8.6.0`).

Setup: add remote `upstream` = `https://github.com/obsidian-tasks-group/obsidian-tasks.git` if missing,
then `git fetch upstream --tags`.

- **Base tag** (what `custom` is on now): `git tag --merged custom --sort=-v:refname | grep -E '^[0-9]+\.[0-9]+\.[0-9]+$' | head -1`
- **Latest tag**: `git tag --sort=-v:refname | grep -E '^[0-9]+\.[0-9]+\.[0-9]+$' | head -1`

1. **Check (read-only), then STOP.**
   - Sanity check: `git log --oneline <base-tag>..custom` must list only custom commits (a handful, all
     `(custom)`-scoped). If upstream commits appear, `custom` is not sitting on a release tag; stop and report.
   - List every upstream release between the base tag and the latest tag, summarized from upstream's GitHub
     release notes, grouped as features / fixes / breaking changes.
   - Flag upstream changes to files that the custom commits modify
     (`git diff --stat <base>..<latest> -- $(git diff --name-only <base>..custom)`).
   - Recommend sync now or skip, with reasons.
   - Change nothing. Wait for an explicit OK in this session ("sync", "go ahead", "sync to 8.7.0").
     If "check upstream" was the request, stop here. A new session never acts on an OK from an earlier one.
2. Rebase `custom` onto the chosen tag: `git rebase --onto <tag> <base-tag> custom`. Resolve conflicts in favor of
   keeping the custom changes; list every conflict and how it was resolved.
3. Run `yarn install --frozen-lockfile`, `yarn test`, `yarn run lint:no-fix`, `yarn run build`.
4. Set `manifest.json` version to `<tag>-jb.1` and commit it (`chore(custom): version <tag>-jb.1`).
5. Only if step 3 is green: `git push --force-with-lease origin custom`. Never push `main` or tags.
6. Summarize upstream changes that touch the customizations and what to verify by hand in Obsidian.

Force-pushing is expected here: the rebase rewrites the custom commits. Old release tags keep pointing at the
pre-rebase commits; that is fine.

### Procedure: "pre-release check"

Run when asked for a "pre-release check".

1. List the custom commits: `git log --oneline <base-tag>..custom` (base tag as defined in "upstream sync").
   If the last `-jb` release tag is still an ancestor of `custom` (no sync since), also list
   `git log --oneline <last-jb-tag>..custom` as "new since last release".
2. Run the full suite and a production build (`yarn test`, `yarn run lint:no-fix`, `yarn run build`).
3. Bump the `-jb.<n>` version in `manifest.json`, commit, and push `custom`.
4. List what to verify by hand in Obsidian before clicking **Run workflow**, based on the commits in step 1.

## Commands

| Task | Command |
| --- | --- |
| Install | `yarn install --frozen-lockfile` |
| Build | `yarn run build` |
| Test (all) | `yarn test` |
| Test (one file) | `yarn vitest run tests/Renderer/TaskLineRenderer.test.ts` |
| Lint (check only) | `yarn run lint:no-fix` |
| Markdown lint | `yarn run lint:markdown` |

## Known caveats of running next to official Tasks

The plugin id is changed, but some identifiers are still shared with the official plugin
(see `src/Obsidian/TasksEvents.ts` event names on `app.workspace`, and the `tasks` code-block processor in
`src/Renderer/QueryRenderer.ts`). Do not change these without discussing it: renaming them diverges from upstream
and the `tasks` code block is part of the user-facing syntax.
