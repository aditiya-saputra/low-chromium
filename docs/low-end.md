# Low-end preset

This fork targets low-RAM x86-64 machines. The changes here are configuration
and default-feature changes only. They do **not** remove code from the build
graph, and they have **not** been compiled yet (the machine used to author them
cannot build Chromium: 2 logical cores, 3.7 GB RAM).

Upstream reference: `chromium/chromium@1253a244ded71881cd95f9e7376fc95405134043`.

## Feature defaults changed in this fork

All changes are one-line swaps of `base::FEATURE_ENABLED_BY_DEFAULT` to
`base::FEATURE_DISABLED_BY_DEFAULT`. No symbol is removed, so no build target
breaks.

| Feature | File:line | Effect |
| --- | --- | --- |
| `kBackForwardCache` | `content/public/common/content_features.cc:182` | Pages are no longer kept alive in memory after navigating away. Back/Forward reloads the page. |
| `kPrerender2FallbackPrefetchSpecRules` | `content/public/common/content_features.cc:880` | No speculative prefetch driven by speculation rules. |
| `kPrerender2ReuseHost` | `content/public/common/content_features.cc:885` | No prerender host reuse; less speculative renderer work. |
| `kOptimizationGuideFetchingForSRP` | `components/optimization_guide/core/optimization_guide_features.cc:87` | No optimization-hints fetch for search result pages. |
| `kModelQualityLogging` | `components/optimization_guide/core/optimization_guide_features.cc:90` | Model quality telemetry stays off. |
| `kOptimizationGuideModelExecution` | `components/optimization_guide/core/optimization_guide_features.cc:100` | On-device model execution (TFLite) does not start. |

Because `is_official_build` stays `false` for a fork, the variations seed is not
fetched (`components/variations/service/variations_service.cc:276-289`). Server
side field trials therefore cannot re-enable the features above.

## GN args for a low-end build

`out/low-end/args.gn`:

```gn
# Non-official build: disables the variations seed fetch and most
# server-driven experiments. Keep this false.
is_official_build = false

# is_debug defaults to !is_official_build (build/config/BUILDCONFIG.gn:160),
# so it must be set explicitly to get an optimized build.
is_debug = false

# build/config/compiler/compiler.gni:161
symbol_level = 0

# components/safe_browsing/buildflags.gni:11-28 (0 = disabled)
safe_browsing_mode = 0

# build/config/features.gni:31
proprietary_codecs = false

# printing/buildflags/buildflags.gni:30
enable_print_preview = false
```

Build with:

```sh
mkdir -p out/low-end
# put the args above in out/low-end/args.gn
gn gen out/low-end
autoninja -C out/low-end chrome
```

## Runtime flags (no rebuild)

Verified switch names:

| Switch | Source |
| --- | --- |
| `--renderer-process-limit=N` | `content/public/common/content_switches.cc:632` |
| `--process-per-site` | `content/public/common/content_switches.cc:560` |
| `--site-per-process` | `content/public/common/content_switches.cc:684` |
| `--disable-site-isolation-trials` | `content/public/common/content_switches.cc:697` |
| `--metrics-upload-interval=<seconds>` | `components/metrics/metrics_service_client.cc:179-196` |

Example for a 4 GB machine:

```sh
chromium --renderer-process-limit=3 --process-per-site \
  --disable-features=BackForwardCache,Prerender2FallbackPrefetchSpecRules,Prerender2ReuseHost,OptimizationGuideModelExecution,OptimizationHintsFetchingSRP,ModelQualityLogging
```

Note: `kOptimizationGuideFetchingForSRP` carries the explicit feature name
`OptimizationHintsFetchingSRP` (`optimization_guide_features.cc:85-87`), so that
is the name to use on the command line.

## Trade-offs

- Back/Forward navigation becomes slower (full reload).
- Search-result navigation loses speculative prefetch/prerender.
- No on-device model features (summaries, compose, etc.).
- `--disable-site-isolation-trials` lowers the process isolation security
  boundary. Do not use it for everyday browsing of untrusted sites.
- `proprietary_codecs = false` removes hardware-friendly H.264/AAC support;
  playback may fall back to software decoding and use *more* CPU.

## What this preset does not do

- It does not reduce the baseline memory of a renderer process.
- It does not strip telemetry code from the binary; it disables defaults.
- It does not make Chromium buildable on machines below ~8 GB RAM.

## Verification status

- Inspected: `git diff` is one line per feature; all files compile-time only
  swap an enum constant.
- Not run: `gn gen`, `autoninja`, or any test. Recorded as debt.
