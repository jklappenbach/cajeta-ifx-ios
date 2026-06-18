# cajeta-ifx-ios — Spec

**Status:** Draft. Requirements for the **iOS** backend of `ifx` (window / input / audio).
This is a requirements document (the *what* and *why*); the task breakdown is the plan.

## 1. Overview & why
`dev.cajeta.ifx.ios` is the iOS realization of the portable `ifx` contract
(`cajeta.ifx`). It FFI-binds iOS system APIs and registers `WindowBackend` / `AudioBackend` /
`InputBackend` implementations through `BackendRegistry`. The app codes once against `ifx`; this
library is selected at runtime on iOS (registry + probe + priority, with `CAJETA_IFX_*`
overrides). Umbrella design: the cajeta repo's `documents/cajeta-gfx/cajeta-gfx-spec.md` §9.

## 2. Capabilities (what it binds)
- **Window + Surface:** UIKit (UIView + CAMetalLayer); produces the opaque `Surface` the gfx swapchain pairs with
  `VK_EXT_metal_surface (MoltenVK)`.
- **Input:** UIKit touch + GameController — keyboard/mouse arrive via the window event queue; gamepads via the input
  API.
- **Audio:** CoreAudio / AVAudioEngine — output and capture.

> **Notes.** No desktop windows: a single fullscreen surface; app lifecycle via UIApplication. Present via MoltenVK/Metal.

## 3. Goals
- Implement the full `ifx` SPI for iOS behind the one portable API surface — no iOS concept
  leaks to callers.
- Register + `probe()`/`priority()` so the dispatcher binds it on iOS; honor `CAJETA_IFX_*`.
- Hand a correct opaque `Surface` to the gfx swapchain (`VK_EXT_metal_surface (MoltenVK)`).
- Track iOS API versioning **here**, isolated from the other backends.

## 4. Non-goals
- No portable-contract changes (those live in stdlib `ifx`).
- No graphics/render code — gfx owns the swapchain; this only supplies the `Surface`.
- No UI toolkit, input action-mapping, or audio mixing/DSP (engine / Glorias concerns).
- No codecs — capture/recording is the `cajeta-ifx-harness` backend + optional `VideoSink` providers.

## 5. References
- Umbrella design: `cajeta-gfx-spec.md` §9 (platform layer — §9.1 codegen-vs-HAL, §9.3 packaging,
  §9.4 binding model).
- Contract: `cajeta.ifx` — `Backend` / `WindowBackend` / `AudioBackend` / `InputBackend`.
- Selector: the `cajeta-ifx-backend` melt (target → backend).

---

## Appendix A — Vendor SDK details (2025-2026 research)

### Binding: Objective-C runtime everywhere (only Core Audio is C)
**UIKit, GameController, AVFoundation, MetalKit, QuartzCore are Obj-C/Swift only.** Only **Core
Audio + AudioToolbox** expose C. → This backend ships an **Obj-C/Obj-C++ (`.m`) shim** (`extern "C"`)
for the UIKit window/scene/view + `CAMetalLayer` bring-up, touch + GameController, and the
`AVAudioSession` layer; binds Core Audio / `RemoteIO` directly via C for the low-latency mixer.
(Direct `objc_msgSend` is possible but brittle: arm64 has no variadic `objc_msgSend` — every call
site must be cast to its concrete prototype.)

### SDK reference
| Domain | Framework | C-ABI? | Notes | Deprecated |
|---|---|---|---|---|
| Window/surface | **UIKit** `UIWindow`/`UIView`/`UIViewController` (single fullscreen) + `CAMetalLayer` (override `layerClass`) or **MTKView**; **`UIScene`/`UISceneDelegate` MANDATORY** | Obj-C (shim) | `CADisplayLink` pacing (120 Hz ProMotion); `drawableSize` = `UIScreen.nativeScale` | OpenGL ES (iOS 12) |
| Surface (Vulkan) | **MoltenVK** `VK_EXT_metal_surface` (`CAMetalLayer*`) | C interop | Vulkan 1.4 subset, `VK_KHR_portability_subset` | `VK_MVK_ios_surface` |
| Input | **UIKit touch** (`UITouch`/`UIGestureRecognizer`) primary; **GameController** optional (must stay playable touch-only); **`GCVirtualController`** on-screen pad (iOS 15+) | Obj-C (shim) | force/Pencil via `UITouch`; gamepad optional-by-policy | — |
| Audio | **`AVAudioSession`** (mandatory: category/interruptions/route changes) + **Core Audio / RemoteIO** (C, low-latency, output+capture) | session=Obj-C, IO=**C** | mic needs `NSMicrophoneUsageDescription` + grant | OpenAL |

### Capability support & gaps (vs spec §9.7) — diverges from macOS
- **No real windows** → the contract collapses "window" to one fullscreen surface + safe-area
  insets + orientation; **multi-window/positioning/warp = n/a**.
- **Touch is the floor; keyboard/mouse optional/absent**; physical gamepad optional → **`supports()`**
  drives this; provide `GCVirtualController` or engine-drawn on-screen controls.
- **OS-owned, interruptible audio (`AVAudioSession`)** → must handle interruption (calls/Siri — and
  the "interruption ended" notification is **not guaranteed**, resume defensively), route changes,
  and `AVAudioEngineConfigurationChange` (rebuild graph). These map to `ifx`'s portable audio-route
  + lifecycle events.
- **Tight lifecycle/sandbox** → release drawable, pause `CADisplayLink`, stop audio on background;
  recreate on foreground → `ifx` surface-lost/recreated events. **UIScene adoption mandatory** with
  the iOS 26/27 SDK (verify exact enforced SDK at implementation time).
- **Floor: iOS 13+ (15+ recommended).** No native Vulkan (MoltenVK only).

### References
- UIKit/UIScene: https://developer.apple.com/documentation/uikit/uiscene · CAMetalLayer: https://developer.apple.com/documentation/quartzcore/cametallayer
- GameController/GCVirtualController: https://developer.apple.com/documentation/gamecontroller/gcvirtualcontroller
- AVAudioSession: https://developer.apple.com/documentation/avfaudio/avaudiosession · Core Audio: https://developer.apple.com/documentation/coreaudio
- MoltenVK `VK_EXT_metal_surface`: https://github.com/KhronosGroup/MoltenVK
