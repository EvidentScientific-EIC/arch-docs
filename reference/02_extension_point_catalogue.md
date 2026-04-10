# Extension Point Catalogue
**Product:** FluoView / HPF / VPA  
**Last Updated:** April 2026

> Extension points are the primary extensibility mechanism. To add new behaviour, contribute to an existing extension point or define a new one. Never modify existing plugin source code for cross-cutting concerns.

---

## How to Read This Catalogue

Each entry shows:
- **ID** — the fully-qualified extension point ID used in `plugin.xml`
- **Defined in** — which plugin owns the schema
- **Interface to implement** — Java interface/class your contribution must extend
- **Key attributes** — what your `plugin.xml` entry must declare
- **Known contributors** — plugins already using this extension point

---

## HPF Core Extension Points (`jp.co.olympus.hpf.core`)

### `jp.co.olympus.hpf.core.components`
Register a component (logical service container) with the framework.

| Field | Value |
|---|---|
| Defined in | `hpf.core` |
| Base class | `FVComponentDomain` |
| Key attributes | `id` (unique string), `name`, `domain` (class), `prereqs` (comma-separated IDs) |
| Ordering | Topological sort by `prereqs` graph |

```xml
<extension point="jp.co.olympus.hpf.core.components">
  <component id="my.component.id"
             name="My Component"
             domain="com.example.MyComponentDomain"
             prereqs="jp.co.olympus.hpf.core.component.message"/>
</extension>
```

---

### `jp.co.olympus.hpf.core.services`
Register a service within a component.

| Field | Value |
|---|---|
| Defined in | `hpf.core` |
| Base class | `FVComponentService` |
| Key attributes | `id`, `class`, `componentId`, `prereqs`, `isPrereqOf` |
| Ordering | Topological sort by `prereqs`/`isPrereqOf` within component |

---

### `jp.co.olympus.hpf.core.handler`
Register an application lifecycle handler executed at startup.

| Field | Value |
|---|---|
| Defined in | `hpf.core` |
| Interface | `ApplicationRootHandler` |
| Key attributes | `class`, `priority` (integer) |
| Ordering | **DESCENDING** — higher integer runs first |
| Known contributors | `AcquisitionApplicationHandler` (priority=100), `DeviceApplicationHandler` (priority=50) |

```xml
<extension point="jp.co.olympus.hpf.core.handler">
  <handler class="com.example.MyHandler" priority="75"/>
</extension>
```

---

### `jp.co.olympus.hpf.core.serviceObserver`
Observe service state changes across components.

| Field | Value |
|---|---|
| Defined in | `hpf.core` |
| Base class | `ServiceObserverAdapter` |
| Key attributes | `id`, `class`, `prereqs` |

---

## HPF Device Extension Points (`jp.co.olympus.hpf.device`)

### `jp.co.olympus.hpf.device.deviceController`
Register an external process (executable) that controls a device.

| Field | Value |
|---|---|
| Key attributes | `name`, `path`, `process` (executable), `plugin`, `paramsGenerator` (class: `ParametersGenerator`) |

---

### `jp.co.olympus.hpf.device.commandObjectGeneratorFactory`
Register a factory that creates device command objects.

| Field | Value |
|---|---|
| Elements | `commandObjectGeneratorFactory` (class: `CommandObjectGeneratorFactory`), `deviceInfoGeneratorFactory` (class: `DeviceInformationGeneratorFactory`) |

---

### `jp.co.olympus.hpf.device.defaultCommandTargetFilter`
Filter which device receives a command (e.g., route laser commands only to the laser device).

| Field | Value |
|---|---|
| Interface | `CommandTargetFilter` |
| Known contributors | `MicroscopeLightPathDefaultCommandTargetFilter` (in `fluoview.acquisition`) |

---

### `jp.co.olympus.hpf.device.initializationExtender`
Extend device initialization sequence.

| Field | Value |
|---|---|
| Interface | (defined in schema) |

---

### `jp.co.olympus.hpf.device.lsmLightpathIndexProvider`
Provide lightpath index for LSM optical path management.

| Field | Value |
|---|---|
| Key attributes | `class`, `target` |

---

## HPF Data Extension Points (`jp.co.olympus.hpf.data`)

### `jp.co.olympus.hpf.data.FileFormat`
Register a new image file format handler. **Use this to add new format support.**

| Field | Value |
|---|---|
| Defined in | `hpf.data` |
| Key attributes | `type` (format identifier string), `PersistentAdapterFactory` (class), `AbstractSaveFrameIndexListFactory` (class), `InfomationUtil` (class) |
| Known contributors | IDA (`.vsi`), IDA2 (`.ida`), OIR (`.oir`), POIR (`.poir`), OMP2INFO (`.omp2info`), HDF5 (`.hdf5`), OIF (`.oif`), OIB (`.oib`) |

