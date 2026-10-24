# SiYuan (Fork) - Android

A community fork of [siyuan-note/siyuan-android](https://github.com/siyuan-note/siyuan-android) building an account-free Android client with third-party sync. For the kernel and frontend changes, see [antebudimir/siyuan](https://github.com/siyuan-note/siyuan).

* General SiYuan questions and upstream bugs: [siyuan-note/siyuan issues](https://github.com/siyuan-note/siyuan/issues)
* Fork-specific problems (sync, build, signing): [open an issue here](https://github.com/antebudimir/siyuan-android/issues)

## Fork notes

* Single `fork` flavor (renamed from upstream `official`), arm64-v8a only, app label `SiYuan Fork`
* Own version line: `siyuanVersionName` follows the fork tag `X.Y.Z-fork.N`, `siyuanVersionCode` counts independently
* Aliyun Maven mirrors removed; Google/Maven Central direct
* Releases are built by [.github/workflows/fork-release.yml](.github/workflows/fork-release.yml): it builds the kernel `.aar` (with `fts5 sqlcipher` tags - sqlcipher is mandatory on mobile) and frontend `app.zip` from the matching `antebudimir/siyuan` ref, signs with the `ANDROID_KEYSTORE_*` repository secrets, and attaches the APK to the tag's GitHub Release

## Build

### CI (recommended)

1. Push the release tag to `antebudimir/siyuan` first (also produces the Docker image and Linux release)
2. Push the same tag here; `fork-release.yml` does the rest
3. For a test build without a tag, run the workflow manually via `workflow_dispatch` and download the APK artifact

### Local

1. Compile the kernel and frontend following [antebudimir/siyuan](https://github.com/antebudimir/siyuan), then place:
   * `kernel.aar` in `app/libs/`
   * `app.zip` (containing `appearance`, `guide`, `stage`, `changelogs`) in `app/src/main/assets/`
2. Copy `signings.templates.gradle` to `signings.gradle` and fill in your keystore details; place the keystore at `app/siyuan-android.jks` (never commit it)
3. Build:
   ```shell
   ./gradlew clean assembleForkRelease
   ```
   The APK is written to `app/build/outputs/apk/fork/release/siyuan-${versionName}-fork-release.apk`

Directory structure reference:

![project-tree](project-tree.png)

![app.zip](app-zip.png)
