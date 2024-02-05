Before we can start using our client, we need to have a server available. This
short tutorial will show you how to set up a simple OpenVPN server with the aim
of testing our client: we strongly advise you **not** to consider our
configuration as 'production-ready'. We'll be using Debian 12 and OpenVPN 2.6.3.

## Installation & Generation

So you probably have SSH access on your server. Let's connect to the server and
install OpenVPN.
```sh
$ export OPENVPN_IP=<ipv4> # Must be set!
$ ssh root@$OPENVPN_IP
$ apt update
$ apt upgrade
$ apt install openvpn ufw
```

We then need to create a [Public Key Infrastructure][pki] so that we can manage
who can connect via our VPN server. We'll use `easy-rsa` for this. The aim of a
PKI is to be able to add new users without reloading our server (among other
things). We are going to create an "authority" and create our users' keys via
this authority. In this way, our OpenVPN server will be able to authenticate
users not by their keys but by the fact that the keys were created via our
authority.
```sh
$ mkdir easy-rsa
$ ln -s /usr/share/easy-rsa/* ~/easy-rsa/
$ chmod 700 easy-rsa
$ cd easy-rsa
$ cat >vars<<EOF
set_var EASYRSA_ALGO "ec"
set_var EASYRSA_DIGEST "sha512"
EOF
$ ./easyrsa init-pki
$ ./easyrsa build-ca nopass
$ ./easyrsa build-dh
$ ./easyrsa gen-req server nopass
$ ./easyrsa sign server server
```

We then need to copy the elements we need to set up our server and generate the
final material required.
```sh
$ cp pki/issued/server.crt pki/ca.crt pkg/dh.pem /etc/openvpn/server/
$ cp pki/private/server.key /etc/openvpn/server
$ cd /etc/openvpn/server
$ openvpn --genkey secret ta.key
```

Now we need to create a client identity[^note].
```sh
$ cd ~/easyrsa
$ ./easyrsa gen-req alice nopass
$ ./easyrsa sign-req client alice
```

The two files we are ultimately interested in for our client are the newly
created `alice.crt` and `ta.key`. The first is used for authentication (to prove
your identity to the server). The second encrypts the connection between you and
the server.

[^note]: The example of generating the materials needed for the server here is
very simplified. More robust methods (such as generating the server and client
keys elsewhere than on the target server) are recommended. As a reminder, this
description does **not** produce a "production-ready" OpenVPN server.

## Configuration

We can now move on to configuring the server. This consists of a simple file and
setting up a private network. The configuration file simply describes where the
materials needed to launch the server are located.

```sh
$ cat >/etc/openvpn/server/server.conf<<EOF
proto tcp
port 1194
dev tun
ca ca.crt
server server.crt
key server.key
dh dh.pem
topology subnet
server 10.8.0.0 255.255.255.0
ifconfig-pool-persist ipp.txt
keepalive 10 30
tls-auth ta.key 0
persist-key
persist-tun
user nobody
group nogroup
EOF
```

There's a lot to be said for this configuration. The first is the use of
`tls-auth` to encrypt our control channel. There are other ways of doing this,
which we'll describe later, but let's start by having a working server. TCP is
also used as a protocol (instead of UDP). We will also assign a specific IP for
our `alice` client:

```sh
$ echo "alice,10.8.0.2," >> /etc/openvpn/server/ipp.txt
```

Finally, we configure our network so that our customers can communicate with
Internet:

```sh
$ sysctl -w net.ipv4.ip_forward=1
$ ip route list default | cut -d' ' -f5
eth0
$ cat >>/etc/ufw/before.rules<<EOF
*nat
:POSTROUTING ACCEPT [0:0]
-A POSTROUTING -s 10.8.0.0/8 -o eth0 -j MASQUERADE
COMMIT
EOF
$ sed 's/DEFAULT_FORWARD_POLICY="DROP"/DEFAULT_FORWARD_POLICY="ACCEPT"/' \
  -i /etc/default/ufw
$ ufw allow 1194/tcp
$ ufw allow OpenSSH
$ systemctl -f enable openvpn-server@server.service
$ systemctl start openvpn-server@server.service
```

And here's a working OpenVPN server for testing our client!
