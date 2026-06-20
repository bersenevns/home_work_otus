
# Итоговый проект

## Цель

Настроить сеть двух ЦОД по модели Active-Active с межсегментной фильтрацией трафика при помощи VxLAN EVPN.

Хосты в одной сети могут взаимодействовать друг с другом по L2 между ЦОД.

Выход из каждого ЦОД должен быть локальным, без использования межцодового канала.

Маршрутизация между сегментами сети должна осуществляться только через Firewall.

Проблему симметричного обратного трафика решитьс помощью SNAT на локальном Firewall.

---

## Схема

Протокол динамической маршрутизации BGP будет использоваться и в качестве underlay (eBGP), и в качестве overlay (AF evpn).

Выделено три сегмента: INSIDE (vlan 10), DMZ (vlan 20), SECURITY (vlan 30).

Vlan 10 - vni 10010, Vlan 20 - vni 10020, vrf INSIDE - vni 10100, vrf DMZ - vni 10200, vrf SECURITY - vni 10300.

VyOS будет выполнять роль FW, FW подключен к Border Leaf в каждом ЦОД. FW анонсирует дефолтный маршрут по всей фабрике (для простоты дефолт будет статический redistributed).

![схема](VXLAN_routing.png)

---

## Адресация

### Loopback-интерфейсы

| Устройство      | Loopback0       |
|------------     |-----------------|
| Spine-1-1       | 10.10.10.101/32 |
| Border-Leaf-1-1 | 10.10.10.11/32  |
| Leaf-1-2        | 10.10.10.12/32  |
| Leaf-1-3        | 10.10.10.13/32  |
| Spine-2-1       | 10.10.10.201/32 |
| Border-Leaf-2-1 | 10.10.10.21/32  |
| Leaf-2-2        | 10.10.10.22/32  |
| Leaf-2-3        | 10.10.10.23/32  |

### AS-numbering

| Устройство      | ASN   |
|-----------------|-------|
| FW-1            | 65501 |
| Spine-1-1       | 65510 |
| Border-Leaf-1-1 | 65511 |
| Leaf-1-2        | 65512 |
| Leaf-1-3        | 65513 |
| FW-2            | 65502 |
| Spine-2-1       | 65520 |
| Border-Leaf-2-1 | 65521 |
| Leaf-2-2        | 65522 |
| Leaf-2-3        | 65523 |

---

## Конфигурация

1) Настраиваем BGP-соседство в address-family ipv4
2) Настраиваем BGP-соседство с loopback0 в address-family evpn
3) Настраиваем интерфейсы VLAN 10,20,30  разных VRF
4) Настраиваем Anycast gateway на Leafs
5) Настраиваем BGP-соседство между Leaf2 и FW для передачи дефолтного маршрута в фабрику и префиксов фабрики из фабрики
6) Проверяем связанность между VRF на хостах как в одном ЦОД, так и между ЦОД

Конфигурация устройств представлена ниже, лишние строки удалены в целях читаемости.

<details>
<summary><b>Показать конфигурацию Spine-1-1</b></summary>

```bash
service routing protocols model multi-agent
!
hostname Spine-1-1
!
interface Ethernet1
   description Border-Leaf-1-1
   mtu 9214
   no switchport
   ip address 10.11.11.1/31
!
interface Ethernet2
   description Leaf-1-2
   mtu 9214
   no switchport
   ip address 10.11.12.1/31
!
interface Ethernet3
   description Leaf-1-3
   mtu 9214
   no switchport
   ip address 10.11.13.1/31
!
interface lo0
   ip add 10.10.10.101/32
!
ip routing
!
router bgp 65510
   router-id 10.10.10.101
   no bgp default ipv4-unicast
   timers bgp 3 9
   maximum-paths 4 ecmp 4
   neighbor 10.10.10.11 remote-as 65511
   neighbor 10.10.10.11 update-source Loopback0
   neighbor 10.10.10.11 description Border-Leaf-1-1
   neighbor 10.10.10.11 ebgp-multihop 3
   neighbor 10.10.10.11 send-community extended
   neighbor 10.10.10.12 remote-as 65512
   neighbor 10.10.10.12 update-source Loopback0
   neighbor 10.10.10.12 description Leaf-1-2
   neighbor 10.10.10.12 ebgp-multihop 3
   neighbor 10.10.10.12 send-community extended
   neighbor 10.10.10.13 remote-as 65513
   neighbor 10.10.10.13 update-source Loopback0
   neighbor 10.10.10.13 description Leaf-1-3
   neighbor 10.10.10.13 ebgp-multihop 3
   neighbor 10.10.10.13 send-community extended
   neighbor 10.11.11.0 remote-as 65511
   neighbor 10.11.11.0 bfd
   neighbor 10.11.11.0 description Border-Leaf-1-1
   neighbor 10.11.12.0 remote-as 65512
   neighbor 10.11.12.0 bfd
   neighbor 10.11.12.0 description Leaf-1-2
   neighbor 10.11.13.0 remote-as 65513
   neighbor 10.11.13.0 bfd
   neighbor 10.11.13.0 description Leaf-1-3
   !
   address-family evpn
      bgp next-hop-unchanged
      neighbor 10.10.10.11 activate
      neighbor 10.10.10.12 activate
      neighbor 10.10.10.13 activate
   !
   address-family ipv4
      neighbor 10.11.11.0 activate
      neighbor 10.11.12.0 activate
      neighbor 10.11.13.0 activate
      network 10.10.10.101/32
!
end

```
</details>

<details>
<summary><b>Показать конфигурацию Border-Leaf-1-1</b></summary>