```xml
<extension point="jp.co.olympus.hpf.data.FileFormat">
  <FileFormat type="myformat"
              PersistentAdapterFactory="com.example.MyFormatAdapterFactory"
              AbstractSaveFrameIndexListFactory="com.example.MyIndexFactory"
              InfomationUtil="com.example.MyInfoUtil"/>
</extension>
```

---

### `jp.co.olympus.hpf.data.propertyExtractor`
Extract metadata properties from image files.

---

### `jp.co.olympus.hpf.data.fileReaderSelector`
Select the appropriate reader for a given file.

---

### `jp.co.olympus.hpf.data.imagingChecker`
Validate imaging data before use.

---

## HPF Acquisition Extension Points (`jp.co.olympus.hpf.acquisition`)

### `jp.co.olympus.hpf.acquisition.laserModelProvider`
Provide laser model configuration.

| Field | Value |
|---|---|
| Interface | `LaserModelProvider` |
| Key attributes | `class`, `priority` (integer) |
| Ordering | **ASCENDING** — lower integer runs first (opposite of handler!) |
| Known contributors | `LaserModelProviderImpl` (priority=10) in `fluoview.acquisition` |

---

## FluoView Acquisition Extension Points (`jp.co.olympus.fluoview.acquisition`) — 26 total

### `jp.co.olympus.fluoview.acquisition.applyParametersCommandGenerator`
Generate device commands from observation parameters. Called during acquisition setup.

| Field | Value |
|---|---|
| Key classes | `LSMParametersCommandGenerator`, `BrightnessAdjustmentParamCommandGenerator` |

---

### `jp.co.olympus.fluoview.acquisition.loadParametersExtender`
Enrich `ObservationSettings` with device-specific parameters before applying to hardware.

| Field | Value |
|---|---|
| Interface | `LoadParametersExtender` |

---

### `jp.co.olympus.fluoview.acquisition.deviceModelSetter`
Apply enriched parameters to the `DeviceConfiguration` EMF model.

---

### `jp.co.olympus.fluoview.acquisition.laserParamFactory`
Create laser parameter objects for the current device.

| Field | Value |
|---|---|
| Interface | `LaserParamFactory` |

---

### `jp.co.olympus.fluoview.acquisition.localAcquisitionHandler`
Register a custom acquisition data source (e.g., virtual/dummy, camera).

---

### `jp.co.olympus.fluoview.acquisition.activePhaseCommandGenerator`
Generate commands for the active acquisition phase.

| Field | Value |
|---|---|
| Interface | `LSMPhaseCommandGenerateExtender` |

---

### `jp.co.olympus.fluoview.acquisition.lsmLightpathExtender`
Extend LSM light path configuration.

---

### `jp.co.olympus.fluoview.acquisition.autoZoomExtender`
Extend auto-zoom behaviour.

---

### `jp.co.olympus.fluoview.acquisition.autoPinholeExtender`
Extend auto-pinhole adjustment.

---

### `jp.co.olympus.fluoview.acquisition.zdcSettingsExtender`
Extend Z-drift compensation settings.

---

### `jp.co.olympus.fluoview.acquisition.phasePropertiesInitializer`
Initialize phase-specific acquisition properties.

---

## FluoView Analysis Extension Points (`jp.co.olympus.fluoview.analysis`)

### `jp.co.olympus.fluoview.analysis.CommandFactoryProvider`
Provide analysis command factories for specific module types.

| Field | Value |
|---|---|
| Base class | `AbstractCommandFactoryProvider` |

---

### `jp.co.olympus.fluoview.analysis.AnalysisCommandHandlerProvider`
Handle a specific type of analysis module command.

| Field | Value |
|---|---|
| Interface | `AnalysisCommandHandler` |

---

### `jp.co.olympus.fluoview.analysis.AnalysisEventProvider`
Provide analysis lifecycle events.

---

## Priority Ordering Reference

| Extension Point | Order | Rule |
|---|---|---|
| `hpf.core.handler` | DESCENDING | Higher integer = runs first |
| `hpf.acquisition.laserModelProvider` | ASCENDING | Lower integer = runs first |
| `hpf.core.components` | Topological | Based on `prereqs` graph |
| `hpf.core.services` | Topological | Based on `prereqs`/`isPrereqOf` |
| `hpf.core.serviceObserver` | Topological | Based on `prereqs` |

> **Warning:** The inconsistency between handler (DESCENDING) and laserModelProvider (ASCENDING) is a known tech debt item. Always check the `*Loader` class for the specific ordering before registering a new contributor.
