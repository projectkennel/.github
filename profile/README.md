# Project Kennel

**Keep the identity. Limit the authority.**

Unix treats a user ID as both an identity and a source of authority. A process running as you can generally reach what you can reach. That made sense when most code under your account was your own, supplied by the operating system, or obtained from sources you had chosen to trust. Today the same account routinely runs package scripts, build dependencies, fresh repositories, and AI agents. They inherit your authority simply because they run as your user ID.

The dog is a good boy. Leave him loose in the house while you are out, though, and he may enthusiastically wreck it. A kennel does not need to decide whether the dog is good. It gives him what he needs and keeps the rest of the house out of reach.

Project Kennel does that for code. It runs a workload under your identity with only the authority a signed policy grants. It does not judge intent or classify a program as benign or malicious. The boundary applies just as much to an agent trying hard to finish a task as to a hostile dependency.

## How the boundary works

- **Construct a smaller world.** Only granted files and directories appear in the workload's view. Landlock, seccomp, namespaces, capability limits, and an execution allowlist reinforce it.
- **Mediate access that crosses the boundary.** TCP and UDP egress, local sockets, and D-Bus capabilities are granted specifically. The contained UDP path answers workload DNS queries locally with synthetic addresses for approved names; those queries do not escape onto the network.
- **Use SSH without handing over your key.** The workload holds a disposable synthetic key for a granted destination. A host-side bastion binds it to that destination and uses the real key outside the kennel. Neither the real key nor an SSH agent socket enters the workload.
- **Show what was granted.** Signed, inheritable policies carry reasons for grants. `kennel policy risks` and `kennel policy diff` expose their consequences and changes; runtime decisions are audited. Agents can delegate narrower work to scoped sub-kennels, and OCI images run within the same authority model.

Kennel confines what code can reach, regardless of how it behaves. Data a workload is allowed to read can still leave through a service it is allowed to use. The policy makes those permissions explicit, enforceable, and reviewable.

## Get started

Kennel runs on Linux (kernel 6.10 or newer, Landlock ABI 6 or newer). It is available from signed Debian/Ubuntu and Fedora/RHEL package repositories. Follow the [installation guide](https://github.com/projectkennel/projectkennel/blob/main/INSTALL.md), then the [how-to guide](https://github.com/projectkennel/projectkennel/blob/main/HOWTO.md). For example:

```bash
kennel run claude
```

## Explore the project

- [Source, releases, and current status](https://github.com/projectkennel/projectkennel)
- [The design book](https://github.com/projectkennel/books)
- [Threat catalogue](https://github.com/projectkennel/projectkennel/blob/main/docs/reference/THREATS.md)
- [Contributing](https://github.com/projectkennel/projectkennel/blob/main/.github/CONTRIBUTING.md) · [Report a vulnerability](https://github.com/projectkennel/projectkennel/blob/main/.github/SECURITY.md)
- [Website](https://projectkennel.org/) · [Packages](https://packages.projectkennel.org/)

Designed and maintained by [Remco van Mook](https://www.linkedin.com/in/remcovm/). Licensed under Apache-2.0, with GPL-2.0 for the host-mode egress BPF; see the [licence](https://github.com/projectkennel/projectkennel/blob/main/LICENSE) and [notice](https://github.com/projectkennel/projectkennel/blob/main/NOTICE).
