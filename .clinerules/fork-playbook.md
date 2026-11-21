```markdown
# SiYuan Android Fork — Merge Playbook

This repo (`antebudimir/siyuan-android`) is a soft fork of
`siyuan-note/siyuan-android`. Same philosophy as the core repo
(`/home/admin/Projects/siyuan`): fully synced with upstream, fork changes
supersede upstream on overlapping code. Invariant details and release flow
also live in the core repo's `fork-playbook.md` (Release flow / Android).

## Upstream merge procedure

1. `git fetch upstream` (add the remote first if a fresh clone:
   `git remote add upstream https://github.com/siyuan-note/siyuan-android.git`)
2. Back up the current branch (include unpushed work):
   `git branch backup/fork-before-upstream-merge-$(date +%F) refs/heads/main`
3. Note the merge base: `git merge-base HEAD upstream/main`
4. Merge on a throwaway branch first:
   `git switch -c test-upstream-merge refs/heads/main && git merge upstream/main --no-commit --no-ff`
5. Classify every overlapping file **from its actual fork diff** — never from
   the path name or a prior plan:
   ```bash
   git diff <merge_base>...HEAD -- <file>
   ```

- Fork diff empty → **Neutral**: `git checkout --theirs -- <file>`
- Fork diff is only enumerated fork deletions → **`--theirs`-then-strip**:
     take upstream, re-strip those symbols inside the merge commit
- Anything else (fork versioning, workflows, scripts, build config) →
     **Fork-supersede**:

     ```bash
     git checkout --ours -- <file>
     git diff <merge_base> upstream/main -- <file> | git apply --3way \
       || echo "MANUAL RESOLUTION REQUIRED: <file>"
     ```

     If apply fails (the `--ours` step can drop the 3-way anchor blobs),
     resolve that file manually line-level.

1. Pre-commit checks:

   ```bash
   git diff --name-only --diff-filter=U        # must be empty
   grep -rn '^<<<<<<< ' . --exclude-dir=.git   # must be empty
   ```

2. Validate (commands below).
3. Commit the merge:
   `Merge remote-tracking branch 'upstream/main'`
4. **Version-alignment commit** — always the second commit, never folded into
   the merge (see invariants below):
   `:wrench: fork: align siyuanVersionName with the merged <version> package version (<version>-fork.<N+1>)`
5. Fast-forward `main`, delete the test branch, keep the backup branch.
6. Absorption proof (only valid after the merge commit exists — with
    `--no-commit` it still returns the old base):

    ```bash
    git merge-base HEAD upstream/main            # must equal upstream/main SHA
    git merge --is-ancestor upstream/main HEAD   # must exit 0
    ```

7. Push: `git push origin refs/heads/main`

## Mechanical git rules (violating these caused failed merges before)

- Never chain `git checkout --ours` with `git checkout -p --theirs` on the
  same path — after `--ours` the path is no longer unmerged and the second
  command errors. Use the ours + diff-apply recipe above.
- Diff-apply is only valid on unmerged paths. Auto-merged files already
  contain both sides — applying the upstream diff again errors or duplicates
  hunks. Auto-merged files need no action.
- `checkout --ours/--theirs` are no-ops on non-conflicted paths (`|| true`
  masks it). Post-resolve greps, not the checkouts, guarantee the invariants.
- Before committing: no unmerged paths, no leftover conflict markers.

## Fork invariants

- **Version alignment (critical — never let a merge overwrite this):**
  - NEVER take upstream's `versionName`/`versionCode` from `build.gradle`,
    even if upstream `main` is a version ahead. Upstream Android can be one
    version ahead of the core repo — merge its code, but keep the fork
    version aligned to the **core** repo's `app/package.json`, not upstream's.
  - `siyuanVersionName = "<core package.json version>-fork.<N+1>"` — read the
    version from `/home/admin/Projects/siyuan/app/package.json`; CD validates
    `"$VERSION"-fork.*`.
  - `siyuanVersionCode` increments independently and monotonically
    (history: 38001 → 38005; check the current value in `build.gradle` and
    increment).
- **`.github/workflows/fork-release.yml`**: fork-owned. Keep it; adopt
  upstream workflow refactors only when compatible. Never take upstream's
  release or signing key configuration.
- **`scripts/*.test.cjs`**: fork-owned; keep, port upstream test additions only.
- **Kernel build tags**: the `.aar` must build with `-tags "fts5 sqlcipher"`
  (sqlcipher mandatory on Android; fts5 always on).
- **Never rebase** fork commits — always merge, so hashes stay stable.

## Gates (all must pass before push)

```bash
node --test scripts/*.test.cjs
bash ./gradlew :app:tasks --no-daemon   # gradlew is mode 100644 in git;
                                        # CI runs chmod +x — always use bash ./
```

A full `assembleForkRelease` additionally needs `kernel.aar` + `app.zip` from
the core build. If the Android SDK/Java toolchain can't run locally, say so
explicitly and do NOT push until CI runs green. (Environment: SDK at
`~/Android/Sdk`, Java 21 installed.)

## Known pitfalls

- `build.gradle` is the highest-risk conflict: a "helpful" resolution taking
  upstream's version breaks the fork release contract. Its fork diff is
  non-empty by definition (versioning) — it is always Fork-supersede.
- Upstream Android `main` can be one version ahead of core — that's expected;
  the version-alignment commit exists precisely to normalize it.
- Use `--no-pager` or `GIT_PAGER=cat` in automation.
- Auto-merge can silently reintroduce removed features. Always grep after
  resolving — symbol-level AND identifier-level.

## Post-merge record

After each successful sync, append the per-file classification table
(file → fork-diff summary → policy) here, so the next merge starts from
recorded reality instead of rediscovering diffs.

### 2026-11-21: Merge upstream/main (v3.9.0-alpha.1)

| File | Fork Diff Summary | Policy |
|------|-------------------|--------|
| `build.gradle` | Changed versionCode 476→38007, versionName "3.8.11-alpha.1"→"3.8.11-fork.1", removed Aliyun Maven mirrors | Fork-supersede |
| `app/src/main/AndroidManifest.xml` | No fork changes | Neutral (auto-merged) |

**Resolution:**
- `build.gradle`: Kept fork versioning and Aliyun removal; upstream only changed version numbers which fork doesn't adopt
- `AndroidManifest.xml`: Auto-merged upstream change (Wi-Fi hardware optional)

**Result:**
- Merge commit: ba121ba
- Version-alignment commit: d00844f (3.8.11-fork.2, versionCode 38008)
- All gates passed (24 tests, gradle tasks)
- Backup branch: `backup/fork-before-upstream-merge-2026-11-21`
