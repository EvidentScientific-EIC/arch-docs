# Domain Glossary — FluoView / HPF / VPA
**Last Updated:** April 2026

---

## Microscopy & Hardware Terms

| Term | Full Name | Definition |
|---|---|---|
| **LSM** | Laser Scanning Microscopy | Primary imaging modality. A laser beam scans across a sample point-by-point; emitted fluorescence is detected per point. Produces high-resolution optical sections. |
| **PMT** | Photomultiplier Tube | Detector used in LSM systems. Converts photons to electrical signal. Parameters: voltage, gain, offset. Configured via `PMTParam` class. |
| **MPE** | Multi-Photon Excitation | Imaging technique using femtosecond pulsed lasers. Excites fluorophores with two or more photons simultaneously. Deeper tissue penetration than single-photon. Plugin: `fluoview.acquisition.mpelaser` |
| **Piezo** | Piezoelectric Stage | Motorized stage using piezoelectric actuators for nanometer-precision Z-positioning. Used for z-stack acquisition. Plugin: `fluoview.acquisition.piezo` |
| **Objective** | Objective Lens | The primary optical element closest to the sample. Characterized by magnification, NA, immersion type. Managed in `DeviceConfiguration`. |
| **Lightpath** | Optical Light Path | The complete path of light through the microscope: laser → sample → detectors. Configured via lightpath extension points (26+). |
| **Lambda (λ)** | Lambda/Spectral Scanning | Acquisition mode that scans across wavelengths. `LambdaChannel` represents a specific excitation/emission wavelength pair. |
| **Fluorochrome** | Fluorescent Probe/Dye | A molecule that absorbs light at one wavelength and emits at another. Each channel in multi-channel imaging uses a specific fluorochrome. |
| **Pinhole** | Confocal Pinhole | Aperture that rejects out-of-focus light in confocal microscopes. Size (in Airy units) affects optical sectioning. |
| **NA** | Numerical Aperture | Measure of an objective's light-gathering ability. Higher NA = better resolution. |
| **ZDC** | Z-Drift Compensation | Automatic correction of sample drift in the Z-axis during time-lapse imaging. Plugin: `fluoview.acquisition.ui` with `zdcSettingsExtender`. |
| **Turret** | Mirror/Objective Turret | Rotating holder for objectives or mirrors. Turret position selects the active optical element. |

---

## Acquisition & Protocol Terms

| Term | Full Name | Definition |
|---|---|---|
| **Acquisition** | Image Acquisition | The controlled process of capturing images from the microscope hardware. Managed by `AcquisitionService`. |
| **Repeat / Live** | Repeat Scanning | Continuous scanning loop — "live view". The microscope scans repeatedly, updating the display in real time. State: `S_PF_LSM_REPEAT_SCANNING`. |
| **Series** | Series Acquisition | A defined sequence of image captures (z-stack, time-lapse, multi-channel, or combination). Managed by `SequenceManagerImpl`. State: `S_PF_LSM_SERIES_SCANNING`. |
| **MATL** | Multi-Area Time-Lapse | Protocol for acquiring images from multiple physical stage positions (areas) over multiple time points. The most complex subsystem — 70 states, 70+ events. Plugin: `hpf.protocol.matl`. |
| **Protocol** | Acquisition Protocol | A defined script/procedure for automated experiments. Consists of Tasks organized in Cycles. Executed by `ProtocolExecutorImpl`. |
| **Cycle** | Acquisition Cycle | One complete iteration of a repeating acquisition pattern in a protocol. A time-lapse protocol has N cycles. |
| **Task** | Protocol Task | The fundamental unit of protocol execution. Has three phases: `TASK_BEGIN`, `TASK_MIDDLE`, `TASK_END`. |
| **Sequence** | Acquisition Sequence | An ordered set of acquisition steps, each potentially with different parameters (laser, channel, z-position). |
| **Phase** | Acquisition Phase | A sub-unit of a channel acquisition step. Tracks which part of an operation is executing (`LSMPhase`). |
| **ROI** | Region of Interest | A user-defined area within an image for targeted acquisition or analysis. Used heavily in analysis modules. |
| **ASAC** | Advanced Sequential Acquisition Control | FluoView-specific module for complex multi-step acquisition sequences. Plugin group: `fluoview.asac.*`. |

---

## Analysis Terms

