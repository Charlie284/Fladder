# User guide

For platform installation instructions, see [INSTALL.md](../INSTALL.md). Features and controls vary by platform, selected player and server capabilities; the source references below identify the implemented behavior, not a guarantee that a server supplies every media format or metadata field.

## Servers, login and profiles

Select or add a Jellyfin server, then sign in with your Jellyfin credentials. The login UI includes server discovery and, when the server advertises it, Quick Connect code login. Saved users can be selected from the user grid (`lib/screens/login/screens/server_selection_screen.dart`, `lib/screens/login/widgets/discover_servers_widget.dart`, `lib/screens/login/login_screen_credentials.dart`, `lib/screens/login/login_user_grid.dart`).

Saved accounts hold their server credentials and authentication choice. Account editing and lock-screen behavior support returning to an account without treating every saved profile as an unrestricted automatic login (`lib/models/account_model.dart`, `lib/screens/login/login_edit_user.dart`, `lib/screens/login/lock_screen.dart`). Profile settings expose server-side preferences and Seerr connection settings (`lib/screens/settings/profile_settings_page.dart`).

## Browse libraries and search

Use the dashboard, libraries and favourites sections to find media. Library search provides filters, sorting, saved filters and different views; item pages provide the media-specific details and play actions (`lib/routes/auto_router.dart`, `lib/screens/library_search/library_search_screen.dart`, `lib/screens/library_search/widgets/library_saved_filters.dart`, `lib/screens/details_screens/`). Global search is separate from library filtering (`lib/screens/search/`, `lib/providers/search_provider.dart`).

## Playback

Open Player settings to choose an available backend. Outside lean-back mode, Android offers MPV, MDK and Native; other native platforms offer MPV/MDK; web offers MPV. Lean-back mode restricts the choice to Native. The default is MPV except in lean-back mode (`lib/models/settings/video_player_settings.dart`, `lib/screens/settings/player_settings_page.dart`). Backends implement different rendering and track capabilities, so switching a backend can change playback behavior (`lib/wrappers/players/`).

| Feature | How it works |
| --- | --- |
| Audio and subtitles | Player controls select streams; Player settings include subtitle customization and remembering selections (`lib/screens/video_player/video_player_controls.dart`, `lib/screens/settings/player_settings_page.dart`, `lib/screens/settings/widgets/subtitle_editor.dart`). |
| Media segment skipping | When the server supplies segment data, configure actions for intros, outros, recaps, previews and commercials. Defaults ask to skip; actions also include none, skip once and skip (`lib/models/items/media_segments_model.dart`, `lib/screens/settings/player_settings_page.dart`). |
| Trickplay previews | The timeline shows preview images when trickplay data is available; a normal progress bar does not imply thumbnails exist (`lib/screens/video_player/components/video_progress_bar.dart`). |
| Speed boost | Enable it and set a rate in Player settings. In Flutter video controls, hold Space on desktop or long-press during playback; releasing restores the prior rate (`lib/screens/video_player/video_player_controls.dart`, `lib/screens/settings/player_settings_page.dart`). |
| Double-tap seeking | When enabled, double-tap the left/right thirds to seek back/forward; the middle toggles playback (`lib/screens/video_player/video_player_controls.dart`). |
| Brightness and volume gestures | With edge gestures enabled, vertical drags on the left adjust brightness and on the right adjust volume; reverse the sides in settings. Up increases and down decreases; the top/bottom safety margins are excluded (`lib/screens/video_player/video_player_controls.dart`). |
| Picture in Picture | Supported Android/iOS devices can use PiP; actual support and auto-entry are checked through the plugin (`lib/wrappers/pip_manager.dart`, `lib/widgets/pip_lifecycle_controller.dart`). |
| Audio queues | Audio playback supports queue operations, shuffle and repeat state; media controls are integrated through the audio handler (`lib/wrappers/audio_queue_handler.dart`, `lib/models/playback/playback_queue_state.dart`, `lib/wrappers/media_control_wrapper.dart`). |

Gesture descriptions refer to Flutter controls; the Native backend launches a separate native activity (`lib/wrappers/players/native_player.dart`).

## Default keyboard shortcuts

