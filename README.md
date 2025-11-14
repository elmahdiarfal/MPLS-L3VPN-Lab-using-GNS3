# MPLS-L3VPN-Lab----GNS3
This repository contains a hands-on GNS3 lab implementing an MPLS Layer-3 VPN (L3VPN). 
# MPLS L3VPN Lab (GNS3) — README

## Project summary

This repository contains a hands-on GNS3 lab implementing an MPLS Layer-3 VPN (L3VPN) using Cisco IOSv (vios-adventerprisek9).
It demonstrates the core Service Provider (SP) building blocks: OSPF IGP, LDP, MPLS forwarding, VRFs on PE devices, and MP-BGP (vpnv4) to exchange customer routes. Customer edge (CE) routers run OSPF with their attached PE.

---

## Objectives / Learning outcomes

* Configure SP IGP (OSPF) across PE and P routers.
* Enable LDP for label distribution and MPLS data plane.
* Create VRF instances on PEs and attach CE links.
* Configure MP-BGP (address-family vpnv4) between PEs to exchange VPN routes.
* Redistribute between OSPF (customer) and BGP (vpnv4/VRF) to achieve full L3VPN reachability.
* Verify with `show` commands and end-to-end pings (including `ping vrf ...`).

---

## Topology (overview)

![Topology](./Topology.png)

* **SP network**

  * `SP1-PE` — `SP2-P` — `SP3-PE` (linear)
* **Customer network (Customer-1)**

  * `HQ-CE` — `SP1-PE` (VRF CUSTOMER-1)
  * `BRANCH-CE` — `SP3-PE` (VRF CUSTOMER-1)

---

## Addressing plan

### SP backbone / loopbacks

* `SP1-PE` Loopback0: `172.16.1.1/32`
* `SP2-P`  Loopback0: `172.16.0.1/32`
* `SP3-PE` Loopback0: `172.16.2.1/32`

### SP interlinks

* `SP1-PE` Gi0/0: `172.16.10.2/30` <-> `SP2-P` Gi0/0: `172.16.10.1/30`
* `SP2-P` Gi0/1: `172.16.20.1/30` <-> `SP3-PE` Gi0/0: `172.16.20.2/30`

### Customer links / CE

* `SP1-PE` Gi0/1 (vrf CUSTOMER-1): `10.0.100.1/30` <-> `HQ-CE` Gi0/0: `10.0.100.2/30`
* `SP3-PE` Gi0/1 (vrf CUSTOMER-1): `10.0.200.1/30` <-> `BRANCH-CE` Gi0/0: `10.0.200.2/30`

### Customer loopbacks

* `HQ-CE` Loopback0: `10.0.1.1/32`
* `BRANCH-CE` Loopback0: `10.0.2.1/32`

---

## Files & repo structure (suggested)

```
.
├── Topology.png
├── README.md
├── configs/
│   ├── SP1-PE.cfg
│   ├── SP2-P.cfg
│   ├── SP3-PE.cfg
│   ├── HQ-CE.cfg
│   └── BRANCH-CE.cfg
└── gns3-project/
```

---

## Key configuration snippets

> These snippets reflect the lab design. Place full configs inside `configs/`.

### SP — common OSPF (backbone)

```
router ospf 1
 network 172.16.0.0 0.0.255.255 area 0
```

### MPLS / LDP (enable per router)

```
mpls ldp router-id Loopback0 force

interface GigabitEthernet0/0
 mpls ip
```

### VRF on PE (SP1-PE and SP3-PE)

```
ip vrf CUSTOMER-1
 rd 100:1
 route-target export 1:100
 route-target import 1:100

interface GigabitEthernet0/1
 ip vrf forwarding CUSTOMER-1
 ip address 10.0.100.1 255.255.255.252
 no shutdown
```

### MP-BGP (iBGP between PEs using loopbacks)

```
router bgp 100
 neighbor 172.16.2.1 remote-as 100
 neighbor 172.16.2.1 update-source Loopback0

 address-family vpnv4
  neighbor 172.16.2.1 activate
  neighbor 172.16.2.1 send-community both
 exit-address-family
```

### VRF <-> IGP/BGP redistribution (PE)

```
router ospf 10 vrf CUSTOMER-1
 network 10.0.0.0 0.0.255.255 area 0
 redistribute bgp 100 subnets

router bgp 100
 address-family ipv4 vrf CUSTOMER-1
  redistribute ospf 10 vrf CUSTOMER-1
 exit-address-family
```

### CE OSPF (HQ-CE / BRANCH-CE)

```
router ospf 1
 log-adjacency-changes
 network 10.0.0.0 0.0.255.255 area 0
```

---

## Verification commands

```
show mpls ldp neighbor
show ip vrf detail
show ip route vrf CUSTOMER-1
show ip bgp vpnv4 all
```

---

## How to import / run in GNS3

1. Create a new GNS3 project.
2. Drag 5 Cisco router nodes (SP1-PE, SP2-P, SP3-PE, HQ-CE, BRANCH-CE).
3. Apply the interface mappings exactly as in `Topology.png`.
4. Paste configs from `configs/` into each router.
5. Start all nodes.
6. Verify OSPF adjacencies, LDP neighbors, and MP-BGP vpnv4 peering.
7. Test connectivity from PE Routers:

   * `ping vrf CUSTOMER-1 10.0.100.2`
   * `ping vrf CUSTOMER-1 10.0.200.2`
   * CE-to-CE loopback reachability.
  
8. Chack CE Routers routing tables:

   * `show ip route`

---

## Requirements

* **GNS3**
* **Cisco IOSv 15.9(3)M9** (image: `vios-adventerprisek9-m.spa.159-3.m9`)

---

## Notes

This lab mirrors real SP MPLS L3VPN deployments and is a solid foundation for:

* Understanding VRF separation
* MP-BGP vpnv4 route exchange
* MPLS label switching fundamentals
