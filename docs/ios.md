# Horizon on iOS (plan)

Status: **phase 3, first device test pending.** CI builds an unsigned `.ipa`; the app is written to boot
to the title screen with touch controls and a controller. This page covers how the iOS port is built,
packaged and run, and will become the player-facing iOS guide once the app is confirmed on a device. The first target is an iPhone 14
(A15 Bionic, 6 GB RAM, iOS 16+), sideloaded as an unsigned `.ipa` through SideStore or AltStore.

The port follows the Android one ([android.md](android.md)): the same simulation, renderer and touch
controls, with a thin platform layer gated behind `#[cfg(target_os = "ios")]`. Desktop and Android
builds must not change behaviour.

## Scope

| In the first versions | Left out for now (gated off, not deleted) |
| --- | --- |
| Title screen, single player on the server thread, Java multiplayer (direct connect) | **On-device resource preparation.** `horizon-prepare` runs Java exporters, and iOS has no JVM and does not allow JIT. The player copies in resources prepared on a computer |
| Touch controls (`touch.rs`, `touch_ui.rs`), the on-screen keyboard (`osk.rs`) | **Code mods.** The `mods` feature (wasmtime + cranelift) generates machine code at run time, which iOS does not allow. Content mods (`.hmod` without code) still load |
| MFi / Xbox / DualSense controllers through Apple's GameController framework | The launcher UI, resource-pack downloads, LAN discovery (multicast needs an entitlement that free signing lacks) |
| Sound through cpal (CoreAudio) | App Store distribution; TestFlight |

## App shell: winit's iOS backend

**Decision: use winit's UIKit backend** (`third_party/winit/src/platform_impl/ios/`, the patched
0.30.13 the workspace already uses), not a hand-written `UIApplicationMain` shell.

Why:

- The game already *is* a winit `ApplicationHandler` (`impl ApplicationHandler for App` in `game.rs`).
  On iOS, `EventLoop::run_app` calls `UIApplicationMain`, creates the `UIWindow` + `UIViewController`
  + `UIView`, and turns UIKit callbacks into the same `WindowEvent`s the desktop and Android paths
  handle: `Touch` (id, phase, position in physical pixels, force), `Resized`, `ScaleFactorChanged`,
  `suspended`/`resumed` and `memory_warning`.
- Touch already goes through `WindowEvent::Touch` → `horizon_client::touch` on Android. Using winit on
  iOS reuses that path with no new event translation.
- `raw-window-handle` gives wgpu a `UiKitWindowHandle` (the `UIView`). wgpu's Metal backend attaches a
  `CAMetalLayer` to that view's layer, so the renderer needs no iOS-specific code beyond picking Metal.
- winit's `WindowExtIOS` already covers what an iOS game needs: `set_prefers_home_indicator_hidden`,
  `set_preferred_screen_edges_deferring_system_gestures` (so a swipe from the bottom edge does not
  leave the game), `set_prefers_status_bar_hidden`, `set_valid_orientations` and
  `set_scale_factor`.
- A custom `UIApplication` shell would mean writing and maintaining Objective-C or objc2 code for the app
  delegate, the view, touch forwarding, layer resizing and the run loop. That duplicates winit and keeps
  iOS separate from the code that Android and desktop already test.

Costs of this choice, and how we deal with them:

- `run_app` never returns on iOS, because `UIApplicationMain` does not return. Anything after
  `event_loop.run_app(...)` in `game::run` does not run. The iOS start path must not rely on it: saving
  happens on `suspended`, not on exit.
- Text input: winit's iOS backend shows the system keyboard for a focused text field
  (`set_ime_allowed`), which the game's desktop text path already calls (confirmed on the device).
