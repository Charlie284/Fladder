# Build and release

## Versioning and workflow entry points

`pubspec.yaml` holds the application version, currently `0.10.3+1`. `.fvmrc` pins Flutter. Build jobs pass the GitHub run number as `--build-number`; `fetch-info` separates the version name from the build suffix and derives a `-nightly` name (`.github/workflows/build.yml`). These are the checked-in workflow definitions, not a claim that every external publishing service is configured or every job currently succeeds.

| Workflow | Triggers | Jobs and results |
| --- | --- | --- |
| `.github/workflows/checks.yaml` | Pull requests targeting `develop`; manual dispatch | `linting`: installs dependencies and runs the Dart analysis action with `fail-on: info` and line length 120. No uploaded artifacts. |
| `.github/workflows/prepare-release.yml` | Manual dispatch | `prepare-release`: uses a GitHub App token, checks out `develop`, reads the version, and if the version tag is new, updates metadata, commits it, merges `develop` into `main`, then pushes `vVERSION`. No uploaded artifacts. |
| `.github/workflows/validate-fastlane.yml` | Pull requests changing `fastlane/**`; pushes to `develop` changing that path | `validate`: runs the Fastlane Supply metadata validator. No uploaded artifacts. |
| `.github/workflows/build.yml` | `v*` tag pushes; pushes to `master`; opened/reopened pull requests matching its three path filters; daily midnight UTC schedule; manual release/nightly dispatch | Build and publishing jobs listed below. |

The build PR filters are `pubspec.yaml`, `.github/workflows/build.yml` and `.fvmrc`. `check-trigger` maps tags to release, dispatches to the chosen type and other qualifying events to development. Scheduled runs check for commits to `develop` in the previous 24 hours and skip if none exist. The explicit push branch is **master**, while release preparation merges into **main**; these are different settings in the code, not interchangeable branch names (`.github/workflows/build.yml`, `.github/workflows/prepare-release.yml`).

## Build workflow jobs and artifacts

All rows below describe `.github/workflows/build.yml`.

| Job | Work | Uploaded artifact names |
| --- | --- | --- |
| `check-trigger` | Determines whether to run and chooses build type. | None |
| `fetch-info` | Reads Flutter/application versions and nightly name. | None |
| `build-android` | Java 17; production flavor; split ABI APKs and, outside development builds, release AAB; release signing uses secrets. | `fladder-android` |
| `build-windows` | Windows build plus Inno Setup installer. | `fladder-windows-portable`, `fladder-windows-installer` |
| `build-ios` | Production IPA build without codesigning; zips `Runner.app` into an unsigned IPA. | `fladder-iOS` |
| `build-macos` | Production macOS build; installs create-dmg and runs `scripts/create_dmg.sh`. | `fladder-macOS` |
| `build-linux` | Linux release bundle, RPATH adjustment and AppImage packaging. | `fladder-linux`, `fladder-linux-appimage` |
| `build-linux-flatpak` | Release/nightly only; checks out submodules, downloads the Linux bundle, runs Flatpak builder. | `fladder-linux-flatpak` |
| `build-web` | Release web bundle; release builds additionally generate a repository-subpath Pages bundle. | `fladder-web`; release-only `fladder-web-pages` |
| `create_release` | Release/nightly only; gathers artifacts and creates the GitHub release. Release builds are drafts; nightly builds replace the nightly release/tag and are prereleases. | Publishes release assets, not another Actions artifact. |
| `release_play_console` | Depends on release creation; uploads AAB to internal/completed for nightly or production/draft otherwise. | Google Play upload; no Actions artifact. |
| `release_web` | Release only; publishes standard/rootless GHCR images and GitHub Pages. | Registry images and Pages deployment. |

Release assets are versioned Android APKs for armeabi-v7a, arm64-v8a and x86_64 plus AAB; Windows portable ZIP and Setup EXE; unsigned iOS IPA; macOS DMG; web and Linux ZIPs; AppImage and zsync; and release-only Flatpak. The Flatpak job also runs for nightly, but `create_release` only attaches the Flatpak bundle for release builds. Winget submission is commented out, not an active job (`.github/workflows/build.yml`).

## Local builds

Complete [development setup](development.md) first. The workflow uses these build commands, with its build-number flags omitted here for local builds (`.github/workflows/build.yml`):

```sh
flutter build apk --release --split-per-abi --flavor production
flutter build appbundle --release --flavor production
flutter build windows
flutter build ipa --no-codesign --flavor production
flutter build macos --flavor production
flutter build linux --release
flutter build web --release
```

Run only the command for your host/target. Android release signing expects `android/app/key.properties` and a keystore as configured in `android/app/build.gradle`; do not commit signing material. The CI workflow supplies signing and publishing credentials through secrets. An unsigned IPA is not an App Store-signed installation package.

