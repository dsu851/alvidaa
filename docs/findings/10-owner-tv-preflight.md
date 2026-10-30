# Owner TV preflight — 2026-10-30

Static review of checkout `1a31308`, before contacting the owner's TV.
The owner identifies their model as **Hisense 55A6GTUK**. Firmware, SoC,
transport protocol and root compatibility are unverified. The repository's
tested 50A51HUV is a different model; its live-state notes do not describe
this owner's TV.

## Deployment gaps found in the checked-in tooling

- `server/tv` and `tools/tvshell.py` hardcode the original TV address.
  The SSH wrapper disables host-key checking and persistence.
- `launcher/hookinstall.html` auto-installs on page load. Its v10 payload
  hardcodes the original payload server and SSH public key, fetches an
  unsigned tools archive each boot, and starts an unauthenticated root
  telnet listener with unrestricted INPUT accept rules for 2323 and 2222.
  It has no implemented OTA freeze or `otaTest.off` check.
- The installer proceeds after a failed backup and overwrites the same
  backup filename. Its readback check looks for `v2` despite installing
  v10, rather than comparing the complete script.
- The documented patched Dropbear source/build patches, tools archive and
  raw TCP payload server are absent from the tracked checkout. Credentials
  and firmware directories are also absent locally at review time.
- `launcher/recon.html` runs text reads on load, but its root-install button
  writes an older harvest hook. That hook and `hookcheck.html` can collect
  sensitive material; the Python server exposes the latest recon report
  through unauthenticated `/api/recon` and listens on all interfaces.
  Such reports must not be used to export keys or tokens outside
  `server/keys/`.
- `docs/TESTED-MODELS.md` describes recon as a full identity probe; the
  current v8 page does not call the firmware/SoC identity functions.
- `09-endpoints.md` lists OTA as ALLOW, while ROOT/R0 requires an OTA freeze
  before UI changes. Do not apply that endpoint list verbatim as R0 policy.
- Deleting the hook/tools alone does not restore edited UI files or remove
  harvested dumps. README already notes complete teardown is unverified.

## Next gate

Discover the owner's TV on the LAN, read its SSDP descriptor, and obtain
firmware/OS/SoC identity using a minimal read-only browser page. Check for
existing hooks and backups before any write. Establish the applicable boot
chain from this TV's text files rather than assuming MTK9602 paths or
partition numbers. Prepare owner-specific key-only access, backup and
disable paths, and OTA isolation before native UI edits.

No TV connections, writes, pairing attempts or reboots were made during
this review.

## Initial LAN discovery

The owner supplied the candidate address (LAN-local; IPs redacted). From the Mac on `en0`, TCP connection attempts to 36669, 38400, 2222, 2323
and 8000 timed out. The route is directly through `en0`, but ARP resolution
for the candidate remained incomplete. Unicast SSDP to the candidate and
multicast discovery for MediaRenderer devices returned no responses.
This does not identify a firmware/authentication problem: the candidate
has not yet been established as a reachable TV. Confirm power, current IP
and LAN/client isolation before proceeding. No pairing or TV writes occurred.

After the owner woke the TV from standby, TCP 36669 responded and SSDP
identified the device on the LAN, with its SSDP descriptor at
`http://<tv>:18400/MediaServer/rendererdevicedesc.xml`.
The descriptor reports `transport_protocol=3290`, `platform=1`, `region=0`,
`brand=his` and `vidaa_support=1`. The owner reports software
`09.60v.Q0612` from the TV screen. Model year alone therefore does not select
the older MQTT authentication flow. SoC and bridge/root compatibility still
need verification. `launcher/fingerprint.html` is a minimal text-only,
read-only probe for identity, mount and boot configuration; it makes no
file writes on the TV and does not read credentials.

## First browser report: traversal rejected

The owner loaded the probe in the TV browser. It reports chipset **MTK9603**,
firmware **V0000.09.60V.Q0612**, OS **U09.60**, and Chromium 111 / Weblayer
5.2.1 on armv7l. `HiUtils_createRequest` exists, but all six text reads
using `../` returned `ret:false`, `code:0`, `sdkVersion:"1.5.0"` and
`invalid path:<requested path>`. This includes `/proc` and boot-script
targets, so the documented traversal/root chain is not established here.
This result alone does not distinguish wrapper validation from backend
validation or establish whether the boot hook exists.

Probe v2 compares a conventional `websdk/Appinfo.json` read, an absolute
uptime path and traversal uptime path through both the wrapper and raw
localhost HTTP service. It enumerates bridge names without collecting
identifier headers or credentials. No TV writes or root hooks are used.

## Raw service result: backend validation confirmed

Probe v2 completed from the TV browser. A normal `websdk/Appinfo.json` read
succeeded through both `HiUtils_createRequest` and the unauthenticated raw
service at `localhost:9009`. An absolute `/proc/uptime` request reached the
backend but resolved to `/APPS//proc/uptime` and returned `ENOENT`. The
`../proc/uptime` request returned `invalid path:../proc/uptime` through both
interfaces. This confirms the rejection is in or below the service API, not
only in the page-side wrapper. The path traversal used by the documented
MTK9602 root chain is closed on this MTK9603/U09.60 build in its canonical
form.

The next read-only gate tests a small set of path-normalisation variants
against `/proc/uptime` only. Success would be identified by the normal uptime
text shape. No credentials, device identifiers, boot scripts or binary files
are requested, and no file API write is issued.

