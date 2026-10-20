# PTY Host Rollover (sub-project G) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** After a PTY update, new panes run the new host and old hosts drain away. Running agents move to the new host
by themselves while idle and off screen, unnoticed.

**Architecture:**
- **One host per build.** The host's instance name becomes `p<protocol>-<version tag>`, and every file the instance
  keys follows it, so two builds of one protocol run side by side in one home. Protocol 3 adds a seeded `create`.
- **Create and drain.** The daemon computes its target instance from the binary it would start. That host is the
  create backend; every other host is a drain backend, reusing today's drain.
- **The mover.** A daemon service moves one idle, quiet, unwatched `conversation` agent at a time. It captures the
  old pane, opens a seeded pane on the new host, relaunches the engine with its resume argv, swaps the row's runtime,
  and closes the old pane. On failure it relaunches in the old pane. A gate holds input and stream opens during the
  swap.
- **On screen.** The desktop reports which tiles are on screen through its existing `app_focus` message.

**Tech Stack:**
- Rust 2024 (`pty/`, alacritty_terminal).
- TypeScript (Node 22) in `cli/`, vitest.
- Flutter/Dart in `desktop/`, one message field.
- GitHub Actions `windows.yml`, `ci.yml`.

**Spec:** `docs/superpowers/specs/2026-10-20-pty-host-rollover-design.md`. Read it first: its Anti-goals are the
acceptance bar.

**Where and when.**
- Branch `feat/windows-pty-rollover`, cut from `feat/windows-desktop`.
- The worktree is `~/harnesses/worktrees/openharness/windows-pty-rollover`; create it if it is missing.
- First commit: this plan and the spec (`docs: PTY host rollover spec and plan`).
- Commits and pushes of `feat/windows-pty-rollover` are authorized. Never push `feat/windows-desktop` or `main`; never
  force-push.
- It merges into `feat/windows-desktop` only with the owner's OK, and ships in the one Windows PR.
- E (`feat/windows-dial`) and F (`feat/windows-installer`) run in parallel. G does not touch their files. If a merge
  meets `windows.yml`, keep both sides.

## Global Constraints

**Platform and behaviour**
- Per user, with no administrator rights and no UAC prompt.
- Scope is Windows, where the PTY host is the terminal backend. tmux is not touched.
- No UI on any client, and no new frame from the daemon to clients. The only new client signal is
  `app_focus.visibleAgents`, sent from the desktop to the daemon.
- Never interrupt a turn, and never end a terminal or an engine that cannot resume. Never lose input. Never give an
  agent a new id.

**The host**
- `PTY_PROTOCOL` becomes `3` (Rust `pty/src/protocol.rs`, TS `cli/src/lib/ptyHost/protocol.ts`).
  `SUPPORTED_PTY_PROTOCOLS` becomes `[2, 3]`, `SHUTDOWN_PROTOCOL` stays `2`, and a new `SEED_PROTOCOL` is `3`.
  A protocol 1 host is now unsupported and is reported `incompatible`, as unsupported hosts are today.
- **Instance name:** `p<protocol>-<tag>`.
  - `<tag>` is the version when it matches `^[0-9A-Za-z.-]{1,32}$`.
  - Otherwise `<tag>` is `h` followed by the first 12 hex digits of SHA-256(version).
  - Legacy hosts keep `p<protocol>`.
- **Files, all in `<home>/run/`:**
  - `pty-host-<instance>.json` (discovery);
  - `.lock`, `.panes` and `.sock`, with the same stem;
  - when `<home>/run/pty-host-<instance>.sock` would be longer than 100 bytes, the socket is
    `pty-host-s<12 hex of SHA-256(instance)>.sock`;
  - the Windows pipe suffix is `-<instance>`;
  - the host log is `<home>/logs/pty-host-<instance>.log`.
  - Legacy names (`pty-host-<protocol>.json`) are still read.

**The mover**
- `QUIET_MS = 300_000`: idle with no input for this long.
- `MOVE_TIMEOUT_MS = 30_000`: the new engine must be proven within this.
- `SEED_MAX_BYTES = 4 * 1024 * 1024`: the oldest lines are dropped first.
- `SEED_HISTORY_LINES = 2000`.
- One move at a time per daemon. It runs as `restartJobs.run(agentId, fn, 'move')`.

**Testing and safety**
- Never against the developer's own `~/.harness`, the daemon on 18473, engine homes or tmux.
- No production publish. A staging publish needs the owner's OK.
- On the ARM64 VM, never touch `C:\work\e2e` or the admin profile's engine settings.
- The repo is public: no addresses, account names, ids or passwords in committed evidence.
- Docs, comments and commit messages are in English. Comments say why.
- Never bare `git stash`, never `git clean -fdx`.
- Conventional commits ending with the engine's `Co-Authored-By` line.

## Review Focus

1. **A version unsafe in a name** (`0.4.21-dev.win1+meta`, a space, 40 characters): the instance falls back to the
   hash tag, and host, daemon and `HARNESS_PANE` parsers all accept it. Pinned by G1.1's and G2.1's shared vectors.
2. **A long home path** pushes the Unix socket past `sun_path`: the hashed socket name is used, and discovery still
   names it. Pinned by G1.1 `a_long_home_uses_the_hashed_socket_name`.
