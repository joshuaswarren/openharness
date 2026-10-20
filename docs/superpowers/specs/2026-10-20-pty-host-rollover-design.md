# PTY host updates while agents run (sub-project G)

Date: 2026-10-20
Status: draft, for review

## Why

On Windows every agent and terminal runs in a pane of the Harness PTY host, `harness-pty.exe`. Today a PTY update
with the same wire protocol never takes effect while that host runs. It was measured in C5.2 on the ARM64 VM:
- `install.ps1` put PTY 0.4.19 in place and moved `current-pty` to it;
- the host still running was 0.4.18, with seven agents in it;
- the restarted daemon reconnected to that host, and new panes opened on it too.

The new host code (fixes included) runs only after every pane has gone and the host has exited. In practice that
means a reboot. Stopping the host to force the update would kill the person's agents.

A protocol bump already handles this well (`cli/src/lib/ptyHost/drain.ts`):
- the new protocol's host makes the new panes;
- the old host keeps its panes and is shut down once it has none;
- then its version folder is deleted.

G applies the same rule to every host version. It also moves running agents onto the new host by itself, at moments
no one is using them. The person never sees any of this: no notice, no button, no choice to make.

## Goals

1. **New panes run the new code.** After a PTY update, every new agent or terminal opens on a host of the installed
   version.
2. **Running agents are never interrupted by an update.**
   - An agent or terminal on the old host keeps running, streaming and taking input, exactly as now.
   - It stays there until it ends, or until the daemon moves it unnoticed (§4).
3. **Old hosts go away by themselves.**
   - A host of an old version is shut down once it has no pane, and its version folder is deleted, as the drain
     does today.
   - A host of an old version that has no pane when the daemon starts is shut down then, without waiting for a pane.
4. **Running agents move to the new version by themselves, unnoticed.**
   - The daemon moves an agent to the new host only while it is idle and no one is looking at it.
   - A moved agent keeps its agent id, its scrollback and its engine conversation. The next time the person opens
     it, they find the same agent and the same conversation, with the scrollback above it as before.
5. **Nothing to see.** There is no notice, dialog, badge, setting or button for any of this, on any client. The
   update is visible only in the daemon's log.

## Non-goals

- Moving a running process to the new host without restarting it: handing ConPTY handles and screen state from one
  host to another. ConPTY's internals are not documented, and a failure loses the person's agent. Rejected.
- tmux. Scope is Windows only, where the PTY host is the terminal backend.
- Any UI for the rollover, on any client.
- Moving terminals (plain shells): a shell has nothing to resume. A terminal stays on its host until the person
  closes it.
- Moving engines that cannot resume (`fresh` in `resumeCapability.ts`, e.g. `devin`): moving one would lose its
  conversation. It stays on its host until it ends.

## Anti-goals: a failure even if it works

- The person notices the move:
  - an agent interrupted mid-turn;
  - a screen that went blank or lost its scrollback;
  - a tile that flickered or reloaded while they watched;
  - keystrokes lost while they typed;
  - any message about it.
- An update ends a terminal, or an agent that cannot resume.
- A moved agent loses its conversation.
- A moved agent comes back with a new agent id: its tile, links and history break.
- A failed move leaves an agent worse off than if it had not been moved.
- Two hosts left running for good, one of them with no pane.
- A pane that is listed, streams, or takes input on the wrong host, or is listed twice.
- A version folder deleted while a host still runs from it.

## Current state (as of `feat/windows-desktop` at `8fb220f15`)

- **The host** (`pty/src`) has one instance per protocol, `p<PROTOCOL>` (`discovery.rs` `instance()`). That name keys
  its discovery file `<home>/run/pty-host-<protocol>.json`, its lock (`host_lock.rs`), its pane-id counter
  (`pane_ids.rs`, `pty-host-<protocol>.panes`), its Unix socket (`pty-host-<protocol>.sock`) and the
  `HARNESS_PANE=<instance>:%N` its children see. The Windows pipe name is random on each start
  (`transport/windows.rs` `pipe_name_for`, suffix `-p<protocol>`); the lock is what keeps one host per protocol per
  home. The host log `<home>/logs/pty-host.log` is per home, and only the lock holder rotates it (`logfile.rs`).
  The version is any non-empty build string (`cli.rs` `release_version`), unvalidated. The discovery
  file already carries the host's `version` (`cli.rs` `publish_and_serve`). A release stamps that version with the
  CLI's version (`HARNESS_PTY_VERSION`), so every CLI release ships a host with a new version.
