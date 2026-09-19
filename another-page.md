---
layout: default
title: CSP Anatomy
tags:
  - csp
  - anatomy
  - registry
---

# CSP Anatomy

The component-level counterpart to Configuration Service Provider, which covers what a CSP *is*. This note covers **parts**: how a URI resolves to code, where that code lives, where the schema lives, and which stores hold state.

> A CSP is a **registration, a COM class, and whatever subsystem it fronts**. There is no CSP file, folder, process, or service to find.

## The defining contrast: nothing to point at

Ask "where is the Wi-Fi CSP?" and there is no satisfying answer — no `WifiCSP.exe`, no install directory, no service. The CSP exists as a **registry registration that names a COM class**, and the work it does happens inside whatever subsystem it fronts.

The comparison worth holding is the one Configuration Service Provider already draws — CSP as the MDM equivalent of a Group Policy **client-side extension** — because the two are registered along strikingly similar lines, and the differences are where the diagnostics diverge:

| | CSP | Group Policy CSE |
|---|---|---|
| Registration | `Provisioning\CSPs\`, keyed by URI root | `Winlogon\GPExtensions\{CSE GUID}` |
| Names its code as | A COM class id, resolved onward to a DLL | A `DllName` value, stated directly |
| Schema | DDF XML — **published, not on the device** | ADMX/ADML — on the device in `PolicyDefinitions` |
| Addressed by | OMA-URI | Registry policy key |
| Scope split | Separate registrations per context root | Separate User/Computer halves of one GPO |
| Runs as | In-proc, inside its caller | In-proc, inside `gpsvc` |
| "Which file is it?" | Two lookups away, varies by build | Named in the key |

**Practical consequence:** "the CSP isn't installed" is never the diagnosis. The equivalent failure is *the node does not exist on this build* — a versioning question, not a deployment one.

## Registration — how a URI becomes code

This chain is the whole mechanism, and it is the answer to what physically constitutes a CSP:

```
OMA-URI    ./Device/Vendor/MSFT/Policy/Config/Update/ManagePreviewBuilds
   │
   │  longest-prefix match against the registered roots in
   ▼
HKLM\SOFTWARE\Microsoft\Provisioning\CSPs\
   └── .\Device\Vendor\MSFT\Policy          ← one key per CSP, per context root
          │
          │  the key names a COM class id
          ▼
HKLM\SOFTWARE\Classes\CLSID\{…}\InprocServer32
          │
          ▼
%SystemRoot%\System32\<provider>.dll
```

Three things follow from the shape of it:

- **The registry is the address space.** DDF declares what a CSP *should* expose; this key decides what actually resolves on this machine. A URI that resolves nowhere in `Provisioning\CSPs` returns a not-found status before any CSP code runs.
- **Context roots are separate registrations.** `.\Device\Vendor\MSFT\Policy` and `.\User\Vendor\MSFT\Policy` are distinct keys. A CSP registered only under one root genuinely does not exist in the other — this is the mechanical basis for the dual-context rules in Context Roots - Device and User.
- **Value names under each key have shifted across builds.** Enumerate them rather than memorising; the check below does this.

> **Provenance**
> Microsoft confirms **that per-CSP registration keys exist in the registry**, in an unexpected place — the **DeviceManageability CSP** page, which states that for performance it "directly reads the CSP version from the registry" and that *"The `csp_version` is a value under each of the CSP registration keys."* It never gives their path.
>
> So `csp_version` is the one value name with documented backing. The **location** (`Provisioning\CSPs`, keyed by URI root) and the **COM resolution step** through `CLSID\{guid}\InprocServer32` are practitioner knowledge — verifiable on any device with the check below, but absent from Microsoft's documentation. Treat the mechanism as sound and the specific path as something to confirm on the build in front of you.
>
> That same page is also a reminder the registry is not a passive mirror: Microsoft notes the CSP's own `GetProperty` implementation had to be **updated to read from the registry too**, "so that both the paths return the same information." Two independently maintained read paths, deliberately reconciled.

## The code — in-proc, hosted, never standalone

A CSP is loaded into whichever component is calling it. It has no process of its own:

| Caller              | When                                                                     |
| ------------------- | ------------------------------------------------------------------------ |
| `omadmclient.exe`   | During an OMA-DM session — see Session Lifecycle                     |
| The WMI bridge host | Local access via `root\cimv2\mdm\dmmap`, see MDM Bridge WMI Provider |
| `dcsvc`             | The declared configuration channel |

This mirrors the task-triggered model in MDM Client Anatomy, and produces the same shift in questioning:

- **Wrong question:** "Is the CSP service running?"
- **Right questions:** "Is this node registered on this build?" "Did the URI resolve?" "What status did the caller return?"

Because the CSP runs inside its caller, a CSP fault never appears as a crashed service. It appears as a status code attributed to the session — which is why Alerts and Status Codes is the triage surface, not Task Manager.

## The schema lives off the device

The DDF is published documentation, and the arrow runs the opposite way from what the word "schema" suggests:

```
CSP implementation  ──►  DDF published  ──►  docs · Intune's catalog · you
 (the rules live here)   (a description of      (consumers, all off-device)
                          what was built)
