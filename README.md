# VTium for Arch Linux

The Arch Linux packages of vtium, a terminal for the Linux console that draws through KMS/DRM. Each recipe is published on the AUR (the Arch User Repository) from its own directory here.

| Directory | AUR repository | Packages it builds | What it builds from |
|---|---|---|---|
| `vtium/` | `vtium` | `vtium`, the console for an ordinary desktop or laptop | a release: the tag `v0.1.0-alpha.5`, checked against the maintainer's signature |
| `vtium-git/` | `vtium-git` | `vtium-git`, the same console, and `vtium-terminfo-git` | the tip of vtium's `main` branch on GitHub |
| `vtium-minspec/` | `vtium-minspec` | `vtium-minspec`, vtium's smallest console, with no pictures and no GPU path | the same release tag |
| `vtium-minspec-gpu/` | `vtium-minspec-gpu` | `vtium-minspec-gpu`, the smallest console with the GPU path | the same release tag |
| `vtium-terminfo/` | `vtium-terminfo` | `vtium-terminfo`, vtium's terminfo entry | the same release tag |
| `vtium-fonts/` | `vtium-fonts` | `vtium-fonts`, which installs no font and depends on the fonts the frits name | this repository alone |

The AUR keeps one recipe per git repository, named for the recipe's `pkgbase`, with its `PKGBUILD` and `.SRCINFO` at the root. Here each recipe is a directory, and `aur-publish` pushes one directory's history to its AUR repository as that repository's root (below). vtium keeps this repository as a submodule at `packaging/arch`, beside a repository for each other package manager.

The packaged vtium lags development on purpose. It is a real-world test of how vtium is built, deployed and updated, while a development install (`make vt-install` in a vtium checkout) tests the work in progress. The two never collide, so each can run on its own VT at the same time.

## What the packages install

There are four console packages: `vtium`, `vtium-git`, `vtium-minspec` and `vtium-minspec-gpu`. Each installs `/usr/bin/vtium`, its helpers and its units, so a machine holds one of them at a time. Every console package but `vtium` provides `vtium` at its own version and conflicts with `vtium`, and `vtium` names each of the others in its conflicts, so installing one console package asks to remove the one installed before, and a package that depends on `vtium` is met by any of them. `vtium-minspec-git` and `vtium-minspec-gpu-git` are named in `vtium`'s conflicts for when they are published. Each console package depends on vtium's terminfo entry: the packages built from the release tag on `vtium-terminfo`, and `vtium-git` on `vtium-terminfo-git`, which provides and conflicts with `vtium-terminfo`. `vtium-terminfo` also installs alone, on a machine that reaches a vtium console over ssh and runs none itself.

| File | Package | What it is |
|---|---|---|
| `/usr/bin/vtium` | every console package | the console (below), and its subcommand `vtium console`, which gives VTs to vtium and takes them back |
| `/usr/bin/vtium-config` | every console package | the font and settings tool that `vtium config` runs, which is also the glyph helper that vtium starts to draw the glyphs of a frit (a font set file) |
| `/usr/bin/vtium-image` | `vtium`, `vtium-git` | the image helper, which vtium starts to decode the pictures that programs send, in a process of its own |
| `/usr/lib/sysusers.d/vtium-image.conf` | `vtium`, `vtium-git` | the file from which pacman's sysusers hook creates the image helper's account, `vtium-image`, when the package is installed |
| `/usr/bin/vtium-gallery` | `vtium`, `vtium-git` | a bash script that shows PNG pictures in vtium as a grid of thumbnails, each named under it |
| `/usr/lib/systemd/system/vtium@.service` | every console package | the unit that runs vtium on one VT in place of its getty |
| `/usr/lib/systemd/system/vtium-getty@.service` | every console package | the fallback that starts a getty on the VT when vtium fails |
| `/etc/vtium/vtium.conf.example` | every console package | the example configuration, beside `/etc/vtium/vtium.conf`, the file vtium reads |
| `/etc/vtium/fonts/desktop.vtfrit`, `terminus.vtfrit` | `vtium`, `vtium-git` | two frits (below) |
| `/etc/vtium/fonts/minspec-16.vtfrit`, `minspec-8.vtfrit` | every console package | two frits (below) |
| `/usr/share/doc/vtium/examples/` | every console package | the example udev rules and hwdb fragment, and `heads.conf`, the head file's commented template, as documentation |
| `/usr/share/terminfo/x/xterm-vtium` | `vtium-terminfo`, `vtium-terminfo-git` | vtium's terminfo entry, which sessions in vtium name in `TERM` |
| `/usr/share/licenses/<package>/LICENSE` | each | the license, and the only file `vtium-fonts` installs |