```bash
service routing protocols model multi-agent
!
hostname Border-Leaf-1-1
!
vlan 10
   name INSIDE
!
vlan 101
   name FW_INSIDE
!
vlan 102
   name FW_DMZ
!
vlan 103
   name FW_SECURITY
!
vlan 20
   name DMZ
!
vlan 30
   name SECURITY
!
vrf instance DMZ
!
vrf instance INSIDE
!
vrf instance SECURITY
!
interface Port-Channel1
   mtu 9214
   no switchport
!
interface Port-Channel1.101
   encapsulation dot1q vlan 101
   vrf INSIDE
   ip address 10.11.101.0/31
!
interface Port-Channel1.102
   encapsulation dot1q vlan 102
   vrf DMZ
   ip address 10.11.102.0/31
!
interface Port-Channel1.103
   encapsulation dot1q vlan 103
   vrf SECURITY
   ip address 10.11.103.0/31
!
interface Ethernet1
   description Spine-1-1
   mtu 9214
   no switchport
   ip address 10.11.11.0/31
!
interface Ethernet4
   description Leaf-2-1
   mtu 9214
   no switchport
   ip address 10.11.21.1/31
!
interface Ethernet5
   channel-group 1 mode on
!
interface Ethernet6
   channel-group 1 mode on
!
interface Loopback0
   ip address 10.10.10.11/32
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan vlan 20 vni 10020
   vxlan vlan 30 vni 10030
   vxlan vrf DMZ vni 10200
   vxlan vrf INSIDE vni 10100
   vxlan vrf SECURITY vni 10300
!
ip routing
ip routing vrf DMZ
ip routing vrf INSIDE
ip routing vrf SECURITY
!
router bgp 65511
   router-id 10.10.10.11
   no bgp default ipv4-unicast
   timers bgp 3 9
   maximum-paths 4 ecmp 4
   neighbor 10.10.10.21 remote-as 65521
   neighbor 10.10.10.21 update-source Loopback0
   neighbor 10.10.10.21 description Leaf-2-1
   neighbor 10.10.10.21 ebgp-multihop 3
   neighbor 10.10.10.21 send-community extended
   neighbor 10.10.10.101 remote-as 65510
   neighbor 10.10.10.101 update-source Loopback0
   neighbor 10.10.10.101 description Spine-1-1
   neighbor 10.10.10.101 ebgp-multihop 3
   neighbor 10.10.10.101 send-community extended
   neighbor 10.11.11.1 remote-as 65510
   neighbor 10.11.11.1 bfd
   neighbor 10.11.11.1 description Spine-1-1
   neighbor 10.11.21.0 remote-as 65521
   neighbor 10.11.21.0 bfd
   neighbor 10.11.21.0 description Leaf-2-1
   neighbor 10.11.21.2 remote-as 65521
   neighbor 10.11.21.2 bfd
   neighbor 10.11.21.2 description Leaf-2-1
   !
   vlan 10
      rd 10.10.10.11:10
      route-target both 10:10
      redistribute learned
   !
   vlan 20
      rd 10.10.10.11:20
      route-target both 20:20
      redistribute learned
   !
   vlan 30
      rd 10.10.10.11:30
      route-target both 30:30
      redistribute learned
   !
   address-family evpn
      neighbor 10.10.10.21 activate
      neighbor 10.10.10.101 activate
   !
   address-family ipv4
      neighbor 10.11.11.1 activate
      neighbor 10.11.21.0 activate
      neighbor 10.11.21.2 activate
      network 10.10.10.11/32
   !
   vrf DMZ
      rd 10.10.10.11:200
      route-target import evpn 200:200
      route-target export evpn 200:200
      neighbor 10.11.102.1 remote-as 65501
      !
      address-family ipv4
         neighbor 10.11.102.1 activate
   !
   vrf INSIDE
      rd 10.10.10.11:100
      route-target import evpn 100:100
      route-target export evpn 100:100
      neighbor 10.11.101.1 remote-as 65501
      !
      address-family ipv4
         neighbor 10.11.101.1 activate
   !
   vrf SECURITY
      rd 10.10.10.11:300
      route-target import evpn 300:300
      route-target export evpn 300:300
      neighbor 10.11.103.1 remote-as 65501
      !
      address-family ipv4
         neighbor 10.11.103.1 activate
!
end
```
</details>

<details>
<summary><b>Показать конфигурацию Leaf-1-2</b></summary>

```bash
service routing protocols model multi-agent
!
hostname Leaf-1-2
!
vlan 10
   name INSIDE
!
vlan 20
   name DMZ
!
vlan 30
   name SECURITY
!
vrf instance DMZ
!
vrf instance INSIDE
!
vrf instance SECURITY
!
interface Ethernet1
   description Spine-1-1
   mtu 9214
   no switchport
   ip address 10.11.12.0/31
!
interface Ethernet7
   description VLAN_20
   mtu 9214
   switchport access vlan 20
!
interface Ethernet8
   description VLAN_10
   mtu 9214
   switchport access vlan 10
!
interface Loopback0
   ip address 10.10.10.12/32
!
interface Vlan10
   mtu 9214
   vrf INSIDE
   ip address virtual 192.168.10.1/24
!
interface Vlan20
   mtu 9214
   vrf DMZ
   ip address virtual 192.168.20.1/24
!
interface Vlan30
   mtu 9214
   vrf SECURITY
   ip address virtual 192.168.30.1/24
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan vlan 20 vni 10020
   vxlan vlan 30 vni 10030
   vxlan vrf DMZ vni 10200
   vxlan vrf INSIDE vni 10100
   vxlan vrf SECURITY vni 10300
!
ip virtual-router mac-address 00:00:11:11:22:22
!
ip routing
ip routing vrf DMZ
ip routing vrf INSIDE
ip routing vrf SECURITY
!
router bgp 65512
   router-id 10.10.10.12
   no bgp default ipv4-unicast
   timers bgp 3 9
   maximum-paths 4 ecmp 4
   neighbor 10.10.10.101 remote-as 65510
   neighbor 10.10.10.101 update-source Loopback0
   neighbor 10.10.10.101 description Spine-1-1
   neighbor 10.10.10.101 ebgp-multihop 3
   neighbor 10.10.10.101 send-community extended
   neighbor 10.11.12.1 remote-as 65510
   neighbor 10.11.12.1 bfd
   neighbor 10.11.12.1 description Spine-1-1
   !
   vlan 10
      rd 10.10.10.12:10
      route-target both 10:10
      redistribute learned
   !
   vlan 20
      rd 10.10.10.12:20
      route-target both 20:20
      redistribute learned
   !
   vlan 30
      rd 10.10.10.12:30
      route-target both 30:30
      redistribute learned
   !
   address-family evpn
      neighbor 10.10.10.101 activate
   !
   address-family ipv4
      neighbor 10.11.12.1 activate
      network 10.10.10.12/32
   !
   vrf DMZ
      rd 10.10.10.12:200
      route-target import evpn 200:200
      route-target export evpn 200:200
      redistribute connected
   !
   vrf INSIDE
      rd 10.10.10.12:100
      route-target import evpn 100:100
      route-target export evpn 100:100
      redistribute connected
   !
   vrf SECURITY
      rd 10.10.10.12:300
      route-target import evpn 300:300
      route-target export evpn 300:300
      redistribute connected
!
end
```
</details>

