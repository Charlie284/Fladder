# Development

## Prerequisites

Use Flutter **3.35.7**, the version pinned in `.fvmrc` and read by `.github/workflows/checks.yaml` and `.github/workflows/build.yml`. FVM is optional: either select that SDK yourself or prefix Flutter/Dart commands with `fvm`. Run `flutter doctor` and install the toolchain for your target. The existing setup guide recommends Android Studio and the VS Code Flutter/Dart extensions (`DEVELOPEMENT.md`). Android CI uses Java 17 (`.github/workflows/build.yml`). Apple targets require an Apple development host; CI builds them on macOS, and Windows CI uses Windows 2022.

For Ubuntu Linux builds, CI installs the following packages (`.github/workflows/build.yml`):

```sh
sudo apt-get update
sudo apt-get install ninja-build libgtk-3-dev libmpv-dev patchelf cmake clang libfuse2
```

`DEVELOPEMENT.md` specifically calls out `libmpv-dev`. `patchelf` and `libfuse2` also support the packaging steps. `AppImageBuilder.yml` copies an existing Linux bundle; it does not define another apt dependency list. Flatpak dependencies are built in its runtime, not installed through apt (`flatpak/nl.jknaapen.fladder.yaml`).

## Clone and install

```sh
git clone --recurse-submodules https://github.com/DonutWare/Fladder.git
cd Fladder
git switch develop
flutter pub get
```

The clone URL follows `README.md`. `.gitmodules` declares `flatpak/shared-modules`, sourced from Flathub. For an existing checkout, use `git submodule update --init --recursive`, especially before Flatpak packaging. Dependencies and generator packages are declared in `pubspec.yaml`.

## Generate code

Run from the repository root (`DEVELOPEMENT.md`, `.vscode/tasks.json`):

```sh
dart run build_runner build
# During development, use this instead of the one-shot build:
dart run build_runner watch
```

| Generator | Inputs and output |
| --- | --- |
| Swagger and Chopper | `swagger/jellyfin-open-api.json` to the client, DTOs and converter under `lib/jellyfin/`; `build.yaml` limits Swagger generation to Jellyfin. |
| Chopper | Annotated services, including `lib/seerr/seerr_chopper_service.dart`, to `.chopper.dart` implementations. |
| Riverpod | Annotated providers such as `lib/providers/api_provider.dart` to `.g.dart` provider code. |
| Freezed and JSON Serializable | Model declarations such as `lib/models/account_model.dart` to `.freezed.dart` and `.g.dart` parts. |
| Dart Mappable | Annotated models such as `lib/models/item_base_model.dart` to `.mapper.dart` parts. |
| Drift | `lib/models/syncing/database_item.dart` to `lib/models/syncing/database_item.g.dart`. |
| AutoRoute | `lib/routes/auto_router.dart` and annotated screen/page files to `lib/routes/auto_router.gr.dart`. |

Generator dependencies are in `pubspec.yaml`; source selection and builder options are in `build.yaml`. Read those restrictions before adding generator inputs. For the architectural roles of generated code, see [architecture](architecture.md).

### Native bridges

There is no regeneration shell script under `scripts/` or `pigeons/`. The actual task is in `.vscode/tasks.json` and runs:

```sh
for f in pigeons/*.dart; do dart run pigeon --input "$f"; done
```

Its Windows equivalent is:

```powershell
Get-ChildItem -Path pigeons/*.dart | ForEach-Object { dart run pigeon --input $_.FullName }
```

Each definition has `@ConfigurePigeon` output paths for Dart under `lib/src/` and Kotlin or Swift under the native platform tree, for example `pigeons/video_player.dart` and `pigeons/directory_bookmark.dart`. Pigeon runs separately from build_runner.

### Localization

```sh
flutter gen-l10n
```

`l10n.yaml` reads `lib/l10n/`, uses `app_en.arb` as the template and writes `app_localizations.dart` under `lib/l10n/generated/`. See [localization](localization.md) for strings, placeholders and languages.

## Format, analyze and test

```sh
dart format --line-length 120 ./lib/
flutter analyze --fatal-infos
flutter test
```

The format command follows `DEVELOPEMENT.md`. `analysis_options.yaml` includes recommended lints, custom_lint and generated-file exclusions. CI uses `ValentinVignal/action-dart-analyze` with `line-length: 120` and `fail-on: info`; the analysis command above expresses that severity locally (`.github/workflows/checks.yaml`).

`test/pip_manager_test.dart` tests PiP setup, support, entry, disable, events and disposal with a fake client. `test/widget_test.dart` still contains a counter-style smoke test that pumps `Main`; do not assume this represents the current UI or that the full suite passes. The checks workflow does not run `flutter test` (`test/`, `.github/workflows/checks.yaml`).

## Run a target

Select installed devices with `flutter devices`. These development commands use the checked-in platform targets and flavors (`android/app/build.gradle`, `ios/Runner.xcodeproj/xcshareddata/xcschemes/`, `macos/Runner.xcodeproj/xcshareddata/xcschemes/`, `.vscode/launch.json`, `.github/workflows/build.yml`):

| Target | Command |
| --- | --- |
| Android device/emulator | `flutter run -d DEVICE_ID --flavor development` |
| iOS device/simulator on macOS | `flutter run -d DEVICE_ID --flavor development` |
| macOS | `flutter run -d macos --flavor development` |
| Windows | `flutter run -d windows` |
| Linux | `flutter run -d linux` |
| Web in Chrome | `flutter run -d chrome --web-port 9090` |

Replace `DEVICE_ID` with a device identifier. Android defines `development` with an application ID suffix `.dev` and `production` without one. Apple schemes also include both names. The launcher task generates separate development/production artwork with `dart run icons_launcher:create --flavors development,production`; its configurations are `icons_launcher-development.yaml` and `icons_launcher-production.yaml` (`.vscode/tasks.json`).

Web startup reads `config/config.json`; unset URLs allow normal server selection (`lib/bootstrap/app_bootstrap.dart`, `lib/util/fladder_config.dart`). The hosted web build requires HTTPS server connections as noted in `README.md`. Container configuration is covered in [build and release](build-and-release.md).

## Demo server

The existing developer guide points to the public Jellyfin demo at <https://demo.jellyfin.org/stable/web/> (`DEVELOPEMENT.md`). Its availability and credentials are managed externally; this documentation does not promise a working account. This is distinct from Fladder's internal fake API in `lib/fake/fake_jellyfin_open_api.dart`, selected by `lib/providers/service_provider.dart`.
