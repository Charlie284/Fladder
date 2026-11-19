# Localization

## Translation workflow

Translations are contributed through [Weblate](https://hosted.weblate.org/engage/fladder/), linked from `README.md` and `CONTRIBUTING.md`. The source resources are the ARB files in `lib/l10n/`. English is the template; translated files follow the existing names such as `lib/l10n/app_fr.arb` and `lib/l10n/app_pt_BR.arb` (`l10n.yaml`).

`pubspec.yaml` enables Flutter localization generation. `l10n.yaml` configures:

| Setting | Value |
| --- | --- |
| Resource directory | `lib/l10n/` |
| Template | `app_en.arb` |
| Output directory | `lib/l10n/generated/` |
| Output file | `app_localizations.dart` |
| Nullable getter | `false` |

## Add or change a string

1. Add a named message to `lib/l10n/app_en.arb`. Follow its paired `@messageName` metadata convention; describe ambiguous text and declare placeholders with their types. Existing messages such as `actor` demonstrate ICU plural syntax and typed placeholders.
2. Translate the same key through Weblate or the corresponding ARB resources. Preserve placeholder names and ICU structure rather than translating identifiers.
3. Run from the repository root:

   ```sh
   flutter gen-l10n
   ```

4. Use the generated getter or method from `context.localized`, the extension in `lib/util/localization_helper.dart`. Do not hand-edit generated Dart. Existing UI examples are in `lib/screens/settings/client_sections/client_settings_visual.dart`.
5. Check the affected UI in the chosen language, including interpolation, plural cases and text fit. Generation alone does not establish that the translated layout fits.

## Add a language

Add an ARB resource following the existing `app_LANGUAGE.arb` naming and `@@locale` convention; use existing regional/script examples such as `lib/l10n/app_pt_BR.arb` and `lib/l10n/app_zh_Hant.arb`. Set `nativeName` to the language's own display name, as in `lib/l10n/app_fr.arb`, then regenerate.

There is no separate application-maintained locale list to extend in the language picker: both it and `MaterialApp.router` use generated `AppLocalizations.supportedLocales` (`lib/screens/settings/client_sections/client_settings_visual.dart`, `lib/main.dart`). Verify that the new locale appears there after generation. Do not add entries to generated output manually.

The app chooses the saved language or the platform locale. Resolution tries an exact supported locale, then its language code, then English. Custom Material, Cupertino and Widgets delegates fall back to English when Flutter's Material supported-language set lacks the language; date and number formats have their own fallback logic (`lib/main.dart`, `lib/localization_delegates.dart`).

## Native and background text

`LocalizationContextWrapper` updates the current context, downloader notification translations and the Pigeon translation callback. When adding text consumed by native playback, inspect the translation contract rather than assuming every new Dart message is automatically exposed (`lib/util/localization_helper.dart`, `pigeons/translations_pigeon.dart`). The headless notification worker loads the saved locale, defaulting to English (`lib/background/update_notifications_worker.dart`).

See [development](development.md) for SDK and generation setup.
