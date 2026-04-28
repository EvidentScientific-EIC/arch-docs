# FluoView / HPF / VPA — Architecture & Documentation Index
**Product:** Olympus FluoView (LSM) / HPF Framework / VPA (Volumetric)  
**Last Updated:** April 2026  
**Maintained by:** Bangalore Team

---

## Quick Reference

| Fact | Value |
|---|---|
| Java version | 21 |
| Build system | Maven + Tycho 4.0.7 |
| UI framework | Eclipse RCP 4.x (E4) |
| OSGi runtime | Eclipse Equinox |
| FluoView version | 4.1.1 |
| HPF version | 3.2.2-SNAPSHOT |
| VPA version | 3.1.2 |
| Platform | Windows only (Win32 native DLL dependency) |
| JVM heap | 1228 MB (Xms + Xmx) |
| Codebase size | ~19,116 Java files, ~150+ OSGi plugins |
| Test plugins | ~99 across all projects |

> **CRITICAL for Bangalore:** The H-PF P2 repository is on an internal Olympus network share (`\\microscope\pj_confidential\...`). VPN to Japan is required to build. See TD-073 and the Build Runbook.

### Key Architectural Facts

- **HPFRoot** is the singleton bootstrap — everything flows from `HPFRoot.getInstance()`
- **Extension points** (not OSGi services) are the primary extensibility mechanism — `plugin.xml` + `*.exsd`
- **Priority ordering is inconsistent**: `ApplicationRootHandler` = DESCENDING (100 runs first), `LaserModelProvider` = ASCENDING (1 runs first)
- **State transitions are guarded** by `synchronized(semaphore)` in `StateMachine.java`
- **All async dispatch** goes through `EventProviderImpl` (Active Object: blocking queue + worker thread)
- **Win32 native DLLs** are mandatory — no Linux/Mac; fragments: `*.win32.win32.x86_64`
- **MATL engine** has 70 states and 70+ event types — most complex subsystem in the codebase
- **EMF/ECORE** models drive device configuration (`camera.ecore`, `configuration.ecore`) — never edit generated code directly
- **H-PF must be built before FluoView** — consumed as a local P2 repository
- **VPA is FluoView + volume specialisation** — shares most protocols and data models

---

## Diagrams