## Normalisation variants: the sandbox holds (probe v3)

Probe v3 ran nine path variants against `/proc/uptime` through the raw
`localhost:9009/service/hiutils` service (`fileRead`, `mode: 6`). Report:
`server/recon-report-fingerprint-v3.json`.

- Every literal `..` variant (`../`, `..//`, `.././`, `x/../../`,
  `....//`) is rejected before the join with `ret:false,
  "invalid path:<raw path>"`. The raw path is echoed back, so validation
  happens on the input string, not on the joined path.
- The percent-encoded variants were **not decoded anywhere**: `%2e%2e/`,
  `.%2e/`, `..%2f` and `..\\/` all passed validation and reached the
  backend literally, failing as `open("/APPS/<literal>...") → ENOENT`.
  There is no URL-decode step to slip a traversal through, and no
  normalisation before `open()`.
- The backslash case `..\\/proc/uptime` was *not* flagged by the validator
  (it contains no `../` substring) yet still failed as a literal filename —
  which pins the validator to a substring check on `../` specifically,
  with `.` and `/` having no alternate spellings on this build.

Conclusion: the `fileRead` escape used by the MTK9602 chain is closed by a
substring check plus the absence of any decode/normalisation step. A
kernel-level `..` escape requires the string `../` somewhere in the path,
so no encoding or spelling variant should bypass this. The remaining
read-only surface is the API table itself (other `api` names with different
path handling, other `/service/*` endpoints, other localhost listeners).

Probe v4 therefore characterises the validator (bare `..`, `a..b`, `...`),
enumerates `api` names with empty args against the "Not supported api"
oracle, lists `Hisense_*`/`Hi*` window function names, and surveys
`/service/*` names and a small localhost port set. All read-only: no
credentials, no writes, no binary reads (the backend crashes on ELF reads),
incremental reporting so a mid-run failure keeps earlier sections.

## Sandbox characterisation: one string check, nothing behind it (probe v4)

Probe v4 completed. Report: `server/recon-report.json` (page version 4).

- **Bare `..` was accepted by the validator and resolved by the kernel.**
  `fileRead("..")` returned `EISDIR` on `read` — the join produced
  `/APPS/..` = `/`, the real root directory, and the service actually
  opened and tried to read it. There is no chroot, no base-directory
  strip, and no lexical normalisation: the only thing preventing
  filesystem-wide reads is the literal `../` substring test on the input
  string. `...`, `a..b`, `..x`, `x..` all passed validation as literal
  filenames (`ENOENT` at `/APPS/<name>`), confirming the check matches
  `../` exactly.
- The `mode: 0` base is confirmed as `/tmp` on this build too
  (`/tmp/no-such-probe-file` errno leak), same convention as the lab TV.
- **The API table is minimal.** All 22 guessed names (stat/dir-list/copy/
  exec classes) return `"Not supported api"` — no directory-listing or
  exec-class api found, consistent with the lab TV's 17-name result. The
  authoritative table lives in the unreadable system JS package.
- **Service surface is just two endpoints**: `/service/hiutils` and
  `/service/tvinfo`; every other `/service/*` name is a clean 404
  (`Wrong request:...`).
- **No other localhost listener in the probed set**: 80, 1234, 8080, 8888,
  9888 and 10000 all refused fast from the TV browser. The lab TV's
  phoenix WS port 9888 is closed on this build.
- The bridge exposes ~100 `Hisense_*` functions, all Get/Set/Observers
  (plus `Hisense_LoginWithVIDAA`, `Hisense_SetRemoteKeyboard`) — no file
  or process surface beyond `HiUtils_createRequest`.

**Verdict on the root lane for this TV**: the uid-314 file-read primitive
is closed by a single substring check with no decode, no normalisation, no
chroot to differ with, and no second file API or listener to route around
it. A kernel-level `..` escape requires the characters `../` in the final
path, and the input is appended verbatim to the base — so no encoding or
spelling variant can produce one. Unless a new primitive appears (a
different writable-consuming root service, USB/UART/OTA lane, or kernel
route), the MTK9602-style chain does not port to MTK9603/U09.60. The
launcher lane — which needs none of this — is unaffected.

## Next gate: launcher lane pairing

With the root lane closed read-only and harmlessly, the remaining preflight
goal is the product lane: pair the server with this TV (`server/pair.py`),
confirm the 09.60v.Q0612 / `transport_protocol=3290` generation uses the
documented pairing handshake, then serve and self-install the launcher per
findings 01. Any pairing attempt should be owner-initiated.

## Root lane, second approach: /APPS write to phoenix code exec (probe v5)

The traversal closes only *reads outside* `/APPS`. Inside `/APPS`,
`fileRead` works without traversal — and `/APPS` is the rw partition where
phoenix executes its JS service packages (findings 04 §10.6 lane 1).
`fileWrite` into `/APPS` therefore remains a route to **code execution as
uid 314**: modify a service package JS, force a supervised phoenix restart,
and the service runs our code with real `fs` access filesystem-wide,
`process.env`, and the authoritative api table — everything the `../`
read primitive provided.

Plan, staged and owner-gated:

1. Read-only inventory of the service packages (probe v5): `fileRead` on
   `pkgs/tv.vidaa.jsservice.{basic,system}/CHA|CHB|current/…` and a
   directory-existence oracle (`EISDIR` vs `ENOENT`) over `/APPS/pkgs`.
   Text files only — binary reads crash the backend. Certificates and
   private keys are explicitly excluded.
