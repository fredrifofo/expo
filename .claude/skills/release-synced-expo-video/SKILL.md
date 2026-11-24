---
name: release-synced-expo-video
description: Rebase our expo-video patches onto the latest stable upstream expo-video from npm and open a draft PR. Use only when asked to sync or upgrade our expo-video fork to a new upstream version. Do not use it for a plain "release" request: a release is always a canary, in publish-canary-expo-video.
---

# Sync expo-video with upstream

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

   **The PR commit list shows every patch, not only the new ones.** The base is the upstream mirror, so GitHub lists all our patches again, and they have new hashes after each rebase. Do not rely on the commit list. After the first published canary, compare against the previous release tag instead of a branch: `git range-diff remote/<previous-x-y-z>..release/mobile-<previous-x-y-z>-canary-<YYYYMMDD>-<hash> remote/<x-y-z>..release/mobile-<x-y-z>`. Find the tag with `git tag --list 'release/*-canary-*'`. Under `How`, add a `New patches since <previous>` list and a `Carried over` list from `git range-diff`:
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

8. **Prompt the user to test the bare app. Stop here until they answer.** Ask: "Should I test the bare app (`apps/bare-expo`) with IMA on, or will you test it?" Do not publish without an answer. The test needs `expo.video.useInteractiveMediaAds=true`, and iOS needs `pod install` in `apps/bare-expo/ios`. Use the `run-bare-expo` skill (untracked in `.claude/skills/`, restored from the stash "Expo bare app builds and release"). That skill says pnpm 10. This repo needs pnpm `^12.2.1`, so ignore the pnpm 10 advice. Mark the PR ready after the test passes.

Build, the canary name and version in `package.json`, publish, and tagging are in the `publish-canary-expo-video` skill. Use it after the PR is ready.

Confirm with the user before any `git push`.
