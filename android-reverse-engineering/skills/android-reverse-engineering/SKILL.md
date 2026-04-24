---
name: android-reverse-engineering
description: Decompile Android APK, XAPK, JAR, and AAR files using jadx or Fernflower/Vineflower. Reverse engineer Android apps, extract HTTP API endpoints, trace call flows from UI to network layer, and analyze runtime behavior with Frida, network capture, JNI/SO inspection, and signature generation. Use when the user wants to decompile, analyze, hook, inspect network traffic, bypass SSL pinning for analysis, locate crypto or signing logic, or follow call flows in Android packages.
trigger: decompile APK|decompile XAPK|reverse engineer Android|extract API|analyze Android|jadx|fernflower|vineflower|follow call flow|decompile JAR|decompile AAR|Android reverse engineering|find API endpoints|frida|hook android|ssl pinning|network capture|android packet capture|jni|so analysis|sign analysis|signature generation|unidbg
---

# Android Reverse Engineering

Decompile Android APK, XAPK, JAR, and AAR files using jadx and Fernflower/Vineflower, trace call flows through application code and libraries, produce structured documentation of extracted APIs, and escalate to runtime analysis only after static triage shows that it is needed. Two decompiler engines are supported: jadx for broad Android coverage and Fernflower/Vineflower for higher-quality output on complex Java code.

## Core Principle

Do not jump straight into Frida, packet capture, or SO analysis. Start with JADX and identify:

- The network stack in use
- The request builder or interceptor chain
- Where signing or encryption appears to happen
- Whether the logic is visible in Java or delegated to native code

Use dynamic analysis only to confirm or bridge gaps that static analysis cannot resolve.

## Prerequisites

This skill requires **Java JDK 17+** and **jadx** to be installed. **Fernflower/Vineflower**, **dex2jar**, and **rizin** are optional but recommended for better decompilation quality and native `.so` analysis. Run the dependency checker to verify:

```bash
bash skills/android-reverse-engineering/scripts/check-deps.sh
```

If anything is missing, follow the installation instructions in `skills/android-reverse-engineering/references/setup-guide.md`.

## Workflow

### Phase 1: Verify and Install Dependencies

Before decompiling, confirm that the required tools are available — and install any that are missing.

**Action**: Run the dependency check script.

```bash
bash skills/android-reverse-engineering/scripts/check-deps.sh
```

The output contains machine-readable lines:
- `INSTALL_REQUIRED:<dep>` — must be installed before proceeding
- `INSTALL_OPTIONAL:<dep>` — recommended but not blocking

**If required dependencies are missing** (exit code 1), install them automatically:

```bash
bash skills/android-reverse-engineering/scripts/install-dep.sh <dep>
```

The install script detects the OS and package manager, then:
- Installs without sudo when possible (downloads to `~/.local/share/`, symlinks in `~/.local/bin/`)
- Uses sudo and the system package manager when necessary (apt, dnf, pacman)
- If sudo is needed but unavailable or the user declines, it prints the exact manual command and exits with code 2 — show these instructions to the user

**For optional dependencies**, ask the user if they want to install them. Vineflower and dex2jar are recommended for best results. Rizin is recommended when JNI or `.so` inspection is likely.

After installation, re-run `check-deps.sh` to confirm everything is in place. Do not proceed to Phase 2 until all required dependencies are OK.

### Phase 2: Decompile

Use the decompile wrapper script to process the target file. The script supports three engines: `jadx`, `fernflower`, and `both`.

**Action**: Choose the engine and run the decompile script. The script handles APK, XAPK, JAR, and AAR files.

```bash
bash skills/android-reverse-engineering/scripts/decompile.sh [OPTIONS] <file>
```

For **XAPK** files (ZIP bundles containing multiple APKs, used by APKPure and similar stores): the script automatically extracts the archive, identifies all APK files inside (base + split APKs), and decompiles each one into a separate subdirectory. The XAPK manifest is copied to the output for reference.

Options:
- `-o <dir>` — Custom output directory (default: `<filename>-decompiled`)
- `--deobf` — Enable deobfuscation of names when readability matters more than runtime class fidelity
- `--no-res` — Skip resources, decompile code only (faster)
- `--engine ENGINE` — `jadx` (default), `fernflower`, or `both`

**Engine selection strategy**:

| Situation | Engine |
|---|---|
| First pass on any APK | `jadx` (fastest, handles resources; keep original runtime class names by default) |
| JAR/AAR library analysis | `fernflower` (better Java output) |
| jadx output has warnings/broken code | `both` (compare and pick best per class) |
| Complex lambdas, generics, streams | `fernflower` |
| Quick overview of a large APK | `jadx --no-res` |
| You need Frida/JNI/unidbg/runtime class names | `jadx` without `--deobf` |
| You only need readability for heavily obfuscated code | `jadx --deobf` |