3. **The daemon killed mid-move**, after the old engine stopped and before the swap.
   - After restart the row still names the old pane, which the existing restore relaunches.
   - The new pane carries `HARNESS_MOVE_OF=<agentId>`; it is killed at start before discovery, so the reconciler
     never mints a second agent for it.
   - Pinned by G3.5 `a_move_pane_left_by_a_dead_daemon_is_removed_before_discovery`.
4. **Someone arrives during the swap.**
   - Keystrokes, a paste, or a `harness send` are delivered once, in order, to the new pane.
   - A stream opened then opens on the new pane.
   - Pinned by G3.4.
5. **The old host dies, or its capture fails, before the swap.** The move aborts, nothing is created, and the agent
   is left to the existing drain and restore. Pinned by G3.5 `capture_failure_changes_nothing`.

## File map

| File | Task | Responsibility |
|---|---|---|
| `pty/src/instance.rs` (new), `discovery.rs`, `host_lock.rs`, `pane_ids.rs`, `transport/{unix,windows}.rs`, `logfile.rs`, `cli.rs`, `protocol.rs` | G1.1 | Instance per build; every keyed file |
| `pty/tests/fixtures/instances.json` (new), `pty/tests/common/mod.rs`, `pty/tests/control.rs` | G1.1 | Shared vectors; tests that hard-code names |
| `pty/src/protocol.rs`, `panes.rs`, `pane.rs`, `pty/tests/fixtures/ops.json`, `pty/tests/stream.rs` | G1.2 | `create.seed` |
| `cli/src/lib/ptyHost/instance.ts` (new), `launch.ts`, `protocol.ts`, `client.ts`, `cli/src/lib/terminalRuntime.ts`, every `HARNESS_PANE` parser | G2.1 | Discovery by instance; the target; names |
| `cli/src/lib/ptyHostBackend.ts`, `cli/src/lib/ptyHost/drain.ts`, `cli/src/core/main.ts` | G2.2 | Create and drain per instance |
| `cli/src/lib/uninstall/ptyHosts.ts` | G2.3 | Uninstall ends every instance |
| `desktop/lib/state/app_state.dart`, `desktop/lib/super_tabs/workspace.dart`, `cli/src/localWsServer.ts`, `cli/src/backendSocket.ts`, `cli/src/lib/terminalStreamManager.ts`, `cli/src/core/terminalWatch.ts` | G3.1 | Who is looking |
| `cli/src/lib/hostMove/quiet.ts` (new) | G3.2 | Idle and input clocks |
| `cli/src/lib/terminalTypes.ts`, `ptyHostBackend.ts`, `cli/src/lib/hostMove/seed.ts` (new) | G3.3 | Seeded create; seed from capture |
| `cli/src/lib/hostMove/gate.ts` (new), `terminalStreamManager.ts`, `cli/src/lib/sessionInput.ts` | G3.4 | Hold input and stream opens |
| `cli/src/lib/hostMove/moveAgent.ts` (new) | G3.5 | One move, with its fallback |
| `cli/src/lib/hostMove/mover.ts` (new), `cli/src/core/main.ts` | G3.6 | Which agent, when; wiring |
| `cli/src/lib/hostMove/hostMove.real.spec.ts` (new), `.github/workflows/windows.yml` | G4.1 | Two real builds |
| `docs/testing/pty-rollover-e2e.md` (new), the spec | G4.2 | The VM run; status |

---

## Phase G1: The host

### Task G1.1: One instance per host build

**Files:** `pty/src/instance.rs` (new). Modify `pty/src/{lib.rs,discovery.rs,host_lock.rs,pane_ids.rs,cli.rs,
logfile.rs,protocol.rs}`, `pty/src/transport/{unix.rs,windows.rs}`. Tests: in-file, `pty/tests/fixtures/instances.json`
(new), `pty/tests/common/mod.rs`, `pty/tests/control.rs`.

**Interfaces:**
- Produces (Rust):
  - `pub fn instance_name(protocol: u32, version: &str) -> String`;
  - `pub fn current() -> String` (`instance_name(PROTOCOL, VERSION)`);
  - `pub fn run_file(run_dir: &Path, instance: &str, ext: &str) -> PathBuf` (`pty-host-<instance>.<ext>`);
  - `pub fn socket_path(run_dir: &Path, instance: &str) -> PathBuf`, with the 100-byte rule;
  - `pub fn log_path(home: &Path, instance: &str) -> PathBuf`.
  - `discovery::instance()` is removed; callers use `instance::current()`.
- Produces (shared): `pty/tests/fixtures/instances.json`, an array of `{ "protocol": n, "version": s, "instance": s }`.
  It covers:
  - `0.4.20`;
  - `0.4.21-dev.win1`;
  - `0.1.0`;
  - `0.4.21+meta`, `1 2` and a 40-character version, all hash form.

