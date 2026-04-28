# Technical Debt Register
**Product:** FluoView / HPF / VPA
**Last Updated:** April 2026
**Total Issues:** 70+ across 11 categories
**Status:** Canonical single source of tech-debt truth for the platform

> This is the **one** tech-debt document. The Master Categorized Index immediately below is the at-a-glance view; per-category sections that follow give detailed evidence and references for each item.

---

## Risk Level Legend
- **CRITICAL** — Blocks production diagnostics, causes data loss, creates security risk, or blocks delivery
- **HIGH** — Likely to cause bugs or make debugging very difficult
- **MEDIUM** — Reduces maintainability; should be fixed in next major refactor
- **LOW** — Code hygiene; fix opportunistically

## Priority Legend
- **P0** — Fix immediately (current sprint)
- **P1** — Fix before first Bangalore-led delivery
- **P2** — Fix in next sprint cycle
- **P3** — Planned major refactor / long-term

## Tech Debt Type Taxonomy

| Type | Meaning |
|------|---------|
| **God Class** | Single Responsibility violation; oversized class with many concerns |
| **Error Handling** | Lost / silenced / mishandled exceptions; `printStackTrace`; broad `catch` |
| **Concurrency** | `InterruptedException` swallowed; non-thread-safe sharing; broken sync |
| **Resource Mgmt** | Streams/files/locks not closed in all paths; missing try-with-resources |
| **Type Safety** | Raw types, unchecked casts, no `instanceof` guards |
| **Null Safety** | Dereferenced lookups without null checks |
| **Architecture** | Structural debt — inconsistent patterns, deprecated services still in use, deps |
| **Stub / Unimplemented** | `TODO Auto-generated method stub` returning `null` silently |
| **Process / Build** | CI/CD missing, SNAPSHOT versions, network-share dependencies |
| **UI Framework** | SWT / CoolBar / composite-lifecycle gotchas with regression history |
| **Comment Debt** | Untranslated Japanese, undated TODOs, dead commented-out code |

---

## Master Categorized Index

Items grouped by **Priority cohort**, sorted within each cohort by **Risk descending** (CRITICAL → HIGH → MEDIUM → LOW). Each row carries a one-line **Risk Rationale** explaining *why* the assigned risk — i.e. what specifically goes wrong in production if the item is left untreated.

### P0 — Fix immediately (current sprint)

| ID | Type | Concern | Risk | Risk Rationale | Effort |
|----|------|---------|------|----------------|--------|
| TD-073 | Process / Build | H-PF P2 repo on internal share — Bangalore can't build w/o VPN | **CRITICAL** | Hard blocker on Bangalore independence; no CI/CD possible while dependency is on internal share | Medium |
| TD-010 | Error Handling | `FVApplicationSkinHandler.java` — 8× `printStackTrace` | CRITICAL | Production skin/init errors vanish into stderr; no remote diagnostics path; bug reports become unactionable | Low |
| TD-011 | Error Handling | `FVLiveFrameGeneratorExtender.java` — 6× `printStackTrace` | CRITICAL | Frame-generation failures during live acquisition silently lost; user sees a frozen image with no error trail | Low |
| TD-012 | Error Handling | `SPEWithMPESystemFileManager.java` — system init `printStackTrace` | CRITICAL | MPE laser init errors hidden; system runs in broken state with no fault indication | Low |
| TD-013 | Error Handling | `FillMenuHandler.java` — 5× `printStackTrace` on menu build | HIGH | Menu items silently missing; user-visible UI breakage with no diagnostic | Low |

### P1 — Fix before first Bangalore-led delivery

