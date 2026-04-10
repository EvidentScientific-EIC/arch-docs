# Test Plan
**Product:** FluoView / HPF / VPA  
**Last Updated:** April 2026  
**Scope:** Test strategy for Bangalore team taking over development

---

## Test Strategy Overview

The platform has three distinct test tiers, based on hardware dependency:

```
┌─────────────────────────────────────────────────┐
│  Tier 3: Hardware-in-Loop (HIL)                 │
│  Requires physical microscope hardware           │
│  Run: Pre-release only, Japan lab               │
├─────────────────────────────────────────────────┤
│  Tier 2: Integration / OSGi Context             │
│  Requires Eclipse PDE runtime (no hardware)      │
│  Run: Per-PR, CI with display server            │
├─────────────────────────────────────────────────┤
│  Tier 1: Unit Tests                             │
│  Pure Java, no OSGi/hardware dependency         │
│  Run: Every commit, plain mvn test              │
└─────────────────────────────────────────────────┘
```

---

## Tier 1: Unit Tests

**Goal:** Fast feedback on logic correctness. Must run without Eclipse, hardware, or network.

**Tooling:** JUnit 5 (new tests), JUnit 4 (legacy), Mockito 5.x

### What to Unit Test

| Subsystem | Test Target | Mock |
|---|---|---|
| Analysis algorithms | `MyAnalysisCommandHandler.execute()` | `HPFImageModel` (in-memory) |
| File format parsing | `PersistentAdapter.read()` / `write()` | File I/O with temp files |
| State machine logic | `StateMachine.doTransition()` | None — pure logic |
| Error dispatch | `ErrorDispatchService.putError()` | Listener stub |
| Utility classes | `CsvUtil`, `LutUtil`, `FVStringUtil` | None |
| Module validation | `Module.validate()` | None |
| Protocol parsing | `ProtocolParser.parse()` | Reader over string |

### Unit Test Template (JUnit 5)

```java
@ExtendWith(MockitoExtension.class)
class MyAnalysisCommandHandlerTest {

    private MyAnalysisCommandHandler handler;

    @BeforeEach
    void setUp() {
        handler = new MyAnalysisCommandHandler();
    }

    @Test
    void canHandle_returnsTrue_forOwnType() {
        assertTrue(handler.canHandle("MY_ANALYSIS_TYPE"));
    }

    @Test
    void canHandle_returnsFalse_forOtherType() {
        assertFalse(handler.canHandle("OTHER_TYPE"));
    }

    @Test
    void execute_returnsSuccess_whenThresholdValid() {
        MyAnalysisModule module = new MyAnalysisModule();
        module.setThreshold(0.5);

        ModuleResult result = handler.execute(module);

        assertTrue(result.isSuccess());
        assertNotNull(result.getData());
    }

    @Test
    void execute_returnsFailure_whenThresholdNegative() {
        MyAnalysisModule module = new MyAnalysisModule();
        module.setThreshold(-1.0);

        ModuleResult result = handler.execute(module);

        assertFalse(result.isSuccess());
        assertNotNull(result.getErrorMessage());
    }
}
```

### File Format Round-Trip Template

```java
class MyFormatRoundTripTest {

    @TempDir Path tempDir;

    @Test
    void roundTrip_preservesFrameCount() throws Exception {
        HPFImageModel model = createTestModel(10 /* frames */);
        Path file = tempDir.resolve("test.myfmt");

        MyFormatPersistentAdapter writer = new MyFormatPersistentAdapter(model);
        writer.setFilePath(file.toString());
        writer.write();

        HPFImageModel readModel = createEmptyModel();
        MyFormatPersistentAdapter reader = new MyFormatPersistentAdapter(readModel);
        reader.setFilePath(file.toString());
        reader.read();

        assertEquals(10, readModel.getFrameCount());
    }

    @Test
    void roundTrip_preservesChannelMetadata() throws Exception { ... }

    @Test
    void read_throwsHPFException_onCorruptFile() {
        assertThrows(HPFException.class, () -> {
            MyFormatPersistentAdapter reader = new MyFormatPersistentAdapter(createEmptyModel());
            reader.setFilePath("nonexistent.myfmt");
            reader.read();
        });
    }
}
```

---

## Tier 2: Integration Tests (OSGi Context)

**Goal:** Verify component wiring, extension point loading, and service contracts inside a real OSGi runtime.

**Tooling:** Tycho Surefire Plugin, Eclipse PDE test launcher

**When to run:** On every pull request in CI (requires display server for SWT).

### OSGi Test Template

```java
// Must extend TestCase or use @RunWith(JUnit4.class) for PDE test runner
public class MyExtensionLoadingTest extends TestCase {

    public void testHandlerExtensionIsLoaded() {
        IExtensionRegistry registry = Platform.getExtensionRegistry();
        IExtensionPoint point = registry.getExtensionPoint(
            "jp.co.olympus.hpf.core.handler");
        assertNotNull("Extension point must exist", point);

        IExtension[] extensions = point.getExtensions();
        assertTrue("At least one handler must be registered", extensions.length > 0);
    }
}
```

### HPFExtensionRegistry Test Injection

For tests that need to inject fake extensions:

```java
public class FakeHandlerIntegrationTest {

    @Before
    public void setUp() {
        // Enable test mode — allows programmatic extension registration
        HPFExtensionRegistry.getInstance().setTestMode(true);
        HPFExtensionRegistry.getInstance().registerTestExtension(
            "jp.co.olympus.hpf.core.handler",
            new FakeApplicationRootHandler()
        );
    }

    @After
    public void tearDown() {
        HPFExtensionRegistry.getInstance().setTestMode(false);
    }

    @Test
    public void testFakeHandlerIsInvoked() { ... }
}
```

