---
name: release-synced-expo-video
description: WIP. Release an upgraded expo-video with our patches on top of the latest stable upstream expo-video from npm. Use when asked to sync or upgrade our expo-video fork to a new upstream version.
---

# Release a synced expo-video (WIP)

Steps 1-7 are verified (done for 58-0-14). Steps 8+ are not done yet.

- `origin` is our fork, `upstream` is `expo/expo`.
- `remote/<x-y-z>` is a clean mirror of upstream at the commit that published `expo-video@<x.y.z>`. Never commit to it.
- `release/mobile-<x-y-z>` is our patched build.

## Steps

1. **Pick the version.** Use the latest stable unless the user names one.
   ```sh
   npm view expo-video@<version> version gitHead --json
   ```

2. **Find the upstream commit.** If `gitHead` is present, use it. Otherwise find the commit that set the version:
   ```sh
   git fetch upstream
   git log upstream --all --format='%h %ad %s' --date=short \
     -S'"version": "<version>"' -- packages/expo-video/package.json
   ```
   Check that `git show <sha>:packages/expo-video/package.json` shows `<version>`.

3. **Create the mirror branch.**
   ```sh
   git branch remote/<x-y-z> <sha>
   ```

4. **Stash local work and switch.**
   ```sh
   git stash push --include-untracked -m "<name>"
   git checkout remote/<x-y-z>
   ```
   `react-native-lab/react-native` may show as modified. That is a submodule pointer difference. Fix it with `git submodule update --init react-native-lab/react-native`.

5. **Create the release branch.** Copy the previous one, then move our patches onto the new base:
   ```sh
   git branch release/mobile-<x-y-z> release/mobile-<previous>
   git rebase --onto remote/<x-y-z> remote/<previous-x-y-z> release/mobile-<x-y-z>
   ```
   Resolve conflicts, `git add` each file, then `GIT_EDITOR=true git rebase --continue`.
   Conflict rules from 57-0-2 to 58-0-14:
   - Version strings (`build.gradle`, `package.json`): keep upstream's.
   - Same function changed by both sides (`DataSourceUtils.kt`): keep upstream's logic, apply our signature change on top.
   - `apps/bare-expo/app.json`: keep upstream's plugin list, keep our `expo-video` options.
   - `apps/bare-expo/ios/Podfile.lock`: take upstream's (`git checkout --ours` during rebase). Regenerate with `pod install` later. This drops our IMA pod entries until then.

6. **Push and open a draft PR.** Ask the user before pushing. The PR goes to our fork, never to `expo/expo` (`gh` defaults to `expo/expo`, so pass `--repo`).
   ```sh
   git push -u origin release/mobile-<x-y-z>
   gh pr create --repo fredrifoUni/expo --draft \
     --base remote/<x-y-z> --head release/mobile-<x-y-z> \
     --title "Release/mobile <x.y.z>" --body-file <file>
   ```
   Body headings: `Why`, `How`, `Test Plan`, `Screenshots`, `Checklist`. List each conflict and how it was resolved. Write the body to a temp file. End it with the Claude Code attribution line.

   **The PR commit list shows every patch, not only the new ones.** The base is the upstream mirror, so GitHub lists all our patches again, and they have new hashes after each rebase. Do not rely on the commit list. Under `How`, add a `New patches since <previous>` list and a `Carried over` list from `git range-diff`:
   ```sh
   git range-diff remote/<previous-x-y-z>..release/mobile-<previous> \
                  remote/<x-y-z>..release/mobile-<x-y-z>
   ```
   - `>` is a new patch. List it under new.
   - `=` is carried over unchanged.
   - `!` is carried over, but the diff changed. That is a conflict resolution or a changed upstream context, not new work. Check it in step 7.
   - `<` is a patch that was dropped or squashed. Say so in the PR.

