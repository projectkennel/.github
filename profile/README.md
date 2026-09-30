# Project Kennel: keep the identity, limit the authority

Unix treats a user ID as both an identity and a source of authority. A process running as you can generally reach what you can reach. That was a workable assumption when most programs under your account were your own tools and scripts, supplied by the operating system, or obtained from sources you had explicitly chosen to trust. Today the same account routinely runs package install scripts, build dependencies, fresh repositories, and AI agents that execute code they or others have written. They inherit your authority simply because they run as your user ID.

The dog is a good boy. Leave him loose in the house while you are out, though, and he may enthusiastically wreck it. A kennel does not need to decide whether the dog is good. It gives him room to do what he needs to do and keeps the rest of the house out of reach.

Project Kennel makes that distinction for processes. The workload keeps the user’s identity but receives only the authority its task requires. Kennel does not classify code as benign or malicious, or try to infer its intent. An agent pursuing a legitimate goal can be just as capable of crossing a boundary as a hostile dependency. The boundary must hold either way.

A kennel starts with a constructed view of the machine. Its signed policy grants specific files, writable paths, and executable programs; other parts of the user’s home are absent. Trusted construction code builds that view, then the workload runs as the ordinary user without the authority used to construct it. Namespaces, Landlock, seccomp, and capability limits reinforce the boundary.

Access to host services is mediated throughout the workload’s life. TCP and UDP connections are constrained by policy and brokered across its network boundary. The UDP path answers DNS queries locally, using synthetic addresses for approved names rather than leaking workload DNS questions onto the network. Local sockets and D-Bus services can be exposed as specific capabilities.

SSH shows why preserving identity need not mean handing over authority. For each granted destination, the workload receives a disposable synthetic key. The user’s real key and SSH agent socket stay outside. A host-side bastion binds the synthetic key to its approved destination before making the real connection, so the workload can use SSH without acquiring a general signing capability.

The operator can inspect the bargain. Policy grants carry reasons; risk and diff commands show their consequences and how a change alters them. Runtime decisions are audited. An agent can give a child task a narrower sub-kennel rather than passing along all its own access, and an imported OCI image runs within the same authority model.

**Kennel separates the Unix question “who is this process?” from “what may this process do?”** It lets code work under your identity without giving it the run of your account. It does not promise that permitted data cannot leave through a permitted service; it makes the permissions explicit, enforceable, and reviewable.

```bash
apt install kennel        # Debian/Ubuntu   (dnf install kennel on Fedora/RHEL)
kennel run claude         # run an agent confined to a repo, a toolchain, a few registries
```

<!-- DEMO: asciinema/GIF here — an agent inside `kennel run claude` denied on ~/.ssh, allowed on the repo. ~20 seconds. -->

It's the enforcement the user level never had. The host has confined untrusted code for decades (SELinux, AppArmor, seccomp, the LSM framework), but your *account* — where agents and unsigned code now run — never did. Kennel keeps your uid and splits the **authority** off it: the workload runs as you, with exactly what its policy grants, checked one access at a time. That is a **reference monitor**. Where a sandbox or container draws its line once at launch and steps back, the monitor stays in the path for as long as the workload runs — cheap enough (≈3.7 ms to spawn) to do per task and throw away.

## Install

Signed package repositories the project hosts and signs itself: no `curl | sh` anywhere (that's threat T1.4). You import **one** key, cross-check its fingerprint against three independent channels (this repo, the GitHub release, the domain's DNS `TXT` record), and `apt`/`dnf` verify every package and metadata refresh against it thereafter.

**Debian / Ubuntu:**
```bash
curl -fsSL https://packages.projectkennel.org/kennel-archive-keyring.asc | gpg --dearmor | sudo tee /usr/share/keyrings/kennel.gpg >/dev/null
gpg --show-keys /usr/share/keyrings/kennel.gpg          # cross-check the fingerprint first
echo "deb [signed-by=/usr/share/keyrings/kennel.gpg] https://packages.projectkennel.org/deb stable main" | sudo tee /etc/apt/sources.list.d/kennel.list
sudo apt update && sudo apt install kennel
```
**Fedora / RHEL** (the `.rpm` loads the SELinux module for you):
```bash
sudo curl -fsSL https://packages.projectkennel.org/rpm/kennel.repo -o /etc/yum.repos.d/kennel.repo
sudo rpm --import https://packages.projectkennel.org/kennel-archive-keyring.asc
sudo dnf install kennel
```
Signing key **`663C 67B0 9FDD A9EE E57F A295 88D5 8446 1C4D 6EE9`** (also at `_kennel-key.projectkennel.org`). Installing from a tarball or source, and the full post-install setup, are in [INSTALL.md](INSTALL.md).

