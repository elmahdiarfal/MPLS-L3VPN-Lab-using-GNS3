# MPLS-L3VPN Lab — GNS3 🛠️

**Author:** El Mahdi ARFAL  
**Academic Year:** 2025/2026  
**Institution:** Institut National des Postes et Télécommunications  

A hands-on GNS3 lab implementing an MPLS Layer-3 VPN (L3VPN) using Cisco IOSv (vios-adventerprisek9). It demonstrates core Service Provider (SP) building blocks: OSPF IGP, LDP, MPLS forwarding, VRFs on PE devices, and MP-BGP (vpnv4) to exchange customer routes. Customer Edge (CE) routers run OSPF with their attached PE.

---

## 🚀 Project summary

This lab shows how to configure an MPLS L3VPN with:

- OSPF as SP IGP  
- LDP label distribution and MPLS forwarding  
- VRFs on Provider Edge (PE) routers  
- MP-BGP vpnv4 peering between PEs  
- Redistribution between OSPF and BGP to ensure full VPN reachability  
- End-to-end verification with `ping vrf` and `show` commands

---

## ✅ Objectives / Learning outcomes

- Configure SP IGP (OSPF) across PE and P routers  
- Enable LDP for label distribution and MPLS data plane  
- Create VRF instances on PEs and attach CE links  
- Configure MP-BGP vpnv4 between PEs to exchange VPN routes  
- Redistribute between OSPF and BGP for L3VPN reachability  
- Verify MPLS and VPN configurations with show commands and pings

---

## 🌐 Topology overview

![Topology](./Topology.png)

- **SP Network**:  
  - `SP1-PE` — `SP2-P` — `SP3-PE` (linear)  
- **Customer Network (Customer-1)**:  
  - `HQ-CE` — `SP1-PE` (VRF CUSTOMER-1)  
  - `BRANCH-CE` — `SP3-PE` (VRF CUSTOMER-1)

---

## 🌍 Addressing Plan

### SP Backbone / Loopbacks

| Device   | Interface   | IP Address       |
|----------|-------------|------------------|
| SP1-PE   | Loopback0   | 172.16.1.1/32    |
| SP2-P    | Loopback0   | 172.16.0.1/32    |
| SP3-PE   | Loopback0   | 172.16.2.1/32    |

### SP Interlinks

| Link                     | IP Addresses                |
|--------------------------|----------------------------|
| SP1-PE Gi0/0 <-> SP2-P Gi0/0  | 172.16.10.2/30 <-> 172.16.10.1/30 |
| SP2-P Gi0/1 <-> SP3-PE Gi0/0  | 172.16.20.1/30 <-> 172.16.20.2/30 |

### Customer Links / CE

| Link                              | IP Addresses                |
|----------------------------------|----------------------------|
| SP1-PE Gi0/1 (VRF CUSTOMER-1) <-> HQ-CE Gi0/0 | 10.0.100.1/30 <-> 10.0.100.2/30 |
| SP3-PE Gi0/1 (VRF CUSTOMER-1) <-> BRANCH-CE Gi0/0 | 10.0.200.1/30 <-> 10.0.200.2/30 |

### Customer Loopbacks

| Device   | Loopback0 IP    |
|----------|-----------------|
| HQ-CE    | 10.0.1.1/32     |
| BRANCH-CE| 10.0.2.1/32     |

---

## 📁 Files & Repo Structure (suggested)

```
├── Topology.png
├── README.md
├── configs/
│ ├── SP1-PE.txt
│ ├── SP2-P.txt
│ ├── SP3-PE.txt
│ ├── HQ-CE.txt
│ └── BRANCH-CE.txt
└── MPLS-L3VPN.gns3project
```

---

## 🔧 Key configuration snippets

# These reflect the lab design. Full configs are inside configs/.

### SP — Common OSPF (Backbone)
```bash
router ospf 1
 network 172.16.0.0 0.0.255.255 area 0
```


### MPLS / LDP (Enable per router)
```bash
mpls ldp router-id Loopback0 force
```
```bash
interface GigabitEthernet0/0
 mpls ip
```

### VRF on PE (SP1-PE and SP3-PE)
```bash
ip vrf CUSTOMER-1
 rd 100:1
 route-target export 1:100
 route-target import 1:100
```
```bash
interface GigabitEthernet0/1
 ip vrf forwarding CUSTOMER-1
 ip address 10.0.100.1 255.255.255.252
 no shutdown
```

### MP-BGP (iBGP between PEs using Loopbacks)
```bash
router bgp 100
 neighbor 172.16.2.1 remote-as 100
 neighbor 172.16.2.1 update-source Loopback0

 address-family vpnv4
  neighbor 172.16.2.1 activate
  neighbor 172.16.2.1 send-community both
 exit-address-family
```

### VRF <-> IGP/BGP Redistribution (PE)
```bash
router ospf 10 vrf CUSTOMER-1
 network 10.0.0.0 0.0.255.255 area 0
 redistribute bgp 100 subnets
```
```bash
router bgp 100
 address-family ipv4 vrf CUSTOMER-1
  redistribute ospf 10 vrf CUSTOMER-1
 exit-address-family
```

### CE OSPF (HQ-CE / BRANCH-CE)
```bash
router ospf 1
 log-adjacency-changes
 network 10.0.0.0 0.0.255.255 area 0
```

---

## ✅ Verification Commands
```bash
show mpls ldp neighbor
show ip vrf detail
show ip route vrf CUSTOMER-1
show ip bgp vpnv4 all
```
---

## ⚙️ How to import / run in GNS3

### Method 1: Manual Setup

1. Create a new GNS3 project.  
2. Drag 5 Cisco router nodes (SP1-PE, SP2-P, SP3-PE, HQ-CE, BRANCH-CE).  
3. Apply interface mappings exactly as in `Topology.png`.  
4. Paste configs from `configs/` into each router.  
5. Start all nodes.  
6. Verify OSPF adjacencies, LDP neighbors, and MP-BGP vpnv4 peering.  
7. Test connectivity from PE routers using:

   - `ping vrf CUSTOMER-1 10.0.100.2`  
   - `ping vrf CUSTOMER-1 10.0.200.2`  
   - Verify CE-to-CE loopback reachability.

8. Check CE routers' routing tables:

   - `show ip route`

### Method 2: Portable Project Import (Recommended) 📁

You can use the portable project file included: `MPLS-L3VPN.gns3project`. Anyone with GNS3, GNS3 VM, and Cisco IOSv router image (`vios-adventerprisek9-m.spa.159-3.m9`) can:

1. Open GNS3.  
2. Go to **File > Import portable project**.  
3. Select `MPLS-L3VPN.gns3project` from the repo root.  
4. All topology, nodes, and configurations will be preloaded.  
5. Start the project and begin lab exercises immediately.

---

## ⚠️ Requirements

- **GNS3**  
- **Cisco IOSv 15.9(3)M9** (image: `vios-adventerprisek9-m.spa.159-3.m9`)

---

## 📚 Notes

This lab mirrors real Service Provider MPLS L3VPN deployments and is an excellent foundation for learning:

- VRF separation  
- MP-BGP vpnv4 route exchange  
- MPLS label switching fundamentals
