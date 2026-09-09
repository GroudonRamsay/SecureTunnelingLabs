## ANYsec Laboratory

ANYSec is a Nokia tunnelling technology that provides low-latency and line-rate native encryption for any transport (IP, MPLS, segment routing, Ethernet or VLAN), on any service, at any time and for any load conditions without impacting performance. It is scalable, flexible and uses quantum-safe encryption.

Based on MACSec standards as the foundation, it introduces the flexibility to offset the authentication and encryption to allow L2, L2.5 and L3 encryption.

In this laboratory, we will create a Containerlab topology that uses both ANYsec and MACsec to compare their functionality, similarities, differences, and available features.

## Laboratory Topology

The laboratory topology is based on the topology from Nokia's public laboratory, [SR OS FP5 ANYSec and MACSec Demo](https://github.com/srl-labs/sros-anysec-macsec-lab/tree/main), although with significant simplifications and modifications. The focus of our laboratory will be on the tunneling aspects of ANYsec and its comparison with similar protocols such as MACsec and IPsec. If you want to further investigate ANYsec's functionality beyond this laboratory, we strongly recommend Nokia's public laboratories.

<figure markdown id="figure-1">
  ![Figure 1: Laboratory Topology](../images/ANYTOPO.png)
  <figcaption>Figure 1: Laboratory Topology</figcaption>
</figure>

Our laboratory topology includes:

- Two linux hosts at the edges that act as the users/sources of three distinct services.
- Two Consumer Edge routers, named CE5 and CE6, which receive packets from different services and send them using MACsec to the following routers.
- Two Provider Edge routers, named PE1 and PE2, which receive the MACsec packets from the consumers and then switch them for ANYsec packets to send through the network to the other provider.
- Two routers acting as network between Provider Edges, P3 and P4.

With this topology, we will be able to observe both MACsec and ANYsec in operation and examine some of ANYsec's most important tunneling and security features, such as MKA over UDP, end-to-end encryption, tunnel encryption, and service encryption.

## Router Configuration

To begin the configuration, create a folder for this laboratory named:

```bash
anysec-main-lab
```

Then, create the topology file named:

```bash
anysec-main-lab.clab.yaml
```

!!!warning "Do not forget to create the configs folder and the configuration files for all six routers."

For the topology file, use the following:

```yaml
name: anysec-main-lab
prefix: "m"

mgmt:
  network: anysec-main
  ipv4-subnet: 172.100.100.0/24

topology:
  kinds:
    nokia_srsim:
      type: sr-1x-48d
      image: localhost/nokia/srsim:25.10.R3
      license: (Your License File).txt

    linux:
      image: ghcr.io/srl-labs/network-multitool

  nodes:
    pe1:
      kind: nokia_srsim
      startup-config: configs/pe1.partial.cfg
      components:
        - slot: A
        - slot: 1

    pe2:
      kind: nokia_srsim
      startup-config: configs/pe2.partial.cfg
      components:
        - slot: A
        - slot: 1

    p3:
      kind: nokia_srsim
      type: sr-1
      startup-config: configs/p3.partial.cfg
      env:
        NOKIA_SROS_MDA_1: me12-100gb-qsfp28

    p4:
      kind: nokia_srsim
      type: sr-1
      startup-config: configs/p4.partial.cfg
      env:
        NOKIA_SROS_MDA_1: me12-100gb-qsfp28

    ce5:
      kind: nokia_srsim
      startup-config: configs/ce5.partial.cfg
      components:
        - slot: A
        - slot: 1

    ce6:
      kind: nokia_srsim
      startup-config: configs/ce6.partial.cfg
      components:
        - slot: A
        - slot: 1

    host1:
      kind: linux
      mgmt-ipv4: 172.100.100.31
      exec:
        - ip -6 address add 2002::192:168:51:7/96 dev eth1
        - ip address add 192.168.51.7/24 dev eth1
        - ip address add 192.168.52.7/24 dev eth2
        - ip address add 192.168.53.7/24 dev eth3
        - ip route add 192.168.63.0/24 via 192.168.53.1
      group: server

    host2:
      kind: linux
      mgmt-ipv4: 172.100.100.32
      exec:
        - ip -6 address add 2002::192:168:51:8/96 dev eth1
        - ip address add 192.168.51.8/24 dev eth1
        - ip address add 192.168.52.8/24 dev eth2
        - ip address add 192.168.63.8/24 dev eth3
        - ip route add 192.168.53.0/24 via 192.168.63.1
      group: server

  links:
    - endpoints: ["ce5:1/1/c7/1", "pe1:1/1/c3/1"]
    - endpoints: ["ce6:1/1/c7/1", "pe2:1/1/c3/1"]
    - endpoints: ["pe1:1/1/c1/1", "p3:1/1/c2/1"]
    - endpoints: ["pe1:1/1/c2/1", "p4:1/1/c2/1"]
    - endpoints: ["pe2:1/1/c1/1", "p3:1/1/c3/1"]
    - endpoints: ["pe2:1/1/c2/1", "p4:1/1/c3/1"]
    - endpoints: ["p3:1/1/c1/1", "p4:1/1/c1/1"]
    # Host1
    - endpoints: ["host1:eth1", "ce5:1/1/c1/1"]
    - endpoints: ["host1:eth2", "ce5:1/1/c2/1"]
    - endpoints: ["host1:eth3", "ce5:1/1/c3/1"]
    # Host2
    - endpoints: ["host2:eth1", "ce6:1/1/c1/1"]
    - endpoints: ["host2:eth2", "ce6:1/1/c2/1"]
    - endpoints: ["host2:eth3", "ce6:1/1/c3/1"]
```

!!!warning "Remember to change the path and names of your router image and license file."

You might notice that the hosts have three IP addresses. This is done to simulate the presence of the three different services we will be using, with each IP address assigned to a different interface connected to the Consumer Edge routers.

We will now begin configuring the routers. We start with the configuration for P3 and P4 because there is nothing related to tunneling, only routing-related configuration.

The configuration for P3 is:

```srl
/configure card 1 admin-state enable
/configure card 1 card-type iom-1
/configure card 1 mda 1 mda-type me12-100gb-qsfp28
/configure port 1/1/c1 admin-state enable
/configure port 1/1/c1 connector breakout c1-100g
/configure port 1/1/c1/1 admin-state enable
/configure port 1/1/c1/1 description "to_P4"
/configure port 1/1/c1/1 ethernet mode hybrid
/configure port 1/1/c1/1 ethernet encap-type dot1q
/configure port 1/1/c2 admin-state enable
/configure port 1/1/c2 connector breakout c1-100g
/configure port 1/1/c2/1 admin-state enable
/configure port 1/1/c2/1 description "to_PE1"
/configure port 1/1/c2/1 ethernet mode hybrid
/configure port 1/1/c2/1 ethernet encap-type dot1q
/configure port 1/1/c3 admin-state enable
/configure port 1/1/c3 connector breakout c1-100g
/configure port 1/1/c3/1 admin-state enable
/configure port 1/1/c3/1 description "to_PE2"
/configure port 1/1/c3/1 ethernet mode hybrid
/configure port 1/1/c3/1 ethernet encap-type dot1q
/configure router "Base" autonomous-system 65000
/configure router "Base" ecmp 2
/configure router "Base" router-id 10.0.0.3
/configure router "Base" interface "system" ipv4 primary address 10.0.0.3
/configure router "Base" interface "system" ipv4 primary prefix-length 32
/configure router "Base" interface "to_P4" description "to_P4"
/configure router "Base" interface "to_P4" port 1/1/c1/1:2
/configure router "Base" interface "to_P4" ipv4 primary address 10.3.4.3
/configure router "Base" interface "to_P4" ipv4 primary prefix-length 24
/configure router "Base" interface "to_PE1" description "to_PE1"
/configure router "Base" interface "to_PE1" port 1/1/c2/1:2
/configure router "Base" interface "to_PE1" ipv4 primary address 10.1.3.3
/configure router "Base" interface "to_PE1" ipv4 primary prefix-length 24
/configure router "Base" interface "to_PE2" description "to_PE2"
/configure router "Base" interface "to_PE2" port 1/1/c3/1:2
/configure router "Base" interface "to_PE2" ipv4 primary address 10.2.3.3
/configure router "Base" interface "to_PE2" ipv4 primary prefix-length 24
/configure router "Base" mpls-labels static-label-range 1968
/configure router "Base" mpls-labels sr-labels start 16000
/configure router "Base" mpls-labels sr-labels end 24000
/configure router "Base" mpls-labels reserved-label-block "Anysec" start-label 2000
/configure router "Base" mpls-labels reserved-label-block "Anysec" end-label 5999
/configure router "Base" bgp rapid-withdrawal true
/configure router "Base" bgp family ipv4 true
/configure router "Base" bgp family vpn-ipv4 true
/configure router "Base" bgp family ipv6 true
/configure router "Base" bgp family vpn-ipv6 true
/configure router "Base" bgp family l2-vpn true
/configure router "Base" bgp family route-target true
/configure router "Base" bgp family evpn true
/configure router "Base" bgp family label-ipv4 true
/configure router "Base" bgp family label-ipv6 true
/configure router "Base" bgp family sr-policy-ipv4 true
/configure router "Base" bgp family sr-policy-ipv6 true
/configure router "Base" bgp cluster cluster-id 10.0.0.3
/configure router "Base" bgp local-as as-number 65000
/configure router "Base" bgp rapid-update vpn-ipv4 true
/configure router "Base" bgp rapid-update vpn-ipv6 true
/configure router "Base" bgp rapid-update evpn true
/configure router "Base" bgp rapid-update label-ipv4 true
/configure router "Base" bgp rapid-update label-ipv6 true
/configure router "Base" bgp multipath max-paths 2
/configure router "Base" bgp group "ibgp" keepalive 5
/configure router "Base" bgp group "ibgp" min-route-advertisement 5
/configure router "Base" bgp group "ibgp" type internal
/configure router "Base" bgp group "ibgp" peer-as 65000
/configure router "Base" bgp group "ibgp" local-address 10.0.0.3
/configure router "Base" bgp group "ibgp" peer-ip-tracking true
/configure router "Base" bgp neighbor "10.0.0.1" description "PE1"
/configure router "Base" bgp neighbor "10.0.0.1" group "ibgp"
/configure router "Base" bgp neighbor "10.0.0.2" description "PE2"
/configure router "Base" bgp neighbor "10.0.0.2" group "ibgp"
/configure router "Base" bgp neighbor "10.0.0.4" description "P4"
/configure router "Base" bgp neighbor "10.0.0.4" group "ibgp"
/configure router "Base" isis 0 admin-state enable
/configure router "Base" isis 0 advertise-router-capability as
/configure router "Base" isis 0 iid-tlv true
/configure router "Base" isis 0 level-capability 2
/configure router "Base" isis 0 traffic-engineering true
/configure router "Base" isis 0 flexible-algorithms admin-state enable
/configure router "Base" isis 0 flexible-algorithms flex-algo 128 participate true
/configure router "Base" isis 0 flexible-algorithms flex-algo 128 loopfree-alternate
/configure router "Base" isis 0 flexible-algorithms flex-algo 128 micro-loop-avoidance
/configure router "Base" isis 0 traffic-engineering-options application-link-attributes legacy true
/configure router "Base" isis 0 segment-routing admin-state enable
/configure router "Base" isis 0 segment-routing prefix-sid-range global
/configure router "Base" isis 0 interface "system" admin-state enable
/configure router "Base" isis 0 interface "system" passive true
/configure router "Base" isis 0 interface "system" ipv4-node-sid index 1003
/configure router "Base" isis 0 interface "to_P4" admin-state enable
/configure router "Base" isis 0 interface "to_P4" interface-type point-to-point
/configure router "Base" isis 0 interface "to_P4" level 2 metric 10
/configure router "Base" isis 0 interface "to_PE1" admin-state enable
/configure router "Base" isis 0 interface "to_PE1" interface-type point-to-point
/configure router "Base" isis 0 interface "to_PE1" level 2 metric 10
/configure router "Base" isis 0 interface "to_PE2" admin-state enable
/configure router "Base" isis 0 interface "to_PE2" interface-type point-to-point
/configure router "Base" isis 0 interface "to_PE2" level 2 metric 10
/configure router "Base" isis 0 level 2 wide-metrics-only true
/configure router "Base" isis 1 admin-state enable
/configure router "Base" isis 1 advertise-router-capability area
/configure router "Base" isis 1 all-l1isis 01:00:01:49:00:01
/configure router "Base" isis 1 all-l2isis 01:00:01:49:00:01
/configure router "Base" isis 1 iid-tlv true
/configure router "Base" isis 1 level-capability 2
/configure router "Base" isis 1 traffic-engineering true
/configure router "Base" isis 1 area-address [49.0001]
/configure router "Base" isis 1 entropy-label override-tunnel-elc true
/configure router "Base" isis 1 traffic-engineering-options application-link-attributes legacy true
/configure router "Base" isis 1 segment-routing admin-state enable
/configure router "Base" isis 1 segment-routing prefix-sid-range global
/configure router "Base" isis 1 interface "system" admin-state enable
/configure router "Base" isis 1 interface "system" passive true
/configure router "Base" isis 1 interface "system" level-capability 2
/configure router "Base" isis 1 interface "system" ipv4-node-sid index 1103
/configure router "Base" isis 1 interface "to_P4" admin-state enable
/configure router "Base" isis 1 interface "to_P4" interface-type point-to-point
/configure router "Base" isis 1 interface "to_P4" level-capability 2
/configure router "Base" isis 1 interface "to_P4" level 2 metric 100
/configure router "Base" isis 1 interface "to_PE1" admin-state enable
/configure router "Base" isis 1 interface "to_PE1" interface-type point-to-point
/configure router "Base" isis 1 interface "to_PE1" level-capability 2
/configure router "Base" isis 1 interface "to_PE1" level 2 metric 10
/configure router "Base" isis 1 interface "to_PE2" admin-state enable
/configure router "Base" isis 1 interface "to_PE2" interface-type point-to-point
/configure router "Base" isis 1 interface "to_PE2" level-capability 2
/configure router "Base" isis 1 interface "to_PE2" level 2 metric 10
/configure router "Base" isis 1 level 2 wide-metrics-only true
/configure router "Base" isis 2 admin-state enable
/configure router "Base" isis 2 advertise-router-capability area
/configure router "Base" isis 2 all-l1isis 01:00:01:49:00:01
/configure router "Base" isis 2 all-l2isis 01:00:01:49:00:01
/configure router "Base" isis 2 iid-tlv true
/configure router "Base" isis 2 level-capability 2
/configure router "Base" isis 2 traffic-engineering true
/configure router "Base" isis 2 area-address [49.0001]
/configure router "Base" isis 2 entropy-label override-tunnel-elc true
/configure router "Base" isis 2 traffic-engineering-options application-link-attributes legacy true
/configure router "Base" isis 2 segment-routing admin-state enable
/configure router "Base" isis 2 segment-routing prefix-sid-range global
/configure router "Base" isis 2 interface "system" admin-state enable
/configure router "Base" isis 2 interface "system" passive true
/configure router "Base" isis 2 interface "system" ipv4-node-sid index 1203
/configure router "Base" isis 2 interface "to_P4" admin-state enable
/configure router "Base" isis 2 interface "to_P4" interface-type point-to-point
/configure router "Base" isis 2 interface "to_P4" level 2 metric 100
/configure router "Base" isis 2 interface "to_PE1" admin-state enable
/configure router "Base" isis 2 interface "to_PE1" interface-type point-to-point
/configure router "Base" isis 2 interface "to_PE1" level 2 metric 1000
/configure router "Base" isis 2 interface "to_PE2" admin-state enable
/configure router "Base" isis 2 interface "to_PE2" interface-type point-to-point
/configure router "Base" isis 2 interface "to_PE2" level 2 metric 1000
/configure router "Base" isis 2 level 2 wide-metrics-only true
/configure router "Base" mpls admin-state enable
/configure router "Base" mpls interface "system" admin-state enable
/configure router "Base" mpls interface "to_P4" admin-state enable
/configure router "Base" mpls interface "to_P4" te-metric 50
/configure router "Base" mpls interface "to_PE1" admin-state enable
/configure router "Base" mpls interface "to_PE1" te-metric 500
/configure router "Base" mpls interface "to_PE2" admin-state enable
/configure router "Base" mpls interface "to_PE2" te-metric 50
/configure router "Base" rsvp interface "system" admin-state enable
/configure router "Base" rsvp interface "to_P4" admin-state enable
/configure router "Base" rsvp interface "to_PE1" admin-state enable
/configure router "Base" rsvp interface "to_PE2" admin-state enable
/configure routing-options ip-fast-reroute true
/configure service customer "1"
/configure service vprn "1003" admin-state enable
/configure service vprn "1003" customer "1"
/configure service vprn "1003" autonomous-system 65000
/configure service vprn "1003" router-id 10.0.0.3
/configure service vprn "1003" bgp-ipvpn mpls admin-state enable
/configure service vprn "1003" bgp-ipvpn mpls route-distinguisher "10.0.0.3:1003"
/configure service vprn "1003" bgp-ipvpn mpls vrf-target community "target:65000:1003"
/configure service vprn "1003" bgp-ipvpn mpls auto-bind-tunnel resolution filter
/configure service vprn "1003" bgp-ipvpn mpls auto-bind-tunnel resolution-filter sr-isis true
/configure service vprn "1003" interface "loopback" loopback true
/configure service vprn "1003" interface "loopback" ipv4 primary address 10.0.0.3
/configure service vprn "1003" interface "loopback" ipv4 primary prefix-length 32
/configure system name "p3"
```