- **The daemon** (`cli/src/lib/ptyHost/`):
  - `ensurePtyHost` connects to any running host of `PTY_PROTOCOL`, whatever its version, and starts one only
    when there is none.
  - `ptyHostBackends` makes one create backend (`PTY_PROTOCOL`) and one drain backend for each older supported
    protocol with a discovery file or registry rows.
  - The drain shuts an empty old host down after `EMPTY_PASSES` reconcile passes (`shutdownHost`; protocol 2 has
    the `shutdown` op; a protocol 1 host is left idle).
  - Then `removeUnusedPtyVersions` deletes the version folders older than `current-pty` that no running host
    uses. Newer folders always stay (another install's, perhaps not finished).
  - Uninstall (`uninstall/ptyHosts.ts`) and `discoverHosts` find hosts only through `pty-host-<protocol>.json`.
  - Registry rows name their host by runtime instance (`ptyhost:<instance>`, `terminalRuntime.ts`). The instance is
    an opaque string to everything above `ptyHost/`.
- **Updates on Windows.**
  - Only `install.ps1` writes `<runtime>/pty/current-pty`.
  - `install.ps1` then restarts the daemon once (`Restart-HarnessDaemon`: `harness stop`, then start).
  - The CLI's self-update does not change the PTY.
- **Restarting agents.**
  - `restartAgent.ts` relaunches an agent's engine in the same pane, on the same host.
  - `restoreAgents.ts` (after a reboot) makes a new pane for a row whose pane is gone, and relaunches the engine
    there with its resume argv. It keeps the same agent id.
  - `resumeCapability.ts` says per engine whether a conversation comes back (`conversation`), a shell is relaunched
    (`shell`), or only a fresh session is possible (`fresh`).

## Design

### 1. One host instance per host build

- A host's instance name is derived from both its protocol and its version. It is unique for each (protocol,
  version) pair, and stable across restarts of the same build.
- Everything the instance keys today is keyed by the new name, so two hosts of the same protocol and different
  versions can run side by side in one home:
  - the discovery file, the lock and the pane-id counter;
  - the Unix socket, and the Windows pipe's suffix (its name is already random per start);
  - `HARNESS_PANE`'s instance;
  - the host log, so two hosts never rotate one file.
- The exact format is the builder's. The version is unvalidated, so the name must be made safe from it (a
  sanitised form or a short hash) for a file name, a Windows pipe name and a Unix socket path. The socket path must
  stay inside `sun_path` (104 bytes on macOS), which nothing guards today.