| ID | Type | Concern | Risk | Risk Rationale | Effort |
|----|------|---------|------|----------------|--------|
| TD-014 | Error Handling | `FVApplication.java` — broad `catch(Throwable)` | HIGH | Masks programmer errors and `OutOfMemoryError` equally; impossible to triage failure mode | Low |
| TD-015 | Error Handling | `FVApplicationWorkbenchAdvisor.java` — broad `catch(Exception)` | HIGH | Same as TD-014 — startup failures arrive as opaque generic catches | Low |
| TD-020 | Concurrency | `SendTicker.java` — `InterruptedException` swallowed | HIGH | Interrupt status lost → worker thread can't be cleanly stopped; deadlock risk on app shutdown | Low |
| TD-021 | Concurrency | `ImagingHelper.java` — `InterruptedException` swallowed | HIGH | Same shutdown-deadlock risk on the imaging path; stuck threads hold device locks | Low |
| TD-022 | Concurrency | `CellSensManager.java` — `InterruptedException` swallowed | HIGH | CellSens lifecycle hangs on shutdown; native resources leaked | Low |
| TD-030 | Resource Mgmt | `AboutDialog.java` — nested streams, manual close | HIGH | Exception in middle of nested-stream init leaks the outer; file handles accumulate | Low |
| TD-031 | Resource Mgmt | `FVApplication.java` — `FileOutputStream` w/o try-with-resources | HIGH | Lock-file streams leak on exception → false multi-launch detection on next run | Low |
| TD-032 | Resource Mgmt | `FVUserLutPropertiesAccessor.java` — `close()` not in finally | HIGH | LUT property file handle leaks on read error → eventual file-handle exhaustion | Low |
| TD-036 | Resource Mgmt | `RicsFcsManager.java` — `BufferedReader` over process stream never closed | HIGH | Native process pipe buffer fills → analysis subprocess hangs | Low |
| TD-077 | Stub / Unimplemented | `MATLScanningStateProviderImpl.java` — 8 stubs returning null | HIGH | API contract returns `null` without warning; consumers NPE deep in the call stack | Medium |
| TD-080 | Stub / Unimplemented | `MATLScanningStateProviderImpl.java` 8× "TODO Auto-generated stub" | HIGH | Same as TD-077 — overlapping evidence; documents the markers | Medium |
| TD-083 | Concurrency | `MPELaserDisabler.java` — `setForcedOff` race-prone (per Japanese comment) | HIGH | Code itself documents the danger (`複数個所から呼ぶのは危険`); concurrent calls leave laser in inconsistent state — safety implication | Medium |
| TD-033 | Resource Mgmt | `CsvUtil.java` — stream `close()` not guaranteed | MEDIUM | Less-frequently-hit code path; CSV utility in non-critical export flow | Low |
| TD-034 | Resource Mgmt | `LayoutFile.java` — intermediate `FileInputStream` orphaned | MEDIUM | Layout-load path; small leak per invocation, bounded by usage | Low |
| TD-035 | Resource Mgmt | `AnalysisProcessUtil.java` — channel-based file copy uncleaned | MEDIUM | Channel leak on copy failure; analysis-only path, not acquisition-critical | Low |

### P2 — Fix in next sprint cycle

