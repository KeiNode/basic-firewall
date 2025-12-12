🎉 Router1 & Router2 Network Infrastructure Guide
Production-Ready MikroTik + Cisco Topology (Interactive & Easy to Understand)
📘 Daftar Isi

1. Overview

2. Router1 — Internet Gateway

2.1 Firewall Flow Diagram

2.2 Filter Rules

2.3 NAT

2.4 Hardening

3. Router2 — VLAN Distribution

4. Cisco Catalyst Configuration

5. Public IP Mode

6. Checklist

7. Future-Ready ISP Practices

8. Diagram Topologi

1. Overview

Dokumentasi ini menjelaskan:

Router1 = Gateway Internet (Security + NAT)

Router2 = Distribution Router (VLAN + DHCP)

Switch Cisco = Access + Trunk VLAN

Fokus utama:

✔ Firewall best-practice
✔ Pembatasan akses management
✔ Pemisahan VLAN
✔ Inter-VLAN security
✔ Mode IP Public (optional)

Cocok untuk:

SOHO

Lab training

Small ISP

Production environment ringan–menengah

2. Router1 — Internet Gateway

Router ini:

Menjadi titik keluar masuk Internet

Melakukan NAT

Melindungi seluruh jaringan dari serangan luar

Mengontrol akses management router

2.1 Firewall Flow Diagram
            +---------------------------+
            | INPUT CHAIN               |
            |---------------------------|
        ┌──→| 1. Accept established     |
        │   | 2. Drop invalid           |
        │   | 3. Accept mgmt (ether4)   |
        │   | 4. Drop WAN input         |
        │   | 5. Drop all               |
        │   +---------------------------+
Incoming │
Traffic │
        │   +---------------------------+
        └──→| FORWARD CHAIN             |
            |---------------------------|
            | 1. Accept established     |
            | 2. Drop invalid           |
            | 3. LAN → WAN (allowed)    |
            | 4. WAN → LAN (drop)       |
            | 5. Drop all               |
            +---------------------------+

2.2 Filter Rules (Urut Wajib dari Atas ke Bawah)
1) Accept established, related (input)
chain=input connection-state=established,related action=accept

2) Accept established, related (forward)
chain=forward connection-state=established,related action=accept

3) Drop invalid (input & forward)
chain=input connection-state=invalid action=drop
chain=forward connection-state=invalid action=drop

4) Allow Management dari Router2 (MGMT)

✨ Port Winbox/SSH dapat kamu ganti sesuka hati, misal Winbox = 1983

chain=input in-interface=ether4 protocol=tcp dst-port=1983,22 action=accept comment="Allow mgmt from Router2"

5) Drop Semua Input dari WAN
chain=input in-interface=ether1 action=drop comment="Drop WAN access"

6) Allow LAN → WAN
chain=forward in-interface=ether4 out-interface=ether1 action=accept

7) Drop WAN → LAN
chain=forward in-interface=ether1 out-interface=ether4 action=drop

8) Drop All (default deny)
chain=input action=drop
chain=forward action=drop

2.3 NAT
/ip firewall nat
add chain=src-nat out-interface=ether1 action=masquerade

2.4 Hardening Router1
Disable services yang tidak dipakai:
/ip service disable telnet
/ip service disable ftp
/ip service disable www
/ip service disable api

Ganti port Winbox
/ip service set winbox port=1983

Disable MAC-Winbox (wajib demi keamanan)
/tool mac-server set allowed-interface-list=none
/tool mac-server mac-winbox set allowed-interface-list=none

3. Router2 — Distribution Router

Router ini:

Menyediakan VLAN

Menjadi gateway per VLAN

Memberikan DHCP

Melakukan isolasi antar VLAN

Memungkinkan trunk ke Cisco

3.1 VLAN Creation
/interface vlan
add name=vlan10-lan vlan-id=10 interface=ether3
add name=vlan20-guest vlan-id=20 interface=ether3
add name=vlan99-mgmt vlan-id=99 interface=ether3

3.2 IP Address per VLAN
/ip address
add address=192.168.10.1/24 interface=vlan10-lan
add address=192.168.20.1/24 interface=vlan20-guest
add address=192.168.99.2/24 interface=vlan99-mgmt

3.3 DHCP Server per VLAN
/ip pool add name=pool10 ranges=192.168.10.10-192.168.10.200
/ip pool add name=pool20 ranges=192.168.20.10-192.168.20.200

/ip dhcp-server add name=dhcp10 interface=vlan10-lan address-pool=pool10
/ip dhcp-server add name=dhcp20 interface=vlan20-guest address-pool=pool20

3.4 Firewall Router2 (Segmentasi VLAN)
Allow LAN → Internet
chain=forward src-address=192.168.10.0/24 out-interface=ether1 action=accept

Block Guest → LAN
chain=forward in-interface=vlan20-guest out-interface=vlan10-lan action=drop

Allow MGMT VLAN → Router/Switch
chain=input in-interface=vlan99-mgmt protocol=tcp dst-port=1983,22 action=accept

Drop others
chain=forward action=drop

4. Cisco Catalyst Configuration
Trunk port ke Router2
interface GiX/Y
 switchport mode trunk
 switchport trunk encapsulation dot1q
 switchport trunk allowed vlan 10,20,99
 spanning-tree portfast trunk

Access port contoh VLAN10
interface GiX/Z
 switchport mode access
 switchport access vlan 10
 switchport port-security
 switchport port-security maximum 2
 switchport port-security violation restrict

Management VLAN
interface Vlan99
 ip address 192.168.99.3 255.255.255.0

5. Public IP Mode
Mode 1 — Public di Router1 (paling mudah)

gunakan NAT

bisa dst-nat ke server internal

Mode 2 — Public Routed ke Router2

ISP route subnet → Router1 → Router2

Router2 bisa assign public ke server

Tidak NAT, lebih “ISP-grade”

6. Checklist
Router1

 established/related accept

 invalid drop

 mgmt only from ether4

 drop WAN input

 allow LAN → WAN

 drop WAN → LAN

 NAT masquerade

 disable service unnecessary

 change winbox port

 disable MAC winbox

Router2

 VLAN created

 DHCP per VLAN

 Guest→LAN blocked

 MGMT VLAN allowed

 Trunk to Cisco

7. Future-Ready ISP Practices

VRRP failover

BGP multi-homing

Syslog server + SNMP monitoring

Port security + Storm control

Automation (Ansible / scripts)

Centralized authentication (RADIUS/TACACS+)

DDoS scrubbing

8. Diagram Topologi
              INTERNET
                   │
                [Router1]
          ether1 ─ WAN / NAT
          ether4 ─ MGMT link → Router2
                   │
                [Router2]
         VLAN10 ───┤─── Clients LAN
         VLAN20 ───┤─── Guest
         VLAN99 ───┤─── Management
                   │
              [Cisco Switch]
         ├── access VLAN10
         ├── access VLAN20
         └── access VLAN99
