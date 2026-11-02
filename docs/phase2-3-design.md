# ToxiBrowser Phase 2/3 design: RAM and weak-GPU tuning

Base: ungoogled-chromium-windows, Chromium 156.0.8078.79. Every file path below is at that tag
(fetched from chromium.googlesource.com). Line numbers are for that tag and are approximate anchors.
Target: AMD FX-8320, 7.5 GB RAM, ATI Radeon 3000 (blocklisted). Stock baseline: idle 7 procs / 354 MB;
5 tabs 14 procs / 1001 MB.

RAM numbers are ESTIMATES derived from typical per-process sizes (354 MB / 7 = about 50 MB average).
Nothing here was built or measured. Section 0 gives the measurement procedure that must gate Phase 3.

## 0. Ground truth to establish first (no build needed)

Hypothesis for the 7 idle processes: browser, GPU, network service, storage service, NTP renderer,
preloaded Top Chrome WebUI renderer, spare renderer. Audio service starts lazily. Verify on the stock build:

    Get-CimInstance Win32_Process -Filter "Name='chrome.exe'" | Select ProcessId,WorkingSetSize,CommandLine

Read `--type=` and `--utility-sub-type=` from each command line, and open chrome://gpu to see which GPU mode
is active (see D). Re-run the same two measurements (idle NTP, 5 fixed tabs) after each patch group.

## 1. How the repo applies flags (constraints for every item)

- `build.py:284-303` concatenates `ungoogled-chromium/flags.gn` then `flags.windows.gn` into `out/Default/args.gn`.
  Later assignment wins (`chrome_pgo_phase` is already set twice), so ToxiBrowser overrides go in `flags.windows.gn`.
- `build.py:316` runs `gn gen ... --fail-on-unused-args`. Any arg that is not in a `declare_args()` block breaks
  the build. This rules out `enable_spellcheck` and `enable_dice_support` (see A).
- Toxi patches go under `patches/toxibrowser/` and are appended to `patches/series` after all windows patches.
- Ungoogled does not touch these files, so the one-line patches below should not conflict:
  `content/public/common/content_features.cc` is touched only by `disable-fedcm-by-default.patch`; the other
  target files (`ui_features.cc`, `gl_features.cc`, `gpu_mode_manager.cc`, `user_tuning/prefs.cc`,
  `memory_saver_mode_policy.cc`, `back_forward_cache_impl.cc`, `spellcheck_factory.cc`, `media_router_feature.cc`,
  `chrome_feature_list_creator.cc`) are untouched.
  `chrome_content_browser_client.cc`, `chrome/browser/BUILD.gn` and `about_flags.cc` are touched by several
  ungoogled patches, so keep hunks small and anchored away from their hunks.

## E. Mechanism recommendation (decided first because everything else uses it)

One rule: change the default at the declaration site, not with injected command lines.

1. Anything with a pref: change the `Register*Pref` default. The existing Settings UI shows the correct state and
   is the user toggle (Memory Saver, graphics acceleration, spellcheck).
2. Anything with a `BASE_FEATURE`: flip to `FEATURE_DISABLED_BY_DEFAULT` (or change a `FeatureParam` default).
   Child processes inherit the browser's feature list automatically, no merge logic, and chrome://flags or
   `--enable-features=` still overrides. For user visibility add a `FEATURE_VALUE_TYPE` entry (see below).
3. Only for knobs that exist solely as command-line switches (smooth scrolling, forced reduced motion,
   renderer process limit): one tiny hook, `toxi::ApplyDefaultSwitches(base::CommandLine*)`, called from
   `ChromeFeatureListCreator::CreateFeatureList()` in `chrome/browser/metrics/chrome_feature_list_creator.cc`
   (lines 63-69) between `ConvertFlagsToSwitches()` and `SetUpFieldTrials()`. At that point flags from
   chrome://flags are already switches, so the hook adds a default only when the user has not set the switch or
   its flag. It runs once in the browser process before the GPU data manager and before any child launch.
   `ChromeMainDelegate::PostEarlyInitialization` (`chrome/app/chrome_main_delegate.cc:864`) calls this function
   (or runs it earlier if `IsInitFeatureListEarly()`), so both paths are covered.

