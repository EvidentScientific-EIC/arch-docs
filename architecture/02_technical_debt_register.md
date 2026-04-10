# Technical Debt Register
**Product:** FluoView / HPF / VPA  
**Last Updated:** April 2026  
**Total Issues:** 60+ across all categories

---

## Severity Legend
- **CRITICAL** — Blocks production diagnostics, causes data loss, or creates security risk
- **HIGH** — Likely to cause bugs or make debugging very difficult
- **MEDIUM** — Reduces maintainability; should be fixed in next major refactor
- **LOW** — Code hygiene; fix opportunistically

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

## Recommended Remediation Priority

| Priority | Items | Effort |
|---|---|---|
| P0 — Fix immediately | TD-073 (P2 on network share), TD-010 to TD-013 (`printStackTrace`) | Low-Medium |
| P1 — Fix before first delivery | TD-020 to TD-022 (InterruptedException), TD-030 to TD-036 (resource leaks), TD-077 (MATL stubs) | Medium |
| P2 — Fix in next sprint | TD-040 to TD-043 (thread safety), TD-060 to TD-062 (null checks), TD-074 (SNAPSHOT), TD-075 (CI/CD) | Medium-High |
| P3 — Planned refactor | TD-001 (AnalysisProcessUtil), TD-070 (priority inconsistency), TD-079 (Japanese comments) | High |
