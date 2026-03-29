# Android Reverse Engineering & API Extraction

This repository includes both a `Claude Code` plugin and a `Codex` plugin wrapper for the same Android reverse-engineering skill.

A Claude Code skill that decompiles Android APK/XAPK/JAR/AAR files, **extracts the HTTP APIs** used by the app, and guides deeper reverse-engineering work such as Frida triage, runtime request inspection, JNI/SO analysis, and sign-location analysis.

## What it does

- **Decompiles** APK, XAPK, JAR, and AAR files using jadx and Fernflower/Vineflower (single engine or side-by-side comparison)
- **Extracts and documents APIs**: Retrofit endpoints, OkHttp calls, hardcoded URLs, auth headers and tokens
- **Traces call flows** from Activities/Fragments through ViewModels and repositories down to HTTP calls
- **Analyzes** app structure: manifest, packages, architecture patterns
- **Handles obfuscated code**: strategies for navigating ProGuard/R8 output
- **Guides runtime analysis**: when to use Frida, where to hook, and how to narrow the request/sign pipeline
- **Guides native analysis**: JNI/SO triage, sign-generation boundary identification, and when unidbg is worth the cost

## Frida Scope

The skill now includes documentation for Frida-oriented workflows, but it does **not** yet ship runnable Frida helper scripts or an auto-attach command.

Current state:

- The skill can guide static triage before hooking
- The skill can tell you which Java or JNI boundary is the best candidate to hook
- The skill can structure runtime findings and native sign analysis
- The repository does **not** yet include `scripts/` for `frida`, device attach, or automatic hook generation

In short: the project currently supports **Frida analysis workflow guidance**, not **built-in Frida execution tooling**.

## Requirements

**Required:**
- Java JDK 17+
- [jadx](https://github.com/skylot/jadx) (CLI)

**Optional (recommended):**
- [Vineflower](https://github.com/Vineflower/vineflower) or [Fernflower](https://github.com/JetBrains/fernflower) — better output on complex Java code
- [dex2jar](https://github.com/pxb1988/dex2jar) — needed to use Fernflower on APK/DEX files

See `plugins/android-reverse-engineering/skills/android-reverse-engineering/references/setup-guide.md` for detailed installation instructions.

## Installation

### For Codex

This repository now includes Codex-compatible plugin metadata:

- `.agents/plugins/marketplace.json`
- `plugins/android-reverse-engineering/.codex-plugin/plugin.json`

The skill content remains in:

- `plugins/android-reverse-engineering/skills/android-reverse-engineering/`

### From GitHub (recommended)

Inside Claude Code, run:

```
/plugin marketplace add SimoneAvogadro/android-reverse-engineering-skill
/plugin install android-reverse-engineering@android-reverse-engineering-skill
```

The skill will be permanently available in all future sessions.

### From a local clone

```bash
git clone https://github.com/SimoneAvogadro/android-reverse-engineering-skill.git
```

Then in Claude Code:

```
/plugin marketplace add /path/to/android-reverse-engineering-skill
/plugin install android-reverse-engineering@android-reverse-engineering-skill
```

## Usage

### Slash command

```
/decompile path/to/app.apk
```

This runs the full workflow: dependency check, decompilation, and initial structure analysis.

### Natural language

The skill activates on phrases like:

- "Decompile this APK"
- "Reverse engineer this Android app"
- "Extract API endpoints from this app"
- "Follow the call flow from LoginActivity"
- "Analyze this AAR library"
- "Find where this app generates its sign"
- "Use Frida to inspect the request pipeline"
- "Analyze the JNI or SO behind this token"

### Manual scripts

The scripts can also be used standalone:

```bash
# Check dependencies
bash plugins/android-reverse-engineering/skills/android-reverse-engineering/scripts/check-deps.sh

# Install a missing dependency (auto-detects OS and package manager)
bash plugins/android-reverse-engineering/skills/android-reverse-engineering/scripts/install-dep.sh jadx
bash plugins/android-reverse-engineering/skills/android-reverse-engineering/scripts/install-dep.sh vineflower

# Decompile APK with jadx (default)
bash plugins/android-reverse-engineering/skills/android-reverse-engineering/scripts/decompile.sh app.apk

# Decompile XAPK (auto-extracts and decompiles each APK inside)
bash plugins/android-reverse-engineering/skills/android-reverse-engineering/scripts/decompile.sh app-bundle.xapk

# Decompile with Fernflower
bash plugins/android-reverse-engineering/skills/android-reverse-engineering/scripts/decompile.sh --engine fernflower library.jar

# Run both engines and compare
bash plugins/android-reverse-engineering/skills/android-reverse-engineering/scripts/decompile.sh --engine both --deobf app.apk

# Find API calls
bash plugins/android-reverse-engineering/skills/android-reverse-engineering/scripts/find-api-calls.sh output/sources/
bash plugins/android-reverse-engineering/skills/android-reverse-engineering/scripts/find-api-calls.sh output/sources/ --retrofit
bash plugins/android-reverse-engineering/skills/android-reverse-engineering/scripts/find-api-calls.sh output/sources/ --urls
```

## Repository Structure

```
android-reverse-engineering-skill/
├── .agents/
│   └── plugins/
│       └── marketplace.json                # Codex marketplace catalog
├── .claude-plugin/
│   └── marketplace.json                    # Marketplace catalog
├── plugins/
│   └── android-reverse-engineering/
│       ├── .codex-plugin/
│       │   └── plugin.json                 # Codex plugin manifest
│       ├── .claude-plugin/
│       │   └── plugin.json                 # Plugin manifest
│       ├── skills/
│       │   └── android-reverse-engineering/
│       │       ├── SKILL.md                # Core workflow (static + dynamic triage)
│       │       ├── references/
│       │       │   ├── setup-guide.md
│       │       │   ├── jadx-usage.md
│       │       │   ├── fernflower-usage.md
│       │       │   ├── api-extraction-patterns.md
│       │       │   ├── call-flow-analysis.md
│       │       │   ├── dynamic-analysis.md
│       │       │   └── native-analysis.md
│       │       └── scripts/
│       │           ├── check-deps.sh
│       │           ├── install-dep.sh
│       │           ├── decompile.sh
│       │           └── find-api-calls.sh
│       └── commands/
│           └── decompile.md                # /decompile slash command
├── LICENSE
└── README.md
```

## References

- [jadx — Dex to Java decompiler](https://github.com/skylot/jadx)
- [Fernflower — JetBrains analytical decompiler](https://github.com/JetBrains/fernflower)
- [Vineflower — Fernflower community fork](https://github.com/Vineflower/vineflower)
- [dex2jar — DEX to JAR converter](https://github.com/pxb1988/dex2jar)
- [apktool — Android resource decoder](https://apktool.org/)
- [Frida — Dynamic instrumentation toolkit](https://frida.re/)

## Disclaimer

This plugin is provided strictly for **lawful purposes**, including but not limited to:

- Security research and authorized penetration testing
- Interoperability analysis permitted under applicable law (e.g., EU Directive 2009/24/EC, US DMCA §1201(f))
- Malware analysis and incident response
- Educational use and CTF competitions

**You are solely responsible** for ensuring that your use of this tool complies with all applicable laws, regulations, and terms of service. Unauthorized reverse engineering of software you do not own or do not have permission to analyze may violate intellectual property laws and computer fraud statutes in your jurisdiction.

The authors disclaim any liability for misuse of this tool.

## License

Apache 2.0 — see [LICENSE](LICENSE)