And the configuration for P4 is:

```srl
/configure card 1 admin-state enable
/configure card 1 card-type iom-1
/configure card 1 mda 1 mda-type me12-100gb-qsfp28
/configure port 1/1/c1 admin-state enable
/configure port 1/1/c1 connector breakout c1-100g
/configure port 1/1/c1/1 admin-state enable
/configure port 1/1/c1/1 description "to_P3"
/configure port 1/1/c1/1 ethernet mode hybrid
/configure port 1/1/c1/1 ethernet encap-type dot1q
/configure port 1/1/c2 admin-state enable
/configure port 1/1/c2 connector breakout c1-100g
/configure port 1/1/c2/1 admin-state enable
/configure port 1/1/c2/1 description "to_PE1"
/configure port 1/1/c2/1 ethernet mode hybrid
/configure port 1/1/c2/1 ethernet encap-type dot1q
/configure port 1/1/c3 admin-state enable
/configure port 1/1/c3 connector breakout c1-100g
/configure port 1/1/c3/1 admin-state enable
/configure port 1/1/c3/1 description "to_PE2"
/configure port 1/1/c3/1 ethernet mode hybrid
/configure port 1/1/c3/1 ethernet encap-type dot1q
/configure router "Base" autonomous-system 65000
/configure router "Base" ecmp 2
/configure router "Base" router-id 10.0.0.4
/configure router "Base" interface "system" ipv4 primary address 10.0.0.4
/configure router "Base" interface "system" ipv4 primary prefix-length 32
/configure router "Base" interface "to_P3" description "to_P3"
/configure router "Base" interface "to_P3" port 1/1/c1/1:2
/configure router "Base" interface "to_P3" ipv4 primary address 10.3.4.4
/configure router "Base" interface "to_P3" ipv4 primary prefix-length 24
/configure router "Base" interface "to_PE1" description "to_PE1"
/configure router "Base" interface "to_PE1" port 1/1/c2/1:2
/configure router "Base" interface "to_PE1" ipv4 primary address 10.1.4.4
/configure router "Base" interface "to_PE1" ipv4 primary prefix-length 24
/configure router "Base" interface "to_PE2" description "to_PE2"
/configure router "Base" interface "to_PE2" port 1/1/c3/1:2
/configure router "Base" interface "to_PE2" ipv4 primary address 10.2.4.4
/configure router "Base" interface "to_PE2" ipv4 primary prefix-length 24
/configure router "Base" mpls-labels static-label-range 1968
/configure router "Base" mpls-labels sr-labels start 16000
/configure router "Base" mpls-labels sr-labels end 24000
/configure router "Base" mpls-labels reserved-label-block "Anysec" start-label 2000
/configure router "Base" mpls-labels reserved-label-block "Anysec" end-label 5999
/configure router "Base" bgp rapid-withdrawal true
/configure router "Base" bgp family ipv4 true
/configure router "Base" bgp family vpn-ipv4 true
/configure router "Base" bgp family ipv6 true
/configure router "Base" bgp family vpn-ipv6 true
/configure router "Base" bgp family l2-vpn true
/configure router "Base" bgp family route-target true
/configure router "Base" bgp family evpn true
/configure router "Base" bgp family label-ipv4 true
/configure router "Base" bgp family label-ipv6 true
/configure router "Base" bgp family sr-policy-ipv4 true
/configure router "Base" bgp family sr-policy-ipv6 true
/configure router "Base" bgp cluster cluster-id 10.0.0.4
/configure router "Base" bgp local-as as-number 65000
/configure router "Base" bgp rapid-update vpn-ipv4 true
/configure router "Base" bgp rapid-update vpn-ipv6 true
/configure router "Base" bgp rapid-update evpn true
/configure router "Base" bgp rapid-update label-ipv4 true
/configure router "Base" bgp rapid-update label-ipv6 true
/configure router "Base" bgp multipath max-paths 2
/configure router "Base" bgp group "ibgp" keepalive 5
/configure router "Base" bgp group "ibgp" min-route-advertisement 5
/configure router "Base" bgp group "ibgp" type internal
/configure router "Base" bgp group "ibgp" peer-as 65000
/configure router "Base" bgp group "ibgp" local-address 10.0.0.4
/configure router "Base" bgp group "ibgp" peer-ip-tracking true
/configure router "Base" bgp neighbor "10.0.0.1" description "PE1"
/configure router "Base" bgp neighbor "10.0.0.1" group "ibgp"
/configure router "Base" bgp neighbor "10.0.0.2" description "PE2"
/configure router "Base" bgp neighbor "10.0.0.2" group "ibgp"
/configure router "Base" bgp neighbor "10.0.0.3" description "P3"
/configure router "Base" bgp neighbor "10.0.0.3" group "ibgp"
/configure router "Base" isis 0 admin-state enable
/configure router "Base" isis 0 advertise-router-capability as
/configure router "Base" isis 0 iid-tlv true
/configure router "Base" isis 0 level-capability 2
/configure router "Base" isis 0 traffic-engineering true
/configure router "Base" isis 0 flexible-algorithms admin-state enable
/configure router "Base" isis 0 flexible-algorithms flex-algo 128 participate true
/configure router "Base" isis 0 flexible-algorithms flex-algo 128 loopfree-alternate
/configure router "Base" isis 0 flexible-algorithms flex-algo 128 micro-loop-avoidance
/configure router "Base" isis 0 traffic-engineering-options application-link-attributes legacy true
/configure router "Base" isis 0 segment-routing admin-state enable
/configure router "Base" isis 0 segment-routing prefix-sid-range global
/configure router "Base" isis 0 interface "system" admin-state enable
/configure router "Base" isis 0 interface "system" passive true
/configure router "Base" isis 0 interface "system" ipv4-node-sid index 1004
/configure router "Base" isis 0 interface "to_P3" admin-state enable
/configure router "Base" isis 0 interface "to_P3" interface-type point-to-point
/configure router "Base" isis 0 interface "to_P3" level 2 metric 10
/configure router "Base" isis 0 interface "to_PE1" admin-state enable
/configure router "Base" isis 0 interface "to_PE1" interface-type point-to-point
/configure router "Base" isis 0 interface "to_PE1" level 2 metric 10
/configure router "Base" isis 0 interface "to_PE2" admin-state enable
/configure router "Base" isis 0 interface "to_PE2" interface-type point-to-point
/configure router "Base" isis 0 interface "to_PE2" level 2 metric 10
/configure router "Base" isis 0 level 2 wide-metrics-only true
/configure router "Base" isis 1 admin-state enable
/configure router "Base" isis 1 advertise-router-capability area
/configure router "Base" isis 1 all-l1isis 01:00:01:49:00:01
/configure router "Base" isis 1 all-l2isis 01:00:01:49:00:01
/configure router "Base" isis 1 iid-tlv true
/configure router "Base" isis 1 level-capability 2
/configure router "Base" isis 1 traffic-engineering true
/configure router "Base" isis 1 area-address [49.0001]
/configure router "Base" isis 1 entropy-label override-tunnel-elc true
/configure router "Base" isis 1 traffic-engineering-options application-link-attributes legacy true
/configure router "Base" isis 1 segment-routing admin-state enable
/configure router "Base" isis 1 segment-routing prefix-sid-range global
/configure router "Base" isis 1 interface "system" admin-state enable
/configure router "Base" isis 1 interface "system" passive true
/configure router "Base" isis 1 interface "system" level-capability 2
/configure router "Base" isis 1 interface "system" ipv4-node-sid index 1104
/configure router "Base" isis 1 interface "to_P3" admin-state enable
/configure router "Base" isis 1 interface "to_P3" interface-type point-to-point
/configure router "Base" isis 1 interface "to_P3" level-capability 2
/configure router "Base" isis 1 interface "to_P3" level 2 metric 100
/configure router "Base" isis 1 interface "to_PE1" admin-state enable
/configure router "Base" isis 1 interface "to_PE1" interface-type point-to-point
/configure router "Base" isis 1 interface "to_PE1" level-capability 2
/configure router "Base" isis 1 interface "to_PE1" level 2 metric 1000
/configure router "Base" isis 1 interface "to_PE2" admin-state enable
/configure router "Base" isis 1 interface "to_PE2" interface-type point-to-point
/configure router "Base" isis 1 interface "to_PE2" level-capability 2
/configure router "Base" isis 1 interface "to_PE2" level 2 metric 1000
/configure router "Base" isis 1 level 2 wide-metrics-only true
/configure router "Base" isis 2 admin-state enable
/configure router "Base" isis 2 advertise-router-capability area
/configure router "Base" isis 2 all-l1isis 01:00:01:49:00:01
/configure router "Base" isis 2 all-l2isis 01:00:01:49:00:01
/configure router "Base" isis 2 iid-tlv true
/configure router "Base" isis 2 level-capability 2
/configure router "Base" isis 2 traffic-engineering true
/configure router "Base" isis 2 area-address [49.0001]
/configure router "Base" isis 2 entropy-label override-tunnel-elc true
/configure router "Base" isis 2 traffic-engineering-options application-link-attributes legacy true
/configure router "Base" isis 2 segment-routing admin-state enable
/configure router "Base" isis 2 segment-routing prefix-sid-range global
/configure router "Base" isis 2 interface "system" admin-state enable
/configure router "Base" isis 2 interface "system" passive true
/configure router "Base" isis 2 interface "system" ipv4-node-sid index 1204
/configure router "Base" isis 2 interface "to_P3" admin-state enable
/configure router "Base" isis 2 interface "to_P3" interface-type point-to-point
/configure router "Base" isis 2 interface "to_P3" level 2 metric 100
/configure router "Base" isis 2 interface "to_PE1" admin-state enable
/configure router "Base" isis 2 interface "to_PE1" interface-type point-to-point
/configure router "Base" isis 2 interface "to_PE1" level 2 metric 10
/configure router "Base" isis 2 interface "to_PE2" admin-state enable
/configure router "Base" isis 2 interface "to_PE2" interface-type point-to-point
/configure router "Base" isis 2 interface "to_PE2" level 2 metric 10
/configure router "Base" isis 2 level 2 wide-metrics-only true
/configure router "Base" mpls admin-state enable
/configure router "Base" mpls interface "system" admin-state enable
/configure router "Base" mpls interface "to_P3" admin-state enable
/configure router "Base" mpls interface "to_P3" te-metric 50
/configure router "Base" mpls interface "to_PE1" admin-state enable
/configure router "Base" mpls interface "to_PE1" te-metric 50
/configure router "Base" mpls interface "to_PE2" admin-state enable
/configure router "Base" mpls interface "to_PE2" te-metric 500
/configure router "Base" rsvp interface "system" admin-state enable
/configure router "Base" rsvp interface "to_P3" admin-state enable
/configure router "Base" rsvp interface "to_PE1" admin-state enable
/configure router "Base" rsvp interface "to_PE2" admin-state enable
/configure routing-options ip-fast-reroute true
/configure service customer "1"
/configure service vprn "1003" admin-state enable
/configure service vprn "1003" customer "1"
/configure service vprn "1003" autonomous-system 65000
/configure service vprn "1003" router-id 10.0.0.4
/configure service vprn "1003" bgp-ipvpn mpls admin-state enable
/configure service vprn "1003" bgp-ipvpn mpls route-distinguisher "10.0.0.4:1003"
/configure service vprn "1003" bgp-ipvpn mpls vrf-target community "target:65000:1003"
/configure service vprn "1003" bgp-ipvpn mpls auto-bind-tunnel resolution filter
/configure service vprn "1003" bgp-ipvpn mpls auto-bind-tunnel resolution-filter sr-isis true
/configure service vprn "1003" interface "loopback" loopback true
/configure service vprn "1003" interface "loopback" ipv4 primary address 10.0.0.4
/configure service vprn "1003" interface "loopback" ipv4 primary prefix-length 32
/configure system name "p4"
```

