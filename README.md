# GNOME Wine Tray Fix

Put Wine launcher icons in your GNOME tray instead of a separate tiny window.

This is a setup guide and two autostart templates for KDE's **xembedsniproxy**. It is not a Wine patch and does not modify games, Proton, or anti-cheat components. KDE developed the actual bridge; this project documents how to use it with GNOME.

## What it fixes

Some Wine launchers use the legacy X11/XEmbed tray protocol. If your GNOME tray extension only supports StatusNotifierItem (SNI), Wine may display a separate window containing the icon.

`Wine tray icon → xembedsniproxy → StatusNotifierItem → GNOME tray extension`

With the bridge, the launcher can remain in the tray after closing the game. Its own menu can then reopen the launcher or exit it normally, where the launcher supports those actions.

This does **not** fix Wine's Windows-style title bars, missing maximize buttons, or game compatibility.

## Tested setup

- CachyOS, GNOME 51 on Wayland with XWayland
- GNOME extension: Status Tray (`status-tray@keithvassallo.com`)
- Lutris / UMU and DW-Proton 11.0-13
- GODDESS OF VICTORY: NIKKE launcher
- KDE xembedsniproxy from plasma-workspace 6.8.2

In this setup, starting the bridge while the launcher was running moved the icon into the tray and removed Wine's fallback window. Other desktops, extensions and launchers are not tested here. This is not a statement of game publisher approval or a guarantee about account policies.

## Requirements

- GNOME with XWayland and a working **StatusNotifierItem** tray extension.
- A systemd user session.
- KDE's `xembedsniproxy` and its runtime libraries.
- No other legacy X11 tray manager already running. Do not run two bridges at once.

Check the SNI host:

```sh
busctl --user get-property org.kde.StatusNotifierWatcher /StatusNotifierWatcher org.kde.StatusNotifierWatcher IsStatusNotifierHostRegistered
```

The result should be `b true`. If the service is missing, enable a compatible GNOME tray extension first.

## Get xembedsniproxy

First check whether it is already installed:

```sh
command -v xembedsniproxy
```

Use your distribution's packages if available. KDE includes this program in **plasma-workspace**, though distributions may split it into a separate package. Review dependencies before installing a full Plasma workspace just for this helper.

### Minimal Arch / CachyOS approach

The following downloads the distribution's signed Plasma package without installing the desktop, then extracts only the helper. It assumes Bash, `pacman`, `bsdtar`, and the standard pacman cache location. It is intended for an up-to-date system; do not run `pacman -Sy` to create a partial upgrade.

```bash
sudo pacman -S --needed kdbusaddons kwindowsystem kcoreaddons kcrash
sudo pacman -Swdd plasma-workspace

# Resolve the exact package filename from the same repository metadata.
package_name=$(pacman -Spdd --print-format '%f' plasma-workspace)
test -f "/var/cache/pacman/pkg/$package_name" || exit 1

mkdir -p "$HOME/.local/lib/gnome-xembed-tray"
bsdtar -xOf "/var/cache/pacman/pkg/$package_name" usr/bin/xembedsniproxy \
  > "$HOME/.local/lib/gnome-xembed-tray/xembedsniproxy" || exit 1
chmod 755 "$HOME/.local/lib/gnome-xembed-tray/xembedsniproxy"
ldd "$HOME/.local/lib/gnome-xembed-tray/xembedsniproxy"
```

Stop if `ldd` reports a library as **not found**; install its distribution package before continuing. Package contents and dependencies can change.

**Maintenance:** this extracted binary is a local copy. The package manager updates its installed libraries, but does not update the copy. Repeat the download/extraction when needed, with the bridge stopped. A distribution-managed standalone package is preferable when available. No KDE binaries are distributed by this repository.

## Configure GNOME autostart

Clone/download this repository and open a terminal in its directory. Inspect any existing files with these names before replacing them.

```sh
mkdir -p "$HOME/.config/systemd/user" "$HOME/.config/autostart"
cp -i templates/gnome-xembed-tray.service "$HOME/.config/systemd/user/"
cp -i templates/gnome-xembed-tray.desktop "$HOME/.config/autostart/"
```

The service template uses the local binary path from the Arch/CachyOS instructions. If your distribution installed it elsewhere, edit `ExecStart` in the copied service to use that absolute path, for example `/usr/bin/xembedsniproxy`.

Run these commands from a terminal in your **GNOME desktop session**, as your normal user:

```sh
systemctl --user import-environment DISPLAY XAUTHORITY
systemctl --user daemon-reload
systemctl --user start gnome-xembed-tray.service
```

The desktop file starts the service at future GNOME logins. It has `OnlyShowIn=GNOME;` so it does not start another bridge in Plasma. The service stops with the graphical session. No `systemctl enable` is needed.

You can leave the launcher running for the first attempt. If its icon does not move, close the game and launcher normally, then reopen them with the bridge already running. You do not need to restart GNOME.

## Verify / troubleshoot

```sh
systemctl --user status gnome-xembed-tray.service
journalctl --user -u gnome-xembed-tray.service -n 30 --no-pager
busctl --user get-property org.kde.StatusNotifierWatcher /StatusNotifierWatcher org.kde.StatusNotifierWatcher RegisteredStatusNotifierItems
```

Confirm that the launcher icon appears, its menu works, and the small fallback window disappears. A running service alone does not prove successful icon forwarding.

- **No SNI host:** enable/fix your GNOME tray extension.
- **Cannot connect to X:** check XWayland and import `DISPLAY` / `XAUTHORITY` from your graphical terminal as above.
- **Another tray manager exists:** use one legacy tray provider, not several.
- **Missing libraries after an upgrade:** check `ldd` and refresh the local binary.
- **Icon or click problems after restarting the extension:** restart the bridge; a launcher restart may also be needed.
- **Focus problems:** stop the bridge and report your desktop, extension and versions. This setup has only been verified on the system listed above.

## Undo

```sh
systemctl --user stop gnome-xembed-tray.service
rm -f "$HOME/.config/autostart/gnome-xembed-tray.desktop"
rm -f "$HOME/.config/systemd/user/gnome-xembed-tray.service"
systemctl --user daemon-reload
```

If you used the extraction method, you may also remove the local binary. Remove newly installed library packages only if no other software needs them. Wine may show its fallback tray window again after the bridge stops or the launcher restarts.

## Credits and license

- [KDE Plasma: xembed-sni-proxy source](https://invent.kde.org/plasma/plasma-workspace/-/tree/master/xembed-sni-proxy)
- [StatusNotifierItem specification](https://www.freedesktop.org/wiki/Specifications/StatusNotifierItem/)

The documentation and templates in this repository are MIT licensed. KDE's program retains its own upstream licenses; it is not included here.