| ID | Type | Concern | Risk | Risk Rationale | Effort |
|----|------|---------|------|----------------|--------|
| TD-040 | Concurrency | `VariableROIData.java` — `synchronized(resultList)` on mutable ArrayList | HIGH | Lock object is the same reference being reassigned — readers and writers may end up on different monitors; classic broken-sync pattern | Medium |
| TD-042 | Concurrency | `MeasurementTableComposite.java` — `volatile List<String>` (false safety) | HIGH | `volatile` covers the *reference* only, not element-level mutation; gives misleading sense of safety to maintainers | Low |
| TD-043 | Concurrency | `FVLSMImagePropertiesUpdater.java` — `synchronized` commented out | HIGH | Code that was once thread-safe now silently racy on image-property updates | Low |
| TD-060 | Null Safety | `AdvancedComposite.java` — `map.get()` unchecked | HIGH | NPE on user-input path → user-visible crash dialog | Low |
| TD-061 | Null Safety | `AutoCompensationComposite.java` — `list.get(0/1/2)` no size check | HIGH | `IndexOutOfBoundsException` on partial data → user-visible crash | Low |
| TD-074 | Process / Build | `h-pf` is `3.2.2-SNAPSHOT` — non-reproducible builds | HIGH | Today's build ≠ yesterday's; bug repro and release auditing become impossible | Medium |
| TD-075 | Process / Build | No CI/CD pipeline | HIGH | No automated quality gate → regressions reach manual test → discovered late and expensive | Medium |
| TD-092 | Comment Debt → Concurrency | `FVLSMImagePropertiesUpdater.java` — commented `synchronized` | HIGH | Same defect surface as TD-043; preserved as a separate marker because the comment is the trail | Low |
| TD-100 | UI Framework | `FVCommonCoolBar.addItemComposite()` not idempotent | HIGH | Every CoolItem-bearing subsystem must remember to handle non-idempotency itself; recurring source of duplicate-item bugs | Medium |
| TD-101 | UI Framework | `FVCommonCoolBar.paintControl` draws drag-handle for ALL CoolItems incl. zero-size | HIGH | Direct cause of US-62077 visual artifact; hidden/empty CoolItems leak phantom drag-handles | Low |
| TD-103 | UI Framework | `MATLAxisComposite.createPart` axis-presence assumption regression-prone | HIGH | Mode → axis matrix incomplete; non-TIMELAPSE area images bypass intended branch and produce empty composites | Low |
| TD-041 | Concurrency | `VariableLiveSeriesModelData.java` — `synchronized(waitingObj)` arbitrary | MEDIUM | Coordination unclear but currently functional; latent re-entrance risk if anyone refactors | Medium |
| TD-050 | Type Safety | `ExtensibleSplashHandler.java` — raw `ArrayList` | MEDIUM | Lost type safety contained to splash handler; failures appear as ClassCastException at boundary | Low |
| TD-051 | Type Safety | `ExtensibleSplashHandler.java` — raw `Iterator` | MEDIUM | Same containment as TD-050 | Low |
| TD-052 | Type Safety | `ExtensibleSplashHandler.java` — unsafe cast `(Image) iter.next()` | MEDIUM | CCE at boundary if iterator yields non-Image; failure is loud, not silent corruption | Low |
| TD-053 | Type Safety | `MPELaserParamFactory`, `AlignmentParamUtil` — unchecked casts | MEDIUM | Same shape as TD-052 across multiple sites | Low |
| TD-062 | Null Safety | `ContrastGraphComposite.java` — `chartHandleMap.get()` unchecked | MEDIUM | Chart fails to render but app does not crash; degraded UX rather than broken | Low |
| TD-087 | Stub / Unimplemented | `RequestGetProtocolVersion.java` — 2× auto-generated constructor stubs | MEDIUM | Empty stubs construct objects in incomplete state; caught at usage in normal QA | Low |
| TD-104 | UI Framework | `FVCommonDraw2dspin.setVisible` bug — workaround via size-zero in MATLAxisComposite | MEDIUM | Workaround is functional; every consumer must know it; documented in source comments | Medium |

### P3 — Planned major refactor

