# cmux-t3code shutdown behavior

## Symptom

Clicking the red macOS close button only closes the current window. The app does
not fully quit, the dock dot remains visible, and any per-workspace t3code
servers keep running.

This can look like "t3code is stuck in the dock", but that is not what is
happening. The process that remains alive is the main `cmux-t3code.app`
application. The t3code servers are child processes of that app, so they stay
alive as long as the app stays alive.

## What is still running after window close

After closing the last window, these components can still be alive:

- The main `cmux` app process
- The menu bar extra, if enabled
- The local `TerminalController` socket listener, if socket control is enabled
- The session autosave timer
- One t3code sidecar `node` process per workspace that is still open in the
  in-memory session

The dock dot belongs to `cmux`, not to a t3code sidecar.

## Why this happens

Closing a window is not the same as quitting the app on macOS. `cmux` currently
observes window close and unregisters the window, but it does not terminate the
application when the last window closes.

That means the full termination path never runs:

- `applicationShouldTerminate(_:)`
- `applicationWillTerminate(_:)`

Those are the places where `cmux` performs real shutdown work such as:

- saving the final session snapshot
- stopping the autosave timer
- shutting down all t3code sidecars
- stopping `TerminalController`
- stopping other background helpers such as `VSCodeServeWebController`

Because app termination never happens on red-button close, those background
components remain alive.

## Menu bar behavior

The app also has an `NSStatusItem` menu bar extra ("Show in Menu Bar"), and that
setting defaults to `true`.

That does not itself keep the app alive; the app is already still running
because it never opted into "quit after last window closed". But the menu bar
extra makes the still-running state more intentional and visible:

- the app can keep showing notification state
- the menu bar menu still offers app actions
- the menu bar menu includes an explicit Quit action

So the observed behavior is:

1. Red close button closes the window.
2. `cmux` continues running.
3. The dock still shows the app as running.
4. The menu bar extra can still be present.
5. t3code sidecars and the local control socket remain alive until the app
   actually quits.

## What shuts down t3code sidecars

t3code sidecars are shut down in two cases:

1. When the app really quits.
2. When the owning workspace is explicitly closed.

Window close is not enough on its own. If a workspace still exists in the
application state, its sidecar can stay alive even after the window is gone.

## What shuts down the cmux socket

The local `TerminalController` socket is started during app startup when socket
control is enabled. The default mode is `cmuxOnly`, so the socket is normally on
unless the user explicitly turned it off.

That socket is stopped during app termination. It is not stopped just because a
window closed.

## Root cause in code

`cmux` is missing:

```swift
func applicationShouldTerminateAfterLastWindowClosed(_ sender: NSApplication) -> Bool {
    true
}
```

Without that delegate method, closing the last window does not quit the app.

## Practical answer to "what is still running?"

If you close the window and still see the dock dot:

- `cmux` is still running
- the t3code servers are still running because `cmux` is still running
- the socket listener may still be running if socket control is enabled

So the answer is not "cmux or t3code?". It is "cmux, and therefore also its
child t3code sidecars".

## Current workaround

Use one of these instead of the red close button:

- `Cmd+Q`
- `cmux-t3code` menu -> Quit
- the menu bar extra -> Quit

Those paths trigger the real shutdown chain.

## Recommended fix

Add `applicationShouldTerminateAfterLastWindowClosed(_:)` to
`cmux/Sources/AppDelegate.swift` and return `true`.

If a softer behavior is desired, make it conditional on a setting, for example:

- quit after last window closed
- keep running in menu bar when last window closes

That would make the behavior explicit instead of surprising.
