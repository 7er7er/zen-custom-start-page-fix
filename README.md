# Setting a Custom Start Page in Zen Browser (Without Breaking External Links)

How to make Zen Browser on Linux open a fixed start page every time it's launched from an icon/app launcher, without that page also getting pulled into links handed to Zen by other applications via `xdg-open`.

## The fix

**1. Set the start page with a launcher wrapper script, not `about:config`.**

The wrapper passes the start page as a command-line argument whenever Zen is launched with no arguments (icon/app launcher click). That's what makes it load every time. (`browser.startup.homepage`/`page` don't — on Zen 1.21.8b they're ignored on a bare launch, and the previous session gets resumed instead.)

Save as `~/.local/bin/zen-launcher` (shipped here as [`zen-launcher`](./zen-launcher) — edit the two variables at the top first):

```bash
#!/bin/bash
START_PAGE="https://your-desired-website.example"
ZEN_BIN="/opt/zen-browser-bin/zen-bin"

if [ "$#" -eq 0 ]; then
  exec "$ZEN_BIN" "$START_PAGE"
else
  exec "$ZEN_BIN" "$@"
fi
```

```bash
chmod +x ~/.local/bin/zen-launcher
```

Then point Zen's `.desktop` launcher at the wrapper instead of the browser binary. If you don't already have a personal copy, copy the packaged one first — **don't edit the file under `/usr/share` directly**, it's owned by the package manager and gets overwritten on the next update:

```bash
mkdir -p ~/.local/share/applications
cp /usr/share/applications/zen.desktop ~/.local/share/applications/zen.desktop
```

In `~/.local/share/applications/zen.desktop`, change the `Exec=` line of the main `[Desktop Entry]` (leave the `Exec=` lines under `[Desktop Action ...]` sections alone):

```ini
Exec=/home/YOUR-USERNAME/.local/bin/zen-launcher %u
```

```bash
update-desktop-database ~/.local/share/applications
```

Result: no argument (icon/launcher click) → wrapper opens `START_PAGE`. Argument present (a link handed off by another app) → wrapper forwards it untouched.

**2. Set this in `about:config` so external links open as a tab, not a new window:**

```
browser.link.open_newwindow.override.external = 3
```

(`3` = always open in a new tab in the most recently active window. `2` = always a new window; `1` = replace the current tab; `-1` is the ambiguous default that falls back to `browser.link.open_newwindow`.)

Shipped here as [`user.js`](./user.js) — copy it into your Zen profile folder and restart Zen. Find your profile folder via `about:support` → "Profile Folder":

- Linux: `~/.config/zen/<profile-name>/`
- macOS: `~/Library/Application Support/zen/<profile-name>/`
- Windows: `%APPDATA%\zen\<profile-name>\`

## Notes

- `Exec=zen-bin https://start-page %u` (URL hardcoded directly in `Exec=`) breaks external links: `%u` is substituted with the clicked link, so Zen gets two URLs on the command line and opens both as separate tabs.
- Zen's not honoring `browser.startup.page`/`homepage` on a bare launch is unconfirmed/undocumented — an observation on 1.21.8b, not a cited fact. Possibly related to Workspaces persisting tabs across restarts.
- Related upstream reports on external links opening a new window: [zen-browser/desktop#14914](https://github.com/zen-browser/desktop/issues/14914), [#10409](https://github.com/zen-browser/desktop/issues/10409), [#1009](https://github.com/zen-browser/desktop/issues/1009).

## Files

- [`zen-launcher`](./zen-launcher) — the wrapper script (step 1). Edit `START_PAGE` and `ZEN_BIN` first.
- [`user.js`](./user.js) — the `about:config` fix (step 2), plus `browser.startup.homepage`/`page` for completeness (not sufficient on its own — that's what the wrapper is for).