<details>
<summary><b>Показать конфигурацию Leaf-1-3</b></summary>

```bash
service routing protocols model multi-agent
!
hostname Leaf-1-3
!
vlan 10
   name INSIDE
!
vlan 20
   name DMZ
!
vlan 30
   name SECURITY
!
vrf instance DMZ
!
vrf instance INSIDE
!
vrf instance SECURITY
!
interface Ethernet1
   description Spine-1-1
   mtu 9214
   no switchport
   ip address 10.11.13.0/31
!
interface Ethernet7
   description VLAN_20
   mtu 9214
   switchport access vlan 20
!
interface Ethernet8
   description VLAN_10
   mtu 9214
   switchport access vlan 10
!
interface Loopback0
   ip address 10.10.10.13/32
!
interface Vlan10
   mtu 9214
   vrf INSIDE
   ip address virtual 192.168.10.1/24
!
interface Vlan20
   mtu 9214
   vrf DMZ
   ip address virtual 192.168.20.1/24
!
interface Vlan30
   mtu 9214
   vrf SECURITY
   ip address virtual 192.168.30.1/24
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan vlan 20 vni 10020
   vxlan vlan 30 vni 10030
   vxlan vrf DMZ vni 10200
   vxlan vrf INSIDE vni 10100
   vxlan vrf SECURITY vni 10300
!
ip virtual-router mac-address 00:00:11:11:22:22
!
ip routing
ip routing vrf DMZ
ip routing vrf INSIDE
ip routing vrf SECURITY
!
router bgp 65513
   router-id 10.10.10.13
   no bgp default ipv4-unicast
   timers bgp 3 9
   maximum-paths 4 ecmp 4
   neighbor 10.10.10.101 remote-as 65510
   neighbor 10.10.10.101 update-source Loopback0
   neighbor 10.10.10.101 description Spine-1-1
   neighbor 10.10.10.101 ebgp-multihop 3
   neighbor 10.10.10.101 send-community extended
   neighbor 10.11.13.1 remote-as 65510
   neighbor 10.11.13.1 bfd
   neighbor 10.11.13.1 description Spine-1-1
   !
   vlan 10
      rd 10.10.10.13:10
      route-target both 10:10
      redistribute learned
   !
   vlan 20
      rd 10.10.10.13:20
      route-target both 20:20
      redistribute learned
   !
   vlan 30
      rd 10.10.10.13:30
      route-target both 30:30
      redistribute learned
   !
   address-family evpn
      neighbor 10.10.10.101 activate
   !
   address-family ipv4
      neighbor 10.11.13.1 activate
      network 10.10.10.13/32
   !
   vrf DMZ
      rd 10.10.10.13:200
      route-target import evpn 200:200
      route-target export evpn 200:200
      redistribute connected
   !
   vrf INSIDE
      rd 10.10.10.13:100
      route-target import evpn 100:100
      route-target export evpn 100:100
      redistribute connected
   !
   vrf SECURITY
      rd 10.10.10.13:300
      route-target import evpn 300:300
      route-target export evpn 300:300
      redistribute connected
!
end
```
</details>

<details>
<summary><b>Показать конфигурацию FW-1</b></summary>