Each recipe builds and stages these files with vtium's own make targets, so the knowledge of which file goes where lives in vtium's tree: `make install DESTDIR="$pkgdir" PREFIX=/usr` stages the console's package, and `make install-terminfo DESTDIR="$pkgdir" PREFIX=/usr` stages the terminfo package. vtium starts its helpers at paths fixed when it is built, and `PREFIX=/usr` builds the console to start its glyph helper at `/usr/bin/vtium-config` and its image helper at `/usr/bin/vtium-image`. vtium's `tools/vt/units.sh` makes the two units from vtium's shipped `vtium@.service` and `vtium-getty@.service`: `Conflicts=` and `After=` lines naming `kmsconvt@%i.service` make starting vtium on a VT stop the kmscon console there first, and `DefaultInstance=tty1` gives way to `DefaultInstance=name-a-VT/vtium@ttyN`, which names no VT (below). `Alias=autovt@.service` stays. vtium reads its configuration from `/etc/vtium/vtium.conf`, its head file from `/etc/vtium/heads.conf` and its frits from `/etc/vtium/fonts`. The packages install neither `vtium.conf` nor `heads.conf`, so a vtium with no configuration runs with its defaults. To start a configuration of your own, copy the example beside it, with `sudo cp /etc/vtium/vtium.conf.example /etc/vtium/vtium.conf`, and to give each display a terminal of its own, copy `/usr/share/doc/vtium/examples/heads.conf` to `/etc/vtium/heads.conf` and edit it. vtium reads nothing under `/etc/kmscon`, so a kmscon configuration never reaches vtium.

The packages live beside the `kmscon` package and take none of its files: `kmscon` keeps `/usr/bin/kmscon`, `kmsconvt@.service`, `kmscon.service`, `/etc/kmscon/kmscon.conf.example` and the link `/etc/systemd/system/autovt@.service`, and `kmscon-terminfo` keeps the terminfo entry named `kmscon`. vtium names itself `TERM=xterm-vtium`, so its sessions read its own entry, and the kmscon console's sessions keep the one they were built for. A vtium run with `term=kmscon` needs `kmscon-terminfo`, which is an optional dependency.

### The console's build

Every console package builds the console with cargo's minspec profile, and the packages differ in the console's features. `vtium` and `vtium-git` build it with the features `images logind gpu`, for an ordinary desktop or laptop. `images` draws pictures, sent through sixel or the kitty graphics protocol, and it is why the packages ship the image helper and `vtium-gallery`. `logind` lets the power keys suspend the machine through logind. `gpu` builds in the GPU path, which draws through the graphics card. The console keeps three buffers for each display.

