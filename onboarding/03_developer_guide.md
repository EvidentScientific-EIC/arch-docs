# Developer Guide — FluoView / HPF / VPA
**Last Updated:** April 2026

---

## What Is This Product?

FluoView is Olympus's **laser scanning microscope (LSM) control and analysis platform**. It controls physical microscope hardware (lasers, cameras, stages, scanners), acquires fluorescence images, and provides analysis tools (FRAP, FRET, RRCM, TRU-AI).

Three codebases work together:

| Project | Role | Think of it as |
|---|---|---|
| `h-pf` (HPF) | Generic microscopy framework | The "Spring Boot" of microscopy platforms |
| `fv` (FluoView) | LSM-specific application | The product built on HPF |
| `vpa` (VPA) | Volume/3D imaging variant | FluoView + volumetric extensions |

---

## Architecture in 60 Seconds

```
User (Lab Operator)
  ↓
FluoView / VPA Application  (Eclipse RCP 4.x, SWT UI)
  ↓
HPF Framework               (OSGi services, extension points)
  ↓
Win32 Native DLLs           (CIF, COMM, image processing, logging, licensing)
  ↓
Hardware                    (LSM microscope, lasers, stages, cameras)
```

**Key design decisions:**
- **OSGi plugin architecture** — ~150 bundles, loosely coupled via extension points
- **Extension points** (not OSGi services) are the primary extensibility mechanism — everything goes through `plugin.xml`
- **EMF/ECORE models** define device configuration and image metadata schemas
- **Windows-only** — native DLLs cannot be ported without significant effort
- **HPFRoot** is the singleton bootstrap — all services flow from it

---

## Key Concepts Every Developer Must Know

### 1. Extension Points (Most Important)
This codebase extends behaviour via Eclipse extension points, NOT subclassing or OSGi services.

```xml
<!-- In plugin.xml: DEFINE an extension point -->
<extension-point id="applyParametersCommandGenerator"
                 schema="schema/applyParametersCommandGenerator.exsd"/>

<!-- In plugin.xml: CONTRIBUTE to an extension point -->
<extension point="jp.co.olympus.fluoview.acquisition.applyParametersCommandGenerator">
  <applyParametersCommandGenerator class="com.example.MyCommandGenerator"/>
</extension>
```

Extension points are discovered at startup via `HPFExtensionRegistry` → `Platform.getExtensionRegistry()`.

See `reference/02_extension_point_catalogue.md` for the full list.

### 2. HPFRoot — The Bootstrap
```java
HPFRoot root = HPFRoot.getInstance();
```
Everything starts here. Components and services are loaded in dependency order via extension points. Never instantiate services directly — always get them from the component domain.

### 3. Component → Service → Consumer Pattern
```
FVComponentDomain (component)
  └─ FVComponentService (service within the component)
       └─ consumed by handlers, UI, other services
```

### 4. ApplicationRootHandler — Your Entry Point
To run code at startup, implement `ApplicationRootHandler` and register it:
```xml
<extension point="jp.co.olympus.hpf.core.handler">
  <handler class="com.example.MyHandler" priority="50"/>
</extension>
```
Higher priority integer = runs first. `AcquisitionApplicationHandler` uses priority 100.

### 5. FunctionEnablerService — Feature Gating
All UI enable/disable states are controlled by `FunctionEnablerService`. Never enable/disable UI directly. Instead, activate/deactivate a `StateID`:
```java
functionEnablerService.activate("S_PF_LSM_IMAGING_IDLING");
functionEnablerService.addFunctionConditionChangedListener(functionId, listener);
```

### 6. ErrorDispatchService — Never Use printStackTrace()
```java
// Wrong
} catch (HPFException e) { e.printStackTrace(); }

// Right
} catch (HPFException e) {
    errorDispatchService.putError(new ErrorEvent(errId, e.getMessage(), e));
}
```

### 7. State Machine — Acquisition Flow
Acquisition has a formal FSM in `StateMachine.java`. State transitions are guarded by `synchronized(semaphore)`. All transitions go through `doTransition(fromState, toState)`. Never modify state directly.

---

## Finding Your Way Around the Code

| "I want to find..." | Look here |
|---|---|
| How acquisition starts | `AcquisitionApplicationHandler.onPostInitialization()` |
| How device commands are built | `ApplyParametersCommandCoordinator` + `*CommandGenerator` classes |
| How images are saved | `DataManagementService` + format adapters in `hpf.data.format.*` |
| How analysis runs | `AnalysisSequenceManagerService` + `AnalysisCommandHandler` implementations |
| How MATL works | `MATLManagerEngine` (70-state FSM in `hpf.protocol.matl.core`) |
| How a feature is enabled/disabled | Search for `StateID` in `vpa.core` or `fluoview.mode` packages |
| How extension points are loaded | `FVComponentLoader`, `ApplicationRootHandlerLoader` in `hpf.core` |
| How errors propagate | `ErrorDispatchService` → `EventProviderImpl` (async) → listeners |

---

## Coding Conventions

### Naming
- Package prefix: `jp.co.olympus.{project}.{subsystem}` — never deviate
- Interfaces: no `I` prefix (use `AcquisitionService` not `IAcquisitionService`)
- Implementations: suffix with `Impl` (e.g., `AcquisitionServiceImpl`)
- Handlers: suffix with `Handler` or `ApplicationHandler`

### Extension Points
- Define schema in `schema/*.exsd`
- Register in `plugin.xml`
- Load via `HPFExtensionRegistry.INSTANCE.getConfigurationElementsFor(pointId)`
- Never hardcode class names — always use `element.createExecutableExtension("class")`

### Error Handling
- Use `HPFException(HPFError, ...)` for all domain errors
- Dispatch via `ErrorDispatchService` — never use `System.out` or `printStackTrace()`
- Catch `HPFException` in state handlers, transition to safe state, then dispatch

### Threading
- Never update SWT UI from a non-UI thread — use `Display.asyncExec()`
- All async dispatch goes through `EventProviderImpl` (Active Object)
- State machine transitions are guarded — always use `doTransition()`
- Re-interrupt thread when catching `InterruptedException`:
  ```java
  } catch (InterruptedException e) {
      Thread.currentThread().interrupt();
      // handle
  }
  ```

---

## Development Workflow

1. **Identify the extension point** for your feature (see `reference/02_extension_point_catalogue.md`)
2. **Create a new plugin** following existing naming: `jp.co.olympus.{project}.{feature}`
3. **Implement the required interface**
4. **Register in `plugin.xml`** under the correct extension point
5. **Add dependency** in `META-INF/MANIFEST.MF` under `Require-Bundle`
6. **Add to `pom.xml`** module list in the appropriate parent POM
7. **Write tests** in a companion `.test` plugin under `test/plugins/`
8. **Build** with `mvn clean install` from the project root
