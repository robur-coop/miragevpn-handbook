The MirageVPN server is a unikernel which listens on a port for client
connections, and provides Internet connectivity (with NAT) for all
authenticated clients. A single network interface is used for both client
connections and providing connectivity.

The unikernel will encrypt all packets received from the Internet and send
them to the respective client. All packets received from a client will be
decrypted and forwarded to the Internet. If "client-to-client" is enabled
in the configuration, packets from one client which destination is another
client will be forwarded by the server.

# Configuration

The configuration needs to be stored in a block device. Use the provided
tool `openvpn-config-parser` to embed all external files into a single file.

# Network configuration

All you need is a tap interface to run the unikernel on. 