- [ ] **Step 1: Write the failing tests.**
  - `instance.rs`:
    - `names_follow_the_shared_vectors` reads `../tests/fixtures/instances.json` and asserts each entry.
    - `a_long_home_uses_the_hashed_socket_name`: a run dir of 90 bytes gives `pty-host-s<12hex>.sock`, and a short
      one gives `pty-host-<instance>.sock`.
    - `two_builds_have_different_files`: `run_file` differs for `0.4.20` and `0.4.21`.
  - `protocol.rs`: assert `PROTOCOL == 3`.
  - `tests/control.rs`, a new `two_versions_serve_side_by_side`, run only when `HARNESS_PTY_BIN_OLD` names a build
    of this source with another `HARNESS_PTY_VERSION` (built as in G4.1):
    1. start it and `CARGO_BIN_EXE_harness-pty` on one home;
    2. both publish their own discovery files and both answer hello, with different instances.
  - Update `discovery_names_the_host`, `children_see_harness_pane` and every hard-coded `pty-host-{PROTOCOL}` or
    `p{PROTOCOL}` in `tests/control.rs:75,93,104,127,334,347,376,416,430,434,718,907` and `tests/common/mod.rs:70`
    to go through `instance::current()` and `run_file`.
- [ ] **Step 2: Run them and see them fail.**
  - Run: `cd pty && cargo test --locked instance protocol`.
  - Expected: FAIL. `instance` is not found, and `PROTOCOL` is 2.
- [ ] **Step 3: Implement.**
  - `instance.rs` per the Global Constraints (SHA-256 via the existing `sha2` dependency).
  - Bump `PROTOCOL` to 3, and document `3: create.seed` in its doc comment. Rewrite the comment's "Each value is its
    own host instance" to "each (protocol, version) is its own instance".
  - Route `discovery::file_path`, the `.tmp` name, `host_lock::file_path`, `pane_ids::file_path`,
    `transport::unix::socket_path`, `set_discovery_aside`, the Windows `pipe_name` suffix, `logfile` and
    `Panes::child_env`'s instance through `instance::current()` and `run_file`.
  - The log rotation stays lock-holder-only, now per instance.
- [ ] **Step 4: Run.**
  - Run: `cd pty && cargo test --locked`.
  - Expected: PASS, every unit and integration test.
  - On Windows this runs in `windows.yml` (G4.1); on Linux in `ci.yml`'s `cargo test --manifest-path pty/Cargo.toml`.
- [ ] **Step 5: Commit** `feat(pty): one host instance per build, so two versions run side by side`.

### Task G1.2: A pane created with a seed

**Files:**
- Modify: `pty/src/protocol.rs` (`Op::Create`), `pty/src/panes.rs`, `pty/src/pane.rs` (`PaneSpec`, `Pane::spawn`).
- Tests: `pty/tests/fixtures/ops.json`, `pty/tests/ops_fixture.rs`, `pty/tests/stream.rs`, and `parses_every_op` in
  `protocol.rs`.

**Interfaces:**
- Produces:
  - `Op::Create { …, seed: Option<String> }`, where the JSON field is `"seed"` and its default is absent. Its text is
    the `capture`/`history` style: SGR-only ANSI, one line per row.
  - `PaneSpec.seed: Option<Vec<u8>>`.
- Consumes: G1.1's `PROTOCOL = 3`.

- [ ] **Step 1: Write the failing tests.**
  - `ops.json` gains `"seed": "a\r\nb"` on its `create` entry.
  - `tests/stream.rs` gains `a_seeded_pane_starts_with_its_seed_in_history`:
    1. create with `seed = "one\r\ntwo\r\n\x1b[31mthree\x1b[0m"`, `rows = 10`, and a command that sleeps before it
       prints;
    2. take the snapshot before any output: `history` ends with the three lines in order, colour kept, and
       `capture` is blank with the cursor at 0,0;
    3. after the child prints `ready`, `capture`'s first line is `ready`.
  - `a_create_without_seed_is_unchanged`: today's snapshot is unchanged.
- [ ] **Step 2: Run.**
  - Run: `cd pty && cargo test --locked seed ops_fixture`.
  - Expected: FAIL. `seed` is an unknown field in the fixture check, and the test is missing.
- [ ] **Step 3: Implement.**
  - In `Pane::spawn`, between `Term::new` (`pane.rs:~329`) and `attach`:
    1. advance the seed bytes;
    2. then advance `"\r\n"` repeated `rows` times and `"\x1b[H"`, so the seed sits wholly in scrollback and the
       live screen is blank;
    3. bump `seq` once.
  - `draw_from_host`'s `cfg` gate does not apply here: this is spawn's own code path.
  - Update the TS `PtyOp` create type in the same commit, so the `ops.json` check in
    `cli/src/lib/ptyHost/protocol.spec.ts` stays green.
- [ ] **Step 4: Run.**
  - Run: `cd pty && cargo test --locked` and `cd cli && npx vitest run src/lib/ptyHost/protocol.spec.ts`.
  - Expected: PASS.
- [ ] **Step 5: Commit** `feat(pty): create a pane seeded with another pane's scrollback (protocol 3)`.

---

## Phase G2: The daemon finds hosts by instance

### Task G2.1: Instances, discovery and the target

**Files:**
- Create: `cli/src/lib/ptyHost/instance.ts`.
- Modify: `cli/src/lib/ptyHost/{launch.ts,protocol.ts,client.ts}`, `cli/src/lib/terminalRuntime.ts`, and every
  `HARNESS_PANE` parser. Find them with
  `grep -rn "HARNESS_PANE\|p\\\\d{1,4}" cli/src cli/hook --include=*.ts --include=*.mjs`; today that includes
  `terminalRuntime.ts` `PTY_HOST_PANE`, `hook/notify.mjs`, `messages/command.ts` and `remoteCommand.ts`.