Rejected: injecting in `BasicStartupComplete` (runs before flags are converted, cannot honor user flags, and runs
in every child process), and a `--enable-features/--disable-features` merge layer (fragile string merging).

User-visible flags: add ToxiBrowser entries through ONE new header pair, mirroring ungoogled's own pattern
(`add-flags-for-existing-switches.patch`): new files `chrome/browser/toxi_flag_choices.h` and
`chrome/browser/toxi_flag_entries.h`, plus two `#include` lines in `about_flags.cc` placed after the
`existing_switch_flag_entries.h` include (about_flags.cc around line 5207). Because the entries live in our own
header, later ungoogled flag patches cannot conflict with them. Flags need a relaunch (standard chrome://flags button).

## A. GN args (edit `flags.windows.gn`)

Append (later wins over `ungoogled-chromium/flags.gn`):

    enable_widevine=false
    enable_background_mode=false
    enable_pdf=false
    enable_openxr=false

| Item | Declared at (tag) | Ungoogled does it? | Decision |
|---|---|---|---|
| enable_widevine | `third_party/widevine/cdm/widevine.gni` first `declare_args()`; upstream default false for non-branded builds | No: `flags.gn` sets it TRUE | Set false. This is Chromium's own default config, so build-break risk is very low. Removes CDM component registration. RAM at idle about 0, binary and page-in a few MB. No user toggle (DRM sites simply do not work). `enable_library_cdms` (`media/media_options.gni:254`, `toolkit_views`) stays; do not touch it. |
| enable_pdf | `pdf/features.gni` (`declare_args`, default `!android && ...`) | No | Set false. Derived args follow automatically: `enable_print_preview = enable_printing && enable_pdf` (`printing/buildflags/buildflags.gni`), `enable_pdf_ink2` and `enable_pdf_save_to_drive` (second `declare_args` in `pdf/features.gni`). Print preview disappears; Windows system print dialog remains (`enable_basic_print_dialog`, `enable_oop_basic_print_dialog` are win-true). Build code is guarded: `chrome/browser/BUILD.gn` (`if (enable_pdf)` near 1895, 3308, 4212), `chrome/browser/ui/BUILD.gn` (4623), `chrome/common/BUILD.gn` (805). Risk MEDIUM-LOW: I could not verify Google ships a desktop config with it off, and ungoogled's `disable-ai.patch` edits pdf files (they would simply be unbuilt). RAM about 0 at idle (the PDF extension/plugin only runs when opening a PDF); saves binary size. User story: PDFs download and open in the system viewer. Fallback if the build breaks: leave `enable_pdf` true and default the pref `plugins.always_open_pdf_externally` (`prefs::kPluginsAlwaysOpenPdfExternally`, read in `chrome/browser/plugins/plugin_prefs.cc:58`) to true; I did not locate its `Register*Pref` line, grep for it when implementing. |
| enable_spellcheck | NOT an arg: plain `enable_spellcheck = use_blink` in `components/spellcheck/spellcheck_build_features.gni` | n/a | Cannot be set (would fail `--fail-on-unused-args`). Use the pref default instead (C7). |
| enable_printing | `printing/buildflags/buildflags.gni` `declare_args` | n/a | KEEP true. Gated in 8+ places across `chrome/browser`, `chrome/common`, `chrome/BUILD.gn`; not an exercised desktop-off config; RAM benefit nil (print backend is on-demand). |
| enable_hangout_services_extension | `chrome/common/features.gni` (default `enable_src_internal`, false) | Yes, `flags.gn` | Already done. |
| enable_media_remoting | `media/media_options.gni:365` (+ derived rpc/redirection args) | No | Do NOT set in Phase 2. About 0 RAM; touches media/Cast conditionals; also leaves the `[multimedia-redirection]` download in `downloads.ini` pointless. Kill Cast with the `kMediaRouter` feature instead (C6). |
| enable_extensions | `extensions/buildflags/buildflags.gni` `declare_args` | No | Confirmed NOT viable on desktop: that file's header says `enable_extensions` is the legacy desktop gate, `enable_extensions_core` gates most code, and ungoogled patches (`extensions-manifestv2`, `add-flag-to-hide-extensions-menu`, `0005-disable-default-extensions`) edit extension code. Keep compiled. `enable_platform_apps`, `enable_hosted_apps` derive from it. |
| enable_supervised_users | `components/supervised_user/buildflags.gni` | No | IMPOSSIBLE: `chrome/browser/BUILD.gn:47` has `assert(enable_supervised_users)`. Do not touch. |
| enable_background_mode | `chrome/common/features.gni` (default `!android && !chromeos`) | Partly: `0006-modify-default-prefs.patch` defaults pref `kBackgroundModeEnabled` false | Set false. Guarded in `chrome/browser/BUILD.gn` (1421, 3053) and `chrome/common/BUILD.gn:60`. Risk low. Removes background-mode manager and startup keep-alive code. Keep `enable_background_contents`. |
| enable_openxr / enable_vr | `device/vr/buildflags/buildflags.gni`; `enable_vr = enable_openxr || cardboard || arcore || linux`, so false on Windows once openxr is off | No | Set `enable_openxr=false`. Risk low-medium (WebXR disappears, which suits a privacy build). If `third_party/openxr` was not checked out it is already effectively false; if `gn gen` complains, drop it. |
| enable_dice_support | plain assignment in `components/signin/features.gni` | n/a | Not an arg. Skip. |
| symbol levels | already `symbol_level=0`, `blink_symbol_level=0`, `v8_symbol_level=0` in `flags.windows.gn` | Yes | Nothing to do. |
| Optimization | `is_official_build=true`; `chrome_pgo_phase=2` in `flags.windows.gn` overrides ungoogled's 0. PGO profiles are fetched by `ungoogled-chromium/utils/clone.py:174-178` (`--pgo`, default win64); `build.py:292-293` forces phase 0 for `--tarball` builds. LLVM 23.x is required for the profdata version (`downloads.ini` note). | Yes | Keep. V8 pointer compression is already on for x64 (`v8/gni/v8.gni:348-352`, V8 revision from `DEPS`). Do NOT enable lite mode or jitless. |

Other candidate args found not worth the risk: `enable_printing`, `enable_library_cdms`, `enable_extensions`,
`enable_supervised_users`, `enable_media_remoting`. Net build-size gain from A is small; A is mostly about
removing code paths, not RAM.

## B. Process model

### B1. Spare renderer and preloaded WebUI renderer (largest idle win)

- Spare renderer: `content/public/common/content_features.cc:1191`
  `BASE_FEATURE(kSpareRendererForSitePerProcess, base::FEATURE_ENABLED_BY_DEFAULT)` -> `FEATURE_DISABLED_BY_DEFAULT`.
  Consumed in `content/browser/renderer_host/render_process_host_impl.cc` `IsSpareProcessKeptAtAllTimes()`
  (about line 3579). That function also already returns false when site isolation is off, or when total RAM
  <= 1077 MB (`kAndroidSpareRendererMemoryThreshold`, `content_features.cc:136-138`).
  Ungoogled: not done. Cost: first navigation to a new site spawns a renderer on demand (tens of ms on FX-8320).
- Preloaded Top Chrome WebUI: `chrome/browser/ui/ui_features.cc:205`
  `BASE_FEATURE(kPreloadTopChromeWebUI, FEATURE_ENABLED_BY_DEFAULT)` -> DISABLED. Gate:
  `chrome/browser/ui/webui/top_chrome/webui_contents_preload_manager.cc:95`. This preloads a hidden WebContents
  (so a renderer) for tab search / side panel at window creation. Ungoogled: not done.
  Cost: first open of tab search or side panel is slower.
- Expected: one or two fewer processes at idle, about 40-100 MB. Risk: build none, behavior low.
- Toggle: add `FEATURE_VALUE_TYPE(features::kSpareRendererForSitePerProcess)` and
  `FEATURE_VALUE_TYPE(features::kPreloadTopChromeWebUI)` entries to `toxi_flag_entries.h`
  (`content_features.h` and `ui_features.h` are already included by `about_flags.cc`).

### B2. Renderer process limit

- Switch `renderer-process-limit` (`content/public/common/content_switches.cc:629`), applied in
  `content/browser/browser_main_loop.cc:607-613` via `RenderProcessHost::SetMaxRendererProcessCount`.
  Stock limit for this PC: RAM/2/85 MB = about 45 (`render_process_host_impl.cc:1560-1575`).
- HONEST CAVEAT: under strict site isolation (default on desktop) the limit is only a soft cap. When it is reached
  the code tries to reuse an existing process (`render_process_host_impl.cc` around 5330-5350, `GetExistingProcessHost`),
  but `MayReuseAndIsSuitable` rejects any process locked to a different site. So with site isolation ON the limit
  barely lowers the count. It only matters with site isolation OFF (B3) or for unlocked pages.
- Recommendation: add a chrome://flags multi-choice `toxi-renderer-process-limit` (Auto, 2, 3, 4, 6, 8) mapped
  directly to `renderer-process-limit`; no default injected. Document that it pairs with site isolation off.
- Better lever with site isolation on: `kProcessPerSiteUpToMainFrameThreshold`
  (`content_features.cc:1005`, ENABLED on desktop; threshold param `ProcessPerSiteMainFrameThreshold` = 2 at
  line 1014; memory cap 2 GB at line 1033). Raising the param default 2 -> 4 lets more same-site tabs share one
  process. Cost: tabs of one site hang together. Optional, Phase 3.

### B3. Site isolation as a user setting, default ON

- Default ON is already the stock state on desktop: `chrome/common/chrome_features.cc:1700-1706`
  `kSitePerProcess` is ENABLED on non-Android, read in
  `ChromeContentBrowserClient::ShouldEnableStrictSiteIsolation()` (`chrome_content_browser_client.cc:2725`).
  There is no desktop memory threshold that turns it off (the thresholds in `components/site_isolation/features.cc`
  are Android-only, except the default-disabled `kOriginIsolationMemoryThreshold`).
- Best existing hook, zero code: chrome://flags entry `site-isolation-trial-opt-out`
  (`chrome/browser/about_flags.cc:5962-5965`, choices at line 1248, internal name defined in
  `chrome/browser/site_isolation/about_flags.h:12`), `kOsAll`, which emits `--disable-site-isolation-trials`.
  That switch is read (cached) in `content/public/browser/site_isolation_policy.cc:36-47` and makes
  `UseDedicatedProcessesForAllSites()` false (lines 85-92). User story: "chrome://flags/#site-isolation-trial-opt-out
  = Disabled (not recommended)" and relaunch.
- Ungoogled exposes custom flags through `chrome/browser/ungoogled_flag_entries.h`,
  `bromite_flag_entries.h`, `ungoogled_platform_flag_entries.h` and `existing_switch_flag_entries.h`, all `#include`d
  inside `kFeatureEntries[]` in `about_flags.cc` (`add-ungoogled-flag-headers.patch`,
  `add-flags-for-existing-switches.patch`). Toxi adds its own pair (section E).
- Consequences when the user turns it OFF (document in the flag description): spare renderer is automatically off
  (B1 logic), OOPIFs fold back into parent renderers, the process limit (B2) becomes effective, and security drops.
  Chromium's own published estimate for strict site isolation overhead is roughly 10-13% memory; not measured here.
- Optional later (heavier, only if a Settings toggle is wanted): add a Local State pref and return true from
  `ChromeContentBrowserClient::ShouldDisableSiteIsolation()` (`chrome_content_browser_client.cc:2775`) when set,
  plus a Settings WebUI row. Pref availability that early must be checked; the flag route avoids this. Not recommended now.

## C. Memory

### C1. Memory Saver default on and aggressive
- `components/performance_manager/user_tuning/prefs.cc:30-38`: `kMemorySaverModeState` default
  `MemorySaverModeState::kDisabled` -> `kEnabled`; `kMemorySaverModeAggressiveness` default `kMedium` -> `kAggressive`.
- Timers: `chrome/browser/performance_manager/policies/memory_saver_mode_policy.cc:202-211` currently 6 h / 4 h /
  2 h (conservative/medium/aggressive). Even "aggressive" waits 2 h, so default-on alone does little. Change the
  aggressive return to a short value. DECISION NEEDED: aggressive at 15, 30 or 60 minutes; I recommend 30 minutes
  (and medium 2 h, conservative 4 h to keep the ladder). Revisit counts at lines 214-224 can stay.
- Background tab freezing is already on: `kTabFreezingEnabled` default true in the same `prefs.cc`;
  `kUrgentPageDiscarding` is ENABLED (`components/performance_manager/features.cc:176`).
- Ungoogled: not changed. `remove-uneeded-ui.patch` only strips learn-more links, so chrome://settings/performance
  (Memory Saver toggle plus mode picker) remains. That page is the user toggle.
- RAM: biggest runtime win; each discarded tab frees roughly 50-150 MB. Cost: discarded tabs reload on revisit.
  Risk: none for build. Existing profiles keep old pref values; new profiles get the new defaults.

### C2. Back-forward cache
- `content/browser/back_forward_cache/back_forward_cache_impl.cc:77-84`: `kBackForwardCacheSize` param
  `cache_size` = 6 -> 2; TTL `kDefaultTimeToLiveInBackForwardCacheInSeconds` (line 121) 600 -> 120.
  Cached pages keep their renderer processes alive, so this also trims process count.
- Alternative (DECISION NEEDED): disable fully at `content_features.cc:187`
  (`kBackForwardCache` -> DISABLED). Recommendation: size 2, not disabled, to keep instant back/forward.
- Ungoogled: not touched. I did not find a `back-forward-cache` entry in `about_flags.cc`, so add
  `FEATURE_VALUE_TYPE(features::kBackForwardCache)` to the toxi flag header if a toggle is wanted.

### C3. Preconnect / prerender
Already done by ungoogled: `0006-modify-default-prefs.patch` sets `NetworkPredictionOptions::kDefault = kDisabled`
(`chrome/browser/preloading/preloading_prefs.h`, registered in `preloading_prefs.cc:17-19`) and search suggest off.
Nothing to add.

### C4. Optimization Guide, on-device models, Lens, side panel, HaTS
`core/ungoogled-chromium/disable-ai.patch` already edits the optimization-guide, glic, screen-ai, lens,
read-anything side panel, page-content-annotations, component-updater, autofill-ML and HaTS-related files, and
`flags.windows.gn`/`flags.gn` plus `safe_browsing_mode=0` and `disable-fetching-field-trials` block the network
side. `ChromeBrowserMainExtraPartsOptimizationGuide` is compiled only under `BUILDFLAG(USE_ON_DEVICE_MODEL_SERVICE)`
(`chrome_browser_main.cc` around line 860). Nothing new to patch; the Section 0 process listing confirms no
`on_device_model` utility process starts.

### C5. Utility processes at idle and what can be merged

| Process | Starts | Switch to merge | Risk | RAM gain | Recommendation |
|---|---|---|---|---|---|
| Network service | idle | `kNetworkServiceInProcess` ("NetworkServiceInProcess2"), `content_features.cc:763`, off on desktop. Sandbox is decided by `ChromeContentBrowserClient::ShouldSandboxNetworkService()` | Loses the network sandbox and crash isolation; an HTTP/QUIC/cert bug takes down the browser. Supported config on Android/WebView, thinly exercised on Windows desktop | about 15-25 MB | DECISION NEEDED; opt-in flag only, not default |
| Storage service | idle | `ChromeContentBrowserClient::GetSandboxedStorageServiceDataDirectory()` (`chrome_content_browser_client.cc:7795`) returns `user_data_dir`; returning an empty `base::FilePath` makes `content/browser/storage_partition_impl.cc:244-246` pick the in-process service (runs on the IO thread) | Loses isolation; leveldb work moves to the IO thread, possible jank on a weak CPU. Supported config (Android) | about 15-25 MB | Phase 3, measure first |
| Audio service | lazily, on first playback or device enumeration | `kAudioServiceOutOfProcess` (`content_features.cc:161`, ENABLED on Win) -> DISABLED; logic in `browser_main_loop.cc:1625-1650` | Driver faults hit the browser | about 10-20 MB, only while audio is in use | Phase 3, low value |
| GPU | idle | see D | | | see D |
| Others (data decoder, unzip, proxy resolver, print backend, CDM) | on demand and exit | n/a | | | leave |

`--enable-low-end-device-mode` already exists as a flag (`add-flags-for-existing-switches.patch`). It makes
`SysInfo::AmountOfTotalPhysicalMemory()` report 512 MB (`base/system/sys_info.cc:36-47`), which would also shrink
V8 heap sizing and risks "Aw, Snap" on heavy web apps. Not a default; leave as an opt-in flag.

### C6. Cast / media router
`chrome/browser/media/router/media_router_feature.cc:47` `BASE_FEATURE(kMediaRouter, ENABLED)` -> DISABLED.
Ungoogled already builds with `enable_mdns=false`, `enable_service_discovery=false`, and
`disable-dial-repeating-discovery.patch`, so this just removes the router service and toolbar entry. RAM small;
risk low. Toggle: `FEATURE_VALUE_TYPE(media_router::kMediaRouter)` in the toxi flag header (needs that header include).

### C7. Spellcheck
`chrome/browser/spellchecker/spellcheck_factory.cc:75`: `RegisterBooleanPref(spellcheck::prefs::kSpellCheckEnable, true, ...)`
-> false. Ungoogled: not done (its `block-requests` and `remove-unused-preferences-fields` only touch other things).
Toggle: chrome://settings/languages "Check spelling". RAM: avoids loading the Windows spellchecker and dictionaries
on first text focus. Risk: none.

### C8. Already handled by ungoogled
Translate default off (`0006` plus `toggle-translation-via-switch.patch`), autofill/password/payment defaults off,
background mode pref off, Safe Browsing off, sync/GAIA disabled, Privacy Sandbox disabled.

### C9. V8 / Blink knobs
No safe knob worth shipping without benchmarks. Pointer compression is already on. Do not ship jitless or lite
mode as a default; the existing ungoogled flag `disabling-javascript-optimization-disables-jit` stays opt-in.

## D. GPU and rendering on weak hardware

### D1. What actually happens today (code reading, needs chrome://gpu confirmation)
In `content/browser/gpu/gpu_data_manager_impl_private.cc`, `InitializeGpuModes()` (about lines 478-530) builds the
fallback stack. On Windows `SoftwareGLAllowed()` is true because `ui/gl/gl_features.cc` `kAllowD3D11WarpFallback`
is ENABLED (SwiftShader is not built: `enable_swiftshader=false`). So for a blocklisted GPU, or with
`--disable-gpu` alone, the mode is SOFTWARE_GL: a D3D11 WARP device plus GL context running on the CPU inside the
GPU process. That is heavier in RAM and CPU than pure software compositing. The pure software mode is
`DISPLAY_COMPOSITOR`: the GPU process only hosts Viz with `--use-gl=disabled` (set in `AppendGpuCommandLine`,
lines 1243-1275), no GL context, no WebGL.

### D2. Plan (no command-line injection needed)
1. `chrome/browser/gpu/gpu_mode_manager.cc` `RegisterPrefs`: `kHardwareAccelerationModeEnabled` default true -> false.
   Startup then calls `GpuDataManager::DisableHardwareAcceleration()` (same file, constructor), which pops the fallback stack.
2. `ui/gl/gl_features.cc`: `BASE_FEATURE(kAllowD3D11WarpFallback, ...ENABLED)` -> DISABLED, so the stack bottoms out
   in DISPLAY_COMPOSITOR, not WARP. (`IsAnySoftwareGLAllowed` = WARP or SwiftShader; both then false.)
3. User toggle to re-enable: native Settings > System > "Use graphics acceleration when available"
   (this pref; restart required; the code comment in `GpuAccessAllowed` already references it). Re-enabling on a
   blocklisted GPU still lands in DISPLAY_COMPOSITOR, which is the correct outcome for this PC; on a capable GPU it
   gives real hardware GL.
- Ungoogled: not done. Build risk none (pref and feature defaults). CPU cost: all raster and video color
  conversion on the CPU. For an 8-thread FX-8320 this is normally acceptable; the stock outcome on this GPU is already CPU-based.
  Expected RAM: removes WARP/D3D11 state from the GPU process (my estimate 30-80 MB, unmeasured); adds
  shared-memory raster tile buffers in renderers.

### D3. WebGL
Falls out for free: DISPLAY_COMPOSITOR has no GL, so WebGL is unavailable by default and returns when the user
re-enables acceleration on capable hardware. If "WebGL off even with GPU on" is wanted, the existing ungoogled flag
`disable-webgl` covers it (GPU manager reads it at line 310). Defaulting the `kDisable3DAPIs` pref
(`chrome_content_browser_client.cc:1629`) to true would also work but has no UI; not recommended.

### D4. Video decode
With no hardware GL there is no D3D11/DXVA hardware decode; playback uses FFmpeg/dav1d/libvpx in software
(`proprietary_codecs=true`, `ffmpeg_branding="Chrome"`, so H.264 works). 1080p30 is fine on this CPU; 1080p60 and
4K VP9/AV1 will be CPU-bound. With a 2012 driver the hardware path is blocklisted today anyway, so this is
no regression for the target PC.

### D5. Smooth scrolling and reduced motion (switch-only, uses the hook)
- Add to `toxi::ApplyDefaultSwitches`: append `disable-smooth-scrolling` unless `enable-smooth-scrolling` or
  `disable-smooth-scrolling` is present (the stock chrome://flags entry `smooth-scrolling`,
  `about_flags.cc:5362-5367`, is the user toggle). Append `force-prefers-reduced-motion`
  (`ui/gfx/switches.cc:26`, honored in `ui/gfx/animation/animation.cc:146-152`) unless either reduced-motion
  switch is present (ungoogled already exposes a `force-reduced-motion` flag).
- Honest scope: on Windows, Views UI animations follow the OS "show animations" setting; the switch mainly sets the
  `prefers-reduced-motion` media query for web content and some UI. It is cheap and harmless but not a full animation kill.

### D6. Optional: merge the GPU process
`--in-process-gpu` (`content_switches.cc:466`; honored in `gpu_data_manager_impl_private.cc:461-464` and
`gpu_process_host.cc`) runs the Viz/GPU thread inside the browser, saving one process (about 15-25 MB). It removes
the GPU sandbox boundary and ties GPU crashes to the browser. Ship only as an opt-in flag
(`SINGLE_VALUE_TYPE("in-process-gpu")` in the toxi header), not a default. DECISION NEEDED.

## F. Prioritized implementation list

Phase 2 (low risk, high return). Create these under `patches/toxibrowser/` and append to `patches/series` in this order.

| # | Patch file | Change | Est. RAM | Risk |
|---|---|---|---|---|
| 1 | `toxibrowser/toxi-disable-spare-and-preload-renderers.patch` | `content_features.cc:1191` spare renderer DISABLED; `ui_features.cc:205` `kPreloadTopChromeWebUI` DISABLED | -40 to -100 MB idle, -1 to -2 procs | low |
| 2 | `toxibrowser/toxi-memory-saver-defaults.patch` | `user_tuning/prefs.cc:30-38` enabled + aggressive; `memory_saver_mode_policy.cc:202-211` shorter timers | largest for many tabs | low |
| 3 | `toxibrowser/toxi-software-gpu-default.patch` | `gpu_mode_manager.cc` pref default false; `gl_features.cc` WARP fallback DISABLED | -30 to -80 MB (unmeasured), smoother on blocklisted GPU | low |
| 4 | `toxibrowser/toxi-reduce-back-forward-cache.patch` | `back_forward_cache_impl.cc` size 6 -> 2, TTL 600 -> 120 | -50 to -200 MB with browsing history | low |
| 5 | `toxibrowser/toxi-disable-spellcheck-and-cast.patch` | `spellcheck_factory.cc:75` false; `media_router_feature.cc:47` DISABLED | -5 to -20 MB | low |
| 6 | `flags.windows.gn` edit (not a patch) | `enable_widevine=false`, `enable_background_mode=false`, `enable_openxr=false` | binary/page-in only | very low to low |
| 7 | `toxibrowser/toxi-defaults-and-flags.patch` | new `chrome/browser/toxi/toxi_defaults.{h,cc}` + `chrome/browser/BUILD.gn` source entry + 1-line call in `chrome_feature_list_creator.cc`; new `toxi_flag_choices.h`/`toxi_flag_entries.h` + 2 includes in `about_flags.cc` (renderer-process-limit choices, spare renderer, preload WebUI, BFCache, MediaRouter, in-process-gpu entries); smooth scrolling off and reduced motion on | UX, small RAM | low-medium (touches `BUILD.gn` and `about_flags.cc`, files other ungoogled patches also touch) |
| 8 | `flags.windows.gn` edit | `enable_pdf=false` (try it alone first; fallback is the `always_open_pdf_externally` pref default) | binary only | medium-low |

Phase 3 (measure after Phase 2, fork decisions required)

| # | Patch file | Change | Est. RAM | Risk |
|---|---|---|---|---|
| 9 | `toxibrowser/toxi-storage-service-in-process.patch` | `GetSandboxedStorageServiceDataDirectory()` returns empty path | -15 to -25 MB, -1 proc | medium (isolation, IO thread) |
| 10 | `toxibrowser/toxi-audio-in-process.patch` | `kAudioServiceOutOfProcess` DISABLED | -10 to -20 MB while playing | low-medium |
| 11 | `toxibrowser/toxi-process-per-site-threshold.patch` | `content_features.cc:1014` threshold 2 -> 4 | fewer procs with many same-site tabs | low-medium (shared hangs) |
| 12 | opt-in flags only, no default change | `NetworkServiceInProcess2` entry, `in-process-gpu`, existing `enable-low-end-device-mode` | -15 to -25 MB each | medium-high (security, stability) |

Decisions still needed from the owner: Memory Saver aggressive timer (15/30/60 min), BFCache size 2 vs off,
whether in-process network/GPU stay opt-in or become default, and whether `enable_pdf=false` is worth the build
risk versus the pref default.

Verification after each phase: `gn gen` must pass with `--fail-on-unused-args` (it validates every arg above);
then rerun the Section 0 process listing and the 5-tab test; check chrome://gpu shows "Software only" with GL
disabled and chrome://version shows no unexpected command-line switches.