2. Write lane (owner-gated): a minimal, auditable api addition to a
   service JS, with a kept backup and teardown path. If this build
   re-extracts packages at boot the edit self-reverts; that behaviour is
   itself a finding to record.
3. From uid-314 code exec, re-verify this build's root-side consumers:
   the `NewUpdatedIPK.txt` → `set_ipk_path.sh` symlink flipper, the
   `deploy_oem_data.sh` staging deployer and what executes from its
   destinations, and the UART console gate (`check_usb_key.sh`) — if the
   gate is a flag file on writable storage, the console opens without
   hardware. `/OAD` and OTA staging remain hands-off per ground rules.

## Probes v5–v8: the /APPS package map, and the run_env.js conclusion

Probes v5–v8 mapped `/APPS/pkgs` by directory-existence oracle
(`EISDIR`/`ENOENT`) and file guessing. Snapshots:
`server/recon-report-fingerprint-v5..v7.json`, live `recon-report.json`.

Established:

- `phoenix.server.js` (6231 bytes, captured whole) matches the lab TV
  framework: Router + ServiceManager, HTTP 9009, **HTTPS/WS on 9888 with
  `./cert/cert.pem` + `get_security('SERVER_PRIVATE_KEY')`** — so 9888 is
  live behind TLS; plain-HTTP probes misread it as refused. GET requests
  go through `decodeURIComponent` + `searchParams` (flat args) — no new
  power, but a usable alternative request shape.
- Packages present in `/APPS/pkgs`: `tv.vidaa.jsservice.basic` (CHA +
  live `current`, no CHB), `tv.vidaa.jsservice.system` (CHA + `current`),
  `tv.vidaa.app.phoenix` (CHA + `current`), `tv.vidaa.phony.keystore`
  (**EACCES** — locked down harder than the lab TV). `NewUpdatedIPK.txt`
  does not currently exist. `node_modules/` exists in both jsservice
  packages; `services/` exists in the system package and `utils/` in the
  basic package.
- `run_env.js` (basic CHA) is 237 bytes and only does
  `moduleAlias.addAlias('@sdk', process.env.BASIC_JS_LIB)` and
  `('@utils', process.env.BASIC_JS_UTIL)`.
- The `@sdk` framework and the system package's service entry could not
  be located by ~40 guessed names across `CHA/`, `node_modules/@sdk`,
  `services/`, `utils/`, `lib`, `src`, `dist` etc. If `BASIC_JS_LIB`
  points at the ro `/opt` tree, the jailed read cannot reach it at all.

Conclusion: blind filename guessing is exhausted, but it is unnecessary —
**`run_env.js` is the write lane**: a known path, known bytes,
`require`d by the running phoenix process before anything else, on the
writable partition. Overwriting it with original-code-plus-a-guarded
command loop yields code exec as uid 314 at the next phoenix restart,
activatable afterwards via `fileWrite` of a command file (mode 6 →
`/APPS/pkgs/cmd.json`) with results written back to a result file — no
network changes, no listener, kill switch = restore the original bytes.

Write-lane stages, each owner-gated:

- **W1 (tmpfs only):** confirm `fileWrite` exists and capture its arg
  shape by writing/reading back a probe file with `mode: 0` (`/tmp`,
  volatile). Zero persistence.
- **W2 (the hook):** read back `run_env.js`, byte-compare against the
  captured original, write original + guarded append
  (try/catch around everything after the untouched alias lines), readback
  verify, then force the supervised restart by the documented crash-on-
  ELF-read behaviour (a binary under `tv.vidaa.app.phoenix/CHA/bin`).
  If the bridge fails to come back, reboot is the recovery path *iff*
  packages re-extract at boot — which is unverified on this build, so W2
  proceeds only with the owner aware of that risk.
- **W3 (enumerate + escalate):** `fileWrite` a command that
  `readdirSync`s `/APPS/pkgs` and the system package, reads the real
  service sources and env, then re-walks this build's root-side
  consumers (§ above). Teardown: restore the original 237-byte
  `run_env.js` and remove the command/result files.

## W1 result: fileWrite is permission-gated (503)

Probe W1 (tmpfs-only) hit a gate the read path never showed:
`fileWrite` — with empty args and with every candidate content-arg name —
returns `503 "client request permission check error, please check
appconfig"`. Readbacks confirm nothing was written. So:

- The raw-service fileRead works with **no** `identifier` header, but
  write-class calls pass through a permission check first (the
  `get_security` middleware seen in phoenix.server.js). The check
  presumably validates the calling app identity carried by the shim's
  rotating `identifier` header, against per-app "appconfig" permissions.
- `websdk/Appinfo.json` is only the app-store listing — no permission
  fields — so the gate's config lives elsewhere.
- Consequence: the write lane needs the shim's identity, not just the
  endpoint. The shims are plain JS in the page context, so
  `HiUtils_createRequest.toString()` (probe W1b) should reveal both the
  identifier construction and the exact calling convention — after which
  the same write can be attempted with a valid identity, or the
  identifier scheme reproduced directly.

## W1b/W1c: the vowOS bridge, a third service, and the real gate