- Tests: `instance.spec.ts` (new), `launch.spec.ts`, `terminalRuntime.spec.ts`.

**Interfaces:**
- Produces:
  - `hostInstanceName(protocol: number, version: string): string`, held to `pty/tests/fixtures/instances.json`.
  - `PTY_HOST_INSTANCE: RegExp`, which is `^p\d{1,4}(?:-(?:[0-9A-Za-z.-]{1,32}|h[0-9a-f]{12}))?$`, used by every
    `HARNESS_PANE` parser as `^(<instance>):(%\d{1,9})$`.
  - `PTY_PROTOCOL = 3`, `SUPPORTED_PTY_PROTOCOLS = [2, 3]`, `SEED_PROTOCOL = 3`.
  - `interface PtyHostTarget { protocol: number; version: string; instance: string }`.
  - `listDiscoveries(home: string): PtyHostDiscovery[]`. It reads every `run/pty-host-*.json`, legacy included,
    keeps only files whose own `protocol` and `instance` fields are well formed, and sorts by protocol, then
    `compareVersions`.
  - `readInstanceDiscovery(home: string, instance: string): PtyHostDiscovery | null`. It reads legacy `p<n>` from
    `pty-host-<n>.json`.
  - `binaryVersion(binary: string): string | null`. It runs `binary version` (one JSON line, `pty/src/cli.rs`
    `version_line`) with a 5 s timeout, `windowsHide: true`, and an argument array. It is cached by path and mtime.
  - `targetHost(options?: PtyHostBinaryOptions): PtyHostTarget | null`. The binary is `ptyHostBinary()`; the
    version is `currentPtyVersion()` when the binary is that folder's, else `binaryVersion()`; null when neither
    is known.
  - `ensurePtyHost(home, options & { target: PtyHostTarget | null })`.
    - With a target, it connects only to that instance's host, else starts the binary.
    - With `null`, it does what it does today: the newest running `PTY_PROTOCOL` host, else start.
    - `startHost` dedupes by home and instance.
  - `reachPtyHost(home: string, instance: string, liveness?)`.
  - `forgetExitedHost(home: string, instance: string, pid: number)`.
  - `discoveredHostDead(home, discovery: Pick<PtyHostDiscovery, 'instance' | 'pid'>, liveness?)`.
  - `discoverHosts(home)` becomes `listDiscoveries`. Its old protocol-only callers go through it.
  - `removeUnusedPtyVersions(home, runtimeDir?)`. Its running set comes from `listDiscoveries`, so every instance
    counts. Its rule is otherwise unchanged: only versions older than `current-pty`; newer folders stay.

- [ ] **Step 1: Write the failing tests.**
  - `instance.spec.ts`: `names follow the host's shared vectors`, which reads `../pty/tests/fixtures/instances.json`.
  - `launch.spec.ts`. Extend the helpers `writeDiscoveryOf(home, protocol, value)` with an instance and file name.
    - `listDiscoveries finds legacy and versioned hosts, oldest first`.
    - `targetHost reads current-pty`.
    - `targetHost asks a dev binary its version` (a fake binary printing `{"version":"9.9.9","protocol":3}`).
    - `targetHost is null when neither is known`.
    - `ensurePtyHost with a target never connects to another version's host` (two fake answering hosts).
    - `ensurePtyHost without a target connects to the newest running host`.
    - `removeUnusedPtyVersions keeps a folder a versioned host runs from`.
    - Update the existing `readDiscovery`, `reachPtyHost`, `forgetExitedHost` and `removeUnusedPtyVersions` cases to
      the instance signatures.
  - `terminalRuntime.spec.ts`: `sessionForPaneId accepts p3-0.4.21-dev.win1:%7 and p3-h0123456789ab:%7`.
  - Add the same case to each other parser's spec.
- [ ] **Step 2: Run.**
  - Run: `cd cli && npx vitest run src/lib/ptyHost src/lib/terminalRuntime.spec.ts`.
  - Expected: FAIL on the new cases.
- [ ] **Step 3: Implement** to the signatures above. `client.ts` keeps speaking the protocol its discovery file names.
- [ ] **Step 4: Run.**
  - Run the same command, plus `npx tsc --noEmit -p .`.
  - Expected: PASS. Fix every type error the signature changes cause in their callers (G2.2 and G2.3 own the
    behaviour; here only compile).
- [ ] **Step 5: Commit** `feat(cli): find pty hosts by instance and pick the installed build as the target`.

### Task G2.2: Create on the target, drain the rest

**Files:**
- Modify: `cli/src/lib/ptyHostBackend.ts`, `cli/src/lib/ptyHost/drain.ts`, `cli/src/core/main.ts:573-613`.
- Tests: `drain.spec.ts`, `ptyHostBackend.spec.ts`.