```bash
configure

set interfaces bonding bond0 mode xor-hash
set interfaces bonding bond0 member interface eth1
set interfaces bonding bond0 member interface eth2

set interfaces bonding bond0 vif 101 address '10.11.101.1/31'
set interfaces bonding bond0 vif 102 address '10.11.102.1/31'
set interfaces bonding bond0 vif 103 address '10.11.103.1/31'

set protocols bgp system-as '65501'

set protocols bgp neighbor 10.11.101.0 remote-as '65511'
set protocols bgp neighbor 10.11.102.0 remote-as '65511'
set protocols bgp neighbor 10.11.103.0 remote-as '65511'

set protocols static route 0.0.0.0/0 next-hop 1.1.1.1

set protocols bgp neighbor 10.11.101.0 address-family ipv4-unicast default-originate
set protocols bgp neighbor 10.11.102.0 address-family ipv4-unicast default-originate
set protocols bgp neighbor 10.11.103.0 address-family ipv4-unicast default-originate

set nat source rule 10 source address '192.168.10.0/24'
set nat source rule 10 destination address '192.168.20.0/24'
set nat source rule 10 outbound-interface name 'bond0.102'
set nat source rule 10 translation address 'masquerade'

set nat source rule 20 source address '192.168.10.0/24'
set nat source rule 20 destination address '192.168.30.0/24'
set nat source rule 20 outbound-interface name 'bond0.103'
set nat source rule 20 translation address 'masquerade'

set nat source rule 30 source address '192.168.20.0/24'
set nat source rule 30 destination address '192.168.10.0/24'
set nat source rule 30 outbound-interface name 'bond0.101'
set nat source rule 30 translation address 'masquerade'

set nat source rule 40 source address '192.168.20.0/24'
set nat source rule 40 destination address '192.168.30.0/24'
set nat source rule 40 outbound-interface name 'bond0.103'
set nat source rule 40 translation address 'masquerade'

set nat source rule 50 source address '192.168.30.0/24'
set nat source rule 50 destination address '192.168.10.0/24'
set nat source rule 50 outbound-interface name 'bond0.101'
set nat source rule 50 translation address 'masquerade'

set nat source rule 60 source address '192.168.30.0/24'
set nat source rule 60 destination address '192.168.20.0/24'
set nat source rule 60 outbound-interface name 'bond0.102'
set nat source rule 60 translation address 'masquerade'

commit
save
```
</details>

<details>
<summary><b>Показать конфигурацию Spine-2-1</b></summary>

```bash
service routing protocols model multi-agent
!
hostname Spine-2-1
!
interface Ethernet1
   description Border-Leaf-2-1
   mtu 9214
   no switchport
   ip address 10.21.21.1/31
!
interface Ethernet2
   description Leaf-2-2
   mtu 9214
   no switchport
   ip address 10.21.22.1/31
!
interface Ethernet3
   description Leaf-2-3
   mtu 9214
   no switchport
   ip address 10.21.23.1/31
!
interface lo0
   ip add 10.10.10.201/32
!
ip routing
!
router bgp 65520
   router-id 10.10.10.201
   no bgp default ipv4-unicast
   timers bgp 3 9
   maximum-paths 4 ecmp 4
   neighbor 10.10.10.21 remote-as 65521
   neighbor 10.10.10.21 update-source Loopback0
   neighbor 10.10.10.21 description Border-Leaf-2-1
   neighbor 10.10.10.21 ebgp-multihop 3
   neighbor 10.10.10.21 send-community extended
   neighbor 10.10.10.22 remote-as 65522
   neighbor 10.10.10.22 update-source Loopback0
   neighbor 10.10.10.22 description Leaf-2-2
   neighbor 10.10.10.22 ebgp-multihop 3
   neighbor 10.10.10.22 send-community extended
   neighbor 10.10.10.23 remote-as 65523
   neighbor 10.10.10.23 update-source Loopback0
   neighbor 10.10.10.23 description Leaf-2-3
   neighbor 10.10.10.23 ebgp-multihop 3
   neighbor 10.10.10.23 send-community extended
   neighbor 10.21.21.0 remote-as 65521
   neighbor 10.21.21.0 bfd
   neighbor 10.21.21.0 description Border-Leaf-2-1
   neighbor 10.21.22.0 remote-as 65522
   neighbor 10.21.22.0 bfd
   neighbor 10.21.22.0 description Leaf-2-2
   neighbor 10.21.23.0 remote-as 65523
   neighbor 10.21.23.0 bfd
   neighbor 10.21.23.0 description Leaf-2-3
   !
   address-family evpn
      bgp next-hop-unchanged
      neighbor 10.10.10.21 activate
      neighbor 10.10.10.22 activate
      neighbor 10.10.10.23 activate
   !
   address-family ipv4
      neighbor 10.21.21.0 activate
      neighbor 10.21.22.0 activate
      neighbor 10.21.23.0 activate
      network 10.10.10.201/32
!
end
```
</details>

<details>
<summary><b>Показать конфигурацию Border-Leaf-2-1</b></summary>

