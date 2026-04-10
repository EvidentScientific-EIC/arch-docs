# Dependency & Version Matrix
**Product:** FluoView / HPF / VPA  
**Last Updated:** April 2026

---

## Product Versions

| Product | Version | Maven GroupId |
|---|---|---|
| FluoView (fv) | 4.1.1 | `jp.co.olympus.fluoview` |
| HPF Framework (h-pf) | 3.2.2-SNAPSHOT | `jp.co.olympus.hpf` |
| VPA | 3.1.2 | `jp.co.olympus.vpa` |

---

## Build Toolchain

| Tool | Version | Configured In |
|---|---|---|
| Java / JDK | 21 (JavaSE-21) | `fv/pom.xml` `<maven.compiler.release>21</maven.compiler.release>` |
| Apache Maven | 3.9+ | System install |
| Tycho | 4.0.7 | `fv/.mvn/maven.config` `-Dtycho-version=4.0.7` |
| Tycho Build Extension | 4.0.7 | `fv/.mvn/extensions.xml` |
| Eclipse Target Platform | 2023-09 (4.29) | `fv/pom.xml` P2 repository URL |
| Target OS/WS/Arch | win32 / win32 / x86_64 | `target-platform-configuration` in parent POM |

---

## Platform Dependencies (P2 Repositories)

| Repository | URL | Contents |
|---|---|---|
| Eclipse 2023-09 | `file:\\microscope\pj_confidential\SSWG_L\hpf\eclipse202309updatesite\202309131000` | Eclipse platform, SWT, E4, EMF |
| H-PF Framework | `file:${basedir}/../../../../../hpf/Application` | HPF built P2 repository |

> **Bangalore Action Item:** Both P2 repositories must be mirrored locally or via proxy. The Eclipse update site is on an Olympus internal network share not accessible outside Japan.

---

## Eclipse / OSGi Platform Bundles (from Target Platform)

| Bundle | Version | Purpose |
|---|---|---|
| Eclipse Platform | 4.29 (2023-09) | Core RCP runtime |
| Eclipse E4 Core DI | 1.x | Dependency injection |
| Eclipse E4 Core Services | 1.x | E4 service layer |
| Eclipse E4 UI Workbench | 0.x | Workbench framework |
| SWT (Standard Widget Toolkit) | 3.x | UI toolkit |
| EMF Core | 2.x | Eclipse Modeling Framework |
| EMF Common | 2.x | EMF utilities |
| Jakarta Inject API | 2.x | JSR-330 DI annotations |

---

## Key HPF Native Dependencies (Win32 x86_64)

| Bundle | Fragment Host | Purpose |
|---|---|---|
| `hpf.comm.win32.win32.x86_64` | `hpf.comm` | Device communication native DLL |
| `hpf.image.win32.win32.x86_64` | `hpf.image` | Image processing native DLL |
| `hpf.log.win32.win32.x86_64` | `hpf.log` | Logging native DLL |
| `hpf.license.win32.win32.x86_64` | `hpf.license` | License validation native DLL |
| `hpf.cif.win32.win32.x86_64` | `hpf.cif` | Camera Interface Framework native DLL |
| `hpf.data.format.ida.win32.win32.x86_64` | `hpf.data.format.ida` | IDA format native support |
| `hpf.protocol.matl.acquisition.win32.win32.x86_64` | `hpf.protocol.matl.acquisition` | MATL native acquisition |
| `fluoview.comm.win32.win32.x86_64` | `fluoview.comm` | FluoView communication native |

---

## JVM Launch Configuration (from FluoView.product)

```
-Xms1228M                    Initial heap
-Xmx1228M                    Maximum heap
-XX:MetaspaceSize=192M       Initial metaspace
-XX:MaxDirectMemorySize=6144M  Direct/native memory (image buffers)
```

Workspace: `@user.home/AppData/Local/ESS/cellSens FV/workspace`

---

## Module Count Summary

| Project | Plugins | Features | Test Plugins |
|---|---|---|---|
| h-pf | ~100 | 39 | 61 |
| fv | ~100 | ~20 | 38 |
| vpa | ~35 | ~10 | 0 (uses testscenario) |
| **Total** | **~235** | **~69** | **99** |