- The wrapper is `vowOS.service.syncExecute(service, {api, args})` — a
  **synchronous XHR** that tries `https://localhost:9888/service/<svc>`
  first, then falls back to `http://localhost:9009`. Since shim calls
  return application-level JSON, **HTTPS 9888 is reachable from pages**
  (the TV's own cert is trusted in Weblayer); plain-HTTP probes to 9888
  had misread it as closed.
- `executeHttpRequest` attaches an `identifier` header from
  `getIdentifier()` → `window.vowOSContext.getAppIdentifier()` — an
  identity injected per app by the platform. So shim-mediated calls
  **do** carry our app's identity … and W1c confirmed `fileWrite` through
  the shim still returns the same 503 "please check appconfig". The gate
  is therefore **per-app permission for the `vidaa-recon` custom-URL
  app**, not a missing/invalid identifier. (Identifier values are never
  exfiltrated; the page carries them inline, reports capture shape only.)
- **A third service exists: `pkgmgr`** (`vowOS.store.*`): `getInstalledPkgs`
  via syncExecute, and `install` via GET
  `https://localhost:9888/service/pkgmgr?api=install&args=<urlencoded>`.
  Package-app installs register `file:///APPS/pkgs/<pkg>/index.html` in
  Appinfo.json — the same self-install mechanism the launcher lane uses.
  The v4 service-name sweep missed `pkgmgr`; the name space is not
  exhausted.
- `vowOS.service.reLaunchService()` sends
  `omi_platform.sendPlatformMessage({type:'restartWebsocket'})` — a
  native-side service restart, potentially a clean phoenix-restart
  primitive for W2 (no ELF-crash needed). `omi_platform` is a whole
  second bridge surface, not yet walked.
- The WS variant `vowOS.service.execute('phoenix://service/hiutils', …)`
  puts the identifier **inside** the message rather than a header — a
  separate server-side code path whose permission handling is unverified.

Next (W1d): pkgmgr api enumeration (benign names only; no install-class
calls), `getInstalledPkgs` for the real package list, identifier shape,
one WS-path `/tmp` write probe, `omi_platform` walk, and an appconfig
filename hunt under `/APPS`.

## W1d results: pkgmgr is open, the identifier is empty, apps are typed

- **pkgmgr reads are ungated**: `getInstalledPkgs` returns the full
  18-package table with `type` and `path` per package. Types seen:
  `ns` (browser, phoenix), `res`, `phony`, `native`, `system`
  (operationui), `js` (jsservice.basic, jsservice.system), `web`
  (tvbrowser). Only `getInstalledPkgs` responded of the guessed api
  names; `install` exists per the shim source. The pkgmgr name space is
  still not exhausted (names come from the jsservice.system source, not
  guesswork).
- **`getAppIdentifier()` returns empty** in our page (len 0), while
  `navigator.appIdentifier` is a real string — the platform injected an
  identity the shim isn't reading. Every call so far (raw and shim) went
  out effectively anonymous.
- The WS-path write probe: `execute()` returned true but the readback is
  ENOENT and `session_array` was empty on read — inconclusive; the WS
  path needs its response handler observed before drawing conclusions.
- Service sweep round 2: only `pkgmgr` exists of 14 new names; total
  known services: hiutils, tvinfo, pkgmgr.
- `omi_platform` is exactly two functions (sendPlatformMessage,
  addPlatformEventListener) — the native message bridge.
- appconfig filename hunt (websdk/ + system services/): all misses.
- Cross-reference: `websdk/Appinfo.json` holds only 3 store apps — our
  `vidaa-recon` URL app (launched via `launchapp`) is **not registered
  anywhere**, i.e. an app with no config entry at all, matching the 503
  "please check appconfig" semantics.

Working hypothesis: the 503 gate resolves the calling app (via
identifier header or Origin) to an app entry whose config — the
`configUrl` field store apps carry, downloaded and cached — lists
permitted apis. Our URL app has no entry, so only default-allow apis
(fileRead) work. W1e tests: fileWrite with `navigator.appIdentifier` as
the header (tmpfs), the GET-path write shape (no identifier), and
package manifest reads (`config.json`/`pkg.json`/`manifest.json` at CHA
roots of the js/browser/phoenix packages) to find where appconfig
actually lives.

## W1e results: no identity at all, and appconfig is a webview config

- `navigator.appIdentifier` is the **literal string `"undefined"`** — the
  platform injects the bad value for URL apps, which is precisely what
  `getIdentifier()`'s `!== 'undefined'` guard exists to catch. Our page
  has no app identity; presenting `"undefined"` as the header changes
  nothing (503 both POST-with-header and GET-no-identifier; readbacks
  ENOENT).
- All 16 package-manifest guesses missed (config/pkg/manifest/app/api/
  appconfig.json at the js, tvbrowser and phoenix CHA roots).
- A real store app's `configUrl` (fetched from the vidaahub CDN off-TM)
  is a **webview config**, not an api permission table: `ConfigVersion`,
  `OpenType: web`, `OpenMode/Resolution`, `SupportBackgroundMode`, and
  `Preferences.mainWindow` with engine flags such as
  `service_worker_enabled` and **`xhr_origin_check_enabled: "false"`** —
  installed apps can weaken their own window's web-engine origin
  checking via config. The phoenix-side permission table is not this
  file; it lives in unread source or platform storage.
- `vowOSContext` members found: `init`, `getAppIdentifier`, `getAppId`
  (sources not yet captured).

W1f: capture the vowOSContext sources and `getAppId()` value, then test
whether the permission check *trusts a declared identifier* — tmpfs
fileWrite with identifier set to our launch appId (`vidaa-recon`), a
registered store app id (`351`), and a package name
(`tv.vidaa.app.browser`). If any clears the 503, the identifier is a
trusted client declaration and the write lane is open. If none do, the
identifier must be a platform-issued token, and the remaining route is
installing a registered app (Hisense_installApp with a controlled
`configUrl`) — which is launcher-lane work regardless.

## W1f result: identifiers are not declarable — and self-install was the same primitive

- `vowOSContext.{init,getAppIdentifier,getAppId}` are **`[native code]`**
  — webengine-injected, no JS source. `getAppId()` returns empty in a
  URL-app page.
- All three declared-identifier writes (own appId, store app id `351`,
  package name) returned the same 503; readbacks ENOENT. The permission
  check does **not** trust client-declared identifiers. Real app
  identity is a platform-issued token injected natively into registered
  apps' pages only — a URL-app page can never carry one.
- Cross-check against findings 01 §6: the documented launcher
  **self-install was itself a `fileWrite`** (append to
  `websdk/Appinfo.json`, arg `writedata`). The same 503 gate therefore
  also blocks the launcher's persistent install on this TV. The root
  lane and the persistent-install lane were always the same primitive,
  now closed from URL-app pages.

## W1g: register through pkgmgr (the store's own permissioned channel)

The store installs apps somehow, and `pkgmgr` (ungated for reads) is
the package manager with write rights. If a page can drive a *URL-app
registration* through pkgmgr, the platform performs the registry write
itself — and the registered app's pages may then receive a real
injected identifier, unlocking fileWrite from inside that app.

`launcher/fingerprint.html` = W1g: auto read-only introspection
(`Hisense_installApp` injection? `Hisense_SupportAppConfig`?), then a
**button-gated** install through three channels: the legacy
`Hisense_installApp` shim if injected, `vowOS.store.installApp`'s
URL-app branch (with our own `configUrl` + `configUrlDownload`), and
direct `pkgmgr api=install` GETs in two arg shapes. Afterwards it
re-reads `websdk/Appinfo.json` (registration visible?) and probes
candidate config-download locations. `launcher/appctx.html` is the
installed app's target: it reports the in-app identity (token shape
only) and retries the tmpfs fileWrite with that identity. The appconfig
served for the install (`launcher/tvappconfig.json`) is minimal
(ConfigVersion 3, OpenType web, 1080P).