```bash
service routing protocols model multi-agent
!
hostname Border-Leaf-2-1
!
vlan 10
   name INSIDE
!
vlan 101
   name FW_INSIDE
!
vlan 102
   name FW_DMZ
!
vlan 103
   name FW_SECURITY
!
vlan 20
   name DMZ
!
vlan 30
   name SECURITY
!
vrf instance DMZ
!
vrf instance INSIDE
!
vrf instance SECURITY
!
interface Port-Channel1
   mtu 9214
   no switchport
!
interface Port-Channel1.101
   encapsulation dot1q vlan 101
   vrf INSIDE
   ip address 10.21.101.0/31
!
interface Port-Channel1.102
   encapsulation dot1q vlan 102
   vrf DMZ
   ip address 10.21.102.0/31
!
interface Port-Channel1.103
   encapsulation dot1q vlan 103
   vrf SECURITY
   ip address 10.21.103.0/31
!
interface Ethernet1
   description Spine-2-1
   mtu 9214
   no switchport
   ip address 10.21.21.0/31
!
interface Ethernet4
   description Leaf-1-1
   mtu 9214
   no switchport
   ip address 10.11.21.0/31
!
interface Ethernet5
   channel-group 1 mode on
!
interface Ethernet6
   channel-group 1 mode on
!
interface Loopback0
   ip address 10.10.10.21/32
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan vlan 20 vni 10020
   vxlan vlan 30 vni 10030
   vxlan vrf DMZ vni 10200
   vxlan vrf INSIDE vni 10100
   vxlan vrf SECURITY vni 10300
!
ip routing
ip routing vrf DMZ
ip routing vrf INSIDE
ip routing vrf SECURITY
!
router bgp 65521
   router-id 10.10.10.21
   no bgp default ipv4-unicast
   timers bgp 3 9
   maximum-paths 4 ecmp 4
   neighbor 10.10.10.11 remote-as 65511
   neighbor 10.10.10.11 update-source Loopback0
   neighbor 10.10.10.11 description Leaf-1-1
   neighbor 10.10.10.11 ebgp-multihop 3
   neighbor 10.10.10.11 send-community extended
   neighbor 10.10.10.201 remote-as 65520
   neighbor 10.10.10.201 update-source Loopback0
   neighbor 10.10.10.201 description Spine-2-1
   neighbor 10.10.10.201 ebgp-multihop 3
   neighbor 10.10.10.201 send-community extended
   neighbor 10.21.21.1 remote-as 65520
   neighbor 10.21.21.1 bfd
   neighbor 10.21.21.1 description Spine-2-1
   neighbor 10.11.21.1 remote-as 65511
   neighbor 10.11.21.1 bfd
   neighbor 10.11.21.1 description Leaf-1-1
   neighbor 10.11.21.3 remote-as 65511
   neighbor 10.11.21.3 bfd
   neighbor 10.11.21.3 description Leaf-1-1
   !
   vlan 10
      rd 10.10.10.21:10
      route-target both 10:10
      redistribute learned
   !
   vlan 20
      rd 10.10.10.21:20
      route-target both 20:20
      redistribute learned
   !
   vlan 30
      rd 10.10.10.21:30
      route-target both 30:30
      redistribute learned
   !
   address-family evpn
      neighbor 10.10.10.11 activate
      neighbor 10.10.10.201 activate
   !
   address-family ipv4
      neighbor 10.21.21.1 activate
      neighbor 10.11.21.1 activate
      neighbor 10.11.21.3 activate
      network 10.10.10.21/32
   !
   vrf DMZ
      rd 10.10.10.21:200
      route-target import evpn 200:200
      route-target export evpn 200:200
      neighbor 10.21.102.1 remote-as 65502
      !
      address-family ipv4
         neighbor 10.21.102.1 activate
   !
   vrf INSIDE
      rd 10.10.10.21:100
      route-target import evpn 100:100
      route-target export evpn 100:100
      neighbor 10.21.101.1 remote-as 65502
      !
      address-family ipv4
         neighbor 10.21.101.1 activate
   !
   vrf SECURITY
      rd 10.10.10.21:300
      route-target import evpn 300:300
      route-target export evpn 300:300
      neighbor 10.21.103.1 remote-as 65502
      !
      address-family ipv4
         neighbor 10.21.103.1 activate
!
end
```
</details>

<details>
<summary><b>Показать конфигурацию Leaf-2-2</b></summary>

```bash
service routing protocols model multi-agent
!
hostname Leaf-2-2
!
vlan 10
   name INSIDE
!
vlan 20
   name DMZ
!
vlan 30
   name SECURITY
!
vrf instance DMZ
!
vrf instance INSIDE
!
vrf instance SECURITY
!
interface Ethernet1
   description Spine-2-1
   mtu 9214
   no switchport
   ip address 10.21.22.0/31
!
interface Ethernet7
   description VLAN_20
   mtu 9214
   switchport access vlan 20
!
interface Ethernet8
   description VLAN_10
   mtu 9214
   switchport access vlan 10
!
interface Loopback0
   ip address 10.10.10.22/32
!
interface Vlan10
   mtu 9214
   vrf INSIDE
   ip address virtual 192.168.10.1/24
!
interface Vlan20
   mtu 9214
   vrf DMZ
   ip address virtual 192.168.20.1/24
!
interface Vlan30
   mtu 9214
   vrf SECURITY
   ip address virtual 192.168.30.1/24
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan vlan 20 vni 10020
   vxlan vlan 30 vni 10030
   vxlan vrf DMZ vni 10200
   vxlan vrf INSIDE vni 10100
   vxlan vrf SECURITY vni 10300
!
ip virtual-router mac-address 00:00:11:11:22:22
!
ip routing
ip routing vrf DMZ
ip routing vrf INSIDE
ip routing vrf SECURITY
!
router bgp 65522
   router-id 10.10.10.22
   no bgp default ipv4-unicast
   timers bgp 3 9
   maximum-paths 4 ecmp 4
   neighbor 10.10.10.201 remote-as 65520
   neighbor 10.10.10.201 update-source Loopback0
   neighbor 10.10.10.201 description Spine-2-1
   neighbor 10.10.10.201 ebgp-multihop 3
   neighbor 10.10.10.201 send-community extended
   neighbor 10.21.22.1 remote-as 65520
   neighbor 10.21.22.1 bfd
   neighbor 10.21.22.1 description Spine-2-1
   !
   vlan 10
      rd 10.10.10.22:10
      route-target both 10:10
      redistribute learned
   !
   vlan 20
      rd 10.10.10.22:20
      route-target both 20:20
      redistribute learned
   !
   vlan 30
      rd 10.10.10.22:30
      route-target both 30:30
      redistribute learned
   !
   address-family evpn
      neighbor 10.10.10.201 activate
   !
   address-family ipv4
      neighbor 10.21.22.1 activate
      network 10.10.10.22/32
   !
   vrf DMZ
      rd 10.10.10.22:200
      route-target import evpn 200:200
      route-target export evpn 200:200
      redistribute connected
   !
   vrf INSIDE
      rd 10.10.10.22:100
      route-target import evpn 100:100
      route-target export evpn 100:100
      redistribute connected
   !
   vrf SECURITY
      rd 10.10.10.22:300
      route-target import evpn 300:300
      route-target export evpn 300:300
      redistribute connected
!
end
```
</details>

<details>
<summary><b>Показать конфигурацию Leaf-2-3</b></summary>

