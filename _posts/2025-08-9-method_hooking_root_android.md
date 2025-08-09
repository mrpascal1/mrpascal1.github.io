---
layout: post
author: Shahid Raza
title: Method Hooking Detection & Runtime Code Integrity Checks — Hardcore Android Guide
tags: Engineering Android Security
---

Deep dive, practical code, trade-offs, and a real-world implementation plan you can drop into a production Android app.
<!--more--> 

**TL;DR:** attackers use runtime instrumentation (Frida, Xposed, inline/native hooking) to change app behavior without touching your APK on disk. The right defense combines artifact detection (root / hooking frameworks), runtime behavioral checks (method prologues, stack inspection), integrity checks (SHA-256 of `classes.dex` and native libraries), hardware-backed attestation (Play Integrity / SafetyNet), and a well-designed response policy. Nothing is 100% — this is a cat-and-mouse game — but layered defenses raise the cost of attack and catch most common threats.

---

## Table of Contents

1. [Goals & Threat Model](#goals--threat-model)
2. [How Hooking Works — technical primer](#how-hooking-works---technical-primer)
3. [Categories of Detection](#categories-of-detection)
4. [Practical Kotlin implementations — code you can drop in today](#practical-kotlin-implementations)
5. [Advanced native / ART-level checks (concept + example)](#advanced-native--art-level-checks)
6. [Runtime integrity checks — building a trustworthy chain](#runtime-integrity-checks)
7. [Attestation & server-side verification](#attestation--server-side-verification)
8. [Hardening, deployment, and telemetry](#hardening-deployment-and-telemetry)
9. [Testing, false positives & UX](#testing-false-positives--ux)
10. [Limitations & how attackers will try to bypass you](#limitations--how-attackers-will-try-to-bypass-you)
11. [Action plan & checklist](#action-plan--checklist)

---

## Goals & Threat Model

**Goal:** Detect and respond to runtime method hooking and unauthorized runtime code modification (in-process instrumentation) to protect sensitive flows — e.g., crypto, payments, MFA, or banking logic.

**We protect against**
- Java-level hooking (Xposed, Substrate, custom classloader-injection)
- Dynamic instrumentation (Frida: frida-server / frida-gadget / Gum hooks)
- Inline native hooking (patching native prologues / GOT/PLT / trampolines)
- Runtime method replacement via reflection or dex/classloader swapping

**We don't (realistically) guarantee protection against**
- Full device compromise where attacker controls kernel or can modify hardware
- Highly motivated attackers who patch native memory and your native code continuously
- Emulated or instrumented environments where the attacker modifies everything (but we'll make detection likely)

This guide expects you have control of the app build pipeline and a server backend where you can perform attestation/verification when needed.

---

## How Hooking Works — technical primer

### Java-level hooking
- **Xposed**: system or framework-level module that intercepts class loading and substitutes method calls. It usually injects `XposedBridge` and modifies method pointers. Works best on rooted devices, but there are variants.
- **Classloader replacement / custom classloaders**: load a modified class from another dex or jar and replace original implementation.

### Native-level hooking
- **Inline patching**: change the first few bytes of a function to jump into attacker code (hot-patching, trampolines).
- **PLT/GOT rewriting**: change entries in the Procedure Linkage Table to redirect calls.
- **LD_PRELOAD / library injection**: preloading a library to replace libc or symbol implementations.
- **Frida/Gum**: runtime instrumentation framework that can hook at JS level using a gadget library injected into the process.

### ART / Method table manipulation
- On modern Android (`ART`), a `jmethodID` points to an ArtMethod structure; attackers can change ArtMethod entrypoints to alter behavior. This usually requires native code and knowledge of ART internals.

Understanding these mechanisms matters because each leads to different observable artifacts (files, loaded libraries, modified memory) and different detection strategies.

---

## Categories of Detection

We'll group detection strategies by *what they observe*:

1. **Environment / Artifact detection** — filesystem/process/package artifacts (su, Magisk, XposedBridge, frida-gadget libraries, suspicious packages)
2. **Behavioral detection** — unusual flows, changed function prologues, stack frames showing instrumentation, unexpected classloader sources
3. **Integrity checks** — compute checksums of `classes.dex`, native `.so` files, certificate pins, and compare with known-good values
4. **External attestation** — Play Integrity / SafetyNet attestation to get a Google-signed statement about device / app integrity

Each strategy has pros/cons: artifact detection is cheap but can be evaded. Integrity checks are strong for on-disk tampering; they fail to detect in-memory-only patching. Behavioral/native checks can detect in-memory hooking but are fragile and platform-dependent.

---

## Practical Kotlin implementations

Below are battle-tested snippets (Kotlin) you can use right away. Keep each snippet small and modular — compose them into a `SecurityManager` class for your app.

> **Note**: Kotlin code is written for modern Android (API level checks used where necessary). Add required permissions (e.g., `READ_EXTERNAL_STORAGE` when relevant) and ensure you run blocking IO off the main thread.

### Helper: hex / sha utilities

```kotlin

fun ByteArray.toHex(): String = joinToString("") { "%02x".format(it) }

fun InputStream.sha256(): String {
    val md = MessageDigest.getInstance("SHA-256")
    val buffer = ByteArray(4 * 1024)
    var read = this.read(buffer)
    while (read > 0) {
        md.update(buffer, 0, read)
        read = this.read(buffer)
    }
    return md.digest().toHex()
}

fun File.sha256(): String = this.inputStream().use { it.sha256() }
```

### 1) Root detection (artifact + behavioral)

This is standard first-line detection.

```kotlin

fun isDeviceRooted(): Boolean {
    // Common su locations
    val paths = listOf(
        "/system/app/Superuser.apk",
        "/sbin/su",
        "/system/bin/su",
        "/system/xbin/su",
        "/data/local/xbin/su",
        "/data/local/bin/su",
        "/system/sd/xbin/su",
        "/system/bin/failsafe/su",
        "/data/local/su"
    )
    if (paths.any { File(it).exists() }) return true

    // Try which su
    try {
        val process = Runtime.getRuntime().exec(arrayOf("/system/xbin/which", "su"))
        val r = BufferedReader(InputStreamReader(process.inputStream))
        if (r.readLine() != null) return true
    } catch (_: Throwable) { /* ignore */ }

    // TracerPid check - is debugger/ptrace attached?
    try {
        val status = File("/proc/self/status").readText()
        val tracerPid = "TracerPid:\\s*(\\d+)".toRegex().find(status)?.groups?.get(1)?.value?.toIntOrNull()
        if (tracerPid != null && tracerPid > 0) return true
    } catch (_: Throwable) { }

    return false
}
```

**Why this helps:** many hooking frameworks and instrumentation setups require root. Not always the case (Frida gadget can be used on non-rooted devices), so combine with other checks.

---

### 2) Xposed detection (Java-level)

```kotlin
fun isXposedPresent(): Boolean = try {
    Class.forName("de.robv.android.xposed.XposedBridge") != null
} catch (_: Throwable) {
    false
}
```

**Caveat:** attackers may rename classes; this is heuristic-only.

---

### 3) Frida / Frida-Gadget / Gum detection (artifact + memory map scan)

A simple yet effective check is to scan `/proc/self/maps` for known substrings like `frida`, `gum`, or `frida-gadget` which indicate a loaded instrumentation library.

```kotlin
fun isFridaGadgetLoaded(): Boolean {
    return try {
        val maps = File("/proc/self/maps").readText()
        val lower = maps.lowercase()
        listOf("frida", "gum", "frida-gadget").any { lower.contains(it) }
    } catch (_: Throwable) {
        false
    }
}
```

You can extend this to search `/proc/net/tcp` for unusual listening ports (frida-server often listens on TCP), or to enumerate loaded libraries from the process maps and match suspicious names.

---

### 4) APK signing / certificate pin (static integrity)

Check whether the app's signing certificate matches the one you expect. Do **not** rely on this alone — an attacker could re-sign and install modified APK — but it’s an important check.

```kotlin

fun getSigningCertSha256(context: Context): String? {
    val pm = context.packageManager
    val packageName = context.packageName
    val packageInfo = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.P) {
        pm.getPackageInfo(packageName, PackageManager.GET_SIGNING_CERTIFICATES)
    } else {
        @Suppress("DEPRECATION")
        pm.getPackageInfo(packageName, PackageManager.GET_SIGNATURES)
    }

    val certBytes: ByteArray? = when {
        Build.VERSION.SDK_INT >= Build.VERSION_CODES.P -> {
            val signingInfo = packageInfo.signingInfo
            val signers = signingInfo.apkContentsSigners
            signers?.firstOrNull()?.toByteArray()
        }
        else -> {
            @Suppress("DEPRECATION")
            packageInfo.signatures?.firstOrNull()?.toByteArray()
        }
    }

    if (certBytes == null) return null
    val cf = CertificateFactory.getInstance("X.509")
    val cert = cf.generateCertificate(ByteArrayInputStream(certBytes))
    val md = MessageDigest.getInstance("SHA-256")
    val fingerprint = md.digest(cert.encoded)
    return fingerprint.toHex()
}
```

At build time compute the expected SHA-256 of your signing cert, store it server-side or embed it in native code to compare at runtime.

---

### 5) `classes.dex` checksum (detect on-disk tamper)

If an attacker modifies the APK or replaces classes.dex, its hash will change. Compute a SHA-256 of `classes.dex` inside the APK and compare against a known-good value produced at build time.

```kotlin

fun classesDexSha256(context: Context): String? {
    val apk = File(context.applicationInfo.sourceDir)
    ZipFile(apk).use { zip ->
        val entry = zip.getEntry("classes.dex") ?: return null
        zip.getInputStream(entry).use { return it.sha256() }
    }
}
```

**Important:** This detects on-disk tampering. It will not catch in-memory-only hooks (Frida) where the APK is unchanged.

---

### 6) Watchdog thread (periodic re-checks)

Some instrumentation happens after app start. Run lightweight checks periodically and escalate if anomalies appear.

```kotlin

object RuntimeWatchdog {
    private val scheduler = Executors.newSingleThreadScheduledExecutor()

    fun start(context: Context) {
        scheduler.scheduleAtFixedRate({
            try {
                if (isFridaGadgetLoaded() || isXposedPresent() || isDeviceRooted()) {
                    // handle detection
                    onTamperDetected(context)
                }
            } catch (_: Throwable) { }
        }, 5, 10, TimeUnit.SECONDS)
    }

    fun stop() { scheduler.shutdownNow() }

    private fun onTamperDetected(context: Context) {
        // DON'T show attacker-specific messages. degrade functionality or require attestation.
        // Example: clear session, lock sensitive features, notify backend, or exit.
        // We'll decide policy in the "Responses" section below.
    }
}
```

Run this watchdog when a sensitive activity starts and scale checks according to performance budget.

---

## Advanced native / ART-level checks

These checks are more powerful but **platform-dependent**, brittle, and require native code.

### 1) Why native helps
- Attackers commonly hook at Java or instrumentation level. Native code can inspect raw memory and file contents in ways Java cannot easily replicate.
- You can keep sensitive expected values (hashes, secrets) in native compiled libraries rather than plain Java constants. (Note: native constants are not unbreakable.)

### 2) ArtMethod / jmethodID inspection (high-level idea)

On ART, a `jmethodID` (obtained from a reflected `Method`) can be interpreted as a pointer to an `ArtMethod` structure. By using `FromReflectedMethod` in JNI you can get a `jmethodID`, then read bytes from its compiled entrypoint or the ArtMethod structure to see whether it was patched.

**VERY IMPORTANT:** offsets and structure layout differ across Android versions and OEM builds. This is fragile and needs per-OS-version handling. But when you get it right, you can detect that the function entry pointer was replaced by a trampoline.

**Conceptual JNI snippet (illustrative):**

```cpp
// C++ (conceptual) - compile in NDK
#include <jni.h>
#include <string.h>

extern "C"
JNIEXPORT jboolean JNICALL
Java_com_example_security_NativeChecks_isMethodPatched(JNIEnv* env, jclass, jobject reflectiveMethod) {
    // Get the internal method pointer
    void* methodPtr = env->FromReflectedMethod(reflectiveMethod); // jmethodID
    // read first N bytes at the compiled entry point, compute a checksum and compare
    // WARNING: platform-dependent offsets need to be applied here
}
```

Implementing this **requires** a lot of per-version reverse engineering (ArtMethod offsets and entrypoints). There are public write-ups and projects that map offsets per Android release. Use them as reference and maintain this code as you support new OS versions.

### 3) Native prologue checks (less fragile)

Instead of relying on ArtMethod internals, you can choose to compute SHA-256 of your native `.so` files on disk and validate them at runtime (Kotlin can do this). To detect in-memory inline-patching of a native function, a native routine can read the first 16-32 bytes of the function in memory and compute a hash. Attackers that patch code need to also patch these checks or hook them; moving checks into multiple native places helps.

**Caveat:** reading `/proc/self/mem` and scanning your own code pages is possible but requires care. This section is intentionally conceptual — if you want, I can provide a tested NDK implementation that reads an exported function's prologue and hashes it, but it will require per-ABI and ABI-version testing.

---

## Runtime integrity checks — building a trustworthy chain

An effective integrity check system should combine **on-device checks** with **server-side verification** and hardware-backed attestation.

### Build-time: compute expected values

At build or CI time compute SHA-256 of:
- `classes.dex` (or multiple dex files: `classes2.dex`, ...)
- All `.so` native libraries
- Your signing certificate (SHA-256)

Store these values in a secure place: server-side DB or baked into a native component that is harder to tamper with. Prefer server storage + signature.

**Gradle task (conceptual):** compute SHA and print to a file or resource that CI reads and uploads to a server.

### Runtime: verify values quickly

At app start and periodically:
1. compute `classes.dex` SHA-256 and compare to expected. If mismatch → likely APK tamper.
2. compute `.so` SHA-256 or verify file size + checksum.
3. verify app signing certificate SHA-256.

If all these pass but you still suspect instrumentation (Frida hooking in-memory), perform behavioral/native checks.

### Protecting secrets & checksums

- Don’t store expected values only in Java strings — attackers can change them. Put them on a server and require remote attestation or put them in native code.
- Even native is patchable; therefore, combine multiple checks and server-side verification.

---

## Attestation & server-side verification

Google’s Play Integrity (formerly SafetyNet) is the recommended way to get a Google-signed attestation of the device and app state. Flow at a high level:

1. App requests an integrity token from Play Integrity (client-side SDK). Provide a `nonce` created by your server.
2. Play Integrity returns a signed token. The app sends this token back to your server.
3. Server verifies the signature of the token (Google public keys) and checks fields: APK signing, package name, device integrity, timestamp, CTS profile match, etc.
4. Server compares token contents with server-side expected values (class hash, signing certificate, version) and decides whether to allow sensitive operations.

**Why server-side?** Because server-side logic is under your control and can't be easily bypassed by client manipulations.

**Important notes:**
- Play Integrity is not 100% — attackers can sometimes pass integrity checks on rooted devices or emulate responses — but it raises the bar significantly.
- Always verify the integrity token on the server using Google’s libraries and public keys.

---

## Hardening, deployment, and telemetry

**Defense in depth checklist:**

- Obfuscate code (R8 / ProGuard). Consider commercial tools (DexGuard) for stronger protection.
- Keep critical code in native where feasible.
- Bake in multiple, redundant runtime checks (artifact + integrity + behavior + attestation).
- Rate-limit and backoff: if tamper detected repeatedly, throttle or require re-authentication.
- Remote logging: send tamper events to your server (careful of privacy). Centralize telemetry to spot large-scale attacks.
- Implement staged responses: soft-fail (warn + log) → feature gating (sensitive features locked) → hard fail (exit) depending on confidence.

**Performance:** Keep expensive checks (full file hash) off the UI thread and run them occasionally — not every frame.

**Privacy / compliance:** Be careful about reporting device files or personal data back to server. Log only meaningful detection events.

---

## Testing, false positives & UX

- **Unit testing**: write tests that manipulate expected values and ensure detection triggers.
- **Manual testing**: install Xposed, Frida, or other frameworks on a controlled lab device and measure detection rates.
- **False positives**: tolerate them early — incorrectly blocking a legitimate user is worse than logging an event. Gradually harden response after observing telemetry.
- **User messaging**: avoid accusatory language. Use neutral messages like: “We can’t verify this device’s security. For your safety some features are disabled.”

---

## Limitations & how attackers will try to bypass you

**Attack patterns to expect:**
- Attackers modify APK to bypass `classes.dex` checks and re-sign the APK; requiring server-side verification with Play Integrity helps here.
- For in-memory-only hooks (Frida), attackers load `frida-gadget` into process memory — you detect that by scanning `/proc/self/maps` or loaded library list.
- Skilled attackers will patch your native checks or patch the check code itself; counter by putting checks in multiple places and doing some checks on server (attestation).

**Never assume a single check is sufficient.** Each check can be bypassed. Stack them.

---

## Action plan & checklist

**Phase 0 — baseline:**
- Add root detection, Xposed detection, Frida map scan.
- Compute `classes.dex` sha256 at app start and compare to build-time known value.
- Add lightweight watchdog to periodically re-check.

**Phase 1 — strengthen:**
- Move expected hashes to server and require a signed attestation (Play Integrity)
- Put at least one hash/secret inside native library (NDK).
- Add certificate pinning for backend communications.

**Phase 2 — advanced:**
- Implement a small native routine that computes a prologue hash for one or two exported functions (per-ABI and per-OS). Re-check at runtime.
- Maintain a mapping of ART offsets for supported Android versions and use `FromReflectedMethod` to inspect method entry points.

**Phase 3 — monitoring & response:**
- Ship telemetry for tamper events.
- Implement progressive response: log → lock sensitive features → force reauth / require attestation → exit.

---

## Example `SecurityManager` integration plan (Kotlin)

1. On app cold start, synchronous lightweight checks:
   - `isDeviceRooted()`, `isXposedPresent()`, `getSigningCertSha256()` against expected
2. Offload heavy checks to background: `classesDexSha256()` and `.so` file hashes
3. Query Play Integrity with server-provided nonce and include server-side verification
4. Start `RuntimeWatchdog` when users enter sensitive flows
5. On detection: notify server, downgrade functionality, require re-auth

---

## Closing notes

- There is no 100% protection — do not claim otherwise in your blog. Instead explain the layered approach and concrete tradeoffs.
- Keep your detection logic under active maintenance; Android internals change often.
- Prioritize server-side attestation for high-sensitivity operations — the device can be hacked, your server cannot (assuming proper security).


