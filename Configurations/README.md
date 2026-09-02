
# Configurations

Detailed device-by-device configuration and verification output for the **Gaming Center: Complete Network Design** lab. Each section below documents what was configured on a device and shows the corresponding `show` command output from Packet Tracer.

> Screenshots referenced here live in [`screenshots/`](./screenshots).

---

## Access-SW1 & Access-SW2 (Cisco 2960-24TT)

Both access switches carry the same VLAN set and connect to the core over a 2-port LACP EtherChannel trunk.

### VLAN Database

```
vlan 10
 name GAMING
vlan 20
 name STAFF
vlan 30
 name GUEST_WIFI
vlan 40
 name SERVICES
vlan 99
 name MANAGEMENT
```

**Port assignments**

| VLAN | Name | Ports |
|---|---|---|
| 10 | GAMING | Fa0/3–Fa0/6 |
| 20 | STAFF | Fa0/21 |
| 30 | GUEST_WIFI | Fa0/11 |
| 40 | SERVICES | (none assigned) |
| 99 | MANAGEMENT | (none assigned) |

![VLAN brief - Access-SW1](screenshots/01_VLAN_ACCESS-SW1.png)
![VLAN brief - Access-SW2](screenshots/02_VLAN_ACCESS-SW2.png)

### EtherChannel (LACP) to Core

```
interface range FastEthernet0/1-2
 channel-group 1 mode active
 switchport mode trunk
!
interface Port-channel1
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,40,99
```
*(Access-SW2 mirrors this with `channel-group 2` / `Port-channel2`.)*

Both uplinks bundle Fa0/1 and Fa0/1/2 into a single logical LACP port-channel, giving the access layer redundant, load-balanced connectivity to L3-SW1.

![EtherChannel summary - Access-SW1](screenshots/05_ETHERCHANNEL_ACCESS-SW1.png)
![EtherChannel summary - Access-SW2](screenshots/06_ETHERCHANNEL_ACCESS-SW2.png)

### Trunk Configuration

```
interface Port-channel1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,40,99
 switchport trunk native vlan 1
```

The trunk carries all five VLANs (10, 20, 30, 40, 99) with native VLAN 1, confirmed active and forwarding for all of them in spanning tree.

![Trunk status - Access-SW1](screenshots/03_TRUNK_ACCESS-SW1.png)
![Trunk status - Access-SW2](screenshots/04_TRUNK_ACCESS-SW2.png)

---

## L3-SW1 (Cisco 3560-24PS) — Routed Core

L3-SW1 is the routing backbone of the network: it terminates both access-layer EtherChannels, hosts the SVI (gateway) for every VLAN, runs DHCP for the user VLANs, and applies the security ACLs.

### EtherChannel — Both Access Uplinks

```
interface range FastEthernet0/1-2
 channel-group 1 mode active
 switchport mode trunk
!
interface range FastEthernet0/3-4
 channel-group 2 mode active
 switchport mode trunk
```

Two independent LACP port-channels are formed: **Po1** to Access-SW1 and **Po2** to Access-SW2.

![EtherChannel summary - L3-SW1](screenshots/07_ETHERCHANNEL_L3-SW1.png)

### SVI (Inter-VLAN Gateway) Configuration

```
interface Vlan10
 description GAMING_GATEWAY
 mac-address 00e0.a354.6401
 ip address 192.168.10.1 255.255.255.0
 ip access-group SECURE_VLAN99 in
!
interface Vlan20
 description STAFF_GATEWAY
 mac-address 00e0.a354.6402
 ip address 192.168.20.1 255.255.255.0
 ip access-group SECURE_VLAN99 in
!
interface Vlan30
 description GUEST_WIFI_GATEWAY
 mac-address 00e0.a354.6403
 ip address 192.168.30.1 255.255.255.0
 ip access-group GUEST_INBOUND_ACL in
!
interface Vlan40
 ip address 192.168.40.1 255.255.255.0
!
interface Vlan99
 ip address 192.168.99.1 255.255.255.0
```

![SVI addresses - L3-SW1](screenshots/08_SVI_L3-SW1.png)

### Routing

```
ip route 0.0.0.0 0.0.0.0 10.10.10.1
```

L3-SW1 connects to EDGE-R1 over `10.10.10.0/30` (GigabitEthernet0/1) and sends all non-local traffic to it via a default static route. All five VLAN subnets plus the transit link show as directly connected.

![IP route table - L3-SW1](screenshots/09_ROUTES_L3-SW1.png)

