---
name: publish-canary-expo-video
description: Publish a canary build of our patched expo-video (`@fredrifoversant/expo-video`) from a `release/mobile-<x-y-z>` branch and tag the published commit. Use when asked to release, publish, or ship expo-video, or to "run the release skill". A release is always a canary. Run release-synced-expo-video first only when the user asks to sync to a new upstream version.
---

# Publish a canary expo-video

A "release" of our expo-video is always a canary. There is no other release type. When the user says "run the release skill", use this skill, not `release-synced-expo-video`.

Publish `release/mobile-<x-y-z>` after `release-synced-expo-video` is done.

Start from a clean checkout of `release/mobile-<x-y-z>`. The bare-app test (sync skill, step 9) must have passed.

## Steps

1. **Build.** Run `pnpm build` in `packages/expo-video` and `et check-packages expo-video`. Never hand-edit or stage `build/`.
   If `pnpm build` says `expo-build` is missing, `node_modules` is stale from an older SDK. The repo needs pnpm `^12.2.1` (`volta install pnpm@12`). Then run `pnpm install --frozen-lockfile` at the repo root to relink `@expo/internal-scripts`.

2. **Set the canary name and version. Never commit this.** Edit `packages/expo-video/package.json` in the working tree only:
   - `name`: `@fredrifoversant/expo-video`
   - `version`: `<x.y.z>-canary-<YYYYMMDD>-<7-char hash>`, for example `58.0.14-canary-20261123-adc2725`. `x.y.z` comes from the branch name. The hash is `git rev-parse --short=7 HEAD`. Take the hash after the last commit, because the package.json edit is never committed.

   Do not stage this file. Revert it with `git checkout packages/expo-video/package.json` after publishing.

3. **Check the tarball.**
   ```sh
   cd packages/expo-video
   npm pack --dry-run --ignore-scripts --json
   ```
   It must have no `local-maven-repo/` or `prebuilds/` paths.

4. **Publish.** The user runs it:
   ```sh
   cd packages/expo-video
   npm login
   npm publish --access public --tag canary --ignore-scripts
   ```
   `--ignore-scripts` is required for 58+. Upstream's `prepack` fails on `publishPrebuilds: true` (no xcframework), and it would stage prebuilt artifacts that skip our IMA code.

5. **Tag the published commit.** Do this right after publishing, while `HEAD` is still the commit the canary hash came from. Use the exact published version in the tag name:
   ```sh
   git tag release/mobile-<x-y-z>-canary-<YYYYMMDD>-<hash> <hash>
   ```
   The `package.json` edit from step 2 is never committed, so this tag is the only record of what was published. The name starts with the release branch name, so the tag sorts next to its branch. List releases with `git tag --list 'release/*-canary-*'`.
   The next sync compares against this tag. See step 6 of `release-synced-expo-video`.

6. **Revert the working tree.** `git checkout packages/expo-video/package.json`.

Confirm with the user before any `git push`, including `git push origin --tags`.
