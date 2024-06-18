# Disposable Session Playground

Disposable Session refers to short-lived workload that are isolated from other parts of the system, for example, a disposable web session will not leave any file behind when its ended.  This repository contains info for setting up and running Chromium browser as a disposable session inside a MicroVM backed by KVM. Using [firecracker][2], minimal attack surface is exposed to the guest system of the session.

Firefox is used here as an example of running complex application but almost any untrusted workload can be made to run inside a MicroVM container.

### Notes

- Arm64 and AMD64 are supported.
- Audio passthrough is not supported yet.
- State clean-up is done by reverting to a known good btrfs snapshot, the host is expected to support btrfs.

### Setup

1. [Setup firecracker and firectl(1)](https://github.com/firecracker-microvm/firectl)

(Compile your own firecracker binary or use the [official binary release](https://github.com/firecracker-microvm/firecracker/releases) as the binary linked by firectl's README is outdated)

2. Prepare workspace

```
host$ sudo btrfs subvolume create demo
host$ sudo chown -R $(id -u):$(id -g) demo/
```

4. Obtain kernel, rootfs, and preconfigured ssh identity key:

```
host$ cd runtime
host$ wget https://s3.amazonaws.com/spec.ccfc.min/firecracker-ci/v1.9/$(uname -m)/{vmlinux-5.10.217,ubuntu-22.04.ext4,ubuntu-22.04.id_rsa}
host$ chmod 400 ./ubuntu-22.04.id_rsa
```
3. Increase volume size

The default image size is too small for running desktop applications, to increase it:
```
host$ dd if=/dev/zero bs=1G count=5 >> ./ubuntu-22.04.ext4
```
3. Start MicroVM and grow filesystem

```
TAPDEV=tap1
host$ firectl -m 1024 --kernel=vmlinux-5.10.217 --root-drive=ubuntu-22.04.ext4 --tap-device=$TAPDEV/AA:FC:00:00:00:04
```

```
vm# resize2fs /dev/vda
```

5. Configure network bridge on host

```
UPSTREAM=eth0
TAPDEV=tap1
sudo iptables -t nat -D POSTROUTING -o "$UPSTREAM" -j MASQUERADE || true
sudo iptables -D FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT \
    || true
sudo iptables -D FORWARD -i $TAPDEV -o "$UPSTREAM" -j ACCEPT || true
sudo iptables -t nat -A POSTROUTING -o "$UPSTREAM" -j MASQUERADE
sudo iptables -I FORWARD 1 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
sudo iptables -I FORWARD 1 -i $TAPDEV -o "$UPSTREAM" -j ACCEPT
sudo ip addr add 172.20.0.1/24 dev $TAPDEV
sudo ip link set $TAPDEV up
```

### Inside MicroVM

1. Configure internet access

```
vm# ip addr add 172.20.0.2/24 dev eth0
vm# ip route add default via 172.20.0.1
vm# echo nameserver 1.1.1.1 > /etc/resolv.conf
```

Now the MicroVM should have access to the internet

2. Prepare environment

```
vm# unminimize
vm# apt update; apt install wget sudo -y
vm# adduser user # non-root user is required to run browser
```
3. [Install Firefox inside MicroVM](https://support.mozilla.org/en-US/kb/install-firefox-linux#w_install-firefox-deb-package-for-debian-based-distributions)


### Take a known good snapshot

```
host$ sudo btrfs subvolume snapshot demo demo-disposable
```
### Run disposable firefox session

```
host$ cd demo-disposable
host$ firectl -m 1024 --kernel=vmlinux-5.10.217 --root-drive=ubuntu-22.04.ext4 --tap-device=$TAPDEV/AA:FC:00:00:00:04
host$ ssh -X user@172.20.0.2 firefox
```

### Rollback

```
host$ sudo rm -r demo-disposable/*
host$ sudo btrfs subvolume snapshot demo demo-disposable # Reset the content of demo-disposable
```


----------
#### This project is part of a sponsored project supported by [Linux Australia][1]

[1]: https://linux.org.au/grants/
[2]: https://github.com/firecracker-microvm/firecracker
