---
name: run-bare-expo
description: Build, launch and drive the bare-expo test app (apps/bare-expo) on the iOS simulator and Android emulator. Use when asked to run bare-expo, the bare app, or the Expo test suite app on iOS or Android, or to confirm a change works in the real app.
---

# Run bare-expo on iOS and Android

All paths are relative to the repo root. Run the steps in order and skip any that are already satisfied.

## 1. Environment

- **pnpm 10.** The root `package.json` requires `engines.pnpm: ^10.33.0`, and pnpm 11 refuses to run. If `pnpm -v` shows 11, use `npx -y pnpm@10 <cmd>` everywhere below.
- **direnv.** If `.envrc` is blocked, don't run `direnv allow` on the user's behalf. Export its variables for each command instead:
  ```bash
  R="$(git rev-parse --show-toplevel)"
  export PATH="$R/bin:$PATH" EXPO_ROOT_DIR="$R" CCACHE_BASEDIR="$R" EXPO_USE_SOURCE=1 \
    NODE_ENV=development EXPO_ROUTER_DISABLE_RN_NAVIGATION_CHECK=1 CI=1
  ```
  `bin/pod` wraps `bundle exec pod`, so `bin/` must be on the PATH.
- **Ruby gems.** If `bundle check` reports missing gems, run `bundle install` in the repo root. Otherwise `pod install` fails with `Bundler::GemNotFound`.

## 2. Dependencies and CLI

- `npx -y pnpm@10 install`. If the existing `node_modules` came from a different pnpm major, pnpm prompts to purge it and aborts when there's no TTY. Add `--config.confirmModulesPurge=false`.
- `npx expo` needs `packages/@expo/cli/build/src/index.js`. If it's missing, build the CLI with `npx -y pnpm@10 --dir packages/@expo/cli build`. The user normally runs builds themselves, so ask first.

## 3. Clear stale native caches (only after a dependency or React Native version change)

A reinstall moves packages to new `node_modules/.pnpm/...` paths, and these caches keep pointing at the old ones.

| Symptom | Fix |
|---|---|
| Gradle: `Configuring project ':<lib>' without an existing directory` | `rm apps/bare-expo/android/build/generated/autolinking/autolinking.json` |
| Gradle: `Hash for artifact '…jar' is null` | `(cd apps/bare-expo/android && ./gradlew --stop)` |
| CocoaPods: `could not find compatible versions for pod "<X>"`, differs from `Pods/Local Podspecs` | `rm -rf "apps/bare-expo/ios/Pods/Local Podspecs"`, then `pod install` in `apps/bare-expo/ios` |

Don't fix the CocoaPods error with `pod update <X>` one pod at a time. It moves on to the next stale pod, and there are about 185.

## 4. Start Metro

If port 8081 is taken (`lsof -i :8081`), it may be another project's Metro, so leave it alone and use 8082. Always start Metro with `--clear` to reset its cache. A stale cache serves old module versions and gives errors such as a Worklets or React Native version mismatch. Run this in the background from `apps/bare-expo`:

```bash
npx expo start --clear --port 8082 > /tmp/bare-expo-metro.log 2>&1
```

## 5. Build and launch

Run both in the background from `apps/bare-expo` (with the env from step 1). They can build in parallel. A cold build takes about 5 minutes for Android and longer for iOS.

```bash
# Android: --device takes the AVD name, not the adb serial (`adb -s emulator-5554 emu avd name`)
adb reverse tcp:8082 tcp:8082
npx expo run:android --port 8082 --device <AVD_NAME> > /tmp/bare-expo-android.log 2>&1

# iOS: --device takes the simulator UDID (`xcrun simctl list devices booted`)
npx expo run:ios --port 8082 --device <SIM_UDID> > /tmp/bare-expo-ios.log 2>&1
```

- Don't pass `--no-bundler` together with `--port`, because the two flags are mutually exclusive. With `--port`, the CLI reuses the Metro that's already running.
- To watch progress, grep the logs for `BUILD SUCCESSFUL|BUILD FAILED|What went wrong|Build Succeeded|error: |CommandError|Opening on`.

## 6. Connect and verify

- **iOS shows "Searching for development servers".** Point the dev client at Metro:
  ```bash
  xcrun simctl openurl <SIM_UDID> "exp+bare-expo://expo-development-client/?url=http%3A%2F%2Flocalhost%3A8082"
  ```
- **Android shows a blank screen with only the gear button for 10–30 s** on its first launch while it loads assets. Check logcat for `Running "main"` before you treat it as a failure.
- **Screenshots.** Use `xcrun simctl io <SIM_UDID> screenshot /tmp/ios.png` and `adb exec-out screencap -p > /tmp/android.png`. When it works, you see "Expo Test Suite" with Tests / Playground / APIs / Components tabs. Tap a tab to confirm the app responds (`adb shell input tap <x> <y>`; the Android app ID is `dev.expo.payments`).
