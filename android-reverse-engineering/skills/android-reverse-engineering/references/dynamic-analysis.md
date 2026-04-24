# Dynamic Analysis

Use runtime analysis only after static analysis has narrowed the search space.

## Goals

- Confirm the final request URL, headers, and body
- Capture sign-related inputs and outputs
- Identify the best hook point with the least noise
- Decide whether SSL pinning bypass or native inspection is actually required

## Triage Order

1. Decompile the APK and identify the network stack first
2. Locate request builders, interceptors, callbacks, or adapter layers
3. Hook the narrowest layer that still exposes the final outbound request
4. Escalate to SSL pinning or packet capture only if Java-side hooks are not enough

## What to Look For in JADX

Search for:

- `Retrofit`, `OkHttp`, `Volley`, `Cronet`, `HttpURLConnection`, `TTNet`
- `Interceptor`, `intercept`, `addInterceptor`
- `Callback`, `onResponse`, `onFailure`, `onSuccess`
- `sign`, `token`, `encrypt`, `decrypt`
- `native `, `System.loadLibrary`, `System.load`

Produce a short triage summary before hooking:

- network framework
- request builder class
- interceptor or middleware chain
- likely sign generator
- whether sign logic appears in Java or native

## Hooking Strategy

Prefer these hook points in order:

1. Final request object construction
2. Interceptor methods
3. Request execution entrypoints
4. Sign or token generation methods
5. Native methods only if Java no longer exposes the needed values

For each hook, capture:

- class and method name
- request URL
- HTTP method
- headers
- body or serialized payload
- sign-related input parameters
- sign-related return value

## SSL Pinning Guidance

Do not start by bypassing SSL pinning. First try to observe requests in-process.

Escalate only when:

- the app sends traffic but Java hooks do not expose the final request
- the target values are visible only after TLS is established
- the user explicitly asks for packet capture or certificate bypass

If pinning bypass is required, describe:

- why Java-layer inspection was insufficient
- which library likely performs pinning
- the exact point you intend to patch or hook

## Packet Capture Guidance

Packet capture is a verification tool, not the default first step.

Use it when:

- you need to validate that a reconstructed request matches runtime traffic
- the app uses a custom transport stack
- the user needs evidence of exact on-wire behavior

Pair capture results with code locations. Raw traffic without code context is not enough.

## Output Template

```markdown
## Runtime Finding

- Hook point: `com.example.net.SignInterceptor.intercept`
- Why this point: closest Java layer before outbound request
- URL: `https://example.com/api/...`
- Method: `POST`
- Headers: `x-sign`, `x-t`, `cookie`, ...
- Body shape: `{...}`
- Sign source: Java / native / unknown
- Next step: inspect native method `nativeGetSign(...)`
```
