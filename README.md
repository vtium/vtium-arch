# VTium for Arch Linux

This repository is the Arch Linux packaging of [vtium](https://github.com/vtium/vtium), a terminal for the Linux console that draws through KMS/DRM. Each directory holds one recipe, a `PKGBUILD` and the files beside it. Five of them are published on the AUR (Arch User Repository), each as the package of the same name: `vtium`, `vtium-git`, `vtium-terminfo`, `vtium-terminfo-git` and `vtium-fonts`. vtium keeps this repository as its submodule `packaging/arch`. vtium's [docs/install.md](https://github.com/vtium/vtium/blob/main/docs/install.md) says how to install the packages and give vtium a VT, and the comments in each `PKGBUILD` say what the package installs and how it builds.

## The recipes

| Directory | What it builds | What it builds from |
|---|---|---|
| `vtium/` | the console for a desktop or laptop, with the features `images logind gpu bidi` and the GPU path on by default | a release: the tag `v0.1.0-alpha.6`, checked against the maintainer's signature |
| `vtium-git/` | the same console with the feature `libseat` too, for seat mode; `vtium` builds it that way from the first release that has the feature | the tip of vtium's `main` |
| `vtium-minspec/` | vtium's smallest console, with no feature | the same release tag |
| `vtium-minspec-git/` | the same console | the tip of `main` |
| `vtium-minspec-gpu/` | the smallest console with the feature `gpu`, and the GPU path off by default | the same release tag |
| `vtium-minspec-gpu-git/` | the same console | the tip of `main` |
| `vtium-terminfo/` | vtium's terminfo entry `xterm-vtium`, which each console built from a release depends on | the same release tag |
| `vtium-terminfo-git/` | the same entry, for the `-git` consoles | the tip of `main` |
| `vtium-fonts/` | a curated set of fonts chosen to go with vtium's frits (font set files), which grows as new frits are added; it installs no font itself and depends on the distribution's font packages | this repository |

The four minspec recipes are never published on the AUR, because they are build options of `vtium` rather than packages of their own. `ci/unpublished` names them, and the build still builds and checks them. In `vtium/`, `VTIUM_FEATURES= makepkg` builds the minspec console and `VTIUM_FEATURES=gpu makepkg` the minspec console with the GPU path, each with the `vtium` recipe's dependencies, frits and build settings.

The maintainer signs each release tag, and makepkg refuses to build a release recipe until the tag's signature checks out against the key in `validpgpkeys`. Each release recipe carries that key in `keys/pgp/`, so import it once with `gpg --import vtium/keys/pgp/*.asc`. A release recipe's `pkgver` is its tag without the `v` and without the hyphen before a pre-release, so `v0.1.0-alpha.6` is `0.1.0alpha.6`, which pacman sorts before `0.1.0`.

## What a console package does to your VTs

A VT, or virtual terminal, is one of the full-screen text screens the Linux kernel keeps beside a desktop. When you first install a console package, its install script runs `vtium console enable --direct --all`. That gives vtium every free VT in direct mode, which is every VT that no display manager, greeter, graphical session or kmscon holds. systemd then starts vtium at boot on each of those VTs, in place of its getty, so vtium holds each of them open from boot. With a display manager enabled, VT 1 stays the display manager's. The script starts nothing now, so the change takes effect at the next boot. The install message shows what `vtium console` did, and names each VT it left alone and why.

vtium runs on a VT in one of two modes. In direct mode, which the install turns on, vtium opens the screen and the keyboards itself, and each login at its prompt gets an active session, as a login at a getty does. In seat mode, vtium takes the screen, the keyboards and VT switching from logind through libseat, and each login gets a session with no seat. polkit treats that session as inactive, and udev gives it no devices, such as sound cards. Seat mode is your choice, on a console built with the features `libseat` and `logind`, as `vtium-git` is: `sudo vtium console enable --seat --all` moves every VT vtium holds to seat mode, and `vtium console enable` with no mode changes nothing and says which mode fits your machine and why. vtium's install guide describes both, in [Direct mode and seat mode](https://github.com/vtium/vtium/blob/main/docs/install.md#direct-mode-and-seat-mode).

`vtium console` refuses to change anything while kmscon holds the template link `/etc/systemd/system/autovt@.service`, while `loginctl` cannot say which sessions run, and on a machine not booted with systemd, such as a chroot. The install still succeeds. Its message then says that vtium claimed no VT, gives the reason, and names the command to run once the reason is gone, `sudo vtium console enable --direct --all`.

An upgrade changes no VT, with one exception, for machines that vtium `v0.1.0-alpha.6` set up. That release claimed no VT when it installed. Its install message told you to run `sudo vtium console enable --all`, which links the template `/etc/systemd/system/autovt@.service` to vtium and gives no VT a link of its own. That link starts vtium on a VT only when you switch to that VT, so vtium does not start at boot. When an upgrade or an install finds that template link and no other link to vtium, it gives vtium the free VTs once, as a first install does, so that vtium starts at boot on them. Its message says why, and shows what `vtium console` did. This happens when `vtium` upgrades from `v0.1.0-alpha.6`, and also when `vtium-git` or another console package replaces it, because the removal script of `v0.1.0-alpha.6` gives no VT back and leaves the template link in place. If you gave the VTs back with `sudo vtium console disable --all`, the template link no longer names vtium, so the upgrade changes nothing.

On a machine where vtium holds VTs through the template link and some VTs have links of their own, the upgrade changes no VT. vtium then does not start at boot on the VTs from 2 to 5 that have no link of their own. The upgrade's message names those VTs and the command that starts vtium at boot there, `sudo vtium console enable --direct --all`, and `vtium console status` names them too.

You can choose otherwise at any time:

```sh
vtium console status                    # what each VT runs
sudo vtium console disable --all        # take every VT back
sudo vtium console enable --direct 3    # then give vtium VT 3 alone
sudo systemctl start vtium@tty3.service
```

The `systemctl start` line starts vtium on VT 3 now, and stops the getty there. Run it from another VT or over ssh, once no session on VT 3 needs to survive. Without it, vtium starts on VT 3 at the next boot. vtium's [install guide](https://github.com/vtium/vtium/blob/main/docs/install.md#giving-vtium-a-vt) gives the full rules.

### Sharing VTs with kmscon

kmscon and vtium can each keep VTs of their own on one machine. A VT's own link, `/etc/systemd/system/autovt@ttyN.service`, wins over the template link. `vtium console enable --direct N` refuses a VT that kmscon holds, so to give that VT to vtium, disable `kmsconvt@ttyN` there first. To give kmscon back a VT that vtium holds, run `sudo vtium console disable N` and then link the VT to kmscon. To move from kmscon to vtium on every VT, take the template link back from kmscon, then run `sudo vtium console enable --direct --all`. vtium's install guide gives each step, in [Sharing VTs with kmscon](https://github.com/vtium/vtium/blob/main/docs/install.md#sharing-vts-with-kmscon) and [Moving from another DRM/KMS VT](https://github.com/vtium/vtium/blob/main/docs/install.md#moving-from-another-drmkms-vt).

### Removing vtium

Before the package's files go, its removal script runs `vtium console disable --all`, which gives back the template link and the gettys that the claim took. It stops nothing, so a vtium that runs keeps its VT until it stops or the machine restarts. The removal message names each vtium that still runs. pacman removes a console package that another one replaces in the same way, so the new package then claims the free VTs again when it installs.

When `vtium console disable --all` cannot give a VT back, the removal still succeeds, and the message at its end names each link to vtium's units, `vtium@.service` and `vtium-seat@.service`, that is left. A left link names a unit that no longer exists, so logind starts nothing on that VT. For each link, the message gives the `sudo rm` line that removes it, and the line that enables that VT's getty at boot again. Then run `sudo systemctl daemon-reload`.

## How a new release reaches the AUR

The watcher, `.github/workflows/update.yml`, looks for a new vtium release every day, and at once when vtium sends this repository a `repository_dispatch` event of type `vtium-release`. When vtium has published a release newer than the tag the release recipes build, `./bump-release` moves `vtium`, `vtium-minspec`, `vtium-minspec-gpu` and `vtium-terminfo` to that tag together, and the watcher opens a pull request from the branch `bump/vtium-release` and queues it to merge. It takes pre-releases too, while `bump-release` sets `include_prereleases=yes`. The build, `.github/workflows/build.yml`, builds every recipe in a clean Arch container for each pull request, lints the packages with namcap, and runs `aur-publish --dry-run` and the tests. Its check `gate`, which main's branch rule requires, lets the pull request merge. On main, the build's job `publish` compares each published recipe with its AUR repository and runs `aur-publish` for each one the AUR is behind on or does not have yet, so a run publishes what an earlier run missed. The consoles go first and `vtium-fonts` and the terminfo recipes after them, and `publish` pushes no console whose terminfo recipe cannot follow it in the same run. The watcher needs the secret `BUMP_PAT` and publishing needs the secret `AUR_SSH_KEY`, and each job stops with an error when its secret is missing.

To hold a bump for review, add an empty file `ci/<recipe>/.no-auto-merge`: the watcher still opens the pull request, which then waits for the maintainer to merge it. The watcher turns off the auto-merge of a held pull request, and it leaves an open bump pull request alone when it already proposes the newest tag or carries a commit the watcher did not write, so a commit pushed onto it stays. `ci/vtium/.no-auto-merge` holds the move past `v0.1.0-alpha.6` until a person adds the feature `libseat` and its dependency `seatd` to that pull request, as the file says, since the install script runs `vtium console enable --direct --all`, which the console of `v0.1.0-alpha.6` does not take. The files that only CI reads live under `ci/`, because `aur-publish` pushes a recipe's whole directory to the AUR.

## Publishing and testing by hand

`aur-publish` pushes one recipe's history to its AUR repository under the SSH key of whoever runs it, and its header lists every check it makes first. Publish `vtium` before `vtium-terminfo`, and `vtium-git` before `vtium-terminfo-git`, since the console recipes built the terminfo packages before those had recipes of their own.

```sh
./aur-publish --dry-run vtium-git   # runs every check and prints the push
./aur-publish vtium-git             # pushes
./test-aur-publish && ./test-install-messages && ./test-bump-release && ./test-publish-aur
```

Before you commit a change to a `PKGBUILD`, make its `.SRCINFO` again with the build variables unset: `env -u VTIUM_GIT_URL -u VTIUM_GIT_BRANCH -u VTIUM_FEATURES makepkg --printsrcinfo > .SRCINFO`.

## License

The PKGBUILDs and the files in this repository are under the MIT license, as vtium is. Each recipe's directory carries the license, so each AUR repository does too.