The GPU path is off by default in vtium 0.1.0-alpha.5, so the console draws every frame on the CPU. To turn it on, add a line that reads `hwaccel` to `/etc/vtium/vtium.conf`, and restart vtium on each VT, as [Upgrades](#upgrades) shows. A display whose GPU set-up fails is drawn on the CPU. `vtium config config hwaccel` prints the value `/etc/vtium/vtium.conf` gives it, or the build's default. A later release turns the GPU path on by default.

`vtium` and `vtium-git` depend on `libglvnd` and `mesa`. The GPU path loads `libEGL.so.1`, which `libglvnd` ships, and `libgbm.so.1`, which `mesa` ships, when it makes a GL context. The console links neither library, so it would start without them and draw on the CPU, but these two packages require both so that turning `hwaccel` on works. EGL also needs a driver for the card, which `mesa` gives for most cards and NVIDIA's own driver gives for its cards. `logind` reaches logind through sd-bus in `systemd-libs`, which every console package already depends on for udev. The image helper's decoders are written in Rust and link no library beyond the C runtime. `bash` runs `vtium-gallery`. `kbd`, whose console fonts vtium draws when no frit is set, is an optional dependency of every console package.

### The smallest builds

`vtium-minspec` and `vtium-minspec-gpu` are for a single-board computer, a server's console or a kiosk, where a picture decoder or logind is weight that nothing uses. `vtium-minspec` builds the console with no feature at all: the build that vtium's tier 3 tests judge in a virtual machine, so their figures for size and memory speak for it. `vtium-minspec-gpu` adds the feature `gpu` alone, for a small machine whose GPU is worth using. Neither draws pictures, so neither ships the image helper or `vtium-gallery`, and both keep two buffers for each display. `vtium --version` names the features each was built with: `features none` and `features gpu`.

`vtium-minspec-gpu` leaves `hwaccel` off, and a line `hwaccel` in `/etc/vtium/vtium.conf` turns its GPU path on, as for `vtium`. It lists `libglvnd` and `mesa` as optional dependencies, so pacman installs neither unless you ask, and its console draws on the CPU on a machine without them. Install both to turn `hwaccel` on. `vtium-minspec` has no GPU path and needs neither.

### The frits

A frit (a font set file) names the fonts the glyph helper draws from. The console packages install their frits in `/etc/vtium/fonts`. `vtium` and `vtium-git` install four:

- `desktop`, which draws JetBrains Mono with the symbols of Nerd Fonts and Noto Color Emoji over vtium's built-in unifont, at 18 points;
- `terminus`, which draws the Terminus console font at 8x16 cells over the built-in unifont;
- `minspec-16` and `minspec-8`, which add the symbols of Nerd Fonts and color emoji over vtium's built-in fonts.

`vtium-minspec` and `vtium-minspec-gpu` install `minspec-16` and `minspec-8`. A frit changes nothing until you choose it, with `vtium config` or with a line such as `font-name=desktop` in `/etc/vtium/vtium.conf`.

The packages install no font file. The glyph helper finds a frit's fonts among the fonts installed on the machine, under `/usr/share/fonts` and, for a console font, under `/usr/share/kbd/consolefonts`, and leaves out a font the machine lacks. `vtium-fonts` installs the fonts these frits name, through four of Arch's packages: `ttf-jetbrains-mono`, `ttf-nerd-fonts-symbols-mono`, `noto-fonts-emoji` and `terminus-font`. It installs no file of its own beyond its license. Every console package lists it as an optional dependency, so the fonts stay optional: install `vtium-fonts`, or only the font packages you want.

The `terminus` frit names its font by the file name that Arch's `terminus-font` gives it, `ter-v16n`, so this repository carries the frit, in `vtium/` and `vtium-git/`, rather than vtium's own tree.

pacman keeps the edits you make to a frit across an upgrade. When the package's copy of a frit changes too, pacman installs it beside yours with the suffix `.pacnew`.

## Building a package

Build a recipe with makepkg from its directory, and install its packages with pacman. A console package built from the release tag depends on `vtium-terminfo`, which makepkg cannot fetch from the AUR, so build and install `vtium-terminfo` first, or let an AUR helper such as `yay` do both:

```sh
cd vtium-terminfo && makepkg -si && cd ..
cd vtium-minspec && makepkg -si
```

`vtium-git` builds `vtium-terminfo-git` beside its console, so one run installs both:

```sh
cd vtium-git
makepkg -si
```

makepkg clones vtium from GitHub and, for a console, fetches its crates with `cargo fetch --locked`. From there it builds offline, with the host's own toolchain, through vtium's Makefile: `build()` runs `make install-build`, and each package's `package()` runs `make install` or `make install-terminfo`, which find that build and build nothing again. A console needs `cargo`, `git`, `libxkbcommon` and `systemd-libs`, and the terminfo entry needs `git` and `ncurses` (for `tic`), besides the `base-devel` group that every makepkg build needs, which holds `make`, `pkgconf` and `binutils`. `vtium-fonts` builds nothing.

Each console recipe names its features, which it builds while `VTIUM_FEATURES` is unset: `images logind gpu` for `vtium` and `vtium-git`, none for `vtium-minspec`, and `gpu` for `vtium-minspec-gpu`. `VTIUM_FEATURES` in makepkg's environment names other features, such as `VTIUM_FEATURES='images logind' makepkg -si`, and the recipe hands it to make as `FEATURES`. Set but empty, as in `VTIUM_FEATURES= makepkg -si`, it names no feature, and the recipe builds the minspec console, vtium's smallest build. Whatever `VTIUM_FEATURES` names, the package keeps the dependencies, the frits and the two build settings of its recipe's default build: `hwaccel` off in every package, three buffers for each display in `vtium` and `vtium-git`, and two in `vtium-minspec` and `vtium-minspec-gpu`.

A build in a clean chroot checks that the dependencies are complete, since it starts from a system that holds only `base-devel` and what the recipe names. It needs root, so the maintainer runs it, from the recipe's directory, with `devtools` installed:

```sh
extra-x86_64-build
```

### The signature on a release

The `vtium`, `vtium-minspec`, `vtium-minspec-gpu` and `vtium-terminfo` recipes build from a git tag that the maintainer signs, and makepkg refuses the build unless the tag's signature is by the key in `validpgpkeys`: `33F9B54FD2A4B6A4110749B3BA464FC9847A832A`, Aaron Bockelie's. Each of their directories carries the key under `keys/pgp/`. makepkg checks it against your own keyring, so import the key once, from this repository or from a keyserver:

```sh
gpg --import vtium/keys/pgp/33F9B54FD2A4B6A4110749B3BA464FC9847A832A.asc
gpg --recv-keys 33F9B54FD2A4B6A4110749B3BA464FC9847A832A
```

`vtium-git` is not checked against a signature. The tip of vtium's `main` is a merge commit that GitHub makes and signs with GitHub's own key, not the maintainer's.

### Versions

The `pkgver` of each recipe built from the release tag is the tag without its `v` and without the hyphen before a pre-release: the tag `v0.1.0-alpha.1` is `0.1.0alpha.1`. pacman forbids a hyphen in a version, and the Arch package guidelines replace one with an underscore, but pacman sorts `0.1.0_alpha.1` after `0.1.0`, so an upgrade to the final release would look like a downgrade. `0.1.0alpha.1` sorts before `0.1.0`, as `vercmp` shows.

`vtium-git`'s `pkgver` is `r<commits>.<hash>` until vtium has a version tag. After that it is the last tag, mapped the same way, then `.r<commits since the tag>.g<hash>`, such as `0.1.0alpha.1.r5.g1234abc`. The console is built with that version as its revision, so the `Revision` line vtium logs at start names the build that `pacman -Q vtium-git` names.

### Building from a local checkout

Two environment variables point the `vtium-git` build at another repository and branch, such as a vtium checkout on the same machine. `VTIUM_GIT_URL` names the repository and `VTIUM_GIT_BRANCH` names the branch; unset, they are GitHub's repository and `main`.

```sh
VTIUM_GIT_URL=file:///home/me/src/vtium VTIUM_GIT_BRANCH=my-branch makepkg -f
```

makepkg builds the branch's last commit, so commit the changes first. It keeps its clone of the repository in `vtium-git/vtium`, and the next build fetches into it. When the variables name another repository than the clone came from, makepkg stops with "vtium is not a clone of" that repository; delete `vtium-git/vtium` and build again.

## Running vtium on a VT

Installing a package turns vtium on nowhere and starts nothing, the same as Arch's `kmscon` package, which has no install script. You give vtium a VT, or every VT, with `vtium console`, a subcommand of `/usr/bin/vtium`, and the same subcommand takes them back. `enable` and `disable` need root, and `status` runs as any user:

```sh
vtium console status               # for each VT: what systemd starts at boot, what logind starts when you switch to it, what runs now
sudo vtium console enable 3        # VT 3 is vtium's from its next start
sudo vtium console disable 3       # VT 3 goes back to what it had before
sudo vtium console enable --all    # every VT that nothing else holds is vtium's
sudo vtium console disable --all   # every VT goes back to what it had before
```

`vtium console` starts and stops nothing, because starting or stopping a console on a VT ends every session there, the one you type in included. It changes which console systemd and logind start on a VT the next time, and it prints the command that makes the change now, such as `sudo systemctl start vtium@tty3.service`. Run that command after you log out of VT 3, from another VT or over ssh, and press Ctrl-Alt-F3 to see vtium. Before `vtium console enable` changes anything, it records what each VT had in `/var/lib/vtium/console/`, and `vtium console disable` puts back exactly that. `vtium console enable N` refuses a VT that kmscon, a display manager or a graphical session holds, changes nothing, and says what holds the VT and how to free it. `vtium console enable 6` also refuses while someone is logged in on VT 6 through anything but vtium, and warns that the next switch stops a getty there if one waits at its login prompt. `vtium console enable --all` passes such a VT by, leaves it as it is, and names it, with two exceptions, where it refuses and says what frees the VT. The first is VT 6, which logind reserves and starts a console on at every switch. While VT 6 has no link of its own, `enable --all` refuses when someone is logged in there through anything but vtium, or when kmscon or vtium's development install (`make vt-install`) is enabled or running there. It also refuses while a display manager or a graphical session holds VT 6, whatever links VT 6 has. An idle getty on VT 6 is allowed, and `enable --all` warns that the next switch to VT 6 stops it. The second is a VT whose `vtium@ttyN.service` is masked while logind would still start vtium there through the template link.

`systemctl enable vtium@.service`, with no VT named, fails and links nothing. Without that refusal it would do harm: `systemctl` links each `Alias=` before it looks at `WantedBy=`, so on a machine with no `/etc/systemd/system/autovt@.service` yet it would link that name to `vtium@.service`, and logind would start vtium on every VT it spawns. The unit's `DefaultInstance=name-a-VT/vtium@ttyN` names no VT, so `systemctl` refuses it before it links anything; it ignores `DefaultInstance=` when you name a VT. The refusal shows elsewhere too: `systemctl list-unit-files` lists `vtium@.service` as `bad`, and `systemctl is-enabled vtium@.service` fails, while `systemctl is-enabled vtium@tty3.service` answers as usual.

When vtium on a VT fails, its unit starts it again after 100 ms, up to five times in ten seconds, and each failure also starts `vtium-getty@tty3`, which starts a getty on the VT unless vtium's head file sets a closed policy, so a VT is never left at a dead screen.

### Sharing VTs with kmscon

When you switch to a VT where no console runs, logind starts `autovt@ttyN.service` there, and that is the only name it starts. A VT can have a link of its own under that name, `/etc/systemd/system/autovt@ttyN.service`, and otherwise systemd follows the template link, `/etc/systemd/system/autovt@.service`; a VT's own link wins over the template link (systemd.unit(5)). Only one console holds the template link, and it gets every VT that has no link of its own. A VT whose getty systemd starts at boot is not free, so logind starts nothing there, except on VT 6: logind reserves that VT (`ReserveVT=6`, logind.conf(5)) and starts `autovt@tty6.service` on it at every switch.

kmscon holds the template link when `/etc/systemd/system/autovt@.service` links to `kmsconvt@.service`. Then `sudo vtium console enable 3` gives vtium VT 3. It links VT 3's own `autovt@tty3.service` to vtium, and kmscon keeps every other VT.

When vtium holds the template link, after `sudo vtium console enable --all`, give kmscon VT 3 with a link of VT 3's own:

```sh
sudo systemctl disable getty@tty3.service    # only where VT 3's getty is enabled at boot
sudo ln -s /usr/lib/systemd/system/kmsconvt@.service /etc/systemd/system/autovt@tty3.service
sudo systemctl daemon-reload
```

The link makes `autovt@tty3.service` name kmscon, so logind starts kmscon on VT 3 when you switch there, and every other VT keeps vtium through the template link. `daemon-reload` makes systemd read the new link. Nothing starts kmscon at boot, and nothing starts it now: logind starts it the next time you switch to VT 3 while no console runs there. A getty or a vtium that still runs on VT 3 keeps it until it stops, and `sudo systemctl stop getty@tty3.service` or `sudo systemctl stop vtium@tty3.service` stops it, which ends every session there. The first command frees VT 3 from a getty that systemd starts there at boot; `vtium console enable --all` has already disabled those on VTs 2 to 6.

To give VT 3 back, log out there, then:

```sh
sudo systemctl stop kmsconvt@tty3.service    # ends every session on VT 3
sudo rm /etc/systemd/system/autovt@tty3.service
sudo systemctl daemon-reload
sudo systemctl add-wants getty.target getty@tty3.service    # only if you disabled VT 3's getty above
```

With kmscon stopped and the link gone, logind starts vtium on VT 3 through the template link the next time you switch there. The last command enables VT 3's getty at boot again. It uses `add-wants` rather than `systemctl enable getty@tty3.service`, because `enable` also links `autovt@tty3.service` to the getty, and `vtium console enable 3` refuses a VT whose own link names another console.

`sudo systemctl enable --now kmsconvt@tty3.service`, run after the same `disable getty@tty3.service`, also gives kmscon VT 3, through the same link, and it also adds kmscon to the consoles systemd starts at boot and starts it now. Without the getty's disable, systemd's boot list would hold both kmscon and the getty on VT 3, which conflict. This guide prefers the link alone because two consoles that open the display card at boot compete for it, which is why `vtium console enable --all` starts vtium at boot on VT 1 alone.

To undo `enable --now`, log out of VT 3, then:

```sh
sudo systemctl disable --now kmsconvt@tty3.service    # stops kmscon and removes its boot entry
sudo rm /etc/systemd/system/autovt@tty3.service      # disable leaves this link
sudo systemctl daemon-reload
sudo systemctl add-wants getty.target getty@tty3.service    # only if you disabled VT 3's getty
```

While vtium holds the template link, don't run kmscon's own `systemctl enable kmsconvt@`. It fails, because `autovt@.service` already links to `vtium@.service`, and it still enables kmscon at boot on VT 1, through the `DefaultInstance=tty1` of kmscon's unit. `sudo systemctl disable kmsconvt@tty1.service` undoes that.

### Moving from kmscon to vtium

To give every VT to vtium on a machine where kmscon holds the template link, run these from a desktop terminal or over ssh, after logging out of each VT where kmscon runs:

```sh
sudo systemctl disable kmsconvt@.service
systemctl list-units 'kmsconvt@*'
sudo systemctl stop 'kmsconvt@*.service'    # ends every session on each kmscon VT
sudo vtium console enable --all
```

`systemctl disable kmsconvt@.service` removes every link to kmscon's unit: the template link, and each VT's own link and boot entry. It stops no kmscon that is running, and logging out of a VT leaves kmscon running there. `systemctl list-units` shows each VT where kmscon is loaded, and `systemctl stop` stops kmscon on all of them. `vtium console enable --all` leaves a VT where kmscon still runs to kmscon, getty and link included, and names it, except on VT 6, where it refuses until kmscon stops there. So stop kmscon first; `sudo vtium console enable N` takes such a VT later, once kmscon has stopped there. `vtium console enable --all` then links the template to vtium, enables vtium at boot on VT 1 unless a display manager keeps VT 1, and disables the gettys that systemd starts at boot on VTs 2 to 6, so vtium starts on each of them the first time you switch to it. Run the start command it prints to start vtium on VT 1 now rather than at the next boot.

kmscon is then installed but runs nowhere, and `sudo pacman -R kmscon` removes it. That leaves `kmscon-terminfo`, which kmscon depends on and which holds the terminfo entry named `kmscon`. Programs read that entry in a session whose `TERM` is `kmscon`: a vtium run with `term=kmscon` (above), or an ssh login to this machine from a machine running kmscon. Keep `kmscon-terminfo` while anything here runs with `TERM=kmscon`, and otherwise `sudo pacman -R kmscon-terminfo` removes it.

To go back to kmscon on every VT, run `sudo vtium console disable --all`, which gives back the gettys and the template link as `vtium console enable --all` found them, and then `sudo systemctl enable kmsconvt@.service` with kmscon installed. That links the template to kmscon and enables kmscon at boot on VT 1; where a display manager starts on VT 1, `sudo systemctl disable kmsconvt@tty1.service` leaves VT 1 to it.

## Upgrades

An upgrade never restarts a running vtium, because a restart ends every session on that VT. Each VT runs the new build from its next start: at a reboot, after a failure, or when you restart it on purpose. After each upgrade pacman says so and names the VTs whose vtium still runs the build it started with. To restart VT 3 now, log out there first, then, from another VT or over ssh:

```sh
sudo systemctl restart vtium@tty3.service
```

## Removing vtium

Before you remove the package, while `/usr/bin/vtium` is still installed, give every VT back, and stop vtium on each VT where it runs:

```sh
sudo vtium console disable --all
systemctl list-units 'vtium@*'
sudo systemctl stop vtium@tty3.service    # for each VT listed; ends every session on that VT
sudo pacman -R vtium
```

The last command names the console package you installed, such as `sudo pacman -R vtium-minspec`.

`vtium console disable --all` stops nothing, so a vtium that runs keeps its VT until it stops. Stopping it ends every session on that VT, so log out there first, and run the stop from another VT or over ssh. A vtium left running keeps running from memory after the removal until it stops, and pacman names it when the removal starts.

Removing the package changes no unit, so a VT you did not give back keeps links to `vtium@.service`, which then point at nothing, and logind starts no console on that VT. pacman names each such link after the removal, with the commands that remove it and enable its VT's getty at boot again. The records that `vtium console` kept stay in `/var/lib/vtium/console/`, so installing vtium again and running `sudo vtium console disable --all` puts back exactly what was there.

## Publishing to the AUR

`aur-publish DIR` pushes the recipe in DIR to `ssh://aur@aur.archlinux.org/<pkgbase>.git`, under the SSH key of whoever runs it, so the maintainer runs it after a change here is reviewed. It splits DIR's history out with `git subtree split --prefix DIR`, so each push carries on from the last one, and pushes it to the AUR repository's `master` branch, the only branch the AUR takes. First it checks:

- that DIR is committed, and its `.SRCINFO` is what `makepkg --printsrcinfo` prints;
- that no checksum is a placeholder;
- that a recipe and its `-git` twin, such as `vtium` and `vtium-git`, share the same steps, everything in `PKGBUILD` from the line `# Everything below is the same`, and the same copy of every other file both carry, such as `LICENSE` and the `terminus` frit, since each AUR repository must stand alone and so carries its own copy; a recipe whose twin is not in this repository, such as `vtium-minspec` today, has nothing to compare;
- that a release's tag exists on GitHub, and that it is signed by a key in `validpgpkeys`, checked against the key files in DIR's `keys/pgp/` alone, in a keyring of its own, with no configuration of the user's, so it never reaches the user's own keys or asks for a passphrase;
- that every commit it pushes holds `PKGBUILD` and `.SRCINFO`, as the AUR asks.

`--dry-run` runs the checks and prints the push without running it. `./test-aur-publish` runs each refusal against a scratch clone, a scratch source repository and scratch keys, and pushes nothing. `./test-install-messages` checks what pacman prints after an install, an upgrade and a removal, for every console package's install script, against a scratch `/etc/systemd/system` and `/var/lib/vtium`, and that each console package's script is the same as `vtium`'s.

```sh
./aur-publish --dry-run vtium-git
./aur-publish vtium-git
```

Before you commit a change to a `PKGBUILD`, make its `.SRCINFO` again with the build variables unset, and set `vtium-git`'s `pkgver` back to the value for GitHub's main, since makepkg rewrites it after each build:

```sh
env -u VTIUM_GIT_URL -u VTIUM_GIT_BRANCH -u VTIUM_FEATURES makepkg --printsrcinfo > .SRCINFO
```

Each recipe built from a release tag waits for its tag: `aur-publish` refuses it until the maintainer has signed and pushed the tag its `_tag` names.

The AUR gives each package name to one AUR repository. `vtium-terminfo` was built by the `vtium` recipe until it moved to a recipe of its own, so publish `vtium` first, which no longer builds it, and `vtium-terminfo` after. `vtium-git` still builds `vtium-terminfo-git`.

## License

The PKGBUILDs and the files in this repository are under the MIT license, as vtium is. Each recipe's directory carries the license, so each AUR repository does too.
