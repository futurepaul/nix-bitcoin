nix-bitcoin
===

[![Build Status](https://travis-ci.org/fort-nix/nix-bitcoin.svg?branch=master)](https://travis-ci.org/fort-nix/nix-bitcoin)

Nix packages and nixos modules for easily installing Bitcoin nodes and higher layer protocols with an emphasis on security.
This is a work in progress - don't expect it to be bug-free, secure or stable.

The default configuration sets up a Bitcoin Core node and c-lightning. The user can enable spark-wallet in `configuration.nix` to make c-lightning accessible with a smartphone using spark-wallet.
A simple webpage shows the lightning nodeid and links to nanopos letting the user receive donations.
It also includes elements-daemon.
Outbound peer-to-peer traffic is forced through Tor, and listening services are bound to onion addresses.

A demo installation is running at [http://6tr4dg3f2oa7slotdjp4syvnzzcry2lqqlcvqkfxdavxo6jsuxwqpxad.onion](http://6tr4dg3f2oa7slotdjp4syvnzzcry2lqqlcvqkfxdavxo6jsuxwqpxad.onion).
The following screen cast shows a fresh deployment of a nix-bitcoin node.

<p align="center">
  <a href="https://asciinema.org/a/223630/?speed=2&autoplay=1"><img src="https://asciinema.org/a/223630.png" height="500"></a>
</p>



The goal is to make it easy to deploy a reasonably secure Bitcoin node with a usable wallet.
It should allow managing bitcoin (the currency) effectively and providing public infrastructure.
It should be a reproducible and extensible platform for applications building on Bitcoin.

Examples
---
The easiest way to try out nix-bitcoin is to use one of the provided examples.

```bash
git clone https://github.com/fort-nix/nix-bitcoin
cd nix-bitcoin/examples/
nix-shell
```

The following example scripts set up a nix-bitcoin node according to [`examples/configuration.nix`](examples/configuration.nix) and then
shut down immediately. They leave no traces (outside of `/nix/store`) on the host system.

- [`./deploy-container.sh`](examples/deploy-container.sh) creates a [NixOS container](https://github.com/erikarvstedt/extra-container).\
  This is the fastest way to set up a node.\
  Requires: [Nix](https://nixos.org/), a systemd-based Linux distro and root privileges

- [`./deploy-qemu-vm.sh`](examples/deploy-qemu-vm.sh) creates a QEMU VM.\
  Requires: [Nix](https://nixos.org/nix/)

- [`./deploy-nixops.sh`](examples/deploy-nixops.sh) creates a VirtualBox VM via [NixOps](https://github.com/NixOS/nixops).\
  NixOps can be used to deploy to various other backends like cloud providers.\
  Requires: [Nix](https://nixos.org/nix/), [VirtualBox](https://www.virtualbox.org)

- [`./deploy-container-minimal.sh`](examples/deploy-container-minimal.sh) creates a
  container defined by [minimal-configuration.nix](examples/minimal-configuration.nix) that
  doesn't use the [secure-node.nix](modules/presets/secure-node.nix) preset.
  Also shows how to use nix-bitcoin in an existing NixOS config.\
  Requires: [Nix](https://nixos.org/), a systemd-based Linux distro and root privileges

Run the examples with option `--interactive` or `-i` to start a shell for interacting with
the node:
```bash
./deploy-qemu-vm.sh -i
```

#### Tests
The internal test suite is also useful for exploring features.\
The following `run-tests.sh` commands leave no traces (outside of `/nix/store`) on
the host system.

```bash
git clone https://github.com/fort-nix/nix-bitcoin
cd nix-bitcoin/test

# Run a Python test shell inside a VM node
./run-tests.sh debug
print(succeed("systemctl status bitcoind"))

# Run a node in a container. Requires systemd and root privileges.
./run-tests.sh container
c systemctl status bitcoind

# Explore a single feature
./run-tests.sh --scenario electrs container
```
See [`run-tests.sh`](test/run-tests.sh) for a complete documentation.

Available modules
---
By default the `configuration.nix` provides:
* bitcoind with outbound connections through Tor and inbound connections through a hidden service. By default loaded with banlist of spy nodes.
* [clightning](https://github.com/ElementsProject/lightning) with outbound connections through Tor, not listening
* includes "nodeinfo" script which prints basic info about the node
* adds non-root user "operator" which has access to bitcoin-cli and lightning-cli

In `configuration.nix` the user can enable:
* a clightning hidden service
* [liquid](https://github.com/elementsproject/elements)
* [lightning charge](https://github.com/ElementsProject/lightning-charge)
* [nanopos](https://github.com/ElementsProject/nanopos)
* an index page using nginx to display node information and link to nanopos
* [spark-wallet](https://github.com/shesek/spark-wallet)
* [electrs](https://github.com/romanz/electrs)
* recurring-donations, a module to repeatedly send lightning payments to recipients specified in the configuration.
* [bitcoin-core-hwi](https://github.com/bitcoin-core/HWI).
  * You no longer need extra software to connect your hardware wallet to Bitcoin Core. Use Bitcoin Core's own **H**ardware **W**allet **I**nterface with one `configuration.nix` setting.

The data directories of the services can be found in `/var/lib` on the deployed machines.

Installation
---
See [install.md](docs/install.md) for a detailed tutorial.

Security
---
* **Simplicity:** Only services you select in `configuration.nix` and their dependencies are installed, packages and dependencies are [pinned](pkgs/nixpkgs-pinned.nix), most packages are built from the [nixos stable channel](https://github.com/NixOS/nixpkgs-channels/tree/nixos-19.03), with a few exceptions that are built from the nixpkgs unstable channel, builds happen in a [sandboxed environment](https://nixos.org/nix/manual/), code is continuously reviewed and refined.
* **Integrity:** Nix package manager, NixOS and packages can be built from source to reduce reliance on binary caches, nix-bitcoin merge commits are signed, all commits are approved by multiple nix-bitcoin developers, upstream packages are cryptographically verified where possible, we use this software ourselves.
* **Principle of Least Privilege:** Services operate with least privileges; they each have their own user and are restricted further with [systemd options](modules/nix-bitcoin-services.nix), there's a non-root user *operator* to interact with the various services.
* **Defense-in-depth:** nix-bitcoin is built with a [hardened kernel](https://github.com/NixOS/nixpkgs/blob/master/nixos/modules/profiles/hardened.nix) by default, services are confined through discretionary access control, Linux namespaces, and seccomp-bpf with continuous improvements.

Note that nix-bitcoin is still experimental.
Also, by design if the machine you're deploying *from* is insecure, there is nothing nix-bitcoin can do to protect itself.

Hardware requirements
---
* Disk space: 300 GB (235GB for Bitcoin blockchain + some room)
  * Bitcoin Core pruning is not supported at the moment because it's not supported by c-lightning. It's possible to use pruning but you need to know what you're doing.
* RAM: 2GB of memory. ECC memory is better. Additionally, it's recommended to use DDR4 memory with targeted row refresh (TRR) enabled (https://rambleed.com/).

Tested hardware includes [pcengine's apu2c4](https://pcengines.ch/apu2c4.htm), [GB-BACE-3150](https://www.gigabyte.com/Mini-PcBarebone/GB-BACE-3150-rev-10), [GB-BACE-3160](https://www.gigabyte.com/de/Mini-PcBarebone/GB-BACE-3160-rev-10#ov).
Some hardware (including Intel NUCs) may not be compatible with the hardened kernel turned on by default (see https://github.com/fort-nix/nix-bitcoin/issues/39#issuecomment-517366093 for a workaround).

Usage
---
For usage instructions, such as how to connect to spark-wallet, electrs and the ssh Tor Hidden Service, see [usage.md](docs/usage.md).

Troubleshooting
---
If you are having problems with nix-bitcoin check the [FAQ](docs/faq.md) or submit an issue.
There's also a `#nix-bitcoin` IRC channel on freenode.
We are always happy to help.

Docs
---
* [FAQ](docs/faq.md)
* [Install instructions](docs/install.md)
* [Usage instructions](docs/usage.md)