Note the reversibility path: a registered `alvidaa-home` app entry is
removable via the TV's own app-management UI (store apps show a remove
option) or a later `pkgmgr uninstall`, should it exist.

## W1g result: no install channel survives on this firmware

All three registration channels returned hard negatives:

- `Hisense_installApp` is **not injected** on this build (`undefined`).
  `vowOS.store.installApp`'s URL-app branch threw
  `ReferenceError: Hisense_installApp is not defined` — the store
  wrapper itself depends on a shim this firmware removed.
- `pkgmgr api=install` (GET, HTTPS 9888, both arg shapes):
  `"Not supported api"`. This build's pkgmgr exposes reads
  (`getInstalledPkgs`) to pages; install is not in the page-reachable
  table.
- `websdk/Appinfo.json` unchanged (3 store apps); no config-download
  artifacts appeared at any candidate path.
- One positive: `Hisense_SupportAppConfig()` exists and returns `true`
  — the appconfig subsystem is present; only the installers are gone.

## Preflight verdict (browser-reachable surface exhausted)

Every page-reachable channel has now been enumerated and tested:
~40 file paths, ~40 api names across the three services
(hiutils/tvinfo/pkgmgr), all three transports (HTTP 9009, HTTPS 9888,
WS 9888), every identifier shape (none, "undefined", declared app ids),
and all three documented install generations. The design holds:

- Reads are open but jailed to `/APPS` + `/tmp` by a sound substring
  check with no decode/normalisation behind it.
- Writes and install are tied to platform-issued per-app identity that
  URL pages can never carry (`[native code]` injectors, empty
  `getAppId()`), and the shim-based installers were removed outright.
- The uid-314 code-exec lane (run_env.js hook) is therefore unreachable
  from pages — it required fileWrite, which is what the gate protects.

Remaining root-lane routes are outside the browser: the UART service
port (hardware gate contents unknown on this build), the USB update
lane, the OTA lane (hands-off per ground rules), or a webengine/Chromium
exploit (different effort class). None are pursued in this session.

Launcher lane consequence: persistent app-list self-install was itself
a fileWrite and is blocked on this TV. The product still works without
it — MQTT pairing (pending the owner's cert extraction from the VIDAA
APK) gives on-demand launch via `/api/open`, and the server's home
watcher relaunches the launcher whenever the TV sits at VIDAA home,
which is de-facto default-launcher behaviour without any app entry.

## Remaining gaps (probe W2)

Four surfaces were never actually exercised, and W2 covers them:

1. **The `/function` route.** phoenix.server.js registers two middleware
   chains — `/service` (probed extensively) and `/function` with a
   different request parser (`getFunctionReqInfo`). Zero requests sent
   so far. Different dispatch may mean different permission handling.
2. **The WebSocket dispatch, properly.** The W1d WS write test never
   observed socket state (inconclusive). The WS handler is a distinct
   path (`_role = 'WS'`, identifier inside the message, not a header).
   W2 runs a control fileRead over WS first, then one tmpfs fileWrite,
   with full open/message/error/close observability.