**Interfaces:**
- Consumes: G2.1.
- Produces:
  - `PtyHostBackendOptions` gains `instance?: string` (a drain backend's host) and `target?: PtyHostTarget | null`
    (the create backend's). `protocol` goes.
  - `draining` is `instance !== undefined`.
  - The drain backend's `instance` is set from the option before it is reached, as `p${protocol}` is today.
  - `PtyHostBackend.mayShutDown: boolean` is false when the drain host's discovery `version` is newer than the
    target's (`compareVersions > 0`), or when there is no target.
  - `ptyHostBackends(home, { target, rowsOn, rowInstances })`:
    - the create backend is the target's;
    - there is one drain backend per discovered instance other than the target, with a supported protocol;
    - there is also one per `rowInstances()` instance with no discovery file, so its rows go gone as today.
  - `rowInstances: () => string[]` returns the `ptyhost:` instances named by registry rows.
  - `PtyHostDrain.step()` skips `shutdownHost` for a backend whose `mayShutDown` is false. Its panes stay reachable.
- Main: the target is computed once at daemon start (`targetHost()`), and logged as
  `[terminal] ptyhost: target <instance>` or `[terminal] ptyhost: target unknown`.

- [ ] **Step 1: Write the failing tests.**
  - `drain.spec.ts`:
    - `an older version's host of the same protocol drains`.
    - `a legacy p2 host drains`.
    - `a newer version's host is reached but never shut down`.
    - `rows on an instance with no discovery get a drain backend`.
    - `an empty older host is shut down after EMPTY_PASSES and its folder removed`.
  - `ptyHostBackend.spec.ts`:
    - `the create backend never makes a pane on a non-target host`.
    - `a drain backend refuses create`, which is today's behaviour, now by instance.
- [ ] **Step 2: Run.** `cd cli && npx vitest run src/lib/ptyHost src/lib/ptyHostBackend.spec.ts`. Expected: FAIL.
- [ ] **Step 3: Implement.** In `main.ts`, pass `target` and `rowInstances`. Everything else around
  `prepareTerminalBackends` is unchanged.
- [ ] **Step 4: Run.**
  - Unit: the same command.
  - Real: `cd pty && cargo +stable build`, then
    `cd cli && HARNESS_PTY_BIN=$(pwd)/../pty/target/debug/harness-pty npx vitest run src/lib/ptyHost/drain.spec.ts src/lib/ptyHostBackend.restart.spec.ts`.
    The drain spec's `HARNESS_PTY_BIN_PREV` case now means "the previous version", built as in G4.1.
  - Expected: PASS.
- [ ] **Step 5: Commit** `feat(cli): new panes go to the installed pty host build; others drain`.

### Task G2.3: Uninstall ends every instance

**Files:** `cli/src/lib/uninstall/ptyHosts.ts`, `ptyHosts.spec.ts`.

**Interfaces:**
- Consumes: G2.1's `listDiscoveries`, `reachPtyHost(home, instance)` and `discoveredHostDead`.
- Produces: `PtyHostEnd` keyed by `instance`; `deps.reach(instance)`; `written` keyed by instance.

- [ ] **Step 1: Write the failing test.** `ends a legacy and two versioned hosts of one protocol in one home`: three
  fake discoveries; each is reached and ended once.
- [ ] **Step 2: Run.** `cd cli && npx vitest run src/lib/uninstall/ptyHosts.spec.ts`. Expected: FAIL. Two hosts of one
  protocol collide.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run.** The same command. Expected: PASS.
- [ ] **Step 5: Commit** `fix(cli): uninstall ends every pty host instance`.

---

## Phase G3: Moving agents unnoticed

### Task G3.1: Who is looking

**Files:**
- Modify: `desktop/lib/state/app_state.dart` (the `app_focus` send, `:4711-4730`), `desktop/lib/super_tabs/workspace.dart`.
- Modify: `cli/src/localWsServer.ts:747-760`, `cli/src/backendSocket.ts:706-710`, `cli/src/lib/terminalStreamManager.ts`,
  `cli/src/core/terminalWatch.ts`.
- Tests: `desktop/test/app_focus_visible_agents_test.dart` (new), `cli/src/lib/terminalStreamManager.spec.ts`,
  `cli/src/localWsServer.spec.ts`.

**Interfaces:**
- Produces (wire): the `app_focus` payload gains `visibleAgents?: string[]`.
  - It holds the ids of every agent whose tile is on screen in that window.
  - It is `[]` while the window is minimized or hidden to the tray.
  - The desktop sends it whenever the set changes: tab switch, layout change, minimize, restore, hide or show.
- Produces (TS):
  - `TerminalStreamManager.setVisibleAgents(connId: string, agentIds: readonly string[] | null): void`, where null
    means the window has never sent the list.
  - `TerminalStreamManager.isLookedAt(agentId: string): boolean`. It is true when any non-closing stream for the agent
    is:
    - from a loopback connection whose visible list is null;
    - from a loopback connection whose visible list contains the agent;
    - from any non-loopback connection.
  - `TerminalWatch.isWatched(agentId: string): boolean`, for Share viewers.
  - `BackendSocket.setLocalTerminalFocus(connId, agentId, visibleAgents?)`.
- Consumes: the existing `app_focus` handling and `setFocusedAgent`. Those stay as they are.

- [ ] **Step 1: Write the failing tests.**
  - TS:
    - `a loopback stream whose window lists other agents is not looked at`.
    - `a loopback stream whose window never sent a list is looked at`.
    - `a mobile or web stream is looked at`.
    - `a minimized window's empty list frees its agents`.
    - `closing the connection forgets its list`.
    - In `localWsServer`: `app_focus with visibleAgents reaches setLocalTerminalFocus`.
  - Desktop:
    - `app_focus carries the on-screen tiles`: two super tabs, only the active tab's tiles are listed.
    - `minimizing sends an empty list`.
- [ ] **Step 2: Run.**
  - Run: `cd cli && npx vitest run src/lib/terminalStreamManager.spec.ts src/localWsServer.spec.ts`, and
    `cd desktop && flutter test test/app_focus_visible_agents_test.dart`.
  - Expected: FAIL.
- [ ] **Step 3: Implement.** The desktop derives the list from the active super tab's mounted, non-`Offstage` tiles
  and the window's visibility state.
- [ ] **Step 4: Run.** The same commands. Expected: PASS.
- [ ] **Step 5: Commit** `feat: the desktop tells its daemon which terminals are on screen`.

### Task G3.2: Quiet clocks

**Files:** `cli/src/lib/hostMove/quiet.ts` (new), `quiet.spec.ts`. Wire it in G3.6.

**Interfaces:**
- Produces `class QuietClock`, constructed with `{ now: () => number }`:
  - `noteInput(agentId: string): void`.
  - `noteActivity(agentId: string, state: ActivityState): void`. It records `idleSince` the first time it sees `idle`
    after anything else, and clears it on `working` or `unknown`.
  - `quietFor(agentId: string): number`. It is `now - max(idleSince, lastInput)`, or 0 when not idle.
  - `forget(agentId: string): void`.
- Fed from:
  - `core/main.ts:1106` `onInput`;
  - the `SessionInputController` submit paths (a `harness send` is input);
  - the mover's own tick, which reads `turnActivity.snapshot(sessionId)`.

- [ ] **Step 1: Write the failing tests.**
  - `idle without input for QUIET_MS is quiet`.
  - `input resets quiet`.
  - `working or unknown is never quiet`.
  - `idle seen again after working restarts the clock`.
- [ ] **Step 2: Run.** `cd cli && npx vitest run src/lib/hostMove/quiet.spec.ts`. Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run.** The same command. Expected: PASS.
- [ ] **Step 5: Commit** `feat(cli): track how long an agent has been idle and untouched`.

### Task G3.3: Seeded create, and a seed from a capture

**Files:** `cli/src/lib/terminalTypes.ts`, `cli/src/lib/ptyHostBackend.ts` (`create`, `:518`), `cli/src/lib/hostMove/seed.ts`
(new). Tests: `seed.spec.ts`, `ptyHostBackend.spec.ts`.

**Interfaces:**
- Produces:
  - `TerminalCreateRequest.seed?: string`. Only the ptyhost backend uses it; tmux ignores it.
  - `PtyHostBackend.create` sends `seed` when `client.protocol >= SEED_PROTOCOL`. It refuses with `not_started`
    ("this host cannot seed a pane") when a seed is given to an older host.
  - `seedFromCapture(text: string, maxBytes = SEED_MAX_BYTES): string`. It normalises line ends to `\r\n` and drops
    whole lines from the top until it fits.
- Consumes: the coordinator's `capture(session, { historyLines: SEED_HISTORY_LINES, ansi: true })` on the old pane.

- [ ] **Step 1: Write the failing tests.**
  - `seed.spec.ts`:
    - `keeps every line under the cap`.
    - `drops the oldest lines first`.
    - `uses CRLF`.
  - `ptyHostBackend.spec.ts`:
    - `create passes the seed to a protocol 3 host`.
    - `create refuses a seed for a protocol 2 host`.
- [ ] **Step 2: Run.** `cd cli && npx vitest run src/lib/hostMove/seed.spec.ts src/lib/ptyHostBackend.spec.ts`. Expected:
  FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run.** The same command. Expected: PASS.
- [ ] **Step 5: Commit** `feat(cli): open a pty pane seeded with another pane's scrollback`.

### Task G3.4: The move gate

**Files:**
- Create: `cli/src/lib/hostMove/gate.ts`.
- Modify: `cli/src/lib/terminalStreamManager.ts` (the input frame path that ends in `tookInput`, `:725`; stream open),
  `cli/src/lib/sessionInput.ts` (`SessionInputController` submit paths).
- Tests: `gate.spec.ts`, `terminalStreamManager.spec.ts`, `sessionInput.spec.ts`.

**Interfaces:**
- Produces `class MoveGate`:
  - `hold(agentId: string): void`.
  - `holding(agentId: string): boolean`.
  - `enqueue(agentId: string, deliver: () => Promise<void>): void`. It queues in arrival order.
  - `whenOpen(agentId: string): Promise<void>`. It resolves at release.
  - `release(agentId: string): Promise<void>`. It runs the queue in order, after the row's runtime has switched (or
    fallen back), so each `deliver` reaches the row's current pane.
- Behaviour:
  - While the gate holds an agent, a stream input frame for it becomes an `enqueue` of a raw write to the row's
    runtime. That uses the backend's raw input op: ptyhost `write`, adding `TerminalBackend.writeInput?(runtime,
    bytes)` if no such method exists.
  - A `SessionInputController` submission is enqueued as itself.
  - A stream open waits `whenOpen`, then opens on the row's current runtime.
  - A held agent counts as looked at from the moment input or an open arrives. The mover reads that through
    `isLookedAt` on its next tick; it does not cancel a swap already past step 3.

- [ ] **Step 1: Write the failing tests.**
  - `input during a hold is delivered once, in order, after release`.
  - `a harness send during a hold is delivered after release`.
  - `a stream opened during a hold opens on the new runtime`.
  - `nothing is held for other agents`.
- [ ] **Step 2: Run.**
  - Run: `cd cli && npx vitest run src/lib/hostMove/gate.spec.ts src/lib/terminalStreamManager.spec.ts src/lib/sessionInput.spec.ts`.
  - Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run.** The same command. Expected: PASS.
- [ ] **Step 5: Commit** `feat(cli): hold an agent's input and new streams while it changes host`.