When using `--engine both`, the outputs go into `<output>/jadx/` and `<output>/fernflower/` respectively, with a comparison summary at the end showing file counts and jadx warning counts. Review classes with jadx warnings in the Fernflower output for better code.

For APK files with Fernflower, the script automatically uses dex2jar as an intermediate step. dex2jar must be installed for this to work.

See `skills/android-reverse-engineering/references/jadx-usage.md` and `skills/android-reverse-engineering/references/fernflower-usage.md` for the full CLI references.

### Phase 3: Analyze Structure

Navigate the decompiled output to understand the app's architecture.

**Actions**:

1. **Read AndroidManifest.xml** from `<output>/resources/AndroidManifest.xml`:
   - Identify the main launcher Activity
   - List all Activities, Services, BroadcastReceivers, ContentProviders
   - Note permissions (especially `INTERNET`, `ACCESS_NETWORK_STATE`)
   - Find the application class (`android:name` on `<application>`)

2. **Survey the package structure** under `<output>/sources/`:
   - Identify the main app package and sub-packages
   - Distinguish app code from third-party libraries
   - Look for packages named `api`, `network`, `data`, `repository`, `service`, `retrofit`, `http` — these are where API calls live

3. **Identify the architecture pattern**:
   - MVP: look for `Presenter` classes
   - MVVM: look for `ViewModel` classes and `LiveData`/`StateFlow`
   - Clean Architecture: look for `domain`, `data`, `presentation` packages
   - This informs where to look for network calls in the next phases

### Phase 4: Trace Call Flows

Follow execution paths from user-facing entry points down to network calls.

**Actions**:

1. **Start from entry points**: Read the main Activity or Application class identified in Phase 3.

2. **Follow the initialization chain**: Application.onCreate() often sets up the HTTP client, base URL, and DI framework. Read this first.

3. **Trace user actions**: From an Activity, follow:
   - `onCreate()` → view setup → click listeners
   - Click handler → ViewModel/Presenter method
   - ViewModel → Repository → API service interface
   - API service → actual HTTP call

4. **Map DI bindings** (if Dagger/Hilt is used): Find `@Module` classes to understand which implementations are provided for which interfaces.

5. **Handle obfuscated code**: When class names are mangled, use string literals and library API calls as anchors. Retrofit annotations and URL strings are never obfuscated.
6. **Prefer runtime-faithful names for hooks/emulation**: Do the first jadx pass without `--deobf` whenever the next step may involve Frida, JNI `FindClass`, unidbg, or runtime class loading. `--deobf` can generate readable aliases that are useful for source navigation but are not guaranteed to exist at runtime.

See `skills/android-reverse-engineering/references/call-flow-analysis.md` for detailed techniques and grep commands.

### Phase 5: Extract and Document APIs

Find all API endpoints and produce structured documentation.

**Action**: Run the API search script for a broad sweep.

```bash
bash skills/android-reverse-engineering/scripts/find-api-calls.sh <output>/sources/
```

Targeted searches:
```bash
# Only Retrofit
bash skills/android-reverse-engineering/scripts/find-api-calls.sh <output>/sources/ --retrofit

# Only hardcoded URLs
bash skills/android-reverse-engineering/scripts/find-api-calls.sh <output>/sources/ --urls

# Only auth patterns
bash skills/android-reverse-engineering/scripts/find-api-calls.sh <output>/sources/ --auth
```

Then, for each discovered endpoint, read the surrounding source code to extract:
- HTTP method and path
- Base URL
- Path parameters, query parameters, request body
- Headers (especially authentication)
- Response type
- Where it's called from (the call chain from Phase 4)

**Document each endpoint** using this format:

