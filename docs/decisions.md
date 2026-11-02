# Decisions

## 2026-11-02

- WebRTC removed: Blink runtime feature ToxiWebRTC gates the API; re-enable with --enable-blink-features=ToxiWebRTC. Reason: privacy (IP leaks) and fewer processes and code paths.
- Phase 1 needed no patches: ungoogled already disables network prediction, hyperlink auditing, search suggestions, alternate error pages. Reason: nothing left to change.
- Spare renderer and preloaded Top Chrome WebUI disabled (toxi-disable-spare-and-preload-renderers.patch). Reason: removes one or two idle processes; chrome://flags can re-enable.
- Memory Saver on by default with discard after 15 minutes hidden (toxi-memory-saver-defaults.patch; aggressive mode default, aggressive timer 15 min). Reason: biggest RAM win with many tabs; chrome://settings/performance toggle still works. Existing profiles keep their old values.
- Software rendering by default (toxi-software-gpu-default.patch; hardware acceleration pref default false, D3D11 WARP fallback disabled). Reason: GPU is blocklisted, so skip the heavier WARP path; Settings > System toggle re-enables.
- Back-forward cache reduced to 1 entry with 120 s TTL (toxi-reduce-back-forward-cache.patch). Reason: cached pages keep renderer processes and memory alive.
- Spellcheck off by default and Media Router (Cast) feature disabled (toxi-disable-spellcheck-and-cast.patch). Reason: avoids loading spellchecker and the router service; Settings > Languages re-enables spellcheck, chrome://flags re-enables Cast.
- flags.windows.gn: enable_widevine=false, enable_background_mode=false. Reason: drops unused CDM and background-mode code; each is declared via declare_args at the tag.
- Smooth scrolling off and reduced motion on by default, plus chrome://flags entries for renderer process limit, network service in process and in-process GPU (toxi-defaults-and-flags.patch). Reason: lowers CPU and animation cost; in-process network and GPU stay opt-in because they remove sandboxing. Smooth scrolling and reduced motion can be restored via chrome://flags smooth-scrolling and force reduced motion.
- Low-end-device mode is not enabled anywhere. Reason: it makes Chromium report 512 MB of RAM and risks crashes on heavy pages.
- PDF: enable_pdf stays true; the "always open PDF externally" pref default is true instead (toxi-pdf-external-default.patch). Reason: avoids build-break risk of removing the PDF engine; users can turn it off in site settings.
- PDF engine still compiled; full removal deferred to later build to avoid build-break risk. Reason: enable_pdf=false touches print preview and many BUILD.gn guards that were not verified.
- Reverted enable_openxr=false in flags.windows.gn. Reason: it also interacts with checkout_openxr and can raise gn errors, and it gives no measurable RAM benefit.
- Branding is ToxiBrowser (toxi-branding.patch: BRANDING file, product name strings, install-mode product path L"ToxiBrowser" for a separate user data dir). Packaging outputs keep the ungoogled-chromium* names because the CI stage action uploads files matching ungoogled-chromium*; rename them later together with that action.
