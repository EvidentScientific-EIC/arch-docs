# Feature-to-Bundle Map
**Product:** FluoView / HPF / VPA  
**Last Updated:** April 2026

---

| Feature | Primary Plugin(s) | License-Gated | Test Coverage | Notes |
|---|---|---|---|---|
| Live / Repeat Scanning | `fluoview.acquisition`, `hpf.acquisition`, `hpf.acquisition.camera` | No | Good (44 test classes) | Core feature — always available |
| Series Acquisition (Z-stack, time-lapse, multi-channel) | `fluoview.protocol.sequence`, `hpf.protocol.core` | No | Good (5–10 test classes) | |
| Multi-Area Time-Lapse (MATL) | `fluoview.protocol.matl.*`, `hpf.protocol.matl*` | Yes | Good (25+ test classes) | Most complex subsystem |
| FRAP Analysis | `hpf.image.frap`, `hpf.image.frap.win32.x86_64` | Yes | Good (4 test classes) | Requires Win32 native DLL |
| FRET Analysis | `hpf.image.fret`, `hpf.image.fret.win32.x86_64` | Yes | Good (4 test classes) | Requires Win32 native DLL |
| RRCM Analysis | `fluoview.rrcm.core`, `fluoview.rrcm.analysis.*`, `fluoview.rrcm.acquisition` | Yes | Good (core + analysis) | Live and post-process variants |
| TRU-AI Analysis | `fluoview.analysis.truai.*`, `fluoview.acquisition.aiassist` | Yes | **NONE** | Zero test coverage — high risk |
| CellSens Integration | `fluoview.acquisition.cellsens`, `fluoview.analysis.cellsens` | Yes | Minimal | External product integration |
| MPE Laser Control | `fluoview.device.mpelaser.*`, `fluoview.acquisition.mpelaser.*` | Yes (MATL variant) | Good (25+ test classes for MATL variant) | Femtosecond laser support |
| Piezo Stage Control | `fluoview.device.stage.*`, `fluoview.acquisition.piezo` | Yes | Good (10 test classes) | Z-axis precision positioning |
| Image Save — OIF/OIB | `fluoview.data.format.oif` | No | Good | Primary FluoView format |
| Image Save — IDA/VSI | `hpf.data.format.ida`, `hpf.data.format.ida2` | No | Good | Legacy and forward-compatible |
| Image Save — OIR (Raw) | `hpf.data.format.raw` | No | Good (6 test classes) | Raw format |
| Image Save — HDF5 | `hpf.data.format.multiresolution` | No | Excellent (49 test classes) | Multi-resolution pyramid |
| Multi-Area Export | `hpf.data.format.multiarea` | No | Good | OMP2INFO format |
| Image Export (movie/image) | `hpf.data.export` | No | Good | Delegates to `ExportService` |
| Macro / Scripting | `hpf.macro`, `hpf.macro.ui` | Yes | Minimal (7 test classes) | Python/Jython-based |
| RPC / External API | `hpf.rpc`, `hpf.rpc.engine.xmlrpc` | No | **NONE** | Used by CellSens, macro, external tools |
| Volume Rendering (VPA) | `vpa.ui.image.volume`, `hpf.ui.image.volume` | Yes | Good | VPA-only feature |
| MATL Volume (VPA) | `vpa.protocol.matl` | Yes | Good | VPA-specific MATL |
| Maintenance Mode | `hpf.maintenance.core`, `vpa.maintenance.ui` | Yes | **NONE** | Zero test coverage — high risk |
| License Management | `hpf.core.enabler`, `fluoview.license.*`, `hpf.license` (native) | N/A | Weak | Win32 native validation |
| Device Configuration (EMF) | `hpf.device.model` | No | **NONE** | EMF models — no dedicated tests |
| Camera Interface (CIF) | `hpf.cif`, `hpf.cif.win32.x86_64` | No | **NONE** | Win32 native — untestable without HW |
| Acquisition Streaming | `hpf.acquisition.stream` | No | **NONE** | No test plugin |
| RICS/FCS Analysis | `fluoview.analysis.ricsfcs` | Yes | Minimal | Advanced spectroscopy analysis |
| Info View | `hpf.infoview.model`, `hpf.infoview.ui` | No | Minimal | Metadata display panel |
| ASAC | `fluoview.asac.*` | Yes | Minimal | Advanced sequential acquisition; has known provisional code |
| AI-Assisted Acquisition | `fluoview.acquisition.aiassist` | Yes | **NONE** | AI parameter optimisation |

---

## License-Gated Feature Summary

Features gated via `FunctionEnablerService` and `*.license` stub plugins:

| Plugin Pattern | Controls |
|---|---|
| `fluoview.acquisition.mpelaser.matl.license` | MATL + MPE laser combined |
| `fluoview.protocol.matl.license` | MATL protocol |
| `fluoview.protocol.matl.microplate.acquisition.license` | Microplate MATL |
| `fluoview.acquisition.cellsens.license` | CellSens acquisition |
| `fluoview.analysis.cellsens.license` | CellSens analysis |
| `fluoview.rrcm.acquisition.license` | RRCM acquisition |
| `fluoview.rrcm.analysis.license` | RRCM analysis |
| `fluoview.ui.image.volume.license` | Volume rendering |
| `fluoview.acquisition.stage.license` | Stage control |

---

## Test Coverage Heat Map

```
Excellent (>40 tests): HDF5 format, HPF data, Acquisition commands
Good     (10–40):      MATL, FRAP, FRET, RRCM, MPE laser, Stage, OIR format
Minimal  (<10):        Macro, CellSens, RICS/FCS, ASAC, License
NONE:                  TRU-AI, Maintenance, RPC, CIF, Device model, Streaming, AI-Assist
```
