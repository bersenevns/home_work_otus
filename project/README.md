
# Итоговый проект

## Цель

Настроить сеть двух ЦОД по модели Active-Active с межсегментной фильтрацией трафика при помощи VxLAN EVPN.
Хосты в одной сети могут взаимодействовать друг с другом по L2 между ЦОД.
Выход из каждого ЦОД должен быть локальным, без использования межцодового канала.
Маршгрутизация между сегментами сети должна осуществляться только через Firewall.
Обратный трафик сделать симметричным с помощью SNAT на локальной Firewall.

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
3) Настраиваем интерфейсы VLAN 10,20  разных VRF 1 и 2
4) Настраиваем Anycast gateway на Leafs
5) Настраиваем BGP-соседство между Leaf2 и FW для передачи дефолтного маршрута в фабрику
6) Проверяем связанность между VRF на хостах

Конфигурация устройств представлена ниже, лишние строки удалены в целях читаемости.

<details>
<summary><b>Показать конфигурацию S1</b></summary>

```bash
service routing protocols model multi-agent
!
hostname S1
!
interface Ethernet1
   description L1
   mtu 9214
   no switchport
   ip address 10.10.11.1/30
!
interface Ethernet2
   description L2
   mtu 9214
   no switchport
   ip address 10.10.12.1/30
!
interface Ethernet3
   description L3
   mtu 9214
   no switchport
   ip address 10.10.13.1/30
!
interface Loopback0
   ip address 10.10.10.1/32
!
ip routing
!
router bgp 65500
   router-id 10.10.10.1
   timers bgp 3 9
   maximum-paths 10 ecmp 10
   neighbor EVPN peer group
   neighbor EVPN update-source Loopback0
   neighbor EVPN ebgp-multihop 3
   neighbor EVPN send-community extended
   neighbor 10.10.10.11 peer group EVPN
   neighbor 10.10.10.11 remote-as 65501
   neighbor 10.10.10.12 peer group EVPN
   neighbor 10.10.10.12 remote-as 65502
   neighbor 10.10.10.13 peer group EVPN
   neighbor 10.10.10.13 remote-as 65503
   neighbor 10.10.11.2 remote-as 65501
   neighbor 10.10.12.2 remote-as 65502
   neighbor 10.10.13.2 remote-as 65503
   !
   address-family evpn
      bgp next-hop-unchanged
      neighbor EVPN activate
   !
   address-family ipv4
      neighbor 10.10.11.2 activate
      neighbor 10.10.12.2 activate
      neighbor 10.10.13.2 activate
      network 10.10.10.1/32
!
end
```
</details>

<details>
<summary><b>Показать конфигурацию S2</b></summary>

```bash
service routing protocols model multi-agent
!
hostname S2
!
interface Ethernet1
   description L1
   mtu 9214
   no switchport
   ip address 10.10.21.1/30
!
interface Ethernet2
   description L2
   mtu 9214
   no switchport
   ip address 10.10.22.1/30
!
interface Ethernet3
   description L3
   mtu 9214
   no switchport
   ip address 10.10.23.1/30
!
interface Loopback0
   ip address 10.10.10.2/32
!
ip routing
!
router bgp 65500
   router-id 10.10.10.2
   timers bgp 3 9
   maximum-paths 10 ecmp 10
   neighbor EVPN peer group
   neighbor EVPN update-source Loopback0
   neighbor EVPN ebgp-multihop 3
   neighbor EVPN send-community extended
   neighbor 10.10.10.11 peer group EVPN
   neighbor 10.10.10.11 remote-as 65501
   neighbor 10.10.10.12 peer group EVPN
   neighbor 10.10.10.12 remote-as 65502
   neighbor 10.10.10.13 peer group EVPN
   neighbor 10.10.10.13 remote-as 65503
   neighbor 10.10.21.2 remote-as 65501
   neighbor 10.10.22.2 remote-as 65502
   neighbor 10.10.23.2 remote-as 65503
   !
   address-family evpn
      bgp next-hop-unchanged
      neighbor EVPN activate
   !
   address-family ipv4
      neighbor 10.10.21.2 activate
      neighbor 10.10.22.2 activate
      neighbor 10.10.23.2 activate
      network 10.10.10.2/32
!
end
```
</details>

<details>
<summary><b>Показать конфигурацию L1</b></summary>