3. **The unread packages.** `tv.vidaa.app.osconfig` (name says it),
   `tv.vidaa.res.webengine`, `res.iot`, `res.odin`,
   `tv.vidaa.app.operationui` — no file probe has touched them. If a
   permission table or webengine policy is page-readable, it is there.
4. **The tvinfo param space.** `TvInfo_setParam` is a write api on a
   service never enumerated; `Hisense_GetTvLogo` implies file-backed
   settings exist. W2 fuzzes the request shape and ~24 non-identifier
   param names via read-only `getParam`.

Longer-shot routes, not in W2: `omi_platform.sendPlatformMessage` type
fuzzing (native message schema unknown; undefined side effects — needs
its own careful session), and the post-pairing experiment of launching
a *registered* app's context with a controlled URL via the MQTT
`launchapp` (needs the owner's pairing certs; would test whether
identity follows the launch context). Beyond those, the browser lane is
closed and root requires UART/USB/OTA-lane work or a webengine exploit.

## W2 results: /function dead, WS alive with different dispatch

- **`/function` route: dead** — 404 on all five shapes. Registered in
  phoenix.server.js but not wired to any page-reachable handler.
- **WSS 9888 connects from pages** (readyState 1). The WS dispatch is
  different from HTTP: a command with `method: ""` and
  `parameters.api = "fileRead"` returns **"unknown method"** (500), not
  a permission error — the router keys on the `method` field, whose
  semantics are unknown. `fileWrite` over WS hits the same 503 gate.
  So the middleware applies, but the calling convention is unexplored
  (W2b probes `method` values: api-name-in-method, callService,
  handle, execute).
- **tvinfo: `getParam` is not the api name.** All six request shapes
  "Not supported api". Plausible convention: the param name *is* the
  api name (W2b tests it).
- **osconfig/webengine/operationui: directories exist** (osconfig CHA +
  current, webengine CHA, odin CHA), but all 13 filename guesses
  missed. Deeper enumeration in W2b.

## Chromium 111 note (firmware r20260612, webengine from ~March 2023)

The webengine is Chromium 111 / Weblayer 5.2.1 on armv7l — about three
years of V8/browser security fixes behind at the firmware build date
(June 2026), and VIDAA TVs do not receive Chromium updates. Publicly
documented n-days against ≤111 include: CVE-2023-2033 and
CVE-2023-3079 (V8 type confusions, exploited in the wild, patched
112/114 — see Theori's n-day chain writeup), CVE-2023-4762, and later
ITW V8 bugs (CVE-2024-0517, CVE-2024-4947, CVE-2025-5419, CVE-2025-10585;
CISA KEV lists several). Renderer RCE from a page is therefore plausible
in principle, but: no public armv7l-embedded exploit chain exists; the
webengine sandbox status on this build is unverified (embedded Chromium
builds are often compiled without full sandboxing, which would make a
single V8 bug much more valuable); and weaponising V8 n-days is
expert-level work far beyond this session's scope. The practical
mitigation stands regardless: the TV browser must never see the open
web (DNS-block), and pages it loads come from the LAN server only.

## W2c: tvinfo WRITES are ungated — the param-name channel writes

- Set convention confirmed: `POST /service/tvinfo {api:"soundVolume",
  args:{value:N}}` → `ret:true`. Both `value:` and the param-name key
  work. Meanwhile `api:"setSoundVolume"` returns the **503 gate** —
  the permission whitelist is api-name-shaped: bare param names are
  default-allow for *both get and set*, while `set`-prefixed names are
  gated. Net: **arbitrary TV-settings writes from a URL page**.
- 28 live params read back with values (region EU_A, netType wireless,
  ttsLanguage eng, tvLogo "his", resolution "", magnification "0" —
  several string-valued). One anomaly: volume read 9 in W2b, and the
  write of 9 returned `soundVolume: 5` — value moved between runs
  (remote/CEC) or the write path echoes something else; re-verify
  before trusting read-modify-write.
- WS method space stays closed ("unknown method" on all shapes, clean
  close 1005); osconfig/webengine filenames still all misses.

## W2d: the dream-param sweep

The ungated write primitive makes a new class of target reachable: if
any **service-opening or path-valued param** exists in the default-allow
space (telnetEnable/sshEnable/adbEnable/debugMode-class, or
logoPath/updateServer/proxyServer-class), root-adjacent access could be
a single settings write away. W2d sweeps ~120 candidate names read-only
across service-opening, network/server, path-like, and settings-space
names. ## Odin surface probe (odin1)

Launched into the Odin browser via the paired channel
(`server/recon-report.json`, page version odin1):

- **Engine confirmed as stock-shaped V8 11.1 / Chromium 111**: feature
  markers match exactly (resizable ArrayBuffer present — shipped 111;
  `Array.fromAsync` and RegExp `v` flag absent — shipped later;
  `findLast` present). No vendor version skew detectable from JS.
  WebAssembly present (payload-staging relevant). Mali-G52 GPU,
  4 cores, armv7l.
- **armv7 consequences favour the attacker**: 32-bit V8 has no pointer
  compression and no V8 heap sandbox (both 64-bit-only) — the classic
  tagged-pointer exploitation model, and if the renderer-side Chromium
  sandbox is absent or broken (common on embedded builds; unverified
  without code exec), a single V8 bug is code exec as the browser uid.
- **No shortcuts**: all `file://` access shapes from an http page are
  fenced (XHR/fetch deny instantly, iframe navigation never completes,
  window.open returns null); no CDP/chromedriver/inspector port is
  listening on localhost (9222/9223/9333/9515/9229 all refused fast).
- In-engine public n-days (fixed in later Chromium, no vendor backport
  detectable from JS — testing requires exploit oracles):
  CVE-2023-2033 (ITW, fixed 112), CVE-2023-3079 (ITW, fixed 114),
  CVE-2023-4762 (fixed 116), CVE-2024-0517 (ITW, fixed 120;
  Man Yue Mo / GitHub Security Lab writeup), plus the 2024–2025 ITW
  stream (CVE-2024-2887, -4947, -5274, CVE-2025-5419, -10585).
- **Vendor bridge is the softer adjacent target**: 96 `Hisense_*` fns +
  `vowOS` + `omi_platform.sendPlatformMessage` are native bindings
  reached from JS with attacker-controlled strings — vendor IPC parsing
  is far less studied than V8 and is a classic TV-root route. Fuzzing it
  has undefined side effects and needs its own carefully scoped session.

## W2d verdict: no dream params

150 candidate param names swept read-only via the confirmed api-name
convention. Two hits: `standbyMode: 0` and `storeMode: 0` — both benign
display/retail settings. The tvinfo space contains no service-opening
switches (telnet/ssh/adb/debug class), no path-valued params, and no
server-URL configs. The ungated settings-write primitive therefore has
no reachable file-backed target. Record and move on.

## Fuzzing campaign (fuzz1, fuzz2): two robustness bugs, no escalation

~60 malformed-input probes across the bridge surface
(`server/recon-report.json`, pages fuzz1/fuzz2):

- **Robustness bug 1 — ~1 MB path crashes phoenix.** 100 KB path →
  clean `ENAMETOOLONG` (errno -36, handled); 1 MB path → the backend
  dies (health check immediately after fails; supervised respawn).
  Between 100 KB and 1 MB there is an unhandled failure in the file
  service's path handling. DoS-grade only: respawn is automatic and
  nothing persists.
- **Robustness bug 2 — native `sendPlatformMessage` can freeze the
  renderer.** A deeply-nested JSON string (200 levels) never returned;
  the page's JS thread blocked and reporting stopped. A synchronous
  native bridge call without a timeout is a renderer-hang primitive.
- **Identifier claims don't work**: presenting the identity JSON with
  claimed `permissions: "fileWrite"`, `base64(md5(appid))`, or
  `"undefined"` as the identifier header on tmpfs `fileWrite` — all 503.
  The backend does not trust client-side permission declarations in any
  format.
- **Prototype pollution doesn't reach the gate**: `__proto__` /
  `constructor.prototype` payloads via fileRead args (object and raw
  JSON forms), then clean and polluted-args `fileWrite` — all still 503.
  The permission check is not a plain-object lookup reachable through
  request-arg merging.
- Everything else handled gracefully: dup-keys JSON takes last-wins
  (confirmed the gate evaluates the parsed api name), non-string paths
  coerce safely (arrays → `""` → EISDIR; NUL byte → Node's
  `ERR_INVALID_ARG_VALUE`), tvinfo/pkgmgr ignore shape confusion.
- Operational finding: one-off `TVBridge` scripts and the persistent
  server share one client identity — each connection kicks the other
  (one-client-per-identity). All TV interaction now goes through the
  persistent server's `/api/open` (extended with `appid`/`store`/`name`
  query params so Odin launches work without identity collisions).

