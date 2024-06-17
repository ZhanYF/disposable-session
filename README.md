# Disposable Session Playground

Disposable Session refers to short-lived workload that are isolated from other parts of the system, for example, a disposable web session will not leave any file behind when its ended.  This repository contains info for setting up and running Chromium browser as a disposable session inside a MicroVM backed by KVM. Using [firecracker][2], minimal attack surface is exposed to the guest system of the session.

Chromium is used here as an example of running complex application but almost any untrusted workload can be made to run inside a MicroVM container.

### Notes

- Arm64 and AMD64 are supported.
- Audio passthrough is not supported yet.
- State clean-up is done by reverting to a known good btrfs snapshot, the host is expected to support btrfs and wayland.

### Setup

#### On Host

1. [Setup firecracker and firectl(1)](https://github.com/firecracker-microvm/firectl)
2. Install waypipe
3. Obtain kernel, rootfs, and preconfigured ssh identity key:

```
wget https://s3.amazonaws.com/spec.ccfc.min/firecracker-ci/v1.9/$(uname -m)/{vmlinux-5.10.217,ubuntu-22.04.ext4,ubuntu-22.04.id_rsa}
chmod 400 ./ubuntu-22.04.id_rsa
```
### Inside MicroVM


----------
#### This project is part of a sponsored project supported by [Linux Australia][1]

[1]: https://linux.org.au/grants/
[2]: https://github.com/firecracker-microvm/firecracker
