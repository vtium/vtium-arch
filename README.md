# VTium for Arch Linux

The Arch Linux packages of vtium, a terminal for the Linux console that draws through KMS/DRM. Each recipe is published on the AUR (the Arch User Repository) from its own directory here.

| Directory | AUR repository | Packages it builds | What it builds from |
|---|---|---|---|
| `vtium-git/` | `vtium-git` | `vtium-git`, `vtium-terminfo-git` | the tip of vtium's `main` branch on GitHub |
| `vtium/` | `vtium` | `vtium`, `vtium-terminfo` | a release: the tag `v0.1.0-alpha.1`, checked against the maintainer's signature |

The AUR keeps one recipe per git repository, named for the recipe's `pkgbase`, with its `PKGBUILD` and `.SRCINFO` at the root. Here each recipe is a directory, and `aur-publish` pushes one directory's history to its AUR repository as that repository's root (below). vtium keeps this repository as a submodule at `packaging/arch`, beside a repository for each other package manager.

The packaged vtium lags development on purpose. It is a real-world test of how vtium is built, deployed and updated, while a development install (`make vt-install` in a vtium checkout) tests the work in progress. The two never collide, so each can run on its own VT at the same time.

## What the packages install

Each recipe builds two packages: the console, and vtium's terminfo entry, which the console depends on. `vtium-git` provides and conflicts with `vtium`, and `vtium` conflicts with `vtium-git`; `vtium-terminfo-git` provides and conflicts with `vtium-terminfo`. So a machine holds the release or the git build, never both.

| File | Package | What it is |
|---|---|---|
| `/usr/bin/vtium` | `vtium`, `vtium-git` | the console, built with vtium's minspec profile, its default and smallest build |
| `/usr/bin/vtium-vitrify` | `vtium`, `vtium-git` | the glyph helper, which vtium starts to draw the glyphs of a frit (a font set file) |
| `/usr/lib/systemd/system/vtium@.service` | `vtium`, `vtium-git` | the unit that runs vtium on one VT in place of its getty |
| `/usr/lib/systemd/system/vtium-getty@.service` | `vtium`, `vtium-git` | the fallback that starts a getty on the VT when vtium fails |
| `/usr/share/doc/vtium/` | `vtium`, `vtium-git` | the example configuration and the example udev rules and hwdb fragment, as documentation |
| `/usr/share/terminfo/x/xterm-vtium` | `vtium-terminfo`, `vtium-terminfo-git` | vtium's terminfo entry, which sessions in vtium name in `TERM` |
| `/usr/share/licenses/<package>/LICENSE` | each | the license |

vtium starts its helper at a path fixed when it is built, so each recipe builds the console to start `/usr/bin/vtium-vitrify`. The two units are vtium's shipped `kmsconvt@` units under vtium's own names, made in `package()` the way vtium's `tools/vt/units.sh` makes them: the name and the fallback's name change, `Conflicts=` and `After=` lines naming `kmsconvt@%i.service` make starting vtium on a VT stop the kmscon console there first, and `DefaultInstance=` goes, so `systemctl enable vtium@.service` with no VT named links nothing. `Alias=autovt@.service` stays. vtium reads its configuration from `/etc/kmscon/kmscon.conf`, its head file from `/etc/vtium/heads.conf` and its frits from `/etc/vtium/fonts`; the packages install none of them, so a vtium with no configuration runs with its defaults.

The packages live beside the `kmscon` package and take none of its files: `kmscon` keeps `/usr/bin/kmscon`, `kmsconvt@.service`, `kmscon.service`, `/etc/kmscon/kmscon.conf.example` and the link `/etc/systemd/system/autovt@.service`, and `kmscon-terminfo` keeps the terminfo entry named `kmscon`. vtium names itself `TERM=xterm-vtium`, so its sessions read its own entry, and the kmscon console's sessions keep the one they were built for. A vtium run with `term=kmscon` needs `kmscon-terminfo`, which is an optional dependency.

