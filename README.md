<div align="center">

# claude-code-react-native

**Claude knows React. It doesn't know why your pod install just failed.**

A Claude Code plugin that adds the React Native knowledge you only get from
shipping apps: build failure triage, SDK upgrades that don't eat a weekend,
Hermes crash decoding, push notification debugging, subscription pipelines
and store submission checklists.

[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-plugin-blue)](https://code.claude.com/docs/en/discover-plugins)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](#contributing)

</div>

---

## Why this exists

Claude Code is genuinely good at React Native application code. Where it
struggles is the layer underneath: the Gradle log with the real error buried
at line 400, the OTA update that bricks old binaries because the runtime
fence moved, the sandbox subscription that expires mid-test, the push token
that belongs to a different delivery service than the one you sent through.

That knowledge lives in the heads of people who shipped apps and got burned.
This plugin writes it down in a form Claude loads exactly when it's needed.
Each skill is a distilled runbook, not a doc dump: what breaks, in what
order to check, and which fixes are safe.

## Install

Inside Claude Code:

```
/plugin marketplace add AnilBurcu/claude-code-react-native
/plugin install rn@claude-code-react-native
```

That's it. The knowledge skills activate on their own when your conversation
touches their territory. The workflow skills are yours to invoke.

To try it from a clone without installing anything:

```
git clone https://github.com/AnilBurcu/claude-code-react-native
claude --plugin-dir ./claude-code-react-native/plugins/rn
```

Tested with Claude Code 2.1. The skill listing costs about 1,200 tokens of
descriptions per turn (`/skill-doctor` shows the exact figure on your
machine); a skill's full text loads only when it is used.

## What's inside

### Knowledge that loads itself

These activate automatically when the task matches. You never call them;
Claude reads them when your build fails or your crash report is unreadable.

| Skill | Kicks in when |
|---|---|
| `native-build-errors` | pod install, Gradle, Xcode or linker failures |
| `expo-sdk-upgrade` | upgrading Expo SDK / React Native, post-upgrade breakage |
| `new-architecture` | Fabric, TurboModules, codegen errors, library compatibility |
| `hermes-crashes` | minified stack traces, Sentry symbolication, SIGSEGV, ANRs |
| `push-notifications` | FCM / APNs setup, tokens, notifications that never arrive |
| `in-app-purchases` | RevenueCat, sandbox weirdness, webhooks, entitlement design |
| `supabase-mobile-auth` | native Apple / Google sign-in, session refresh, RLS from mobile |
| `rn-performance` | slow lists, re-renders, janky animations, startup time |
| `eas-release` | build profiles, OTA update safety, store submission |
| `deep-linking` | universal links / app links, AASA and assetlinks traps, cold-start links |
| `monorepo-metro` | workspace apps, duplicate react-native, hoisting, EAS in monorepos |
| `patching-libraries` | patch vs fork vs replace decisions, patch hygiene across upgrades |
| `log-triage` | pulling the one error line out of thousand-line build and device logs |

### Workflows you invoke

| Command | What it does |
|---|---|
| `/rn:fix-build [ios\|android]` | reproduces the failing build, finds the first real error, fixes it, rebuilds to verify |
| `/rn:upgrade-sdk [version]` | guided SDK upgrade with preflight, changelog review and both-platform verification |
| `/rn:new-arch-check` | audits every dependency for New Architecture support and abandonment risk |
| `/rn:perf-audit [path]` | sweeps the code for the performance antipatterns that actually cost milliseconds |
| `/rn:prep-release [platform]` | pre-submission audit: versioning, OTA fence, permissions, secrets, store policy |

### Agents for the long jobs

Subagents keep multi-round investigations out of your main conversation:

| Agent | Use for |
|---|---|
| `build-doctor` | build failures that need several rounds of log digging |
| `crash-analyst` | production crash reports, from minified trace to root cause |
| `perf-auditor` | read-only performance sweeps of larger codebases |
| `release-checker` | full pre-submission verification with evidence |

## What changes in practice

Say the Android build breaks after you add a camera library. Without the
plugin, the usual loop is generic: clean caches, delete node_modules, hope.
With `native-build-errors` loaded, Claude captures the log to a file, greps
for the deepest `Caused by:` instead of reading the last line, quotes the
actual error (say, a minCompileSdk mismatch in AAR metadata), applies the
smallest fix for that class of failure, and rebuilds to verify before
declaring anything solved.

Same shape everywhere else: sandbox subscriptions expiring mid-test get
read as normal sandbox behavior instead of a bug hunt, and an OTA publish
after a native change gets stopped at the runtime fence instead of
shipping a crash to your existing users.

## Works with

- Expo projects (CNG/prebuild) and bare React Native
- The skills are written version-durable: they teach the method (find the
  first error, check the pairing, verify the fence) rather than pinning
  version numbers that rot

## Other agents and editors

The skills follow the open Agent Skills format, so they work outside
Claude Code too. The skills CLI finds all of them in this repo:

```
npx skills add AnilBurcu/claude-code-react-native
```

Pick the ones you want and it installs them for the agents it detects.

You can also use any skill fully by hand: copy its folder from
`plugins/rn/skills/` into `~/.claude/skills/` (personal) or
`.claude/skills/` (per project). The agents go into `~/.claude/agents/`
the same way.

There's also a [CLAUDE.md template](templates/CLAUDE.md) with the project
rules that prevent the expensive mistakes (npx expo install over npm
install, never hand-editing generated native folders, the OTA rule). Copy
what applies, delete the rest.

## Contributing

The bar for a PR: it happened to you in a real project. This repo is field
notes, not documentation coverage, and it stays useful only if every line
earns its place. Fixes for things that changed in newer SDK or store policy
versions are especially welcome.

Keep the writing plain. Short sentences, no marketing adjectives, name the
exact error strings people will search for.

## License

[MIT](LICENSE)
