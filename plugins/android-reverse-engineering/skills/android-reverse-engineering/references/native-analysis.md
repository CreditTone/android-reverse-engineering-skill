# Native Analysis

Use this reference when Java code delegates signing, token generation, encryption, or request assembly to JNI or `.so` libraries.

## Decision Tree

After JADX triage:

- If the algorithm is visible in Java, stay in Java
- If Java calls a `native` method, inspect the SO
- If the app works without a sign, do not reverse the SO unnecessarily
- If offline reproduction is required, consider unidbg after confirming the native call boundary

## First Pass on Native Code

Identify:

- `native` method declarations in Java
- `System.loadLibrary` call sites
- the target SO file name
- whether JNI is static (`Java_*` exports) or dynamic (`JNI_OnLoad`, `RegisterNatives`)

## Questions to Answer

- What native method generates the value of interest?
- What arguments are passed in from Java?
- Which arguments vary per request?
- Which arguments are device- or session-bound?
- Is the return value the final sign, or an intermediate token?

## Practical Workflow

1. Find the Java call site
2. Record the Java-side arguments and nearby context
3. Inspect the SO for matching entrypoints
4. Confirm inputs and outputs with runtime hooks if possible
5. Only then decide whether deeper reversing is worth the effort

## Signs That Native Inspection Is Needed

- `native` methods are called during request preparation
- Java code only marshals arguments and immediately returns the native result
- sign-related constants or headers appear, but no Java implementation exists

## When to Escalate to unidbg

Use unidbg only when the user specifically needs one of these:

- offline signature generation
- repeatable execution without a live device
- deeper understanding of a native algorithm that cannot be recovered from light hooks

Do not jump to unidbg before identifying:

- the exact JNI entrypoint
- the minimal required arguments
- whether environment dependencies exist

## Output Template

```markdown
## Native Sign Assessment

- Java entrypoint: `com.example.Signer.nativeGetSign(...)`
- SO: `libsign.so`
- JNI style: dynamic registration
- Inputs: body, timestamp, deviceId, session token
- Output: request signature string
- Confidence: medium
- Recommended next step: Frida hook entry and return value before deeper SO analysis
```
