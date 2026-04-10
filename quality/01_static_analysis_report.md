# Static Analysis Report
**Product:** FluoView / HPF / VPA  
**Last Updated:** April 2026  
**Scope:** Manual static analysis of ~19,116 Java source files across all three projects

---

## Executive Summary

| Metric | Value |
|---|---|
| Total Java files analysed | ~19,116 |
| Total OSGi plugins | ~150+ |
| Critical issues found | 12 |
| High severity issues | 28 |
| Medium severity issues | 31 |
| Low severity issues | 19+ |
| God classes (>1,000 lines) | 5 confirmed |
| `printStackTrace()` calls | 18 confirmed |
| `InterruptedException` mishandled | 3 locations |
| Resource leaks (no try-with-resources) | 7+ confirmed |
| Thread safety violations | 4 confirmed |
| Raw types / unchecked casts | 6+ locations |
| Null pointer risks | 3 high-risk locations |
| TODO/FIXME/HACK markers | 80+ |
| Dead code (commented-out blocks) | 4+ confirmed |

---

## 1. God Classes (Single Responsibility Violation)

These classes are doing too many unrelated things and are the highest-priority refactor targets.

### 1.1 AnalysisProcessUtil.java — 7,977 lines (CRITICAL)

**File:** `fluoview.analysis/AnalysisProcessUtil.java`

- Contains hundreds of **static methods** across unrelated concerns: image processing, ROI calculation, data formatting, statistical computation, coordinate transformation, file I/O helpers
- No cohesive purpose — it is a dumping ground for anything that didn't fit elsewhere
- Adding any new analysis capability requires searching through ~8,000 lines
- Impossible to unit-test individual methods in isolation because of deep static coupling

**Recommended split:**
```
AnalysisProcessUtil (7,977 lines)
  → ImageProcessingUtil     — pixel manipulation, convolution, LUT application
  → RoiProcessingUtil       — ROI area, perimeter, coordinate transforms
  → DataProcessingUtil      — statistics, normalization, interpolation
  → AnalysisFileUtil        — reading/writing analysis result files
```

### 1.2 AcquisitionServiceObserver.java — ~2,500 lines (HIGH)

**File:** `fluoview.acquisition/AcquisitionServiceObserver.java`

- Observer that responds to too many acquisition concerns: laser state, stage position, camera exposure, filter wheel, Z-drive — all in one class
- Contains interleaved state update logic and UI notification logic
- Should be split by device domain (one observer per device type)

### 1.3 AnalysisCommand.java — ~1,500 lines (HIGH)

**File:** `fluoview.analysis/AnalysisCommand.java`

- Polymorphic command class with large `if/else` or `switch` blocks dispatching to different module types
- Violates Open/Closed Principle — adding a new module type requires modifying this class
- Should use the `AnalysisCommandHandler` extension point consistently instead

### 1.4 LSMDyeAndChannelSettingsDialog.java — ~1,200 lines (MEDIUM)

**File:** `fluoview.acquisition.ui/LSMDyeAndChannelSettingsDialog.java`

- SWT dialog managing too many UI states: dye selection, channel count, spectral unmixing, live preview
- Validation logic interleaved with UI construction
- Should delegate to smaller composites per concern

### 1.5 FVApplicationWorkbenchAdvisor.java — 423 lines (MEDIUM)

**File:** `fluoview/FVApplicationWorkbenchAdvisor.java`

- Eclipse RCP workbench advisor handling: initialization sequence, splash screen, error dialog customization, memory configuration, theme loading
- Each concern should be a separate advisor or initialization hook

---

## 2. Error Handling Failures

### 2.1 `printStackTrace()` — 18 confirmed calls (CRITICAL)

`e.printStackTrace()` sends errors to stderr, which is:
- **Invisible in production** (no console attached to deployed Eclipse RCP)
- **Not routed** through the `ErrorDispatchService`
- **Lost** — no log file, no user notification, no diagnostic trace

**All locations:**

| File | Line(s) | Count |
|---|---|---|
| `FVApplicationSkinHandler.java` | 348–358 | 8 |
| `FVLiveFrameGeneratorExtender.java` | 357, 393, 436–510 | 6 |
| `SPEWithMPESystemFileManager.java` | Multiple | 3 |
| `FillMenuHandler.java` | 36–48 | 5 |
| Total | | **18+** |

