# Existing-instance document opening

Windows document launches join the PdfCraft process that owns the resolved
PdfCraft settings profile (the user profile, or portable `PdfCraftData`). The first process starts normally; subsequent launches
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

`--create-images` remains an independent upstream image-import workflow: it neither
forwards to nor owns the document listener. Its paths stay together in the DPI
chooser in its own window, even if a document instance already exists. Subsequent
ordinary document launches still select the document owner, not the image window.

## Integration with current upstream

The original implementation was re-evaluated against `storytold/pdfcraft` main at
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

The tested snapshot `ffb1f49` is preserved locally as
`backup/windows-open-in-tabs-ffb1f49`. Integration with upstream `7c35809` retains:

- Portable settings selection (`settings_dir`) for persistence and IPC identity.
  The profile directory is canonicalized before deriving the pipe name and lock,
  so path aliases rendezvous with the same owner. Separate profiles remain separate.
- File-URI decoding before path validation/forwarding, and Windows Unicode checks.
- Upstream startup order: restore settings and recovery, reopen the previous
  session excluding explicitly supplied files, open initial files, then apply
  startup options before the first received OS-event batch. Forwarded files never
  replay session restoration or startup options.
- Upstream image-import staging, logging, secure control endpoint publication and
  picker-worker synchronization. No new image-import forwarding is introduced.

## Relationship to PR #518

[PR #518](https://github.com/storytold/pdfcraft/pull/518), reviewed at
`9cefa46a1e433e8160d3b1993b06754d8581d95c`, proposes Combine Files shell integration
and a general desktop handoff service. Combine Files is outside this contribution.
The infrastructure overlaps even though the user-facing features differ:

| Concern | This contribution | #518 proposal |
| --- | --- | --- |
| Scope | Windows ordinary document launches | Desktop ordinary opens, Combine and image staging |
| Transport | Owner-scoped named pipe; no network listener | Loopback TCP, advertised port/PID file |
| Ownership | OS-held lock released on process exit | Placeholder file and timed stale-state handling |
| Delivery | Bounded worker queue → OS events → ordinary open | UI polling → ordinary open or staging |
| Secondary control launch | Joins owner; control option ignored | Independent window, no handoff listener |
| Bare launch | Activates document owner | Opens a new window |
| Uncertain acknowledgement | Error; no automatic retransmission | Retry/fallback to another window |

They must not both register independent owners/listeners on Windows. An early
path-only handoff would discard Combine/image staging intent; independent ownership
rules can choose different windows. Different endpoint names do not solve that.

The recommended convergence is **one profile-scoped owner, one bounded inbox and
one UI wake/delivery path**. Once Combine integration is ready, an explicitly
versioned, validated request purpose can distinguish ordinary document opens from
Combine staging. Ordinary opens must still use `open_path`; Combine owns its
staging behavior. Align control-launch, bare-launch and acknowledgement policies
before integrating the callers. This PR adds no speculative request variants or
transport interface.

The tested Windows pipe is the practical starting point here: its owner ACL,
OS-held lock and native race/crash tests meet the current requirement. This is not
a requirement that it replace #518's entire cross-platform design. Loopback TCP
uses standard-library facilities across desktop platforms and may simplify shared
code, but its owner authentication, stale-election races, bounded waits and
ambiguous-delivery behavior need an equivalent review and native tests before
adoption. A loopback address alone is not a per-user authorization boundary.
Maintainers can choose a different shared backend after that validation; activating
two competing services is not an acceptable intermediate state. No Combine shell
verb, Combine handler, TCP listener or Unix forwarding is imported here.

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
| Linux | Existing `Exec=pdfcraft %U` launch behavior and local file-URI decoding remain; separate processes still possible. |
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

Before upstream integration, native Windows CI at `275cee6f5a9adb87c01888a40a3ba851a71dd78f` compiled and
executed the Windows-gated tests in the existing workflow, without adding a job.
They cover named-pipe creation/connection, simultaneous owner election, startup
races, multiple batches, bounded queues, malformed/oversized requests, idle reads,
stalled clients, missing acknowledgement receipts, listener recovery, a killed
primary process and bounded shutdown. The OS-event UI regressions also passed.

Historical results from the completed jobs (not validation of the later merge):

- [Windows](https://github.com/mjshfrvstj-sys/printcraft/actions/runs/37866274912/job/113613646961):
  836 passed, 0 failed, 4 ignored.
- [Linux](https://github.com/mjshfrvstj-sys/printcraft/actions/runs/37866274912/job/113613646817):
  834 passed, 0 failed, 4 ignored (with craft-fonts).
- [macOS](https://github.com/mjshfrvstj-sys/printcraft/actions/runs/37866274912/job/113613646915):
  823 passed, 0 failed, 5 ignored.
- Workspace formatting, all-target Clippy with warnings denied, layering, assets,
  parity, WASM and dependency checks passed. Parity retains the existing 93
  missing-automation-tool notices.
- Local checks also passed: CLI without default features and MSVC-target protocol/
  backend test compilation and Clippy in an isolated dependency harness.

An optional repeat of the Windows job remained queued for over 12 hours without
creating a job and was cancelled. GitHub consequently shows the overall run as
cancelled; the individual completed jobs linked above remain successful. The
repeat supplied no additional execution evidence.

The 15-minute mutation-fuzz job completed 156,818 iterations with zero crashes and
two timeout findings, so it is **not a clean fuzz result** despite its successful
job status. Both findings (`bb2abb8046a87f10` and `1cf87f3b852b5003`) also exceeded
15 seconds in optimized CLI builds of unchanged upstream base `355de90`, using
`check-one <finding.pdf> --dpi 18 --edit`. A sample of the first finding was in
`hayro-jbig2` symbol decoding. The relevant decoder, rendering and CLI sources are
unchanged by this branch. Upstream `7c35809` includes subsequent JBIG2 hardening (#408); these historical
findings do not establish whether current upstream still times out. Corpus-derived
artifacts are not committed here.

Still required on a real Windows desktop: Explorer/Open With/MSI association,
Unicode and multiple-file selection, minimized-window activation, foreground
restrictions, and different accounts/elevation. Native IPC tests do not establish
those shell, focus or security-boundary behaviors. Windows ARM64 and FreeBSD
runtime validation were not performed. macOS/Linux behavior and PR #172 workspace
semantics remain covered by their existing tests and the OS-event regressions.

### Upstream-integration validation

New regressions cover portable versus installed profile ownership, canonical path
aliases, URI/relative/Unicode batches, missing profiles, image-import bypass and
restored-session tabs followed by incoming documents. The original pipe and
PR #172 regressions remain in place. Local integration checks passed: 1,338 tests passed, zero failed, five ignored;
formatting, all-target Clippy, WASM (30 crates), layering (34 crates), assets,
parity and dependency checks passed. Parity reports 95 existing missing-tool
notices. Native CI revalidation is pending for this merge.
The existing CI workflow has a default-off `test_macos` dispatch input so all three
platforms can be tested explicitly without consuming macOS slots on routine runs.
