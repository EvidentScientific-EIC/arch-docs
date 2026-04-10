# Code Quality Report
**Product:** FluoView / HPF / VPA  
**Last Updated:** April 2026  
**Scope:** Broader quality dimensions beyond static analysis — architectural cohesion, API design, OSGi hygiene, testability, and team readiness

---

## 1. API Design Quality

### 1.1 Typo in Public API (MEDIUM)

**File:** `fluoview.acquisition` — `AcquisitionServiceObserver`

```java
// Published method — cannot rename without breaking all implementors
void notifyProtcolGroupStarted();  // ← "Protcol" (missing 'o')
```

This typo is locked in by backward compatibility. Every implementation of this interface must spell it wrong. Recommended fix: deprecate and add correctly-spelled `notifyProtocolGroupStarted()`, delegate to old method during a migration period.

### 1.2 Inconsistent Naming Conventions

Across the codebase, similar concepts are named differently:

| Pattern A | Pattern B | Problem |
|---|---|---|
| `getFrameCount()` | `frameCount()` | Inconsistent getter prefix |
| `addListener(L)` | `registerListener(L)` | Both patterns exist |
| `dispose()` | `close()` | Lifecycle methods inconsistent |
| `initialize()` | `init()` | Setup method inconsistent |
| `FVComponentDomain` | `HPFComponentDomain` | Prefix inconsistency (FV vs HPF) |

This creates friction when searching for methods and when implementing interfaces — developers must check which pattern a given interface uses.

### 1.3 Null Return vs Optional

Service lookup via `FVComponentDomain.getService(id)` returns `null` if the service is not found. This is a frequent source of NPEs — callers assume the service is present without null-checking.

**Recommendation:** Return `Optional<FVComponentService>` from lookup methods. Force callers to explicitly handle the absent case.

```java
// Current — silent NPE risk
DeviceControlService dcs = (DeviceControlService)
    DeviceDomain.getService("jp.co.olympus.hpf.device...deviceservice");
dcs.doSomething();  // NPE if service not registered

// Better
DeviceDomain.findService("...deviceservice", DeviceControlService.class)
    .orElseThrow(() -> new HPFException(HPFError.SERVICE_NOT_FOUND, "..."))
    .doSomething();
```

---

## 2. OSGi Bundle Hygiene

### 2.1 Missing `uses:` Directives in Bundle Manifests (MEDIUM)

OSGi `Export-Package` headers should declare `uses:` constraints when a package's exported types reference types from other packages. Without this, class loading order can vary and cause `ClassCastException` in wiring.

Observed: Several `MANIFEST.MF` files export packages without `uses:` declarations even though the exported types reference types from imported packages.

### 2.2 Over-broad `Import-Package` Declarations (LOW)

Some bundles import entire packages when they only use one or two classes. This increases coupling and makes dependency graphs harder to manage. Use `optional:=true` for packages only needed at runtime.

### 2.3 Split Packages (MEDIUM — suspected)

A split package occurs when two different bundles export the same Java package. OSGi handles this unpredictably — only one bundle's version of the package will be visible to any given bundle. This is a known source of hard-to-diagnose `ClassNotFoundException` and `NoClassDefFoundError`.

**Action needed:** Run `mvn dependency:analyze` and inspect for any `uses constraint violation` warnings from Tycho at build time.

### 2.4 Fragment Bundle Risk (MEDIUM)

Native Win32 fragments (`*.win32.win32.x86_64`) attach to host bundles and expose native methods. If the host bundle version changes, the fragment may fail to attach silently. All native fragment `Fragment-Host` declarations should pin to a specific version range, not just a symbolic name.

```
# Current (risky)
Fragment-Host: jp.co.olympus.hpf.cif

# Better (pinned range)
Fragment-Host: jp.co.olympus.hpf.cif;bundle-version="[3.2.0,4.0.0)"
```

---

## 3. Testability Assessment

### 3.1 Static Method Proliferation

`AnalysisProcessUtil` (7,977 lines, all static methods) cannot be mocked, subclassed, or injected. Any code that calls these static methods becomes untestable in isolation.

