# cajeta-ifx-ios — Plan

Derived from `cajeta-ifx-ios-spec.md`. Outline numbering `1` / `a` / `1`.
Checkbox legend: `[x]` done · `[~]` partial · `[ ]` not started.

## 1. Window + Surface backend
iOS window creation via UIKit (UIView + CAMetalLayer), producing the opaque `Surface` for `VK_EXT_metal_surface (MoltenVK)`.

   **TDD**
   a. [ ] `probe()` returns true under iOS, false elsewhere (env-forced).
   b. [ ] `createWindow` + `surfaceOf` yield a non-headless `Surface` with the requested extent.

   **Deliverables**
   a. [ ] `UIKitWindowBackend implements WindowBackend` + self-registration.
   b. [ ] FFI bindings for UIKit (UIView + CAMetalLayer) (create / destroy / poll) + the surface handle.

   **Acceptance Criteria**
   a. [ ] A window opens on iOS and the gfx swapchain presents to its `Surface`.

## 2. Input
   **TDD**
   a. [ ] Synthetic UIKit touch + GameController events surface as `WindowEvent` / `InputDevice` state.

   **Deliverables**
   a. [ ] keyboard/mouse via the window queue; gamepads via UIKit touch + GameController.

   **Acceptance Criteria**
   a. [ ] Input round-trips on a real iOS device.

## 3. Audio
   **TDD**
   a. [ ] An output stream plays a known buffer; capture reads a known input.

   **Deliverables**
   a. [ ] `AudioBackend` over CoreAudio / AVAudioEngine (output + capture).

   **Acceptance Criteria**
   a. [ ] Audio output is audible / captured PCM matches a reference within tolerance.

## 4. Registration & dispatch
   **TDD**
   a. [ ] Registers at load; the dispatcher selects it on iOS; `CAJETA_IFX_*` override works.

   **Deliverables**
   a. [ ] static-init registration into `BackendRegistry`; `priority()` == 100.

   **Acceptance Criteria**
   a. [ ] On iOS with this library linked, `ifx` binds it with no app changes.

## Dependencies & sequencing
- Gated on the stdlib `cajeta.ifx` contract landing (`runtime/src/cajeta/ifx/`).
- The Surface/WSI hand-off is gated on the gfx swapchain (`cajeta.gpu.gfx`, spec Part GP-1 §4.2).
- Versioned independently of the other backends (own iOS API cadence).

---

## Appendix A — Binding & capability-gap work (from vendor research)

### 5. Binding (Objective-C shim + C audio)
   **TDD**
   a. [ ] The shim creates the `UIScene`/`UIWindow`/`UIView` + `CAMetalLayer` fullscreen surface on
      the main thread; touch + GameController events marshal to C.

   **Deliverables**
   a. [ ] Obj-C (`.m`) shim: **UIScene/UISceneDelegate** lifecycle, UIKit view + `CAMetalLayer`,
      `UITouch`/gestures, GameController + `GCVirtualController`, and the `AVAudioSession` layer.
   b. [ ] Direct C FFI for Core Audio / `RemoteIO` (low-latency output + capture).
   c. [ ] MoltenVK `VK_EXT_metal_surface` from the `CAMetalLayer`.

   **Acceptance Criteria**
   a. [ ] Renders fullscreen via the shim; UIScene adopted (won't-launch deadline met); only C
      crosses into Cajeta.

### 6. Capability gaps & fallbacks (vs spec §9.7)
   **TDD**
   a. [ ] Audio interruption (begin/end-not-guaranteed) and route-change → defensive resume + graph
      rebuild; surface-lost/recreated on background/foreground.

   **Deliverables**
   a. [ ] collapse "window" → single fullscreen surface + safe-area/orientation; touch floor +
      optional gamepad via `supports()`; `GCVirtualController` / engine-drawn controls.
   b. [ ] `AVAudioSession` category/interruption/route handling mapped to `ifx` audio-route +
      lifecycle events; mic permission via the contract's permission state.

   **Acceptance Criteria**
   a. [ ] Playable touch-only; audio survives a phone-call interruption + headphone unplug; app
      resumes cleanly from background (drawable + audio recreated).