With this done, we will move on to the Provider Edge routers.

We will begin with PE1:

```srl
/configure anysec reserved-label-block "Anysec"
/configure anysec mka-over-ip mka-udp-port 10000
/configure anysec security-termination-policies policy "STP_VPRN-1003" admin-state enable
/configure anysec security-termination-policies policy "STP_VPRN-1003" local-address 10.0.0.1
/configure anysec security-termination-policies policy "STP_VPRN-1003" rx-must-be-encrypted false
/configure anysec security-termination-policies policy "STP_VPRN-1003" protocol sr-isis
/configure anysec security-termination-policies policy "STP_VPRN-1003" igp-instance-id 0
/configure anysec security-termination-policies policy "STP_VPRN-1003" flex-algo-id 128
/configure anysec security-termination-policies policy "STP_VLL-1001" admin-state enable
/configure anysec security-termination-policies policy "STP_VLL-1001" local-address 10.0.0.11
/configure anysec security-termination-policies policy "STP_VLL-1001" rx-must-be-encrypted false
/configure anysec security-termination-policies policy "STP_VLL-1001" protocol sr-isis
/configure anysec security-termination-policies policy "STP_VLL-1001" igp-instance-id 1
/configure anysec security-termination-policies policy "STP_SERV-1002" admin-state enable
/configure anysec security-termination-policies policy "STP_SERV-1002" local-address 10.0.0.12
/configure anysec security-termination-policies policy "STP_SERV-1002" rx-must-be-encrypted false
/configure anysec security-termination-policies policy "STP_SERV-1002" protocol sr-isis
/configure anysec security-termination-policies policy "STP_SERV-1002" igp-instance-id 2
```

This first section starts ANYsec and selects the port that MKA will use, which is 10000. It also includes the configuration of the security policies that each service will use within ANYsec.

The next configuration is as follows:

```srl
/configure anysec tunnel-encryption encryption-group "EG_VPRN-1003" admin-state enable
/configure anysec tunnel-encryption encryption-group "EG_VPRN-1003" security-termination-policy "STP_VPRN-1003"
/configure anysec tunnel-encryption encryption-group "EG_VPRN-1003" encryption-label 2001
/configure anysec tunnel-encryption encryption-group "EG_VPRN-1003" ca-name "CA_VPRN-1003"
/configure anysec tunnel-encryption encryption-group "EG_VPRN-1003" peer-tunnel-attributes protocol sr-isis
/configure anysec tunnel-encryption encryption-group "EG_VPRN-1003" peer-tunnel-attributes igp-instance-id 0
/configure anysec tunnel-encryption encryption-group "EG_VPRN-1003" peer-tunnel-attributes flex-algo-id 128
/configure anysec tunnel-encryption encryption-group "EG_VPRN-1003" peer 10.0.0.2 admin-state enable
/configure anysec tunnel-encryption encryption-group "EG_VLL-1001" admin-state enable
/configure anysec tunnel-encryption encryption-group "EG_VLL-1001" security-termination-policy "STP_VLL-1001"
/configure anysec tunnel-encryption encryption-group "EG_VLL-1001" encryption-label 2101
/configure anysec tunnel-encryption encryption-group "EG_VLL-1001" ca-name "CA_VLL-1001"
/configure anysec tunnel-encryption encryption-group "EG_VLL-1001" peer-tunnel-attributes protocol sr-isis
/configure anysec tunnel-encryption encryption-group "EG_VLL-1001" peer-tunnel-attributes igp-instance-id 1
/configure anysec tunnel-encryption encryption-group "EG_VLL-1001" peer 10.0.0.21 admin-state enable
/configure anysec service-encryption encryption-group "EG_SERV-1002" admin-state enable
/configure anysec service-encryption encryption-group "EG_SERV-1002" security-termination-policy "STP_SERV-1002"
/configure anysec service-encryption encryption-group "EG_SERV-1002" encryption-label 2201
/configure anysec service-encryption encryption-group "EG_SERV-1002" ca-name "CA_SERV-1002"
/configure anysec service-encryption encryption-group "EG_SERV-1002" peer 10.0.0.22 admin-state enable
```

In this step, we create the encryption group that each service will use, associating the security policy, the CA it will use, the encryption label and its peer.

We can also see that two of the services will be using tunnel encryption, which means that these two services will be protected the same way by the same tunnel. Then we can see that EG_SERV-1002 is using service encryption, which means that the encryption targets the service itself instead of all the content within the tunnel.

Afterwards, the next step is:

```srl
/configure macsec connectivity-association "CA_VPRN-1003" admin-state enable
/configure macsec connectivity-association "CA_VPRN-1003" description "Anysec ISIS 0"
/configure macsec connectivity-association "CA_VPRN-1003" clear-tag-mode none
/configure macsec connectivity-association "CA_VPRN-1003" cipher-suite gcm-aes-xpn-128
/configure macsec connectivity-association "CA_VPRN-1003" anysec true
/configure macsec connectivity-association "CA_VPRN-1003" static-cak active-psk 1
/configure macsec connectivity-association "CA_VPRN-1003" static-cak mka-hello-interval 5
/configure macsec connectivity-association "CA_VPRN-1003" static-cak pre-shared-key 1 encryption-type aes-128-cmac
/configure macsec connectivity-association "CA_VPRN-1003" static-cak pre-shared-key 1 cak "0123456789ABCDEF0123456789ABCDEF"
/configure macsec connectivity-association "CA_VPRN-1003" static-cak pre-shared-key 1 cak-name "CC0123456789ABCDEF"
/configure macsec connectivity-association "CA_VPRN-1003" static-cak pre-shared-key 2 encryption-type aes-128-cmac
/configure macsec connectivity-association "CA_VPRN-1003" static-cak pre-shared-key 2 cak "123456789ABCDEF0123456789ABCDEF0"
/configure macsec connectivity-association "CA_VPRN-1003" static-cak pre-shared-key 2 cak-name "CC123456789ABCDEF0"
/configure macsec connectivity-association "CA_VLL-1001" admin-state enable
/configure macsec connectivity-association "CA_VLL-1001" description "Anysec ISIS 1"
/configure macsec connectivity-association "CA_VLL-1001" clear-tag-mode none
/configure macsec connectivity-association "CA_VLL-1001" cipher-suite gcm-aes-xpn-128
/configure macsec connectivity-association "CA_VLL-1001" anysec true
/configure macsec connectivity-association "CA_VLL-1001" static-cak active-psk 1
/configure macsec connectivity-association "CA_VLL-1001" static-cak mka-hello-interval 5
/configure macsec connectivity-association "CA_VLL-1001" static-cak pre-shared-key 1 encryption-type aes-128-cmac
/configure macsec connectivity-association "CA_VLL-1001" static-cak pre-shared-key 1 cak "0123456789ABCDEF0123456789ABCDEF"
/configure macsec connectivity-association "CA_VLL-1001" static-cak pre-shared-key 1 cak-name "AA0123456789ABCDEF"
/configure macsec connectivity-association "CA_VLL-1001" static-cak pre-shared-key 2 encryption-type aes-128-cmac
/configure macsec connectivity-association "CA_VLL-1001" static-cak pre-shared-key 2 cak "123456789ABCDEF0123456789ABCDEF0"
/configure macsec connectivity-association "CA_VLL-1001" static-cak pre-shared-key 2 cak-name "AA123456789ABCDEF0"
/configure macsec connectivity-association "CA_SERV-1002" admin-state enable
/configure macsec connectivity-association "CA_SERV-1002" description "Anysec ISIS 2"
/configure macsec connectivity-association "CA_SERV-1002" clear-tag-mode none
/configure macsec connectivity-association "CA_SERV-1002" cipher-suite gcm-aes-xpn-128
/configure macsec connectivity-association "CA_SERV-1002" anysec true
/configure macsec connectivity-association "CA_SERV-1002" static-cak active-psk 1
/configure macsec connectivity-association "CA_SERV-1002" static-cak mka-hello-interval 5
/configure macsec connectivity-association "CA_SERV-1002" static-cak pre-shared-key 1 encryption-type aes-128-cmac
/configure macsec connectivity-association "CA_SERV-1002" static-cak pre-shared-key 1 cak "0123456789ABCDEF0123456789ABCDEF"
/configure macsec connectivity-association "CA_SERV-1002" static-cak pre-shared-key 1 cak-name "BB0123456789ABCDEF"
/configure macsec connectivity-association "CA_SERV-1002" static-cak pre-shared-key 2 encryption-type aes-128-cmac
/configure macsec connectivity-association "CA_SERV-1002" static-cak pre-shared-key 2 cak "123456789ABCDEF0123456789ABCDEF0"
/configure macsec connectivity-association "CA_SERV-1002" static-cak pre-shared-key 2 cak-name "BB123456789ABCDEF0"
/configure macsec connectivity-association "CA_MACSec1" admin-state enable
/configure macsec connectivity-association "CA_MACSec1" description "MACSec CE/PE"
/configure macsec connectivity-association "CA_MACSec1" macsec-encrypt true
/configure macsec connectivity-association "CA_MACSec1" clear-tag-mode none
/configure macsec connectivity-association "CA_MACSec1" cipher-suite gcm-aes-xpn-128
/configure macsec connectivity-association "CA_MACSec1" static-cak active-psk 1
/configure macsec connectivity-association "CA_MACSec1" static-cak mka-hello-interval 5
/configure macsec connectivity-association "CA_MACSec1" static-cak pre-shared-key 1 encryption-type aes-128-cmac
/configure macsec connectivity-association "CA_MACSec1" static-cak pre-shared-key 1 cak "0123456789ABCDEF0123456789ABCDEF"
/configure macsec connectivity-association "CA_MACSec1" static-cak pre-shared-key 1 cak-name "0123456789ABCDEF"
/configure macsec connectivity-association "CA_MACSec1" static-cak pre-shared-key 2 encryption-type aes-128-cmac
/configure macsec connectivity-association "CA_MACSec1" static-cak pre-shared-key 2 cak "123456789ABCDEF0123456789ABCDEF0"
/configure macsec connectivity-association "CA_MACSec1" static-cak pre-shared-key 2 cak-name "123456789ABCDEF0"
```

In this step, we create the CAs that ANYsec will use for each service/tunnel and the CA used to communicate with the Consumer Edge.

The next step of the setup involves configuring routing and ports:

```srl
/configure multicast-management chassis-level per-mcast-plane-capacity total-capacity dynamic
/configure multicast-management chassis-level per-mcast-plane-capacity mcast-capacity primary-percentage 87.5
/configure multicast-management chassis-level per-mcast-plane-capacity mcast-capacity secondary-percentage 87.5
/configure multicast-management chassis-level per-mcast-plane-capacity redundant-mcast-capacity primary-percentage 87.5
/configure multicast-management chassis-level per-mcast-plane-capacity redundant-mcast-capacity secondary-percentage 87.5
/configure policy-options community "vrf_1003" member "target:65000:1003"
/configure policy-options policy-statement "vrf_1003_import" entry 10 from community name "vrf_1003"
/configure policy-options policy-statement "vrf_1003_import" entry 10 action action-type accept
/configure policy-options policy-statement "vrf_1003_import" entry 10 action flex-algo 128
/configure policy-options policy-statement "vrf_1003_import" default-action action-type accept
/configure policy-options policy-statement "vrf_1003_import" default-action flex-algo 128
/configure port 1/1/c1 admin-state enable
/configure port 1/1/c1 connector breakout c1-100g
/configure port 1/1/c1/1 admin-state enable
/configure port 1/1/c1/1 description "to_P3"
/configure port 1/1/c1/1 ethernet mode hybrid
/configure port 1/1/c1/1 ethernet encap-type dot1q
/configure port 1/1/c2 admin-state enable
/configure port 1/1/c2 connector breakout c1-100g
/configure port 1/1/c2/1 admin-state enable
/configure port 1/1/c2/1 description "to_P4"
/configure port 1/1/c2/1 ethernet mode hybrid
/configure port 1/1/c2/1 ethernet encap-type dot1q
/configure port 1/1/c3 admin-state enable
/configure port 1/1/c3 connector breakout c1-100g
/configure port 1/1/c3/1 admin-state enable
/configure port 1/1/c3/1 description "to_CE5"
/configure port 1/1/c3/1 ethernet mode access
/configure port 1/1/c3/1 ethernet encap-type dot1q
/configure port 1/1/c3/1 ethernet mtu 9000
/configure port 1/1/c3/1 ethernet dot1x admin-state enable
/configure port 1/1/c3/1 ethernet dot1x tunnel-dot1q false
/configure port 1/1/c3/1 ethernet dot1x tunnel-qinq false
/configure port 1/1/c3/1 ethernet dot1x macsec sub-port 1 admin-state enable
/configure port 1/1/c3/1 ethernet dot1x macsec sub-port 1 ca-name "CA_MACSec1"
/configure port 1/1/c3/1 ethernet dot1x macsec sub-port 1 max-peers 5
/configure port 1/1/c3/1 ethernet dot1x macsec sub-port 1 encap-match all-match true
/configure router "Base" autonomous-system 65000
/configure router "Base" ecmp 4
/configure router "Base" router-id 10.0.0.1
/configure router "Base" interface "lo1" loopback
/configure router "Base" interface "lo1" ipv4 primary address 10.0.0.11
/configure router "Base" interface "lo1" ipv4 primary prefix-length 32
/configure router "Base" interface "lo2" loopback
/configure router "Base" interface "lo2" ipv4 primary address 10.0.0.12
/configure router "Base" interface "lo2" ipv4 primary prefix-length 32
/configure router "Base" interface "system" ipv4 primary address 10.0.0.1
/configure router "Base" interface "system" ipv4 primary prefix-length 32
/configure router "Base" interface "to_P3" description "to_P3"
/configure router "Base" interface "to_P3" port 1/1/c1/1:2
/configure router "Base" interface "to_P3" ipv4 primary address 10.1.3.1
/configure router "Base" interface "to_P3" ipv4 primary prefix-length 24
/configure router "Base" interface "to_P4" description "to_P4"
/configure router "Base" interface "to_P4" port 1/1/c2/1:2
/configure router "Base" interface "to_P4" ipv4 primary address 10.1.4.1
/configure router "Base" interface "to_P4" ipv4 primary prefix-length 24
/configure router "Base" mpls-labels static-label-range 1968
/configure router "Base" mpls-labels sr-labels start 16000
/configure router "Base" mpls-labels sr-labels end 24000
/configure router "Base" mpls-labels reserved-label-block "Anysec" start-label 2000
/configure router "Base" mpls-labels reserved-label-block "Anysec" end-label 5999
/configure router "Base" bgp rapid-withdrawal true
/configure router "Base" bgp family ipv4 true
/configure router "Base" bgp family vpn-ipv4 true
/configure router "Base" bgp family ipv6 true
/configure router "Base" bgp family vpn-ipv6 true
/configure router "Base" bgp family l2-vpn true
/configure router "Base" bgp family route-target true
/configure router "Base" bgp family evpn true
/configure router "Base" bgp family label-ipv4 true
/configure router "Base" bgp family label-ipv6 true
/configure router "Base" bgp family sr-policy-ipv4 true
/configure router "Base" bgp family sr-policy-ipv6 true
/configure router "Base" bgp local-as as-number 65000
/configure router "Base" bgp rapid-update vpn-ipv4 true
/configure router "Base" bgp rapid-update vpn-ipv6 true
/configure router "Base" bgp rapid-update evpn true
/configure router "Base" bgp rapid-update label-ipv4 true
/configure router "Base" bgp rapid-update label-ipv6 true
/configure router "Base" bgp multipath max-paths 2
/configure router "Base" bgp group "ibgp" keepalive 5
/configure router "Base" bgp group "ibgp" min-route-advertisement 5
/configure router "Base" bgp group "ibgp" type internal
/configure router "Base" bgp group "ibgp" peer-as 65000
/configure router "Base" bgp group "ibgp" local-address 10.0.0.1
/configure router "Base" bgp group "ibgp" peer-ip-tracking true
/configure router "Base" bgp neighbor "10.0.0.3" description "P3"
/configure router "Base" bgp neighbor "10.0.0.3" group "ibgp"
/configure router "Base" bgp neighbor "10.0.0.4" description "P4"
/configure router "Base" bgp neighbor "10.0.0.4" group "ibgp"
/configure router "Base" isis 0 admin-state enable
/configure router "Base" isis 0 advertise-router-capability as
/configure router "Base" isis 0 iid-tlv true
/configure router "Base" isis 0 level-capability 2
/configure router "Base" isis 0 traffic-engineering true
/configure router "Base" isis 0 area-address [49.0001]
/configure router "Base" isis 0 entropy-label override-tunnel-elc true
/configure router "Base" isis 0 flexible-algorithms admin-state enable
/configure router "Base" isis 0 flexible-algorithms flex-algo 128 participate true
/configure router "Base" isis 0 flexible-algorithms flex-algo 128 advertise "Flex-Algo-128_TE-metric"
/configure router "Base" isis 0 flexible-algorithms flex-algo 128 loopfree-alternate
/configure router "Base" isis 0 flexible-algorithms flex-algo 128 micro-loop-avoidance
/configure router "Base" isis 0 traffic-engineering-options application-link-attributes legacy false
/configure router "Base" isis 0 segment-routing admin-state enable
/configure router "Base" isis 0 segment-routing prefix-sid-range global
/configure router "Base" isis 0 interface "system" admin-state enable
/configure router "Base" isis 0 interface "system" passive true
/configure router "Base" isis 0 interface "system" ipv4-node-sid index 1001
/configure router "Base" isis 0 interface "system" flex-algo 128 ipv4-node-sid label 18001
/configure router "Base" isis 0 interface "to_P3" admin-state enable
/configure router "Base" isis 0 interface "to_P3" interface-type point-to-point
/configure router "Base" isis 0 interface "to_P3" level 2 metric 10
/configure router "Base" isis 0 interface "to_P4" admin-state enable
/configure router "Base" isis 0 interface "to_P4" interface-type point-to-point
/configure router "Base" isis 0 interface "to_P4" level 2 metric 10
/configure router "Base" isis 0 level 2 wide-metrics-only true
/configure router "Base" isis 1 admin-state enable
/configure router "Base" isis 1 advertise-router-capability area
/configure router "Base" isis 1 all-l1isis 01:00:01:49:00:01
/configure router "Base" isis 1 all-l2isis 01:00:01:49:00:01
/configure router "Base" isis 1 iid-tlv true
/configure router "Base" isis 1 level-capability 2
/configure router "Base" isis 1 traffic-engineering true
/configure router "Base" isis 1 area-address [49.0001]
/configure router "Base" isis 1 entropy-label override-tunnel-elc true
/configure router "Base" isis 1 traffic-engineering-options application-link-attributes legacy false
/configure router "Base" isis 1 segment-routing admin-state enable
/configure router "Base" isis 1 segment-routing prefix-sid-range global
/configure router "Base" isis 1 interface "lo1" admin-state enable
/configure router "Base" isis 1 interface "lo1" passive true
/configure router "Base" isis 1 interface "lo1" ipv4-node-sid index 1101
/configure router "Base" isis 1 interface "to_P3" admin-state enable
/configure router "Base" isis 1 interface "to_P3" interface-type point-to-point
/configure router "Base" isis 1 interface "to_P3" level 2 metric 10
/configure router "Base" isis 1 interface "to_P4" admin-state enable
/configure router "Base" isis 1 interface "to_P4" interface-type point-to-point
/configure router "Base" isis 1 interface "to_P4" level 2 metric 1000
/configure router "Base" isis 1 level 2 wide-metrics-only true
/configure router "Base" isis 2 admin-state enable
/configure router "Base" isis 2 advertise-router-capability area
/configure router "Base" isis 2 all-l1isis 01:00:01:49:00:01
/configure router "Base" isis 2 all-l2isis 01:00:01:49:00:01
/configure router "Base" isis 2 iid-tlv true
/configure router "Base" isis 2 level-capability 2
/configure router "Base" isis 2 traffic-engineering true
/configure router "Base" isis 2 area-address [49.0001]
/configure router "Base" isis 2 entropy-label override-tunnel-elc true
/configure router "Base" isis 2 traffic-engineering-options application-link-attributes legacy false
/configure router "Base" isis 2 segment-routing admin-state enable
/configure router "Base" isis 2 segment-routing prefix-sid-range global
/configure router "Base" isis 2 interface "lo2" admin-state enable
/configure router "Base" isis 2 interface "lo2" passive true
/configure router "Base" isis 2 interface "lo2" ipv4-node-sid index 1201
/configure router "Base" isis 2 interface "to_P3" admin-state enable
/configure router "Base" isis 2 interface "to_P3" interface-type point-to-point
/configure router "Base" isis 2 interface "to_P3" level 2 metric 1000
/configure router "Base" isis 2 interface "to_P4" admin-state enable
/configure router "Base" isis 2 interface "to_P4" interface-type point-to-point
/configure router "Base" isis 2 interface "to_P4" level 2 metric 10
/configure router "Base" isis 2 level 2 wide-metrics-only true
/configure router "Base" ldp targeted-session peer 10.0.0.2 admin-state enable
/configure router "Base" ldp targeted-session peer 10.0.0.21 admin-state enable
/configure router "Base" ldp targeted-session peer 10.0.0.21 local-lsr-id interface-name "lo1"
/configure router "Base" ldp targeted-session peer 10.0.0.22 admin-state enable
/configure router "Base" ldp targeted-session peer 10.0.0.22 local-lsr-id interface-name "lo2"
/configure router "Base" mpls admin-state enable
/configure router "Base" mpls interface "system" admin-state enable
/configure router "Base" mpls interface "to_P3" admin-state enable
/configure router "Base" mpls interface "to_P3" te-metric 500
/configure router "Base" mpls interface "to_P4" admin-state enable
/configure router "Base" mpls interface "to_P4" te-metric 50
/configure router "Base" mpls path "Flex-Algo-128" admin-state enable
/configure router "Base" mpls path "Flex-Algo-128" hop 1 sid-label 18002
/configure router "Base" mpls lsp "PE1-to-PE2-Flex-Algo-128" admin-state enable
/configure router "Base" mpls lsp "PE1-to-PE2-Flex-Algo-128" type p2p-sr-te
/configure router "Base" mpls lsp "PE1-to-PE2-Flex-Algo-128" to 10.0.0.2
/configure router "Base" mpls lsp "PE1-to-PE2-Flex-Algo-128" max-sr-labels label-stack-size 3
/configure router "Base" mpls lsp "PE1-to-PE2-Flex-Algo-128" max-sr-labels additional-frr-labels 2
/configure router "Base" mpls lsp "PE1-to-PE2-Flex-Algo-128" primary "Flex-Algo-128" admin-state enable
/configure router "Base" rsvp interface "system" admin-state enable
/configure router "Base" rsvp interface "to_P3" admin-state enable
/configure router "Base" rsvp interface "to_P4" admin-state enable
/configure routing-options ip-fast-reroute true
/configure routing-options flexible-algorithm-definitions flex-algo "Flex-Algo-128_TE-metric" admin-state enable
/configure routing-options flexible-algorithm-definitions flex-algo "Flex-Algo-128_TE-metric" description "Use TE-Metrics. Assimetric paths: PE1->P3->P4->PE2 & PE2->P3->PE1"
/configure routing-options flexible-algorithm-definitions flex-algo "Flex-Algo-128_TE-metric" metric-type te-metric
```