| Term | Full Name | Definition |
|---|---|---|
| **FRAP** | Fluorescence Recovery After Photobleaching | Analysis technique measuring molecular mobility. A region is photobleached (laser-killed), then recovery of fluorescence over time is measured. Plugin: `hpf.image.frap`. |
| **FRET** | Förster Resonance Energy Transfer | Analysis technique measuring proximity of molecules (<10 nm). Energy transfers between donor and acceptor fluorophores. Plugin: `hpf.image.fret`. |
| **RRCM** | Resonant/Rotational Confocal Microscopy | High-speed confocal imaging using resonant scanners. Requires specialized analysis. Plugins: `fluoview.rrcm.*`. |
| **TRU-AI** | Trusted AI | Olympus neural-network-based image analysis and acquisition assistance. Requires native AI inference modules. Plugins: `fluoview.analysis.truai.*`. |
| **CellSens** | CellSens Integration | Integration with Olympus CellSens cell imaging and analysis platform. Provides cell segmentation in FluoView. Plugins: `fluoview.acquisition.cellsens`, `fluoview.analysis.cellsens`. |
| **RICS/FCS** | Raster Image Correlation Spectroscopy / Fluorescence Correlation Spectroscopy | Advanced analysis techniques for molecular diffusion. Plugin: `fluoview.analysis.ricsfcs`. |

---

## File Format Terms

| Term | Full Name | Extension | Description |
|---|---|---|---|
| **OIF** | Olympus Image File | `.oif` | Folder-based format. Metadata in INI-style text files, image data in companion TIFF files. Primary FluoView format. |
| **OIB** | Olympus Image Binary | `.oib` | ZIP archive of OIF contents. Single-file portable version of OIF. |
| **OIR** | Olympus Image Raw | `.oir` | Raw binary image format from newer Olympus systems. |
| **POIR** | Progressive OIR | `.poir` | Progressive/partial variant of OIR. |
| **IDA** | Image Data Archive | `.vsi` | Legacy Olympus format. XML metadata + binary blocks. Multi-level, multi-area support. |
| **IDA2** | IDA version 2 | `.ida` | Updated IDA format with forward compatibility. |
| **OMP2INFO** | Olympus Multi-area Package | `.omp2info` | Container for multi-area acquisitions. |
| **HDF5** | Hierarchical Data Format 5 | `.hdf5` | Multi-resolution pyramid format for large images. |
| **VSI** | Virtual Slide Image | `.vsi` | Compatible with IDA format in this codebase. |

---

## Software Architecture Terms

| Term | Definition |
|---|---|
| **HPF** | High Performance Framework — the generic microscopy platform (`h-pf` project). All products are built on top of it. |
| **VPA** | Volume Presentation Application — the 3D/volume imaging variant of FluoView (`vpa` project). |
| **OSGi** | Open Services Gateway initiative — the modular Java runtime used by Eclipse. Each plugin is an OSGi bundle. |
| **Extension Point** | Eclipse mechanism for defining a hook that other plugins can contribute to. Defined in `plugin.xml` with schema in `*.exsd`. |
| **HPFRoot** | Singleton bootstrap class in `hpf.core`. All services and components are initialized through it at startup. |
| **FVComponentDomain** | A logical grouping of related services (analogous to a DI container). Examples: `MessageDomain`, `DeviceDomain`. |
| **FVComponentService** | Base interface for all HPF services. Accessed via domain lookups, not direct instantiation. |
| **ApplicationRootHandler** | Interface for code that runs during application startup. Priority-ordered (DESCENDING). |
| **FunctionEnablerService** | State machine controlling which UI functions are enabled, based on acquisition state, license, and device status. |
| **StateID** | String identifier for a system state in `FunctionEnablerService`. Examples: `S_PF_LSM_IMAGING_IDLING`, `S_PF_LSM_SERIES_SCANNING`. |
| **MessageDomain** | Component domain providing `ErrorDispatchService` and legacy `MessageDeliveryService`. |
| **RiJEvent** | Base class for all MATL protocol engine events. Each event has a unique `long` ID. |
| **EventProviderImpl** | Active Object implementation for async event dispatch. Uses a blocking queue + worker thread. |
| **CIF** | Common Image Framework / Camera Interface Framework — native Win32 camera abstraction layer. |
| **EMF** | Eclipse Modeling Framework — used to define domain models via `.ecore` schema files. Code is generated from these. |
| **ECORE** | The metamodel format used by EMF. Files: `camera.ecore`, `configuration.ecore` in `hpf.device.model`. |
| **Tycho** | Maven plugin for building Eclipse RCP/OSGi applications. Version 4.0.7 is used. |
| **P2** | Eclipse's provisioning platform — package management system for Eclipse plugins/features. |
| **Feature** | Eclipse grouping of related plugins for installation/update purposes. Defined in `feature.xml`. |
| **Fragment** | An OSGi bundle that extends another bundle (the "host"). Used for platform-specific native code (e.g., `*.win32.win32.x86_64`). |