| ID | Type | Concern | Risk | Risk Rationale | Effort |
|----|------|---------|------|----------------|--------|
| TD-001 | God Class | `AnalysisProcessUtil.java` (7,977 lines, all static) | CRITICAL | Brittle to any change; very high blast radius on bugs touching analysis paths | High |
| TD-002 | God Class | `AcquisitionServiceObserver.java` (~2,500 lines) | HIGH | Acquisition-observer mutation paths obscure; high regression risk on any touch | High |
| TD-003 | God Class | `AnalysisCommand.java` (~1,500 lines, polymorphic branching) | HIGH | Polymorphic command branching is hard to reason about; introduces regressions per change | Medium |
| TD-070 | Architecture | Priority ordering inconsistency (DESCENDING vs ASCENDING) | HIGH | New contributors silently get priority direction wrong → ordering bugs that surface only at runtime | High |
| TD-004 | God Class | `LSMDyeAndChannelSettingsDialog.java` (~1,200 lines) | MEDIUM | Large but localized impact (single dialog) | Medium |
| TD-005 | God Class | `FVApplicationWorkbenchAdvisor.java` (423 lines, mixed concerns) | MEDIUM | Bootstrap concerns intertwined; localized | Low |
| TD-071 | Architecture | `MessageDeliveryService` deprecated but still in use | MEDIUM | Service is functional; risk lives in unknown consumers blocking removal | Medium |
| TD-072 | Architecture | `DataProviderService` deprecated but still in use | MEDIUM | Same shape as TD-071 | Medium |
| TD-076 | Architecture | `WorkbenchHelper.java` — global `menuBar` assumes single window | MEDIUM | Single-window assumption holds today; latent risk if multi-window perspective is ever added | Medium |
| TD-078 | Architecture | API typo `notifyProtcolGroupStarted` (locked by back-compat) | MEDIUM | Cosmetic but locked by ABI; perpetual onboarding confusion | Low |
| TD-079 | Comment Debt | Japanese comments untranslated mark known issues | MEDIUM | Onboarding friction; issues invisible to non-Japanese readers — risk is *missed* known limits | Medium |
| TD-081 | Comment Debt | `CameraFramePropertiesFactoryImpl.java` — undated `暫定対応` | MEDIUM | Marks a `暫定対応` (temp workaround) with no date; risk of forgotten landmine | Low |
| TD-082 | Comment Debt | `FanModeComboEnabler.java` — `複数カメラ接続未対応` | MEDIUM | Documents an unsupported configuration; if hardware ever ships in that config the code silently misbehaves | Medium |
| TD-084 | Architecture | `GelImmersionStageSettingsImpl.java` — refactor-to-DeviceControlService never done | MEDIUM | Settings live in non-canonical location; no functional defect today, but architectural drift | Medium |
| TD-093 | Comment Debt | `MaintenanceUIMacroFunction.java` — disabled UI sync call | MEDIUM | Disabled call may or may not be needed; unknown until tested | Low |
| TD-102 | UI Framework | `ContentView.coolbar` private with no accessor — forces hierarchy walks | MEDIUM | Forces fragile traversal code in subclasses; works today but couples subclasses to layout | Low |
| TD-085 | Comment Debt | `CTabRendering2.java` — magic numbers TODO | LOW | Cosmetic / readability only | Low |
| TD-086 | Comment Debt | `AutoAdjustmentControl.java` — duplicate `// TODO TODO` | LOW | Marker noise; no functional impact | Low |
| TD-090 | Comment Debt | `HelpHandler.java` — 5× commented `printStackTrace` | LOW | Dead-code stubs of error handling that was once present | Low |
| TD-091 | Comment Debt | `VolumeSaveAnimationDialog.java` — commented catch block | LOW | Dead code; intent unclear | Low |

---

## Category 1: God Classes (Single Responsibility Violation)

| ID | File | Size | Issue | Severity |
|---|---|---|---|---|
| TD-001 | `fluoview.analysis/AnalysisProcessUtil.java` | **7,977 lines** | Hundreds of static methods across unrelated concerns. Should be split into `ImageProcessingUtil`, `RoiProcessingUtil`, `DataProcessingUtil` | CRITICAL |
| TD-002 | `fluoview.acquisition/AcquisitionServiceObserver.java` | ~2,500 lines | Observer managing too many acquisition concerns | HIGH |
| TD-003 | `fluoview.analysis/AnalysisCommand.java` | ~1,500 lines | Polymorphic command class with excessive branching | HIGH |
| TD-004 | `fluoview.acquisition.ui/LSMDyeAndChannelSettingsDialog.java` | ~1,200 lines | Dialog managing multiple UI states and validations | MEDIUM |
| TD-005 | `fluoview/FVApplicationWorkbenchAdvisor.java` | 423 lines | Workbench advisor handling initialization, splash, error dialogs, memory config | MEDIUM |

---

## Category 2: Error Handling Failures

| ID | File | Lines | Issue | Severity |
|---|---|---|---|---|
| TD-010 | `fluoview/FVApplicationSkinHandler.java` | 348–358 (8 calls) | `e.printStackTrace()` — lost in console, invisible in production | **CRITICAL** |
| TD-011 | `fluoview.acquisition/FVLiveFrameGeneratorExtender.java` | 357, 393, 436–510 (6 calls) | Frame generation errors silently printed to stderr | CRITICAL |
| TD-012 | `fluoview.device.mpelaser/SPEWithMPESystemFileManager.java` | Multiple | System init errors hidden via `printStackTrace()` | CRITICAL |
| TD-013 | `fluoview/FillMenuHandler.java` | 36–48 (5 calls) | Menu building failures silent | HIGH |
| TD-014 | `fluoview/FVApplication.java` | 276, 294 | `catch(Exception e)` and `catch(Throwable e)` — overly broad | HIGH |
| TD-015 | `fluoview/FVApplicationWorkbenchAdvisor.java` | 218, 271 | Broad `catch(Exception)` with only stderr output | HIGH |
| **Total `printStackTrace()` calls** | | **18 confirmed** | Replace all with `ErrorDispatchService` | CRITICAL |