Finally, we configure the services that we will use:

```srl
/configure service customer "1"
/configure service epipe "1001" admin-state enable
/configure service epipe "1001" description "Epipe using ISIS 1 best IGP metric on Top"
/configure service epipe "1001" customer "1"
/configure service epipe "1001" service-mtu 8100
/configure service epipe "1001" spoke-sdp 1222:1001 admin-state enable
/configure service epipe "1001" sap 1/1/c3/1:1001 admin-state enable
/configure service sdp 222 admin-state enable
/configure service sdp 222 description "To R1 - VPRN using ISIS 0 with Flex-Algo"
/configure service sdp 222 delivery-type mpls
/configure service sdp 222 far-end ip-address 10.0.0.2
/configure service sdp 222 lsp "PE1-to-PE2-Flex-Algo-128"
/configure service sdp 1222 admin-state enable
/configure service sdp 1222 description "To R1 - Epipe using ISIS 1 best IGP metric on Top"
/configure service sdp 1222 delivery-type mpls
/configure service sdp 1222 sr-isis true
/configure service sdp 1222 far-end ip-address 10.0.0.21
/configure service sdp 2222 admin-state enable
/configure service sdp 2222 description "To R1 - SERV using ISIS 2 best IGP metric on Bottom"
/configure service sdp 2222 delivery-type mpls
/configure service sdp 2222 sr-isis true
/configure service sdp 2222 far-end ip-address 10.0.0.22
/configure service epipe "1002" admin-state enable
/configure service epipe "1002" description "SERV using ISIS 2 best IGP metric on Bottom"
/configure service epipe "1002" customer "1"
/configure service epipe "1002" service-mtu 8100
/configure service epipe "1002" spoke-sdp 2222:1002 admin-state enable
/configure service epipe "1002" spoke-sdp 2222:1002 anysec-encryption-group "EG_SERV-1002"
/configure service epipe "1002" sap 1/1/c3/1:1002 admin-state enable
/configure service vprn "1003" admin-state enable
/configure service vprn "1003" description "VPRN using ISIS 0 with Flex-Algo"
/configure service vprn "1003" customer "1"
/configure service vprn "1003" autonomous-system 65000
/configure service vprn "1003" router-id 10.0.0.1
/configure service vprn "1003" bgp-ipvpn mpls admin-state enable
/configure service vprn "1003" bgp-ipvpn mpls route-distinguisher "10.0.0.1:1003"
/configure service vprn "1003" bgp-ipvpn mpls vrf-target community "target:65000:1003"
/configure service vprn "1003" bgp-ipvpn mpls vrf-import policy ["vrf_1003_import"]
/configure service vprn "1003" bgp-ipvpn mpls auto-bind-tunnel resolution filter
/configure service vprn "1003" bgp-ipvpn mpls auto-bind-tunnel allow-flex-algo-fallback true
/configure service vprn "1003" bgp-ipvpn mpls auto-bind-tunnel resolution-filter sr-isis true
/configure service vprn "1003" bgp-ipvpn mpls auto-bind-tunnel resolution-filter sr-te true
/configure service vprn "1003" interface "loopback" loopback true
/configure service vprn "1003" interface "loopback" ipv4 primary address 10.0.0.1
/configure service vprn "1003" interface "loopback" ipv4 primary prefix-length 32
/configure service vprn "1003" interface "to_client7" admin-state enable
/configure service vprn "1003" interface "to_client7" description "To client7 via CE5"
/configure service vprn "1003" interface "to_client7" ipv4 primary address 192.168.53.1
/configure service vprn "1003" interface "to_client7" ipv4 primary prefix-length 24
/configure service vprn "1003" interface "to_client7" sap 1/1/c3/1:1003 admin-state enable
/configure system name "pe1"
```

In this final step, we configure each of the services: an Epipe that goes through P3 to reach PE2, a VPRN that uses a flexible routing algorithm, and an Epipe that goes through P4 to reach PE2.

The configuration for PE2 is as follows. It is very similar to PE1, with minor changes to addresses and labels:

```srl
/configure anysec reserved-label-block "Anysec"
/configure anysec mka-over-ip mka-udp-port 10000
/configure anysec security-termination-policies policy "STP_VPRN-1003" admin-state enable
/configure anysec security-termination-policies policy "STP_VPRN-1003" local-address 10.0.0.2
/configure anysec security-termination-policies policy "STP_VPRN-1003" rx-must-be-encrypted false
/configure anysec security-termination-policies policy "STP_VPRN-1003" protocol sr-isis
/configure anysec security-termination-policies policy "STP_VPRN-1003" igp-instance-id 0
/configure anysec security-termination-policies policy "STP_VPRN-1003" flex-algo-id 128
/configure anysec security-termination-policies policy "STP_VLL-1001" admin-state enable
/configure anysec security-termination-policies policy "STP_VLL-1001" local-address 10.0.0.21
/configure anysec security-termination-policies policy "STP_VLL-1001" rx-must-be-encrypted false
/configure anysec security-termination-policies policy "STP_VLL-1001" protocol sr-isis
/configure anysec security-termination-policies policy "STP_VLL-1001" igp-instance-id 1
/configure anysec security-termination-policies policy "STP_SERV-1002" admin-state enable
/configure anysec security-termination-policies policy "STP_SERV-1002" local-address 10.0.0.22
/configure anysec security-termination-policies policy "STP_SERV-1002" rx-must-be-encrypted false
/configure anysec security-termination-policies policy "STP_SERV-1002" protocol sr-isis
/configure anysec security-termination-policies policy "STP_SERV-1002" igp-instance-id 2
/configure anysec tunnel-encryption encryption-group "EG_VPRN-1003" admin-state enable
/configure anysec tunnel-encryption encryption-group "EG_VPRN-1003" security-termination-policy "STP_VPRN-1003"
/configure anysec tunnel-encryption encryption-group "EG_VPRN-1003" encryption-label 2002
/configure anysec tunnel-encryption encryption-group "EG_VPRN-1003" ca-name "CA_VPRN-1003"
/configure anysec tunnel-encryption encryption-group "EG_VPRN-1003" peer-tunnel-attributes protocol sr-isis
/configure anysec tunnel-encryption encryption-group "EG_VPRN-1003" peer-tunnel-attributes igp-instance-id 0
/configure anysec tunnel-encryption encryption-group "EG_VPRN-1003" peer-tunnel-attributes flex-algo-id 128
/configure anysec tunnel-encryption encryption-group "EG_VPRN-1003" peer 10.0.0.1 admin-state enable
/configure anysec tunnel-encryption encryption-group "EG_VLL-1001" admin-state enable
/configure anysec tunnel-encryption encryption-group "EG_VLL-1001" security-termination-policy "STP_VLL-1001"
/configure anysec tunnel-encryption encryption-group "EG_VLL-1001" encryption-label 2102
/configure anysec tunnel-encryption encryption-group "EG_VLL-1001" ca-name "CA_VLL-1001"
/configure anysec tunnel-encryption encryption-group "EG_VLL-1001" peer-tunnel-attributes protocol sr-isis
/configure anysec tunnel-encryption encryption-group "EG_VLL-1001" peer-tunnel-attributes igp-instance-id 1
/configure anysec tunnel-encryption encryption-group "EG_VLL-1001" peer 10.0.0.11 admin-state enable
/configure anysec service-encryption encryption-group "EG_SERV-1002" admin-state enable
/configure anysec service-encryption encryption-group "EG_SERV-1002" security-termination-policy "STP_SERV-1002"
/configure anysec service-encryption encryption-group "EG_SERV-1002" encryption-label 2202
/configure anysec service-encryption encryption-group "EG_SERV-1002" ca-name "CA_SERV-1002"
/configure anysec service-encryption encryption-group "EG_SERV-1002" peer 10.0.0.12 admin-state enable
/configure macsec connectivity-association "CA_VPRN-1003" admin-state enable
/configure macsec connectivity-association "CA_VPRN-1003" description "Anysec ISIS 0"
/configure macsec connectivity-association "CA_VPRN-1003" clear-tag-mode none
/configure macsec connectivity-association "CA_VPRN-1003" cipher-suite gcm-aes-xpn-128
/configure macsec connectivity-association "CA_VPRN-1003" anysec true
/configure macsec connectivity-association "CA_VPRN-1003" static-cak active-psk 1
/configure macsec connectivity-association "CA_VPRN-1003" static-cak mka-hello-interval 5
/configure macsec connectivity-association "CA_VPRN-1003" static-cak pre-shared-key 1 encryption-type aes-128-cmac
/configure macsec connectivity-association "CA_VPRN-1003" static-cak pre-shared-key 1 cak "0123456789ABCDEF0123456789ABCDEF"
/configure macsec connectivity-association "CA_VPRN-1003" static-cak pre-shared-key 1 cak-name "CC0123456789ABCDEF"
/configure macsec connectivity-association "CA_VPRN-1003" static-cak pre-shared-key 2 encryption-type aes-128-cmac
/configure macsec connectivity-association "CA_VPRN-1003" static-cak pre-shared-key 2 cak "123456789ABCDEF0123456789ABCDEF0"
/configure macsec connectivity-association "CA_VPRN-1003" static-cak pre-shared-key 2 cak-name "CC123456789ABCDEF0"
/configure macsec connectivity-association "CA_VLL-1001" admin-state enable
/configure macsec connectivity-association "CA_VLL-1001" description "Anysec ISIS 1"
/configure macsec connectivity-association "CA_VLL-1001" clear-tag-mode none
/configure macsec connectivity-association "CA_VLL-1001" cipher-suite gcm-aes-xpn-128
/configure macsec connectivity-association "CA_VLL-1001" anysec true
/configure macsec connectivity-association "CA_VLL-1001" static-cak active-psk 1
/configure macsec connectivity-association "CA_VLL-1001" static-cak mka-hello-interval 5
/configure macsec connectivity-association "CA_VLL-1001" static-cak pre-shared-key 1 encryption-type aes-128-cmac
/configure macsec connectivity-association "CA_VLL-1001" static-cak pre-shared-key 1 cak "0123456789ABCDEF0123456789ABCDEF"
/configure macsec connectivity-association "CA_VLL-1001" static-cak pre-shared-key 1 cak-name "AA0123456789ABCDEF"
/configure macsec connectivity-association "CA_VLL-1001" static-cak pre-shared-key 2 encryption-type aes-128-cmac
/configure macsec connectivity-association "CA_VLL-1001" static-cak pre-shared-key 2 cak "123456789ABCDEF0123456789ABCDEF0"
/configure macsec connectivity-association "CA_VLL-1001" static-cak pre-shared-key 2 cak-name "AA123456789ABCDEF0"
/configure macsec connectivity-association "CA_SERV-1002" admin-state enable
/configure macsec connectivity-association "CA_SERV-1002" description "Anysec ISIS 2"
/configure macsec connectivity-association "CA_SERV-1002" clear-tag-mode none
/configure macsec connectivity-association "CA_SERV-1002" cipher-suite gcm-aes-xpn-128
/configure macsec connectivity-association "CA_SERV-1002" anysec true
/configure macsec connectivity-association "CA_SERV-1002" static-cak active-psk 1
/configure macsec connectivity-association "CA_SERV-1002" static-cak mka-hello-interval 5
/configure macsec connectivity-association "CA_SERV-1002" static-cak pre-shared-key 1 encryption-type aes-128-cmac
/configure macsec connectivity-association "CA_SERV-1002" static-cak pre-shared-key 1 cak "0123456789ABCDEF0123456789ABCDEF"
/configure macsec connectivity-association "CA_SERV-1002" static-cak pre-shared-key 1 cak-name "BB0123456789ABCDEF"
/configure macsec connectivity-association "CA_SERV-1002" static-cak pre-shared-key 2 encryption-type aes-128-cmac
/configure macsec connectivity-association "CA_SERV-1002" static-cak pre-shared-key 2 cak "123456789ABCDEF0123456789ABCDEF0"
/configure macsec connectivity-association "CA_SERV-1002" static-cak pre-shared-key 2 cak-name "BB123456789ABCDEF0"
/configure macsec connectivity-association "CA_MACSec1" admin-state enable
/configure macsec connectivity-association "CA_MACSec1" description "MACSec CE/PE"
/configure macsec connectivity-association "CA_MACSec1" macsec-encrypt true
/configure macsec connectivity-association "CA_MACSec1" clear-tag-mode none
/configure macsec connectivity-association "CA_MACSec1" cipher-suite gcm-aes-xpn-128
/configure macsec connectivity-association "CA_MACSec1" static-cak active-psk 1
/configure macsec connectivity-association "CA_MACSec1" static-cak mka-hello-interval 5
/configure macsec connectivity-association "CA_MACSec1" static-cak pre-shared-key 1 encryption-type aes-128-cmac
/configure macsec connectivity-association "CA_MACSec1" static-cak pre-shared-key 1 cak "0123456789ABCDEF0123456789ABCDEF"
/configure macsec connectivity-association "CA_MACSec1" static-cak pre-shared-key 1 cak-name "0123456789ABCDEF"
/configure macsec connectivity-association "CA_MACSec1" static-cak pre-shared-key 2 encryption-type aes-128-cmac
/configure macsec connectivity-association "CA_MACSec1" static-cak pre-shared-key 2 cak "123456789ABCDEF0123456789ABCDEF0"
/configure macsec connectivity-association "CA_MACSec1" static-cak pre-shared-key 2 cak-name "123456789ABCDEF0"
/configure multicast-management chassis-level per-mcast-plane-capacity total-capacity dynamic
/configure multicast-management chassis-level per-mcast-plane-capacity mcast-capacity primary-percentage 87.5
/configure multicast-management chassis-level per-mcast-plane-capacity mcast-capacity secondary-percentage 87.5
/configure multicast-management chassis-level per-mcast-plane-capacity redundant-mcast-capacity primary-percentage 87.5
/configure multicast-management chassis-level per-mcast-plane-capacity redundant-mcast-capacity secondary-percentage 87.5
/configure policy-options community "vrf_1003" member "target:65000:1003"
/configure policy-options policy-statement "vrf_1003_import" entry 10 from community name "vrf_1003"
/configure policy-options policy-statement "vrf_1003_import" entry 10 action action-type accept
/configure policy-options policy-statement "vrf_1003_import" entry 10 action flex-algo 128
/configure policy-options policy-statement "vrf_1003_import" default-action action-type accept
/configure policy-options policy-statement "vrf_1003_import" default-action flex-algo 128
/configure port 1/1/c1 admin-state enable
/configure port 1/1/c1 connector breakout c1-100g
/configure port 1/1/c1/1 admin-state enable
/configure port 1/1/c1/1 description "to_P3"
/configure port 1/1/c1/1 ethernet mode hybrid
/configure port 1/1/c1/1 ethernet encap-type dot1q
/configure port 1/1/c2 admin-state enable
/configure port 1/1/c2 connector breakout c1-100g
/configure port 1/1/c2/1 admin-state enable
/configure port 1/1/c2/1 description "to_P4"
/configure port 1/1/c2/1 ethernet mode hybrid
/configure port 1/1/c2/1 ethernet encap-type dot1q
/configure port 1/1/c3 admin-state enable
/configure port 1/1/c3 connector breakout c1-100g
/configure port 1/1/c3/1 admin-state enable
/configure port 1/1/c3/1 description "to_CE6"
/configure port 1/1/c3/1 ethernet mode access
/configure port 1/1/c3/1 ethernet encap-type dot1q
/configure port 1/1/c3/1 ethernet mtu 9000
/configure port 1/1/c3/1 ethernet dot1x admin-state enable
/configure port 1/1/c3/1 ethernet dot1x tunnel-dot1q false
/configure port 1/1/c3/1 ethernet dot1x tunnel-qinq false
/configure port 1/1/c3/1 ethernet dot1x macsec sub-port 1 admin-state enable
/configure port 1/1/c3/1 ethernet dot1x macsec sub-port 1 ca-name "CA_MACSec1"
/configure port 1/1/c3/1 ethernet dot1x macsec sub-port 1 max-peers 5
/configure port 1/1/c3/1 ethernet dot1x macsec sub-port 1 encap-match all-match true
/configure router "Base" autonomous-system 65000
/configure router "Base" ecmp 4
/configure router "Base" router-id 10.0.0.2
/configure router "Base" interface "lo1" loopback
/configure router "Base" interface "lo1" ipv4 primary address 10.0.0.21
/configure router "Base" interface "lo1" ipv4 primary prefix-length 32
/configure router "Base" interface "lo2" loopback
/configure router "Base" interface "lo2" ipv4 primary address 10.0.0.22
/configure router "Base" interface "lo2" ipv4 primary prefix-length 32
/configure router "Base" interface "system" ipv4 primary address 10.0.0.2
/configure router "Base" interface "system" ipv4 primary prefix-length 32
/configure router "Base" interface "to_P3" description "to_P3"
/configure router "Base" interface "to_P3" port 1/1/c1/1:2
/configure router "Base" interface "to_P3" ipv4 primary address 10.2.3.2
/configure router "Base" interface "to_P3" ipv4 primary prefix-length 24
/configure router "Base" interface "to_P4" description "to_P4"
/configure router "Base" interface "to_P4" port 1/1/c2/1:2
/configure router "Base" interface "to_P4" ipv4 primary address 10.2.4.2
/configure router "Base" interface "to_P4" ipv4 primary prefix-length 24
/configure router "Base" mpls-labels static-label-range 1968
/configure router "Base" mpls-labels sr-labels start 16000
/configure router "Base" mpls-labels sr-labels end 24000
/configure router "Base" mpls-labels reserved-label-block "Anysec" start-label 2000
/configure router "Base" mpls-labels reserved-label-block "Anysec" end-label 5999
/configure router "Base" bgp rapid-withdrawal true
/configure router "Base" bgp family ipv4 true
/configure router "Base" bgp family vpn-ipv4 true
/configure router "Base" bgp family ipv6 true
/configure router "Base" bgp family vpn-ipv6 true
/configure router "Base" bgp family l2-vpn true
/configure router "Base" bgp family route-target true
/configure router "Base" bgp family evpn true
/configure router "Base" bgp family label-ipv4 true
/configure router "Base" bgp family label-ipv6 true
/configure router "Base" bgp family sr-policy-ipv4 true
/configure router "Base" bgp family sr-policy-ipv6 true
/configure router "Base" bgp local-as as-number 65000
/configure router "Base" bgp rapid-update vpn-ipv4 true
/configure router "Base" bgp rapid-update vpn-ipv6 true
/configure router "Base" bgp rapid-update evpn true
/configure router "Base" bgp rapid-update label-ipv4 true
/configure router "Base" bgp rapid-update label-ipv6 true
/configure router "Base" bgp multipath max-paths 2
/configure router "Base" bgp group "ibgp" keepalive 5
/configure router "Base" bgp group "ibgp" min-route-advertisement 5
/configure router "Base" bgp group "ibgp" type internal
/configure router "Base" bgp group "ibgp" peer-as 65000
/configure router "Base" bgp group "ibgp" local-address 10.0.0.2
/configure router "Base" bgp group "ibgp" peer-ip-tracking true
/configure router "Base" bgp neighbor "10.0.0.3" description "P3"
/configure router "Base" bgp neighbor "10.0.0.3" group "ibgp"
/configure router "Base" bgp neighbor "10.0.0.4" description "P4"
/configure router "Base" bgp neighbor "10.0.0.4" group "ibgp"
/configure router "Base" isis 0 admin-state enable
/configure router "Base" isis 0 advertise-router-capability as
/configure router "Base" isis 0 iid-tlv true
/configure router "Base" isis 0 level-capability 2
/configure router "Base" isis 0 traffic-engineering true
/configure router "Base" isis 0 area-address [49.0001]
/configure router "Base" isis 0 entropy-label override-tunnel-elc true
/configure router "Base" isis 0 flexible-algorithms admin-state enable
/configure router "Base" isis 0 flexible-algorithms flex-algo 128 participate true
/configure router "Base" isis 0 flexible-algorithms flex-algo 128 advertise "Flex-Algo-128_TE-metric"
/configure router "Base" isis 0 flexible-algorithms flex-algo 128 loopfree-alternate
/configure router "Base" isis 0 flexible-algorithms flex-algo 128 micro-loop-avoidance
/configure router "Base" isis 0 traffic-engineering-options application-link-attributes legacy false
/configure router "Base" isis 0 segment-routing admin-state enable
/configure router "Base" isis 0 segment-routing prefix-sid-range global
/configure router "Base" isis 0 interface "system" admin-state enable
/configure router "Base" isis 0 interface "system" passive true
/configure router "Base" isis 0 interface "system" ipv4-node-sid index 1002
/configure router "Base" isis 0 interface "system" flex-algo 128 ipv4-node-sid label 18002
/configure router "Base" isis 0 interface "to_P3" admin-state enable
/configure router "Base" isis 0 interface "to_P3" interface-type point-to-point
/configure router "Base" isis 0 interface "to_P3" level 2 metric 10
/configure router "Base" isis 0 interface "to_P4" admin-state enable
/configure router "Base" isis 0 interface "to_P4" interface-type point-to-point
/configure router "Base" isis 0 interface "to_P4" level 2 metric 10
/configure router "Base" isis 0 level 2 wide-metrics-only true
/configure router "Base" isis 1 admin-state enable
/configure router "Base" isis 1 advertise-router-capability area
/configure router "Base" isis 1 all-l1isis 01:00:01:49:00:01
/configure router "Base" isis 1 all-l2isis 01:00:01:49:00:01
/configure router "Base" isis 1 iid-tlv true
/configure router "Base" isis 1 level-capability 2
/configure router "Base" isis 1 traffic-engineering true
/configure router "Base" isis 1 area-address [49.0001]
/configure router "Base" isis 1 entropy-label override-tunnel-elc true
/configure router "Base" isis 1 traffic-engineering-options application-link-attributes legacy false
/configure router "Base" isis 1 segment-routing admin-state enable
/configure router "Base" isis 1 segment-routing prefix-sid-range global
/configure router "Base" isis 1 interface "lo1" admin-state enable
/configure router "Base" isis 1 interface "lo1" passive true
/configure router "Base" isis 1 interface "lo1" ipv4-node-sid index 1102
/configure router "Base" isis 1 interface "to_P3" admin-state enable
/configure router "Base" isis 1 interface "to_P3" interface-type point-to-point
/configure router "Base" isis 1 interface "to_P3" level 2 metric 10
/configure router "Base" isis 1 interface "to_P4" admin-state enable
/configure router "Base" isis 1 interface "to_P4" interface-type point-to-point
/configure router "Base" isis 1 interface "to_P4" level 2 metric 1000
/configure router "Base" isis 1 level 2 wide-metrics-only true
/configure router "Base" isis 2 admin-state enable
/configure router "Base" isis 2 advertise-router-capability area
/configure router "Base" isis 2 all-l1isis 01:00:01:49:00:01
/configure router "Base" isis 2 all-l2isis 01:00:01:49:00:01
/configure router "Base" isis 2 iid-tlv true
/configure router "Base" isis 2 level-capability 2
/configure router "Base" isis 2 traffic-engineering true
/configure router "Base" isis 2 area-address [49.0001]
/configure router "Base" isis 2 entropy-label override-tunnel-elc true
/configure router "Base" isis 2 traffic-engineering-options application-link-attributes legacy false
/configure router "Base" isis 2 segment-routing admin-state enable
/configure router "Base" isis 2 segment-routing prefix-sid-range global
/configure router "Base" isis 2 interface "lo2" admin-state enable
/configure router "Base" isis 2 interface "lo2" passive true
/configure router "Base" isis 2 interface "lo2" ipv4-node-sid index 1202
/configure router "Base" isis 2 interface "to_P3" admin-state enable
/configure router "Base" isis 2 interface "to_P3" interface-type point-to-point
/configure router "Base" isis 2 interface "to_P3" level 2 metric 1000
/configure router "Base" isis 2 interface "to_P4" admin-state enable
/configure router "Base" isis 2 interface "to_P4" interface-type point-to-point
/configure router "Base" isis 2 interface "to_P4" level 2 metric 10
/configure router "Base" isis 2 level 2 wide-metrics-only true
/configure router "Base" ldp targeted-session peer 10.0.0.1 admin-state enable
/configure router "Base" ldp targeted-session peer 10.0.0.11 admin-state enable
/configure router "Base" ldp targeted-session peer 10.0.0.11 local-lsr-id interface-name "lo1"
/configure router "Base" ldp targeted-session peer 10.0.0.12 admin-state enable
/configure router "Base" ldp targeted-session peer 10.0.0.12 local-lsr-id interface-name "lo2"
/configure router "Base" mpls admin-state enable
/configure router "Base" mpls interface "system" admin-state enable
/configure router "Base" mpls interface "to_P3" admin-state enable
/configure router "Base" mpls interface "to_P3" te-metric 50
/configure router "Base" mpls interface "to_P4" admin-state enable
/configure router "Base" mpls interface "to_P4" te-metric 500
/configure router "Base" mpls path "Flex-Algo-128" admin-state enable
/configure router "Base" mpls path "Flex-Algo-128" hop 1 sid-label 18001
/configure router "Base" mpls lsp "PE2-to-PE1-Flex-Algo-128" admin-state enable
/configure router "Base" mpls lsp "PE2-to-PE1-Flex-Algo-128" type p2p-sr-te
/configure router "Base" mpls lsp "PE2-to-PE1-Flex-Algo-128" to 10.0.0.1
/configure router "Base" mpls lsp "PE2-to-PE1-Flex-Algo-128" max-sr-labels label-stack-size 3
/configure router "Base" mpls lsp "PE2-to-PE1-Flex-Algo-128" max-sr-labels additional-frr-labels 2
/configure router "Base" mpls lsp "PE2-to-PE1-Flex-Algo-128" primary "Flex-Algo-128" admin-state enable
/configure router "Base" rsvp interface "system" admin-state enable
/configure router "Base" rsvp interface "to_P3" admin-state enable
/configure router "Base" rsvp interface "to_P4" admin-state enable
/configure routing-options ip-fast-reroute true
/configure routing-options flexible-algorithm-definitions flex-algo "Flex-Algo-128_TE-metric" admin-state enable
/configure routing-options flexible-algorithm-definitions flex-algo "Flex-Algo-128_TE-metric" description "Use TE-Metrics. Assimetric paths: PE1->P3->P4->PE2 & PE2->P3->PE1"
/configure routing-options flexible-algorithm-definitions flex-algo "Flex-Algo-128_TE-metric" metric-type te-metric
/configure routing-options if-attribute admin-group "bottom" value 2
/configure routing-options if-attribute admin-group "midle" value 0
/configure routing-options if-attribute admin-group "top" value 1
/configure service customer "1"
/configure service epipe "1001" admin-state enable
/configure service epipe "1001" description "Epipe using ISIS 1 best IGP metric on Top"
/configure service epipe "1001" customer "1"
/configure service epipe "1001" service-mtu 8100
/configure service epipe "1001" spoke-sdp 1111:1001 admin-state enable
/configure service epipe "1001" sap 1/1/c3/1:1001 admin-state enable
/configure service sdp 111 admin-state enable
/configure service sdp 111 description "To R1 - VPRN using ISIS 0 with Flex-Algo"
/configure service sdp 111 delivery-type mpls
/configure service sdp 111 far-end ip-address 10.0.0.1
/configure service sdp 111 lsp "PE2-to-PE1-Flex-Algo-128"
/configure service sdp 1111 admin-state enable
/configure service sdp 1111 description "To R1 - Epipe using ISIS 1 best IGP metric on Top"
/configure service sdp 1111 delivery-type mpls
/configure service sdp 1111 sr-isis true
/configure service sdp 1111 far-end ip-address 10.0.0.11
/configure service sdp 2111 admin-state enable
/configure service sdp 2111 description "To R1 - SERV using ISIS 2 best IGP metric on Bottom"
/configure service sdp 2111 delivery-type mpls
/configure service sdp 2111 sr-isis true
/configure service sdp 2111 far-end ip-address 10.0.0.12
/configure service epipe "1002" admin-state enable
/configure service epipe "1002" description "SERV using ISIS 2 best IGP metric on Bottom"
/configure service epipe "1002" customer "1"
/configure service epipe "1002" service-mtu 8100
/configure service epipe "1002" spoke-sdp 2111:1002 admin-state enable
/configure service epipe "1002" spoke-sdp 2111:1002 anysec-encryption-group "EG_SERV-1002"
/configure service epipe "1002" sap 1/1/c3/1:1002 admin-state enable
/configure service vprn "1003" admin-state enable
/configure service vprn "1003" description "VPRN using ISIS 0 with Flex-Algo"
/configure service vprn "1003" customer "1"
/configure service vprn "1003" autonomous-system 65000
/configure service vprn "1003" router-id 10.0.0.2
/configure service vprn "1003" bgp-ipvpn mpls admin-state enable
/configure service vprn "1003" bgp-ipvpn mpls route-distinguisher "10.0.0.2:1003"
/configure service vprn "1003" bgp-ipvpn mpls vrf-target community "target:65000:1003"
/configure service vprn "1003" bgp-ipvpn mpls vrf-import policy ["vrf_1003_import"]
/configure service vprn "1003" bgp-ipvpn mpls auto-bind-tunnel resolution filter
/configure service vprn "1003" bgp-ipvpn mpls auto-bind-tunnel allow-flex-algo-fallback true
/configure service vprn "1003" bgp-ipvpn mpls auto-bind-tunnel resolution-filter sr-isis true
/configure service vprn "1003" bgp-ipvpn mpls auto-bind-tunnel resolution-filter sr-te true
/configure service vprn "1003" interface "loopback" loopback true
/configure service vprn "1003" interface "loopback" ipv4 primary address 10.0.0.2
/configure service vprn "1003" interface "loopback" ipv4 primary prefix-length 32
/configure service vprn "1003" interface "to_client8" admin-state enable
/configure service vprn "1003" interface "to_client8" description "To client8 via CE6"
/configure service vprn "1003" interface "to_client8" ipv4 primary address 192.168.63.1
/configure service vprn "1003" interface "to_client8" ipv4 primary prefix-length 24
/configure service vprn "1003" interface "to_client8" sap 1/1/c3/1:1003 admin-state enable
/configure system name "pe2"
```

