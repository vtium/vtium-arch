# VTium for Arch Linux

The Arch Linux packages of vtium, a terminal for the Linux console that draws through KMS/DRM. Each package is published on the AUR (the Arch User Repository) from its own directory here.

| Directory | AUR package | What it builds |
|---|---|---|
| `vtium-git/` | `vtium-git` | the tip of vtium's `main` branch on GitHub |
| `vtium/` | `vtium` | a release: the tag `v0.1.0-alpha.1`, checked against the maintainer's signature |

The AUR keeps one package per git repository, with its `PKGBUILD` and `.SRCINFO` at the root. Here each package is a directory, and `aur-publish` pushes one directory's history to its AUR repository as that repository's root (below). vtium keeps this repository as a submodule at `packaging/arch`, beside a repository for each other package manager.

The packaged vtium lags development on purpose. It is a real-world test of how vtium is built, deployed and updated, while a development install (`make vt-install` in a vtium checkout) tests the work in progress. The two never collide, so each can run on its own VT at the same time.

## What a package installs

Both packages install the same files and cannot be installed together: `vtium-git` provides and conflicts with `vtium`.

| File | What it is |
|---|---|
| `/usr/bin/vtium` | the console, built with vtium's minspec profile, its default and smallest build |
| `/usr/bin/vtium-vitrify` | the glyph helper, which vtium starts to draw the glyphs of a frit (a font set file) |
| `/usr/lib/systemd/system/vtium@.service` | the unit that runs vtium on one VT in place of its getty |
| `/usr/lib/systemd/system/vtium-getty@.service` | the fallback that starts a getty on the VT when vtium fails |
| `/usr/share/doc/vtium/` | the example configuration and the example udev rules and hwdb fragment, as documentation |
| `/usr/share/vtium/kmscon.ti` | the source of vtium's terminfo entry |
| `/usr/share/licenses/<package>/LICENSE` | the license |

vtium starts its helper at a path fixed when it is built, so each package builds the console to start `/usr/bin/vtium-vitrify`. The two units are vtium's shipped `kmsconvt@` units under vtium's own names, made in `package()` the way vtium's `make vt-install` makes them: only the name and the fallback's name change, and `Alias=autovt@.service` stays. vtium reads its configuration from `/etc/kmscon/kmscon.conf`, its head file from `/etc/vtium/heads.conf` and its frits from `/etc/vtium/fonts`; the packages install none of them, so a vtium with no configuration runs with its defaults.

The packages live beside the `kmscon` package and take none of its files: `kmscon` keeps `/usr/bin/kmscon`, `kmsconvt@.service`, `kmscon.service`, `/etc/kmscon/kmscon.conf.example` and the link `/etc/systemd/system/autovt@.service`. The compiled terminfo entry `kmscon`, which sessions in vtium name in `TERM`, comes from the `kmscon-terminfo` package, which both packages depend on. vtium's own entry has more than that one, such as italics, dim, synchronized output and the SGR mouse. To use it for your own sessions, compile it into your home directory:

```sh
tic -x -o ~/.terminfo /usr/share/vtium/kmscon.ti
```

## Building a package

Build a package with makepkg from its directory, and install it with pacman:

```sh
cd vtium-git
makepkg -si
```

makepkg clones vtium from GitHub, fetches its crates with `cargo fetch --locked`, and builds offline from there. It needs `cargo` and `git`, and the `base-devel` group that every makepkg build needs.

A build in a clean chroot checks that the dependencies are complete, since it starts from a system that holds only `base-devel` and what the package names. It needs root, so the maintainer runs it, from the package's directory, with `devtools` installed:

```sh
extra-x86_64-build
```

### The signature on a release

The `vtium` package builds from a git tag that the maintainer signs, and makepkg refuses the build unless the tag's signature is by the key in `validpgpkeys`: `33F9B54FD2A4B6A4110749B3BA464FC9847A832A`, Aaron Bockelie's. makepkg checks it against your own keyring, so import the key once, from this repository or from a keyserver:

```sh
gpg --import vtium/keys/pgp/33F9B54FD2A4B6A4110749B3BA464FC9847A832A.asc
gpg --recv-keys 33F9B54FD2A4B6A4110749B3BA464FC9847A832A
```

`vtium-git` is not checked against a signature. The tip of vtium's `main` is a merge commit that GitHub makes and signs with GitHub's own key, not the maintainer's.

### Versions

`vtium`'s `pkgver` is the tag without its `v` and without the hyphen before a pre-release: the tag `v0.1.0-alpha.1` is `0.1.0alpha.1`. pacman forbids a hyphen in a version, and the Arch package guidelines replace one with an underscore, but pacman sorts `0.1.0_alpha.1` after `0.1.0`, so an upgrade to the final release would look like a downgrade. `0.1.0alpha.1` sorts before `0.1.0`, as `vercmp` shows.

`vtium-git`'s `pkgver` is `r<commits>.<hash>` until vtium has a version tag. After that it is the last tag, mapped the same way, then `.r<commits since the tag>.g<hash>`, such as `0.1.0alpha.1.r5.g1234abc`.

### Building from a local checkout

Two environment variables point the `vtium-git` build at another repository and branch, such as a vtium checkout on the same machine. `VTIUM_GIT_URL` names the repository and `VTIUM_GIT_BRANCH` names the branch; unset, they are GitHub's repository and `main`.

```sh
VTIUM_GIT_URL=file:///home/me/src/vtium VTIUM_GIT_BRANCH=my-branch makepkg -f
```

makepkg builds the branch's last commit, so commit the changes first. It keeps its clone of the repository in `vtium-git/vtium`, and the next build fetches into it. When the variables name another repository than the clone came from, makepkg stops with "vtium is not a clone of" that repository; delete `vtium-git/vtium` and build again.

## Running vtium on a VT

Installing a package starts nothing. vtium runs on a VT only where you enable it, one VT at a time. For VT 3:

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

Removing a package leaves the links that enabling a VT made, which then point at nothing. pacman names each one after the removal, with the command that removes it; disable each VT before you remove the package to leave none.

## Publishing to the AUR

`aur-publish DIR` pushes the package in DIR to `ssh://aur@aur.archlinux.org/<pkgname>.git`, under the SSH key of whoever runs it, so the maintainer runs it after a change here is reviewed. It splits DIR's history out with `git subtree split --prefix DIR`, so each push carries on from the last one, and pushes it to the AUR repository's `master` branch, the only branch the AUR takes. First it checks that DIR is committed, that its `.SRCINFO` is what `makepkg --printsrcinfo` prints, that no checksum is a placeholder, that a release's tag exists on GitHub, and that every commit it pushes holds `PKGBUILD` and `.SRCINFO`, as the AUR asks. `--dry-run` runs the checks and prints the push without running it.

```sh
./aur-publish --dry-run vtium-git
./aur-publish vtium-git
```

Before you commit a change to a `PKGBUILD`, make its `.SRCINFO` again with the local-checkout variables unset, and set `vtium-git`'s `pkgver` back to the value for GitHub's main, since makepkg rewrites it after each build:

```sh
env -u VTIUM_GIT_URL -u VTIUM_GIT_BRANCH makepkg --printsrcinfo > .SRCINFO
```

`vtium` waits for its tag: `aur-publish vtium` refuses until `v0.1.0-alpha.1` is on GitHub. The two packages' `build()` and `package()` are kept the same by hand, since each AUR repository must stand alone.

## License

The PKGBUILDs and the files in this repository are under the MIT license, as vtium is. Each package directory carries the license, so each AUR repository does too.