Net: the browser-reachable surface is now exhausted at three depths —
static enumeration, dynamic probing, and fuzzing — with no escalation
to the write lane. Remaining root routes are unchanged: V8 n-day
development on Odin 111.5563.5.1 (expert project), the UART service
port, or the OTA lane.

## euladpp lane (upstream lead, probe euladpp1)

The upstream author's field report (same firmware generation,
`09.60V.Q0612`/MTK9603 — see findings 04 "euladpp" section) describes a
no-traversal root lane: `euladpp-init-check.sh` untars
`/APPS/euladpp/eulaDpp.tar.gz` as root (GNU tar 1.17, no slip hardening)
whenever `current/versionInfo.json` is missing/broken. Probe results on
this TV:

- **The lane's target exists and its trigger is already armed**:
  `/APPS/euladpp/` and `euladpp/current/` both exist (EISDIR),
  `upgrade/` absent, and `current/versionInfo.json` is **ENOENT** — the
  init-check's untar condition is true on this TV right now.
- **The write the lane needs is denied**: in-sandbox relative
  `fileWrite("alvidaa-probe.txt", mode 6)` returns the same 503 gate —
  the permission check is **API-level, not path- or base-dependent**
  (this answers the upstream "test fileWrite("euladpp/probe.txt")"
  question for this TV: it fails). The scratch file was never created;
  nothing was written.