With the Provider Edge routers complete, we will move on to the final two routers, the Consumer Edge routers. Their configuration is relatively simple, consisting of the MACsec connection to the PEs, the services and ports used by each service, and the routing configuration.

We will first configure CE5:

```srl
/configure macsec connectivity-association "CA_MACSec1" admin-state enable
/configure macsec connectivity-association "CA_MACSec1" description "MACSec CE/PE"
/configure macsec connectivity-association "CA_MACSec1" macsec-encrypt true
/configure macsec connectivity-association "CA_MACSec1" clear-tag-mode none
/configure macsec connectivity-association "CA_MACSec1" cipher-suite gcm-aes-xpn-128
/configure macsec connectivity-association "CA_MACSec1" static-cak active-psk 1
/configure macsec connectivity-association "CA_MACSec1" static-cak mka-hello-interval 5
/configure macsec connectivity-association "CA_MACSec1" static-cak pre-shared-key 1 encryption-type aes-128-cmac
/configure macsec connectivity-association "CA_MACSec1" static-cak pre-shared-key 1 cak 0123456789ABCDEF0123456789ABCDEF
/configure macsec connectivity-association "CA_MACSec1" static-cak pre-shared-key 1 cak-name "0123456789ABCDEF"
/configure macsec connectivity-association "CA_MACSec1" static-cak pre-shared-key 2 encryption-type aes-128-cmac
/configure macsec connectivity-association "CA_MACSec1" static-cak pre-shared-key 2 cak 123456789ABCDEF0123456789ABCDEF0
/configure macsec connectivity-association "CA_MACSec1" static-cak pre-shared-key 2 cak-name "123456789ABCDEF0"
/configure port 1/1/c1 admin-state enable
/configure port 1/1/c1 description "VLL"
/configure port 1/1/c1 connector breakout c1-100g
/configure port 1/1/c1/1 admin-state enable
/configure port 1/1/c1/1 description "VLL"
/configure port 1/1/c1/1 ethernet mode access
/configure port 1/1/c1/1 ethernet mtu 9000
/configure port 1/1/c2 admin-state enable
/configure port 1/1/c2 description "SERV"
/configure port 1/1/c2 connector breakout c1-100g
/configure port 1/1/c2/1 admin-state enable
/configure port 1/1/c2/1 description "SERV"
/configure port 1/1/c2/1 ethernet mode access
/configure port 1/1/c2/1 ethernet mtu 9000
/configure port 1/1/c3 admin-state enable
/configure port 1/1/c3 description "VPRN"
/configure port 1/1/c3 connector breakout c1-100g
/configure port 1/1/c3/1 admin-state enable
/configure port 1/1/c3/1 description "VPRN"
/configure port 1/1/c3/1 ethernet mode access
/configure port 1/1/c3/1 ethernet mtu 9000
/configure port 1/1/c7 admin-state enable
/configure port 1/1/c7 connector breakout c1-100g
/configure port 1/1/c7/1 admin-state enable
/configure port 1/1/c7/1 description "to_PE1"
/configure port 1/1/c7/1 ethernet mode access
/configure port 1/1/c7/1 ethernet encap-type dot1q
/configure port 1/1/c7/1 ethernet mtu 9000
/configure port 1/1/c7/1 ethernet dot1x admin-state enable
/configure port 1/1/c7/1 ethernet dot1x tunneling false
/configure port 1/1/c7/1 ethernet dot1x tunnel-dot1q false
/configure port 1/1/c7/1 ethernet dot1x tunnel-qinq false
/configure port 1/1/c7/1 ethernet dot1x macsec sub-port 1 admin-state enable
/configure port 1/1/c7/1 ethernet dot1x macsec sub-port 1 ca-name "CA_MACSec1"
/configure port 1/1/c7/1 ethernet dot1x macsec sub-port 1 max-peers 5
/configure port 1/1/c7/1 ethernet dot1x macsec sub-port 1 encap-match all-match true
/configure router "Base" autonomous-system 65005
/configure router "Base" ecmp 2
/configure router "Base" router-id 10.0.0.5
/configure router "Base" interface "system" ipv4 primary address 10.0.0.5
/configure router "Base" interface "system" ipv4 primary prefix-length 32
/configure service customer "1"
/configure service epipe "1001" admin-state enable
/configure service epipe "1001" description "Client7 to Client 8 - 192.168.1.0/24"
/configure service epipe "1001" customer "1"
/configure service epipe "1001" service-mtu 8100
/configure service epipe "1001" sap 1/1/c1/1 admin-state enable
/configure service epipe "1001" sap 1/1/c7/1:1001 admin-state enable
/configure service epipe "1002" admin-state enable
/configure service epipe "1002" description "Client7 to Client 8 - 192.168.2.0/24"
/configure service epipe "1002" customer "1"
/configure service epipe "1002" service-mtu 8100
/configure service epipe "1002" sap 1/1/c2/1 admin-state enable
/configure service epipe "1002" sap 1/1/c7/1:1002 admin-state enable
/configure service epipe "1003" admin-state enable
/configure service epipe "1003" description "Client7 - 192.168.3.0/24 to Client 8 - 1.1.1.0/24"
/configure service epipe "1003" customer "1"
/configure service epipe "1003" service-mtu 8100
/configure service epipe "1003" sap 1/1/c3/1 admin-state enable
/configure service epipe "1003" sap 1/1/c7/1:1003 admin-state enable
/configure system name "ce5"
```

