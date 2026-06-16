# Bug: tmux navigation silently fails when user is attached to a different session

## Summary

Clicking a Claude session in the menu bar activates the terminal app (Kitty) but does not switch to the tmux pane where the session is actually running — when the user's tmux client is attached to a different session than the one containing the target pane.

`tmux select-window` and `tmux select-pane` both return exit code 0 in this scenario, so the Swift code sees a successful operation. The change actually happens on the tmux server, just in a session the user is not currently viewing.

## Environment

- macOS (Apple Silicon)
- Terminal: Kitty with multiple tmux sessions attached (e.g. one for daily work, several for project work)
- Claude Status: installed from releases, plugin installed via Settings
- Multiple active Claude sessions, each in a tmux pane in different tmux sessions

## Steps to reproduce

1. Start tmux and create two sessions: `tmux new -s A` and `tmux new -s B -d`
2. From Kitty, attach to session A: `tmux attach -t A`
3. In a pane in session A, start claude: `claude`
4. In session B, start another claude: `tmux send-keys -t B 'claude' Enter`
5. In Claude Status, click the menu bar entry for the session running in session B
6. Observe: Kitty comes to the front, but the active pane in session A is unchanged. The pane in session B is now selected *within session B's server state*, but session B isn't visible.

## Expected behavior

Clicking a Claude session in the menu bar should make that session's tmux pane visible and active in Kitty — including switching the tmux client's view to the correct session if necessary.

## Actual behavior

Kitty is brought to the front, but no tmux navigation occurs. The session in the menu bar and the pane the user is actually looking at are unrelated after the click.

## Root cause

`focusTmuxPane` in `Claude Status/SessionDiscovery/TerminalFocuser.swift` runs:

```swift
// Select the window containing the target pane
let selectWindow = Process()
selectWindow.executableURL = URL(fileURLWithPath: tmuxBin)
selectWindow.arguments = (usesEnv ? ["tmux"] : []) + baseArgs + ["select-window", "-t", paneId]
try? selectWindow.run()
selectWindow.waitUntilExit()
```

followed by `select-pane -t paneId`.

These commands update the **active window and active pane of the session that contains the pane**. They do not move the user's tmux client into that session. If the client is attached to a different session, the change is invisible to the user.

`tmux select-window` and `tmux select-pane` both return exit code 0 in this case — the server operation succeeded; it's just that no client is viewing the result. The Swift code uses `try?` and checks no exit codes, so the failure mode is completely silent.

### Diagnostic evidence

With `os_log` instrumentation added to `focusTmuxPane`, clicking a session produced:

```
focusTmuxPane start: paneId=%5 socket=/private/tmp/tmux-502/default tmuxBin=/opt/homebrew/bin/tmux
select-window: exit=0 stderr=(empty)
display-message: zoomFlag=0 exit=0
select-pane: exit=0 stderr=(empty)
focusTmuxPane done
```

All three tmux commands reported success. `tmux list-clients` then showed the attached client on session `DOTS`, while `tmux list-panes -a` showed pane `%5` in session `PRJ - INTERNAL - PMS - TESTING`. The selection happened in the latter session, but the user was looking at the former.

## Suggested fix

Before calling `select-window`, determine which session contains the target pane and move the user's tmux client into it. Concretely:

1. Look up the session for the pane:
   ```
   tmux list-panes -t %<id> -F '#{session_name}'
   ```
2. Look up the session the client is currently in:
   ```
   tmux display-message -p -F '#{session_name}'
   ```
3. If they differ, move the client:
   ```
   tmux switch-client -t <target_session>
   ```
4. Then proceed with the existing `select-window` and `select-pane`.

A reference implementation in Swift (added to `focusTmuxPane`) is included in commit `<sha>` of the main-app fork.

## Additional improvements

The silent-failure pattern in this function is worth addressing in its own right:

- Replace `try?` with proper error handling and pipe `standardError`, so any future tmux error surfaces in Console.app / `log show`.
- Check each `Process.terminationStatus` and log it via `os.Logger`.
- Log the resolved `paneId`, `socket`, `tmuxBin`, and `usesEnv` at the top of the function for diagnostic clarity — this immediately reveals stale data, wrong socket, or wrong binary path.

A prepared `os_log` instrumentation is included in the same commit. Whether to keep or strip it in the upstream PR is a judgment call, but the underlying `switch-client` fix is the substantive change.

## Related

A separate, real but silent bug exists in the Rust plugin (`get_ppid_of` in `crates/session-status/src/main.rs`): it reads byte offset 24 of `proc_bsdinfo`, which is `pbi_gid`, not `pbi_ppid` (which is at offset 16). On macOS, a user's primary GID is typically 20 (the "staff" group), which is what was leaking into the `.cstatus` file's `ppid` field. This does not affect navigation but corrupts on-disk data. Fix: use a `#[repr(C)]` struct and a `std::mem::offset_of!` compile-time assertion. (Tracked in plugin fork, separate commit.)