All diagrams are in Mermaid format. Render in VS Code (Mermaid extension), GitHub, or [mermaid.live](https://mermaid.live).

| File | Contents |
|---|---|
| [diagrams/01_architecture.md](diagrams/01_architecture.md) | 6-layer system view: User → App → Subsystems → HPF → Win32 Native → Hardware |
| [diagrams/02_component.md](diagrams/02_component.md) | All OSGi bundles across fv/h-pf/vpa with dependency arrows |
| [diagrams/03_sequence.md](diagrams/03_sequence.md) | Image acquisition · App startup · Analysis pipeline (3 sequence diagrams) |
| [diagrams/04_class.md](diagrams/04_class.md) | Core domain models: bootstrap, device config, LSM image, analysis, licensing |
| [diagrams/05_extension_points.md](diagrams/05_extension_points.md) | Extension point definitions, loading mechanism, priority ordering rules |
| [diagrams/06_state_machine.md](diagrams/06_state_machine.md) | Acquisition FSM · Macro FSM · StateID reference · Thread safety |
| [diagrams/07_matl_protocol.md](diagrams/07_matl_protocol.md) | MATL 70-state engine · Protocol event flow · Sequence protocol structure |
| [diagrams/08_data_formats.md](diagrams/08_data_formats.md) | File format hierarchy (IDA/OIF/OIR/HDF5) · Factory pattern · I/O pipeline |
| [diagrams/09_error_messaging.md](diagrams/09_error_messaging.md) | Error dispatch · HPFException · Active Object pattern · Messaging architecture |
| [diagrams/10_build_deployment.md](diagrams/10_build_deployment.md) | Maven/Tycho build · P2 repos · Product definition · Windows runtime layout |
| [diagrams/11_class_expanded.md](diagrams/11_class_expanded.md) | Full domain: Bootstrap · Services · Device/Acq Models · Image EMF · Analysis · Protocol |

---

## Onboarding

Start here if you are new to the team or product.

| File | Contents |
|---|---|
| [onboarding/01_developer_guide.md](onboarding/01_developer_guide.md) | Product overview, architecture, key concepts, code navigation |
| [onboarding/02_environment_setup.md](onboarding/02_environment_setup.md) | Windows prerequisites, JDK 21, Maven, Eclipse 2023-09, network checklist |
| [onboarding/03_build_runbook.md](onboarding/03_build_runbook.md) | Build order (h-pf → fv → vpa), commands, common errors and fixes |
| [onboarding/04_faq.md](onboarding/04_faq.md) | General, build, architecture, and coding FAQs; Japanese comment translations |

---

## Reference

Deep-reference material for day-to-day development.

| File | Contents |
|---|---|
| [reference/01_domain_glossary.md](reference/01_domain_glossary.md) | 50+ terms: microscopy hardware, acquisition, analysis, file formats, software |
| [reference/02_extension_point_catalogue.md](reference/02_extension_point_catalogue.md) | All extension points: IDs, interfaces, attributes, examples, priority table |
| [reference/03_interface_service_catalogue.md](reference/03_interface_service_catalogue.md) | 22 services with method signatures and pre/post conditions |
| [reference/04_feature_bundle_map.md](reference/04_feature_bundle_map.md) | Feature → plugin mapping, license-gating, test coverage heat map |
| [reference/05_dependency_version_matrix.md](reference/05_dependency_version_matrix.md) | Product versions, toolchain, P2 repos, native fragments, JVM config |

---

## How-To Guides

Step-by-step guides for common development tasks.

| File | Contents |
|---|---|
| [how-to/01_add_new_device.md](how-to/01_add_new_device.md) | 8-step guide: device model, driver, service, extension point registration |
| [how-to/02_add_new_file_format.md](how-to/02_add_new_file_format.md) | Adapter, factory, index factory, info util, extension point registration |
| [how-to/03_add_new_analysis_module.md](how-to/03_add_new_analysis_module.md) | Module, CommandHandler, CommandFactoryProvider with code examples |
| [how-to/04_add_new_acquisition_handler.md](how-to/04_add_new_acquisition_handler.md) | ApplicationRootHandler, Component/Service registration, state patterns |
| [how-to/05_function_enabler_feature.md](how-to/05_function_enabler_feature.md) | End-to-end pattern: `.condition` rule file + publisher + subscriber for any UI feature gated by `FunctionEnablerService` |
| [how-to/06_ui_framework_patterns.md](how-to/06_ui_framework_patterns.md) | SWT/CoolBar/Player composite gotchas + MATL composite model — distilled from US-62077 |

---

## Architecture Decisions

Rationale for major architectural choices made during platform design.

| File | Contents |
|---|---|
| [architecture/01_adrs.md](architecture/01_adrs.md) | 6 ADRs: Extension Points, Win32 Native, EMF/ECORE, HPFRoot Singleton, 3-project split, EventProviderImpl |
| [architecture/02_technical_debt_register.md](architecture/02_technical_debt_register.md) | **Single canonical tech debt source** — 70+ items in 11 categories, with type / risk / priority master index, plus per-category detail and delivery-cohort remediation plan |

---

## Testing

Test coverage, gaps, and strategy.

| File | Contents |
|---|---|
| [testing/01_test_coverage_map.md](testing/01_test_coverage_map.md) | ~99 test plugins mapped; critical gaps (TRU-AI, RPC, streaming, VPA); test patterns |
| [testing/02_test_plan.md](testing/02_test_plan.md) | Three-tier strategy (unit / OSGi integration / HIL); templates; CI pipeline target state |

---

## Code Quality

Static analysis findings and broader quality assessment.

| File | Contents |
|---|---|
| [quality/01_static_analysis_report.md](quality/01_static_analysis_report.md) | God classes, error handling, thread safety, resource leaks, raw types, null risks |
| [quality/02_code_quality_report.md](quality/02_code_quality_report.md) | API design, OSGi hygiene, testability, logging, Java 21 adherence, build reproducibility |

---

## Document Count

| Folder | Files |
|---|---|
| `diagrams/` | 11 |
| `onboarding/` | 4 |
| `reference/` | 5 |
| `how-to/` | 6 |
| `architecture/` | 2 |
| `testing/` | 2 |
| `quality/` | 2 |
| Root | 2 (INDEX.md, README.md) |
| **Total** | **34** |

---

## Maintenance Notes

- Diagrams are hand-authored from codebase analysis — update when architecture changes
- ADRs are append-only — never edit an accepted ADR; add a new superseding ADR instead
- Technical Debt Register should be reviewed at the start of each sprint
- Test Coverage Map should be updated when new test plugins are added
- This index should be updated whenever a new document is added to any folder
