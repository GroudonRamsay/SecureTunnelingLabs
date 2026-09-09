## Laboratory Experiments

We will now begin our DTLS experiments, with the aim of deepening our understanding of the protocol and better understanding what makes it different from and similar to TLS. We will compare the handshakes and transport-layer differences of the two protocols, examine how their packets are structured, test how DTLS handles the loss, duplication, and reordering that we subjected TLS to, examine how DTLS handles the fragmentation inherent to datagram-based transport, and test its replay protection.

## DTLS Handshake vs TLS Handshake

For our first experiment, we will compare the DTLS handshake with what we observed in TLS.

An important note is that the version of OpenSSL used in this laboratory only provides DTLS 1.2, so we will compare DTLS 1.2 with TLS 1.2.

If you tested DTLS in the Overview section, you should already have a complete handshake to analyze. If not, start a Wireshark capture between the two DTLS peers and connect the client to the server. The handshake should look identical or similar to the following:

<figure markdown id="figure-1">
  ![Figure 1: DTLS Handshake](../images/DTLSLIVEHAND.png){width="750"}
  <figcaption>Figure 1: DTLS Handshake</figcaption>
</figure>

This handshake bears some similarity to the TLS 1.2 handshake, but it also has several differences.

Although the overall structure is relatively similar, we can immediately identify one major difference: there is no transport-layer session establishment. Since DTLS uses UDP, which is connectionless, the first packets are already part of the DTLS handshake.

The first two packets are related to another DTLS feature, the stateless cookie. The cookie allows the server to verify that the client can receive and respond to packets at its claimed network address before the server allocates significant resources to the handshake, helping protect against resource-exhaustion attacks.

We can see the cookie being sent by the server in Figure 2, and in Figure 3 we can see the second ClientHello with the cookie added to the request.

<figure markdown id="figure-2">
  ![Figure 2: DTLS Server sends a Cookie](../images/DTLSCOOKIE1.png){width="600"}
  <figcaption>Figure 2: DTLS Server sends a Cookie</figcaption>
</figure>

<figure markdown id="figure-3">
  ![Figure 3: DTLS Client sends request with Cookie](../images/DTLSCOOKIE2.png){width="600"}
  <figcaption>Figure 3: DTLS Client sends request with Cookie</figcaption>
</figure>

We can now also examine what the client sends in its ClientHello message in Figure 3. We can see similarities with TLS, including the session ID, random value, available cipher suites, and other negotiated extensions.

However, we can also identify several differences, including the epoch and record sequence number, which help identify the cryptographic state and detect duplicate records; the message sequence number, fragment offset, and fragment length, which allow DTLS to handle handshake-message reordering and fragmentation, and the cookie sent by the server to verify that the client can receive and respond to packets at its claimed address.

Moving forward, we can see the ServerHello and notice another consequence of using UDP: the handshake message can be fragmented across multiple DTLS records because a DTLS record must fit within a single datagram.

<figure markdown id="figure-4">
  ![Figure 4: DTLS fragmented Server Hello](../images/DTLSSERVERHELLO.png){width="750"}
  <figcaption>Figure 4: DTLS fragmented Server Hello</figcaption>
</figure>

This is expected when a handshake message is too large to fit into a single datagram. DTLS provides mechanisms to reassemble fragmented handshake messages and retransmit lost handshake messages when necessary.

<figure markdown id="figure-5">
  ![Figure 5: DTLS Server Hello](../images/DTLSSERVERHELLO2.png){width="600"}
  <figcaption>Figure 5: DTLS Server Hello</figcaption>
</figure>

Looking at Figure 5, we can see that the overall structure remains similar to TLS 1.2, containing elements such as the session ID, random value, selected cipher suite, and certificate, which is fragmented in this case.

We can also identify the DTLS-specific fields, including the epoch and record sequence number, as well as the message sequence number, fragment offset, and fragment length. These fields allow the receiver to identify the cryptographic state of records, detect duplicate records, and reassemble handshake messages that arrive out of order or in multiple fragments.

<figure markdown id="figure-6">
  ![Figure 6: DTLS Server Hello Key Exchange and End](../images/DTLSSERVERHELLO3.png){width="600"}
  <figcaption>Figure 6: DTLS Server Hello Key Exchange and End</figcaption>
</figure>

