One advantage of MirageVPN is its ability to produce a [unikernel][mirageos],
i.e. a fully-fledged operating system for a virtualizer such as [Xen]. In this
respect, [QubesOS][qubesos] users can secure their VMs' Internet connections via
a MirageVPN client.

Unlike a standard OpenVPN client, the VM used is smaller and its attack surface
just as reduced, as the operating system only does VPN: an OpenVPN client would
require a kernel (like Linux), a kernel configuration to redirect VM connections
to the VPN tunnel, and several libraries installed in order to function.

In this chapter, we'll look at how to configure and install a MirageVPN client
for QubesOS. We'll assume that the OpenVPN server is configured as described in
[this handbook](./simple_openvpn_server.md).

## Download and configuration

You can download the unikernel from its official repository:
https://github.com/robur-coop/qubes-miragevpn

Next, copy the unikernel to dom0 with this command (from a dom0 terminal). In
this example, we've uploaded the unikernel to the _app-vm_ `personal`:
```sh
$ mkdir -p /var/lib/qubes/vm-kernels/qubes-miragevpn/
$ cd /var/lib/qubes/vm-kernels/qubes-miragevpn/
$ qvm-run -p personal 'cat qubes-miragevpn.xen' > vmlinuz
```

Still from dom0, we now need to create a new VM from the downloaded image. An
empty initramfs file must also be created (for QubesOS < 4.2):
```sh
$ gzip -n9 < /dev/null > initramfs
$ qvm-create \
  --property kernel=qubes-miragevpn \
  --property kernelopts='' \
  --property memory=256 \
  --property maxmem=256 \
  --property netvm=sys-net \
  --property provides_network=True \
  --property vcpus=1 \
  --property virt_mode=pvh \
  --label=green \
  --class StandaloneVM \
  qubes-miragevpn
$ qvm-features qubes-miragevpn no-default-kernelopts 1
```

The configuration of the MirageVPN client for QubesOS is constrained in the same
way as our MirageVPN client for KVM: the configuration must fit into a single
file. This file must be compressed and named to `config.ovpn` with the `tar`
command and made available to the unikernel via `qvm`.
```sh
$ cp alice.config config.ovpn
$ tar cvf config.tar config.ovpn
$ qvm-volume import qubes-miragevpn:root config.tar
```

Finally, we need to configure a VM (like personal) to use our VPN client for all
connections.
```sh
$ qvm-prefs --set <my-app-vm> netvm qubes-miragevpn
```

[mirageos]: https://mirage.io/
[qubesos]: https://www.qubes-os.org/
