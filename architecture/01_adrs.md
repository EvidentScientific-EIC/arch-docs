# Architecture Decision Records (ADRs)
**Product:** FluoView / HPF / VPA  
**Last Updated:** April 2026

---

## ADR-001: OSGi Extension Points as Primary Extensibility Mechanism

**Status:** Accepted  
**Date:** Pre-2010 (inferred from codebase history)

### Context
The platform needed a way for multiple device vendors and feature teams to contribute functionality without modifying core code. Two options existed: OSGi declarative services, or Eclipse extension points.

### Decision
Use Eclipse extension points (`plugin.xml` + `*.exsd` schemas) rather than OSGi declarative services.

### Rationale
- Extension points support richer metadata: priority ordering, prerequisite declarations, schema validation
- Eclipse RCP tooling (PDE) provides first-class support for extension point authoring and validation
- Extension points allow dependency ordering between contributors (critical for device initialization sequencing)
- OSGi declarative services were less mature at the time this architecture was established

### Consequences
- All extensibility requires `plugin.xml` contributions — no runtime registration
- Discovery happens at startup only — extensions cannot be added at runtime
- Extension loading code (`HPFExtensionRegistry`, `FVComponentLoader`) must be maintained separately from OSGi
- Testing requires mocking the Eclipse registry (`HPFExtensionRegistry` has test injection support)
- **Inconsistency introduced:** Priority ordering convention differs between `ApplicationRootHandler` (DESCENDING) and `LaserModelProvider` (ASCENDING) — never resolved

---

## ADR-002: Win32 Native DLLs for Hardware and System Services

**Status:** Accepted  
**Date:** Pre-2010

### Context
Core platform capabilities — camera I/O, device communication, image processing, logging, and license validation — needed to interface with Olympus proprietary hardware and IP.

### Decision
Implement hardware-critical subsystems as Win32 native DLLs, exposed to Java via OSGi fragment bundles (`*.win32.win32.x86_64`).

### Rationale
- Hardware SDKs (camera, communication protocols) were available only as Win32 libraries
- Image processing performance requirements exceeded what pure Java could deliver at the time
- License protection required native-level obfuscation
- The installed base of microscope hardware was Windows-only

### Consequences
- **Platform lock-in:** The application cannot run on Linux or macOS. Any cross-platform work would require full replacement of the native layer
- **Testing limitation:** Native-dependent subsystems (CIF, COMM, image processing) cannot be unit-tested without physical hardware
- **Build complexity:** Win32 fragments must be compiled separately and are not part of the Maven/Tycho build
- **Maintenance risk:** Native DLL binaries are opaque — source may not be available to Bangalore team
- **Action item:** Confirm with Japan team whether native DLL source code is in the repository or separately managed

---

## ADR-003: EMF/ECORE for Device Configuration Models

**Status:** Accepted  
**Date:** Pre-2015 (inferred)

### Context
Device configuration (camera parameters, laser settings, device capabilities) needed a structured, versioned data model that could be serialized, validated, and tooled.

### Decision
Use Eclipse Modeling Framework (EMF) with ECORE schemas to define device configuration models. Code is generated from `.ecore` files.

### Rationale
- EMF provides free serialization (XMI/XML), validation, and notification infrastructure
- Eclipse tooling supports visual ECORE model editing
- Generated code ensures type safety across configuration access
- Model changes propagate automatically through generated adapters and listeners

### Consequences
- **Code generation dependency:** Model classes in `hpf.device.model` are generated — do not edit generated files directly. Edit `.ecore` → regenerate
- **Learning curve:** Developers unfamiliar with EMF find the generated patterns (factories, adapters, switches) confusing
- **Limited use:** Only device configuration uses EMF; image models use plain Java. This inconsistency is a source of confusion
- **Regeneration process:** After ECORE changes, right-click `.genmodel` → Generate Model Code in Eclipse

---

## ADR-004: HPFRoot Singleton Bootstrap

**Status:** Accepted  
**Date:** Pre-2010

### Context
The platform requires components and services to be initialized in dependency order at startup, with a well-known access point for all services.

### Decision
Use a singleton (`HPFRoot`) as the single bootstrap and service registry entry point.

### Rationale
- Single entry point simplifies service location and startup sequencing
- Dependency-ordered initialization prevents services from being used before their prerequisites are ready
- Singleton avoids injection complexity for a platform that predates modern DI frameworks

### Consequences
- **Testability:** `HPFRoot` singleton makes unit testing harder — tests must mock the registry or use `HPFExtensionRegistry`'s test injection mode
- **Global state:** All services accessible globally via domain lookups — no encapsulation boundary
- **Thread safety:** Initialization assumes single-threaded startup; `ParallelComponentManager` manages parallel initialization with care

---

## ADR-005: Separation of FluoView, HPF, and VPA into Three Projects

**Status:** Accepted  
**Date:** Pre-2015

### Context
Multiple product lines (standard LSM and volumetric/VPA) needed to share infrastructure without coupling.

### Decision
Split into three Maven/Tycho projects: `h-pf` (generic framework), `fv` (LSM product), `vpa` (volume product).

### Rationale
- `h-pf` can be versioned and released independently — other products consume it as a P2 repository
- Enables different release cadences per product
- Enforces architectural discipline: FluoView-specific code cannot accidentally go into HPF

### Consequences
- **Build order dependency:** `h-pf` must always be built before `fv` and `vpa`
- **P2 dependency on network share:** The H-PF P2 repository is hosted on an internal Olympus file server — not accessible outside the Japan network. **This is a critical blocker for Bangalore.**
- **SNAPSHOT versions:** `h-pf` at `3.2.2-SNAPSHOT` means builds are not reproducible without a fixed release. Should be released to a proper artifact repository.

---

## ADR-006: EventProviderImpl (Active Object) for Async Dispatch

**Status:** Accepted  
**Date:** Pre-2015

### Context
Error events, protocol events, and device events needed to be dispatched to multiple listeners without blocking the emitting thread.

### Decision
Implement async dispatch via `EventProviderImpl` — an Active Object pattern using a `BlockingQueue` and a dedicated worker thread.

### Rationale
- Emitter is never blocked by slow listeners
- Sequential dispatch preserves event ordering
- Simple to reason about — one queue, one thread per dispatcher
- Business listeners notify before UI listener (hardcoded ordering via `replaceUiListener`)

### Consequences
- **Error isolation:** An exception in one listener does not prevent others from receiving the event (if properly implemented)
- **Latency:** Events are not processed synchronously — there is a small delay between emit and listener notification
- **Shutdown:** `dispose()` must be called to terminate the worker thread — missed calls cause thread leaks