### Component Lifecycle Integration Test

```java
public class ComponentLifecycleTest {

    @Test
    public void testComponentInitializationOrder() {
        // Use HPFRoot in test mode
        HPFRoot root = HPFRoot.getInstance();
        root.setTestMode(true);

        root.initialize();

        // DeviceComponent must be ready before AcquisitionComponent
        assertTrue(root.isComponentReady("jp.co.olympus.hpf.device.component.device"));
        assertTrue(root.isComponentReady("jp.co.olympus.hpf.acquisition.component"));
    }
}
```

### Running Integration Tests

```bash
# From project root
mvn verify -Dtycho.testRuntime=p2Installable \
           -pl jp.co.olympus.hpf.core.test

# With headless display (Linux CI)
export DISPLAY=:99
Xvfb :99 -screen 0 1024x768x24 &
mvn verify

# Windows CI — requires desktop session or VirtualBox display
```

---

## Tier 3: Hardware-in-Loop (HIL) Tests

**Goal:** Validate end-to-end behavior with real microscope hardware. Run pre-release only.

**Location:** Japan lab environment only (hardware not available in Bangalore)

**Tooling:** VPA `testscenario` framework; manual test scripts; custom harness

### Scenario Test Structure

```
# Standard acquisition scenario
SETUP: Connect to FV3000RS microscope
SETUP: Load configuration "standard_2channel.oif"

STEP: Set objective to 60x oil
STEP: Set laser 488nm to 10%
STEP: Set laser 561nm to 15%
STEP: Set Z-stack: start=-5um, end=5um, step=0.5um
STEP: Set time series: 10 frames, 30s interval

ACTION: Start acquisition
WAIT: Until acquisition completes (timeout 600s)

ASSERT: Image file created at output path
ASSERT: Frame count = 200  (10 time points x 20 Z-slices)
ASSERT: Channel count = 2
ASSERT: Voxel size Z = 0.5um
ASSERT: No error events dispatched
```

### HIL Test Categories

| Category | Example Tests | Hardware Needed |
|---|---|---|
| **Acquisition smoke** | Single frame, Z-stack, time series | Camera + laser |
| **Device control** | Laser power set/readback, stage movement | Full system |
| **MATL** | Multi-area time-lapse, 4x4 grid | Full system |
| **File I/O** | Save/reload, round-trip integrity | Any (file only) |
| **Live view** | Frame rate, live histogram | Camera |
| **AI-Assist** | TRU-AI focus, auto-exposure | Camera |

---

## Test Coverage Targets by Subsystem

| Subsystem | Current | Target (6 months) | Approach |
|---|---|---|---|
| Analysis algorithms | ~60% | 90% | JUnit 5 unit tests |
| File formats (OIF/OIB) | ~70% | 90% | Round-trip tests |
| File formats (OIR/HDF5) | ~10% | 70% | Round-trip tests — **new** |
| State machine (acquisition) | ~50% | 80% | FSM transition table tests |
| MATL state machine | ~30% | 70% | Full state graph traversal |
| Error dispatch | ~65% | 90% | Listener contract tests |
| FunctionEnablerService | ~55% | 85% | State activation tests |
| TRU-AI / ImagingHelper | 0% | 40% | Unit tests for non-HW logic |
| RPC | 0% | 50% | Protocol serialization tests |
| Native CIF/COMM | 0% | 0% | Cannot test without hardware |
| VPA volume rendering | ~5% | 30% | Algorithm-level unit tests |

---

## Test Naming Conventions

All new tests must follow this naming scheme:

```
{ClassName}_{methodName}_{scenario}_{expectedOutcome}

Examples:
    MyAnalysisCommandHandler_execute_whenThresholdNegative_returnsFailure
    OIFPersistentAdapter_read_whenFileCorrupt_throwsHPFException
    StateMachine_doTransition_fromIdleToScanning_succeeds
    ErrorDispatchService_putError_notifiesAllListeners
```

---

## CI/CD Test Pipeline (Target State)

```
On Pull Request:
  ┌── Tier 1: Unit tests (mvn test, no PDE) ───── ~2 min ──── MUST PASS
  └── Tier 2: OSGi integration tests ──────────── ~8 min ──── MUST PASS

Nightly:
  ├── Full Tier 1 + Tier 2
  └── Static analysis (checkstyle, spotbugs) ─── ~15 min

Pre-release (Manual trigger):
  └── Tier 3: HIL tests (Japan lab) ──────────── Variable
```

**CI platform recommendation:** GitHub Actions or Jenkins  
**Current state:** No CI/CD pipeline exists — see TD-075

---

## Priority: Tests to Write First

| Priority | Test | Why |
|---|---|---|
| P0 | `ImagingHelper` non-hardware methods | TRU-AI has zero coverage; production feature |
| P0 | Hardware-dependent `@Disabled` annotations | Prevents CI false positives today |
| P1 | OIR format round-trip | Used in production; untested |
| P1 | HDF5 format round-trip | VPA relies on it |
| P1 | MATL full state graph traversal | 70 states, only ~30% hit by current tests |
| P2 | RPC protocol serialization | Zero coverage; critical for remote control |
| P2 | `VariableROIData` thread-safety | TD-040 — synchronized on mutable list |
| P3 | VPA volume algorithm unit tests | Extract logic from rendering pipeline |