- winit has no GameController support. A small objc2 module polls `GCController` once a frame (see
  [Gamepads](#gamepads)).

**Binary shape.** Android needs a `cdylib` (`crates/horizon-android`) because `NativeActivity` loads a
`.so`. iOS needs a normal Mach-O executable inside `Horizon.app`, so the iOS app is the existing
`horizon-client` binary (`src/main.rs` → `game::run`) built for `aarch64-apple-ios`. No new crate is
needed for the executable. The few objc2 calls that need `unsafe` go in a small crate,
`crates/horizon-ios-sys`, behind safe functions. `horizon-client`, `horizon-client-render` and `main.rs`
keep `#![forbid(unsafe_code)]`.

## Metal surface

- `backend_order()` in `crates/horizon-client-render/src/gfx.rs` returns `Backends::default()` when
  `os == "macos"`. iOS takes the same branch (`os == "macos" || os == "ios"`), and wgpu picks Metal,
  its only backend on iOS. The `Vulkan → DX12 → GL` fallback list does not apply.
- The surface is created as on every platform: `instance.create_surface(window.clone())`. wgpu reads the
  `UiKitWindowHandle` through `raw-window-handle` and attaches a `CAMetalLayer` to the view.
- The swapchain covers the full screen: `window.outer_size()`, which is 2532×1170 on an iPhone 14 in
  landscape at scale factor 3. Rendering the world at native resolution is expensive on a phone, so the
  existing resolution-scale setting defaults lower on iOS. The HUD and touch controls stay at full
  resolution.
- Code paths gated `#[cfg(not(target_os = "macos"))]` that reach Vulkan-only code
  (`horizon_hdr::open_vulkan_device`) and the `gles` wgpu feature become `not(any(macos, ios))`. wgpu
  has no Vulkan or GL backend on Apple targets without MoltenVK, which we do not ship.

## Lifecycle

iOS kills apps that submit GPU work in the background, and may terminate a backgrounded app without
warning. winit sends `suspended` when the app resigns active (app switcher, Control Centre, a call,
locking the phone) and `resumed` when it becomes active again.

- **`suspended` (going to background):** reuse the path Android already takes in `Game::suspended`:
  release held touches and keys, open the pause screen, and call `renderer.suspend_surface()`, which
  drops the `wgpu::Surface` and keeps the device, queue, pipelines and textures. On iOS this also
  **saves the world** (`/save-all flush` on single player's server thread), because the app may never
  come back.
- **No frames while suspended:** `about_to_wait` and redraw requests do nothing while the surface is
  gone, so nothing is encoded for the GPU.
- **`resumed` (back to foreground):** `renderer.recreate_surface()` from the same window, then resize to
  the current `outer_size()` because the device may have rotated. The existing `#[cfg(target_os =
  "android")]` block in `Game::resumed` and `Loading::resumed` widens to Android *or* iOS. The first
  `resumed` at launch creates the renderer, as on desktop.
- **`memory_warning`** (not yet handled): drop the streaming caches and far LOD meshes, and write a log
  line. If it happens again, lower the render distance by 2 for the rest of the session.

## Safe area

Stock winit 0.30 returns the **safe area** from `inner_size()` on iOS, while `Resized` and the
`CAMetalLayer` cover the whole view. The game sizes its swapchain from `inner_size()`, so the vendored
winit is patched (`third_party/winit/PATCHES.md`): on iOS `inner_size()`/`inner_position()` are the
whole view, and the safe area is a separate call, `WindowExtIOS::safe_area_insets()` (physical left,
top, right, bottom).

- The world and the sky render full screen, edge to edge.
- `hud_safe_margins` (game.rs) takes the larger of the player's **HUD margins** (Options → Controls →
  Touch Controls) and the safe area as a fraction of the screen, each frame. The HUD overlays never go
  under the sensor housing or the home indicator; the player can still add margin. On an iPhone 14 in
  landscape the safe area is about 47 pt on each side and 21 pt at the bottom.
- The touch buttons keep their own editable layout over the whole screen (**Customize layout...**),
  because iOS still delivers touches under the sensor housing.
- The app runs landscape-only (`UISupportedInterfaceOrientations` and `ValidOrientations::Landscape`).
  The home indicator and status bar are hidden, and all screen edges defer system gestures, so the
  first swipe from an edge goes to the game and only a second swipe goes to iOS (`ios_window` in
  game.rs).

## Touch → controls

The touch controls are platform-independent Rust: `crates/horizon-client-render/src/touch.rs` (the
schemes, layout, hit-testing and camera deltas, re-exported as `horizon_client::touch`) and
`crates/horizon-client/src/touch_ui.rs` (menu gestures, the `...` key drawer and the virtual mouse).

- winit gives `WindowEvent::Touch { id, phase, location, force }` with `location` in physical pixels:
  the same as Android's NativeActivity backend. `game.rs` already sends these to the touch code, behind
  86 `#[cfg(target_os = "android")]` gates.
- Phase 3 sorted the gates of the client crates into two groups:
  - **mobile** (touch controls, the touch menus of the title screen and the game, the layout editor,
    the touch options, suspend/resume, mobile defaults): now
    `#[cfg(any(target_os = "android", target_os = "ios"))]`. A plain `any(...)` rather than a
    build-script alias, because `game.rs` is compiled by two crates (`horizon-client` and
    `horizon-android`) that would both need the alias;
  - **Android-only** (JNI, the Android keyboard, documents, MediaStore, the display scale, the
    performance reports, vibration, gamepad connections): left as `target_os = "android"`, also
    where they sit inside a mobile block.
- The shared touch glue (`game/android_input.rs`, despite the name) calls a few platform functions;
  `src/ios/platform.rs` gives iOS the same names. Text input needs none of them: the iPhone's
  keyboard follows the focused text field through winit's text input, as on desktop (`game/ime.rs`),
  confirmed on the iPhone 14.
- The buttons' textures are not in the desktop-prepared folder. As on Android, the iOS app
  downloads them on first launch from Mojang's `bedrock-samples` at the pinned, hash-checked
  revision in `packaging/android/bedrock-controls.json`, into
  `horizon-client/26.3/assets/minecraft/textures/gui/bedrock`.
- Multiple touches: winit's iOS view already sets `multipleTouchEnabled` (`view.rs`), so two thumbs (move and look) work at once.
- `horizon_client::android_runtime`'s input mode (plain Rust, shared with Android) decides what shows:
  a touch selects Touch, a hardware key Keyboard/Mouse, a controller's button or stick Controller,
  which hides the touch controls as on Android.

## Gamepads

gilrs has no iOS backend (its `default` stub reports no pads), and `horizon-dualsense` talks to HID,
which iOS does not expose. The iOS build turns off the `controllers` feature and adds a GameController
reader:

- `horizon-ios-sys` polls `GCController.current()` (else the first of `controllers()`) →
  `extendedGamepad` once a frame (no callbacks
  across threads) and fills the existing `horizon_client_input::gamepad::PadState`, the same state the
  Android decoder (`android_gamepad.rs`) and gilrs fill. Mappings, deadzones, glyphs and menu navigation
  then come from the shared code.
- This covers Xbox, PlayStation (DualShock 4, DualSense), MFi and Switch Pro controllers paired over
  Bluetooth. Rumble (`GCDeviceHaptics`) can come later.

## Gyro aiming

Touch Controls → Gyroscope turns the view with the phone, as on Android. `horizon-ios-sys::motion`
reads CoreMotion's device-motion rotation rate (bias-corrected gyroscope) once a frame in pull
mode, and starts the sensor only while gyro aiming applies (the setting on, in a world, touch
input); it stops otherwise and when the app is suspended. The rate goes through Android's
`android/device_math.rs` (`gyro_delta`), with the window scene's interface orientation mapped to
Android's rotation numbers, so the sensitivity and the axes for each landscape side match.

## Microsoft account

Online-mode servers need a signed-in account. On desktop the launcher signs in and hands the game a
session file; Android's own launcher does the same with the shared code. The iOS app has no
launcher, so the game signs in itself (`game/ios_account.rs`):

- **Account** in the touch UI's top bar on menu screens opens a panel
  (`ios/account_panel.rs`, drawn like the touch buttons).
- **Sign in** runs the device-code flow of `launcher/src/account.rs` (included by path, as
  `android_main.rs` includes it): Horizon copies the code, opens the verification page with it filled
  in (`otc`) through `UIApplication openURL`, and polls until the player has allowed access.
- The account list goes to `Library/Application Support/horizon-launcher/accounts.json` (no token);
  the refresh and Minecraft tokens go to the Keychain through the `keyring` crate
  (`ios/secure_store.rs`).
- At start, and after a sign-in, a worker renews the tokens when needed (`account::fresh`), writes
  the game's one-time session file, reads it back as an `AuthSession` and makes it current
  (`auth::set_current`, replaceable on iOS as on Android). Joins read it at connect time.
- **Sign out** removes the account and its Keychain item and clears the session.
- The client ID is Android's public one (`00000000441cc96b`) unless `HORIZON_MSA_CLIENT_ID` is set
  at build time; it is not a registration of Horizon's own.

## Data directory

The app's sandbox `Documents/` folder is visible in the Files app ("On My iPhone → Horizon") and in
Finder's file-sharing tab when the iPhone is plugged into a Mac, through these `Info.plist` keys:

```xml
<key>UIFileSharingEnabled</key><true/>
<key>LSSupportsOpeningDocumentsInPlace</key><true/>
```

iOS sets `HOME` to the app's container, so `$HOME/Documents` can be found without calling Foundation.

```
Documents/
  horizon-client/26.3/   ← the player copies this in: their own .local/horizon-client/26.3,
                           prepared by the desktop launcher (horizon-prepare)
  horizon/               ← the game home (logging::set_game_home): saves/, options.txt,
                           logs/, crash-reports/, screenshots/, resourcepacks/
Library/Caches/horizon/  ← things that can be rebuilt: shader cache, downloaded server packs
```

- At start, the iOS path calls `logging::set_game_home(Documents/horizon)` and passes
  `--resources Documents/horizon-client/26.3`.
- If the resources are missing or incomplete (`missing_resources()` in `game.rs`), the game shows a
  screen saying exactly which folder to copy where, instead of the desktop's "run the launcher"
  message. It does not offer to prepare them.
- Nothing from Mojang is in the repository or the `.ipa`. The player supplies their own prepared copy.
- Logs and crash reports land in `Documents/horizon/logs` and `crash-reports`, so a tester can send
  them from the Files app.

## Content mods in the iPhone's Files app

Horizon iOS supports **content-only Horizon mods**: `.hmod` files or folders
with a `horizon-mod.toml` manifest and `assets/` / `data/` files, using the
`registries` capability. They can add blocks, items, recipes and textures,
but Java Fabric/Forge/NeoForge `.jar` files do not execute in Horizon and
mods requiring WebAssembly execution are not available in the iOS build.

1. Open **Files → On My iPhone → Horizon → horizon → horizon-mods**.
   The folder is created automatically when Horizon starts.
2. Copy a compatible content-only `.hmod` file (or content mod folder) there.
3. Open **Options → Mods** in Horizon and switch the mod **ON**.
4. Fully quit and reopen Horizon. Content registries are read at startup;
   toggling a mod does not inject new blocks into an already-running world.

Code-only mods or malformed packages appear with an explanation rather than
pretending to work. Back up worlds before enabling mods that add blocks/items;
worlds opened without their mods may lose unsupported content.

## Audio

- `cpal` (the `audio` feature of `horizon-client-render`) has a CoreAudio backend for iOS, so the
  mixer in `audio.rs` works unchanged.
- The default audio session category (`SoloAmbient`) respects the silent switch and stops other apps'
  music. Phase 3 keeps it. Setting `Ambient` (mix with the player's music) is a later option in
  `horizon-ios-sys`.
- An interruption (a phone call) arrives as `suspended`. The stream is paused there and restarted on
  `resumed`, because CoreAudio does not always resume a stream that an interruption stopped.

## Memory and performance

- An iPhone 14 has 6 GB of RAM. Jetsam usually kills a foreground app at around 2.8–3 GB. The
  `com.apple.developer.kernel.increased-memory-limit` entitlement raises that limit, but free Apple ID
  signing (SideStore/AltStore without a paid account) may remove it, so we cannot count on it.
- iOS defaults, applied only while `options.txt` or `horizon-graphics.txt` does not exist yet (the player's
  choices always win): **render distance 6**, simulation distance 6, clouds off and a 60 FPS limit
  (`Settings::first_run`), and Battery Mode on with Resolution Scale 75% (`Graphics::first_run`).
- Worker threads: the A15 has 2 performance and 4 efficiency cores. The streaming pool starts with 3
  workers so the render thread and the server thread keep a performance core each.
- `wgpu::MemoryHints::MemoryUsage` (already chosen for Android) on iOS too. `MAPPABLE_PRIMARY_BUFFERS`
  stays off on mobile, as on Android.
- The LOD system and large atlases are the main GPU memory costs. The Android atlas reduction
  (`android_runtime::configure_atlas_limit`) applies only when a texture side goes over
  `max_texture_dimension_2d` (16384 on A15), so it does not trigger on iOS.

## wgpu features on A15 Metal

`open_backend` in `gfx.rs` asks for `adapter.features() ∩ wanted`. **No feature is strictly required**:
the renderer checks `device.features()` and falls back when one is missing. `required_limits` is the
adapter's own limits, so device creation cannot fail on limits. What the A15 (Apple GPU family 8,
Metal 3) gives through wgpu-hal's Metal backend (`third_party/wgpu-hal/src/metal/adapter.rs`):

| Feature the renderer asks for | Used for | A15 / iOS 16+ |
| --- | --- | --- |
| `INDIRECT_FIRST_INSTANCE` | GPU-driven terrain draws | Yes |
| `MAPPABLE_PRIMARY_BUFFERS` | Section pages in system memory | Available, but not requested on mobile (as on Android) |
| `TEXTURE_FORMAT_16BIT_NORM` | Shader-pack buffers | Yes |
| `TEXTURE_ADAPTER_SPECIFIC_FORMAT_FEATURES` | Shader-pack render targets | Yes |
| `RG11B10UFLOAT_RENDERABLE` | Float targets for shader packs | Yes (Apple GPUs render to it) |
| `SUBGROUP` | Shader-pack compute programs | Yes (SIMD-scoped operations, Apple7+) |
| `VERTEX_WRITABLE_STORAGE` | Storage buffers in vertex shaders | Yes |
| `CLEAR_TEXTURE` | Clearing shader-pack images | Yes |
| `TEXTURE_ATOMIC` | Shader-pack image atomics | **Only on iOS 17+** (needs MSL 3.1), so present on the iOS 27 test phone. Missing on iOS 16: shader packs that use image atomics are turned off |
| `FLOAT32_BLENDABLE` (`oit::features`) | Order-independent translucency | wgpu reports it; **check on the device**. If it is missing or broken, OIT turns itself off and sorted translucency is used |
| `TIMESTAMP_QUERY`, `…_INSIDE_ENCODERS` (`GpuTimer`) | F3 GPU timings | Probably only at stage boundaries. When missing, the GPU timer turns itself off |
| `SHADER_EARLY_DEPTH_TEST` (`cull::features`) | Occlusion culling on Vulkan | Not on Metal. `cull::supported` already has a Metal path |

Limits to watch on the device: `max_texture_array_layers` is 2048 (the terrain array needs about
1300 layers, so it fits), `max_texture_dimension_2d` is 16384, and `max_buffer_size` depends on the
device's RAM. Shaders go WGSL → naga → MSL → the system Metal compiler. That compiler is part of iOS,
so it is allowed even though the app itself cannot JIT. Pipeline creation at the first launch may be
slow, and it is logged.

## Build and packaging

**CI (Phase 2): `.github/workflows/ios.yml`** on `macos-latest`:

```sh
rustup target add aarch64-apple-ios
IPHONEOS_DEPLOYMENT_TARGET=16.0 cargo build --release --target aarch64-apple-ios \
  -p horizon-client --bin horizon-client --no-default-features --features audio
```

- `--no-default-features --features audio`: no `controllers` (gilrs/hidapi) and no `mods`.
- Dependencies that do not build for iOS are gated in `crates/horizon-client/Cargo.toml`: `arboard`
  becomes `not(any(android, ios))` (the clipboard goes through `UIPasteboard` in `horizon-ios-sys`
  later), and wgpu's `gles` becomes `not(any(macos, ios))`.
- Packaging, a plain shell script (`tools/ios_package.sh`), with no Xcode project:
  `Payload/Horizon.app/` = the binary (`CFBundleExecutable` = `Horizon`), `Info.plist`
  (`packaging/ios/Info.plist`) and icon PNGs resized from `packaging/icons/hicolor/512x512/apps/horizon.png`
  with `sips` → `zip -r Horizon.ipa Payload`. **Unsigned**: SideStore/AltStore sign it with the
  player's Apple ID when installing.
- `Info.plist` essentials: `CFBundleIdentifier` `dev.horizonlab.horizon.ios`, `MinimumOSVersion` 16.0,
  `UIDeviceFamily` [1] (iPhone), `UIRequiredDeviceCapabilities` [arm64, metal],
  `UISupportedInterfaceOrientations` landscape, `UIRequiresFullScreen`, `UIStatusBarHidden`, an
  empty `UILaunchScreen` dictionary (without it iOS runs the app letterboxed at a legacy screen size),
  the two file-sharing keys, and `NSLocalNetworkUsageDescription` for LAN/direct connect to local
  servers.
- The workflow uploads `Horizon.ipa` as an artifact on every run. Every successful run that is not
  a pull request's is also published as an `ios-build-<run number>` prerelease whose notes list the
  commits since the previous one (`tools/ios_release.sh`); pushes to `ios` also refresh the rolling
  `ios-latest` prerelease. Running `ios.yml` by hand with `backfill` set to earlier run IDs publishes those runs the same way.
- Locally, on Linux: `tools/ios_check.sh` runs `cargo check --target aarch64-apple-ios` with the
  same features. There is no iOS SDK on Linux, so it uses a stand-in C compiler that writes empty
  objects for the C build scripts (ring, mimalloc). Those objects are never linked. It catches Rust
  errors and cfg mistakes. Anything that links against the SDK is only checked for real on CI.

## Phases

1. **Plan:** this page. Done.
2. **CI first:** `ios.yml`, the packaging script and `Info.plist`, then cfg gates until the build
   passes and an `.ipa` is uploaded. Done.
3. **Shell:** the start path (Documents paths, defaults), Metal on iOS in `backend_order`, the touch
   path, lifecycle, safe area, the GameController reader and the missing-resources screen. Done when
   the app boots to the title screen on the iPhone 14, with touch controls and a gamepad. Written;
   waiting for the first device test.

Later: the clipboard (`UIPasteboard`), `memory_warning`, `Ambient` audio,
rumble, opening links, the increased-memory entitlement where signing allows it, portrait mode,
iPad.

## Installing a test build

The steps players follow are in the README ("Install on an iPhone"). In short:

1. Download `Horizon.ipa` from the newest `ios-build-N` release.
2. Install it with [Impactor](https://github.com/claration/impactor) from a computer (it signs with
   your Apple ID and installs over USB), or with SideStore or AltStore on the phone.
3. Copy the desktop launcher's `.local/horizon-client/26.3` into
   **Files → On My iPhone → Horizon → horizon-client/26.3**.
4. Free Apple ID signing expires after 7 days: re-sign with the same tool. Saves in `Documents/`
   are kept.
