[简体中文](./README.md) · **English**

# Loom Build

Turn an Android project on your phone into an APK with GitHub Actions — no computer required.

The interface follows the visual system of [Loom](https://github.com/CelestialCatkins/Loom): paper / ink / a single copper accent, serif headings, monospaced labels, and hairline dividers.

## What it does

1. **Connect GitHub** (once): paste a Personal Access Token.
2. **Pick a project**: choose the root directory of an Android Studio project with the system directory picker.
3. **Upload and build**: create a new repository (or reuse one), write all the files, inject `build-apk.yml`, and trigger Actions.
4. **Fetch the APK**: poll the run status, download the artifact on success, and open the system installer directly.

## Requirements

- Android 8.0 (API 26) or later.
- A GitHub account.
- A target project that is a **Gradle-based Android project** (the root contains `settings.gradle(.kts)` or `build.gradle(.kts)`).

## Create a token

Open <https://github.com/settings/tokens/new?scopes=repo,workflow> and check:

| Permission | Why it is needed |
|---|---|
| `repo` | Create private repositories and write files |
| `workflow` | Commit `.github/workflows/*.yml` |

The token looks like `ghp_…`. Once pasted into the app, it is stored only in the app's private directory.

## How it builds

After upload, `.github/workflows/build-apk.yml` is written to the target repository. It is equivalent to:

- JDK 17 + Gradle 8.7 (the project's `gradlew` wrapper is preferred when present)
- Use the Android SDK preinstalled on the runner directly, installing platform 34/35/36 and build-tools
- `assembleDebug` + `assembleRelease`
- Collect all APKs and upload them as an artifact

> Note: this deliberately avoids `android-actions/setup-android`, because it runs `sdkmanager tools` and that package has been removed from the SDK repository, which makes the step fail outright.

## Scope and known limitations

- Gradle projects only; no attempt is made to support CMake/Flutter or other build systems.
- A single file may be at most 12 MB, and there may be at most 800 files in total (anything beyond that is skipped).
- Actions minutes for private repositories count against your account quota.
- The token is stored in the app's private directory (not encrypted at rest). When you no longer need it, unbind it in Settings and revoke it on GitHub.

## Building

```bash
# No local gradlew needed
gradle :app:assembleDebug
```

Or push to `main` and let this repository's Actions produce the APK.

## New in v1.1.0

### 1. Automatic signing

On the first build, the pipeline uses `keytool` to generate a 2048-bit RSA key, writes it to `.loom/signing/` in the repository, and commits it (`[skip ci]`, so no build loop is triggered). Every later build signs the same package with `zipalign` + `apksigner`, so **upgrade installs do not hit signature conflicts**.

> Trade-off: the key material lives in the repository (not in GitHub Secrets). The GitHub Secrets API requires libsodium sealed box encryption, which would mean shipping a whole native crypto library inside the app; for a private repository, keeping the key with the repo is the simpler and more reliable choice. If the repository is not private, do not use this to sign releases.

### 2. Build an existing repository directly

At the top of the project screen you can switch to “已有仓库” (Existing repository): enter `owner/name`, or tap “载入我的仓库” (Load my repositories) and pick from the list. The app will:

1. Check for `.github/workflows/build-apk.yml` and inject it if missing;
2. Dispatch `workflow_dispatch` on the default branch;
3. Wait for the run and download the artifacts.

Local files are not uploaded.

### 3. Build history

The last 30 builds are persisted locally: repository, time, success or failure, and the path of that build's APK. Scroll to the bottom of the build screen to look back; if a build succeeded and the file is still there, you can tap “安装” (Install) directly, and tapping the row opens the cloud logs.