After reassembling the certificate, we can see the server complete its handshake messages by sending its key-exchange information and ServerHelloDone message, together with the DTLS-specific fields used to provide datagram-oriented handshake handling.

<figure markdown id="figure-7">
  ![Figure 7: DTLS Client Response](../images/DTLSCLIENTRESPONSE.png){width="600"}
  <figcaption>Figure 7: DTLS Client Response</figcaption>
</figure>

Finally, in Figure 7, we can see the client responding to the server, completing the key exchange, sending ChangeCipherSpec, and beginning encrypted communication. This changes the record epoch from 0 to 1, indicating a transition to the next cryptographic state. The server then responds and begins encrypted communication as well.

From this experiment, we were able to compare the DTLS 1.2 and TLS 1.2 handshakes and see that their main elements remain similar, with DTLS adding mechanisms to handle the unreliability and lack of ordering provided by UDP. These mechanisms allow DTLS to establish a secure communication channel while retaining the datagram-based characteristics of UDP.

## DTLS tolerance to loss, duplication and reordering

For this experiment, we will test how DTLS handles the same traffic problems we tested TLS against. We will analyze DTLS's response to packet loss, duplication, and reordering, observing how its mechanisms help the handshake remain functional despite these conditions.

We will begin by testing its tolerance to packet loss. To do so, access the MitM machine and use the following command to introduce packet loss:

```bash
tc qdisc add dev eth1 root netem loss 80%
```

This command configures the netem queue on eth1 to drop 80% of packets transmitted through that interface. This value is intentionally high to ensure that packet loss occurs during the handshake and make its effects easier to observe. We will now restart the client and see how the handshake is processed under these conditions.

<figure markdown id="figure-8">
  ![Figure 8: DTLS Loss Response Part 1](../images/DTLSLOSS1.png){width="750"}
  <figcaption>Figure 8: DTLS Loss Response Part 1</figcaption>
</figure>

<figure markdown id="figure-9">
  ![Figure 9: DTLS Loss Response Part 2](../images/DTLSLOSS2.png){width="750"}
  <figcaption>Figure 9: DTLS Loss Response Part 2</figcaption>
</figure>

As we can see from Figures 8 and 9, the high loss rate leads to a high retransmission rate during the DTLS handshake. When a handshake message is lost and no response is received within the expected time, DTLS retransmits the message in an attempt to complete the handshake. Even with this unusually high loss rate, DTLS was able to complete the handshake and establish a session, although it required many more messages and significantly more time to do so.

Now we will test DTLS's handling of duplicated packets. To do so, first clear the previous rule with:

```bash
tc qdisc del dev eth1 root
```

Then apply the duplication rule with:

```bash
tc qdisc add dev eth1 root netem \
    duplicate 50%
```

This gives each packet transmitted through the interface a 50% chance of being duplicated. Restart the server and client, and let's see what DTLS does in a capture between MitM and DTLSServer.

<figure markdown id="figure-10">
  ![Figure 10: DTLS Duplication Response](../images/DTLSDUPE.png){width="750"}
  <figcaption>Figure 10: DTLS Duplication Response</figcaption>
</figure>

As we can see from Figure 10, despite the duplication of some packets during the handshake, DTLS can identify duplicate records using their epoch and sequence number and discard them. Handshake messages also have their own message sequence numbers, allowing DTLS to identify and correctly process duplicate or out-of-order handshake messages.

For the final test of this section, we will reorder packets arriving at the server and see whether this changes the handshake process.

First, clear the previous command:

```bash
tc qdisc del dev eth1 root
```

Then, apply the new reordering rule with:

```bash
tc qdisc add dev eth1 root netem \
    delay 100ms \
    reorder 50% 50%
```

Restart the connection and search the captures for any anomalies. In this particular case, we could not detect any significant anomalies in Wireshark, and the handshake occurred without noticeable problems. This demonstrates how DTLS can handle reordered handshake messages using message sequence numbers and can identify the ordering and boundaries of fragments using the fragment offset and fragment length.

## DTLS Fragmentation Handling

For this experiment, we will look more closely at how DTLS handles fragmentation and how changing the configured MTU affects the number and size of fragments sent.

From the previous experiments, we could already see that DTLS implements mechanisms to handle the fragmentation that can occur when using datagram-based transport. Through the use of message sequence numbers, DTLS can keep track of which part of a message each fragment belongs to. The fragment offset and fragment length then identify the position and size of each fragment, allowing the receiver to reassemble the original handshake message when all of its fragments have arrived.

