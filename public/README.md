# Confinement, redefined.

**Kennel runs code you haven't vetted** — *an AI agent, an npm install, a freshly-cloned repo* — under your own account, confined to just the files, network, and programs a signed policy allows. 

It's the enforcement the user level never had: the host has boxed untrusted code for decades (SELinux, AppArmor, seccomp), but your account, where agents now run, never did. 

Kennel keeps your uid and splits the authority off it, so the code runs as you with only what its policy grants, checked one access at a time.


## Links
[Website](https://www.projectkennel.org/) is here

[The Design and implementation architecture books](https://github.com/projectkennel/books)

[Packages for Ubuntu and Fedora](https://packages.projetkennel.org/)

[Source repo](https://github.com/projectkennel/projectkennel)
