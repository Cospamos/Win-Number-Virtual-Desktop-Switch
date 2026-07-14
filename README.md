# Win+Number Virtual Desktop Switch

A [Windhawk](https://windhawk.net/) mod that replaces the default Windows
Win+1..9 behavior (activate the Nth app pinned/open on the taskbar) with
Linux-style virtual desktop switching:

- **Win+N** switches to virtual desktop N.
- If desktop N doesn't exist yet, it (and any desktops before it) is
  **created automatically**.
- When you switch away from a desktop that has **no windows left on it**
  (windows pinned to all desktops don't count), that desktop is **removed
  automatically**.

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
| Auto-create missing desktops | on | Create desktop N (and any before it) if it doesn't exist yet. |
| Auto-remove empty desktops | on | Remove a desktop you just left if it has no windows on it. |
| Request admin rights for the background helper | **off** | See below. |

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
