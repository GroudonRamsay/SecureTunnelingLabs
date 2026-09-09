## TLS Laboratory

Transport Layer Security (TLS) is one of the most widely used protocols for securing Internet communications, providing confidentiality, integrity, and authentication for applications such as HTTPS, APIs, email, and enterprise services.

TLS 1.2 served as the industry standard for many years, but its complexity, dependence on numerous cipher suites, and susceptibility to downgrade and mode-specific vulnerabilities motivated the development of a more robust and modern successor.

TLS 1.3 addresses these challenges through a reduced handshake, the removal of legacy algorithms, and stricter cryptographic guarantees, offering faster and more secure session establishment.

## TLS Handshake

TLS 1.2 relies on a multi-round-trip handshake that includes several optional messages, multiple cipher negotiation paths, and support for quantum-vulnerable primitives such as RSA key exchange and CBC-mode ciphers. Although flexible, this design introduced complexity and multiple attack surfaces.

TLS 1.3 significantly simplifies this model by mandating modern AEAD ciphers, enforcing perfect forward secrecy, and reducing the handshake to a typical 1-RTT exchange.

We can observe a comparison between the TLS 1.2 and TLS 1.3 handshakes in Figure 1. We can see several improvements between the two versions, such as the reduced number of round trips required for session establishment through the front-loading of key-exchange material into the ClientHello message. This allows TLS 1.3 to start a session faster and encrypt messages sooner.

<figure markdown id="figure-1">
  ![Figure 1: TLS 1.2 and 1.3 Handshakes](../images/TLSHAND.png){width="600"}
  <figcaption>Figure 1: TLS 1.2 and 1.3 Handshakes</figcaption>
</figure>

A major innovation in TLS 1.3 is its support for 0-RTT data during session resumption, enabling clients to send application data immediately after initiating a new connection. While 0-RTT data is vulnerable to replay and requires careful use, it provides substantial performance gains for repeated connections and is an important feature for low-latency secure communication.

## Laboratory Topology

The topology of this laboratory consists of:

- Three docker containers using the container ghcr.io/groudonramsay/tls:latest, labeled as TLSClient, TLSServer and MitM.

The topology is visible in Figure 2:

<figure markdown id="figure-2">
  ![Figure 2: TLS Topology](../images/TLSTOPO.png){width="600"}
  <figcaption>Figure 2: TLS Topology</figcaption>
</figure>

When adding the container to your templates, use two adapters and the following environment variables:

```bash
--cap-add NET_ADMIN
--cap-add NET_RAW
```

## Device Configuration

We will now configure our three devices. TLSClient and TLSServer will have their IP addresses, a route through the MitM, and the keys and certificates needed for TLS to work properly.

For TLSClient, use the following commands:

```bash
ip addr add 10.0.1.10/24 dev eth0
ip link set eth0 up
ip route add 10.0.2.0/24 via 10.0.1.1
```

For TLSServer, use the following commands to set the IP address and generate the key and certificate that the server will use:

```bash
ip addr add 10.0.2.10/24 dev eth0
ip link set eth0 up
ip route add 10.0.1.0/24 via 10.0.2.1

mkdir -p ~/tls
cd ~/tls

openssl genrsa -out server.key 2048

openssl req -new -x509 \
    -key server.key \
    -out server.crt \
    -days 365 \
    -subj "/CN=10.0.2.10"
```

For MitM, use the following commands to route traffic:

```bash
ip addr add 10.0.1.1/24 dev eth0
ip link set eth0 up
ip addr add 10.0.2.1/24 dev eth1
ip link set eth1 up
sysctl -w net.ipv4.ip_forward=1
```

With these commands, you should be able to ping between the client and server without any problems.

Now let's start a server and client to test whether TLS is working properly. We will begin by creating the server on TLSServer using the following command:

```bash
openssl s_server \
    -accept 4433 \
    -cert server.crt \
    -key server.key \
    -www \
    -state
```

This will activate a server listening for connections on port 4433. Then, connect a client to it using the following commands on TLSClient:

```bash
openssl s_client \
    -connect 10.0.2.10:4433 \
    -state \
    -quiet
```

You should see a successful connection. If you have Wireshark open, you will see the handshake occurring, followed by encrypted TLS traffic. You will also see TCP packets that are used to maintain the TCP connection.

!!! note "Because Docker images and containers are used for our devices, it is also possible to perform this laboratory in Containerlab if that is your preference."

With this successful connection, we will now begin the experiments on TLS in [Experiments](experiments.md).
