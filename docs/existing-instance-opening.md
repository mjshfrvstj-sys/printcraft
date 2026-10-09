# Existing-instance document opening

Windows document launches join the PdfCraft process that owns the current user's
PdfCraft settings profile. The first process starts normally; subsequent launches
forward their document paths and exit after the primary accepts the batch. A launch
without paths requests window activation. Secondary launch options, including
`--mode` and `--control`, are not forwarded or applied to the running session.

Options are parsed before instance selection. Unknown `--name` options also consume
their next argument as a value and are ignored by a secondary; do not put a document
path after an option that is missing its value. A secondary `--control` launch does
not create an endpoint file. To configure the primary session, supply startup
options when no instance is running, or use its already-enabled control channel.
When parsed as an option, `--version` prints the version and exits before election,
forwarding or activation, even if document paths preceded it.

## Integration with current upstream

This implementation was re-evaluated against `storytold/pdfcraft` main at
`355de9038e5bd09147c6b3c81dc557659b453c1c`, after reviewing the 34 first-parent
commits and merged PRs since prototype `153235e`'s base, `1c78ff4`.

- The rename (#174) changed application identity, settings and recovery folders.
  Legacy folder migration runs **before** IPC creates the PdfCraft settings folder;
  otherwise that new folder would prevent migration. Legacy settings-key fallback
  remains unchanged.
- The default workspace preference (#172), including the maintainer's panel fix,
  stays in the ordinary document-opening path. IPC does not select a mode or tool.
  A session mode override wins; without one, upstream selects `default_mode` only
  when it differs from the current mode, retaining whether the left panel is open.
  When the mode already matches, the chosen tool panel is retained.
- Async macOS file pickers (#145), Apple Events and PDF UTI registration (#176)
  remain intact. Windows MSI association (#183) already passes a quoted document
  path to the executable. Linux monitor-aware GPU startup (#105) is unchanged.
- Startup still restores settings, opens initial paths, then applies startup
  options before the first UI frame. Received batches are processed after that
  initialization. PDF Initial View metadata remains independent.

The prototype's path-only protocol, owner-scoped pipe, OS lock, acknowledgement,
OS-event routing and best-effort focus are retained. Its integration was rebuilt
on current upstream rather than replayed. The receiver now permits eight bounded
concurrent clients, and Windows-specific nonblocking error handling lives in the
pipe adapter rather than the portable protocol. No generic transport interface or
new document loader was introduced.

## Windows transport and security

`windows_instance.rs` uses a Windows named pipe through the safe, MIT/Apache-2.0
`interprocess` 2.2.3 API. A stable SHA-256 hash of the settings path separates pipe
names; it is **not** authentication. The pipe has a protected owner-rights DACL
(`D:P(A;;GA;;;OW)`), rejects remote clients and does not inherit handles. A client
opens the pipe with anonymous security quality of service to prevent the server
from impersonating the launcher. This is local document delivery, not the opt-in
TCP UI control channel or MCP, and it exposes no commands.

Its code-only dependencies `doctest-file` 1.1.1 and `recvmsg` 1.0.0 carry reviewed
0BSD licenses. `deny.toml` allows that license only for those exact versions;
the global license allowlist and asset policy are unchanged.

An OS-held file lock elects one primary. The file is never unlinked or truncated;
its contents and existence are irrelevant. Windows releases the lock on process
exit, including crashes. Launches racing startup retry election/connection within
10 seconds. A disconnected pending pipe is recreated while ownership is held.
There is no PID file whose stale contents could select the wrong process.

`open_protocol.rs` is a small framing/validation helper, not an IPC framework:

- A four-byte little-endian length followed by version-1 JSON containing only
  `version` and `paths`; arbitrary commands and options are rejected.
- At most 128 KiB per frame and 256 paths; length is checked before allocation.
- Paths are absolute, Unicode strings without NULs. Relative paths are resolved
  in the launcher. Non-Unicode Windows arguments return an error instead of
  panicking. Spaces and Unicode paths are covered by tests.
- At most 16 queued batches and eight active clients. Each client has a two-second
  I/O deadline. No blocking flush is used. Worker shutdown waits only for bounded
  client work.

Windows idle nonblocking reads can surface as zero bytes through `std::fs::File`.
The pipe adapter treats these as temporary unavailability, not end-of-file. This
also means a disconnected peer may wait until the two-second deadline. The native
idle-read regression covers the premature EOF/acknowledgement race discovered by
the first Windows CI run.

The receiver validates and enqueues a whole batch, then acknowledges acceptance.
The launcher consumes the acknowledgement and sends a receipt before closing.
Acceptance means **queued in memory**, not parsed successfully or durably saved.
A primary crash after acceptance can lose pending delivery. A lost acknowledgement
can leave a launcher uncertain even though files were queued. There is no automatic
resend after transmission; the error dialog asks the user to check the existing
window before retrying, which avoids silently duplicating tabs.

The egui poller drains batches as `OsEvent::Open` and requests repaint. The existing
event handler calls `PdfCraftApp::open_path` for every path, preserving existing
tabs and unsaved work. Invalid PDFs produce ordinary opening errors. Restoring and
focusing the viewport is best effort; Windows foreground restrictions may prevent
focus even when delivery succeeds.

The profile is the instance scope, including sessions sharing that profile.
Processes of the same user are not treated as a security boundary. Mixed elevation
may be denied by Windows integrity/owner rules and is not bypassed; launch failures
are reported instead of starting a competing window. Native testing must verify
the ACL and elevation behavior on supported Windows installations.

## Platform scope and natural follow-up

| Platform | Behavior in this contribution |
| --- | --- |
| Windows | Local owner-scoped pipe forwards multiple paths into existing tabs; OS lock elects/reclaims primary; focus requested. |
| macOS | Existing Launch Services/Apple Events already deliver Finder, Open With and Dock documents to the app's OS-event path. No extra IPC. |
| Linux | Existing `Exec=pdfcraft %F` launch behavior remains; separate processes still possible. |
| FreeBSD | Existing native launch behavior remains; no new IPC. |
| Web | Not applicable to native OS process/file-association launches. |

A follow-up for Linux and FreeBSD can use pathname Unix-domain sockets through
`std::os::unix::net`, without a new dependency. The protocol can be reused, but the
backend needs its own private runtime directory, socket permissions, short path,
owner election, stale-socket cleanup and race tests. Sandbox namespaces can limit
socket visibility. Those are substantive platform decisions, so this contribution
does not add a speculative Unix backend or a common transport trait.

D-Bus offers standard desktop activation and brokered service discovery, including
`org.freedesktop.Application.Open`. It requires a session bus, service/desktop
packaging and sandbox policy; a direct Rust API also adds dependencies beyond the
current app (transitive portal usage is not an application activation service).
This can be worthwhile for Linux desktop integration but is less uniform on
FreeBSD or minimal sessions. UDS is the smaller portable Linux/FreeBSD follow-up;
D-Bus should be chosen separately if packaging and sandbox requirements favor it.

Primary references: [Windows pipe security](https://learn.microsoft.com/en-us/windows/win32/ipc/named-pipe-security-and-access-rights),
[Rust UnixListener](https://doc.rust-lang.org/std/os/unix/net/struct.UnixListener.html),
[desktop D-Bus activation](https://specifications.freedesktop.org/desktop-entry/latest/dbus.html),
[portal sandbox integration](https://flatpak.github.io/xdg-desktop-portal/docs/for-app-developers.html).

## Validation boundaries

Portable tests cover schema/framing limits, malformed input, partial I/O, timeouts,
Unicode paths, migration and OS-event routing. UI regressions cover multiple tabs,
unsaved work, missing files within a batch, closed panels, selected tools and
explicit session modes. Existing `doc_open` automation remains the headless opening
API; this transport does not add another command surface.

Windows-gated tests cover owner election, simultaneous launches, multiple batches,
bounded queues, oversized requests, stalled clients, missing acknowledgement
receipts, listener recovery and a killed primary process. Cross-compiling these
tests verifies types, not runtime behavior. Before release, run them on Windows and
exercise Explorer/Open With/MSI association, Unicode and multiple paths, minimized
window activation, different accounts/elevation, and crash/relaunch. macOS-hosted
checks cannot establish those Windows behaviors.

Validation on the macOS development host for this upstream snapshot:

- Workspace formatting and all-target Clippy with warnings denied: passed.
- Workspace tests: 823 passed, 0 failed, 5 ignored (network release lookup, two
  optional corpus tests, timing probe and temporary Keychain test).
- Layering: 31 crates, no violations. WASM: all 27 checked crates passed.
- Assets: 247 repository, 6 dependency-bundled and 22 build-time assets passed.
- Parity: passed, 829 entries; the existing 93 missing-automation-tool notices
  remain. Dependency advisories, bans, licenses and sources: passed.
- CLI without default features: passed.
- Actual protocol and Windows backend source, including Windows tests: MSVC-target
  `cargo check --tests` and Clippy passed through an isolated dependency harness.
- Full Windows app cross-check: blocked in existing `ring`/`aws-lc-sys` C builds
  by the missing Windows SDK (`assert.h`). Windows tests were not executed;
  native Windows, Linux and FreeBSD execution was unavailable on this host.