### Task G3.5: One move

**Files:** `cli/src/lib/hostMove/moveAgent.ts` (new), `moveAgent.spec.ts`. Dependency-injected like `restartAgent.ts`.

**Interfaces:**
- Produces:
  - `type MoveOutcome = { moved: true; resumed: boolean } | { moved: false; reason: string; fellBack: boolean }`.
  - `moveAgent(session: RegisteredSession, deps: MoveAgentDeps): Promise<MoveOutcome>`.
  - `MOVE_MARKER_ENV = 'HARNESS_MOVE_OF'`.
  - `removeOrphanMovePanes(backends: TerminalBackend[], rows: RegisteredSession[]): Promise<number>`, run once at
    daemon start, before discovery.
- `MoveAgentDeps`:
  - `capture(session): Promise<string | null>`;
  - `createSeeded(session, seed: string): Promise<{ ok: true; runtime } | { ok: false; reason }>`. It calls the create
    backend's `create` with `seed`, `hold: true`, the waiting-shell argv restore uses (`waitingLaunch`,
    `core/main.ts:2368`), the row's cwd, and `HARNESS_MOVE_OF=<agentId>`;
  - `respawn(runtime, launch: RestoreLaunch): Promise<{ ok: boolean; reason?: string }>`. It is the backend's
    `respawn` with the launch's argv, cwd and env, as `restoreTerminals.ts` does, and keeps the pane's scrollback;
  - `buildLaunch: RestoreAgentsDeps['buildLaunch']`, from `core/agents/restoreLaunch.ts` `createRestoreLaunch`,
    shared and not copied;
  - `stopEngine(session): Promise<TerminateOutcome>`; the old pane is kept, as `paneSwapDeps` `terminate` does
    (`core/agents/swap.ts`);
  - `probeProcess(runtime, engine): Promise<ProcessIdentity | null>`, polled until `MOVE_TIMEOUT_MS`;
  - `swap(agentId, runtime, identity): void`. In one `registry.transaction` it calls
    `updateRuntimes(agentId, [runtime], terminalRouteKey(runtime))` and `updateProcessIdentity`, then announces the
    session as a restart does;
  - `killPane(runtime)`;
  - `relaunchOld(session): Promise<boolean>`; `restartAgent` in the old pane, with resume;
  - `gate: MoveGate`;
  - `holdRoute` and `releaseRoute`, for both routes for the whole move;
  - `current(): boolean`;
  - `log(message)`.
