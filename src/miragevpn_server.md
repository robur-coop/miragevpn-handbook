The MirageVPN server is a unikernel which listens on a port for client
connections, and provides Internet connectivity (with NAT) for all
authenticated clients. A single network interface is used for both client
connections and providing connectivity.

The unikernel will encrypt all packets received from the Internet and send them
to the respective client (if there's a NAT table entry for the quadruple source
IP, destination IP, source port, destination port). All packets received from a
client will be decrypted and forwarded to the Internet. If "client-to-client" is
enabled in the configuration, packets from one client which destination is
another client will be forwarded by the server.

# Scope of the server

The scope is at the moment limited to IPv4 traffic over the tunnel (no IPv6),
the server only listens on TCP (no UDP). Only layer 3 networking is supported
(tun device), there's no support for tap devices.

The server will route all traffic to its default gateway. If "client-to-client"
is specified, packets for other clients will be forwarded to the specific
client.

Only a single network interface is used by the server, where both TCP listening
and forwarding packets (using NAT) is done. NAT will always be used.

## Authentication

The server as is only authenticates via X.509 certificates. It is possible to
authenticate username and password, but since we do can not in a MirageOS
unikernel execute shell scripts, the verification hook script won't work.
If you need username and password authentication, please get in touch via our
issue tracker and we will find a solution.

# Getting the unikernel binary

You can download the unikernel binary from [our reproducible build infrastructure](https://builds.robur.coop/job/miragevpn-server/build/latest). Download the `bin/ovpn-server.hvt` artifact.
If you did that, skip to "VPN Configuration".

## Building from source (alternative)

We will provide reproducible binaries in the future, here we document how to
build the unikernel from source.

### Prerequisites

First, make sure to have ["opam"](https://opam.ocaml.org) and
["mirage"](https://mirage.io) installed on your system.

### Git repository

Do a `git clone https://github.com/robur-coop/miragevpn.git` to retrieve the
source code of the MirageVPN server.

### Building

Inside of the cloned repository, execute `mirage configure` (other targets are
available, please check the mirage documentation):

```sh
$ cd miragevpn/mirage-server
$ mirage configure -t hvt
$ make
```

The result is a binary, `dist/ovpn-server.hvt`, which we will use later.

# VPN Configuration

The configuration needs to be stored in a block device. Use the provided
tool `openvpn-config-parser --server` to embed all external files into a
single file. You can use the configuration and keys as described in the
OpenVPN server chapter of this handbook.

```sh
$ dune exec -- openvpn-config-parser --server server.conf > server.conf.full
```

# Network configuration for the unikernel

All you need is a tap interface to run the unikernel on. You also need your
unikernel to be reachable from the outside (on the listening port), and be able
to communicate to the outside. There are multiple approaches to achieve this,
we will focus on setting up your firewall for this:

```sh
$ sysctl -w net.ipv4.ip_forward=1
$ ip route list default | cut -d' ' -f5
eth0

# allow the server to communicate to the outside
$ sudo iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -j MASQUERADE

# redirect port 1194 to the unikernel
$ sudo iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 1194 -j DNAT \
  --to-destination 10.0.0.2:1194
```

Setting up the network interface:

```sh
$ sudo ip tuntap add mode tap tap0
$ sudo ip link set dev tap0 up
$ sudo ip addr add tap0 10.0.0.1/24
```

We're all set now: the unikernel is allowed to communicate to the outside,
port 1194 is forwarded to the unikernel IP address, and a tap0 interface
exists where the host system has the IP address 10.0.0.1 configured.

# Launching MirageVPN-server

To launch the unikernel, you need a solo5 tender (that the Building section
already installed).

```sh
$ solo5-hvt --block:storage=server.conf.full --net:service=tap0 -- \
    dist/ovpn-server.hvt --ipv4=10.0.0.2/24 --ipv4-gateway=10.0.0.1
```

# Connecting client(s)

Now, clients can connect to the running server, either using OpenVPN,
MirageVPN, or any other implementation. The client configuration prepared in
the earlier chapter (alice) can be used for this. Execute the following on
your client machine:

```sh
$ openvpn alice.config
```

Now, all your traffic will be redirected through the VPN server.