7. **Review the rebase.** Start a read-only subagent. Tell it to compare `git diff remote/<previous> remote/<x-y-z> -- packages/expo-video` with our patch diff. Add its findings to the PR body under `Review of upstream changes`. Fix them only when the user asks. These issues came up in 57 to 58, so check them each time:
   - Android `build.gradle`: `buildFeatures { buildConfig true }` is needed for our `buildConfigField`.
   - Prebuilt artifacts: upstream `publication` (`expo-module.config.json`) and `publishPrebuilds` (`spm.config.json`) bypass the IMA code. Do not ship `local-maven-repo` or `prebuilds`.
   - iOS `VideoView.swift`: `begin`/`endAppearanceTransition` calls must stay balanced with upstream's.
   - Android `VideoPlayer.kt`: upstream-added code that calls `prepare()` or `replace` must still reach our `prepareWithAds` path.
   - Callers of `buildMediaSourceFactory` and `toMediaSource` match our `DefaultMediaSourceFactory` return type.

8. **TODO: build and check.** Run `pnpm build` in `packages/expo-video` and `et check-packages expo-video`. Never hand-edit or stage `build/`.

9. **Prompt the user to test the bare app. Stop here until they answer.** Ask: "Should I test the bare app (`apps/bare-expo`) with IMA on, or will you test it?" Do not continue to step 10 without an answer. The test needs `expo.video.useInteractiveMediaAds=true`, and iOS needs `pod install` in `apps/bare-expo/ios`. Use the `run-bare-expo` skill (untracked in `.claude/skills/`, restored from the stash "Expo bare app builds and release"). That skill says pnpm 10. This repo needs pnpm `^12.2.1`, so ignore the pnpm 10 advice. Mark the PR ready after the test passes.

10. **Set the canary name and version. Never commit this.** Edit `packages/expo-video/package.json` in the working tree only:
   - `name`: `@fredrifoversant/expo-video`
   - `version`: `<x.y.z>-canary-<YYYYMMDD>-<7-char hash>`, for example `58.0.14-canary-20261123-adc2725`. `x.y.z` comes from the branch name. The hash is `git rev-parse --short=7 HEAD`. Take the hash after the last commit, because the package.json edit is never committed.
   Do not stage this file. Revert it with `git checkout packages/expo-video/package.json` after publishing.

11. **TODO: publish.** The user runs it:
   ```sh
   cd packages/expo-video
   npm login
   npm publish --access public --tag canary --ignore-scripts
   ```
   `--ignore-scripts` is required for 58+. Upstream's `prepack` fails on `publishPrebuilds: true` (no xcframework), and it would stage prebuilt artifacts that skip our IMA code. Run `pnpm build` yourself first. Check the tarball with `npm pack --dry-run --ignore-scripts --json`. It must have no `local-maven-repo/` or `prebuilds/` paths.
   `build/` must be fresh first (step 8). If `pnpm build` says `expo-build` is missing, `node_modules` is stale from an older SDK. The repo needs pnpm `^12.2.1` (`volta install pnpm@12`). Then run `pnpm install --frozen-lockfile` at the repo root to relink `@expo/internal-scripts`.

11. **Tag the published commit.** Do this right after publishing, while `HEAD` is still the commit the canary hash came from. Use the exact published version in the tag name:
   ```sh
   git tag fork/expo-video@<x.y.z>-canary-<YYYYMMDD>-<hash> <hash>
   ```
   The `package.json` edit from step 9 is never committed, so this tag is the only record of what was published. `fork/` keeps it apart from upstream's `expo-video@<x.y.z>` tags. List releases with `git tag --list 'fork/*'`.
   For the next release, compare against the previous tag instead of a branch in step 6: `git range-diff remote/<previous-x-y-z>..fork/expo-video@<previous-version> remote/<x-y-z>..release/mobile-<x-y-z>`. The `<previous-version>` is the full canary version. Ask the user before `git push origin --tags`.

Confirm with the user before any `git push`.