```bash
service routing protocols model multi-agent
!
hostname L1
!
vlan 10,20
!
vrf instance 1
!
vrf instance 2
!
interface Ethernet1
   switchport access vlan 10
!
interface Ethernet2
   switchport access vlan 20
!
interface Ethernet3
!
interface Ethernet4
!
interface Ethernet5
!
interface Ethernet6
!
interface Ethernet7
   description S2
   mtu 9214
   no switchport
   ip address 10.10.21.2/30
!
interface Ethernet8
   description S1
   mtu 9214
   no switchport
   ip address 10.10.11.2/30
!
interface Loopback0
   ip address 10.10.10.11/32
!
interface Loopback1
   ip address 1.1.1.1/32
!
interface Vlan10
   vrf 1
   ip address virtual 192.168.10.1/24
!
interface Vlan20
   vrf 2
   ip address virtual 192.168.20.1/24
!
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan vlan 20 vni 10020
   vxlan vrf 1 vni 10001
   vxlan vrf 2 vni 10002
!
ip virtual-router mac-address 00:00:11:11:22:22
!
ip routing
ip routing vrf 1
ip routing vrf 2
!
router bgp 65501
   router-id 10.10.10.11
   timers bgp 3 9
   maximum-paths 2 ecmp 2
   neighbor 10.10.10.1 remote-as 65500
   neighbor 10.10.10.1 update-source Loopback0
   neighbor 10.10.10.1 description S1-evpn
   neighbor 10.10.10.1 ebgp-multihop 3
   neighbor 10.10.10.1 send-community extended
   neighbor 10.10.10.2 remote-as 65500
   neighbor 10.10.10.2 update-source Loopback0
   neighbor 10.10.10.2 description S2-evpn
   neighbor 10.10.10.2 ebgp-multihop 3
   neighbor 10.10.10.2 send-community extended
   neighbor 10.10.11.1 remote-as 65500
   neighbor 10.10.11.1 description S1-ipv4
   neighbor 10.10.21.1 remote-as 65500
   neighbor 10.10.21.1 description S2-ipv4
   !
   vlan 10
      rd 10.10.10.11:10
      route-target both 655:10
      redistribute learned
   !
   vlan 20
      rd 10.10.10.11:20
      route-target both 655:20
      redistribute learned
   !
   address-family evpn
      bgp next-hop-unchanged
      neighbor 10.10.10.1 activate
      neighbor 10.10.10.1 encapsulation vxlan 
      neighbor 10.10.10.2 activate
      neighbor 10.10.10.2 encapsulation vxlan 
   !
   address-family ipv4
      neighbor 10.10.11.1 activate
      neighbor 10.10.21.1 activate
      network 1.1.1.1/32
      network 10.10.10.11/32
   !
   vrf 1
      rd 10.10.10.11:1
      route-target import evpn 655:1
      route-target export evpn 655:1
      redistribute connected
   !
   vrf 2
      rd 10.10.10.11:2
      route-target import evpn 655:2
      route-target export evpn 655:2
      redistribute connected
!
end
```
</details>

<details>
<summary><b>Показать конфигурацию L2</b></summary>

```bash
service routing protocols model multi-agent
!
hostname L2
!
vlan 10,20
!
vrf instance 1
!
vrf instance 2
!
interface Ethernet1
   no switchport
   vrf 1
   ip address 172.20.1.2/30
!
interface Ethernet2
   no switchport
   vrf 2
   ip address 172.20.2.2/30
!
interface Ethernet7
   description S2
   mtu 9214
   no switchport
   ip address 10.10.22.2/30
!
interface Ethernet8
   description S1
   mtu 9214
   no switchport
   ip address 10.10.12.2/30
!
interface Loopback0
   ip address 10.10.10.12/32
!
interface Loopback1
   ip address 2.2.2.2/32
!
interface Vlan10
   vrf 1
   ip address virtual 192.168.10.1/24
!
interface Vlan20
   vrf 2
   ip address virtual 192.168.20.1/24
!
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan vlan 20 vni 10020
   vxlan vrf 1 vni 10001
   vxlan vrf 2 vni 10002
!
ip virtual-router mac-address 00:00:11:11:22:22
!
ip routing
ip routing vrf 1
ip routing vrf 2
!
router bgp 65502
   router-id 10.10.10.12
   timers bgp 3 9
   maximum-paths 2 ecmp 2
   neighbor 10.10.10.1 remote-as 65500
   neighbor 10.10.10.1 update-source Loopback0
   neighbor 10.10.10.1 ebgp-multihop 3
   neighbor 10.10.10.1 send-community extended
   neighbor 10.10.10.2 remote-as 65500
   neighbor 10.10.10.2 update-source Loopback0
   neighbor 10.10.10.2 ebgp-multihop 3
   neighbor 10.10.10.2 send-community extended
   neighbor 10.10.12.1 remote-as 65500
   neighbor 10.10.22.1 remote-as 65500
   !
   vlan 10
      rd 10.10.10.12:10
      route-target both 655:10
      redistribute learned
   !
   vlan 20
      rd 10.10.10.12:20
      route-target both 655:20
      redistribute learned
   !
   address-family evpn
      bgp next-hop-unchanged
      neighbor 10.10.10.1 activate
      neighbor 10.10.10.1 encapsulation vxlan 
      neighbor 10.10.10.2 activate
      neighbor 10.10.10.2 encapsulation vxlan 
   !
   address-family ipv4
      neighbor 10.10.12.1 activate
      neighbor 10.10.22.1 activate
      network 2.2.2.2/32
      network 10.10.10.12/32
   !
   vrf 1
      rd 10.10.10.12:1
      route-target import evpn 655:1
      route-target export evpn 655:1
      neighbor 172.20.1.1 remote-as 65555
      redistribute connected
      !
      address-family ipv4
         neighbor 172.20.1.1 activate
   !
   vrf 2
      rd 10.10.10.12:2
      route-target import evpn 655:2
      route-target export evpn 655:2
      neighbor 172.20.2.1 remote-as 65555
      redistribute connected
      !
      address-family ipv4
         neighbor 172.20.2.1 activate
!
end
```
</details>