---

## Category 3: InterruptedException Mishandling

| ID | File | Line | Issue | Severity |
|---|---|---|---|---|
| TD-020 | `fluoview.acquisition/SendTicker.java` | 240–242 | `catch(InterruptedException e) { Log.out(e); }` — thread interrupt status lost | HIGH |
| TD-021 | `fluoview.acquisition.aiassist/ImagingHelper.java` | Multiple | `InterruptedException` swallowed without `Thread.currentThread().interrupt()` | HIGH |
| TD-022 | `fluoview.analysis.cellsens/CellSensManager.java` | 206, 336, 357 | Swallowed in critical sections | HIGH |

---

## Category 4: Resource Leaks (Missing try-with-resources)

| ID | File | Line | Issue | Severity |
|---|---|---|---|---|
| TD-030 | `fluoview/AboutDialog.java` | 229 | Nested streams, manual `close()` — not guaranteed on exception | HIGH |
| TD-031 | `fluoview/FVApplication.java` | 313, 365, 429 | `FileOutputStream` without try-with-resources | HIGH |
| TD-032 | `fluoview.util/FVUserLutPropertiesAccessor.java` | 165 | `close()` not in `finally` block | HIGH |
| TD-033 | `fluoview.util/CsvUtil.java` | 71–72, 123 | Stream `close()` not guaranteed in exception paths | MEDIUM |
| TD-034 | `fluoview.handlers/LayoutFile.java` | 62–63 | Intermediate `FileInputStream` created and not closed | MEDIUM |
| TD-035 | `fluoview.analysis/AnalysisProcessUtil.java` | 6834–6835 | File copy with channels; no guaranteed cleanup | MEDIUM |
| TD-036 | `fluoview.analysis.ricsfcs.ui/RicsFcsManager.java` | 329 | `BufferedReader` wrapping process stream — never closed | HIGH |

---

## Category 5: Thread Safety Issues

| ID | File | Lines | Issue | Severity |
|---|---|---|---|---|
| TD-040 | `fluoview.analysis/VariableROIData.java` | 104, 131, 173 | `synchronized(resultList)` on mutable ArrayList | HIGH |
| TD-041 | `fluoview.analysis/VariableLiveSeriesModelData.java` | 138, 190 | `synchronized(waitingObj)` on arbitrary Object | MEDIUM |
| TD-042 | `fluoview.analysis.ui/MeasurementTableComposite.java` | 168 | `volatile List<String>` — volatile reference doesn't make list thread-safe | HIGH |
| TD-043 | `fluoview.data/FVLSMImagePropertiesUpdater.java` | 78 | Commented-out `synchronized` block — synchronization disabled | HIGH |

---

## Category 6: Raw Types and Type Safety

| ID | File | Lines | Issue | Severity |
|---|---|---|---|---|
| TD-050 | `fv_meavn_product/ExtensibleSplashHandler.java` | 27, 29 | `private ArrayList fImageList` — raw type | MEDIUM |
| TD-051 | `fv_meavn_product/ExtensibleSplashHandler.java` | 104–105 | Raw `Iterator` | MEDIUM |
| TD-052 | `fv_meavn_product/ExtensibleSplashHandler.java` | 113 | Unsafe cast from raw type: `(Image) imageIterator.next()` | MEDIUM |
| TD-053 | Multiple | Various | Unchecked casts in `MPELaserParamFactory`, `AlignmentParamUtil` without instanceof guards | MEDIUM |

---

## Category 7: Null Pointer Risks