## Web containers

Both Dockerfiles package an **existing** `build/web`; neither compiles Flutter. `Dockerfile` uses `nginx:alpine` and exposes port 80. `Dockerfile-rootless` uses the unprivileged Nginx image, switches to user `nginx` and exposes 8080. The release workflow publishes `fladder` and `fladder-rootless` to GHCR for amd64, arm64 and arm/v7 (`Dockerfile`, `Dockerfile-rootless`, `.github/workflows/build.yml`).

```sh
flutter build web --release
docker build -t fladder-local .
docker run --rm -p 8080:80 fladder-local
# Rootless variant:
docker build -f Dockerfile-rootless -t fladder-local-rootless .
docker run --rm -p 8080:8080 fladder-local-rootless
```

`docker-compose.yml` demonstrates `ghcr.io/donutware/fladder:latest` mapped as `80:80`; change the container port to 8080 when selecting the rootless image.

| Environment variable | Actual entrypoint behavior |
| --- | --- |
| `BASE_URL` | Writes `baseUrl` into the served config; used to preset/lock the Jellyfin server. |
| `SEERR_BASE_URL` | Writes `seerrBaseUrl` into the served config. |
| `FLADDER_WEBPATH` | Normalizes a path such as `/fladder/`, rewrites the HTML base href and generates root/subpath Nginx routing. Defaults to `/`. |
| `SEERR_HEADER` | Declared by Dockerfiles and illustrated in Compose, but **not written into config by the current entrypoint**. Do not rely on this variable to inject headers. |

`docker-entrypoint.sh` writes a served configuration file under the web assets, chooses port 80 for UID 0 or 8080 otherwise and starts Nginx. Its JSON generation directly interpolates URL values, so avoid embedded quotes or newlines. The checked-in `config/config.json` includes a `seerrHeader` field, but neither the current entrypoint nor `FladderConfig.fromJson` consumes that field; configure headers through the Seerr connection UI instead (`docker-entrypoint.sh`, `lib/util/fladder_config.dart`, `lib/screens/settings/widgets/seerr_connection_dialog.dart`).

## Linux packaging

### Flatpak

`flatpak/nl.jknaapen.fladder.yaml` uses GNOME runtime/SDK 48 and the ffmpeg-full 24.08 extension. It builds mpv with dependencies including libass, uchardet, libplacebo and the shared-modules LuaJIT recipe, plus Zenity; the Fladder module copies the existing Linux bundle. Initialize the submodule first. The workflow's builder action consumes this manifest and creates `Fladder-Linux.flatpak` (`.gitmodules`, `.github/workflows/build.yml`).

The manifest declares network, display, audio, home-directory and hardware-acceleration access. Verify its referenced assets and available Flatpak SDKs before attempting a local package build. CI currently selects a gnome-46 builder container while the manifest requests runtime 48; no claim of successful packaging is made here (`flatpak/nl.jknaapen.fladder.yaml`, `.github/workflows/build.yml`).

### AppImage

After a Linux build, `appimage-builder --recipe AppImageBuilder.yml` copies `build/linux/x64/release/bundle` into AppDir and produces `Fladder_x86_64.AppImage`, with GitHub-release zsync update information. CI downloads appimage-builder 1.1.0, substitutes the version and archives the AppImage and zsync output (`AppImageBuilder.yml`, `.github/workflows/build.yml`).

### Snap

`snap/gui/` contains a desktop entry and icon, but there is no checked-in snapcraft recipe or active Snap build job. This tree is not enough to document a supported Snap packaging command (`snap/gui/app_icon.desktop`, `.github/workflows/build.yml`).

## Windows installer

`windows/windows_setup.iss` packages the x64 release directory recursively into `fladder_setup.exe`. It defaults to a per-user install under local application data, offers a desktop shortcut and optionally removes application data during uninstall while warning that synced files remain. CI supplies `FLADDER_VERSION` and archives `windows/Output/fladder_setup.exe` (`windows/windows_setup.iss`, `.github/workflows/build.yml`).

## Store metadata and release preparation

`fastlane/metadata/` contains Android store metadata. There is no checked-in Fastfile in `fastlane/`; Google Play upload is performed by the workflow action, not a documented Fastlane lane. Prepare Release writes a version-named English changelog and Flatpak release metadata before merging/tagging. It skips these changes when `vVERSION` already exists; bump `pubspec.yaml` deliberately before dispatching (`.github/workflows/prepare-release.yml`, `.github/workflows/validate-fastlane.yml`).

`altstore.json` is an AltStore source containing app identity, screenshots and version/download entries. The checked-in entries are historical; none of the four workflows updates this file. Do not infer an automated AltStore release from the IPA build (`altstore.json`, `.github/workflows/`).