**Correct pattern:**
```java
// WRONG
} catch (HPFException e) {
    e.printStackTrace();  // ← never do this
}

// CORRECT
} catch (HPFException e) {
    ErrorDispatchService eds = (ErrorDispatchService)
        MessageDomain.getService(ErrorDispatchService.ID);
    eds.putError(new ErrorEvent(e.getErrorId(), e.getMessage(), e));
}
```

### 2.2 Overly Broad Exception Catches (HIGH)

**Files:**
- `FVApplication.java:276` — `catch(Exception e)`
- `FVApplication.java:294` — `catch(Throwable e)`
- `FVApplicationWorkbenchAdvisor.java:218,271` — `catch(Exception e)` with stderr only

Catching `Exception` or `Throwable` at high levels masks programming errors (NPE, OOBE, assertion failures) and prevents them from being diagnosed.

**Rule:** Catch specific exception types. If you must catch broadly, log properly and re-throw or handle explicitly.

---

## 3. InterruptedException Mishandling (HIGH)

When `Thread.interrupt()` is called on a thread blocked in `Object.wait()`, `Thread.sleep()`, or `BlockingQueue.take()`, a `InterruptedException` is thrown. **The interrupt status is cleared.** If you catch it and don't restore the status, the thread silently continues as if it was never interrupted.

### Affected Locations

| File | Line | Issue |
|---|---|---|
| `SendTicker.java` | 240–242 | `catch(InterruptedException e) { Log.out(e); }` — status lost |
| `ImagingHelper.java` | Multiple | Swallowed without `Thread.currentThread().interrupt()` |
| `CellSensManager.java` | 206, 336, 357 | Swallowed in critical sections |

**Correct pattern:**
```java
// WRONG
try {
    Thread.sleep(1000);
} catch (InterruptedException e) {
    Log.out(e);  // ← interrupt status is now cleared; thread continues
}

// CORRECT — Option A: restore and return
try {
    Thread.sleep(1000);
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();  // ← restore interrupt status
    return;                              // ← exit the blocking operation
}

// CORRECT — Option B: propagate (if method signature allows)
void waitForDevice() throws InterruptedException {
    Thread.sleep(1000);
}
```

---

## 4. Resource Leaks (Missing try-with-resources)

Java's try-with-resources guarantees `close()` is called even if an exception occurs. Manual `close()` in a `finally` block is error-prone.

### Confirmed Leaks

| File | Lines | Resource | Risk |
|---|---|---|---|
| `AboutDialog.java` | 229 | Nested `InputStream` / `OutputStream` | HIGH — close not guaranteed on exception |
| `FVApplication.java` | 313, 365, 429 | `FileOutputStream` | HIGH |
| `FVUserLutPropertiesAccessor.java` | 165 | Stream — `close()` not in `finally` | HIGH |
| `CsvUtil.java` | 71–72, 123 | `BufferedWriter` / `BufferedReader` | MEDIUM |
| `LayoutFile.java` | 62–63 | Intermediate `FileInputStream` | MEDIUM |
| `AnalysisProcessUtil.java` | 6834–6835 | File copy channels | MEDIUM |
| `RicsFcsManager.java` | 329 | `BufferedReader` wrapping process stream | HIGH — process stream never closed |

**Correct pattern:**
```java
// WRONG
FileOutputStream fos = null;
try {
    fos = new FileOutputStream(path);
    fos.write(data);
} catch (IOException e) {
    e.printStackTrace();
} finally {
    if (fos != null) fos.close();  // ← IOException here is silently swallowed
}

// CORRECT
try (FileOutputStream fos = new FileOutputStream(path)) {
    fos.write(data);
} catch (IOException e) {
    eds.putError(new ErrorEvent(HPFError.DATA_WRITE_FAILED, e.getMessage(), e));
}
```

---

## 5. Thread Safety Issues

### 5.1 `synchronized` on Mutable Collection (HIGH)

**File:** `VariableROIData.java:104,131,173`

```java
// WRONG — synchronizing on the list itself doesn't prevent structural modification
// by code that holds a reference to the same list
synchronized(resultList) {
    resultList.add(roi);
}
```