```

The rules — which nodes exist, which verbs each accepts, what type each takes — are **compiled into the CSP's own code**, behind the registration above. The DDF describes that behaviour after the fact. Nothing on the device ever reads one: if every DDF file vanished tomorrow, devices would behave identically.

This is the sharpest anatomical difference from Group Policy, and it runs against intuition. ADMX in `%SystemRoot%\PolicyDefinitions` genuinely *is* parsed at runtime, by `gpsvc` and by the policy editor. **ADMX is a real schema; DDF only looks like one.**

The files themselves live on each CSP's reference page on Microsoft Learn, plus the per-release archives Microsoft publishes so a build's added nodes can be diffed. Third-party MDM vendors read them to build their own policy UIs — and so does Intune, which is the whole of the next section.

**So how does the device decide what is legal?** Empirically, at request time. The status codes are the same rules the DDF documents, enforced dynamically:

| DDF declares | Enforced on device by | Failure |
|---|---|---|
| The node exists | Registration lookup, then the CSP's own node table | **404** |
| `AccessType` — which verbs are permitted | The CSP rejects the verb | **405** |
| `DFFormat` — the data type | The CSP rejects the payload | **415** |

See Alerts and Status Codes. This is also why a node can appear in the published DDF and still return 404: the DDF describes the CSP *as designed*, the device has only the code it shipped with, and nothing reconciles the two.

The on-device ways to ask "what is legal here" are therefore all indirect:

1. **The Policy CSP definition store** below — authoritative, but only for Policy CSP.
2. **A WMI bridge class** in `root\cimv2\mdm\dmmap` — its presence proves the projection exists on this build.
3. **A `Get` against an interior node** — returns its children, so the tree can be walked.
4. **Send it and read the status** — the empirical answer, and the only universal one.

## From DDF to device — how a setting actually arrives

The DDF governs *authoring*, then drops out entirely before anything is delivered:

```
1  Microsoft implements the CSP in a Windows build, and publishes its DDF
        │
2  Intune ingests the DDF service-side ──► the settings catalog
        │     name · description · DFFormat · allowed values · OMA-URI · supported builds
        │
3  You pick a setting in a profile and supply only a value
        │
4  Assignment compiles the profile into OMA-DM operations
        │
        ▼ ─────────── the DDF stops here ───────────
5  Next session: omadmclient.exe dials out — scheduled poll, or WNS-nudged
        │
6  Replace / Add commands arrive inside a SyncML envelope
        │
7  Client matches each LocURI against Provisioning\CSPs, loads the CSP in-proc
        │
8  CSP validates against its own compiled rules and returns a status
```

**The settings catalog is Intune reading the DDF so that you do not have to.** It supplies the OMA-URI and the format; you supply the value. A **custom OMA-URI** profile is the identical pipeline with steps 2 and 3 done by hand — which is precisely why type errors surface there and almost never in the catalog. Nothing is checking your DFFormat against the DDF, so the first thing to validate it is the CSP itself, and its answer is 415. Policy CSP makes the same point from the other side: the catalog is a presentation layer over CSPs.

The catalog also carries each setting's supported Windows builds, so it can warn you before you assign. A custom OMA-URI carries no such guard — the version mismatch surfaces as a 404 on the device instead, which is the versioning failure Configuration Service Provider describes.

**The payoff:** steps 5–8 consult nothing but the device's own registration and compiled code. Delivery does not carry a schema and the device never fetches one. Every guarantee you were given at authoring time was Intune's, made service-side — the device re-derives its answer independently, and when the two disagree, the device wins.

## Policy CSP is the exception — its schema *is* on disk

```
HKLM\SOFTWARE\Microsoft\PolicyManager\default\{Area}\{Policy}
```

This is the shipped **definition** store: every Area and Policy the build supports, present whether or not anything has ever been configured. It is distinct from the value stores that PolicyManager Registry covers, and the distinction is easy to lose:

| Branch | Holds | Present when |
|---|---|---|
| `default\` | Definitions — what this build supports | Always, shipped with the OS |
| `providers\{EnrollmentGUID}\` | What one provider asked for | Something was configured |
| `current\` | Merged effective values | Something is in force |

**Why it earns its keep:** `default\` is the only dependable on-device answer to *"does this policy exist on this Windows version?"* — the versioning failure named in Configuration Service Provider. A policy absent from `default` will never apply no matter how the profile is authored, and no amount of console retrying changes that.

## Where a CSP's state actually lands

For Policy CSP the answer is uniform, and PolicyManager Registry covers it. For every other CSP there is **no common store** — each writes into the subsystem it fronts:

| CSP | State lands in |
|---|---|
| `Wifi`, `VPNv2` | The network profile stores those services already use |
| `ClientCertificateInstall` | The certificate stores |
| `BitLocker` | Volume metadata and TPM state |
| `DevDetail`, `DeviceStatus` | Nowhere — computed on read, see CSP Categories |

What *is* uniform is the client's record of what it last reported upward, held in NodeCache under `Provisioning\NodeCache`. That store belongs to the MDM client, not to any CSP — a distinction that matters when a value is correct on the device but stale in the console.

> **Warning**
> Hand-editing values under `PolicyManager` to "fix" a policy desynchronises the device from NodeCache and from the server's view. The device will report the state it believes it last sent, not what you typed. Read these keys freely; change them through the CSP.

## Quick anatomy check

```powershell
# Every CSP registered on this build, by URI root
Get-ChildItem -Recurse HKLM:\SOFTWARE\Microsoft\Provisioning\CSPs |
  Select-Object -ExpandProperty Name