- Order (spec §4). Two engines never hold one conversation at once:
  1. capture;
  2. buildLaunch;
  3. createSeeded (the new pane waits, with no engine);
  4. `gate.hold`;
  5. stopEngine (old);
  6. respawn (new) with the launch;
  7. probe (new);
  8. swap;
  9. killPane (old);
  10. `gate.release`.

  A failure at 1–3 changes nothing, apart from killing a pane step 3 made. From step 4 on, a failure kills the new
  pane, runs `relaunchOld` if step 5 ran, and then releases the gate.

- [ ] **Step 1: Write the failing tests.**
  - `moves an idle conversation agent and keeps its id`.
  - `capture_failure_changes_nothing`.
  - `a refused create changes nothing`.
  - `a new engine that never appears falls back to the old pane`.
  - `the gate is released on every path`.
  - `a_move_pane_left_by_a_dead_daemon_is_removed_before_discovery`.
  - `a cancelled move (current() false) before step 5 changes nothing`.
- [ ] **Step 2: Run.** `cd cli && npx vitest run src/lib/hostMove/moveAgent.spec.ts`. Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run.** The same command. Expected: PASS.
- [ ] **Step 5: Commit** `feat(cli): move one agent to another pty host, falling back to its old pane`.

### Task G3.6: The mover, and the wiring

**Files:**
- Create: `cli/src/lib/hostMove/mover.ts`.
- Modify: `cli/src/core/main.ts`:
  - after `ptyDrain.step()` in `onReconciled` (`:1809-1816`);
  - the `onInput` at `:1106`;
  - before restore and discovery at start, for `removeOrphanMovePanes`.
- Tests: `mover.spec.ts`.

**Interfaces:**
- Produces `class HostMover`, built with
  `{ list, isDrainRow(row), resumesConversation, activity(row), quiet: QuietClock, lookedAt(agentId), busy(agentId), move(row), now, log }`:
  - `step(): Promise<void>`. At most one move in flight. It picks the oldest-quiet candidate that meets every
    condition in spec §4.
  - Each row has at most one failed move per daemon start (`failed: Set<string>`).
