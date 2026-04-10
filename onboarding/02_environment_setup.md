# Environment Setup Checklist
**Product:** Olympus FluoView / HPF / VPA  
**Last Updated:** April 2026

---

## System Requirements

| Requirement | Specification | Notes |
|---|---|---|
| OS | Windows 10/11 x64 | Mandatory — Win32 native DLLs, no Linux/Mac |
| JDK | Java 21 (Temurin/Adoptium recommended) | Must be JDK, not JRE |
| Maven | 3.9+ | Tycho 4.0.7 is pinned in `.mvn/maven.config` |
| Eclipse IDE | 2023-09 RCP edition | Must be RCP/RAP edition for PDE support |
| RAM | 16 GB minimum | JVM configured for up to 7.5 GB heap + metaspace |
| Disk | 20 GB free | Maven local repo + build outputs |

---

## Step-by-Step Setup

### 1. Install JDK 21
```bat
# Verify after install
java -version
# Expected: openjdk version "21.x.x"

# Set JAVA_HOME
setx JAVA_HOME "C:\Program Files\Eclipse Adoptium\jdk-21.x.x.x-hotspot"
```

### 2. Install Maven
```bat
mvn -version
# Expected: Apache Maven 3.9.x ... Java version: 21
```

### 3. Install Eclipse 2023-09 RCP Edition
- Download "Eclipse IDE for RCP and RAP Developers" — **2023-09 release only**
- Do NOT use JEE, Java, or any other edition
- Set Eclipse JRE: Window → Preferences → Java → Installed JREs → Add JDK 21

### 4. Network Access (Bangalore team — critical)
- [ ] Request VPN access to Olympus internal network
- [ ] Verify access to P2 update site: `\\microscope\pj_confidential\SSWG_L\hpf\eclipse202309updatesite\202309131000`
- [ ] If unreachable: request a local mirror (zip archive) of this P2 site
- [ ] Update `fv/pom.xml` repository URL to point to local mirror if needed

### 5. Clone Repositories
```bash
# All three repos MUST be siblings in the same parent directory
mkdir workspace && cd workspace
git clone <fv-repo-url>    fv
git clone <hpf-repo-url>   h-pf
git clone <vpa-repo-url>   vpa
```

Expected layout:
```
workspace/
├── fv/       (FluoView v4.1.1)
├── h-pf/     (HPF Framework v3.2.2)
└── vpa/      (VPA v3.1.2)
```

> The path `../../../../../hpf/Application` is hardcoded in `fv/pom.xml` — do not rename folders.

### 6. Import into Eclipse
1. File → Import → Maven → Existing Maven Projects
2. Select `workspace/h-pf` root → import all
3. Repeat for `workspace/fv` and `workspace/vpa`
4. Window → Preferences → Plug-in Development → Target Platform → set to the `.target` file

### 7. Obtain License File
- Contact the Japan team for a development/evaluation license
- The license is validated by a native Win32 DLL — no workaround exists
- Place license file per instructions from Japan team

---

## Verification Checklist
- [ ] `java -version` shows Java 21
- [ ] `mvn -version` shows Maven 3.9+
- [ ] Eclipse shows no red markers in Package Explorer after import
- [ ] `h-pf` builds cleanly: `cd h-pf && mvn clean install -Dmaven.test.skip=true`
- [ ] `fv` builds cleanly: `cd fv && mvn clean install -Dmaven.test.skip=true`
- [ ] FluoView launches from Eclipse: Run → Run Configurations → Eclipse Application
- [ ] Application main window appears (license required)
