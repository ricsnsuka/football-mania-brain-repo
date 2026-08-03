# Incident Report — Java Version Mismatch

**Date:** 2026-05-26  
**Severity:** CRITICAL (Boot Failure)  
**Status:** RESOLVED  
**Reporter:** User (via IntelliJ IDEA Runtime Error)

---

## 🚨 Issue Summary

Application failed to start with `UnsupportedClassVersionError`:
```
java.lang.UnsupportedClassVersionError: pt/rics/demo/football/FootballApplication 
has been compiled by a more recent version of the Java Runtime (class file version 65.0), 
this version of the Java Runtime only recognizes class file versions up to 61.0
```

**Translation:**
- Application compiled with **Java 21** (class version 65.0) ✅
- IntelliJ IDEA attempting to run with **Java 17** (class version 61.0) ❌

---

## 🔍 Root Cause

IntelliJ IDEA was configured to use **Java 17** as the Project SDK / Run Configuration JRE, 
while the application requires **Java 21**.

**Why This Happened:**
- Gradle toolchain correctly configured for Java 21 (`build.gradle` line 17-19)
- System environment correctly configured (Java 21 installed, `JAVA_HOME` correct)
- IDE settings not synchronized with Gradle toolchain settings

---

## ✅ Resolution

### Verification Steps Performed:
1. ✅ Checked system Java version: `java -version` → **Java 21.0.11**
2. ✅ Checked `JAVA_HOME`: → `C:\Program Files\Eclipse Adoptium\jdk-21.0.11.10-hotspot\`
3. ✅ Verified Gradle builds correctly: `.\gradlew clean bootJar` → **SUCCESS**
4. ✅ Verified application runs via Gradle: `.\gradlew bootRun` → **SUCCESS** (Java 21)

### User Action Required:

**Configure IntelliJ IDEA to use Java 21:**

#### Step 1: Set Project SDK
1. Press `Ctrl + Alt + Shift + S` (Windows)
2. Navigate to **Project** → **SDK**
3. Select **21 (Eclipse Adoptium 21.0.11)**
4. Set **Language level** to **21**

#### Step 2: Set Run Configuration JRE
1. Top-right: **Edit Configurations...**
2. Select `FootballApplication`
3. Set **JRE** to **Project default (21)**
4. Apply

#### Step 3: Rebuild
- **Build** → **Rebuild Project**

---

## 📋 Incident Timeline

| Time     | Event                                      |
|----------|-------------------------------------------|
| 16:30    | User attempted to run via IntelliJ → Error |
| 16:32    | Orchestrator detected emergency            |
| 16:33    | Verified Java 21 on system                 |
| 16:33    | Verified Gradle build works                |
| 16:34    | Verified app runs via `bootRun`            |
| 16:35    | Root cause identified: IDE misconfiguration|
| 16:35    | Resolution documented                      |

---

## 🎓 Lessons Learned

1. **IntelliJ IDEA does not auto-sync Project SDK with Gradle Toolchain:**  
   Even though `build.gradle` specifies Java 21, the IDE may retain an older SDK setting.

2. **Class Version Reference:**
   - Java 17 = class version 61.0
   - Java 21 = class version 65.0

3. **Quick Diagnostic for Future:**
   ```powershell
   # System Java
   java -version
   
   # Gradle build test
   .\gradlew clean bootJar
   
   # Gradle runtime test
   .\gradlew bootRun --args='--spring.profiles.active=dev'
   ```

---

## 🔗 Related Documentation

- [Build Config: `build.gradle`](https://github.com/ricsnsuka/FootMania-Back/blob/main/build.gradle) — Java toolchain line 17-19
- [Gradle Properties](https://github.com/ricsnsuka/FootMania-Back/blob/main/gradle.properties) — Toolchain path line 3
- [System Properties](https://github.com/ricsnsuka/FootMania-Back/blob/main/system.properties) — Heroku Java version

---

**Classification:** Environment / IDE Configuration  
**Handled By:** Orchestrator (Emergency Protocol)  
**Delegated To:** None (IDE config issue, not code)  
**Tests Modified:** None  
**Code Changes:** None  
**Documentation Updated:** This incident report only