- **Hosts already running** (written by today's builds as `pty-host-<protocol>.json`, instance `p<protocol>`) are
  found and treated as instances of their own. Their panes, rows and routes keep working, and they are drained like
  any old host.

### 2. Which host gets new panes

- The daemon's **target host** is the one it would start: `PTY_PROTOCOL`, and the version of the binary
  `ptyHostBinary()` resolves. On Windows that is the folder `current-pty` names. In a dev build it is whatever
  `HARNESS_PTY_BIN` points to, read with `harness-pty version`.
- The daemon looks at every host discovery file in the home, not only those of the supported protocols.
  - The host matching the target becomes the create backend. It is started when there is none.
  - Every other running host of a supported protocol becomes a drain backend.
  - Hosts of unsupported protocols are reported as they are today.
- When the target version cannot be known (no `current-pty` and no readable binary version), the daemon behaves as
  it does today: it creates on any running host of `PTY_PROTOCOL`.
- **When this is decided** (builder's call, below): at daemon start. On Windows a PTY update always comes with
  `install.ps1`'s daemon restart. When that restart fails, new panes move to the new host at the next restart, and
  `install.ps1` already says so.
- A downgrade (an older version installed) rolls the same way: the target is whatever is installed.

### 3. Old hosts

- The drain is unchanged: listed, streamed and controlled panes; nothing new; shutdown after `EMPTY_PASSES` empty
  passes; then `removeUnusedPtyVersions`.
- `removeUnusedPtyVersions` must never delete a folder a running host (any instance) was started from. Its rule
  otherwise stays: only folders older than `current-pty` go, and newer ones stay, after a downgrade too.
- Uninstall (`uninstall/ptyHosts.ts`) finds and ends every instance's host, legacy and versioned.
- **More than one daemon on one PTY home** (another data folder, or an older daemon whose stop failed) needs no
  coordination, under two rules:
  - a host is shut down only when it has no pane of any owner, as `drainState` already counts;
  - a daemon never drains, shuts down or deletes the folder of a host newer than its own target.
- An old host that has no pane when the daemon starts is shut down on the first passes, as the drain already does.
  Nothing waits for a pane to end.

### 4. Moving running agents, unnoticed

**When an agent is moved.** The daemon moves an agent on an old host only when all of these hold:
- its engine can resume (`conversation` in `resumeCapability.ts`) and its row has a recorded session id;
- its activity is `idle` (`turnActivity.ts`). Never on `working`, and never on `unknown`;
- it has been idle, with no input sent to it, for a quiet period. Five minutes is the starting value (builder may
  tune, see below);
- no one is looking at it (below);
- no other move, restart, stop or delete is in flight for it.

**Who is looking.** An agent counts as looked at when any of these holds:
- a desktop window shows its tile on screen. The desktop already tells the daemon which agent its window focuses
  (`setLocalTerminalFocus`, `TerminalStreamManager.setFocusedAgent`). G widens that local message to carry every
  agent whose tile is on screen in that window. The list is empty while the window is minimized or hidden to the
  tray. A tile kept mounted off screen (another super tab, `Offstage`) is not looked at, even though its stream stays
  open;
- a mobile or web client, or a Share viewer (`terminalWatch.ts`), has a stream open to it. Those clients show one
  terminal at a time and close the stream when they leave it;
- a desktop window that has not sent the list (an older desktop) has a stream open to it. That is the safe default:
  fewer moves, never a visible one.

This is a client-to-daemon signal only. Clients still get nothing new, and nothing changes on screen.

Agents are moved one at a time. When a move starts, its conditions are checked again. Anything that breaks them
before the switch (input, a viewer subscribing, a turn starting) cancels the move, and the agent is left untouched.
An agent that never meets the conditions simply stays on the old host until it ends.

**How a move happens.**
1. The old pane's scrollback is captured from the old host. It already answers `Snapshot`, within that op's limits:
   `historyLines` and the frame size.
2. A pane is opened on the target host, seeded with that scrollback. The host has no way to seed a pane today: G adds
   one. That is a new op, so a protocol bump on the new host is allowed; the old host only needs to answer
   `Snapshot`, as it does now.
   - The bar is: the scrollback above reads as before, and the live screen is whatever the resumed engine draws.
     `claude --resume` and `codex resume` clear and redraw their screen with the conversation, so the visible screen
     cannot be pixel-identical. It does not need to be.
3. The engine in the old pane is stopped. The old pane is kept, not closed.
4. The engine is relaunched in the new pane with its resume argv, keeping the same agent id and row.
   - It builds the launch as `restoreAgents.ts` does (`buildLaunch`, the resume argv, grid launches). That code is
     shared, not copied.
   - The route handling is the move's own: restore commits a route before launch and goes through
     `launch: starting`, which clients see. A move prepares the new pane off to the side, then swaps.
5. Once the engine is proven running in the new pane, the row's runtime switches to it, and the old pane is closed.
   The switch is atomic for clients: none sees the agent twice, or with no pane.
   - Input that arrives between step 3 and the switch is held, and delivered to the new pane once the engine there
     is ready. It is never dropped. The window is seconds.

**When a move fails.** If the relaunch on the new host fails, or is not proven in time:
- the engine is relaunched in the **old** pane, as `restartAgent.ts` does;
- the new pane is closed;
- the agent is marked so it is not tried again until the next daemon start.

To the person, the agent is where it was, with its conversation.

**What is never moved:** terminals, engines that cannot resume, agents with no session id, and grid agents whose row
cannot be relaunched as `restoreAgents.ts` already defines. They stay until they end.

**Logs only.** Every move, skip and failure is logged by the daemon, with the agent id and the reason. Clients get no
new frame for it.

## Constraints

- No administrator rights, no UAC prompt, per user. These are the same as A, B and C.
- Scope is Windows. tmux is not touched.
- Seeding a pane (§4) needs a new op, so the new host's protocol may be bumped. Any bump follows
  `ptyHost/protocol.ts`'s rule: the daemon keeps a client for the protocol before it.
- Hosts and daemons of today's builds keep working: a G daemon drains a `p<protocol>` host. A pre-G daemon still
  running beside a G host (its stop failed during an update) keeps its own `p<protocol>` host; that is accepted.
- Never test against the developer's own machine: the real `~/.harness`, the daemon on 18473, the engine homes,
  tmux.
- The repo is public: no addresses, account names, ids or passwords in committed evidence.
- Docs, comments and commit messages in English.

## Testing

- **Unit (every OS):**
  - instance naming;
  - discovery of every instance in a home, including legacy `pty-host-<protocol>.json`;
  - target choice (installed version, unknown version, downgrade);
  - create and drain backends per host;
  - `removeUnusedPtyVersions` with two running versions;
  - the move's conditions: each one alone blocks a move, and breaking one mid-move cancels it;
  - the move's outcomes: moved; failed and back in the old pane; never-moved kinds left alone.
- **Real host (Mac, and Windows CI x64 and arm64):** two host builds with different `HARNESS_PTY_VERSION`, as in
  `ptyHostBackend.restart.spec.ts`:
  - a pane on the old host keeps streaming and taking input while new panes open on the new host;
  - a pane seeded from an old pane's snapshot shows the same screen and scrollback;
  - the old host is shut down after its last pane ends;
  - its folder is then deleted.
- **On the ARM64 VM, the way a person updates:**
  - install a release;
  - start a claude agent, a codex agent and a terminal;
  - install the next build with `install.ps1`;
  - check that a new agent opens on the new host while the three keep running;
  - close every client and wait out the quiet period. Check that each agent moved one at a time, with the same
    agent id, its scrollback as before and its conversation resumed. Check that the terminal was not touched;
  - with the desktop open, check that an agent whose tile is on screen is not moved, and that one in another super
    tab is;
  - minimize the desktop and check that the agents move;
  - close the terminal and check that the old host exits and its folder is deleted.

## Where and when

- Its own branch, cut from `feat/windows-desktop`, beside E (`feat/windows-dial`) and F (`feat/windows-installer`).
- It merges into `feat/windows-desktop` and ships in the same Windows PR.
- It touches the PTY host (`pty/src`), the daemon's `ptyHost/`, agent-restore, stream and input code, and the
  desktop's local focus message. It does not touch the cable, serial or installer code E and F change.
  `install.ps1` is unchanged unless the builder finds a need.

## Left to the builder

- The instance name's format, within §1's rules.
- How the daemon reads the target version where `current-pty` is absent (dev builds).
- The seed op's shape (a snapshot passed at create, or a separate op before the first output), and whether the
  host keeps the seeded scrollback within `--history-limit`.
- The quiet period's exact value and the move timeout, from measurements. Five minutes is the starting value.

## Calls made in this design

The owner decided:
- the rollover is invisible to the person, with no UI;
- the product decides when to move.

These calls are this design's, not yet confirmed by the owner:
- **The host rolls on every version change**, not only when its code changes. Every CLI release stamps a new host
  version, so after each update a second host runs until the old one's panes end or move. This costs one extra
  host process for that time, and keeps one predictable rule.
- **The target is decided at daemon start**, not before each new pane: on Windows an update always restarts the
  daemon.
- **"Looked at" means on screen.** The desktop reports its on-screen tiles through its existing focus message;
  mobile, web and Share count by open stream; an older desktop counts by open stream.
- **Moves wait for idle, unwatched and quiet.** An agent that is never left alone stays on the old host until it
  ends; that is accepted over interrupting the person.
- **Terminals and engines that cannot resume are never moved.** They hold the old host until they end.
- **The screen bar is the scrollback and the conversation**, not a pixel-identical screen.
- **The move shares restore's launch building** but has its own prepare-then-swap route handling.
- **Input during the swap is held and delivered**, never dropped.
- **Several daemons on one PTY home need no coordination**, under §3's two rules.
- **A downgrade keeps the newer version folder**, as today.