### What waits on vtium's main

`vtium-git` builds whatever GitHub's `main` holds, so two things come with the vtium work that has not reached `main` yet:

- `main` has no `crates/vtium/install/xterm-vtium.ti` until the work that names vtium `TERM=xterm-vtium` reaches it. Until then `package_vtium-terminfo-git()` stops with a message naming the missing file, so the whole `vtium-git` build fails and installs nothing.
- The `Restart=on-failure` and `After=getty@%i.service` lines of vtium's unit, which this README's section on failures relies on, are in vtium's `integration` branch and reach `main` with it. Until then a `vtium@.service` built from `main` starts vtium only once, and its fallback starts the getty at the first failure.

## Building a package

Build a recipe with makepkg from its directory, and install its packages with pacman:

```sh
cd vtium-git
makepkg -si
```

makepkg clones vtium from GitHub, fetches its crates with `cargo fetch --locked`, and builds offline from there, with the host's own toolchain. It needs `cargo`, `git`, `ncurses` (for `tic`), `libxkbcommon` and `systemd-libs`, and the `base-devel` group that every makepkg build needs. `VTIUM_FEATURES` in makepkg's environment builds the console with other features, such as `VTIUM_FEATURES='images logind' makepkg -si`; unset, it builds the minspec console.

A build in a clean chroot checks that the dependencies are complete, since it starts from a system that holds only `base-devel` and what the recipe names. It needs root, so the maintainer runs it, from the recipe's directory, with `devtools` installed:

```sh
extra-x86_64-build
```

### The signature on a release

The `vtium` recipe builds from a git tag that the maintainer signs, and makepkg refuses the build unless the tag's signature is by the key in `validpgpkeys`: `33F9B54FD2A4B6A4110749B3BA464FC9847A832A`, Aaron Bockelie's. makepkg checks it against your own keyring, so import the key once, from this repository or from a keyserver:

```sh
gpg --import vtium/keys/pgp/33F9B54FD2A4B6A4110749B3BA464FC9847A832A.asc
gpg --recv-keys 33F9B54FD2A4B6A4110749B3BA464FC9847A832A
```

`vtium-git` is not checked against a signature. The tip of vtium's `main` is a merge commit that GitHub makes and signs with GitHub's own key, not the maintainer's.

### Versions

`vtium`'s `pkgver` is the tag without its `v` and without the hyphen before a pre-release: the tag `v0.1.0-alpha.1` is `0.1.0alpha.1`. pacman forbids a hyphen in a version, and the Arch package guidelines replace one with an underscore, but pacman sorts `0.1.0_alpha.1` after `0.1.0`, so an upgrade to the final release would look like a downgrade. `0.1.0alpha.1` sorts before `0.1.0`, as `vercmp` shows.

`vtium-git`'s `pkgver` is `r<commits>.<hash>` until vtium has a version tag. After that it is the last tag, mapped the same way, then `.r<commits since the tag>.g<hash>`, such as `0.1.0alpha.1.r5.g1234abc`. The console is built with that version as its revision, so the `Revision` line vtium logs at start names the build that `pacman -Q vtium-git` names.

### Building from a local checkout

Two environment variables point the `vtium-git` build at another repository and branch, such as a vtium checkout on the same machine. `VTIUM_GIT_URL` names the repository and `VTIUM_GIT_BRANCH` names the branch; unset, they are GitHub's repository and `main`.

```sh
VTIUM_GIT_URL=file:///home/me/src/vtium VTIUM_GIT_BRANCH=my-branch makepkg -f
```

makepkg builds the branch's last commit, so commit the changes first. It keeps its clone of the repository in `vtium-git/vtium`, and the next build fetches into it. When the variables name another repository than the clone came from, makepkg stops with "vtium is not a clone of" that repository; delete `vtium-git/vtium` and build again.

## Running vtium on a VT

