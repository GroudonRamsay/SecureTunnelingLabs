## SSH Laboratory

Secure Shell (SSH) is a protocol for secure remote access, encrypted command execution, and file transfer. Standardized by the IETF in RFCs 4251 through 4254, SSH provides confidentiality, integrity, and peer authentication through a flexible layered architecture.

Operating at the application layer and running over TCP, SSH offers user-centric access control, key-based authentication, and session multiplexing. These properties make it well suited for administrative access, secure management of remote hosts, and port forwarding.

SSH is commonly used to create local, remote, or dynamic tunnels that protect otherwise insecure application flows. Typical use cases include secure system administration, tunneling database connections across untrusted networks, and enabling secure file transfer through SFTP.

## SSH Architecture and Handshake

SSH is divided into three logical layers: the transport layer, the authentication layer, and the connection layer.

The transport layer, described in RFC 4253, establishes a secure channel, with its handshake shown in Figure 1. It provides confidentiality and integrity for subsequent protocol layers while negotiating encryption algorithms, such as AES-GCM or ChaCha20-Poly1305, MACs, compression, and key-exchange algorithms such as Curve25519 or ECDH.

The authentication layer supports mechanisms such as public-key authentication using RSA, Ed25519, or ECDSA, as well as password authentication. The connection layer multiplexes multiple logical channels, such as shell, exec, subsystem, and port forwarding, over a single transport connection, as defined by RFC 4254.

<figure markdown id="figure-1">
  ![Figure 1: SSH Handshake](../images/SSHHAND.png){width="400"}
  <figcaption>Figure 1: SSH Handshake</figcaption>
</figure>

## Laboratory Topology

The topology of this laboratory consists of:

- Three Docker containers using the image ghcr.io/groudonramsay/ubuntu-ssh:latest, which are SSHClient, SSHServer 1 and SSHServer 2.
- One Docker container acting as a MitM, using the image ghcr.io/groudonramsay/ipsec-replay:latest.

The topology is visible in Figure 2:

<figure markdown id="figure-2">
  ![Figure 2: SSH Topology](../images/SSHTOPO.png){width="500"}
  <figcaption>Figure 2: SSH Topology</figcaption>
</figure>

When adding the container to your templates, use 2 adapters and the following environment variables:

```bash
--cap-add NET_ADMIN
--cap-add NET_RAW
```

Before beginning the configuration, modify the MitM machine to have three adapters.

## Device Configuration

We will begin the configuration by setting up the IP addresses and routes necessary for communication between all devices.

Use the following commands to configure SSHClient:

```bash
ip addr add 10.0.1.10/24 dev eth0
ip link set eth0 up
ip route add 10.0.2.0/24 via 10.0.1.1
ip route add 10.0.3.0/24 via 10.0.1.1
```

Use the following commands to configure SSHServer1:

```bash
ip addr add 10.0.2.10/24 dev eth0
ip link set eth0 up
ip route add 10.0.1.0/24 via 10.0.2.1
ip route add 10.0.3.0/24 via 10.0.2.1
```

Use the following commands to configure SSHServer2:

```bash
ip addr add 10.0.3.10/24 dev eth0
ip link set eth0 up
ip route add 10.0.1.0/24 via 10.0.3.1
ip route add 10.0.2.0/24 via 10.0.3.1
```

Use the following commands to configure MitM:

```bash
ip addr add 10.0.1.1/24 dev eth0
ip link set eth0 up
ip addr add 10.0.2.1/24 dev eth1
ip link set eth1 up
ip addr add 10.0.3.1/24 dev eth2
ip link set eth2 up
sysctl -w net.ipv4.ip_forward=1
```

You can test the configuration by pinging from SSHClient to both servers and ensuring that they respond.

With the network configured and ready to go, we will now configure the two SSH servers so that we can use them.

We will begin with SSHServer1, using the following commands to set up a user authorized to log in using a password and to configure the server itself:

```bash
useradd -m -s /bin/bash sshuser
passwd sshuser
ssh1234
```

These commands create the user named sshuser and assign it the password ssh1234, which we will use from the client to log in to the server.

Now, use the following commands to set up the server:

```bash
mkdir -p /run/sshd
chmod 755 /run/sshd
/usr/sbin/sshd
```

This creates the necessary directory and permissions to run the SSH server and starts the server. We can verify that it is listening for connections with:

```bash
ss -lntp | grep ':22'
```

For SSHServer2, we will configure public-key-based login. To do so, use the following commands on SSHServer2:

```bash
useradd -m -s /bin/bash sshuser
mkdir -p /run/sshd
chmod 755 /run/sshd
```

Then, access the configuration file with:

```bash
nano /etc/ssh/sshd_config
```

Change the line #PubkeyAuthentication no to PubkeyAuthentication yes, and change the line #PasswordAuthentication yes to PasswordAuthentication no.

This changes the server's expected authentication method from password authentication to public-key authentication.

Now, we will generate a key pair on SSHClient and copy the public key to SSHServer2. First, create the key with:

```bash
ssh-keygen -t ed25519
```

Simply press Enter three times to complete the process.

Then, copy the public key using the following command:

```bash
cat ~/.ssh/id_ed25519.pub
```

Ensure that you copy the entire line. Then, back on SSHServer2, create a folder to store the authorized keys and paste the copied key into the file:

```bash
mkdir -p /home/sshuser/.ssh
nano /home/sshuser/.ssh/authorized_keys
```

Then, secure the file with:

```bash
chown -R sshuser:sshuser /home/sshuser/.ssh
chmod 700 /home/sshuser/.ssh
chmod 600 /home/sshuser/.ssh/authorized_keys
```

Now you can start SSHServer 2 and ensure that it is working with:

```bash
/usr/sbin/sshd
ss -lntp | grep ':22'
```

Finally, try logging in with:

```bash
ssh sshuser@10.0.3.10
```

After accepting the server's host key, you will see that no password is required because we used our public key to authenticate.

!!! note "Because Docker images and containers are used for our devices, it is also possible to perform this laboratory in Containerlab if that is your preference."

With all these setups, your client and servers should now be ready for the following [Experiments](experiments.md).