# ONE registration's values, when you already know which one you want.
# -LiteralPath matters here: the path contains a '.' segment that PowerShell
# would otherwise normalise away.
$k = 'HKLM:\SOFTWARE\Microsoft\Provisioning\CSPs\.\Device\Vendor\MSFT\Policy'
Get-ItemProperty -LiteralPath $k

# EVERY registration that carries a class id, with the value name each one used.
# Matched on shape rather than on value name, because the names vary by build —
# so this is where a CLSID comes from, with no path known in advance.
# Enumerating also sidesteps the '.' path segment that trips hand-typed paths.
$regs = Get-ChildItem -Recurse HKLM:\SOFTWARE\Microsoft\Provisioning\CSPs | ForEach-Object {
  $key = $_
  (Get-ItemProperty -LiteralPath $key.PSPath).PSObject.Properties |
    Where-Object { $_.Value -is [string] -and $_.Value -match '^\{[0-9A-Fa-f-]{36}\}$' } |
    ForEach-Object {
      [pscustomobject]@{
        Registration = $key.Name -replace '.*\\CSPs\\', ''
        ValueName    = $_.Name
        ClassId      = $_.Value
      }
    }
}
$regs | Format-Table -AutoSize

# Resolve each class id to the DLL behind it. HKCR is not mounted by default
# in PowerShell, so go via HKLM\SOFTWARE\Classes.
$regs | ForEach-Object {
  [pscustomobject]@{
    Registration = $_.Registration
    Dll = (Get-ItemProperty "HKLM:\SOFTWARE\Classes\CLSID\$($_.ClassId)\InprocServer32" `
             -ErrorAction SilentlyContinue).'(default)'
  }
}

# Does this build support the policy at all?
# The subkeys of an Area are its POLICY NAMES. Swap 'Update' for any other Area,
# or drop the Area entirely to list every Area this build knows.
#   name listed     -> supported by this build (says nothing about it being set)
#   name absent     -> no profile will ever apply it; expect a 404 or a silent no-op
#   path-not-found  -> the Area itself does not exist here, so check the Area name
Get-ChildItem HKLM:\SOFTWARE\Microsoft\PolicyManager\default\Update |
  Select-Object PSChildName

# Is the CSP projected into WMI on this build?
# Get-CimClass asks whether the CLASS EXISTS, not whether it holds data — use
# Get-CimInstance for values, and expect to need SYSTEM rather than just elevation.
#   classes listed   -> the bridge projection exists for those Areas
#   nothing returned -> no Area matches the wildcard on this build
#   namespace error  -> root\cimv2\mdm\dmmap is missing entirely, a larger finding
# Swap Config01 for Result01 to check the read-back surface instead of the write one.
Get-CimClass -Namespace root\cimv2\mdm\dmmap -ClassName MDM_Policy_Config01_* |
  Select-Object CimClassName
```

On the registry provider, `Get-Item` returns the **key** — whose `.Property` member lists value *names* — while `Get-ItemProperty` returns the **values**. Reaching for the first when you want a class id is the easy mistake: it produces plausible-looking output containing nothing you can act on.

The last two commands are **existence** checks, not state checks — one against the registry definition store, one against the WMI projection. Presence proves the build *supports* something; it never proves anything is *configured*. For that, read `current` (PolicyManager Registry) or the `Result/` branch (Policy CSP).

## What breaks, and how it shows

| Symptom | Anatomical cause | Where to look |
|---|---|---|
| Node returns not-found | No registration for that root on this build | `Provisioning\CSPs`, then Alerts and Status Codes |
| Node resolves, write rejected | `AccessType` forbids the verb | Access Types and Scope |
| Policy never applies, no error | Area/Policy absent from `PolicyManager\default` | Version mismatch, not delivery |
| Applies, then reverts | A second provider wrote after you | Compare `providers\{GUID}` against `current` |
| WMI bridge access denied | Running elevated, not as SYSTEM | MDM Bridge WMI Provider |

See also: Configuration Service Provider, MDM Client Anatomy, DDF - Device Description Framework
