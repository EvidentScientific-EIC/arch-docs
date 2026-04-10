# Test Coverage Map
**Product:** FluoView / HPF / VPA  
**Last Updated:** April 2026  
**Source:** Static analysis of test plugins across all three projects

---

## Summary

| Project | Test Plugins | Test Classes (est.) | Coverage Signal |
|---|---|---|---|
| `fv` | ~60 | ~1,900 | Moderate — mostly unit, some integration |
| `h-pf` | ~35 | ~900 | Moderate — core services well tested |
| `vpa` | ~4 | ~85 | LOW — scenario-based only |
| **Total** | **~99** | **~2,885** | **Uneven — critical gaps exist** |

---

## Test Plugin Inventory

### FluoView (`fv`) — Test Plugins

| Plugin | Focus Area | Notes |
|---|---|---|
| `jp.co.olympus.fluoview.analysis.test` | Analysis module commands | Covers FRAP, FRET, RRCM handlers |
| `jp.co.olympus.fluoview.analysis.cellsens.test` | CellSens integration | Partial — CellSens mocking incomplete |
| `jp.co.olympus.fluoview.acquisition.test` | Acquisition state machine | FSM state transitions; no hardware |
| `jp.co.olympus.fluoview.acquisition.ui.test` | Acquisition UI composites | SWT widget tests (fragile) |
| `jp.co.olympus.fluoview.data.test` | Image model, metadata | OIF/OIB read/write round-trips |
| `jp.co.olympus.fluoview.data.format.oif.test` | OIF file format | Round-trip correctness |
| `jp.co.olympus.fluoview.data.format.oib.test` | OIB file format | Round-trip correctness |
| `jp.co.olympus.fluoview.data.format.ida.test` | IDA/VSI file format | Partial |
| `jp.co.olympus.fluoview.lut.test` | LUT properties, user LUTs | LUT read/write |
| `jp.co.olympus.fluoview.roi.test` | ROI model | Shape arithmetic, serialization |
| `jp.co.olympus.fluoview.measurement.test` | Measurement service | Table building, CSV export |
| `jp.co.olympus.fluoview.protocol.test` | Protocol parsing | Protocol file read/write |
| `jp.co.olympus.fluoview.macro.test` | Macro execution | Command dispatch only |
| `jp.co.olympus.fluoview.asac.test` | ASAC/AutoCompensation | Parameter calculation |
| `jp.co.olympus.fluoview.ricsfcs.test` | RICS/FCS analysis | Calculation correctness |
| `jp.co.olympus.fluoview.superres.test` | Super-resolution | Algorithm correctness |
| `jp.co.olympus.fluoview.tiling.test` | Tiling/stitching | Position arithmetic |
| `jp.co.olympus.fluoview.util.test` | Utility classes | CsvUtil, LutUtil, StringUtil |
| `jp.co.olympus.fluoview.preferences.test` | Preference store | Load/save round-trips |
| `jp.co.olympus.fluoview.handler.test` | Application handlers | Handler priority/ordering |

### HPF (`h-pf`) — Test Plugins

| Plugin | Focus Area | Notes |
|---|---|---|
| `jp.co.olympus.hpf.core.test` | HPFRoot, ComponentLoader | Bootstrap lifecycle |
| `jp.co.olympus.hpf.core.enabler.test` | FunctionEnablerService | State activation/deactivation |
| `jp.co.olympus.hpf.core.event.test` | EventProviderImpl | Async dispatch, ordering |
| `jp.co.olympus.hpf.core.error.test` | ErrorDispatchService | Error delivery, listener ordering |
| `jp.co.olympus.hpf.acquisition.test` | Acquisition state machine | StateMachine.java transitions |
| `jp.co.olympus.hpf.acquisition.protocol.test` | Protocol parsing | Protocol file parsing |
| `jp.co.olympus.hpf.data.test` | HPFImageModel, metadata | Image model correctness |
| `jp.co.olympus.hpf.data.io.test` | I/O pipeline | Read/write pipeline |
| `jp.co.olympus.hpf.device.model.test` | EMF device model | Generated model correctness |
| `jp.co.olympus.hpf.device.service.test` | DeviceControlService | Service contracts |
| `jp.co.olympus.hpf.message.test` | MessageDeliveryService | Pub/sub delivery |
| `jp.co.olympus.hpf.util.test` | HPFExtensionRegistry | Extension registry test injection |
| `jp.co.olympus.hpf.protocol.matl.test` | MATL engine | State transitions (partial) |

### VPA — Test Plugins

| Plugin | Focus Area | Notes |
|---|---|---|
| `jp.co.olympus.vpa.testscenario` | Scenario-based E2E tests | NOT JUnit — custom runner |
| `jp.co.olympus.vpa.volume.test` | Volume rendering | Basic rendering smoke tests |
| `jp.co.olympus.vpa.acquisition.test` | VPA acquisition | Thin layer tests |
| `jp.co.olympus.vpa.data.test` | VPA data model | HDF5 read/write |

---

## VPA Test Scenario Framework

VPA uses a **custom scenario-based test framework** (`jp.co.olympus.vpa.testscenario`), not standard JUnit.

```
testscenario/
├── ScenarioRunner.java         — Loads and runs .scenario files
├── ScenarioStep.java           — Single step: action + expected outcome
├── ScenarioAssert.java         — Custom assertions for image data
└── scenarios/
    ├── volume_acquisition.scenario
    ├── stitching_4x4.scenario
    └── multicolor_volume.scenario
```

