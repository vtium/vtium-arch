# VTium for Arch Linux

This repository is the Arch Linux packaging of [vtium](https://github.com/vtium/vtium), a terminal for the Linux console that draws through KMS/DRM. Each directory holds one recipe, a `PKGBUILD` and the files beside it, and is published as the AUR (Arch User Repository) package of the same name. vtium keeps this repository as its submodule `packaging/arch`. vtium's [docs/install.md](https://github.com/vtium/vtium/blob/main/docs/install.md) says how to install the packages and give vtium a VT, and the comments in each `PKGBUILD` say what the package installs and how it builds.

## The recipes

| Directory | What it builds | What it builds from |
|---|---|---|
| `vtium/` | the console for a desktop or laptop, with the features `images logind gpu` and the GPU path off by default | a release: the tag `v0.1.0-alpha.5`, checked against the maintainer's signature |
| `vtium-git/` | the same console | the tip of vtium's `main` |
| `vtium-minspec/` | vtium's smallest console, with no feature | the same release tag |
| `vtium-minspec-git/` | the same console | the tip of `main` |
| `vtium-minspec-gpu/` | the smallest console with the feature `gpu`, and the GPU path off by default | the same release tag |
| `vtium-minspec-gpu-git/` | the same console | the tip of `main` |
| `vtium-terminfo/` | vtium's terminfo entry `xterm-vtium`, which each console built from a release depends on | the same release tag |
| `vtium-terminfo-git/` | the same entry, for the `-git` consoles | the tip of `main` |
| `vtium-fonts/` | no file of its own: it depends on the fonts that vtium's shipped font sets name | this repository |

The maintainer signs each release tag, and makepkg refuses to build a release recipe until the tag's signature checks out against the key in `validpgpkeys`. Each release recipe carries that key in `keys/pgp/`, so import it once with `gpg --import vtium/keys/pgp/*.asc`. A release recipe's `pkgver` is its tag without the `v` and without the hyphen before a pre-release, so `v0.1.0-alpha.6` is `0.1.0alpha.6`, which pacman sorts before `0.1.0`.

## How a new release reaches the AUR

The watcher, `.github/workflows/update.yml`, looks for a new vtium release every day, and at once when vtium sends this repository a `repository_dispatch` event of type `vtium-release`. When vtium has published a release newer than the tag the release recipes build, `./bump-release` moves `vtium`, `vtium-minspec`, `vtium-minspec-gpu` and `vtium-terminfo` to that tag together, and the watcher opens a pull request from the branch `bump/vtium-release` and queues it to merge. It takes pre-releases too, while `bump-release` sets `include_prereleases=yes`. The build, `.github/workflows/build.yml`, builds every recipe in a clean Arch container for each pull request, lints the packages with namcap, and runs `aur-publish --dry-run` and the tests. Its check `gate`, which main's branch rule requires, lets the pull request merge. On main, the build's job `publish` runs `aur-publish` for each recipe whose directory the push changed. The watcher needs the secret `BUMP_PAT` and publishing needs the secret `AUR_SSH_KEY`, and each job stops with an error when its secret is missing.

To hold a bump for review, add an empty file `ci/<recipe>/.no-auto-merge`: the watcher still opens the pull request, which then waits for the maintainer to merge it. The files that only CI reads live under `ci/`, because `aur-publish` pushes a recipe's whole directory to the AUR.

## Publishing and testing by hand

`aur-publish` pushes one recipe's history to its AUR repository under the SSH key of whoever runs it, and its header lists every check it makes first. Publish `vtium` before `vtium-terminfo`, and `vtium-git` before `vtium-terminfo-git`, since the console recipes built the terminfo packages before those had recipes of their own.

```sh
./aur-publish --dry-run vtium-git   # runs every check and prints the push
./aur-publish vtium-git             # pushes
./test-aur-publish && ./test-install-messages && ./test-bump-release
```

Before you commit a change to a `PKGBUILD`, make its `.SRCINFO` again with the build variables unset: `env -u VTIUM_GIT_URL -u VTIUM_GIT_BRANCH -u VTIUM_FEATURES makepkg --printsrcinfo > .SRCINFO`.

## License

The PKGBUILDs and the files in this repository are under the MIT license, as vtium is. Each recipe's directory carries the license, so each AUR repository does too.