**Impact:** Roughly 40% of analysis code is directly or transitively blocked from unit testing by static coupling.

**Mitigation path:** Gradually extract static methods into injectable service classes. Provide a singleton default instance to maintain existing call sites during migration.

### 3.2 Singleton HPFRoot

`HPFRoot.getInstance()` is called throughout the codebase. In tests, this requires either:
- Using `HPFRoot.getInstance().setTestMode(true)` (available but not consistently used), or
- Running full OSGi runtime (slow, fragile)

Tests that bypass HPFRoot by using Mockito cannot test any code path that goes through service lookup.

### 3.3 `new` Operator in Business Logic

Several business classes directly instantiate collaborators with `new` inside methods, making injection impossible:

```java
// Hard to test
public void applySettings() {
    DeviceSettings settings = new DeviceSettings();  // ← can't inject
    settings.load(getPreferenceStore());
    deviceService.apply(settings);
}
```

**Rule for new code:** Accept collaborators via constructor or method parameter. Only use `new` inside factory classes.

### 3.4 UI Logic in Business Classes

Some analysis and acquisition classes mix SWT widget manipulation with business logic in the same method, making the business logic untestable without a display server.

**Rule:** Keep `Display.getDefault().asyncExec()` and `widget.setText()` calls isolated in UI-layer classes. Business logic should return values, not write to widgets.

---

## 4. Logging Quality

### 4.1 No Consistent Logging Framework

The codebase uses at least three different logging approaches:
- `Log.out(message)` — custom HPF logger
- `e.printStackTrace()` — stderr (18 confirmed bad uses)
- `System.out.println()` — scattered in older code
- Eclipse `ILog` — used in some RCP bundles

There is no structured logging, no log levels (DEBUG/INFO/WARN/ERROR), and no way to filter logs by subsystem or severity without changing code.

**Recommendation:** Standardise on `ErrorDispatchService` for errors (already the intended pattern) and adopt SLF4J + Logback for debug/info logging. Logback integrates with Eclipse via a fragment bundle.

### 4.2 Log Content Quality

Where logging does exist, messages often lack context:

```java
// Bad — what device? what was the state? what parameter?
Log.out("Device error occurred");

// Better
Log.out("[LaserController] Failed to set power on channel " + channelIndex
    + ": requested=" + requestedPower + ", actual=" + currentPower);
```

---

## 5. Java Version Adherence

The project uses **Java 21** (confirmed from `fv/pom.xml`). Java 21 features that are available but underused:

| Feature | Since | Where it would help |
|---|---|---|
| Text blocks | Java 15 | Multi-line error messages, SQL, XML strings |
| Records | Java 16 | Immutable data holders in device model |
| `instanceof` pattern matching | Java 16 | Replace unsafe casts with guarded patterns |
| Sealed classes | Java 17 | Model hierarchy (Module subtypes) |
| `switch` expressions | Java 14 | Replace `if/else if` chains in command dispatch |
| `Optional` chaining | Java 8 | Service lookups that can return null |

### Unsafe Cast Pattern (Should Use `instanceof` Pattern Matching)

```java
// Current — in several analysis handlers
Module m = sequence.getNextModule();
if (m instanceof MyAnalysisModule) {
    MyAnalysisModule myModule = (MyAnalysisModule) m;  // ← redundant cast
    myModule.execute();
}

// Java 16+ — cleaner
if (m instanceof MyAnalysisModule myModule) {
    myModule.execute();
}
```

---

## 6. Build Reproducibility

| Issue | Current State | Risk |
|---|---|---|
| H-PF version is SNAPSHOT | `3.2.2-SNAPSHOT` | Builds non-deterministic; different results at different times |
| P2 repository on network share | `\\microscope\pj_confidential\...` | Builds fail without VPN; no CI possible |
| No lock file for P2 dependencies | N/A | Tycho resolves P2 deps at build time; no guarantee of version pinning |
| Native DLL source not in repo | Unknown | Cannot rebuild native layer if DLL becomes corrupt |