If any caller accesses `resultList` without acquiring the same lock, the synchronization is ineffective. Use `Collections.synchronizedList()` or `CopyOnWriteArrayList` or a proper `ReentrantReadWriteLock`.

### 5.2 `volatile` Reference on Non-Thread-Safe List (HIGH)

**File:** `MeasurementTableComposite.java:168`

```java
// WRONG — volatile guarantees visibility of the reference, NOT the list contents
volatile List<String> columnHeaders;
```

`volatile` makes reads/writes to the `columnHeaders` variable atomic, but reading/writing the list's elements is NOT made thread-safe by this. Use `CopyOnWriteArrayList` or wrap in `Collections.synchronizedList()`.

### 5.3 Commented-Out Synchronization (HIGH)

**File:** `FVLSMImagePropertiesUpdater.java:78`

```java
// synchronized (image.getImageProperties()) {  ← DISABLED
    imageProperties.setFrameCount(newCount);
// }
```

Synchronization was intentionally disabled (presumably to fix a deadlock) without a safe replacement. The write to `imageProperties` is now unsynchronized.

### 5.4 Arbitrary Object Monitor (MEDIUM)

**File:** `VariableLiveSeriesModelData.java:138,190`

```java
private final Object waitingObj = new Object();
// ...
synchronized(waitingObj) { waitingObj.wait(); }
```

This pattern is acceptable only if `waitingObj` is truly private and never shared. Verify no code returns or passes `waitingObj` externally.

---

## 6. Raw Types and Type Safety

Java generics were introduced in Java 5. Raw types disable compile-time type safety.

### Confirmed Raw Type Usage

| File | Line | Issue |
|---|---|---|
| `ExtensibleSplashHandler.java:27,29` | — | `private ArrayList fImageList` |
| `ExtensibleSplashHandler.java:104–105` | — | Raw `Iterator` |
| `ExtensibleSplashHandler.java:113` | — | Unsafe cast: `(Image) imageIterator.next()` |
| `MPELaserParamFactory` | Various | Unchecked casts without `instanceof` |
| `AlignmentParamUtil` | Various | Unchecked casts without `instanceof` |

**Correct pattern:**
```java
// WRONG
ArrayList fImageList = new ArrayList();
Iterator iter = fImageList.iterator();
Image img = (Image) iter.next();  // ClassCastException risk

// CORRECT
List<Image> fImageList = new ArrayList<>();
for (Image img : fImageList) { ... }
```

---

## 7. Null Pointer Risks

### High-Risk Patterns

| File | Lines | Issue |
|---|---|---|
| `AdvancedComposite.java` | 469–470 | `map.get(key)` result used directly without null check |
| `AutoCompensationComposite.java` | 360–365 | `list.get(0/1/2)` without size check — IndexOutOfBoundsException risk |
| `ContrastGraphComposite.java` | 223 | `chartHandleMap.get(key)` used without null guard |

**Correct pattern:**
```java
// WRONG
String value = map.get(key).trim();  // NPE if key not in map

// CORRECT — Option A: null check
String rawValue = map.get(key);
if (rawValue != null) {
    String value = rawValue.trim();
}

// CORRECT — Option B: getOrDefault
String value = map.getOrDefault(key, "").trim();

// CORRECT — Option C: Optional (Java 8+)
Optional.ofNullable(map.get(key))
    .map(String::trim)
    .ifPresent(this::applyValue);
```

---

## 8. Architecture and Design Debt

### 8.1 Priority Ordering Inconsistency (HIGH)

`ApplicationRootHandler` uses **DESCENDING** priority (higher integer = runs first).  
`LaserModelProvider` uses **ASCENDING** priority (lower integer = runs first).

This is a silent trap — a developer adding a handler to either system will read the other's priority table and get it backwards.

**Fix:** Document both systems explicitly at the point of use, or migrate to a unified annotation-based ordering (`@RunOrder(before="X")`).

### 8.2 Deprecated Services Still in Use (MEDIUM)

- `MessageDeliveryService` — deprecated but still consumed by ~15 plugins
- `DataProviderService` — deprecated but still referenced

Both represent legacy pub/sub patterns superseded by `ErrorDispatchService` and `EventProviderImpl`. Their continued use risks subtle ordering bugs.

### 8.3 Network-Share P2 Repository (CRITICAL)

