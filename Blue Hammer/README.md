# Blue Hammer Analysis Write-Up

### ⚠ Disclaimer
The analyzed malware was tested on `18-04-2026 (worked)` and `24-04-24 (no longer worked)`. On `Windows 10 Build 21332 x64`, the exploit chain no longer reproduced reliable end-to-end privilege escalation now.

This document is provided strictly for **educational, research, and defensive security purposes**.

I (the author) assumes **no responsibility or liability** for any misuse, abuse, damage, or illegal activity resulting from the use, interpretation, or redistribution of this material. Any actions taken based on this analysis are done solely at the reader’s own risk.

## Scope

This report covers static and dynamic analysis of `Blue Hammer` and its source in this workspace:

- Binary: `x64\Release\FunnyApp.exe`
- Main source: `FunnyApp.cpp`
- RPC definitions: `windefend.idl`, `windefend_h.h`, `windefend_c.c`, `windefend_s.c`

The sample is not treated here as benign software. Based on source and binary behavior, it is a proof-of-concept exploit chain targeting Microsoft Defender update handling and using the resulting access to leak protected files from a volume shadow copy, then abuse recovered credential material for user/session compromise.

## Build

**Requirements:**
- Visual Studio 2022 (v143 toolset)
- Windows SDK 10.0.26100.0
- x64 target platform

```
Open FunnyApp.sln
Select: Release | x64
Build -> Build Solution
Output: x64\Release\FunnyApp.exe
```

**Note:** The pre-built `x64\Release\FunnyApp.exe` is from the original repository and is
untrusted. Always build from source.

## Summary

`FunnyApp.exe` is an unsigned 64-bit Windows console executable that implements a multi-stage local exploit PoC branded in the repository as `BlueHammer`.

At a high level, the program:

<img width="651" height="916" alt="BlueHammer" src="https://github.com/user-attachments/assets/2e9f43dd-e763-45c7-a8d2-f63cbd664355" />

- Waits for or identifies a Microsoft Defender definition update.
- Downloads the official Defender update package and extracts its CAB contents.
- Triggers Defender/VSS activity with an `EICAR` test file.
- Uses Cloud Files callbacks, oplocks, directory change monitoring, a junction, and an Object Manager symbolic link to redirect Defender’s update file access into a chosen file inside a newly created volume shadow copy.
- Opens the leaked file from Defender’s definition update directory and writes a copy to `%TEMP%`.
- Uses the leaked `SAM` hive to decrypt local NTLM password hashes, temporarily changes user passwords, and launches shells as those users.
- If one of those users is an administrator, it creates a temporary service to bounce into `LocalSystem`, then re-launches a console in the original interactive session.

## Artifact Details

- File: `x64\Release\FunnyApp.exe`
- SHA-256: `FBEBC650D02CD9ADCED6D94F7D527DEDD0E1378B34CD0D660224C6D09C20D781`
- Size: `354,816` bytes
- Type: PE32+ / x64 console executable
- Authenticode: not signed
- Manifest: `asInvoker`

### PE Metadata

- Machine: `0x8664` (`x64`)
- Entry point RVA: `0xAE44`
- Image base: `0x140000000`
- Subsystem: Windows CUI
- DLL characteristics: `HighEntropyVA`, `DynamicBase`, `NX-Compatible`, `TerminalServerAware`
- Linker version: `14.44`

### Sections

| Section | Raw Size | Entropy | Notes |
|---|---:|---:|---|
| `.text` | 152064 | 6.429 | Main code |
| `.rdata` | 178688 | 4.164 | Strings, constants, import data |
| `.data` | 3072 | 2.137 | Globals |
| `.pdata` | 9728 | 4.905 | Unwind metadata |
| `.rsrc` | 512 | 4.712 | Manifest resource only |
| `.reloc` | 9728 | 5.433 | Relocations |

No evidence of packing or section-level obfuscation was observed from the section layout and entropy alone.

## Key Imports

The import set is highly consistent with the exploit chain:

- `WININET.dll`
  - `InternetOpenW`, `InternetOpenUrlW`, `InternetReadFile`, `HttpQueryInfoW`
- `wuapi.dll` / COM update interfaces
  - `CoCreateInstance`, `CLSIDFromProgID`
- `RPCRT4.dll`
  - RPC client support for Defender ALPC/RPC calls
- `cldapi.dll`
  - `CfRegisterSyncRoot`, `CfConnectSyncRoot`, `CfExecute`, `CfDisconnectSyncRoot`, `CfUnregisterSyncRoot`
- `Cabinet.dll`
  - CAB extraction routines
- `OFFREG.dll`
  - Offline registry hive access (`SAM`)
- `ADVAPI32.dll`
  - `LogonUserExW`, `CreateServiceW`, `CreateProcessWithLogonW`, crypto, SAM/LSA-facing APIs
- `ntdll.dll`
  - `NtCreateFile`, `RtlInitUnicodeString`

This import mix is unusual for normal software but expected for a Defender-focused local exploit plus offline credential extraction utility.

## Static Analysis

### 1. Defender RPC binding and update trigger

The sample builds an `ncalrpc` binding to a Defender RPC endpoint and invokes `Proc42_ServerMpUpdateEngineSignature`:

<img width="905" height="517" alt="{02989146-B2BA-46F9-B905-1655A8A73D8E}" src="https://github.com/user-attachments/assets/0d2a80fe-7035-4926-901b-9501195e5072" />

- The endpoint UUID is `c503f532-443a-4c69-8300-ccd1fbdb3839`
- The endpoint name is `IMpService77BDAF73-B396-481F-9042-AD358843EC24`

The code comment directly states the PoC may fail with Defender-specific status codes and should wait for the “right update,” which reinforces that this is exploit logic rather than legitimate administration code.

### 2. Windows Update and Defender package retrieval

The sample first enumerates updates through COM:

<img width="754" height="486" alt="{E47A784F-D5A4-4C17-9F32-5C24E1AEB635}" src="https://github.com/user-attachments/assets/fab30de7-c6ec-4b18-b051-f185e838f07c" />

Polls Windows Update Agent COM API (`Microsoft.Update.Session`) every 30 seconds,
searching for updates categorized as both `"Microsoft Defender Antivirus"` AND
`"Definition Updates"`. The exploit cannot proceed without a pending update.

**Dependency:** If Defender definitions are current, the exploit blocks here indefinitely.
In lab testing, you may need to block auto-update or roll back definitions to create
a pending state.

**COM flow:**
```
CoCreateInstance(Microsoft.Update.Session)
  -> IUpdateSession::CreateUpdateSearcher
    -> IUpdateSearcher::Search("")
      -> ISearchResult::get_Updates
        -> iterate: check IUpdate::get_Categories for matching category names
```

<img width="882" height="432" alt="{F70C77A8-A13F-4F8B-BC85-D1C76F06FEF9}" src="https://github.com/user-attachments/assets/23acb18c-125d-4dcb-8979-f41b638fbcf9" />

Downloads Defender update stub from Microsoft CDN:
```
InternetOpen(L"Chrome/141.0.0.0", INTERNET_OPEN_TYPE_DIRECT, ...)
InternetOpenUrl(hint, L"https://go.microsoft.com/fwlink/?LinkID=121721&arch=x64", ...)
```

The downloaded executable is parsed in memory to recover an embedded CAB, then extracted via FDI:

The downloaded file is `mpam-fe.exe` (Microsoft's Defender update self-extractor, ~100MB).
The PoC does NOT execute it. Instead:

1. `GetCabFileFromBuff()` parses the PE header manually, walks section table,
   finds `.rsrc` section, traverses resource directory tree to extract embedded cabinet data
2. FDI (Cabinet API) extracts cab contents into memory via custom callbacks:
   - `CUST_FNOPEN/READ/WRITE/SEEK/CLOSE` — all operate on in-memory buffers, no disk I/O
   - `CUST_FNFDINOTIFY` — captures each file, skips `MpSigStub.exe`
3. Returns linked list of `UpdateFiles` structs (filename, buffer, size)

### 3. VSS and Defender trigger path

The function `TriggerWDForVS` creates a GUID-named temporary directory, drops an `EICAR` test file, and uses an oplock on `RstrtMgr.dll` as part of timing/orchestration:

<img width="791" height="650" alt="{E910FF13-7211-4978-BD6F-8F6CA47A6E1E}" src="https://github.com/user-attachments/assets/94ba3cb0-0751-40bb-8646-b07ba6a8f46b" />

<img width="756" height="185" alt="{1FE7DBFB-F3CE-4C31-AF56-003D97216342}" src="https://github.com/user-attachments/assets/b18cdf0b-d681-4a41-81aa-66828db72dce" />

Notable actions:

- Creates `%TEMP%\{GUID}\foo.exe`
- Writes reversed `EICAR` content into the file
- Opens `%windir%\System32\RstrtMgr.dll`
- Requests `FSCTL_REQUEST_BATCH_OPLOCK`
- Opens the EICAR file to trigger Defender attention
- Starts worker threads:
  - `ShadowCopyFinderThread`
  - `FreezeVSS`

The dedicated thread `ShadowCopyFinderThread` watches the Object Manager namespace for a newly created `HarddiskVolumeShadowCopy*` object and then attempts to open `\Device\HarddiskVolumeShadowCopyX\Windows`:

<img width="738" height="536" alt="{54C7D17A-E386-4A25-AE7B-3AF9E0A5323D}" src="https://github.com/user-attachments/assets/cdf5194f-d6ec-40d4-9ca9-b77f4956be09" />

<img width="819" height="635" alt="{96EBFCEE-90E2-40C1-A2C0-8FCEAED07EBC}" src="https://github.com/user-attachments/assets/00d9996a-42b6-4eca-bd1c-5e32692e5ba9" />

This is the sample’s mechanism for discovering a freshly created VSS path to later target.

### 4. Cloud Files abuse to freeze/manipulate Defender access

**Objective:** Keep Defender frozen on the VSS snapshot while the exploit completes.

1. `CfRegisterSyncRoot()` — registers exploit directory as Cloud Files sync root
   - Provider: `L"IHATEMICROSOFT"`, Version: `L"1.0"`
   - Policy: `CF_HYDRATION_POLICY_PARTIAL` with `CF_HYDRATION_POLICY_MODIFIER_VALIDATION_REQUIRED`

2. Creates placeholder `.lock` file (GUID-named)

3. `CfConnectSyncRoot()` with flags:
   - `CF_CONNECT_FLAG_REQUIRE_PROCESS_INFO` — enables PID identification in callbacks
   - `CF_CONNECT_FLAG_REQUIRE_FULL_FILE_PATH`

<img width="556" height="455" alt="{541B2E74-25A8-422E-8D2D-2821BCDCAC67}" src="https://github.com/user-attachments/assets/657d8265-15d8-49ef-bde0-55e78cefa599" />

4. Callback `CfCallbackFetchPlaceHolders` (line 1370):
   - Checks `CallbackInfo->ProcessInfo->ProcessId` against `GetWDPID()`
   - `GetWDPID()` (line 1344) queries SCM: `OpenService("WinDefend")` -> `QueryServiceStatusEx`
   - If caller is Defender: signals event, creates oplock on .lock file, responds with placeholder
   - Defender tries to hydrate/access .lock file, hits oplock, freezes
  
<img width="646" height="697" alt="{3CABEF62-DB7F-475F-A585-C26ED3A22B83}" src="https://github.com/user-attachments/assets/d7bfffd7-f76e-486c-a142-21b6661941f1" />

5. VSS remains accessible while Defender is frozen

**This is the most novel technique in the exploit.** Cloud Files API is designed for
cloud storage sync engines (OneDrive, Dropbox). Using it for process identification
and selective freezing is creative and currently undermonitored.

Behavior:

- It checks whether the callback request originated from the Defender service process via `GetWDPID()`.
- If Defender is the requester, it signals an event and issues `CfExecute` with a placeholder for a GUID-named `.lock` file.
- It then places an oplock on that lock file and waits for Defender interaction.
- The code prints `WD is frozen and the new VSS can be used.` once the sequence completes.

This appears to be a synchronization/race primitive to hold Defender in a useful state while the file-system redirection stage is set up.

### 5. Junction and Object Manager symbolic link redirection

The core file-redirection stage happens in `wmain`:

<img width="878" height="378" alt="Screenshot 2026-05-19 004009" src="https://github.com/user-attachments/assets/e9860d5f-c5ed-4927-894b-a9b1bf7fe54c" />

<img width="873" height="357" alt="{1368AF4D-635C-4C1A-A5BE-86C1B3211400}" src="https://github.com/user-attachments/assets/cb7379b5-a1da-49e7-a5f0-82bbbd1974fe" />

1. Update files written to `%TEMP%\<GUID>\` on disk
2. RPC call: `Proc42_ServerMpUpdateEngineSignature(bindhandle, NULL, updatepath, &errstat)`
   - UUID: `c503f532-443a-4c69-8300-ccd1fbdb3839`
   - Endpoint: `ncalrpc:IMpService77BDAF73-B396-481F-9042-AD358843EC24`
3. Defender creates new directory under `C:\ProgramData\Microsoft\Windows Defender\Definition Updates\`
4. Oplock on `mpasbase.vdm` — Defender pauses when reading this file

<img width="767" height="417" alt="{C77BB61E-92A0-4EBB-8FE3-2E8588CCDA65}" src="https://github.com/user-attachments/assets/88ff6713-96b8-40cb-986e-02a0bb8c4852" />

**While Defender is paused:**

5. Rename `%TEMP%\<GUID>` -> `%TEMP%\<GUID>.foo`
6. `NtCreateFile` recreates `%TEMP%\<GUID>` as empty directory with `FILE_DELETE_ON_CLOSE`
7. `FSCTL_SET_REPARSE_POINT` sets junction: `%TEMP%\<GUID>` -> `\BaseNamedObjects\Restricted`
8. `NtCreateSymbolicLinkObject`: `\BaseNamedObjects\Restricted\mpasbase.vdm` -> `\Device\HarddiskVolumeShadowCopyN\Windows\System32\Config\SAM`

Defender (running as SYSTEM) reads the SAM file thinking it's a signature database,
copies it to Definition Updates directory.

This is the exploit centerpiece. Instead of allowing Defender to read the expected VDM update file, the PoC redirects that path to a protected file inside the shadow copy.

### 6. File leak and persistence of stolen material

Once the redirect succeeds, the sample repeatedly tries to open the resulting file inside Defender’s own update directory:

<img width="825" height="360" alt="{575ADBF7-6841-451F-916E-5E31F9363AC5}" src="https://github.com/user-attachments/assets/65df2650-0b42-4a99-8256-fb26cde8a3cb" />

It then:

<img width="591" height="258" alt="{9B6D0A4B-CBF8-4003-8BBD-B967389DCC42}" src="https://github.com/user-attachments/assets/ee36855a-f908-4050-be97-19e73d1b6347" />

- Reads the file contents into memory
- Writes a copy to `%TEMP%\{GUID}`

<img width="519" height="212" alt="{EE9FA813-7B54-4506-8975-20A37A9AC407}" src="https://github.com/user-attachments/assets/854ff5b7-88dc-4185-b619-11cdebe4277c" />

- Prints `Exploit succeeded.`
- Prints `SAM file written at : %ws`

The current code is configured to leak only:

- `\Windows\System32\Config\SAM`

### 7. Offline SAM processing and credential abuse

The post-exploitation portion is substantial and not incidental.

Functions:

- `GetLSASecretKey`:
- `UnprotectPasswordEncryptionKey*`:
- `UnprotectNTHash`: 
- `ChangeUserPassword`: 
- `DoSpawnShellAsAllUsers`: 

Capabilities:

<img width="900" height="458" alt="image" src="https://github.com/user-attachments/assets/4f1601ba-15ad-4b8b-b57e-6f498fa11873" />


- Reads the local boot key from `HKLM\SYSTEM\CurrentControlSet\Control\Lsa`
- Opens the leaked `SAM` hive with `OFFREG`
- Decrypts per-user NTLM hashes
- Enumerates local user accounts
- Skips current user and `WDAGUtilityAccount`

<img width="486" height="251" alt="{8B33C225-B5C5-4EA6-9759-2AE96E593A60}" src="https://github.com/user-attachments/assets/9ee2b4c3-b1ec-4f7f-b3e3-27bf3d3b1e20" />

- Temporarily changes passwords to:
  - `$PWNed666!!!WDFAIL`
- Logs on as those users
- Launches `conhost.exe` under those credentials
- Restores the original NTLM hash afterward

If an obtained account is an administrator, the sample:

- Impersonates the user
- Creates a transient service whose binary path points back to the sample itself with the current session ID
- Starts the service so the program re-executes as `LocalSystem`
- Deletes the service

<img width="860" height="440" alt="{D7F00C99-B76D-46CD-A370-3783C8F66220}" src="https://github.com/user-attachments/assets/abab87cd-e011-4f4e-80a0-b83cd6e50dbc" />

The SYSTEM instance calls `LaunchConsoleInSessionId`:
- Duplicates SYSTEM token
- Sets session ID to user's desktop session
- `CreateProcessAsUser` -> `conhost.exe` as SYSTEM in user's session

### 8. MITRE ATT&CK Mapping

| Tactic | Technique | Sub-Technique | BlueHammer Stage |
|---|---|---|---|
| Reconnaissance | — | — | — |
| Resource Development | — | — | — |
| Initial Access | — | — | Not applicable (requires local execution) |
| Execution | T1569 | .002 Service Execution | Stage 7: Temp service start |
| Persistence | — | — | None (single-run) |
| Privilege Escalation | T1068 | — Exploitation for Privilege Escalation | Stages 3-5: Oplock+junction chain |
| Privilege Escalation | T1543 | .003 Windows Service | Stage 7: Service creation for SYSTEM |
| Defense Evasion | T1562 | .001 Disable or Modify Tools | Stage 4: Freezing Defender via Cloud Files |
| Defense Evasion | T1574 | .005 Executable Installer File Permissions Abuse | Stage 5: Junction+symlink redirect |
| Credential Access | T1003 | .002 SAM | Stage 6: SAM extraction via VSS |
| Credential Access | T1552 | .002 Credentials in Registry | Stage 6: LSA boot key from registry |
| Credential Access | T1098 | — Account Manipulation | Stage 6: Password change via SamiChangePasswordUser |
| Lateral Movement | — | — | Not applicable (local only) |
| Collection | T1005 | — Data from Local System | Stage 5: SAM file exfiltration from VSS |

## Dynamic Analysis

### Test Environment Observations

During analysis on this FlareVM:

- `WinDefend` was running normally
- `Get-MpComputerStatus` reported:
  - `AMRunningMode : Normal`
  - `AMServiceEnabled : True`
  - `AntivirusEnabled : True`
  - `RealTimeProtectionEnabled : True`
  - `BehaviorMonitorEnabled : True`
  - `IsTamperProtected : True`
- Existing Defender update directories were present under:
  - `C:\ProgramData\Microsoft\Windows Defender\Definition Updates`

This environment matters because the exploit chain depends heavily on live Defender behavior, Defender update activity, and VSS timing. In practice, the sample behaved very differently on this Defender-enabled VM than on prior disabled-Defender test setups.

### Execution Results

The sample was executed multiple times from:

- `C:\Users\PC02\Downloads\BlueHammer\FunnyApp.exe`
- `C:\Users\PC02\Downloads\BlueHammer\x64\Release\FunnyApp.exe`

Observed runtime behavior:

- Each visible run spawned a child `conhost.exe` process.
- Loaded modules included:
  - `WININET.dll`
  - `wuapi.dll`
  - `Cabinet.dll`
  - `OFFREG.dll`
  - `cldapi.dll`
  - `RPCRT4.dll`
- Successful early-stage runs printed output consistent with the update and bait-file path, including:
  - `Checking for windows defender signature updates...`
  - `Found Update :`
  - `Downloading updates...`
  - `Cab file content extracted.`
  - `Updates downloaded.`
  - `Creating VSS copy...`
- The sample repeatedly created GUID-named working directories under `%TEMP%` and dropped:
  - `%TEMP%\{GUID}\foo.exe`

Representative Defender telemetry captured during these runs:

- `2026-04-24 05:32:48`
  - Event `1116`
  - `Virus:DOS/EICAR_Test_File`
  - `Path: file:_C:\Users\PC02\AppData\Local\Temp\2cc69c20-9f91-4767-99de-38c63f48ee0f\foo.exe`
  - `Process Name: C:\Users\PC02\Downloads\BlueHammer\x64\Release\FunnyApp.exe`
- `2026-04-24 05:57:10`
  - Event `1117`
  - `Action: Quarantine`
  - `Action Status: To finish removing malware and other potentially unwanted software, restart the device.`

Other runs produced additional Defender detections against the sample itself, including:

- `Exploit:Win64/DfndrPEBluHmr.DAA!MTB`
- `Exploit:Win32/DfndrPEBluHmr.BB`

### VSS-Related Runtime Findings

Dynamic testing showed that the shadow-copy stage is timing- and environment-dependent rather than guaranteed.

In earlier runs, stdout showed:

- `No volume shadow copies were found.`
- `Waiting for oplock to trigger...`
- `Oplock triggered.`
- `Failed to get new volume shadow copy path`

This matched the source logic in `TriggerWDForVS` and `ShadowCopyFinderThread`: the sample only proceeds if a brand-new `\Device\HarddiskVolumeShadowCopy*` appears after the watcher thread starts and `\Windows` inside that shadow copy becomes accessible before the main thread checks the worker result.

In a later visible run on this VM, the operator observed that the sample progressed past the shadow-copy-path stage instead of failing there. Based on the source, that means the following condition was satisfied during that run:

- a new `HarddiskVolumeShadowCopy*` object was created and detected during the EICAR/oplock window
- the sample successfully opened `\Device\HarddiskVolumeShadowCopyX\Windows`

That observation is important because it confirms the VSS-dependent stage is operational in at least some environments and is not merely dead code.

### Post-VSS Expectations

Passing the shadow-copy discovery stage does not by itself guarantee a `LocalSystem` console.

Per source, the remaining steps are:

- freeze Defender using the Cloud Files callback/oplock path
- redirect Defender update-file access into the chosen shadow-copy target
- leak `SAM`
- process the hive offline
- obtain an admin-capable local account context
- create/start a transient service
- relaunch back into the interactive session as `LocalSystem`

The code is therefore intended to culminate in interactive `LocalSystem` execution, but successful shadow-copy discovery alone only confirms the exploit chain advanced into its later stages.

### Interpretation of Dynamic Results

What was verified:

- The binary is operational and long-lived rather than crashing immediately.
- It successfully reaches live Defender interaction on this VM.
- It creates the expected `%TEMP%\{GUID}\foo.exe` bait artifact.
- Defender detects that bait artifact in real time.
- Defender can also classify the sample itself as a Defender exploit in some runs.
- At least one run advanced beyond the previously observed `Failed to get new volume shadow copy path` checkpoint.

What was not fully observed end-to-end on this VM:

- No confirmed leaked `SAM` output file was captured during the documented runs
- No confirmed password reset behavior was captured live
- No confirmed transient service creation was captured live
- No confirmed interactive `LocalSystem` console was captured live during the documented runs

The most likely explanation for the mixed outcomes is not absence of Defender, but environmental sensitivity and race timing:

- availability of a qualifying Defender definition update
- whether Defender creates a new VSS instance during the bait-file handling window
- whether Defender remediation of `foo.exe` or the sample itself interrupts later stages
- whether the post-leak account/service stages succeed on the specific VM state

## Behavior Summary

### Initial access and prerequisites

- Requires local execution
- Designed to run as a normal user first (`asInvoker`)
- Appears intended to escalate by abusing Defender internals rather than requiring initial admin rights

### Main actions

- Enumerates Defender definition updates
- Downloads official Defender update package
- Extracts update CAB content in memory
- Triggers Defender scanning and/or update activity
- Watches for new VSS instances
- Uses Cloud Files and oplocks as synchronization primitives
- Redirects Defender file access into protected shadow-copy files
- Exfiltrates `SAM`
- Decrypts NTLM hashes offline
- Changes passwords to gain user execution
- Uses service creation to obtain `LocalSystem` execution

### Likely goals

- Read protected registry hives without direct privileged file access
- Obtain local account credential material
- Achieve lateral movement inside the local host across user contexts
- Reach `LocalSystem`

## Notable IOCs and Artifacts

### Network / URLs

- `https://go.microsoft.com/fwlink/?LinkID=121721&arch=x64`

### Files and paths

- `%TEMP%\{GUID}\foo.exe`
- `%TEMP%\{GUID}\*.lock`
- `%TEMP%\{GUID}\mpasbase.vdm`
- `%TEMP%\{GUID}` output copy of leaked file
- `C:\ProgramData\Microsoft\Windows Defender\Definition Updates\{GUID}\mpasbase.vdm`
- `\BaseNamedObjects\Restricted\mpasbase.vdm`
- `\Device\HarddiskVolumeShadowCopy*\Windows\System32\Config\SAM`

### Service and account abuse

- Temporary service with GUID-like name
- Password reset value:
  - `$PWNed666!!!WDFAIL`

### Strings of interest

- `ServerMpUpdateEngineSignature`
- `IMpService77BDAF73-B396-481F-9042-AD358843EC24`
- `Microsoft Defender Antivirus`
- `Definition Updates`
- `WD is frozen and the new VSS can be used`

## Risk Assessment

This sample should be treated as high risk.

Reasons:

- It targets security product internals.
- It deliberately attempts protected file disclosure.
- It contains post-exploitation credential handling and account abuse code.
- It includes a path to `LocalSystem`.
- It is operationally closer to a weaponized PoC than a harmless research stub.

## Conclusions

From my perspective, `FunnyApp.exe` is a local exploit PoC with built-in post-exploitation functionality. The core exploit logic is the Defender/VSS/file-redirection chain, but the program goes much further by turning the leaked `SAM` into reusable credential material and interactive execution in other user contexts.

The static analysis is strong enough to characterize the intent and expected behavior with high confidence. Dynamic analysis on a Defender-enabled VM showed two materially different outcomes over time.

On `2026-04-18`, the sample was observed progressing through update retrieval, shadow-copy discovery, Defender freezing, and later-stage post-exploitation logic, culminating in an interactive shell under the local `Administrator` account. That demonstrates that the exploit chain was operational enough, at that time, to achieve administrator-level execution on the test system.

On `2026-04-24`, later runs no longer reproduced that earlier outcome reliably. Defender still detected the EICAR bait and, in some runs, also detected the sample/process itself. Although some runs continued to progress into the VSS-dependent phase, the observed executions terminated or stalled before recreating the earlier interactive administrator shell. Based on the available evidence, the most defensible conclusion is that this exploit was observed working on `2026-04-18`, but on the current Defender state/signature set as of `2026-04-24` it no longer completed reliably on this VM.


## References

- BlueHammer PoC source code (analyzed repository)
- Microsoft Cloud Files API: https://learn.microsoft.com/en-us/windows/win32/cfapi/
- Microsoft Offline Registry Library (offreg.h)
- EICAR test standard: https://www.eicar.org/download-anti-malware-testfile/
- SAM database structure and SYSKEY derivation: Moyix blog series
- MITRE ATT&CK: https://attack.mitre.org/