```bash
service routing protocols model multi-agent
!
hostname Leaf-2-3
!
vlan 10
   name INSIDE
!
vlan 20
   name DMZ
!
vlan 30
   name SECURITY
!
vrf instance DMZ
!
vrf instance INSIDE
!
vrf instance SECURITY
!
interface Ethernet1
   description Spine-2-1
   mtu 9214
   no switchport
   ip address 10.21.23.0/31
!
interface Ethernet7
   description VLAN_20
   mtu 9214
   switchport access vlan 20
!
interface Ethernet8
   description VLAN_10
   mtu 9214
   switchport access vlan 10
!
interface Loopback0
   ip address 10.10.10.23/32
!
interface Vlan10
   mtu 9214
   vrf INSIDE
   ip address virtual 192.168.10.1/24
!
interface Vlan20
   mtu 9214
   vrf DMZ
   ip address virtual 192.168.20.1/24
!
interface Vlan30
   mtu 9214
   vrf SECURITY
   ip address virtual 192.168.30.1/24
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan vlan 20 vni 10020
   vxlan vlan 30 vni 10030
   vxlan vrf DMZ vni 10200
   vxlan vrf INSIDE vni 10100
   vxlan vrf SECURITY vni 10300
!
ip virtual-router mac-address 00:00:11:11:22:22
!
ip routing
ip routing vrf DMZ
ip routing vrf INSIDE
ip routing vrf SECURITY
!
router bgp 65523
   router-id 10.10.10.23
   no bgp default ipv4-unicast
   timers bgp 3 9
   maximum-paths 4 ecmp 4
   neighbor 10.10.10.201 remote-as 65520
   neighbor 10.10.10.201 update-source Loopback0
   neighbor 10.10.10.201 description Spine-2-1
   neighbor 10.10.10.201 ebgp-multihop 3
   neighbor 10.10.10.201 send-community extended
   neighbor 10.21.23.1 remote-as 65520
   neighbor 10.21.23.1 bfd
   neighbor 10.21.23.1 description Spine-2-1
   !
   vlan 10
      rd 10.10.10.23:10
      route-target both 10:10
      redistribute learned
   !
   vlan 20
      rd 10.10.10.23:20
      route-target both 20:20
      redistribute learned
   !
   vlan 30
      rd 10.10.10.23:30
      route-target both 30:30
      redistribute learned
   !
   address-family evpn
      neighbor 10.10.10.201 activate
   !
   address-family ipv4
      neighbor 10.21.23.1 activate
      network 10.10.10.23/32
   !
   vrf DMZ
      rd 10.10.10.23:200
      route-target import evpn 200:200
      route-target export evpn 200:200
      redistribute connected
   !
   vrf INSIDE
      rd 10.10.10.23:100
      route-target import evpn 100:100
      route-target export evpn 100:100
      redistribute connected
   !
   vrf SECURITY
      rd 10.10.10.23:300
      route-target import evpn 300:300
      route-target export evpn 300:300
      redistribute connected
!
end
```
</details>

<details>
<summary><b>Показать конфигурацию FW-2</b></summary>

```bash
configure

set interfaces bonding bond0 mode xor-hash
set interfaces bonding bond0 member interface eth1
set interfaces bonding bond0 member interface eth2

set interfaces bonding bond0 vif 101 address '10.21.101.1/31'
set interfaces bonding bond0 vif 102 address '10.21.102.1/31'
set interfaces bonding bond0 vif 103 address '10.21.103.1/31'

set protocols bgp system-as '65502'

set protocols bgp neighbor 10.21.101.0 remote-as '65521'
set protocols bgp neighbor 10.21.102.0 remote-as '65521'
set protocols bgp neighbor 10.21.103.0 remote-as '65521'

set protocols static route 0.0.0.0/0 next-hop 1.1.1.1

set protocols bgp neighbor 10.21.101.0 address-family ipv4-unicast default-originate
set protocols bgp neighbor 10.21.102.0 address-family ipv4-unicast default-originate
set protocols bgp neighbor 10.21.103.0 address-family ipv4-unicast default-originate

set nat source rule 10 source address '192.168.10.0/24'
set nat source rule 10 destination address '192.168.20.0/24'
set nat source rule 10 outbound-interface name 'bond0.102'
set nat source rule 10 translation address 'masquerade'

set nat source rule 20 source address '192.168.10.0/24'
set nat source rule 20 destination address '192.168.30.0/24'
set nat source rule 20 outbound-interface name 'bond0.103'
set nat source rule 20 translation address 'masquerade'

set nat source rule 30 source address '192.168.20.0/24'
set nat source rule 30 destination address '192.168.10.0/24'
set nat source rule 30 outbound-interface name 'bond0.101'
set nat source rule 30 translation address 'masquerade'

set nat source rule 40 source address '192.168.20.0/24'
set nat source rule 40 destination address '192.168.30.0/24'
set nat source rule 40 outbound-interface name 'bond0.103'
set nat source rule 40 translation address 'masquerade'

set nat source rule 50 source address '192.168.30.0/24'
set nat source rule 50 destination address '192.168.10.0/24'
set nat source rule 50 outbound-interface name 'bond0.101'
set nat source rule 50 translation address 'masquerade'

set nat source rule 60 source address '192.168.30.0/24'
set nat source rule 60 destination address '192.168.20.0/24'
set nat source rule 60 outbound-interface name 'bond0.102'
set nat source rule 60 translation address 'masquerade'

commit
save
```
</details>

## Результаты

Проверим соседство ipv4, evpn, маршруты type 3,5, проверим, что дефолтный маршрут внесен в таблицу маршрутизации.
Для примера будем смотреть Leaf-1, тк на Leaf-3 аналогичные настройки. А затем сделам ping и trace с хоста 192.168.10.11 до 192.168.10.33 и 192.168.20.33.

<details>
<summary><b>Показать результаты на L1</b></summary>

