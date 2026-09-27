# AgentOS AOSP bootstrap overlay

This directory contains the AgentOS source overlay for `android-15.0.0_r34`.
It includes neither an AOSP checkout nor proprietary device files. The wiring
and probe were saved in `62223a7`; subsequent commits add target selection,
checkout revision checks, external backups and wiring tests.

The native runtime slice provides:

- a versioned health, Session, automatic Jev selection, and Plugin-session AIDL
  with native `sideagentd` Binder service;
- a native HTTPS Worker for MiniMax-M3, a mode-0600 secret file contract, and
  durable state/recovery fencing under `/data/agent`;
- stable AIDL v2 Plugin endpoint protocol (`agentos_system_aidl-V2`) with
  structured handshake, capability grant, asynchronous tool/resource request,
  typed terminal result/error, cancellation, and host callback boundaries;
- init and SELinux bootstrap files;
- a control plane that discovers Plugin endpoint services from the manifest,
  checks package/UID/signer/version identity, persists per-user grants, handles
  user lifecycle events, binds with `BIND_AUTO_CREATE`, and creates a fresh
  session on each bind with disconnect retry;
- basic `cmd agentos health|plugins|enable|disable|runtime-test|runtime-snapshot`
  and `dumpsys agentos`
  diagnostics; control Binder methods currently require root or system UID;
- the real `AgentOsPluginProbe` test APK for discovery, binding, handshake and
  process-lifecycle checks; it has no model, MCP or tool/resource pipeline;
- explicit freezer verification invariants without a broad exemption.

## Plugin endpoint protocol boundary

Frozen API integrity is checked by `tools/ci/check-aosp-overlay.sh` using the
r34 hash algorithm. V2's checksum uses previous version `1`; the initial
checksum incorrectly used `latest-version`. The corrected checksum is appended
without changing any frozen interface definitions.

`agentos_system_aidl` version 1 remains frozen for the discovery probe and
existing control-plane clients. Version 2 contains the Plugin data-plane
boundary. The current development surface appends frontend Session methods and
an automatic `submitAutoInput` method;
the AIDL module is temporarily unfrozen until those methods are frozen as the
next public version. Version 2 transaction numbers remain unchanged:

- `openPluginSessionV2` negotiates `AgentPluginHostInfo` and receives an
  `IAgentPluginHostCallback` binder;
- `sessionGranted` delivers the immutable per-session tool/resource allowlist;
- `beginInvoke` and `beginReadResource` are `oneway` and carry structured
  request Parcelables plus an `IAgentPluginResultSink` callback;
- `cancelInvoke` and `closePluginSession` are best-effort `oneway` controls;
- host callbacks report resource changes, capability changes, and endpoint
  close requests.
- `ISideagentd` V2 receives `AgentPluginSession` from `AgentManagerService`,
  retains the endpoint Binder, and exposes matching invoke/resource/cancel
  calls to the system data plane.

This protocol and the native session router are validated with the local SDK
AIDL Java/NDK compiler and V1→V2 API checks. The pinned Cuttlefish target has
also passed an AOSP/Soong image build and boot/health probe; a clean-host
rebuild and the complete data plane are still pending. `AgentManagerService` now hands the endpoint Binder and granted names
to `sideagentd`; native `sideagentd` keeps the session, checks the granted tool
or resource name, and forwards asynchronous calls. Plugin endpoint
implementations still need to adopt V2, and attachment/PFD transport, lease
enforcement, deadline cancellation, Binder-death cleanup, and durable
side-effect records remain follow-up work in the Android daemon.

From the AgentOS repository root, preview and apply the complete wiring:

```bash
python3 tools/aosp/wire-platform.py /path/to/aosp
python3 tools/aosp/wire-platform.py /path/to/aosp --apply
```

The default `cuttlefish` target does not require shusky; use `--target pixel8`
explicitly for Pixel 8. The script checks the manifest default revision and
the participating projects' HEADs against the pinned tag, copies this overlay,
adds the product namespace and `sideagentd` package, allocates AID 1096,
adds the versioned Java dependency, starts `AgentManagerService` from
`SystemServer`, defines the binding permission and merges platform-private
SELinux policy. Existing files are backed up alongside the AOSP root under
`agentos-wiring-backups/`, outside Soong's source scan. Inspect the dry-run diff
before applying to a checkout with local changes. `prepare-overlay.sh` remains
a copy-only helper and does not perform these platform modifications.

The probe module is available to build explicitly as `AgentOsPluginProbe`;
wiring does not include it in `PRODUCT_PACKAGES`. The Agenriod and Notes APKs
are generated outside this repository; pass both paths with `--frontend-apk`
and `--notes-apk` to stage them under `system/agent/frontend/prebuilt/` and add
the product packages. The independent `demo-apps` root has a parallel
three-APK staging path using `--demo-alarm-apk`, `--demo-calendar-apk` and
`--demo-meeting-records-apk`, which installs the demos under
`system/agent/demo/prebuilt/`. Their presence as source or a staged prebuilt is not
evidence of a successful image boot. Wiring fixture tests verify target
selection, APK staging, rejection of wrong revisions, idempotence and backups;
they do not compile or boot Android.

As of 2026-09-23, the pinned `android-15.0.0_r34` Cuttlefish target has a
completed AgentOS image build and boot record. The custom instance reached
`sys.boot_completed=1`; `sideagentd`, both AgentOS Binder services, health, and
the discovery/handshake probe were checked. The official stock Cuttlefish
archives remain a separate host baseline and do not contain this overlay. The
synchronous handshake can still exhaust its two workers if endpoints never
return; full MCP/capability transport, native runtime requests, and the
remaining freezer matrix are follow-up work. For the reproducible packaging and
Demo smoke path, see
[`AOSP 镜像打包与 Demo 验证`](../../../docs/aosp-image-demo-quickstart.md) and the
[platform TODO](../aosp-todo.md).
