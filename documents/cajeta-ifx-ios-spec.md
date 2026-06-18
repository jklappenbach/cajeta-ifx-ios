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