```bash
L1#sh ip bgp summary
BGP summary information for VRF default
Router identifier 10.10.10.11, local AS number 65501
Neighbor Status Codes: m - Under maintenance
  Description              Neighbor   V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  S1-evpn                  10.10.10.1 4 65500          10368     10402    0    0 00:45:11 Estab   5      5
  S2-evpn                  10.10.10.2 4 65500          10434     10472    0    0 00:41:10 Estab   5      5
  S1-ipv4                  10.10.11.1 4 65500          10438     10457    0    0 00:46:29 Estab   5      5
  S2-ipv4                  10.10.21.1 4 65500          10407     10413    0    0 00:41:11 Estab   5      5

L1#sh ip bgp 
          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      1.1.1.1/32             -                     -       -          -       0       i
 * >Ec    2.2.2.2/32             10.10.11.1            0       -          100     0       65500 65502 i
 *  ec    2.2.2.2/32             10.10.21.1            0       -          100     0       65500 65502 i
 *  E     2.2.2.2/32             10.10.10.1            0       -          100     0       65500 65502 i
 *  e     2.2.2.2/32             10.10.10.2            0       -          100     0       65500 65502 i
 * >Ec    3.3.3.3/32             10.10.11.1            0       -          100     0       65500 65503 i
 *  ec    3.3.3.3/32             10.10.21.1            0       -          100     0       65500 65503 i
 *  E     3.3.3.3/32             10.10.10.1            0       -          100     0       65500 65503 i
 *  e     3.3.3.3/32             10.10.10.2            0       -          100     0       65500 65503 i
 * >      10.10.10.1/32          10.10.11.1            0       -          100     0       65500 i
 *        10.10.10.1/32          10.10.10.1            0       -          100     0       65500 i
 * >      10.10.10.2/32          10.10.21.1            0       -          100     0       65500 i
 *        10.10.10.2/32          10.10.10.2            0       -          100     0       65500 i
 * >      10.10.10.11/32         -                     -       -          -       0       i
 * >Ec    10.10.10.12/32         10.10.11.1            0       -          100     0       65500 65502 i
 *  ec    10.10.10.12/32         10.10.21.1            0       -          100     0       65500 65502 i
 *  E     10.10.10.12/32         10.10.10.1            0       -          100     0       65500 65502 i
 *  e     10.10.10.12/32         10.10.10.2            0       -          100     0       65500 65502 i
 * >Ec    10.10.10.13/32         10.10.11.1            0       -          100     0       65500 65503 i
 *  ec    10.10.10.13/32         10.10.21.1            0       -          100     0       65500 65503 i
 *  E     10.10.10.13/32         10.10.10.1            0       -          100     0       65500 65503 i
 *  e     10.10.10.13/32         10.10.10.2            0       -          100     0       65500 65503 i

L1#sh bgp evpn summary 
BGP summary information for VRF default
Router identifier 10.10.10.11, local AS number 65501
Neighbor Status Codes: m - Under maintenance
  Description              Neighbor   V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  S1-evpn                  10.10.10.1 4 65500          10390     10424    0    0 00:46:07 Estab   12     12
  S2-evpn                  10.10.10.2 4 65500          10456     10494    0    0 00:42:06 Estab   12     12

L1#sh ip route bgp

 B E      2.2.2.2/32 [200/0] via 10.10.21.1, Ethernet7
                             via 10.10.11.1, Ethernet8
 B E      3.3.3.3/32 [200/0] via 10.10.21.1, Ethernet7
                             via 10.10.11.1, Ethernet8
 B E      10.10.10.1/32 [200/0] via 10.10.11.1, Ethernet8
 B E      10.10.10.2/32 [200/0] via 10.10.21.1, Ethernet7
 B E      10.10.10.12/32 [200/0] via 10.10.21.1, Ethernet7
                                 via 10.10.11.1, Ethernet8
 B E      10.10.10.13/32 [200/0] via 10.10.21.1, Ethernet7
                                 via 10.10.11.1, Ethernet8

L1#sh bgp evpn route-type imet
          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 10.10.10.11:10 imet 1.1.1.1
                                 -                     -       -       0       i
 * >      RD: 10.10.10.11:20 imet 1.1.1.1
                                 -                     -       -       0       i
 * >Ec    RD: 10.10.10.12:10 imet 2.2.2.2
                                 2.2.2.2               -       100     0       65500 65502 i
 *  ec    RD: 10.10.10.12:10 imet 2.2.2.2
                                 2.2.2.2               -       100     0       65500 65502 i
 * >Ec    RD: 10.10.10.12:20 imet 2.2.2.2
                                 2.2.2.2               -       100     0       65500 65502 i
 *  ec    RD: 10.10.10.12:20 imet 2.2.2.2
                                 2.2.2.2               -       100     0       65500 65502 i
 * >Ec    RD: 10.10.10.13:10 imet 3.3.3.3
                                 3.3.3.3               -       100     0       65500 65503 i
 *  ec    RD: 10.10.10.13:10 imet 3.3.3.3
                                 3.3.3.3               -       100     0       65500 65503 i
 * >Ec    RD: 10.10.10.13:20 imet 3.3.3.3
                                 3.3.3.3               -       100     0       65500 65503 i
 *  ec    RD: 10.10.10.13:20 imet 3.3.3.3
                                 3.3.3.3               -       100     0       65500 65503 i

L1#sh bgp evpn route-type ip-prefix ipv4
          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 10.10.10.12:1 ip-prefix 0.0.0.0/0
                                 2.2.2.2               -       100     0       65500 65502 65555 ?
 *  ec    RD: 10.10.10.12:1 ip-prefix 0.0.0.0/0
                                 2.2.2.2               -       100     0       65500 65502 65555 ?
 * >Ec    RD: 10.10.10.12:2 ip-prefix 0.0.0.0/0
                                 2.2.2.2               -       100     0       65500 65502 65555 ?
 *  ec    RD: 10.10.10.12:2 ip-prefix 0.0.0.0/0
                                 2.2.2.2               -       100     0       65500 65502 65555 ?
 * >Ec    RD: 10.10.10.12:1 ip-prefix 172.20.1.0/30
                                 2.2.2.2               -       100     0       65500 65502 i
 *  ec    RD: 10.10.10.12:1 ip-prefix 172.20.1.0/30
                                 2.2.2.2               -       100     0       65500 65502 i
 * >Ec    RD: 10.10.10.12:2 ip-prefix 172.20.2.0/30
                                 2.2.2.2               -       100     0       65500 65502 i
 *  ec    RD: 10.10.10.12:2 ip-prefix 172.20.2.0/30
                                 2.2.2.2               -       100     0       65500 65502 i
 * >      RD: 10.10.10.11:1 ip-prefix 192.168.10.0/24
                                 -                     -       -       0       i
 * >Ec    RD: 10.10.10.12:1 ip-prefix 192.168.10.0/24
                                 2.2.2.2               -       100     0       65500 65502 i
 *  ec    RD: 10.10.10.12:1 ip-prefix 192.168.10.0/24
                                 2.2.2.2               -       100     0       65500 65502 i
 * >Ec    RD: 10.10.10.13:1 ip-prefix 192.168.10.0/24
                                 3.3.3.3               -       100     0       65500 65503 i
 *  ec    RD: 10.10.10.13:1 ip-prefix 192.168.10.0/24
                                 3.3.3.3               -       100     0       65500 65503 i
 * >      RD: 10.10.10.11:2 ip-prefix 192.168.20.0/24
                                 -                     -       -       0       i
 * >Ec    RD: 10.10.10.12:2 ip-prefix 192.168.20.0/24
                                 2.2.2.2               -       100     0       65500 65502 i
 *  ec    RD: 10.10.10.12:2 ip-prefix 192.168.20.0/24
                                 2.2.2.2               -       100     0       65500 65502 i
 * >Ec    RD: 10.10.10.13:2 ip-prefix 192.168.20.0/24
                                 3.3.3.3               -       100     0       65500 65503 i

L1#sh ip route vrf 1
Gateway of last resort:
 B E      0.0.0.0/0 [200/0] via VTEP 2.2.2.2 VNI 10001 router-mac 50:02:0e:3d:67:d3 local-interface Vxlan1

 B E      172.20.1.0/30 [200/0] via VTEP 2.2.2.2 VNI 10001 router-mac 50:02:0e:3d:67:d3 local-interface Vxlan1
 C        192.168.10.0/24 is directly connected, Vlan10

L1#sh ip route vrf 2
Gateway of last resort:
 B E      0.0.0.0/0 [200/0] via VTEP 2.2.2.2 VNI 10002 router-mac 50:02:0e:3d:67:d3 local-interface Vxlan1

 B E      172.20.2.0/30 [200/0] via VTEP 2.2.2.2 VNI 10002 router-mac 50:02:0e:3d:67:d3 local-interface Vxlan1
 C        192.168.20.0/24 is directly connected, Vlan20
```
</details>