As we can see, this configuration is much simpler than that of the PEs. It configures the CA used to protect traffic to the PEs, sets up the ports for each service and for MACsec, and finally assigns the services to their respective ports.

The configuration for CE6 is very similar:

```srl
/configure macsec connectivity-association "CA_MACSec1" admin-state enable
/configure macsec connectivity-association "CA_MACSec1" description "MACSec CE/PE"
/configure macsec connectivity-association "CA_MACSec1" macsec-encrypt true
/configure macsec connectivity-association "CA_MACSec1" clear-tag-mode none
/configure macsec connectivity-association "CA_MACSec1" cipher-suite gcm-aes-xpn-128
/configure macsec connectivity-association "CA_MACSec1" static-cak active-psk 1
/configure macsec connectivity-association "CA_MACSec1" static-cak mka-hello-interval 5
/configure macsec connectivity-association "CA_MACSec1" static-cak pre-shared-key 1 encryption-type aes-128-cmac
/configure macsec connectivity-association "CA_MACSec1" static-cak pre-shared-key 1 cak 0123456789ABCDEF0123456789ABCDEF
/configure macsec connectivity-association "CA_MACSec1" static-cak pre-shared-key 1 cak-name "0123456789ABCDEF"
/configure macsec connectivity-association "CA_MACSec1" static-cak pre-shared-key 2 encryption-type aes-128-cmac
/configure macsec connectivity-association "CA_MACSec1" static-cak pre-shared-key 2 cak 123456789ABCDEF0123456789ABCDEF0
/configure macsec connectivity-association "CA_MACSec1" static-cak pre-shared-key 2 cak-name "123456789ABCDEF0"
/configure port 1/1/c1 admin-state enable
/configure port 1/1/c1 description "VLL"
/configure port 1/1/c1 connector breakout c1-100g
/configure port 1/1/c1/1 admin-state enable
/configure port 1/1/c1/1 description "VLL"
/configure port 1/1/c1/1 ethernet mode access
/configure port 1/1/c1/1 ethernet mtu 9000
/configure port 1/1/c2 admin-state enable
/configure port 1/1/c2 description "SERV"
/configure port 1/1/c2 connector breakout c1-100g
/configure port 1/1/c2/1 admin-state enable
/configure port 1/1/c2/1 description "SERV"
/configure port 1/1/c2/1 ethernet mode access
/configure port 1/1/c2/1 ethernet mtu 9000
/configure port 1/1/c3 admin-state enable
/configure port 1/1/c3 description "VPRN"
/configure port 1/1/c3 connector breakout c1-100g
/configure port 1/1/c3/1 admin-state enable
/configure port 1/1/c3/1 description "VPRN"
/configure port 1/1/c3/1 ethernet mode access
/configure port 1/1/c3/1 ethernet mtu 9000
/configure port 1/1/c7 admin-state enable
/configure port 1/1/c7 connector breakout c1-100g
/configure port 1/1/c7/1 admin-state enable
/configure port 1/1/c7/1 description "to_PE2"
/configure port 1/1/c7/1 ethernet mode access
/configure port 1/1/c7/1 ethernet encap-type dot1q
/configure port 1/1/c7/1 ethernet mtu 9000
/configure port 1/1/c7/1 ethernet dot1x admin-state enable
/configure port 1/1/c7/1 ethernet dot1x tunneling false
/configure port 1/1/c7/1 ethernet dot1x tunnel-dot1q false
/configure port 1/1/c7/1 ethernet dot1x tunnel-qinq false
/configure port 1/1/c7/1 ethernet dot1x macsec sub-port 1 admin-state enable
/configure port 1/1/c7/1 ethernet dot1x macsec sub-port 1 ca-name "CA_MACSec1"
/configure port 1/1/c7/1 ethernet dot1x macsec sub-port 1 max-peers 5
/configure port 1/1/c7/1 ethernet dot1x macsec sub-port 1 encap-match all-match true
/configure router "Base" autonomous-system 65006
/configure router "Base" ecmp 2
/configure router "Base" router-id 10.0.0.6
/configure router "Base" interface "system" ipv4 primary address 10.0.0.6
/configure router "Base" interface "system" ipv4 primary prefix-length 32
/configure service customer "1"
/configure service epipe "1001" admin-state enable
/configure service epipe "1001" description "Client7 to Client 8 - 192.168.1.0/24"
/configure service epipe "1001" customer "1"
/configure service epipe "1001" service-mtu 8100
/configure service epipe "1001" sap 1/1/c1/1 admin-state enable
/configure service epipe "1001" sap 1/1/c7/1:1001 admin-state enable
/configure service epipe "1002" admin-state enable
/configure service epipe "1002" description "Client7 to Client 8 - 192.168.2.0/24"
/configure service epipe "1002" customer "1"
/configure service epipe "1002" service-mtu 8100
/configure service epipe "1002" sap 1/1/c2/1 admin-state enable
/configure service epipe "1002" sap 1/1/c7/1:1002 admin-state enable
/configure service epipe "1003" admin-state enable
/configure service epipe "1003" description "Client7 - 192.168.3.0/24 to Client 8 - 1.1.1.0/24"
/configure service epipe "1003" customer "1"
/configure service epipe "1003" service-mtu 8100
/configure service epipe "1003" sap 1/1/c3/1 admin-state enable
/configure service epipe "1003" sap 1/1/c7/1:1003 admin-state enable
/configure system name "ce6"
```

With all the machines configured, we will now quickly verify that everything is running as intended.

Start the laboratory, then access PE1 and PE2 and verify that ANYsec is working properly using the following commands:

```srl
show anysec tunnel-encryption
show anysec service-encryption
```

For the output of the first command, you should see both of the services that are using tunnel encryption. It should be similar to Figures 2 and 3.

<figure markdown id="figure-2">
  ![Figure 2: Tunnel Encryption Information for VLL](../images/ANYTUNVLL.png){width="500"}
  <figcaption>Figure 2: Tunnel Encryption Information for VLL</figcaption>
</figure>

<figure markdown id="figure-3">
  ![Figure 3: Tunnel Encryption Information for VPRN](../images/ANYTUNVPRN.png){width="500"}
  <figcaption>Figure 3: Tunnel Encryption Information for VPRN</figcaption>
</figure>

For the second command, you should see the single service using service encryption. The output should be similar to Figure 4:

<figure markdown id="figure-4">
  ![Figure 4: Service Encryption Information for VPLS](../images/ANYSERVVPLS.png){width="500"}
  <figcaption>Figure 4: Service Encryption Information for VPLS</figcaption>
</figure>

To verify this in practice, access Host 1 and ping the following three addresses:

```bash
ping 192.168.51.8
ping 192.168.52.8
ping 192.168.63.8
```

These pings target the services on Host 2, using all three ports, different ANYsec encryption methods, and different paths. If all of them succeed, the laboratory is properly set up and ready for our [Experiments](experiments.md).
