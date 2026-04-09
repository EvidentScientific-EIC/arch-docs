# FluoView / HPF / VPA — Architecture Diagram Index

**Platform:** Olympus FluoView — scientific laser scanning microscope control software  
**Stack:** Java 21, Eclipse RCP 4.x, OSGi, EMF/ECORE, Maven/Tycho, Windows x86_64  
**Projects:** `fv` (FluoView v4.1.1), `h-pf` (HPF Framework v3.2.2), `vpa` (VPA v3.1.2)

---

## Diagram Files

| # | File | Diagram Type | What it shows |
|---|------|-------------|---------------|
| 01 | [01_architecture.md](01_architecture.md) | Architecture | 6-layer system view: User → App → Subsystems → HPF → Win32 Native → Hardware |
| 02 | [02_component.md](02_component.md) | Component | All OSGi bundles across fv/h-pf/vpa with dependency arrows |
| 03 | [03_sequence.md](03_sequence.md) | Sequence (×3) | Image Acquisition · App Startup · Analysis Pipeline |
| 04 | [04_class.md](04_class.md) | Class | Core domain models: bootstrap, device config, LSM image, analysis, licensing |
| 05 | [05_extension_points.md](05_extension_points.md) | Component + Sequence | OSGi extension point definitions, loading mechanism, priority ordering rules |
| 06 | [06_state_machine.md](06_state_machine.md) | State Machine (×3) | Acquisition FSM · Macro FSM · StateID reference · Thread safety |
| 07 | [07_matl_protocol.md](07_matl_protocol.md) | State Machine + Sequence | MATL 70-state engine · Protocol event flow · Sequence protocol structure |
| 08 | [08_data_formats.md](08_data_formats.md) | Class + Flowchart | File format hierarchy (IDA/OIF/OIR/HDF5) · Factory pattern · I/O pipeline |
| 09 | [09_error_messaging.md](09_error_messaging.md) | Flowchart + Sequence | Error dispatch · HPFException · Active Object pattern · Messaging architecture |
| 10 | [10_build_deployment.md](10_build_deployment.md) | Flowchart | Maven/Tycho build · P2 repos · Product definition · Windows runtime layout |
| 11 | [11_class_expanded.md](11_class_expanded.md) | Class (×5) | Full domain: Bootstrap · Services · Device/Acq Models · Image EMF · Analysis · Protocol |

---

## Key Architectural Facts (quick reference)

- **HPFRoot** is the singleton bootstrap — everything flows from `HPFRoot.getInstance()`
- **Extension points** (not OSGi services) are the primary extensibility mechanism — `plugin.xml` + `*.exsd`
- **Priority ordering is inconsistent**: `ApplicationRootHandler` = DESCENDING (100 > 50), `LaserModelProvider` = ASCENDING (1 > 10)
- **State transitions are guarded** by `synchronized(semaphore)` in `StateMachine.java`
- **All async dispatch** goes through `EventProviderImpl` (Active Object: blocking queue + worker thread)
- **Win32 native DLLs** are mandatory — no Linux/Mac. Fragments: `*.win32.win32.x86_64`
- **MATL engine** has 70 states and 70+ event types — most complex subsystem
- **EMF/ECORE** models drive device configuration (`camera.ecore`, `configuration.ecore`)
- **H-PF must be built before FluoView** — consumed as a local P2 repository
- **VPA is FluoView + volume specialisation** — shares most protocols and data models