<details>
<summary><b>Показать результаты на хосте</b></summary>

```bash
VPCS> sh ip

NAME        : VPCS[1]
IP/MASK     : 192.168.10.11/24
GATEWAY     : 192.168.10.1
DNS         : 
MAC         : 00:50:79:66:68:37
LPORT       : 20000
RHOST:PORT  : 127.0.0.1:30000
MTU         : 1500

VPCS> ping 192.168.10.33

192.168.10.33 icmp_seq=1 timeout
84 bytes from 192.168.10.33 icmp_seq=2 ttl=64 time=11.006 ms
84 bytes from 192.168.10.33 icmp_seq=3 ttl=64 time=11.146 ms
84 bytes from 192.168.10.33 icmp_seq=4 ttl=64 time=10.739 ms
84 bytes from 192.168.10.33 icmp_seq=5 ttl=64 time=10.557 ms

VPCS> ping 192.168.20.33

84 bytes from 192.168.20.33 icmp_seq=1 ttl=59 time=118.449 ms
84 bytes from 192.168.20.33 icmp_seq=2 ttl=59 time=24.906 ms
84 bytes from 192.168.20.33 icmp_seq=3 ttl=59 time=25.344 ms
84 bytes from 192.168.20.33 icmp_seq=4 ttl=59 time=25.002 ms
84 bytes from 192.168.20.33 icmp_seq=5 ttl=59 time=25.166 ms

VPCS> trace 192.168.20.33
trace to 192.168.20.33, 8 hops max, press Ctrl+C to stop
 1   192.168.10.1   2.212 ms  1.863 ms  2.115 ms
 2   192.168.10.1   8.767 ms  10.127 ms  8.392 ms
 3   172.20.1.1   12.595 ms  12.832 ms  10.693 ms
 4   172.20.2.2   15.520 ms  15.485 ms  14.437 ms
 5   192.168.20.1   21.763 ms  23.522 ms  19.499 ms
 6   *192.168.20.33   25.191 ms (ICMP type:3, code:3, Destination port unreachable)

```
</details>

Как мы можем видеть, устройство FW анонсирует дефолтный маршрут в vrf 1 и в vrf 2. Эти маршруты принимает Leaf-2 и рассылает по всей фабрике, то есть они становятся известны всем Leaf. Сами Leafs не маршрутизируют трафик между подсетями 192.168.10.0/24 и 192.168.20.0/24, потому что они находятся в разных vrf, а значит изолированы. Но у них есть дефолтный маршрут через FW, куда они и направляют трафик если требуется переслать данные между этими подсетями/vrf.

Ping успешен, трассировка подтверждает, что трафик идет через FW. Задача выполнена!
