## DTLS Laboratory

Datagram Transport Layer Security (DTLS) is the datagram-based analogue of TLS, designed to provide authentication, confidentiality, and integrity over the unreliable transport provided by UDP.

Unlike TLS, which operates over a reliable, ordered byte stream, DTLS preserves the essential properties of datagram communication, allowing applications to maintain low latency and tolerate packet loss without the overhead of TCP's connection management.

As a result, DTLS is widely adopted in scenarios such as VoIP, WebRTC media channels, online gaming, IoT protocols, and real-time telemetry, where strict ordering or retransmission could degrade application performance.

## DTLS Handshake

DTLS 1.3, defined in RFC 9147, introduces major improvements that align the protocol closely with TLS 1.3 while addressing datagram-specific challenges.

Like TLS 1.3, DTLS 1.3 provides a simplified handshake, shown in Figure 1, which reduces round-trip time and handshake complexity compared with previous versions, benefiting performance in high-latency and low-power environments.

<figure markdown id="figure-1">
  ![Figure 1: DTLS 1.3 Handshake](../images/DTLSHAND.png){width="500"}
  <figcaption>Figure 1: DTLS 1.3 Handshake</figcaption>
</figure>

## Laboratory Topology

The topology of this laboratory consists of:

- Three Docker containers, using the container in ghcr.io/groudonramsay/tls:latest, labeled as DTLSClient, DTLSServer and MitM.

The topology is visible in Figure 2:

<figure markdown id="figure-2">
  ![Figure 2: DTLS Topology](../images/DTLSTOPO.png){width="500"}
  <figcaption>Figure 2: DTLS Topology</figcaption>
</figure>

When adding the container to your templates, use 2 adapters and the following environment variables:

```bash
--cap-add NET_ADMIN
--cap-add NET_RAW
```

## Device Configuration

This configuration will follow the same approach used for TLS due to the similarities between the two protocols and the aspects of DTLS that we aim to experiment with.

To configure DTLSClient, use the following commands:

```bash
ip addr add 10.0.1.10/24 dev eth0
ip link set eth0 up
ip route add 10.0.2.0/24 via 10.0.1.1
```

To configure DTLSServer, use the following commands:

```bash
ip addr add 10.0.2.10/24 dev eth0
ip link set eth0 up
ip route add 10.0.1.0/24 via 10.0.2.1

mkdir -p ~/dtls
cd ~/dtls

openssl genrsa -out server.key 2048

openssl req -new -x509 \
    -key server.key \
    -out server.crt \
    -days 365 \
    -subj "/CN=10.0.2.10"
```

To configure MitM, use the following commands:

```bash
ip addr add 10.0.1.1/24 dev eth0
ip link set eth0 up
ip addr add 10.0.2.1/24 dev eth1
ip link set eth1 up
sysctl -w net.ipv4.ip_forward=1
```

These commands ensure connectivity between the two DTLS devices and prepare DTLSServer to host the server we will use.

To use DTLS, we will set up the server first using:

```bash
openssl s_server -dtls -cert server.crt -key server.key -accept 4433
```

With the server ready, place a Wireshark probe on the connection to ensure that the handshake occurs smoothly. Then, start the client using:

```bash
openssl s_client -dtls -connect 10.0.2.10:4433
```

Your client should successfully connect to the server, and you should see the handshake occur, as shown in Figure 3:

<figure markdown id="figure-3">
  ![Figure 3: DTLS Handshake](../images/DTLSLIVEHAND.png){width="750"}
  <figcaption>Figure 3: DTLS Handshake</figcaption>
</figure>>

!!! note "Because Docker images and containers are used for our devices, it is also possible to perform this laboratory in Containerlab if that is your preference."

With this successful connection, we will move on to the [Experiments](experiments.md) on DTLS to better understand the protocol and compare it with TLS.
