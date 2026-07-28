---
name: expo-sdk-upgrade
description: Reference for upgrading Expo SDK and React Native versions safely. Use when the user wants to upgrade Expo SDK, bump react-native, or asks why the app broke after an upgrade. Covers upgrade order, dependency alignment, expo-doctor, config plugin churn and OTA update pitfalls.
user-invocable: false
---

# Expo SDK upgrades without a lost weekend

## Ground rules

- One SDK version at a time. Two majors in one jump doubles the breakage and halves your ability to bisect it.
- Upgrade on a branch and keep the app runnable on main. You will want to diff against a working tree.
- Read the SDK changelog before touching anything. The "Breaking changes" and "Deprecations" sections are short and they are the whole game.
- Budget real time for it. A clean upgrade is an afternoon; a project with many native libraries can take days.

## The order of operations

```bash
# 1. Update the expo package itself to the target SDK
#    (57 was current when this was written; check expo.dev/changelog)
npm install expo@^57.0.0

# 2. Let Expo align every SDK-managed dependency to the versions it was tested with
npx expo install --fix

# 3. Ask for a health report
npx expo-doctor
```

`expo install --fix` is the step people skip and then pay for. React Native, React, Reanimated, gesture-handler, screens and the rest are version-coupled per SDK. Mixing "latest of everything" is the most common source of upgrade crashes.

Then:

```bash
# 4. Regenerate native projects if the repo uses CNG (no manual edits under ios/ or android/)
npx expo prebuild --clean

# 5. Run both platforms before declaring victory
npx expo run:ios
npx expo run:android
```

## What expo-doctor tells you and what it cannot

expo-doctor validates dependency versions against React Native Directory metadata and flags unmaintained or incompatible packages. Take its "unmaintained" flags seriously: abandoned native libraries block more upgrades than anything else, and swapping one is often cheaper than patching it.

It cannot catch runtime behavior changes. A library can install cleanly and still break at runtime because of a renamed prop or a New Architecture difference, which is why step 5 is non-negotiable.

## Version-coupled packages

- react-native-reanimated: tightly coupled to the React Native version. Never pin it ahead of or behind what `expo install` chooses.
- react-native-gesture-handler, react-native-screens, react-native-safe-area-context: same rule.
- Anything with a config plugin: plugin APIs shift between SDKs, so a library's plugin can throw during prebuild even when its runtime code is fine. The error names the plugin; check that library's release notes for the SDK you are targeting.

## The OTA trap that takes apps down

Runtime version separates JS updates from native binaries. After an SDK upgrade the native runtime changes, so:

- Never publish an OTA update from the upgraded branch to a channel that old binaries are still reading, unless the runtime version also changed and therefore fences them off.
- With `"runtimeVersion": { "policy": "appVersion" }` bumping the app version fences updates automatically. That policy is the safe default for teams that do not want to think about it.
- The failure mode when this goes wrong is the worst kind: the store binary crashes on launch for every existing user until they update through the store.

Check `runtimeVersion`, `channel` and the update URL in app config as part of every upgrade, not after the incident.

## Monorepo and lockfile notes

- npm workspaces hoist aggressively. A singleton package (react, react-native) resolving twice produces "Invalid hook call" or duplicate-module native errors. Use overrides at the workspace root to pin singletons, and verify with `npm ls react-native`.
- If dependency resolution seems stuck on stale versions after edits, do a full reinstall (`rm -rf node_modules && npm install`), but keep the lockfile unless you intend to re-resolve the world.

## After it builds

- Click through auth, purchases, camera/media, push and deep links. These touch native modules and break silently.
- Ship to the store lane, not OTA, for the first release after any upgrade.
- Delete any patch-package patches the upgrade made obsolete. Stale patches against moved code fail in ways that look supernatural.