| ID | File | Lines | Issue | Severity |
|---|---|---|---|---|
| TD-060 | `fluoview.asac.acquisition.ui/AdvancedComposite.java` | 469–470 | `map.get()` result used without null check | HIGH |
| TD-061 | `fluoview.asac.acquisition.ui/AutoCompensationComposite.java` | 360–365 | `list.get(0/1/2)` without size validation | HIGH |
| TD-062 | `fluoview.asac.acquisition.ui/ContrastGraphComposite.java` | 223 | `chartHandleMap.get()` result used without null guard | MEDIUM |

---

## Category 8: Architecture / Design Debt

| ID | Issue | Impact | Severity |
|---|---|---|---|
| TD-070 | **Priority ordering inconsistency** — `ApplicationRootHandler` DESCENDING, `LaserModelProvider` ASCENDING | Causes confusion and potential priority bugs when adding new contributors | HIGH |
| TD-071 | **`MessageDeliveryService` deprecated but still in use** — legacy pub/sub not replaced | Risk of subtle ordering bugs; removal blocked by unknown consumers | MEDIUM |
| TD-072 | **`DataProviderService` deprecated but still in use** | Same risk as TD-071 | MEDIUM |
| TD-073 | **H-PF P2 repository on internal network share** — `\\microscope\pj_confidential\...` | Bangalore cannot build without VPN; no CI/CD possible | **CRITICAL** |
| TD-074 | **`h-pf` version is SNAPSHOT** — `3.2.2-SNAPSHOT` | Builds not reproducible; dependency resolution non-deterministic | HIGH |
| TD-075 | **No CI/CD pipeline** — no `.github/workflows`, `.gitlab-ci.yml`, or `Jenkinsfile` | No automated quality gate; regressions undetected until manual test | HIGH |
| TD-076 | **`WorkbenchHelper.java`** — `menuBar` global assumes single Workbench window | Structural assumption embedded in global state | MEDIUM |
| TD-077 | **`MATLScanningStateProviderImpl.java`** — 8 unimplemented stub methods (`TODO Auto-generated method stub`) | MATL state provider has unimplemented API surface | HIGH |
| TD-078 | **Typo in API**: `notifyProtcolGroupStarted()` (missing 'o' in Protocol) | Public method name locked by backward compatibility | MEDIUM |
| TD-079 | **Japanese comments with no English translation** — `暫定対応`, `仮実装`, `未対応` mark known issues | Onboarding friction; issues invisible to non-Japanese readers | MEDIUM |

---

## Category 9: TODO / FIXME / HACK Markers (80+ total)

| ID | File | Comment | Severity |
|---|---|---|---|
| TD-080 | `MATLScanningStateProviderImpl.java` (lines 139–180) | 8× "TODO Auto-generated method stub" | HIGH |
| TD-081 | `CameraFramePropertiesFactoryImpl.java` (line 31–32) | `暫定対応` (temporary workaround) — undated | MEDIUM |
| TD-082 | `FanModeComboEnabler.java` (line 87) | `複数カメラ接続未対応` (multiple camera not supported) | MEDIUM |
| TD-083 | `MPELaserDisabler.java` (line 187) | `setForcedOffは複数個所から呼ぶのは危険` (dangerous to call from multiple places) | HIGH |
| TD-084 | `GelImmersionStageSettingsImpl.java` (line 301) | Wants to move to `DeviceControlService` — never done | MEDIUM |
| TD-085 | `CTabRendering2.java` (line 345) | `// XXX: The magic numbers need to be cleaned up` | LOW |
| TD-086 | `AutoAdjustmentControl.java` (lines 215, 407, 419) | `// TODO TODO` — duplicate marker | LOW |
| TD-087 | `RequestGetProtocolVersion.java` (lines 27, 37) | 2× auto-generated constructor stubs | MEDIUM |

---

## Category 10: Dead Code

| ID | File | Lines | Issue |
|---|---|---|---|
| TD-090 | `HelpHandler.java` | 400, 403, 406, 606, 638 | 5× `// e.printStackTrace();` commented out |
| TD-091 | `VolumeSaveAnimationDialog.java` | 274 | `// } catch (InterruptedException e) {` — commented catch block |
| TD-092 | `FVLSMImagePropertiesUpdater.java` | 78 | `// synchronized (image.getImageProperties()) {` — disabled sync |
| TD-093 | `MaintenanceUIMacroFunction.java` | 306 | Disabled UI sync call |