### DHCP Services

```
ip dhcp excluded-address 192.168.10.1 192.168.10.10
ip dhcp excluded-address 192.168.20.1 192.168.20.10
ip dhcp excluded-address 192.168.30.1 192.168.30.10
!
ip dhcp pool GAMING
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.1
 dns-server 8.8.8.8
!
ip dhcp pool STAFF
 network 192.168.20.0 255.255.255.0
 default-router 192.168.20.1
 dns-server 8.8.8.8
!
ip dhcp pool GUEST_WIFI
 network 192.168.30.0 255.255.255.0
 default-router 192.168.30.1
 dns-server 8.8.8.8
```

DHCP is scoped to the three user-facing VLANs (Gaming, Staff, Guest Wi-Fi). Each pool excludes its first 10 addresses for static/infrastructure use — e.g. Staff PC1 is statically pinned to `192.168.20.50`. Services (VLAN 40) and Management (VLAN 99) are not DHCP-served.

![DHCP configuration - L3-SW1](screenshots/10_DHCP_L3-SW1.png)

### Security ACLs

```
ip access-list extended SECURE_VLAN99
 deny ip any 192.168.99.0 0.0.0.255
 permit ip any any
!
ip access-list extended GUEST_INBOUND_ACL
 deny ip 192.168.30.0 0.0.0.255 192.168.10.0 0.0.0.255
 deny ip 192.168.30.0 0.0.0.255 192.168.20.0 0.0.0.255
 deny ip 192.168.30.0 0.0.0.255 192.168.40.0 0.0.0.255
 deny ip 192.168.30.0 0.0.0.255 192.168.99.0 0.0.0.255
 permit ip any any
```

**SECURE_VLAN99** — applied inbound on the Gaming and Staff SVIs. Blocks those VLANs from reaching the 192.168.99.0/24 management network while permitting everything else.

**GUEST_INBOUND_ACL** — applied inbound on the Guest Wi-Fi SVI. Denies Guest Wi-Fi traffic to Gaming, Staff, Services, *and* Management, while still permitting outbound internet access.

![ACL definitions - L3-SW1](screenshots/11_ACL_L3-SW1.png)
![ACLs applied to SVIs - L3-SW1](screenshots/12_ACL-configured-in-VLAN10_20_30_L3-SW1.png)

---

## EDGE-R1 — Internet Edge & NAT

EDGE-R1 sits between the internal network (via L3-SW1) and the ISP, and is responsible for routing and PAT/NAT overload.

### Routing

```
ip route 192.168.0.0 255.255.0.0 10.10.10.2
ip route 0.0.0.0 0.0.0.0 10.10.20.1
```

- `Gi0/1` — `10.10.10.1/30`, transit link to L3-SW1
- `Gi0/0` — `10.10.20.2/30`, WAN link to ISP-R1
- Static route sends the full internal `192.168.0.0/16` supernet back through L3-SW1; default route points to the ISP (gateway of last resort `10.10.20.1`).

![IP route table - EDGE-R1](screenshots/13_ROUTS_EDGE-R1.png)

### NAT / PAT (Overload)

```
ip nat inside source list NAT_TRAFFIC interface GigabitEthernet0/0 overload
!
ip access-list extended NAT_TRAFFIC
 permit ip 192.168.0.0 0.0.255.255 any
```

All traffic sourced from `192.168.0.0/16` (i.e. every internal VLAN) is translated (PAT/overload) to EDGE-R1's WAN interface address before leaving toward the ISP.

![NAT configuration - EDGE-R1](Images/14_NAT_EDGE-R1.png)

**Live NAT translations** (ICMP test traffic from Gaming VLAN hosts to the ISP router, confirming outbound NAT is functioning):

![NAT translation table - EDGE-R1](screenshots/15_NAT-translation_EDGE-R1.png)

---

## Verification Summary

| Command | Purpose | Ran On |
|---|---|---|
| `show vlan brief` | Confirm VLAN creation & port membership | Access-SW1, Access-SW2 |
| `show interfaces trunk` | Confirm trunk state & allowed VLANs | Access-SW1, Access-SW2 |
| `show etherchannel summary` | Confirm LACP bundle formed (`SU` state) | Access-SW1, Access-SW2, L3-SW1 |
| `show ip route` | Confirm SVIs, transit links, and default route | L3-SW1, EDGE-R1 |
| `show ip nat translations` | Confirm PAT overload is translating internal traffic | EDGE-R1 |