<figure markdown id="figure-11">
  ![Figure 11: DTLS Fragmentation](../images/DTLSFRAG.png){width="600"}
  <figcaption>Figure 11: DTLS Fragmentation</figcaption>
</figure>

To change the MTU, or Maximum Transmission Unit, we add the -mtu option to the server and client commands. A higher MTU allows larger records to be sent and can reduce the number of fragments required, while a lower MTU can increase the number of fragments and cause more handshake messages to be fragmented.

An example using an MTU of 500 is shown below:

```bash
openssl s_server \
    -dtls1_2 \
    -accept 4444 \
    -cert server.crt \
    -key server.key \
    -mtu 500 \
    -timeout \
    -msg
```

```bash
openssl s_client \
    -dtls1_2 \
    -connect 10.0.2.10:4444 \
    -mtu 500 \
    -timeout \
    -msg
```

If we look at Wireshark with this MTU, we can see a reduction in the number of fragmented messages and in the number of fragments used for messages that are still fragmented, such as the certificate, as shown in Figure 12.

<figure markdown id="figure-12">
  ![Figure 12: DTLS Fragmentation reduced](../images/DTLSMTUHIGH.png){width="750"}
  <figcaption>Figure 12: DTLS Fragmentation reduced</figcaption>
</figure>

When we use the lower MTU value of 256, we return to a configuration with more fragmented messages and more fragments, as shown in Figure 13.

<figure markdown id="figure-13">
  ![Figure 13: DTLS Fragmentation increased](../images/DTLSMTULOW.png){width="750"}
  <figcaption>Figure 13: DTLS Fragmentation increased</figcaption>
</figure>

From this experiment, we now have a better understanding of how DTLS handles fragmentation and how we can influence fragmentation by configuring the MTU used by the server and client.

## DTLS Replay Protection and Attack

For our final experiment, we will test the replay protection provided by DTLS. To do this, we will use a new MitM device, the one used in the IPsec labs, ghcr.io-groudonramsay-ipsec-replay:latest, which will replace the MitM device we have been using.

We will use this machine for its packet capture and replay capabilities, using tcpdump and tcpreplay. First, configure the new machine with the necessary settings for traffic to flow through it:

```bash
ip addr add 10.0.1.1/24 dev eth0
ip link set eth0 up
ip addr add 10.0.2.1/24 dev eth1
ip link set eth1 up
sysctl -w net.ipv4.ip_forward=1
```

With this configuration in place, we will begin capturing traffic with the MitM device using the command:

```bash
tcpdump -ni eth0 -w /pcaps/replay-capture.pcap
```

This will capture traffic received on eth0, including traffic sent by the client. Now, restart the client and server and send some messages through the DTLS tunnel. After capturing the handshake and some application data, stop the capture on MitM. We now have a .pcap file containing several packets that we can replay to test DTLS's replay protection.

To execute the replay attack, use the command:

```bash
tcpreplay -i eth1 /pcaps/replay-capture.pcap
```

<figure markdown id="figure-14">
  ![Figure 14: DTLS Replay Attack](../images/DTLSREPLAYSHARK.png){width="600"}
  <figcaption>Figure 14: DTLS Replay Attack</figcaption>
</figure>

In Figure 14, we can see the replayed messages between MitM and DTLSServer, including sequence numbers and epochs from previously transmitted records. Because these records have already been processed, DTLS's replay protection identifies them as duplicates or records that fall outside the acceptable receive window and discards them, shown in Figure 15.

As a result, we should see no corresponding change in the server's application state. This demonstrates that simply capturing and retransmitting previously valid DTLS records does not allow an attacker to replay them successfully.

<figure markdown id="figure-15">
  ![Figure 15: DTLS Replay Attack unsuccessful](../images/DTLSREPLAYTERM.png){width="600"}
  <figcaption>Figure 15: DTLS Replay Attack unsuccessful</figcaption>
</figure>

With these experiments, we hope that your understanding of DTLS's inner workings has improved. We have studied how these tunnels are established, how DTLS uses UDP, and how it differs from its TLS and TCP counterparts. We have also tested DTLS in practice through packet loss, duplication, reordering, fragmentation, and replay attacks.

We hope to see you in the next lab: SSH!