<details>
<summary><b>Показать конфигурацию L3</b></summary>

```bash
service routing protocols model multi-agent
!
hostname L3
!
vlan 10,20
!
vrf instance 1
!
vrf instance 2
!
interface Ethernet1
   switchport access vlan 10
!
interface Ethernet2
   switchport access vlan 20
!
interface Ethernet3
!
interface Ethernet4
!
interface Ethernet5
!
interface Ethernet6
!
interface Ethernet7
   description S2
   mtu 9214
   no switchport
   ip address 10.10.23.2/30
!
interface Ethernet8
   description S1
   mtu 9214
   no switchport
   ip address 10.10.13.2/30
!
interface Loopback0
   ip address 10.10.10.13/32
!
interface Loopback1
   ip address 3.3.3.3/32
!
interface Vlan10
   vrf 1
   ip address virtual 192.168.10.1/24
!
interface Vlan20
   vrf 2
   ip address virtual 192.168.20.1/24
!
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan vlan 20 vni 10020
   vxlan vrf 1 vni 10001
   vxlan vrf 2 vni 10002
!
ip virtual-router mac-address 00:00:11:11:22:22
!
ip routing
ip routing vrf 1
ip routing vrf 2
!
router bgp 65503
   router-id 10.10.10.13
   timers bgp 3 9
   maximum-paths 2 ecmp 2
   neighbor 10.10.10.1 remote-as 65500
   neighbor 10.10.10.1 update-source Loopback0
   neighbor 10.10.10.1 ebgp-multihop 3
   neighbor 10.10.10.1 send-community extended
   neighbor 10.10.10.2 remote-as 65500
   neighbor 10.10.10.2 update-source Loopback0
   neighbor 10.10.10.2 ebgp-multihop 3
   neighbor 10.10.10.2 send-community extended
   neighbor 10.10.13.1 remote-as 65500
   neighbor 10.10.23.1 remote-as 65500
   !
   vlan 10
      rd 10.10.10.13:10
      route-target both 655:10
      redistribute learned
   !
   vlan 20
      rd 10.10.10.13:20
      route-target both 655:20
      redistribute learned
   !
   address-family evpn
      bgp next-hop-unchanged
      neighbor 10.10.10.1 activate
      neighbor 10.10.10.1 encapsulation vxlan 
      neighbor 10.10.10.2 activate
      neighbor 10.10.10.2 encapsulation vxlan 
   !
   address-family ipv4
      neighbor 10.10.13.1 activate
      neighbor 10.10.23.1 activate
      network 3.3.3.3/32
      network 10.10.10.13/32
   !
   vrf 1
      rd 10.10.10.13:1
      route-target import evpn 655:1
      route-target export evpn 655:1
      redistribute connected
   !
   vrf 2
      rd 10.10.10.13:2
      route-target import evpn 655:2
      route-target export evpn 655:2
      redistribute connected
!
end
```
</details>

<details>
<summary><b>Показать конфигурацию FW</b></summary>

```bash
service routing protocols model multi-agent
!
hostname FW
!
interface Ethernet1
   no switchport
   ip address 172.20.1.1/30
!
interface Ethernet2
   no switchport
   ip address 172.20.2.1/30
!
ip routing
!
ip route 0.0.0.0/0 Null0
!
router bgp 65555
   router-id 10.10.10.10
   neighbor 172.20.1.2 remote-as 65502
   neighbor 172.20.2.2 remote-as 65502
   redistribute static
   !
   address-family ipv4
      neighbor 172.20.1.2 activate
      neighbor 172.20.2.2 activate
!
end
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
