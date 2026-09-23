# Native macOS app: spawn `scanimage` or link `libsane`?

Research ticket for issue #4 (map: issue #2). Researched on 2026-09-23 from a Linux
container. **Nothing here was run on macOS or against the SCX-4623FW.** Claims marked
*(untested)* are inferences from source code and Apple documentation; they need
confirming on a Mac with the printer on the LAN.

## Recommendation

For v1, **spawn `scanimage` as a child process** from a Developer ID–signed app with the
Hardened Runtime and **without** App Sandbox. Keep the SANE access behind one small Swift
protocol (a "scan engine") so a later version can switch to an in-process or XPC
`libsane` engine without touching the UI.

Why:

- The App Sandbox rules out **both** approaches while SANE comes from Homebrew (see
  [App Sandbox](#app-sandbox)). Sandboxing is not a reason to prefer either one.
- Spawning has no code-signing cost. Linking Homebrew's `libsane` requires
  `com.apple.security.cs.disable-library-validation`, which also triggers extra
  Gatekeeper checks.
- `scanimage` already exposes what v1 needs: percentage progress, SIGINT that becomes
  `sane_cancel()`, an exit code equal to the `SANE_Status`, ADF batch mode, and output
  in PNM, TIFF, PNG, JPEG or PDF.
- A backend crash or hang takes down the child, not the app.

The main cost: the app reads **human-oriented stderr text** (`Progress: 42.0%`,
`Scanning page 3`, `Batch terminated, …`). Those strings are not a stable API. Pin the
version they are tested against and fail safe when a line doesn't parse.

Move to `libsane` only if v2 bundles SANE inside the app (that is out of scope for v1,
see issue #2). At that point the model to copy is NAPS2's: SANE is bundled and signed
with the app's own Team ID, and runs in a separate worker process.

## Facts that frame the decision

### The backend: `xerox_mfp` over TCP

Source is sane-backends at commit
[`cadda80b`](https://gitlab.com/sane-project/backends/-/tree/cadda80b9fec0d69728561aeae57eb8278f99c75).
Homebrew ships sane-backends **1.4.0** under GPL-2.0-or-later
([formula API](https://formulae.brew.sh/api/formula/sane-backends.json)).

- **No preview option.** The backend's options are resolution, mode, threshold, source,
  `jpeg` (advanced) and the four geometry values
  ([`xerox_mfp.h` L35–46](https://gitlab.com/sane-project/backends/-/blob/cadda80b9fec0d69728561aeae57eb8278f99c75/backend/xerox_mfp.h#L35-46)).
  A "preview" is therefore just a normal low-resolution scan (e.g. 75 dpi, full area),
  under either approach.
- **Blocking I/O only.** `sane_set_io_mode(h, SANE_TRUE)` returns `UNSUPPORTED` and
  `sane_get_select_fd` returns `UNSUPPORTED` ("supporting of this will require thread
  creation")
  ([`xerox_mfp.c` L1583–1601](https://gitlab.com/sane-project/backends/-/blob/cadda80b9fec0d69728561aeae57eb8278f99c75/backend/xerox_mfp.c#L1583-1601)).
  An in-process frontend must call `sane_read` on its own thread.
- **Cancel is a flag.** `sane_cancel` only sets `dev->cancel = 1`
  ([L1604–1610](https://gitlab.com/sane-project/backends/-/blob/cadda80b9fec0d69728561aeae57eb8278f99c75/backend/xerox_mfp.c#L1604-1610)).
  The flag is checked before each image block is requested (`cancelled(dev)`, L387–401,
  L1406). Cancellation latency is at most one block plus the network round trip
  *(untested: block size and real latency on the SCX-4623FW are unknown)*.
- **Device states map onto SANE status codes:** `JAMMED`, `NO_DOCS`, `COVER_OPEN`,
  `DEVICE_BUSY` ([L56–76](https://gitlab.com/sane-project/backends/-/blob/cadda80b9fec0d69728561aeae57eb8278f99c75/backend/xerox_mfp.c#L56-76)).
  Both approaches can show these as specific messages (see [Error handling](#error-handling)).
- **TCP transport:** default port 9400 and a 1-second `SO_RCVTIMEO`
  ([`xerox_mfp-tcp.c` L47, L121, L137](https://gitlab.com/sane-project/backends/-/blob/cadda80b9fec0d69728561aeae57eb8278f99c75/backend/xerox_mfp-tcp.c#L47)).
- **A hard-coded `/tmp` file in colour mode.** When JPEG compression is active and the
  mode is RGB, `sane_start` creates `/tmp/stmp_enc.tmp` with `O_CREAT|O_EXCL`. If that
  fails it returns `SANE_STATUS_ACCESS_DENIED`
  ([L96](https://gitlab.com/sane-project/backends/-/blob/cadda80b9fec0d69728561aeae57eb8278f99c75/backend/xerox_mfp.c#L96),
  [L1565–1576](https://gitlab.com/sane-project/backends/-/blob/cadda80b9fec0d69728561aeae57eb8278f99c75/backend/xerox_mfp.c#L1565-1576)).
  JPEG is on by default (`compressionEnabled = SANE_TRUE`, L615). It applies only when
  the device advertises JPEG compression and its model is not blacklisted (L207–224).
  The SCX-4623 is not on the blacklist. Whether it advertises JPEG is *untested*.
  In JPEG mode the whole compressed block is buffered to that file and then
  decompressed (L1447–1452), so colour progress may arrive in bursts *(untested)*.
  Further consequences:
  - The file name is fixed, so **two concurrent colour scans on one Mac collide**.
  - It is a sandbox blocker (see [App Sandbox](#app-sandbox)).
  - Workaround for either approach: set the backend option `jpeg` to false
    (`--jpeg=no` in `scanimage`). This is *untested* on this model.
- The device can be opened by name, as `xerox_mfp:tcp <printer-ip>`, which is what
  `bin/scx-scan` does. The config file `xerox_mfp.conf` still matters for discovery
  through `sane_get_devices` / `scanimage -L`
  ([sane-xerox_mfp(5)](https://gitlab.com/sane-project/backends/-/blob/cadda80b9fec0d69728561aeae57eb8278f99c75/doc/sane-xerox_mfp.man)).

### The SANE API contract

From the [SANE Standard, chapter 4](https://sane-project.gitlab.io/standard/api.html):

- `sane_cancel` "can be called at any time … It is safe to call this function
  asynchronously (e.g., from within a signal handler)". The frontend "must *not* call
  any other operation until the cancelled operation has returned"
  ([§4.3.11](https://sane-project.gitlab.io/standard/api.html#sane-cancel)).
- No other operation is required to be re-entrant, and the standard gives no
  thread-safety guarantees. In practice every call has to go through one serial
  thread or queue.
- `sane_get_parameters` is exact only between `sane_start` and the end of the frame.
  `lines == -1` means the height is unknown in advance
  ([§4.3.8](https://sane-project.gitlab.io/standard/api.html#sane-get-parameters)).
- Status codes include `CANCELLED` (2), `DEVICE_BUSY` (3), `JAMMED` (6), `NO_DOCS` (7),
  `COVER_OPEN` (8), `IO_ERROR` (9) and `ACCESS_DENIED` (11)
  ([§4.2.7](https://sane-project.gitlab.io/standard/api.html#status-type)).
  `sane_strstatus` returns a single-line sentence.

### `scanimage` behaviour (from its source)

Source: [`frontend/scanimage.c` @ `cadda80b`](https://gitlab.com/sane-project/backends/-/blob/cadda80b9fec0d69728561aeae57eb8278f99c75/frontend/scanimage.c).

- **Progress.** `-p` writes `Progress: %3.1f%%\r` (or `Progress: (unknown)\r`) to
  **stderr** after each `sane_read` (L1567–1573). With `-p` the progress arrives in
  bursts, one update per `sane_read` buffer (1 MB by default, adjustable with `-B`).
- **Batch messages.** Batch mode prints `Scanning page N` (L2824) and
  `Scanned page N.` (L2897) to stderr. It ends with `Batch terminated, N pages scanned`
  (L3011). A final `NO_DOCS` after at least one page counts as success (L3015–3018).
- **Signals.** Handlers are installed for SIGHUP, SIGPIPE, SIGINT and SIGTERM
  (L2705–2711). The **first** signal calls `sane_cancel(device)`. The **second** calls
  `_exit(SANE_STATUS_CANCELLED)` (L320–338).
- **Exit status.** The process exits with the `SANE_Status` value itself
  (`scanimage_exit(status)`, L3025). For example 7 means `NO_DOCS` and 8 means
  `COVER_OPEN`.
- **Output formats.** `--format=pnm|tiff|png|jpeg|pdf|pdf-per-page` (L2402). When the
  output is PNM on stdout, the app can read the header and then rows as they arrive,
  so a progressive preview is possible without linking anything *(untested)*.
- **Option discovery.** `-A` / `--all-options` prints options as human-readable text
  ([scanimage(1)](https://man.archlinux.org/man/scanimage.1.en)). There is no
  machine-readable form, so an app that spawns `scanimage` hard-codes the few options
  it needs. `bin/scx-scan` already does this.

## Trade-offs by topic

| Topic | Spawn `scanimage` | Link `libsane` |
|---|---|---|
| **Progress** | Parse `Progress: x%` from stderr in bursts of up to 1 MB, or count bytes of PNM on stdout against the header size. Page events come from stderr text. | Exact: `sane_get_parameters` + bytes returned by each `sane_read`. Rows are available for live drawing as they arrive. |
| **Preview** | A separate low-dpi invocation. Each run reopens the TCP session. | A low-dpi scan on an open handle. Rows can be drawn as they arrive. |
| **Cancellation** | `Process.interrupt()` (SIGINT) → `sane_cancel`. A second signal hard-exits. Kill as a last resort. The device stays consistent because the backend runs its own cancel path. | Call `sane_cancel` (async-safe per the standard) while `sane_read` blocks on the SANE thread. Wait for that call to return `CANCELLED` before making any other call. |
| **Error handling** | Exit code = `SANE_Status`, which maps to a Swift enum. The detail text (`scanimage: sane_start: …`) is only on stderr. | Typed `SANE_Status` on every call, `sane_strstatus`, and the option descriptors with their constraints. |
| **Crash / hang isolation** | Built in: a backend crash or hang ends the child. | A backend fault kills the app. NAPS2 runs SANE in a worker process for this reason (see below). |
| **Threading** | None in the app beyond async pipe reads. | One dedicated serial thread owns every SANE call, as in simple-scan. |
| **Code signing / Hardened Runtime** | No effect: library validation applies only to code loaded into the app's process. `scanimage` runs under its own (ad-hoc) signature. | Needs `disable-library-validation`: Homebrew's `libsane.1.dylib` and the `dlopen`'d backends are not signed with the app's Team ID. |
| **App Sandbox** | Blocked (see below). | Blocked (see below). |
| **Homebrew coupling** | Needs the path to `scanimage` (`/opt/homebrew/bin`, or `/usr/local/bin` on Intel). | The install name `/opt/homebrew/lib/libsane.1.dylib` is baked in at link time, unless the app `dlopen`s at runtime with a path search. |
| **Licensing** | None: the app only runs a separate program. | Fine: the backend libraries carry the "SANE exception", which allows linking without affecting the app's licence ([LICENSE](https://gitlab.com/sane-project/backends/-/blob/cadda80b9fec0d69728561aeae57eb8278f99c75/LICENSE)). |

### Code signing and the Hardened Runtime

- The Hardened Runtime is required for notarization
  ([Apple: Hardened Runtime](https://developer.apple.com/documentation/security/hardened-runtime)).
- Library validation "prevents a program from loading frameworks, plug-ins, or libraries
  unless they're either signed by Apple or signed with the same Team ID as the main
  executable". Disabling it makes "Gatekeeper run extra security checks"
  ([Apple: disable-library-validation](https://developer.apple.com/documentation/bundleresources/entitlements/com.apple.security.cs.disable-library-validation)).
- Apple Silicon requires every executable to be signed, but "a simple ad-hoc signature
  is sufficient", and the linker applies one automatically. Such binaries "cannot pass
  through Gatekeeper"
  ([macOS 11 Universal Apps release notes](https://developer.apple.com/documentation/macos-release-notes/macos-big-sur-11_0_1-universal-apps-release-notes)).
  That is why Homebrew's `libsane` and backends carry no Team ID *(inference: the
  signatures of the installed files were not inspected)*.
- Entitlements apply only to the main executable, and libraries inherit them
  ([Hardened Runtime](https://developer.apple.com/documentation/security/hardened-runtime)).
  Linking therefore means the whole app carries `disable-library-validation`.
  Spawning needs no exception entitlement at all.
- SANE's `dll` meta-backend `dlopen`s each backend from `LIBDIR/sane` and honours
  `LD_LIBRARY_PATH` ([`dll.c` L463–537](https://gitlab.com/sane-project/backends/-/blob/cadda80b9fec0d69728561aeae57eb8278f99c75/backend/dll.c#L463-537)).
  Config files are read from `SANE_CONFIG_DIR` plus the compiled-in directory
  ([`sanei_config.c` L68–121](https://gitlab.com/sane-project/backends/-/blob/cadda80b9fec0d69728561aeae57eb8278f99c75/sanei/sanei_config.c#L68-121)).
  So even with library validation disabled, an in-process engine loads code from
  `/opt/homebrew/lib/sane/`.

### App Sandbox

- Child processes **always inherit the parent's sandbox**. This holds for `posix_spawn`,
  `NSTask` and `Process` alike, and "a process is not allowed to change its sandbox"
  (Quinn, Apple DTS, [forum thread 706390](https://developer.apple.com/forums/thread/706390)).
  Apple's embedding guide requires helper tools to carry only `app-sandbox` + `inherit`
  ([Embedding a command-line tool in a sandboxed app](https://developer.apple.com/documentation/xcode/embedding-a-helper-tool-in-a-sandboxed-app);
  [Enabling App Sandbox Inheritance](https://developer.apple.com/library/archive/documentation/Miscellaneous/Reference/EntitlementKeyReference/Chapters/EnablingAppSandbox.html)).
- For Homebrew tools specifically: "the user would have to place the tool in a directory
  that's within your app's static sandbox, and that's not the default for tools like
  Homebrew". Dynamic extensions from an open panel "only allow read and write, not
  execute" (Quinn, [forum thread 795751](https://developer.apple.com/forums/thread/795751)).
- Opening the TCP connection would need `com.apple.security.network.client`
  ([Apple](https://developer.apple.com/documentation/bundleresources/entitlements/com.apple.security.network.client)).
  That part is easy.
- The blockers are the same whether SANE runs in the app or in a child:
  1. Reading `/opt/homebrew/etc/sane.d/*` and loading `/opt/homebrew/lib/sane/*`.
  2. Executing `/opt/homebrew/bin/scanimage` (spawn case).
  3. The backend's write to `/tmp/stmp_enc.tmp` in colour-JPEG mode.

  Items 1 and 3 would need `temporary-exception.files.absolute-path.*` entitlements,
  which App Store review requires you to justify
  ([Apple: temporary exception entitlements](https://developer.apple.com/library/archive/documentation/Miscellaneous/Reference/EntitlementKeyReference/Chapters/AppSandboxTemporaryExceptionEntitlements.html)).
  *(Untested: the exact sandbox denials were not observed.)*
- Conclusion: with Homebrew SANE as a prerequisite, ship **unsandboxed**, outside the
  Mac App Store. This feeds the map's distribution decision.

## What comparable open-source front-ends do

- **NAPS2** (C#, cross-platform, macOS build notarized). Commit
  [`b5327291`](https://github.com/cyanfish/naps2/tree/b53272914a3f7d6dd15095da82eda7acd6514bb6).
  - **Links `libsane`**, but a **bundled** build of it
    ([`BundledSaneInstallation.cs`](https://github.com/cyanfish/naps2/blob/b53272914a3f7d6dd15095da82eda7acd6514bb6/NAPS2.Sdk/Scan/Internal/Sane/Native/BundledSaneInstallation.cs)).
    It sets `LD_LIBRARY_PATH` and `SANE_CONFIG_DIR` to folders inside the app.
  - Runs SANE in a **separate worker process**. It sets `DYLD_LIBRARY_PATH` on that
    worker ([`WorkerFactory.cs` L24–37](https://github.com/cyanfish/naps2/blob/b53272914a3f7d6dd15095da82eda7acd6514bb6/NAPS2.Sdk/Remoting/Worker/WorkerFactory.cs#L24-37)).
    A code comment admits a SANE crash that "is okay because we're in a worker process"
    ([`SaneScanDriver.cs` L52–53](https://github.com/cyanfish/naps2/blob/b53272914a3f7d6dd15095da82eda7acd6514bb6/NAPS2.Sdk/Scan/Internal/Sane/SaneScanDriver.cs#L52-53)).
  - Entitlements: Hardened Runtime with `allow-dyld-environment-variables` and
    `allow-unsigned-executable-memory`. **No** App Sandbox and **no**
    `disable-library-validation`, because its SANE is signed with its own Team ID
    ([`Entitlements.plist`](https://github.com/cyanfish/naps2/blob/b53272914a3f7d6dd15095da82eda7acd6514bb6/NAPS2.App.Mac/Entitlements.plist)).
  - Cancellation calls `device.Cancel` from a cancellation-token callback (L277–278).
- **simple-scan** (GNOME, Linux). Links `libsane`, and one dedicated thread owns all SANE
  calls, including `sane_cancel`
  ([`scanner.vala` L223–224, L1324](https://gitlab.gnome.org/GNOME/simple-scan/-/blob/698685ef689ba401997ca096bcbe4d5b87e902dd/src/scanner.vala#L223)).
  This is the threading pattern a Swift in-process engine should copy.
- **open-mac-fujitsu-fi-6110** (Swift/SwiftUI, MIT, a tiny project with 0 stars). Commit
  [`1949e445`](https://github.com/federicomarra/open-mac-fujitsu-fi-6110/tree/1949e4456f227a1051d76eaf748fe1acff0cb19a).
  The closest analogue to this repo's goal. It vendors its own build of one backend,
  `dlopen`s it and resolves the `sane_*` symbols directly
  ([`SaneAPI.swift`](https://github.com/federicomarra/open-mac-fujitsu-fi-6110/blob/1949e4456f227a1051d76eaf748fe1acff0cb19a/Sources/ScannerCore/SaneAPI.swift)).
  It is ad-hoc signed only, with no Hardened Runtime and no notarization
  ([`make-app.sh`](https://github.com/federicomarra/open-mac-fujitsu-fi-6110/blob/1949e4456f227a1051d76eaf748fe1acff0cb19a/packaging/make-app.sh)).
  That avoids the signing questions and leaves users with the Gatekeeper warning.
- The pattern across all three: the projects that link `libsane` **own their SANE build**
  (bundled or vendored). None of them loads Homebrew's `libsane` into a signed,
  notarized app. That matches the recommendation: spawn while SANE comes from Homebrew,
  and link only once SANE is bundled.

## Sketch of the v1 engine (spawn)

*(Design notes, untested.)*

- Resolve `scanimage` from `/opt/homebrew/bin`, then `/usr/local/bin`, then `PATH`.
  Show the install instructions if it isn't found.
- Build argv as in `bin/scx-scan`:
  `-d "xerox_mfp:tcp <printer-ip>" --resolution … --mode … --source Flatbed|ADF -p`.
  Output goes to `--format=pnm` on stdout (single page) or `--batch=<tmp>/p%03d.pnm`
  (ADF). Consider `--jpeg=no` if colour scans fail with `ACCESS_DENIED` or collide.
- Read stdout and stderr asynchronously, splitting stderr on both `\r` and `\n`. Map
  `Progress:`, `Scanning page`, `Scanned page` and `Batch terminated` to events.
  Keep the last non-progress stderr line as the error detail.
- Cancel: `interrupt()`. If the child hasn't exited after a few seconds, send a second
  `interrupt()`, then `terminate()`.
- Map `terminationStatus` to `SANE_Status`: 0 = success, 2 = cancelled, 6 = jammed,
  7 = feeder empty, 8 = cover open, 3 = busy, 9 = I/O error.
- Put all of this behind a `ScanEngine` protocol so that a later `libsane` or XPC engine
  can replace it.

## Open questions for the map

- Does the SCX-4623FW advertise JPEG compression? This decides whether the `/tmp` path
  matters and whether `--jpeg=no` is needed. Verify on hardware with
  `SANE_DEBUG_XEROX_MFP=4 scanimage -d … --mode Color`.
- Real cancellation latency, and whether the device needs a pause after a cancelled
  job before accepting the next `sane_start`. Hardware test.
- Distribution: this research implies unsandboxed Developer ID (or unsigned) rather
  than the Mac App Store. That belongs to the distribution ticket.