**Fix path:**
1. Tag and release `h-pf` as `3.2.2` (drop SNAPSHOT)
2. Mirror H-PF P2 to Nexus/Artifactory or GitHub Packages
3. Pin Tycho P2 resolution using target platform definition file (`.target`)
4. Confirm native DLL source availability with Japan team

---

## 7. Code Duplication

Manual inspection found recurring duplicated patterns that should be extracted:

### 7.1 Repeated Service Lookup Pattern

```java
// This block appears >30 times across the codebase
DeviceControlService dcs = (DeviceControlService)
    DeviceDomain.getService("jp.co.olympus.hpf.device.component.deviceservice");
```

Should be extracted to a typed accessor method on `DeviceDomain`:
```java
public static DeviceControlService getDeviceControlService() {
    return (DeviceControlService) getService(DeviceControlService.ID);
}
```

### 7.2 Repeated Error Dispatch Boilerplate

```java
// Repeated pattern across 12+ catch blocks
ErrorDispatchService eds = (ErrorDispatchService)
    MessageDomain.getService(ErrorDispatchService.ID);
eds.putError(new ErrorEvent(e.getErrorId(), e.getMessage(), e));
```

Should be a static helper:
```java
ErrorDispatcher.dispatch(e);  // resolves service and dispatches in one call
```

### 7.3 ROI Shape Arithmetic

Point-in-polygon, area calculation, and bounding-box logic appear in at least 3 separate classes (`AnalysisProcessUtil`, `RoiModel`, `RoiShapeUtil`). These should be consolidated.

---

## 8. Internationalization / Localization

The product has a mix of Japanese and English comments, variable names, and log messages.

### Issues Found

| Type | Example | Risk |
|---|---|---|
| Japanese comments with known workarounds | `暫定対応` (temporary workaround) | Bangalore team cannot read; issue invisible |
| Japanese method stub comments | `仮実装` (provisional implementation) | Signals incomplete feature; not tracked |
| Japanese safety warnings | `setForcedOffは複数個所から呼ぶのは危険` | Thread-safety warning invisible to non-Japanese readers |
| Mixed language variable names | `flag仮`, `temp暫定` (in older code) | Unsearchable; confusing |

**Action:** Translate all Japanese code comments. Add translated meaning to the Technical Debt Register (TD-079 already tracks this).

---

## 9. Documentation Quality

### 9.1 Public API Javadoc Coverage

Spot-check of core interfaces (`ApplicationRootHandler`, `AnalysisCommandHandler`, `ErrorDispatchService`) shows:
- Method-level Javadoc: **<30%** of public methods documented
- Parameter/return documentation: **rare**
- Exception documentation (`@throws`): **absent in most cases**

**Recommended minimum standard:**
```java
/**
 * Executes this analysis module against the current image context.
 *
 * @param module  the configured analysis module; must not be null
 * @return result containing output data and success/failure status
 * @throws IllegalArgumentException if module type is not supported by this handler
 */
ModuleResult execute(Module module);
```

### 9.2 Extension Point Schema Documentation

`.exsd` schema files define extension point contracts. Most have `description` fields left blank. These descriptions appear in the Eclipse PDE UI and are the primary documentation for plugin contributors.

---

## 10. Overall Quality Score

| Dimension | Score | Notes |
|---|---|---|
| Correctness | 3/5 | Known bugs (TD-043 disabled sync, TD-077 stubs) |
| Testability | 2/5 | Static methods, singletons, HW coupling limit unit test scope |
| Maintainability | 2/5 | God classes, 80+ TODO markers, Japanese comments |
| API Design | 3/5 | Extension point pattern is good; service lookup is verbose and null-unsafe |
| Build Reproducibility | 1/5 | SNAPSHOT + network-share P2 = non-deterministic |
| Logging / Observability | 1/5 | 18 `printStackTrace`, no structured logging |
| Java Version Adherence | 2/5 | Java 21 available but mostly using Java 8-era patterns |
| OSGi Hygiene | 3/5 | Extension point pattern is sound; fragment pinning and split package risks exist |
| Documentation | 2/5 | Architecture docs now created; inline Javadoc sparse |

**Overall: 19/45 (42%)** — Needs significant improvement in logging, testability, and build infrastructure before the Bangalore team can operate independently.
