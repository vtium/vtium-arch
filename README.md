# VTium for Arch Linux

The Arch Linux packages of vtium, a terminal for the Linux console that draws through KMS/DRM.

This repository holds one package for now, `vtium-git`, which builds the tip of vtium's main branch on GitHub. A package named `vtium`, built from vtium's version tags, joins it once vtium has tags. vtium keeps this repository as a submodule at `packaging/arch`, beside a repository for each other package manager.

The packaged vtium lags development on purpose. It is a real-world test of how vtium is built, deployed and updated, while a development install (`make vt-install` in a vtium checkout) tests the work in progress. The two never collide, so each can run on its own VT at the same time.

## What the package installs

| File | What it is |
|---|---|
| `/usr/bin/vtium` | the console, built with vtium's minspec profile, its default and smallest build |
| `/usr/bin/vtium-vitrify` | the glyph helper, which vtium starts to draw the glyphs of a frit (a font set file) |
| `/usr/lib/systemd/system/vtium@.service` | the unit that runs vtium on one VT in place of its getty |
| `/usr/lib/systemd/system/vtium-getty@.service` | the fallback that starts a getty on the VT when vtium fails |
| `/usr/share/doc/vtium/` | the example configuration and the example udev rules and hwdb fragment, as documentation |
| `/usr/share/vtium/kmscon.ti` | the source of vtium's terminfo entry |
| `/usr/share/licenses/vtium-git/LICENSE` | the license |

The package's build runs `make install` in the vtium checkout, which stages exactly these files, so the vtium repository decides what an installed vtium holds. vtium reads its configuration from `/etc/kmscon/kmscon.conf`, its head file from `/etc/vtium/heads.conf` and its frits from `/etc/vtium/fonts`; the package installs none of them, so a vtium with no configuration runs with its defaults.

The package lives beside the `kmscon` package and takes none of its files: `kmscon` keeps `/usr/bin/kmscon`, `kmsconvt@.service`, `kmscon.service`, `/etc/kmscon/kmscon.conf.example` and the link `/etc/systemd/system/autovt@.service`. The compiled terminfo entry `kmscon`, which sessions in vtium name in `TERM`, comes from the `kmscon-terminfo` package, which this package depends on. vtium's own entry has more than that one, such as italics, dim, synchronized output and the SGR mouse. To use it for your own sessions, compile it into your home directory:

```sh
tic -x -o ~/.terminfo /usr/share/vtium/kmscon.ti
```

## Building it

Build the package with makepkg from the `vtium-git` directory, and install it with pacman:

```sh
cd vtium-git
makepkg -f
sudo pacman -U vtium-git-*.pkg.tar.zst
```

makepkg clones vtium from GitHub, fetches its crates with `cargo fetch --locked`, and builds offline from there. It needs `cargo` and `git`, and the `base-devel` group that every makepkg build needs.

### Building from a local checkout

Two environment variables point the build at another repository and branch, such as a vtium checkout on the same machine. `VTIUM_GIT_URL` names the repository and `VTIUM_GIT_BRANCH` names the branch; unset, they are GitHub's repository and `main`.

```sh
VTIUM_GIT_URL=file:///home/me/src/vtium VTIUM_GIT_BRANCH=my-branch makepkg -f
```

makepkg builds the branch's last commit, so commit the changes first. It keeps its clone of the repository in `vtium-git/vtium`, and the next build fetches into it. When the variables name another repository than the clone came from, makepkg stops with "vtium is not a clone of" that repository; delete `vtium-git/vtium` and build again.

makepkg rewrites `pkgver` in the PKGBUILD after each build. Before you commit a change to the PKGBUILD, set `pkgver` back to the value for GitHub's main and make `.SRCINFO` again with neither variable set:

```sh
env -u VTIUM_GIT_URL -u VTIUM_GIT_BRANCH makepkg --printsrcinfo > .SRCINFO
```

## Running vtium on a VT

Installing the package starts nothing. vtium runs on a VT only where you enable it, one VT at a time. For VT 3:

```sh
sudo systemctl enable --now vtium@tty3.service
```

That starts vtium on VT 3 in place of its getty, and at every boot after. Enabling `vtium@tty3` also makes the link `/etc/systemd/system/autovt@tty3.service` to `vtium@.service`, so that logind, which starts `autovt@tty3` when you switch to a VT that nothing holds, starts vtium there and never another console beside it. Press Ctrl-Alt-F3 to see it.

To take VT 3 back, log out of it, then:

```sh
sudo systemctl disable --now vtium@tty3.service
sudo rm /etc/systemd/system/autovt@tty3.service
```

`systemctl disable` leaves the `autovt@tty3` link that `enable` made, so it goes by hand. With it gone, logind starts the system's own console on VT 3 again.

When vtium on a VT fails, systemd starts it again after 100 ms, up to five times in ten seconds. Each failure also starts `vtium-getty@tty3`, which starts a getty on the VT, unless vtium's head file sets a closed policy, so a VT is never left at a dead screen.

## Upgrades

An upgrade never restarts a running vtium, because a restart hangs up every session on that VT. Each VT runs the new build from its next start: at a reboot, after a failure, or when you restart it on purpose. pacman says so after each upgrade. To restart VT 3 now, log out there first, then:

```sh
sudo systemctl restart vtium@tty3.service
```

Removing the package leaves the links that enabling a VT made, which then point at nothing. pacman names each one after the removal, with the command that removes it; disable each VT before you remove the package to leave none.

## License

The PKGBUILDs and the files in this repository are under the MIT license, as vtium is.