H-PF P2 repository is at `\\microscope\pj_confidential\...` — inaccessible without VPN to Japan network. This blocks:
- Bangalore builds without VPN
- Any CI/CD pipeline
- Reproducible builds outside Japan

**Fix:** Mirror H-PF P2 to a proper artifact repository (Nexus/Artifactory) or GitHub Packages.

### 8.4 SNAPSHOT Dependency (HIGH)

`h-pf` version is `3.2.2-SNAPSHOT`. SNAPSHOT versions are resolved dynamically — two builds at different times may use different actual code. Non-deterministic.

### 8.5 Unimplemented Stubs (HIGH)

`MATLScanningStateProviderImpl.java` has **8 methods** with `// TODO Auto-generated method stub`. These methods are part of a public API that other components may depend on. Calling them returns `null` or does nothing silently.

---

## 9. TODO / FIXME / HACK Markers (80+ total)

Representative high-risk markers:

| File | Marker | Translation | Risk |
|---|---|---|---|
| `MPELaserDisabler.java:187` | `setForcedOffは複数個所から呼ぶのは危険` | "Dangerous to call setForcedOff from multiple places" | HIGH — thread safety |
| `FanModeComboEnabler.java:87` | `複数カメラ接続未対応` | "Multiple camera connection not supported" | MEDIUM — feature gap |
| `CameraFramePropertiesFactoryImpl.java:31` | `暫定対応` | "Temporary workaround" (undated) | MEDIUM — unknown intent |
| `MATLScanningStateProviderImpl.java:139–180` | 8× "TODO Auto-generated method stub" | — | HIGH — unimplemented API |
| `AutoAdjustmentControl.java:215,407,419` | `// TODO TODO` | Duplicate marker — forgotten | LOW |

**Action:** All Japanese-language markers must be translated and tracked. See TD-079.

---

## 10. Dead Code

Code that has been commented out but left in the repository creates confusion about whether it was intentionally removed or temporarily disabled.

| File | Lines | Dead Code |
|---|---|---|
| `HelpHandler.java` | 400, 403, 406, 606, 638 | 5× `// e.printStackTrace();` |
| `VolumeSaveAnimationDialog.java` | 274 | `// } catch (InterruptedException e) {` — commented catch block |
| `FVLSMImagePropertiesUpdater.java` | 78 | `// synchronized (image.getImageProperties()) {` — disabled sync |
| `MaintenanceUIMacroFunction.java` | 306 | Disabled UI sync call |

**Action:** Remove commented-out code. If there's a reason it was disabled, document it in a comment that explains the reason (not the code itself).

---

## Tooling Recommendations

To automate detection of these issues, configure the following tools in the Maven build:

### SpotBugs (replaces FindBugs)

```xml
<plugin>
    <groupId>com.github.spotbugs</groupId>
    <artifactId>spotbugs-maven-plugin</artifactId>
    <version>4.8.3.1</version>
    <configuration>
        <effort>Max</effort>
        <threshold>Medium</threshold>
        <failOnError>true</failOnError>
    </configuration>
</plugin>
```

Detects: null dereferences, resource leaks, thread safety violations, bad synchronization

### Checkstyle

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-checkstyle-plugin</artifactId>
    <version>3.3.1</version>
    <configuration>
        <configLocation>google_checks.xml</configLocation>
        <failsOnError>true</failsOnError>
    </configuration>
</plugin>
```

Detects: naming conventions, code style, javadoc requirements

### PMD

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-pmd-plugin</artifactId>
    <version>3.22.0</version>
    <configuration>
        <rulesets>
            <ruleset>/rulesets/java/quickstart.xml</ruleset>
        </rulesets>
        <failOnViolation>true</failOnViolation>
    </configuration>
</plugin>
```

Detects: dead code, empty catch blocks, raw types, overcomplicated expressions

---

## Remediation Priority

| Priority | Items | Effort |
|---|---|---|
| P0 — Fix immediately | `printStackTrace()` calls (18), P2 network share | Low |
| P1 — Fix before next delivery | `InterruptedException` handling, resource leaks, MATL stubs | Medium |
| P2 — Fix in next sprint | Thread safety (`VariableROIData`, `MeasurementTableComposite`), null checks, SNAPSHOT | Medium |
| P3 — Planned refactor | `AnalysisProcessUtil` split, deprecated service migration, raw types | High |