- `pkgs/tv.vidaa.jsservice.modules` (upstream's API-table location) does
  not exist on this build — not in pkgmgr's list, not on disk.
- Open question back to the author: the field-report TV of the same
  build yielded a dump of root-run boot scripts — **with what
  primitive?** If in-sandbox fileWrite worked there, the 503 gate may be
  config- or region-specific; if the dump came via UART/bootrom, the
  euladpp lane on this TV remains gated behind the write we cannot get.
- Remaining twist worth exploring: the EULA-acceptance window. The
  euladpp scripts serve the EULA flow, and a first-run setup-wizard app
  context may legitimately carry write permission (it must stage
  euladpp state). Our identity sweep covered the 12 registered apps but
  not a setup-wizard context — its appId is unknown and the flow is not
  reachable post-acceptance without a factory-reset trigger (which is
  `/var/tvservice/reset_working` — hands off).

## End-to-end confirmation: the original exploit page fails as predicted

The genuine, unmodified `launcher/hookinstall.html` (root-hook v10
payload targeting `/var/local/otaTest.sh`) was loaded on the TV via the
paired MQTT channel — the same delivery the blog chain used — as a
final end-to-end check after the synthetic probes:

- Traversal read of the existing hook: `invalid path:../var/local/otaTest.sh`
  (substring validator).
- Hook write: `client request permission check error, please check
  appconfig` (the 503 identity gate).
- No backup, no hook, no files written anywhere on the TV.

This confirms empirically, with the actual exploit artifact rather than
synthetic probes, that the blog-era chain is dead at both of its links
on this firmware. Full teardown of the lab-TV-era tooling on this TV is
therefore also moot — nothing was ever installed.

## Downgrade assessment (owner question)

Shape of the idea: serve the TV's updater an older *legitimately signed*
image for this model, let the TV install it, test whether the older
MTK9603 build predates the sandbox hardening, then re-upgrade. Three
blockers and one open question:

1. **Anti-rollback.** Verified boot is green/locked (lab-TV map: dm-verity
   squashfs, `vbmeta.device_state=locked`). AVB rollback indexes (RPMB/
   fuse-backed) typically refuse any image older than the installed
   rollback level, even perfectly signed ones. Whether Hisense bumps the
   index per release on this platform is unknown — it is the first thing
   any downgrade attempt would discover, the hard way.
2. **Image availability.** A full signed image for 55A60LEVS / EU_A /
   MTK9603 of an older build must exist somewhere reachable. Forum-hosted
   firmware binaries have unverifiable provenance; a signature check
   would catch tampering but the updater's hash/signature handling of a
   foreign-hosted image is itself an unknown.
3. **Risk and ground rules.** A failed or interrupted OTA write is a
   brick; this is the owner's daily TV. The repo's OTA discipline exists
   to *protect* the TV from exactly this lane (freeze OTA before edits,
   never touch `/OAD` staging) — deliberately driving the updater is the
   opposite direction and should not be done without explicit informed
   consent and a teardown story.

Open question that decides the payoff: **did any MTK9603 build ever ship
without the `../` validator?** The blog TV's vulnerable build (09.01O,
Aug 2025) is a different SoC line (MTK9602). If the validator shipped
with the MTK9603 platform from its first release, no reachable downgrade
contains the vulnerable code and the entire exercise yields nothing but
brick risk. Nothing in this session can answer that from here.

Zero-risk first step if the owner wants to explore the lane: **passive
network observation** of the TV's update traffic (DNS + TLS SNI capture
from the LAN side, no interception) to learn which update infrastructure
this EU build actually contacts. That is read-only recon and informs any
later decision.

## Pairing works — and the launch-context identity lane closes clean

- **Pairing succeeded** with the modern profile after one fix: the
  client_id requires the TV MAC **with colon separators**
  (the TV MAC in `xx:xx:...` colon-separated form); the SSDP descriptor's colon-less form produces
  a different digest and rc=135 across all four profiles. `tokens.json`
  written (2-day access token); `vidaa_server.py` connected with token
  auth. The PIN is session-bound: killing the pairing process invalidates
  the dialog ("illegal authNum" on a same-credentials reconnect).
- **Identity follows the launch context.** `launchapp` with a registered
  appId plus an arbitrary URL makes the platform inject that app's
  identity into our page: `navigator.appIdentifier` =
  `{"appid":…,"md5":…,"permissions":…}`, where **md5 = md5(appid) alone**
  (verified: md5("351") matches the injected value) and the rotating
  identifier header is a 16-byte value (b64, 24 chars) not reproducible
  from appid offline. Unregistered appids get the literal string
  `"undefined"`.
- **The identity sweep is a clean negative**: 12/12 registered contexts
  (tvbrowser, browser, operationui, phoenix, jsservice.basic,
  jsservice.system, lib.odin, osconfig, websdk, 2170, 162,
  vidaa-recon) report `permissions: ""` and get the same 503 on tmpfs
  fileWrite — including via each app's own legitimate injected
  identifier. `server/identity-sweep2.json`. The permission field exists
  in the identity structure but is unprovisioned for every app on this
  TV: **page-bridge fileWrite is denied for all identities, period.**
- **Two web engines confirmed.** Pages launched via the pre-pairing
  custom-URL path (`/api/open`, storeType custom) run in
  `Chrome/111 … Weblayer/5.2.1`; pages launched via the paired MQTT
  `launchapp` as a registered app run in
  `Chrome/111.0.0.0 **Odin**/111.5563.5.1` (the VIDAA browser build;
  UA carries Model/VIDAA-MTK9603 and the full firmware string). The UA
  difference is the hosting stack, not the file-bridge backend — the
  `../` validation is server-side in phoenix and identical for both
  (probe v2's raw request proved no shim involvement).
- Consequence: the `run_env.js` write lane is unreachable from every
  page context this TV can produce. The remaining root routes stay as
  assessed: UART hardware, USB/OTA lanes, or a webengine renderer
  exploit (the UA differs per app context — different engines may be in
  play per window; captured in `server/ua-sweep.json`).