## What it does

- **Construction by absence.** The workload's world is built from nothing, granted paths only. What isn't granted is *absent*, not denied: nothing to enumerate, nothing to probe.
- **Deny-by-default network**, four modes (`none` / proxied `constrained` / `unconstrained` / `host`), egress brokered and audited.
- **SSH with no signing oracle.** A per-user re-origination bastion; the sandbox holds a disposable synthetic key bound to one `(host, key)` edge: never your real key, never an agent socket.
- **Dynamic spawn + a service mesh.** A confined agent spawns scoped, signed-template sub-kennels and consumes brokered capabilities by name: deny-by-default, depth-1, reaped with the agent.
- **Confined GUI** (a per-kennel nested Wayland compositor), **OCI images** (digest-pinned rootfs), and **workspace-trust pinning** (a masked manifest the agent can rewrite but cannot forge).
- **Unprivileged by construction.** `kenneld` runs as you with no standing privilege; a single small file-capped helper builds the namespaces, operator-owned (not root). There is no `sudo` in the spawn.
- **Confinement, not detection.** The boundary never judges intent, so being wrong about the code is not a breach. A unified, structured audit log records every decision, and the trusted base only shrinks — 30 crates, every line of `unsafe` quarantined to 5 small ones.

**Isn't this just bubblewrap?** Same engine, different vehicle: both build on unprivileged user namespaces, but bwrap mediates once, at launch, and `exec`s away — Kennel's monitor stays in the path and clears every crossing at runtime, against a signed, versioned policy rather than a wall of flags. The namespace is arranged differently too: root *inside* it belongs to the enforcement scaffolding, not the workload, which runs as your mapped uid — so the kernel's ordinary DAC keeps enforcing against tampering even after the namespace boundary is up. The full comparison is [on the site](https://projectkennel.org/#fit).

Policy is signed, versioned, and inheritable, and describes kernel-level constraints rather than behaviour: the same policy confines an AI agent, a Postgres container, or an `npm install`. The full treatment lives in the book (below).

## Status

**0.7.x**, versioned on a stable-surface cadence ([CHANGELOG](CHANGELOG.md)). It runs the full vertical **unprivileged** on stock Linux (kernel ≥ 6.10, Landlock ABI ≥ 6), proven end-to-end on **Debian/Ubuntu** (AppArmor is the userns substrate) and on **Fedora, enforcing SELinux** (a two-domain module keeps the monitor and the workload as distinct SELinux subjects). Pre-1.0: interfaces may still change.

## Read more

- **The book** ([`books/`](https://github.com/projectkennel/books), separate repo) — the corpus: Vol 1 the platform-neutral design, Vol 2 the Linux realisation. The authoritative "what it is and why."
- **[THREATS.md](docs/reference/THREATS.md)** — the threat catalogue (stable IDs, incident citations, MITRE/compliance mappings). The durable, portable contribution: cite it even if you never run the code.
- **Using it:** [INSTALL.md](INSTALL.md) → [HOWTO.md](HOWTO.md) → [HOWTO-admin.md](HOWTO-admin.md), and the installed man pages (`man kennel`, `man policy.toml`, `man kenneld`).
- **Contributing:** [CONTRIBUTING.md](.github/CONTRIBUTING.md).

## Citing

To cite the threat catalogue or the design in your own security posture or research:

> Project Kennel, *Threat Catalogue for Untrusted Code at the User Level* (THREATS.md), v0.7, 2026. https://github.com/projectkennel/projectkennel

Machine-readable metadata is in [CITATION.cff](CITATION.cff); GitHub's "Cite this repository" button uses it. Threat IDs (`T1.4`, `T2.1`, …) are stable and safe to reference across versions.

## Who

Kennel is designed and maintained by **[Remco van Mook]** (https://www.linkedin.com/in/remcovm/) — a 30-year system and network veteran who finds himself yelling "You're holding it wrong!" at clouds a bit too often. Contributions welcome per [CONTRIBUTING.md](.github/CONTRIBUTING.md); there is no CLA.

## Reporting a vulnerability

See [SECURITY.md](.github/SECURITY.md). Report privately to security@projectkennel.org; do not open a public issue for a specific exploitable flaw.

## Licence

Apache-2.0 (see [LICENSE](LICENSE) and [NOTICE](NOTICE)). One exception: the host-mode egress BPF under [src/bpf/](src/bpf/) is GPL-2.0, as the kernel requires; everything else is Apache-2.0.

- **Website** <https://projectkennel.org> · **Packages** <https://packages.projectkennel.org> · **Source** <https://github.com/projectkennel/projectkennel>
