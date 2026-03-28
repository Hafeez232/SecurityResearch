# DarkSword-RCE  
## Static Technical Analysis Report

---

### ⚠ Disclaimer

The analyzed malware is clearly **dual-use and operationally malicious**.

This report **does not** reproduce:

- Exploit-ready gadget chains  
- Complete memory address tables  
- Step-by-step weaponization instructions  

Instead, it focuses strictly on:

- Architectural design  
- Critical functional components  
- Memory layout patterns  
- Reverse-engineering value for defenders and security analysts

This document is provided strictly for **educational, research, and defensive security purposes**.

I (the author) assumes **no responsibility or liability** for any misuse, abuse, damage, or illegal activity resulting from the use, interpretation, or redistribution of this material. Any actions taken based on this analysis are done solely at the reader’s own risk.

This is a **static, non-executing analysis** 

---

## Executive Overview

The codebase is **not** a benign proof-of-concept implementation.  
It contains a structured exploit chain that:

1. **Bootstraps from web content** into a compromised worker context  
2. **Derives arbitrary memory read/write primitives** within JavaScriptCore/WebKit  
3. **Constructs a native call bridge** by abusing PAC-aware function pointers and dyld interposition state  
4. **Deploys a large post-exploitation payload** enabling:
   - Process injection  
   - Sandbox / token abuse  
   - Collection-oriented operations  

---

## Repository Execution Flow

### Initial web staging

- [`index.html`](./index.html) creates a hidden iframe and points it at `frame.html`.
  
  <img width="470" height="230" alt="Screenshot 2026-03-28 130206" src="https://github.com/user-attachments/assets/1204c087-0095-4a97-9036-672310f06cf7" />
  
- [`frame.html`](./frame.html) injects a remote loader from `https://static[.]cdncounter[.]net/assets/rce_loader.js`.
  
  <img width="999" height="136" alt="Screenshot 2026-03-28 132453" src="https://github.com/user-attachments/assets/d126499d-ef49-4083-b570-daf1b6541ce3" />


### Loader and stage selection

[`rce_loader.js`](./rce_loader.js) is the traffic director:

- `getJS()` fetches stage files synchronously.
- It selects version-specific worker and module code.

  <img width="559" height="381" alt="image" src="https://github.com/user-attachments/assets/b996168c-506f-4e89-8105-98e1f8f85506" />

- It creates helper workers used later to manipulate `dlopen`-related state.

  <img width="540" height="244" alt="image" src="https://github.com/user-attachments/assets/1fa68953-e4a4-4e09-bb0f-654b79969d4d" />

- It evaluates the browser-side exploit module and starts the compromise flow.

  <img width="639" height="828" alt="image" src="https://github.com/user-attachments/assets/ef23b317-8a7c-4f3f-a389-36be123122bd" />


### Browser exploit module

[`rce_module.js`](./rce_module.js) contains:

- a very large per-device / per-build offset corpus,
- logic to identify the device/build indirectly,
- the browser memory corruption path,
- post-corruption setup for worker-global object access.

Critical anchors:

- [`rce_module.js:29`](./rce_module.js#L29) `rce_offsets`
- [`rce_module.js:2744`](./rce_module.js#L2744) `class check_attempt`
- [`rce_module.js:2899`](./rce_module.js#L2899) `linkedit_to_device`
- [`rce_module.js:3069`](./rce_module.js#L3069) `device_chipset`
- [`rce_module.js:3252`](./rce_module.js#L3252) `stage2()`
- [`rce_module.js:3299`](./rce_module.js#L3299) `start()`

### Worker-native bridge

[`rce_worker.js`](./rce_worker.js) turns browser memory corruption into a reusable native invocation mechanism.

Critical anchors:

- [`rce_worker.js:13`](./rce_worker.js#L13) `class Encoder`
- [`rce_worker.js:260`](./rce_worker.js#L260) `loadObjcClass()`
- [`rce_worker.js:271`](./rce_worker.js#L271) `case 'stage1'`
- [`rce_worker.js:391`](./rce_worker.js#L391) `case 'dlopen_workers_prepared'`
- [`rce_worker.js:490`](./rce_worker.js#L490) `case 'check_dlopen1'`
- [`rce_worker.js:588`](./rce_worker.js#L588) `case 'check_dlopen2'`
- [`rce_worker.js:651`](./rce_worker.js#L651) `case 'setup_fcall'`
- [`rce_worker.js:717`](./rce_worker.js#L717) `slow_dlopen()`
- [`rce_worker.js:723`](./rce_worker.js#L723) `slow_dlsym()`
- [`rce_worker.js:853`](./rce_worker.js#L853) `fcall()`
- [`rce_worker.js:890`](./rce_worker.js#L890) `dlopen()`
- [`rce_worker.js:897`](./rce_worker.js#L897) `dlsym()`

### Native/post-exploitation stage

[`pe_main.js`](./pe_main.js) is the large native-capable payload. It wraps libc, CoreFoundation, Objective-C, Mach, and task/thread operations and then performs injection and collection tasks.

Critical anchors:

- [`pe_main.js:223`](./pe_main.js#L223) `setup_fcall_jopchain()`
- [`pe_main.js:242`](./pe_main.js#L242) `js_thread_spawn()`
- [`pe_main.js:308`](./pe_main.js#L308) `js_thread_join()`
- [`pe_main.js:8215`](./pe_main.js#L8215) `class MigFilterBypass`
- [`pe_main.js:8300`](./pe_main.js#L8300) `xnuVersion()`
- [`pe_main.js:8313`](./pe_main.js#L8313) `start()`

---

## High-Level Chain Summary

The chain is easiest to understand as five phases:

1. **Web delivery**
   - `index.html` loads `frame.html`.
   - `frame.html` loads a remote `rce_loader.js`.

2. **Browser compromise**
   - `rce_loader.js` chooses the correct exploit files for the detected iOS version.
   - `rce_module.js` attempts a JavaScriptCore/WebKit memory corruption route and derives overlapping array state.

3. **Stable memory primitives**
   - `rce_module.js` and then `rce_worker.js` convert type confusion / overlap into `addrof`, `fakeobj`, `read64`, `write64`, plus small-width writes.
   - The worker becomes the long-lived control plane.

4. **Native bridge creation**
   - `rce_worker.js` abuses dyld/runtime state, framework soft-link slots, and pointer signing helpers.
   - It recovers callable native entry points (`dlopen`, `dlsym`, signing helpers, thread creation).
   - It assembles a PAC-valid JOP-driven `fcall()` bridge.

5. **Post-exploitation**
   - `pe_main.js` resolves large sets of native APIs.
   - It initializes kernel/task helpers and process-injection helpers.
   - It then targets processes such as `SpringBoard`, `configd`, `wifid`, `securityd`, and `UserEventAgent`.

---

## CVE Mapping

Google Threat Intelligence Group's March 19, 2026 DarkSword write-up describes this chain as using six vulnerabilities across RCE, sandbox escape, and privilege-escalation stages.

Source:

- Google Cloud Blog, "The Proliferation of DarkSword: iOS Exploit Chain Adopted by Multiple Threat Actors"  
  https://cloud.google.com/blog/topics/threat-intelligence/darksword-ios-exploit-chain

### CVEs tied to the modules in this repository

| Module | CVE | Public description | Chain role | Patched in |
|---|---|---|---|---|
| `rce_module.js` | `CVE-2025-31277` | JavaScriptCore memory corruption | RCE for older supported versions | iOS 18.6 |
| `rce_worker_18.4.js` | `CVE-2026-20700` | `dyld` user-mode PAC bypass | Native bridge / code execution support | iOS 26.3 |
| `rce_worker_18.6.js` / `rce_worker_18.7.js` | `CVE-2025-43529` | JavaScriptCore memory corruption | RCE for iOS 18.6-18.7 | iOS 18.7.3, 26.2 |
| `rce_worker_18.6.js` / `rce_worker_18.7.js` | `CVE-2026-20700` | `dyld` user-mode PAC bypass | Native bridge / code execution support | iOS 26.3 |
| `sbox0_main_18.4.js` / `sbx0_main.js` | `CVE-2025-14174` | ANGLE memory corruption | WebContent-to-GPU sandbox escape | iOS 18.7.3, 26.2 |
| `sbx1_main.js` | `CVE-2025-43510` | XNU memory management bug | GPU-to-`mediaplaybackd` sandbox escape | iOS 18.7.2, 26.1 |
| `pe_main.js` | `CVE-2025-43520` | XNU VFS race / kernel memory corruption | Local privilege escalation / kernel stage | iOS 18.7.2, 26.1 |

### How the CVEs map onto this repository's flow

1. `CVE-2025-31277` or `CVE-2025-43529` provides the JavaScriptCore renderer compromise.
2. `CVE-2026-20700` is then used to turn browser memory control into a PAC-bypassing native bridge.
3. `CVE-2025-14174` pivots out of the WebContent sandbox into the GPU process.
4. `CVE-2025-43510` pivots again into a more privileged process.
5. `CVE-2025-43520` provides the final kernel-level privilege escalation used by `pe_main.js`.

🔗 The repo contents line up most directly with the RCE, PAC-bypass, sandbox-escape, and kernel-stage modules described in the public write-up.

---

## Critical Function Analysis

## 1. `check_attempt.start()` in `rce_module.js`

Location: [`rce_module.js:3299`](./rce_module.js#L3299)

Purpose:

- Shapes arrays and object layouts to produce an out-of-bounds condition.
- Uses specially prepared arrays with marker values (`egg1`, `egg2`) to discover overlap between a corrupted array and a victim allocation.
- Validates exploitation success by checking for a victim array whose length is unexpectedly large and whose contents reveal the overlapped target.

What it does structurally:

- Allocates many similarly shaped arrays.
- Creates a specially prepared object (`the_oob_object`) whose length/properties are manipulated to influence allocation layout.
- Uses an iframe-created alternate global object to perturb JIT and allocation behavior.
- Performs a delayed check for the victim array and overlap metadata.

Why it matters:

- This is the main browser-memory corruption entry point.
- It does not just crash memory; it tries to produce a reusable and discoverable overlap primitive suitable for later `addrof`/`fakeobj` conversion.

## 2. `check_attempt.stage1()` in `rce_module.js`

The function body appears immediately before the `stage2()` anchor and completes the transition from overlap to stable primitive setup.

Purpose:

- Resolves the active device model using a `libsystem_pthread` `__LINKEDIT`-based fingerprint.
- Maps that device to a precomputed offset profile in `rce_offsets`.
- Computes the ASLR slide from a known JavaScriptCore symbol.
- Applies the slide to all absolute symbol entries.
- Disables JIT allow-list gating and prepares the worker/global-object pivot.

Key indicators:

- Device identification by `libsystem_pthread` linkedit fingerprint.
- Runtime slide computation.
- Bulk adjustment of symbol addresses.
- Writes into JavaScriptCore JIT-related globals.

Why it matters:

- This is the offset-selection and environment-normalization phase.
- It is what makes the same exploit logic portable across many device/build combinations.

## 3. `check_attempt.stage2()` in `rce_module.js`

Location: [`rce_module.js:3252`](./rce_module.js#L3252)

Purpose:

- Walks WebCore script-execution contexts.
- Selects the dedicated worker global scope with the highest worker ID.
- Rebinds array storage so the worker receives a useful memory view.

Operational effect:

- Finds a `DedicatedWorkerGlobalScope` by vtable comparison.
- Navigates from worker context to script/global wrapper structures.
- Rewrites a butterfly/storage pointer relationship between unboxed and boxed arrays.

Why it matters:

- This is the handoff from the browser exploit into a worker-based control environment.
- The worker then becomes the place where the native bridge is built.

## 4. `rce_worker.js` `case 'stage1'`

Location: [`rce_worker.js:271`](./rce_worker.js#L271)

Purpose:

- Converts the stage-2 array primitive into general object/address primitives.
- Builds the read/write backbone used by the rest of the chain.

Core outcomes:

- `p.addrof(o)`
- `p.fakeobj(addr)`
- `p.read64(addr)`
- `p.read32(addr)`
- `p.write64(addr, value)`
- `p.write16(...)`
- `p.write8(...)`

Why it matters:

- This is the first stable general-purpose arbitrary memory access layer.
- Everything after this point uses the `p` object as the runtime memory API.

## 5. `rce_worker.js` `case 'dlopen_workers_prepared'`

Location: [`rce_worker.js:391`](./rce_worker.js#L391)

Purpose:

- Enumerates execution contexts to rediscover special helper workers.
- Identifies workers by sentinel IDs.
- Rewrites selected framework or dispatch state to influence later library-loading behavior.

Analyst note:

- The helper workers are not for normal browser logic. They are exploitation infrastructure used to influence native loading side effects in a controlled way.

## 6. `rce_worker.js` `case 'check_dlopen1'` and `case 'check_dlopen2'`

Locations:

- [`rce_worker.js:490`](./rce_worker.js#L490)
- [`rce_worker.js:588`](./rce_worker.js#L588)

Purpose:

- Recover dyld/runtime state.
- Find loader/interposition structures on a worker stack.
- Stage a controlled rewrite of interposition metadata.
- Redirect selected function-resolution paths toward useful native helpers.

Important behavior:

- Reads dyld runtime structures from `libdyld` globals.
- Searches worker stack memory for a return-address pattern related to `dlopen`.
- Locates a loader object relative to that stack hit.
- Rebinds metadata pointers and waits for interposition size fields to update.

Why it matters:

- This is the bridge from arbitrary read/write to a recoverable, signed native-call substrate.
- It is also the point where framework soft-link tables and loader metadata are abused rather than traditional shellcode injection.

## 7. `rce_worker.js` `case 'setup_fcall'`

Location: [`rce_worker.js:651`](./rce_worker.js#L651)

Purpose:

- Harvests PAC-signed callable helpers.
- Builds two slower helper call paths first.
- Uses those helpers to recover signed `dlopen`, `dlsym`, `malloc`, `pthread_create`, and pointer-signing support.
- Constructs a reusable fast `fcall()` primitive.

Bridge stages:

- `slow_fcall_1(...)`
- `slow_fcall_2(...)`
- `slow_dlopen(...)`
- `slow_dlsym(...)`
- `slow_pacia(...)`
- `slow_pacib(...)`
- final `fcall(...)`

Why it matters:

- This is the real browser-to-native pivot.
- Once `fcall()` exists, the payload can treat the compromised process as a controllable native runtime.

<img width="1113" height="853" alt="Screenshot 2026-03-28 130409" src="https://github.com/user-attachments/assets/2640f7a1-46c2-4071-b1bd-99018cc8cec0" />

## 8. `setup_fcall_jopchain()` in `pe_main.js`

Location: [`pe_main.js:223`](./pe_main.js#L223)

Purpose:

- Creates the JOP-oriented call scaffold used inside spawned JavaScriptCore contexts.
- Lays out a small call buffer and argument area.
- Seeds the structure with PAC-signed gadget/function pointers.

Why it matters:

- It is the native-stage analogue of the worker-side `fcall()`.
- Instead of relying purely on browser objects, it prepares a structured native call trampoline for later JSContext-based execution.

## 9. `js_thread_spawn()` / `js_thread_join()` in `pe_main.js`

Locations:

- [`pe_main.js:242`](./pe_main.js#L242)
- [`pe_main.js:308`](./pe_main.js#L308)

Purpose:

- Spawns a new Objective-C `JSContext`.
- Patches selected arrays and function pointers inside that context.
- Reuses `isNaN`-related executable state as a control point.
- Starts the payload in a fresh `NSThread`.
- Restores modified array buffer pointers on join.

Why it matters:

- This is a higher-level native bridge for executing follow-on JS payloads inside app processes.
- It combines Objective-C runtime messaging with memory patching and JOP setup.

## 10. `MigFilterBypass` in `pe_main.js`

Location: [`pe_main.js:8215`](./pe_main.js#L8215)

Purpose:

- Creates a shared control block.
- Spawns a helper thread that appears intended to bypass or weaken MIG-related restrictions on newer iOS builds.
- Lets later code pause/resume the helper and monitor remote threads.

Why it matters:

- It shows this repository is targeting operational reliability against modern platform hardening, not just demonstrating a browser exploit.

## 11. `start()` in `pe_main.js`

Location: [`pe_main.js:8313`](./pe_main.js#L8313)

Purpose:

- Initializes the native chain and post-exploitation framework.
- Optionally enables the MIG bypass on newer XNU versions.
- Runs PE logic, initializes task/ROP helpers, obtains a `launchd`-backed remote task, applies sandbox/token logic, and injects multiple secondary payloads.

Observed target processes:

- `SpringBoard`
- `configd`
- `wifid`
- `securityd`
- `UserEventAgent`

Why it matters:

- This is the clearest evidence that the repository has moved past exploitation into collection-oriented intrusive behavior.

---

## Memory Offset Breakdown

## 1. Offset corpus design in `rce_module.js`

Location: [`rce_module.js:29`](./rce_module.js#L29)

The `rce_offsets` object is the repository's central compatibility database. Each entry is keyed by a device/build string and contains three broad classes of offsets:

### A. Absolute symbol addresses

Examples by type:

- JavaScriptCore globals
- WebCore globals
- dyld/libdyld globals
- AVFAudio / TextToSpeech symbols
- CFNetwork / Foundation symbols
- Security / ImageIO / CMPhoto symbols

These are stored as high canonical addresses and are later slide-adjusted at runtime.

### B. Object field offsets

Examples by semantic role:

- `m_gpuProcessConnection`
- `m_imageBuffer`
- `m_platformContext`
- `m_remoteDisplayLists`
- `RemoteRenderingBackendProxy_off`
- `privateState_off`
- `vertexAttribVector_off`
- `rxMtlBuffer_off`
- `rxBufferMtl_off`

These are not full addresses. They are structure-member offsets used while navigating in-memory WebKit / graphics objects.

### C. Gadget or helper slots

Examples by semantic role:

- control gadgets
- loop gadgets
- register-setup gadgets
- pointer-signing helpers
- thread-related entry points

These support the PAC-valid call chain and JOP machinery.

## 2. Slide application model

The offset usage pattern is:

1. Resolve one known JavaScriptCore symbol at runtime.
2. Compute `slide = runtime_symbol - stored_symbol`.
3. For each offset entry that looks like a full address, add the slide.
4. Keep small structure-field offsets unchanged.

This is a standard portability technique for ASLR-managed Apple binaries.

## 3. Device/build identification model

Locations:

- [`rce_module.js:2899`](./rce_module.js#L2899)
- [`rce_module.js:3069`](./rce_module.js#L3069)

The module does not appear to trust simple UA strings alone. Instead it:

- locates `pthread_create`,
- derives the base of `libsystem_pthread`,
- reads a `__LINKEDIT`-related value,
- uses that value as a fingerprint into `linkedit_to_device`,
- then maps the resulting device/build into `device_chipset` and `rce_offsets`.

This gives the chain a more reliable build-specific offset profile.

## 4. Custom memory-layout structures used by the malware

The repository also defines its own argument/control blocks. These are safer to document than the full exploit address map because they reflect attacker-authored layouts rather than Apple-internal symbol tables.

📌 This section is a good place for a close-up screenshot if you want one structure-focused figure instead of a broader code-flow image.

### `MigFilterBypass` shared block

Location: [`pe_main.js:8215`](./pe_main.js#L8215)

Allocated with `calloc(1, 0x100)`, then split as:

| Offset | Field | Purpose |
|---|---|---|
| `+0x00` | `runFlagPtr` | Run state control |
| `+0x04` | `isRunningPtr` | Helper thread started flag |
| `+0x08` | `monitorThread1Ptr` | First thread to monitor |
| `+0x10` | `monitorThread2Ptr` | Second thread to monitor |

### `MigFilterBypass` thread argument block

Location: [`pe_main.js:8245`](./pe_main.js#L8245)

Allocated with `calloc(1, 0x400)`, then populated as:

| Offset | Field | Meaning |
|---|---|---|
| `+0x00` | control socket | Kernel R/W helper state |
| `+0x08` | rw socket | Kernel R/W helper state |
| `+0x10` | kernel base | Kernel slide/base context |
| `+0x18` | thread self kobject | Current thread kernel object |
| `+0x20` | run flag ptr | Shared run-state pointer |
| `+0x28` | is-running ptr | Shared started-state pointer |
| `+0x30` | mutex ptr | Synchronization object |
| `+0x38` | `migLock` offset | Kernel/MIG-related field offset |
| `+0x40` | `migSbxMsg` offset | Kernel/MIG-related field offset |
| `+0x48` | `migKernelStackLR` offset | Kernel/MIG-related field offset |
| `+0x50` | monitor thread 1 ptr | Shared monitor slot |
| `+0x58` | monitor thread 2 ptr | Shared monitor slot |

### Worker-side `fcall` scaffolding

Location: [`rce_worker.js:651`](./rce_worker.js#L651)

Important attacker-authored control buffers:

- `gSecurityd` buffer: scratch area used while driving a slow call path.
- `slowFcallResult`: return-slot buffer for the slow native bridge.
- `invoker_x0`: register/argument staging block.
- `invoker_arg`: compact argument descriptor passed through a soft-link hook.
- `jop_thread`: stores thread creation results.
- `x0_u64`, `x19_u64`, `x20_u64`, `x22_u64`: register-state backing arrays.
- `stack_u64`: large synthetic stack for the fast PAC-valid call bridge.

The code uses fixed offsets within those backing arrays to simulate register frames, loop state, and return slots.

### `setup_fcall_jopchain()` buffer

Location: [`pe_main.js:223`](./pe_main.js#L223)

This function allocates one page and then subdivides it:

| Offset from `jsvm_fcall_buff` | Region | Role |
|---|---|---|
| `+0x000` | header area | root control words / entry pointers |
| `+0x100` | `load_x1x3x8_args` | helper gadget arguments |
| `+0x200` | `jsvm_fcall_args` | main argument area for native call setup |

This is a compact JOP staging page used to drive native calls inside the newly created JSContext thread.

---

## Native Bridge Logic

## 1. Browser primitive to object-address primitive

The bridge starts with array corruption and pointer rebinding:

- `unboxed_arr` and `boxed_arr` are made to alias useful storage.
- Objects are converted into addresses with `addrof`.
- Chosen addresses are converted back into JS objects with `fakeobj`.
- Typed-array and string-backed tricks are then used to expose memory as readable/writable data.

This is the core browser-side primitive layer.

## 2. Worker control plane

`rce_loader.js` and `rce_worker.js` treat the worker as the durable exploitation context:

- helper workers are created up front,
- messages drive deterministic transitions,
- the main thread acts largely as a coordinator,
- the worker performs memory introspection and native-bridge setup.

This separation reduces reliance on a fragile DOM main-thread state after exploitation succeeds.

## 3. Framework soft-link abuse

The code repeatedly manipulates soft-link and framework-loader state rather than dropping traditional shellcode.

Patterns visible in `rce_worker.js`:

- forcing class loads via `loadObjcClass()`,
- modifying bundle / framework bookkeeping,
- repointing soft-link targets,
- reusing ImageIO/CMPhoto dispatch points as indirection layers,
- harvesting PAC-signed function pointers through legitimate framework plumbing.

This is a distinctive "live off the platform" bridge style.

## 4. PAC-aware call bridging

The code cannot just jump to arbitrary unsigned pointers. Instead it:

- recovers signing helpers,
- generates signed pointers for gadgets and function entry points,
- constructs loop/control gadget chains,
- uses a synthetic stack/register layout,
- repeatedly re-enters the chain to invoke native functions.

The flow is:

1. establish slow signed call helpers,
2. resolve signed `dlopen` / `dlsym`,
3. resolve additional native functions,
4. sign required gadgets,
5. build a reusable `fcall()` abstraction.

This is the heart of the native bridge.

🛠️ If you only include one technically deep figure in the whole report, this is the section to anchor it to.

## 5. Objective-C / CoreFoundation bridge in `pe_main.js`

Once native calls are stable, the payload wraps them in higher-level helpers:

- `objc_getClass`
- `objc_alloc`
- `objc_alloc_init`
- `objc_msgSend`
- `sel_registerName`
- `CFStringCreateWithCString`
- dictionary and number helpers

That lets the malware move from raw function pointers into managed Objective-C orchestration:

- allocate `JSContext`,
- invoke `evaluateScript:`,
- create and control `NSThread`,
- build `NSInvocation` objects,
- run follow-on JavaScript inside native app contexts.

## 6. Process injection bridge

At the end of the chain, the bridge is no longer just "browser JS to native function". It becomes:

`Web exploit -> Worker memory primitive -> PAC-valid fcall -> native runtime wrappers -> remote task/process injection`

That is why `pe_main.js` contains:

- task and thread manipulation,
- Mach message operations,
- memory mapping helpers,
- remote execution scaffolding,
- sandbox/token manipulation,
- multi-process injection targets.

---

## Defensive Findings

The repository exhibits multiple traits associated with operational malware rather than research-only exploit code:

- hidden iframe-based staging,
- remote loading from an external asset host,
- large build-specific offset coverage,
- PAC-aware call-chain engineering,
- dyld/soft-link manipulation,
- sandbox/token handling,
- injection into multiple privileged or strategically useful processes,
- modules named for credential/data extraction use cases.

Particularly concerning indicators from `pe_main.js` include:

- `keychain_copier`
- `wifi_password_dump`
- `wifi_password_securityd`
- `icloud_dumper`
- `file_downloader`

Those names align with collection/exfiltration objectives.

🚩 The combination of exploit staging, PAC-aware bridging, and process-targeted follow-on modules is what most strongly distinguishes this repo from a narrow research PoC.

---

## Bottom Line

`DarkSword-RCE` is best understood as a full exploit-and-post-exploitation framework, not a single isolated browser exploit.

The critical path is:

1. remote web staging,
2. JSC/WebKit memory corruption,
3. build-specific offset selection and slide normalization,
4. worker-based arbitrary memory R/W,
5. dyld / soft-link / PAC-assisted native call bridge,
6. large native payload execution,
7. process injection and collection-oriented actions.

For defenders, the most important takeaway is that the repository combines browser exploitation, PAC-aware native bridging, and multi-process post-exploitation in one chain, which substantially raises both sophistication and operational risk.

🧩 In practical terms, the malware is valuable to analysts because it preserves the full transition from delivery, to memory corruption, to native bridge, to operational payloading in one place.
