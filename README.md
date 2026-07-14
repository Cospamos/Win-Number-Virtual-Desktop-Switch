# Win+Number Virtual Desktop Switch

A [Windhawk](https://windhawk.net/) mod that replaces the default Windows
Win+1..9 behavior (activate the Nth app pinned/open on the taskbar) with
Linux-style virtual desktop switching:

- **Win+N** switches to virtual desktop N.
- **Win+Shift+N** moves the currently focused window to desktop N, without
  switching your own view away from the desktop you're on.
- If desktop N doesn't exist yet, it (and any desktops before it) is
  **created automatically**.
- When you switch away from a desktop that has **no windows left on it**
  (windows pinned to all desktops don't count), that desktop is **removed
  automatically**.
- No taskbar flash asking you to switch back after switching desktops (see
  below).
- No "Desktop N" name overlay popping up on builds where that experimental
  animation is enabled (see below).

## Installation

1. Install [Windhawk](https://windhawk.net/) and enable developer mode.
2. Create a new mod and paste in the contents of
   [`virtual-desktop-win-number-switch.wh.cpp`](./virtual-desktop-win-number-switch.wh.cpp).
3. Compile and enable it.

## How it works

The mod runs as a dedicated background process (a Windhawk "tool mod") and
installs a low-level keyboard hook that intercepts Win+1..9 system-wide
before Explorer's own taskbar shortcut handling sees them, then talks to the
undocumented Virtual Desktop COM interfaces (`IVirtualDesktopManagerInternal`
and friends) to switch, create, and remove desktops.

## Settings

| Setting | Default | Description |
|---|---|---|
| Windows Version | `auto` | COM interface IDs differ between Windows builds; auto-detected from the registry, or override manually. |
| Enable Win+Number desktop switching | on | Master toggle for the Win+N remap. |
| Enable Win+Shift+Number move window to desktop | on | Master toggle for the Win+Shift+N remap. |
| Auto-create missing desktops | on | Create desktop N (and any before it) if it doesn't exist yet. |
| Auto-remove empty desktops | on | Remove a desktop you just left if it has no windows on it. |
| Prevent taskbar flash after switching desktops | on | See below. |
| Hide the "current desktop" name overlay | on | See below. |
| Request admin rights for the background helper | **off** | See below. |

## Taskbar flash after switching desktops

Windows sometimes can't silently restore focus to a window on the desktop
you just switched to (the request comes from a background process), and
falls back to flashing a taskbar button asking you to switch back to where
you came from. Before switching, this mod hands focus to Explorer's own
taskbar window (`Shell_TrayWnd`) first, so Explorer's internal focus-restore
runs in the shell's own context instead of getting denied, then takes real
focus for the actual target window itself right after (with a couple of
retried `FLASHW_STOP` calls as a safety net for anything Explorer schedules
slightly late). Turn off "Prevent taskbar flash after switching desktops" in
settings to get Windows' default (flashing) behavior back.

## Hiding the "current desktop" name overlay

Recent Windows 11 builds can show a small on-screen label with the desktop's
name/number when you switch (gated behind an internal, experimental Windows
feature flag - there's no regular Settings toggle for it). Live-monitoring
window events during a switch identified it as a `XamlExplorerHostIslandWindow`
hosted by `explorer.exe` - a class shared by several unrelated shell flyouts
(volume, network, etc.), so this mod doesn't hide that class unconditionally,
only an instance of it tied to a switch this mod itself triggered.

We also tried injecting directly into explorer.exe and hooking
`CreateWindowExW`/`ShowWindow`/`SetWindowPos` to veto the window before it's
ever painted (for a guaranteed zero-flicker fix). Confirmed live that none of
those three calls ever fire for this window - its visibility is toggled
through DirectComposition/DWM instead, which isn't something we can hook
this way - so that approach was reverted; it added risk (patching functions
the whole shell uses constantly) for no benefit.

What the mod does instead: it watches for that window via `SetWinEventHook`
and hides it (`ShowWindow` + moves it off-screen) the instant it sees it, and
since the same window is reused across switches rather than recreated, it
also proactively re-hides that same window right before the *next* switch
even starts - so on repeat switches there's often nothing to react to at
all. This is reactive rather than fully preventive, so a very brief flicker
can still be possible in rare cases, but it should be hard to notice in
practice. There's deliberately no blocking retry loop here - that was tried
and caused noticeable stutter/freezing on fast repeated switches, since it
blocked the same thread that processes the next switch request. A single
fast hide attempt is enough in practice, since the overlay's own lifecycle
already fires several separate events per switch, each giving this mod
another shot at hiding it without blocking anything. Turn off "Hide the
'current desktop' name overlay" in settings to get Windows' default
behavior back.

If you'd rather disable the underlying animation feature entirely (which this
overlay is part of) at the OS level instead, use
[ViVeTool](https://github.com/thebookisclosed/ViVe) as Administrator:

```
vivetool /disable /id:42354458,34508225,40459297
```

then restart. Note these feature IDs are shared with some Cloud PC/Windows
365 switching functionality, so treat this as an experimental tweak.

## A note on admin rights (UIPI)

Windows blocks a normal-privilege keyboard hook from seeing keystrokes while
a higher-privilege ("Run as administrator") window has focus. By default
this mod **never** requests elevation - Win+N just won't intercept while an
elevated app (Task Manager, an admin terminal, Docker Desktop, etc.) has
focus.

If you want Win+N to keep working in that case too, turn on "Request admin
rights for the background helper" in the mod's settings. It's off by
default: the mod never asks for elevation without your explicit opt-in. When
enabled, you'll get a single UAC prompt when the mod starts, instead of
having to manually relaunch Windhawk itself as administrator every time.

## Compatibility

Verified on Windows 11 22621+ (23H2/24H2/25H2). The Virtual Desktop COM
interface IDs and vtable layout change between Windows builds; older
Windows 10 builds use best-effort values that may need adjusting - see
`g_versionIIDs` in the source, and the mod's debug log for the detected
build number.

## License

MIT