---

## Category 11: UI Framework Debt (SWT / CoolBar / Player Composite)

Source: distilled from US-62077 (empty CoolBar regression in MATL image tabs). These are framework-level gotchas that have already caused regressions and will keep doing so unless addressed at the framework layer.

| ID | File | Issue | Severity |
|---|---|---|---|
| TD-100 | `h-pf/.../FVCommonCoolBar.java` — `addItemComposite()` | **Not idempotent.** Every call appends a new `CoolItem`; no built-in protection against duplicate adds, no API for indexed insertion (caller has to use `new CoolItem(coolbar, SWT.NONE, index)` directly). Surface area for ordering bugs. | HIGH |
| TD-101 | `h-pf/.../FVCommonCoolBar.java` — `paintControl` | Draws a drag-handle thumb (3px × 7px dotted line) for **every** `CoolItem` in the bar, including zero-sized ones. Hidden/empty items leave visual artifacts. Hiding via size-zero is *not* sufficient — CoolItem must be removed entirely. Direct cause of the US-62077 visual symptom. | HIGH |
| TD-102 | `h-pf/.../ContentView.java` — `coolbar` field | Declared `private FVCommonCoolBar coolbar` with no accessor. Subclasses that need a reference must walk the widget hierarchy (`getParent()` chain checking `instanceof CoolBar`). Encourages fragile traversal code in subclasses. | MEDIUM |
| TD-103 | `fv/.../MATLAxisComposite.java` — `createPart` | Axis-presence handling assumes only `TIMELAPSE` needs the special "defer until cycle 2" path. Lambda / Z-stack / Lambda-Excitation modes pass through the generic branch but rely on the player having an axis at all — when the area image has no playback axis, downstream composites still get created. Regression-prone area; full mode → axis matrix needs explicit handling. | HIGH |
| TD-104 | `fv/.../MATLAxisComposite.java` (lines 133, 138) | Comments `FVCommonDraw2dspinのsetVisibleバグってるのでサイズ設定` ("setVisible is buggy in FVCommonDraw2dspin so set size [to zero] instead") — the underlying setVisible bug in `FVCommonDraw2dspin` was never fixed; every consumer has to know the size-zero workaround. | MEDIUM |

**Pattern:** all five share the same root cause shape — the framework provides primitives but no safe high-level API, so each subsystem reinvents (often inconsistently) the same workarounds. Recommended direction: introduce idempotent / indexed wrapper APIs in `FVCommonCoolBar` and document the axis-presence contract on `AxisComposite`.

See also: `how-to/06_ui_framework_patterns.md`, `onboarding/04_faq.md` ▸ "UI / SWT" section, `C:\home\claude\US-62077\sessions\US-62077-coolbar-fix.md` for the worked fix.

---

## Recommended Remediation Priority

| Priority | Items | Type Themes | Effort |
|---|---|---|---|
| **P0 — Fix immediately** | TD-073, TD-010 to TD-013 | Process/Build, Error Handling | Low–Medium |
| **P1 — Fix before first delivery** | TD-014, TD-015, TD-020 to TD-022, TD-030 to TD-036, TD-077, TD-080, TD-083 | Error Handling, Concurrency, Resource Mgmt, Stub | Medium |
| **P2 — Fix in next sprint** | TD-040 to TD-043, TD-050 to TD-053, TD-060 to TD-062, TD-074, TD-075, TD-087, TD-092, TD-100 to TD-104 | Concurrency, Type Safety, Null Safety, Process/Build, **UI Framework** | Medium–High |
| **P3 — Planned refactor** | TD-001 to TD-005, TD-070 to TD-072, TD-076, TD-078, TD-079, TD-081 to TD-082, TD-084 to TD-086, TD-090, TD-091, TD-093 | God Class, Architecture, Comment Debt | High |

> The Master Categorized Index at the top of this document is the single source of truth for type, risk and priority per item. The remediation table here groups items into delivery cohorts.