- Wiring:
  - `busy` is `restartJobs.busy(id) || stopJobs.has(id) || restorePanes.busy(id) || pinnedControls.has(id)`.
  - `move` runs inside `restartJobs.run(agentId, …, 'move')`.
  - `lookedAt` is `terminalStreams.isLookedAt(id) || terminalWatch.isWatched(id)`.
  - `isDrainRow` is true when the row's runtime instance is a drain backend's.
- Logs: `[terminal] move <agentId>: moved (resumed|fresh)`, `skipped (<reason>)` and `failed (<reason>)`. The skipped
  line is logged once per reason change, not every tick.

- [ ] **Step 1: Write the failing tests.** One per condition:
  - `a working agent is not moved`.
  - `unknown activity is not moved`.
  - `input within QUIET_MS blocks`.
  - `a looked-at agent is not moved`.
  - `a terminal is never moved`.
  - `a fresh engine is never moved`.
  - `no session id is never moved`.
  - `a busy agent is not moved`.
  - `one at a time`.
  - `a failed agent is not retried until restart`.
  - `an agent on the target host is never a candidate`.
- [ ] **Step 2: Run.** `cd cli && npx vitest run src/lib/hostMove/mover.spec.ts`. Expected: FAIL.
- [ ] **Step 3: Implement, and wire it in `main.ts`.**
- [ ] **Step 4: Run.**
  - Run: `cd cli && npx vitest run src/lib/hostMove src/lib/ptyHost src/lib/terminalStreamManager.spec.ts src/lib/sessionInput.spec.ts && npx tsc --noEmit -p .`.
  - Run the coverage gates the change touches (AGENTS.md): `npm run test:core`, `npm run test:resume`.
  - Expected: PASS. A red test in a suite with known base failures is compared by name against a clean worktree of
    the branch's base.
- [ ] **Step 5: Commit** `feat(cli): move idle, unwatched agents to the installed pty host`.

---

## Phase G4: Proof

### Task G4.1: Two real builds on Windows CI

**Files:** `cli/src/lib/hostMove/hostMove.real.spec.ts` (new), `.github/workflows/windows.yml`.

**Interfaces:**
- Consumes everything above.
- `HARNESS_PTY_BIN` is the new build. `HARNESS_PTY_BIN_OLD` is the same source built with
  `HARNESS_PTY_VERSION=0.0.1-old`. Build the old one first, copy it aside, then build the new one: the env change
  recompiles only the crate.

- [ ] **Step 1: Write the real spec**, run only when both env vars are set. Use a temp home and a stand-in engine
  that resumes: a script that prints its argv and waits for input. Cases:
  - `a pane on the old host keeps streaming and taking input while a new pane opens on the new host`;
  - `a moved pane's history holds the old pane's lines`;
  - `the old host shuts down after its last pane goes, and its version folder is removed`;
  - `input sent during the swap arrives in the new pane`.
- [ ] **Step 2: Run locally first** (the fast loop; Windows CI is the proof).
  - Run: `cd pty && HARNESS_PTY_VERSION=0.0.1-old cargo +stable build && cp target/debug/harness-pty /tmp/<scratch>/harness-pty-old && cargo +stable build`.
  - Then: `cd cli && HARNESS_PTY_BIN=$(pwd)/../pty/target/debug/harness-pty HARNESS_PTY_BIN_OLD=<the copy> npx vitest run src/lib/hostMove/hostMove.real.spec.ts src/lib/ptyHost/drain.spec.ts`.
  - Expected: PASS.
- [ ] **Step 3: Windows CI.**
  - In `windows.yml`'s `windows-pty` job, after the Rust tests, add the old build to the same target and run the real
    spec with both binaries.
  - Add `feat/windows-pty-rollover` to the TEMPORARY push trigger, keeping the other branches.
  - Push once and watch the run on x64 and arm64: `gh run watch <run> --exit-status`.
  - Expected: both pass, and the vitest output names the real cases (not skipped).
- [ ] **Step 4: Commit** `test(windows): pty host rollover against two real builds`, and record the run link in the
  ledger.

### Task G4.2: The VM, the way a person updates; status

**Files:** `docs/testing/pty-rollover-e2e.md` (new), the spec's status line and Known limits.

- [ ] **Step 1: On the ARM64 VM,** with the owner's OK and only when it is free.
  - Use a throwaway standard user, never `C:\work\e2e` and never the admin profile's engine settings.
  - The steps are the spec's Testing list:
    1. install a branch build N;
    2. start a claude agent, a codex agent and a terminal;
    3. install build N+1 with `install.ps1 -Manifest`;
    4. check that a new agent opens on the new host;
    5. keep one tile on screen, put one in another super tab, and minimize;
    6. wait out `QUIET_MS`. Check that the off-screen agents moved, with the same ids, their scrollback, and resumed
       conversations; that the on-screen one did not move; and that the terminal was not touched;
    7. close the terminal. Check that the old host exits and its folder is deleted.
  - Record each item as passed, failed, or not run with the reason, in `docs/testing/pty-rollover-e2e.md`. Leave no
    addresses or ids in it.
- [ ] **Step 2: Clean up** the throwaway user and everything G installed on the VM.
- [ ] **Step 3: Update the spec:** `Status: implemented on feat/windows-pty-rollover; E2E in
  docs/testing/pty-rollover-e2e.md`, and one Known limits line for each limit found and not fixed.
- [ ] **Step 4: Commit** `docs: PTY host rollover E2E and status`.