**Important:** These scenario tests require hardware or a simulation environment. They cannot run in a standard CI environment.

---

## Critical Coverage Gaps

The following subsystems have **zero or near-zero automated test coverage**:

| Subsystem | Gap Type | Risk | Notes |
|---|---|---|---|
| **TRU-AI / AI-Assist** | Zero tests | HIGH | `jp.co.olympus.fluoview.acquisition.aiassist` — no test plugin exists |
| **Maintenance UI** | Zero tests | HIGH | `jp.co.olympus.fluoview.maintenance` — no test plugin; manual only |
| **RPC / Remote Control** | Zero tests | HIGH | `jp.co.olympus.hpf.rpc` — no test plugin |
| **CIF Native Layer** | Zero tests | HIGH | Native DLL bridge — untestable without hardware |
| **COMM Native Layer** | Zero tests | HIGH | Device communication — hardware-dependent |
| **EMF Device Model (runtime)** | Near-zero | MEDIUM | Generated model tested structurally, not behaviorally |
| **Streaming / Live Frame** | Zero tests | HIGH | `FVLiveFrameGeneratorExtender` — no test coverage |
| **MATL Engine (full)** | Partial | HIGH | Only entry/exit state transitions tested; 70-state graph partially covered |
| **VPA Volume Rendering** | Near-zero | MEDIUM | Smoke tests only; no correctness validation |
| **File Format: OIR** | Zero tests | MEDIUM | RawPersistentAdapterFactory has no test plugin |
| **File Format: HDF5** | Near-zero | MEDIUM | VPA test only; no fv-side coverage |
| **RICS/FCS (full pipeline)** | Partial | MEDIUM | Algorithm tested; UI + protocol integration untested |
| **ASAC auto-calibration** | Partial | MEDIUM | Calculation tested; hardware response loop not tested |

---

## Test Pattern Usage

### JUnit 4 (dominant pattern — used in ~85% of test classes)

```java
@RunWith(Suite.class)
@Suite.SuiteClasses({ TestA.class, TestB.class })
public class MyPluginTestSuite {}

public class MyServiceTest {
    @Before
    public void setUp() {
        // HPFExtensionRegistry.getInstance().setTestMode(true);
    }

    @Test
    public void testBehavior() {
        // arrange / act / assert
    }
}
```

### JUnit 5 (minor — found in newer analysis plugins only)

```java
@ExtendWith(MockitoExtension.class)
class MyAnalysisHandlerTest {
    @Mock DeviceControlService mockDevice;

    @Test
    void execute_returnsSuccess_whenThresholdValid() { ... }
}
```

### Eclipse PDE Test (OSGi context tests)

```java
public class MyExtensionPointTest extends TestCase {
    // Runs inside OSGi runtime via PDE test launcher
    public void testExtensionLoading() {
        IExtensionRegistry registry = Platform.getExtensionRegistry();
        // ...
    }
}
```

### VPA Scenario Test (custom DSL)

```
# volume_acquisition.scenario
STEP: Set channel count = 2
STEP: Set Z-depth = 100um, step = 1um
ACTION: Start acquisition
ASSERT: Frame count = 200
ASSERT: Voxel size = [0.1, 0.1, 1.0] um
```

---

## Mocking Strategy

| Mock Target | Approach | Location |
|---|---|---|
| `HPFExtensionRegistry` | `setTestMode(true)` + `registerTestExtension()` | `hpf.util` package |
| `DeviceControlService` | Mockito `@Mock` or hand-rolled stub | Per-test class |
| `HPFImageModel` | Factory method creating in-memory model | `hpf.data.test` helpers |
| `FunctionEnablerService` | Hand-rolled stub tracking state calls | `hpf.core.enabler.test` |
| `ErrorDispatchService` | Capture listener; verify errors received | `hpf.core.error.test` |
| Native CIF/COMM layers | **Cannot be mocked** — hardware required | N/A |

---

## Build and Execution

### Running Tests (Maven/Tycho)

```bash
# Run all tests in a project
mvn verify -pl jp.co.olympus.fluoview.analysis.test

# Run with PDE test launcher (requires OSGi runtime)
mvn tycho-surefire:test -Dtycho.testRuntime=p2Installable

# Skip tests (build only)
mvn package -DskipTests
```

### Test Output Location

```
{plugin}/target/surefire-reports/
    TEST-{TestClass}.xml     — JUnit XML results
    {TestClass}.txt          — Console output
```

### Known Test Infrastructure Issues

| Issue | Impact |
|---|---|
| Tests require Eclipse PDE launcher; cannot run with plain `mvn test` | CI setup blocked |
| SWT UI tests require a display server (`DISPLAY` env var on Linux — not applicable on Windows; needs attached display) | Cannot run headless |
| Hardware-dependent tests silently pass if hardware is absent (no skip logic) | False positives |
| H-PF SNAPSHOT dependency makes test reproducibility uncertain | Flaky builds |

---

## Recommendations

| Priority | Action |
|---|---|
| P0 | Add test coverage for TRU-AI / ImagingHelper (zero tests, production feature) |
| P0 | Add skip annotations to hardware-dependent tests so CI doesn't false-positive |
| P1 | Write integration tests for MATL full state graph (currently ~30% covered) |
| P1 | Add OIR and HDF5 file format round-trip tests |
| P2 | Migrate JUnit 4 suites to JUnit 5 in new plugins |
| P2 | Add CI pipeline (GitHub Actions or Jenkins) to run test subset without hardware |
| P3 | Expand VPA tests from scenario-based to JUnit where possible |
