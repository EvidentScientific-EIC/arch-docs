# Developer FAQ — FluoView / HPF / VPA
**Last Updated:** April 2026

---

## General

**Q: Why is this Windows-only?**  
A: Multiple core subsystems use Win32 native DLLs: camera interface (CIF), device communication (COMM), image processing, logging, and license validation. These are compiled native binaries with no cross-platform equivalent in the codebase.

**Q: What's the difference between FluoView (fv) and VPA (vpa)?**  
A: Both are built on the HPF framework. FluoView is the primary LSM imaging product. VPA adds volume/3D imaging capabilities (bird-eye view, volume rendering, specialized volume protocols). VPA re-uses most FluoView acquisition and data components, adding volume-specific UI and protocol plugins.

**Q: What's HPF (h-pf)?**  
A: High Performance Framework — the generic microscopy platform layer. It knows about microscopes in general (devices, acquisition, data formats, protocols) but nothing FluoView-specific. FluoView and VPA are products built on top of HPF.

**Q: Why extension points instead of OSGi services?**  
A: Extension points predate OSGi services in the Eclipse platform and allow richer metadata (priority, prerequisite ordering, schema validation). This codebase predates the shift to declarative services and was never migrated. See `architecture/01_adr_osgi_extension_points.md`.

---

## Build & Setup

**Q: The build fails with "Cannot resolve P2 repository". What do I do?**  
A: The Eclipse 2023-09 P2 update site is on an Olympus internal network share. Connect VPN or request a local mirror from the Japan team. Update the URL in `fv/pom.xml` to point to the mirror.

**Q: Do I need to build h-pf before fv every time?**  
A: Only when HPF sources change. If you're only working on FluoView plugins, you can build incrementally. But if `h-pf` has changed (new commit), rebuild it first: `cd h-pf && mvn clean install -Dmaven.test.skip=true`.

**Q: Can I run tests without hardware?**  
A: Yes — most unit tests mock the hardware layer. Integration tests and scenario tests in `vpa.testscenario` require actual hardware. Run `mvn verify` in any plugin's test project to run its JUnit tests.

**Q: Eclipse shows "Plug-in jp.co.olympus.hpf.xxx not found". How to fix?**  
A: The Target Platform needs to be set. Window → Preferences → Plug-in Development → Target Platform → reload/activate the `.target` file.

---

## Architecture

**Q: How do I add new behaviour without modifying existing code?**  
A: Define or contribute to an extension point. This is the core extensibility pattern. You define a `plugin.xml` contribution to an existing `extension-point`, implement the required interface, and the framework discovers it at startup. Never modify existing plugin source for new features.

**Q: What is HPFRoot and why is it a singleton?**  
A: `HPFRoot` is the bootstrap entry point that initialises all OSGi components and services in dependency order. It's a singleton because the platform assumes a single application instance. All service access flows through `HPFRoot.getInstance()` or component domain lookups.

**Q: What is FunctionEnablerService and why can't I just call `button.setEnabled()`?**  
A: Feature availability is complex — it depends on acquisition state, license, device connection, and other factors simultaneously. `FunctionEnablerService` manages this as a formal state machine. UI elements subscribe to `functionId` conditions and react automatically. Direct `setEnabled()` calls would break this contract and cause state mismatches.

**Q: Why do some extension points use DESCENDING priority and others ASCENDING?**  
A: This is a known inconsistency. `ApplicationRootHandler` uses descending (100 = first). `LaserModelProvider` uses ascending (1 = first). Always check the schema or `*Loader` class for the specific extension point you're contributing to. See `architecture/04_technical_debt_register.md`.

**Q: What is the MATL engine?**  
A: Multi-Area Time-Lapse — the most complex subsystem. It orchestrates acquisition across multiple physical areas of a sample over time, managing XY stage movement, area pausing, group pausing, cycle rest intervals, and error recovery. It has 70 states and 70+ event types. See `diagrams/07_matl_protocol.md`.

---

## Coding

**Q: Where should I put error handling?**  
A: Catch `HPFException` at state machine boundaries, transition to a safe state (usually Idle), then dispatch via `ErrorDispatchService.putError()`. Never use `printStackTrace()` — it's invisible in production. Never swallow `InterruptedException` — always call `Thread.currentThread().interrupt()`.

**Q: How do I access a service from my handler?**  
A: Through the component domain:
```java
DeviceControlService deviceService =
    (DeviceControlService) DeviceDomain.getService("jp.co.olympus.hpf.device.component.deviceservice");
```

**Q: What is a `RiJEvent`?**  
A: The base class for all events in the MATL protocol engine. Each event has a unique `long` ID and an `isTypeOf(id)` method. Events are dispatched via `EventProviderImpl` (Active Object pattern — blocking queue + worker thread).

**Q: I see Japanese comments in the code. What do I do?**  
A: This is normal — the codebase was written by Japanese teams. Key translations you'll encounter:
- `暫定対応` = "temporary workaround"
- `仮実装` = "provisional/stub implementation"
- `未対応` = "not supported / not implemented"
- `複数カメラ接続未対応` = "multiple camera connection not supported"
- `移動したい` = "want to move (refactor)"

These Japanese comments often mark `TODO` items or known limitations. See `architecture/04_technical_debt_register.md`.

**Q: How do I write a test for a new plugin?**  
A: Create a companion test plugin named `{your-plugin}.test` in the `test/plugins/` directory of the same project. Use `HPFExtensionRegistry` test support to inject mock extension points. See `testing/02_test_plan.md` for the test strategy.
