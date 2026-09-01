<p align="center">
  <img src="docs/branding/watchmaia.svg#gh-dark-mode-only" alt="watchmaia" width="440">
  <img src="docs/branding/watchmaia-light.svg#gh-light-mode-only" alt="watchmaia" width="440">
</p>

<p align="center"><strong>media-kit</strong></p>

# Watchmaia media-kit

This is the [Watchmaia](https://github.com/franciscomfcmaia/watchmaia-app) fork of [media-kit](https://github.com/media-kit/media-kit) (via [jellyflix-app/media-kit](https://github.com/jellyflix-app/media-kit)).

The app owns UI, intros, and Jellyfin. **This repo owns mpv, the Flutter `Video` widget, native textures, and iOS Picture-in-Picture.** If you are changing what the user sees around the player, you are in the wrong tree.

Upstream API docs, examples, and platform notes stay at [media-kit/media-kit](https://github.com/media-kit/media-kit). Do not copy them here.

## What Watchmaia added

| Change | Where | Why |
| --- | --- | --- |
| iOS sample-buffer PiP (iOS 15+) | `media_kit_video/ios/Classes/plugin/common/PiPController.swift` | mpv has no `AVPlayer`. Frames are wrapped as `CMSampleBuffer` and fed to `AVSampleBufferDisplayLayer` |
| Frame tap | `TextureHW.swift`, `TextureSW.swift` (`MediaKitPiP.enqueue`) | Same `CVPixelBuffer` the Flutter texture already rendered |
| `Video.paintVideoFrame` | `media_kit_video/lib/src/video/video_texture.dart` | Keep the texture mounted but hide it if a native surface must be the only picture (Watchmaia leaves this `true`; Flutter is the on-screen player) |
| Controls theme notify fix | `material.dart`, `material_desktop.dart` | Upstream `updateShouldNotify` was inverted, so a PiP button added after first build never appeared |
| Windows libmpv pin | `libs/windows/media_kit_libs_windows_video/windows/CMakeLists.txt` | Nightly **Update Windows DLL** workflow |

## Repository map

```text
media-kit/
  media_kit/                         # Dart Player, tracks, FFI
  media_kit_video/                   # Video widget + native plugins
    lib/src/video/video_texture.dart
    ios/Classes/plugin/common/PiPController.swift
    ios/Classes/plugin/TextureHW.swift
  libs/ios/media_kit_libs_ios_video/
  libs/android/media_kit_libs_android_video/
  libs/macos/media_kit_libs_macos_video/
  libs/windows/media_kit_libs_windows_video/
  libs/universal/media_kit_libs_video/
  .github/workflows/
    update_windows_dll.yml           # nightly libmpv hash
    ci.yml                           # package tests (manual dispatch)
```

Expected sibling checkout (the app path override depends on this):

```text
watchmaia/
  watchmaia-app/
  watchmaia/media-kit/               # this repo
```

## Consume from Watchmaia

Until `pip` is the published default, the app pins most packages to the Jellyflix commit and **overrides `media_kit_video` locally**:

```yaml
# watchmaia-app/pubspec.yaml
dependency_overrides:
  media_kit_video:
    path: ../watchmaia/media-kit/media_kit_video
```

After this fork is the source of truth:

```yaml
dependency_overrides:
  media_kit_video:
    git:
      url: https://github.com/watchmaia/media-kit.git
      ref: pip
      path: ./media_kit_video
```

Swift or plugin changes require a **full iOS rebuild** of the app (`flutter run` from a stopped process). Hot reload will not pick them up.

## iOS Picture-in-Picture

mpv renders into a `CVPixelBuffer` swapchain. Flutter shows that as a `Texture`. AVKit PiP cannot use a Flutter texture, so the same buffers are also enqueued on an `AVSampleBufferDisplayLayer`.

```text
mpv render  →  CVPixelBuffer
                 ├─ Flutter Texture          (what the user sees)
                 └─ MediaKitPiP.enqueue
                        AVSampleBufferDisplayLayer
                        AVPictureInPictureController
```

**Layering rule:** `PiPHostView` is inserted **behind** `FlutterView` on the key window, full alpha, not hidden. AVKit needs a real in-window layer for `canStartPictureInPictureAutomaticallyFromInline`. Putting it on top of Flutter stacks a second picture (broken intro, duplicate frames, chrome insets).

The app wires the controller in `ios/Runner/AppDelegate.swift` on channel `com.maiaac.watchmaiaapp/pip`:

| Method | Native |
| --- | --- |
| `isSupported` | `AVPictureInPictureController.isPictureInPictureSupported()` |
| `enable` | `MediaKitPiPController.enable(mpvHandle:)` — audio session, host view, auto-PiP flag |
| `disable` | Tear down host + observers |
| `start` / `stop` | Manual PiP |
| `pipEvent` (native → Dart) | `willStart`, `didStart`, `didStop`, `failed`, `frame size changed`, … |

`PipChannel.listen()` in the app prints those events. `NSLog` from this plugin does not appear in `flutter run`.

App-side requirements (not this repo):

- `UIBackgroundModes` includes `audio`
- `Video(pauseUponEnteringBackgroundMode: false)` on iOS so mpv keeps playing when the app resigns active
- `PipChannel.enable(await player.handle)` when the player screen opens

Test on a **physical iPhone**. Simulator PiP is not the same as system PiP.

### Do not

- Hide the host (`isHidden`, alpha ≈ 0) and expect auto-PiP — AVKit will refuse
- Call `startPictureInPicture()` from `willResignActive` — rejected even when `isPictureInPicturePossible` is true
- Size the host with guessed control-bar insets — Flutter already paints chrome

## Windows DLL hash

Workflow: [Update Windows DLL](https://github.com/watchmaia/media-kit/actions/workflows/update_windows_dll.yml)

On a nightly cron and on **Run workflow** it:

1. Resolves the latest `mpv-dev-x86_64-2*.7z` from [shinchiro/mpv-winbuild-cmake](https://github.com/shinchiro/mpv-winbuild-cmake)
2. Writes filename, URL, and MD5 into `libs/windows/media_kit_libs_windows_video/windows/CMakeLists.txt`
3. Commits `Update DLL hash` if the pin changed

`github-actions[bot]` / `GITHUB_TOKEN` **cannot push** to this repo. The workflow checks out with secret **`GH_PAT`**.

1. Classic PAT, **`repo`** scope, a user who can push to `watchmaia/media-kit`
2. Repo → Settings → Secrets and variables → Actions → `GH_PAT`
3. Settings → Actions → allow workflows
4. Run **Update Windows DLL** on `main`

Editing `.github/workflows/*.yml` and pushing it requires the PAT **`workflow`** scope (`refusing to allow a Personal Access Token to create or update workflow` otherwise). The GitHub web editor uses your session and does not need that scope.

## Branches

| Branch | Role |
| --- | --- |
| `main` | Default. DLL bot commits land here |
| `pip` | Watchmaia iOS PiP + controls notify fix. What the app should track |

Keep `pip` based on the same Jellyflix pin the app still uses for `media_kit` / `media_kit_libs_*` (`87c447e8` at the time of writing) unless you bump those overrides together.

## Day-to-day

```bash
git clone https://github.com/watchmaia/media-kit.git
cd media-kit
git checkout pip
```

There is no app binary here. Iterate from `watchmaia-app`:

```bash
cd ../watchmaia-app   # or your sibling path
fvm flutter run -d <iphone>
```

Package tests (optional; CI is `workflow_dispatch` only):

```bash
cd media_kit && dart pub get && dart test
```

## License

MIT. See [LICENSE](LICENSE). Copyright remains with the media_kit authors. Watchmaia changes are additional work on that license.