```markdown
### `METHOD /path`

- **Source**: `com.example.api.ApiService` (ApiService.java:42)
- **Base URL**: `https://api.example.com/v1`
- **Path params**: `id` (String)
- **Query params**: `page` (int), `limit` (int)
- **Headers**: `Authorization: Bearer <token>`
- **Request body**: `{ "email": "string", "password": "string" }`
- **Response**: `ApiResponse<User>`
- **Called from**: `LoginActivity → LoginViewModel → UserRepository → ApiService`
```

See `skills/android-reverse-engineering/references/api-extraction-patterns.md` for library-specific search patterns and the full documentation template.

### Phase 6: Dynamic Analysis

Use this phase only when static analysis is insufficient or the user explicitly asks for runtime work such as Frida hook, packet capture, SSL pinning investigation, or signature tracing.

Typical escalation cases:

- Request parameters are assembled indirectly and are hard to reconstruct statically
- Java code calls `native` methods for signing, token generation, or encryption
- The app uses a custom network stack and static search does not expose final requests
- The user needs proof of runtime values rather than only code paths

Recommended order:

1. Hook the Java request builder, interceptor, callback, or request object closest to the final outbound request
2. Inspect request URL, headers, body, and any sign-related inputs and outputs
3. Only then decide whether SSL pinning bypass or native inspection is necessary

See:

- `skills/android-reverse-engineering/references/dynamic-analysis.md`
- `skills/android-reverse-engineering/references/native-analysis.md`

### Phase 7: Native and Signature Analysis

If signing or crypto is delegated to JNI or `.so` code:

1. Find `native` declarations in decompiled Java
2. Find matching `System.loadLibrary` calls and identify the target SO
3. Determine whether the SO uses static JNI exports or dynamic registration
4. Prefer confirming function inputs and outputs with runtime hooks before attempting deeper reversing
5. Use unidbg or further RE only when the user specifically needs offline reproduction or deeper algorithm recovery

Focus on answering these questions:

- Is the signature generated in Java or native code?
- What are the exact inputs to the signing function?
- Which inputs are stable constants vs runtime values such as timestamp, cookie, device ID, or request body?
- Can the algorithm be replayed by calling the app, or must it be reimplemented?

**Preferred CLI for `.so` analysis**:

- `rizin` / `rz-bin` first when available
- Fall back to `readelf`, `nm`, `objdump`, and `strings` if rizin is unavailable

**Recommended `.so` command set**:

```bash
# Basic file identity
file libfoo.so

# ELF metadata, imports/exports, sections
rz-bin -I libfoo.so
rz-bin -s libfoo.so
rz-bin -i libfoo.so
rz-bin -E libfoo.so

# Fast string triage
rz-strings -a libfoo.so | rg 'http|https|Java_|JNI_OnLoad|RegisterNatives|encrypt|sign|ssl|socket'

# Function list and disassembly
rizin -qc "aaa; afl; q" libfoo.so
rizin -qc "aaa; pdf @ sym.JNI_OnLoad; q" libfoo.so
rizin -qc "aaa; pdr @ sym.JNI_OnLoad; q" libfoo.so
```

**Useful fallbacks without rizin**:

```bash
readelf -d libfoo.so
readelf -Ws libfoo.so
nm -D libfoo.so | rg 'Java_|JNI_OnLoad|RegisterNatives'
objdump -T libfoo.so
objdump -d libfoo.so > libfoo.objdump.asm
strings -a libfoo.so | rg 'http|https|Java_|JNI_OnLoad|RegisterNatives|encrypt|sign|ssl|socket'
```

**Default `.so` workflow**:

1. Identify architecture and linked libraries with `file` and `rz-bin -I`
2. Check exports/imports for `JNI_OnLoad`, `Java_*`, and crypto or socket APIs
3. Search strings for URLs, hostnames, log tags, and registration clues
4. Inspect `JNI_OnLoad` first, then look for `RegisterNatives` targets
5. Only do deep function reversing after the Java-side call path is known

## Output

At the end of the workflow, deliver:

1. **Decompiled source** in the output directory
2. **Architecture summary** — app structure, main packages, pattern used
3. **API documentation** — all discovered endpoints in the format above
4. **Call flow map** — key paths from UI to network (especially authentication and main features)
5. **Runtime findings** — when applicable, the hook point, observed request fields, and the reason this point was chosen
6. **Signature assessment** — whether signing is Java-based, native-backed, or not required for the target request

## References

- `skills/android-reverse-engineering/references/setup-guide.md` — Installing Java, jadx, Fernflower/Vineflower, dex2jar, and optional tools
- `skills/android-reverse-engineering/references/jadx-usage.md` — jadx CLI options and workflows
- `skills/android-reverse-engineering/references/fernflower-usage.md` — Fernflower/Vineflower CLI options, when to use, APK workflow
- `skills/android-reverse-engineering/references/api-extraction-patterns.md` — Library-specific search patterns and documentation template
- `skills/android-reverse-engineering/references/call-flow-analysis.md` — Techniques for tracing call flows in decompiled code
- `skills/android-reverse-engineering/references/dynamic-analysis.md` — Frida, runtime request interception, SSL pinning triage, and packet capture workflow
- `skills/android-reverse-engineering/references/native-analysis.md` — JNI/SO inspection, native sign analysis, and when to escalate to unidbg
