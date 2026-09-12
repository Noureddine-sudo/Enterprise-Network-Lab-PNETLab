# Enterprise Network Architecture & FortiGate Security Edge Lab

A complete end-to-end enterprise network topology deployed and verified in **PNETLab**. This project demonstrates a resilient, high-availability campus network integrating Cisco Layer 2/3 switching, HSRP default gateway redundancy, OSPF dynamic routing, a **Fortinet FortiGate Active-Passive HA Cluster configured via Web GUI**, and an ISP edge router running **Port Address Translation (PAT Overload)**.

---

##  Architecture Overview

The multi-tier hierarchical enterprise network is divided into four distinct operational zones:

1. **Access Layer (L2):** Cisco vIOS switches (`Switch12`, `Switch13`, `Switch14`) providing local VLAN access (`VLAN 10` for Sales, `VLAN 20` for HR, `VLAN 30` for IT) with 802.1Q trunking to the distribution layer.
2. **Distribution / Core Layer (L3):** Cisco vIOS switches (`DSW1` and `DSW2`) operating as the internal routing backbone. Interconnected via L2 EtherChannels with Switch Virtual Interfaces (SVIs) and **HSRP** for default gateway redundancy, running **OSPF Area 0** to advertise internal networks.
3. **Security Edge Layer (Firewall):** Dual FortiGate VMs (`Fortinet3` and `Fortinet4`) configured in **Active-Passive High Availability (HA)** mode using the FortiOS Web GUI. Performs stateful firewall inspection, access control, and internal-to-external Source NAT.
4. **WAN / ISP Edge Layer:** Cisco `vIOS5` router acting as the internet service provider gateway, translating internal subnets to the public internet .

---

## IP Addressing Scheme

### VLANs

- **VLAN 10 – SALES:** `10.10.10.0/24`
  - VPC11, VPC6
  - Default Gateway: `10.10.10.1`

- **VLAN 20 – HR:** `10.10.20.0/24`
  - VPC7, VPC8
  - Default Gateway: `10.10.20.1`

- **VLAN 30 – IT:** `10.10.30.0/24`
  - VPC9 ,VPC10
  - Default Gateway: `10.10.30.1`

### FortiGate HA Cluster

- **DSW1 / DSW2:** Transit network to the FortiGate cluster
  - `10.1.1.3/29`
  - `10.1.1.4/29`

- **FortiGate3 – Primary (Fortinet 3)**
  - Port3 (LAN): `10.1.1.1/29`
  - Port2 (WAN): `203.0.113.3/29`
  - WAN Gateway: `203.0.113.1`

- **FortiGate4 – Standby(Fortinet 4)**
  - Port3 (LAN): `10.1.1.2/29`
  - Port2 (WAN): `203.0.113.4/29`
  - WAN Gateway: `203.0.113.1`

- **FortiGate HA Management**
  - FortiGate3: `192.168.28.140/24`
  - FortiGate4: `192.168.28.141/24`
  - Management Gateway: `192.168.28.1`

### ISP Router (vIOS5)

- **Gi0/0 – Primary FortiGate Link:** `203.0.113.1/29`
- **Gi0/1 – Standby FortiGate Link:** `203.0.113.2/29`
- **Gi0/2 – Internet/WAN:** DHCP / Dynamic addressing
  - Gateway: ISP-provided gateway
---

##  FortiGate GUI Configuration Summary

The FortiGate cluster security controls, interfaces, routing, and High Availability were configured through the FortiOS Web Graphical User Interface (GUI):

### 1. Network Interface Setup (`Network > Interfaces`)
* **`port1` (OOB Management):** Static IP `192.168.28.140/24`, administrative access enabled for `HTTPS`, `SSH`, and `PING`.
* **`port2` (WAN Interface):** Static IP `203.0.113.3/29`, role set to **WAN**, addressing linked to ISP `vIOS5` router.
* **`port3` (LAN Interface):** Static IP `10.1.1.1/29`, role set to **LAN**, facing internal distribution switches (`DSW1`/`DSW2`).

### 2. Static Routing (`Network > Static Routes`)
* **Default Route to WAN:**
  * **Destination:** `0.0.0.0/0`
  * **Gateway IP:** `203.0.113.1`
  * **Interface:** `port2`
* **Internal Return Route:**
  * **Destination:** `10.10.0.0/16`
  * **Gateway IP:** `10.1.1.3`
  * **Interface:** `port3`

### 3. Firewall Policy & NAT (`Policy & Objects > Firewall Policy`)
Created an explicit outbound policy enabling stateful inspection and NAT:
* **Policy Name:** `LAN_TO_WAN_INTERNET`
* **Incoming Interface:** `port3` (LAN)
* **Outgoing Interface:** `port2` (WAN)
* **Source:** `all` (or `10.10.0.0/16`)
* **Destination:** `all`
* **Schedule:** `always` \| **Service:** `ALL` \| **Action:** `ACCEPT`
* **NAT:** **Enabled** (*Use Outgoing Interface Address*)

### 4. High Availability Deployment (`System > HA`)
Configured on `Fortinet3` (Primary) and mirrored to `Fortinet4` (Secondary):
* **Mode:** Active-Passive
* **Group Name:** `FW-HA-CLUSTER` \| **Group ID:** `50`
* **Priority:** `200` (Primary / `Fortinet3`), `100` (Secondary / `Fortinet4`)
* **Heartbeat Interface:** `port4` (Heartbeat Priority 50)
* **In-Band Management:** Enabled `ha-mgmt-status` mapping `port1` to preserve individual GUI management access (`192.168.28.140` and `.141`).

