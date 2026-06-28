# plasma-pager-custom

A local rebuild of the `plasma-desktop` package that adds **configurable label font, color and legibility effects** to the KDE Plasma **Pager** widget (and the **Activity Pager**).

## Why a full rebuild?

In Plasma 6 the Pager is a compiled C++/QML applet (`/usr/lib/qt6/plugins/plasma/applets/org.kde.plasma.pager.so`). Its `main.qml` is baked into the binary as an ahead-of-time QML cache, so there is no editable QML on disk — you cannot just copy the plasmoid and tweak it. The only way to change its UI is to patch the source and rebuild the package.

This repository carries that change as a small downstream patch over `plasma-desktop` plus a `PKGBUILD` (based on the official Arch package) that rebuilds it with the patch applied.

## What's in this repo

| File | Purpose |
|---|---|
| `PKGBUILD` | Copy of the official Arch `plasma-desktop` 6.7.1-1 recipe, wired to apply the patch in `prepare()` |
| `0001-pager-custom-label-font-and-color.patch` | The patch itself (3 files) |
| `.gitignore` | Excludes build artifacts (`src/`, `pkg/`, the downloaded tarball, `*.pkg.tar.zst`, logs) |
| `README.md` | This file |

## What the patch changes

It touches only the Pager applet:

- `applets/pager/main.xml` — eight new config keys:
  - font / color: `customFontEnabled`, `fontFamily`, `fontSize`, `customColorEnabled`, `textColor`;
  - label legibility: `labelEffect` (enum None/Outline/Shadow/Scrim), `effectColorAuto` (auto-contrast effect color), `effectColor`.
- `applets/pager/qml/configGeneral.qml` — a "Label appearance" section in the widget settings:
  - custom-font toggle;
  - **font family picker** implemented as a `Button` + popup with a **search field** (`Kirigami.SearchField`) and a list where **each family name is rendered in its own typeface** (live preview). It is a `Button` rather than a `ComboBox` because the `org.kde.desktop` style draws the ComboBox's current text with a native `StyleItem` (not `contentItem`) and couples its background to `popup.exit` — overriding both there breaks rendering;
  - font size (0 = automatic);
  - text color button;
  - legibility-effect dropdown, auto-effect-color toggle and effect color button.
- `applets/pager/qml/main.qml` — applies the settings in the label delegate (wrapped in an `Item` so a scrim can be drawn *behind* the text). Effects:
  - **Outline** — built-in `Text.style = Outline` + `styleColor` (no shader);
  - **Shadow** — `MultiEffect` (`QtQuick.Effects`) as a `layer.effect`, GPU drop shadow;
  - **Scrim** — a translucent `Rectangle` behind the text;
  - in Auto mode the effect color is chosen for contrast via `Kirigami.ColorUtils.brightnessForColor`.

**Default behaviour is unchanged**: with every toggle off and `labelEffect = None`, the labels render byte-for-byte like the stock widget (theme font and text color).

## Build & install

Requires `base-devel`. `makepkg` will install the build dependencies itself (needs `sudo`).

```bash
git clone https://github.com/BobBoba/plasma-pager-custom.git
cd plasma-pager-custom

# Build and install (-s installs makedepends, -i installs the package)
makepkg -si
```

Or build first, then install manually:

```bash
makepkg -s
sudo pacman -U plasma-desktop-6.7.1-1-x86_64.pkg.tar.zst
```

> **Integrity / no PGP keys needed.** The `PKGBUILD` downloads the official `plasma-desktop` tarball from download.kde.org and verifies it against a **pinned SHA-256** (the same hash as Arch's official package), so tampering or corruption is ruled out. The upstream PGP signature is intentionally not used, so the build does **not** require the KDE release keys in your keyring and works out of the box.

> ⚠️ This rebuilds the **whole** `plasma-desktop` package, not just one applet — the first build takes a few minutes and pulls in KDE dev dependencies. Subsequent rebuilds are faster (with ccache enabled).

## Applying the change

A full logout is **not** required — just restart the shell:

```bash
systemctl --user restart plasma-plasmashell.service
# fallback if it is not a systemd user service:
# kquitapp6 plasmashell; kstart plasmashell
```

Then: right-click the Pager widget → "Configure…" → **Label appearance** → enable "Use a custom font" / "Use a custom text color" / pick a "Legibility effect".

## After a plasma-desktop update

A stock `plasma-desktop` from the repositories will replace this package on `pacman -Syu` and revert the patch (expected — the versions match until the repo moves on). To re-apply:

1. Check the new upstream version and checksum in the official package:
   <https://gitlab.archlinux.org/archlinux/packaging/packages/plasma-desktop/-/blob/main/PKGBUILD>
2. Update `pkgver` / `pkgrel` / `sha256sums` in this `PKGBUILD` accordingly.
3. Confirm the patch still applies to the new version (upstream may have rewritten the Pager QML — adjust the patch if needed):
   ```bash
   makepkg -o                                # download + extract the new source
   cd src/plasma-desktop-*/ && patch -Np1 --dry-run < ../../0001-pager-custom-label-font-and-color.patch
   ```
4. Rebuild: `makepkg -si`.

To stop an accidental `pacman -Syu` from reverting your build between rebuilds, you can temporarily add `plasma-desktop` to `IgnorePkg` in `/etc/pacman.conf` (remember to remove it before an intentional update).

## Regenerating the patch (if you edit the QML further)

Sources extract into `src/`. A convenient loop with git:

```bash
makepkg -o
cd src/plasma-desktop-*/
git init -q && git add -A && git commit -qm baseline
# ...edit files under applets/pager/...
git diff > ../../0001-pager-custom-label-font-and-color.patch
```

## Notes for an upstream KDE merge request

The change is structured as a proper feature (config keys + GUI + application), not a hardcode, so it can serve as the basis for a merge request to `plasma-desktop`.

Honest caveat: KDE has historically been reluctant to add per-widget font/color overrides (consistency through the theme / color scheme, per the KDE HIG). "Font size" is more likely to be accepted than an arbitrary "color". Note also that **KDE does not accept merge requests via GitHub** — contributions go through KDE's own GitLab at <https://invent.kde.org/plasma/plasma-desktop> (GitHub `KDE/*` repos are read-only mirrors), must target the `master` branch, and require a `Signed-off-by` (DCO) line. It is worth discussing the feature first (bugs.kde.org or the `#plasma:kde.org` Matrix channel) before investing in the MR.
