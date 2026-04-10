# Build Runbook
**Product:** Olympus FluoView / HPF / VPA  
**Last Updated:** April 2026

---

## Build Order — Non-Negotiable

```
h-pf  →  fv  →  vpa
```

`fv` depends on `h-pf` via a local P2 repository. `h-pf` must be built and installed first, every time.

---

## Standard Build Commands

### Full Clean Build (first time or after dependency changes)
```bash
# 1. Build HPF framework
cd workspace/h-pf
mvn clean install -Dmaven.test.skip=true

# 2. Build FluoView
cd workspace/fv
mvn clean install -Dmaven.test.skip=true

# 3. Build VPA (if needed)
cd workspace/vpa
mvn clean install -Dmaven.test.skip=true
```

### Incremental Build (single plugin changed)
```bash
# Build only a specific plugin
cd workspace/fv/plugins/jp.co.olympus.fluoview.acquisition
mvn clean install -Dmaven.test.skip=true
```

### Run Tests
```bash
# Run all tests for h-pf
cd workspace/h-pf
mvn verify

# Run tests for a specific plugin
cd workspace/h-pf/test/plugins/jp.co.olympus.hpf.data.test
mvn verify
```

### Build with Specific Maven Local Repo
```bash
# Tycho pins local repo via .mvn/maven.config:
# -Dmaven.repo.local=${user.dir}/../.m2/repository
# Override if needed:
mvn clean install -Dmaven.repo.local=/path/to/.m2/repository
```

---

## Maven Configuration Files

| File | Purpose |
|---|---|
| `fv/.mvn/maven.config` | Sets `-Dtycho-version=4.0.7` and local repo path |
| `fv/.mvn/extensions.xml` | Loads `tycho-build` Maven extension |
| `fv/pom.xml` | Parent POM — Java 21, Eclipse 2023-09 P2 URL, H-PF P2 URL |
| `h-pf/plugins/pom.xml` | Aggregates ~100 HPF plugin modules |
| `h-pf/features/pom.xml` | Aggregates 39 HPF feature modules |
| `vpa/plugins/pom.xml` | Aggregates ~35 VPA plugin modules |

---

## Common Build Errors & Fixes

### Error: "Cannot connect to P2 repository"
```
Could not resolve artifact from: file:\\microscope\pj_confidential\...
```
**Fix:** VPN not connected, or P2 site unavailable. Options:
1. Connect VPN and retry
2. Use a local mirror — update the `<url>` in `fv/pom.xml` `<repositories>` section
3. Run Eclipse offline: `mvn clean install -o` (only works if previously resolved)

### Error: "Target platform not defined"
**Fix:** Open Eclipse → Window → Preferences → Plug-in Development → Target Platform → reload

### Error: "Java 21 features not found"
```
Source option 21 is not supported. Use --release 21 or higher.
```
**Fix:** Maven is picking up wrong JDK. Check `JAVA_HOME` and ensure Maven uses JDK 21:
```bash
mvn -version  # Should show Java version: 21
```

### Error: "Bundle jp.co.olympus.hpf.xxx cannot be resolved"
**Fix:** `h-pf` was not installed before building `fv`. Run `h-pf` build first.

### Error: "Tycho version mismatch"
**Fix:** Do not override `tycho-version`. It is pinned to `4.0.7` in `.mvn/maven.config`.

---

## Build Outputs

| Project | Output |
|---|---|
| `h-pf` | P2 repository at `h-pf/repository/target/repository/` |
| `fv` | Product at `fv/plugins/fv_meavn_product/target/products/` |
| `vpa` | Product at `vpa/products/.../target/products/` |

The built FluoView product is a self-contained Windows application directory containing:
- `FluoView.exe` (Eclipse launcher)
- `plugins/` (all OSGi bundles)
- `features/` (feature groupings)
- `configuration/` (Eclipse config)

---

## Platform Note
Only `win32.win32.x86_64` target environment is configured. Cross-platform builds will fail due to native fragment bundles:
- `jp.co.olympus.hpf.comm.win32.win32.x86_64`
- `jp.co.olympus.hpf.image.win32.win32.x86_64`
- `jp.co.olympus.hpf.log.win32.win32.x86_64`
- `jp.co.olympus.hpf.license.win32.win32.x86_64`
