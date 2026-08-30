# cambium-ci

Build and release pipeline for **Cambium**. This repository holds no application
code — only the workflow. The app lives in the private repo `KR4010/cambium`
and is checked out here at build time.

## Why the split

GitHub gives public repositories unlimited Actions minutes, **macOS runners
included**. A private repository on the free plan gets about 200 macOS minutes a
month, and a single iOS build spends a large slice of that. So the workflow lives in
a public repo and reaches into the private one with a PAT.

Builds use `eas build --local`, so the runner does the compiling and no EAS build
credits are consumed. EAS is used only for credentials and for submission.

**Nothing secret belongs in this repository.** It is public. Every credential is a
GitHub Actions secret, and the workflow deliberately never prints one — not the
value, not its length, not its first characters.

## Secrets

| Secret | What it is |
|---|---|
| `APP_REPO_PAT` | Fine-grained PAT, read-only **Contents** on `KR4010/cambium` |
| `EXPO_TOKEN` | Expo access token — expo.dev → Account → Access tokens |
| `EXPO_PUBLIC_REVENUECAT_KEY_IOS` | RevenueCat public SDK key, iOS |
| `EXPO_PUBLIC_REVENUECAT_KEY_ANDROID` | RevenueCat public SDK key, Android |
| `EXPO_APPLE_ID` | Apple ID used for App Store Connect |
| `EXPO_TEAM_ID` | Apple Developer team id |
| `ASC_APP_ID` | App Store Connect app id (the numeric one) |
| `EXPO_APPLE_APP_SPECIFIC_PASSWORD` | appleid.apple.com → Sign-In and Security → App-Specific Passwords |

The PAT needs **Contents: read** and nothing else. It cannot write, so a compromised
runner cannot alter the app repo.

## Triggers

- **Push to `main`** on the app repo → checks, iOS build, TestFlight.
  (Requires a repository-dispatch or a mirrored push; see *Wiring the trigger*.)
- **Manual** → Actions → *Cambium CI/CD* → *Run workflow*, choose `ios`,
  `android`, `all`, or `none` for checks only.

## Wiring the trigger

A push to the private app repo cannot start a workflow in this repo on its own. Two
options:

1. **Manual runs.** Simplest. Nothing to configure.
2. **Repository dispatch.** Add a tiny workflow in the app repo that fires
   `repository_dispatch` at this one on push to `main`. Needs a PAT with
   `Actions: write` on this repo.

Start with manual. Add the dispatch once the pipeline has run green a few times.

## What the workflow refuses to do

- **Build without a RevenueCat key.** The paywall degrades gracefully when the store
  is unavailable, which is correct at runtime and wrong in a release — it would ship
  a paywall that cannot sell. The build fails with a clear message instead.
- **Print secrets.** No `cat .env`, no key lengths, no prefixes.
- **Skip the gate.** Typecheck, lint at zero warnings, the full test suite and a
  bundle check all pass before a runner is spent on a build.
