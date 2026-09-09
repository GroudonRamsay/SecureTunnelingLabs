## Post-Quantum and Hybrid Mechanisms Laboratory

The post-quantum readiness of secure tunneling protocols is primarily determined by how key establishment, authentication, and trust anchors are defined within their handshake mechanisms.

Post-quantum migration affects different parts of secure tunnels and secure channels in different ways. Key establishment focus is how peers derive shared secrets for a session. Authentication focus is how peers prove their identities, for example through certificates, signatures, or host keys.

A protocol may use hybrid post-quantum key establishment while still using classical RSA or ECDSA authentication. Conversely, a post-quantum signature provides authentication but does not by itself establish traffic keys.

Protocols that rely on classical public-key cryptography for key exchange or authentication are vulnerable to future quantum adversaries, particularly under the "harvest now, decrypt later" threat model.

In contrast, symmetric cryptography remains comparatively resilient, as quantum attacks provide only a quadratic advantage. This can be mitigated through the use of larger key sizes, such as the ones used in AES-256.

A central element of current post-quantum transition strategies is the use of hybrid key exchange, where session secrets are derived from both a classical Diffie–Hellman exchange and a post-quantum Key Encapsulation Mechanism (KEM). This approach provides confidentiality as long as at least one of the two underlying primitives remains secure.

## Module-Lattice-based KEMs

The Module-Lattice-Based Key Encapsulation Mechanism (ML-KEM), standardized by NIST as FIPS 203, is derived from the CRYSTALS-Kyber algorithm family.

ML-KEM is based on the hardness of the Module Learning With Errors (Module-LWE) problem, a generalization of the Learning With Errors (LWE) problem. LWE problems rely on the difficulty of solving systems of noisy linear equations over finite rings or modules.

Unlike classical public-key schemes such as RSA or elliptic-curve cryptography, whose security is tied to integer factorization or discrete logarithms, no efficient algorithm is currently known for solving Module-LWE on either classical or quantum computers. Module-LWE is believed to be resistant to Shor's algorithm, which renders RSA and ECC insecure in the presence of sufficiently large quantum computers.

In hybrid constructions, the ML-KEM-derived secret is combined with a classical ECDHE shared secret and expanded via HKDF, providing backward compatibility and supporting incremental deployment.

## Laboratory Topology

The topology of this laboratory consists of:

- Two Docker containers, PQ1 and PQ2, using the image ghcr.io/groudonramsay/ubuntu-tls:latest.

The topology is visible in Figure 1:

<figure markdown id="figure-1">
  ![Figure 1: PQ Topology](../images/PQTOPO.png)
  <figcaption>Figure 1: PQ Topology</figcaption>
</figure>

When adding the container to your templates, use 2 adapters and the following environment variables:

```bash
--cap-add NET_ADMIN
--cap-add NET_RAW
```

## Device Configuration

We will begin the configuration by adding the necessary IP addresses for our TLS devices to function properly.

```bash
PQ1:

ip addr add 10.10.10.2/24 dev eth0
ip link set eth0 up

PQ2:

ip addr add 10.10.10.1/24 dev eth0
ip link set eth0 up
```

Then, we will prepare PQ2 to act as the TLS server:

```bash
mkdir -p /lab/certs
cd /lab/certs

openssl genrsa -out server.key 2048

openssl req -new -x509 \
    -key server.key \
    -out server.crt \
    -days 365 \
    -subj "/CN=tls-server"
```

We can test the TLS connection with the following commands:

```bash
PQ2/Server

openssl s_server \
    -accept 4433 \
    -cert /lab/certs/server.crt \
    -key /lab/certs/server.key \
    -tls1_3 \
    -www

PQ1/Client:

openssl s_client \
    -connect 10.10.10.1:4433 \
    -tls1_3 \
    -groups X25519MLKEM768 \
```

You should see a successful connection, which includes the usage of a hybrid key exchange method, x25519 ML-KEM768.

!!! note "Because Docker images and containers are used for our devices, it is also possible to perform this laboratory in Containerlab if that is your preference."

With all configurations implemented and tested, we can now move forward to the [Experiments](experiments.md).