Shortcuts are configurable. These defaults come from `lib/models/settings/video_player_settings.dart` and `lib/models/settings/client_settings_model.dart`; dispatch is handled by `lib/util/input_handler.dart` and the player controls. Availability depends on focus, player state and backend. Player bindings use left Shift/Ctrl in their default modifier definitions; macOS global bindings use Command/Super (`lib/models/settings/key_combinations.dart`).

| Player action | Default keys |
| --- | --- |
| Play/pause | Space or K |
| Seek forward / back | Right or L / Left or J |
| Immediate seek forward / back | Shift+Right or Shift+L / Shift+Left or Shift+J |
| Step forward / back | Period / Comma |
| Mute | M |
| Volume up / down | Up / Down |
| Speed up / down | Ctrl+Up / Ctrl+Down |
| Previous / next video | Shift+P / Shift+N |
| Next / previous chapter | Page Up / Page Down |
| Full screen | F |
| Skip media segment | S |
| Screenshot / clean screenshot | G / Ctrl+G |
| Toggle subtitles | T |
| Exit player | Escape |

| Global action | Windows/Linux default | macOS default |
| --- | --- | --- |
| Toggle sidebar | Q | Q |
| Search | Ctrl+K | Command+K |
| Close window | Ctrl+W | Command+W |
| Exit | Ctrl+Q | Command+Q |

Holding Space can activate speed boost rather than repeatedly toggling playback when that setting is enabled (`lib/screens/video_player/video_player_controls.dart`).

## Offline sync and downloads

Use sync actions to add media to local storage, then use the synced-media section to manage it. Downloads prepare local metadata and media files; video and music transcoding settings are separate. Download settings include storage, Wi-Fi requirements and concurrent transfers (`lib/screens/syncing/`, `lib/providers/sync_provider.dart`, `lib/screens/settings/client_sections/client_settings_download.dart`, `lib/screens/settings/widgets/transcode_settings_popup.dart`, `lib/screens/settings/widgets/transcode_music_settings_popup.dart`).

Wait for the transfer to finish before relying on offline playback. Downloader status/progress is tracked separately from sync metadata. Local position, played and favourite changes can be saved and pending user data uploaded when connectivity returns; this is not a general-purpose server backup (`lib/providers/sync/background_download_provider.dart`, `lib/providers/sync_provider.dart`, `lib/models/playback/offline_playback_model.dart`).

## Seerr discovery and requests

Configure a Seerr connection in profile settings. The connection dialog supports local or Jellyfin authentication, an API key and custom headers; discovery/search and details screens allow browsing and media requests. Request approval and deletion operations depend on the configured service and account permissions (`lib/screens/settings/widgets/seerr_connection_dialog.dart`, `lib/screens/settings/profile_settings_page.dart`, `lib/screens/seerr/`, `lib/providers/seerr_service_provider.dart`, `lib/providers/seerr/seerr_request_provider.dart`).

## Settings overview

| Area | Examples |
| --- | --- |
| Profile | Account preferences and Seerr connection (`lib/screens/settings/profile_settings_page.dart`). |
| Fladder/client | Downloads, lock timeout, dashboard, theme, display language and global shortcuts (`lib/screens/settings/client_settings_page.dart`, `lib/screens/settings/client_sections/`). |
| Player | Backend, stream limits, subtitles, segments, gestures, speed boost and audio options; platform/backend gates hide unsupported choices (`lib/screens/settings/player_settings_page.dart`). |
| About | Application information (`lib/screens/settings/about_settings_page.dart`). |

Update notifications can check enabled accounts for new Jellyfin items and Seerr requests. Desktop checks run while the application process is running; mobile scheduling uses Workmanager (`lib/providers/update_notifications_provider.dart`, `lib/background/update_notifications_worker.dart`).

## Syncplay scope

The administrative user editor exposes Jellyfin Syncplay access permissions. That is not a group-watch player UI: the checked-in app references outside generated Jellyfin APIs and translations are permission editing/labels, not a client group-session implementation. This guide therefore does not provide join/host instructions (`lib/screens/control_panel/control_user_edit/control_user_edit_general.dart`, `lib/util/jellyfin_extension.dart`).
