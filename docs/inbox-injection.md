# Codex Async Inbox — Design Doc

**Status:** Draft (2026-03-28)
**Scope:** SIGUSR1-based inbox injection for running Codex sessions

---

## Problem

Once a Codex session is running there is no mechanism to steer it without killing and restarting
it. Two distinct use cases motivate this:

1. **Sub-agent direction** — an orchestrator agent needs to send corrections or new context to
   a running subordinate Codex session ("you're on the wrong branch", "the dependency you were
   waiting on just finished").

2. **Long-running remote interaction** — a user wants to inject a message into a session running
   in the background or on a remote machine, without attaching to the terminal.

Both cases share the same underlying need: deliver an async message to a running session and
have it processed as a user turn.

---

## Solution: SIGUSR1 inbox injection

Codex listens for SIGUSR1. On receipt, it reads new messages from an inbox directory, injects
each as a user turn, and continues execution. The signal is a delivery notification; the files
are the messages.

### Signal choice

SIGUSR1 and SIGUSR2 are POSIX user-defined signals, available for application use. SIGHUP,
the conventional config-reload signal, is already in use in Codex (forwarded to the child
process), so SIGUSR1 is used for inbox delivery and SIGUSR2 for config reload.

---

## Inbox directory structure (maildir-style)

A single inbox file has race conditions under concurrent writes and loses messages if two
writes arrive in rapid succession (POSIX does not queue signals). The inbox is therefore a
directory using maildir semantics:

```
/tmp/codex-inbox-<pid>/
  new/    ← writers drop messages here (e.g. 1711234567-abc123.md)
  cur/    ← Codex moves messages here when processing begins
  tmp/    ← writers stage here before atomic rename() into new/
```

**Write protocol (sender):**
1. Write message to `tmp/<uuid>.md`
2. `rename()` to `new/<uuid>.md` — atomic on POSIX filesystems, no partial reads

**Read protocol (Codex):**
1. On SIGUSR1, scan `new/` for files
2. `rename()` each to `cur/` before reading (marks as in-flight)
3. Inject contents as a user turn via `steer_input()`
4. `rename()` from `cur/item` to `cur/item.done` after successful processing

The `.done` suffix follows maildir's own flags convention (`:2,<flags>`). On session
restart, anything in `cur/` without `.done` is stranded mid-processing and can be
re-injected, warned about, or discarded. One atomic `rename()` — no extra directory.

The inotify watcher (see below) fires SIGUSR1 on each new file in `new/`, so rapid writes
each get their own signal.

---

## Full pipeline

```
Writers (multiple, independent)             Signaler          Codex
────────────────────────────────────        ──────────        ─────
  Messaging bridge (e.g. Telegram)   ──┐
  Orchestrator agent                 ──┼──► inbox/new/  ──inotify──► SIGUSR1 ──► steer_input()
  Local terminal / script            ──┘
```

### Components

**Inbox directory** — `/tmp/codex-inbox-<pid>/`
Created by the wrapper script on session start. All writers share it; none need to know
about each other or send signals directly.

**inotify watcher** — systemd template unit, instantiated per session
Watches `inbox/new/` for new files. Sends `kill -USR1 <pid>` on each.

```
codex-inbox-watcher@<pid>.service
```

In Python, `watchdog` (`Observer` + `on_created()` callback) is the recommended library —
cross-platform, production-grade, maps directly to the "file appears in `new/`" trigger.

**Wrapper script** (`codex-session`)
Orchestrates startup: launches Codex, captures PID, creates the inbox directory, starts the
watcher unit. Stops the watcher when Codex exits.

```bash
codex "$@" &
CODEX_PID=$!
mkdir -p /tmp/codex-inbox-$CODEX_PID/{new,cur,tmp}
systemctl --user start codex-inbox-watcher@$CODEX_PID.service
wait $CODEX_PID
systemctl --user stop codex-inbox-watcher@$CODEX_PID.service
```

**Messaging bridge** (optional, per-channel)
Listens on an external channel (Telegram, Slack, etc.), writes messages to the inbox using
the write protocol above. Implemented as a separate systemd template unit per session so
multiple channels can feed the same inbox independently.

**Orchestrator agent**
Writes to the inbox directly using the write protocol. Needs the target PID, which the
wrapper script should record in a well-known location (e.g. `/tmp/codex-session.pid` or
a session state file).

---

## Reply convention (bidirectional, opt-in)

There is no reply infrastructure built into Codex. Replies are prompt-level convention.

If the sender wants a response, it includes a reply-to path in the injected message:

```
Summarize what you have accomplished so far.
Write your response to: /tmp/codex-reply-<sender-pid>/<codex-pid>-<uuid>/new/<uuid>.md
```

The reply-to path ideally follows the same maildir directory convention as the sender's
inbox, so the reply can be watched by the same inotify mechanism. File naming includes
both PIDs to avoid collisions when multiple sessions are running concurrently.

The sender can:
- Watch the reply directory for new files
- Record "pending reply from session X" in session state for later pickup
- Omit the reply-to path entirely for fire-and-forget messages

This is a **per-message decision**, not a per-session configuration.

---

## Evolution path

```
v1    Single inbox file + SIGUSR1 (initial implementation)
v1.5  Maildir inbox directory — race-safe, multi-message, audit trail (this doc)
v2    Wrapper script + inotify watcher + systemd template units
v3    Messaging bridge (Telegram or similar)
v4    Orchestrator records PID in session state; writes to inbox natively
v∞    Internet pub/sub (MQTT/NATS) when agents are distributed across machines
```

---

## Upstream PR scope

The upstream PR covers v1 only:

- SIGUSR1 handler in Node.js launcher (`codex-cli/bin/codex.js`)
- SIGUSR1 inbox reader in Rust core (`codex-rs/core/src/codex.rs`)

The maildir structure, wrapper script, inotify watcher, and messaging bridge are
application-layer concerns that do not require changes to Codex itself.

---

## Open questions

- Should Codex write its PID to a well-known path (e.g. `/tmp/codex-session.pid`) on
  startup to simplify orchestrator discovery? (Small Rust change.)
- Should the inbox path be configurable via env var only, or should there be a
  `config.toml` key as well?
- For the reply-to convention: should there be a standard header format in the message
  file (e.g. `Reply-To: /tmp/...`) rather than embedding the path in the message body?