Installing a package starts nothing. vtium runs on a VT only where you turn it on, one VT at a time, on a VT where no one is logged in. For VT 3, from another VT or over ssh:

```sh
sudo systemctl disable getty@tty3.service
sudo systemctl enable vtium@tty3.service
sudo systemctl start vtium@tty3.service
```

That runs vtium on VT 3 in place of its getty, and at every boot after. Starting it stops the getty, or the kmscon console, on VT 3 first. Enabling `vtium@tty3` also makes the link `/etc/systemd/system/autovt@tty3.service` to `vtium@.service`, so that logind, which starts `autovt@tty3` when you switch to a VT that nothing holds, starts vtium there and never another console beside it. Press Ctrl-Alt-F3 to see it.

To give VT 3 back, log out of it, then:

```sh
sudo systemctl disable --now vtium@tty3.service
sudo rm /etc/systemd/system/autovt@tty3.service
sudo systemctl enable --now getty@tty3.service
```

`systemctl disable` leaves the `autovt@tty3` link that `enable` made, so it goes by hand.

When vtium on a VT fails, its unit starts it again after 100 ms, up to five times in ten seconds, and each failure also starts `vtium-getty@tty3`, which starts a getty on the VT unless vtium's head file sets a closed policy, so a VT is never left at a dead screen. The restart needs the unit from vtium's `integration` branch (above).

## Upgrades

An upgrade never restarts a running vtium, because a restart ends every session on that VT. Each VT runs the new build from its next start: at a reboot, after a failure, or when you restart it on purpose. After each upgrade pacman says so and names the VTs whose vtium still runs the build it started with. To restart VT 3 now, log out there first, then, from another VT or over ssh:

```sh
sudo systemctl restart vtium@tty3.service
```

Removing a package leaves the links that enabling a VT made, which then point at nothing. pacman names each one after the removal, with the commands that remove it and give its VT its getty again; give each VT back before you remove the package to leave none.

## Publishing to the AUR

`aur-publish DIR` pushes the recipe in DIR to `ssh://aur@aur.archlinux.org/<pkgbase>.git`, under the SSH key of whoever runs it, so the maintainer runs it after a change here is reviewed. It splits DIR's history out with `git subtree split --prefix DIR`, so each push carries on from the last one, and pushes it to the AUR repository's `master` branch, the only branch the AUR takes. First it checks:

- that DIR is committed, and its `.SRCINFO` is what `makepkg --printsrcinfo` prints;
- that no checksum is a placeholder;
- that the steps `vtium-git/PKGBUILD` and `vtium/PKGBUILD` share, everything from the line `# Everything below is the same`, are the same in both, since each AUR repository must stand alone and so carries its own copy;
- that a release's tag exists on GitHub, and that it is signed by a key in `validpgpkeys`, checked against the key files in DIR's `keys/pgp/` alone, in a keyring of its own, with no configuration of the user's, so it never reaches the user's own keys or asks for a passphrase;
- that every commit it pushes holds `PKGBUILD` and `.SRCINFO`, as the AUR asks.

`--dry-run` runs the checks and prints the push without running it. `./test-aur-publish` runs each refusal against a scratch clone, a scratch source repository and scratch keys, and pushes nothing.

```sh
./aur-publish --dry-run vtium-git
./aur-publish vtium-git
```

Before you commit a change to a `PKGBUILD`, make its `.SRCINFO` again with the build variables unset, and set `vtium-git`'s `pkgver` back to the value for GitHub's main, since makepkg rewrites it after each build:

```sh
env -u VTIUM_GIT_URL -u VTIUM_GIT_BRANCH -u VTIUM_FEATURES makepkg --printsrcinfo > .SRCINFO
```

`vtium` waits for its tag: `aur-publish vtium` refuses until the maintainer has signed and pushed `v0.1.0-alpha.1`.

## License

The PKGBUILDs and the files in this repository are under the MIT license, as vtium is. Each recipe's directory carries the license, so each AUR repository does too.
