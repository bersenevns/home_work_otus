# VXLAN L3VNI

## Цель

Обеспечить маршрутизацию между разными VNI в топологии Spine–Leaf (2 Spine + 3 Leaf). 

Для проверки успешности настройки:
- хосты в разных VLAN должны пинговать друг друга вне зависимости от того, к какому LEAF они подключены.

---

## Схема

Протокол динамической маршрутизации BGP будет использоваться и в качестве underlay (eBGP), и в качестве overlay (AF evpn), чтобы избежать избыточной конфигурации. Подробнее о настройке eBGP underlay [здесь](https://github.com/bersenevns/home_work_otus/blob/main/hw4/README.md), подробнее о настройке EVPN L2VNI [здесь](https://github.com/bersenevns/home_work_otus/blob/main/hw5/README.md)

К каждому Leaf подключено по 2 хоста:
- хост в VLAN 10 (eth1) попадет в VNI 1010
- хост в VLAN 20 (eth2) попадет в VNI 2020

В качестве L3VNI будет использоваться VLAN 100 (VNI 10100), будет настроен vrf CLIENTS. Также будут настроен anycast gateway, который будет использоваться в качестве шлюза по умолчанию для хостов.

![схема](scheme_l3vni.png)

---

## Адресация

Логика адресации и коммутация представлены [здесь](https://github.com/bersenevns/home_work_otus/blob/main/hw1/README.md). Ниже представлена краткая выжимка.

### Spine 1 connections

| Link    | Network       | Spine Port | Spine IP   | Leaf IP    | Leaf Port |
|---------|---------------|------------|------------|------------|-----------|
| S1 ↔ L1 | 10.10.11.0/30 | eth1       | 10.10.11.1 | 10.10.11.2 | eth8      |
| S1 ↔ L2 | 10.10.12.0/30 | eth2       | 10.10.12.1 | 10.10.12.2 | eth8      |
| S1 ↔ L3 | 10.10.13.0/30 | eth3       | 10.10.13.1 | 10.10.13.2 | eth8      |

---

### Spine 2 connections

| Link    | Network       | Spine Port | Spine IP   | Leaf IP    | Leaf Port |
|---------|---------------|------------|------------|------------|-----------|
| S2 ↔ L1 | 10.10.21.0/30 | eth1       | 10.10.21.1 | 10.10.21.2 | eth7      |
| S2 ↔ L2 | 10.10.22.0/30 | eth2       | 10.10.22.1 | 10.10.22.2 | eth7      |
| S2 ↔ L3 | 10.10.23.0/30 | eth3       | 10.10.23.1 | 10.10.23.2 | eth7      |

---

### Loopback-интерфейсы

| Устройство | Loopback0      |
|------------|----------------|
| S1         | 10.10.10.1/32  |
| S2         | 10.10.10.2/32  |
| L1         | 10.10.10.11/32 |
| L2         | 10.10.10.12/32 |
| L3         | 10.10.10.13/32 |

### AS-numbering

| Устройство | ASN   |
|------------|-------|
| S1         | 65500 |
| S2         | 65500 |
| L1         | 65501 |
| L2         | 65502 |
| L3         | 65503 |

### Адресация хостов

| Устройство | интерфейс | VLAN | IP            |
|------------|-----------|------|---------------|
| Leaf1      | eth1      | 10   | 192.168.10.11 |
| Leaf1      | eth2      | 20   | 192.168.20.11 |
| Leaf2      | eth1      | 10   | 192.168.10.22 |
| Leaf2      | eth2      | 20   | 192.168.20.22 |
| Leaf3      | eth1      | 10   | 192.168.10.33 |
| Leaf3      | eth2      | 20   | 192.168.20.33 |

---

## Конфигурация

1) Включен роутинг
2) На интерфейсах настроен MTU 9214
3) Настроен router-id
4) Настроен BFD
5) Изменены таймеры BGP 3s keep-alive, 9s hold
6) Настроена аутентификация для BGP
7) Настроен ECMP
8) Конфигурация с использованием peer group для удобства
9) Включена модель multi-agent для возможности использования MP-BGP
10) Сделан mapping VLAN - VNI
11) Использование vlan-based
12) redistribute изученных локально mac-адресов
13) Включение отправки extended community (без этого не заработает)
14) Настройка anycast gateway в VLAN 10, VLAN 20
15) Настройка виртуального mac-адреса
16) Настройка vrf CLIENTS
17) Redistribute connected сетей

Конфигурация устройств представлена ниже, лишние строки удалены в целях читаемости.

<details>
<summary><b>Показать конфигурацию S1</b></summary>

```bash
service routing protocols model multi-agent
!
hostname S1
!
interface Ethernet1
   mtu 9214
   no switchport
   ip address 10.10.11.1/30
!
interface Ethernet2
   mtu 9214
   no switchport
   ip address 10.10.12.1/30
!
interface Ethernet3
   mtu 9214
   no switchport
   ip address 10.10.13.1/30
interface Loopback0
   ip address 10.10.10.1/32
!
ip routing
!
router bgp 65500
   router-id 10.10.10.1
   timers bgp 3 9
   maximum-paths 10 ecmp 10
   neighbor LEAFS peer group
   neighbor LEAFS bfd
   neighbor LEAFS password 7 /3ZkUd1QPGBJ7Vztltm37A==
   neighbor LEAFS send-community extended
   neighbor 10.10.11.2 peer group LEAFS
   neighbor 10.10.11.2 remote-as 65501
   neighbor 10.10.11.2 description L1
   neighbor 10.10.12.2 peer group LEAFS
   neighbor 10.10.12.2 remote-as 65502
   neighbor 10.10.12.2 description L2
   neighbor 10.10.13.2 peer group LEAFS
   neighbor 10.10.13.2 remote-as 65503
   neighbor 10.10.13.2 description L3
   !
   address-family evpn
      neighbor LEAFS activate
      neighbor LEAFS encapsulation vxlan
   !
   address-family ipv4
      neighbor LEAFS activate
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
   mtu 9214
   no switchport
   ip address 10.10.21.1/30
!
interface Ethernet2
   mtu 9214
   no switchport
   ip address 10.10.22.1/30
!
interface Ethernet3
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
   neighbor LEAFS peer group
   neighbor LEAFS bfd
   neighbor LEAFS password 7 /3ZkUd1QPGBJ7Vztltm37A==
   neighbor LEAFS send-community extended
   neighbor 10.10.21.2 peer group LEAFS
   neighbor 10.10.21.2 remote-as 65501
   neighbor 10.10.21.2 description L1
   neighbor 10.10.22.2 peer group LEAFS
   neighbor 10.10.22.2 remote-as 65502
   neighbor 10.10.22.2 description L2
   neighbor 10.10.23.2 peer group LEAFS
   neighbor 10.10.23.2 remote-as 65503
   neighbor 10.10.23.2 description L3
   !
   address-family evpn
      neighbor LEAFS activate
      neighbor LEAFS encapsulation vxlan
   !
   address-family ipv4
      neighbor LEAFS activate
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
vrf instance CLIENTS
!
interface Ethernet1
   switchport access vlan 10
!
interface Ethernet2
   switchport access vlan 20
!
interface Ethernet7
   mtu 9214
   no switchport
   ip address 10.10.21.2/30
!
interface Ethernet8
   mtu 9214
   no switchport
   ip address 10.10.11.2/30
!
interface Loopback0
   ip address 10.10.10.11/32
!
interface Vlan10
   vrf CLIENTS
   ip address virtual 192.168.10.1/24
!
interface Vlan20
   vrf CLIENTS
   ip address virtual 192.168.20.1/24
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 10 vni 1010
   vxlan vlan 20 vni 2020
   vxlan vrf CLIENTS vni 10100
!
ip virtual-router mac-address 00:00:00:00:00:01
!
ip routing
ip routing vrf CLIENTS
!
router bgp 65501
   router-id 10.10.10.11
   timers bgp 3 9
   maximum-paths 2 ecmp 2
   neighbor SPINES peer group
   neighbor SPINES remote-as 65500
   neighbor SPINES bfd
   neighbor SPINES password 7 fDU2u9m4KeL5vrpR0VRCug==
   neighbor SPINES send-community extended
   neighbor 10.10.11.1 peer group SPINES
   neighbor 10.10.11.1 description S1
   neighbor 10.10.21.1 peer group SPINES
   neighbor 10.10.21.1 description S2
   !
   vlan 10
      rd auto
      route-target both 655:1010
      redistribute learned
   !
   vlan 20
      rd auto
      route-target both 655:2020
      redistribute learned
   !
   address-family evpn
      neighbor SPINES activate
      neighbor SPINES encapsulation vxlan 
   !
   address-family ipv4
      neighbor SPINES activate
      network 10.10.10.11/32
   !
   vrf CLIENTS
      rd 65501:10100
      route-target import evpn 655:10100
      route-target export evpn 655:10100
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
vrf instance CLIENTS
!
interface Ethernet1
   switchport access vlan 10
!
interface Ethernet2
   switchport access vlan 20
!
interface Ethernet7
   mtu 9214
   no switchport
   ip address 10.10.22.2/30
!
interface Ethernet8
   mtu 9214
   no switchport
   ip address 10.10.12.2/30
!
interface Loopback0
   ip address 10.10.10.12/32
!
interface Vlan10
   vrf CLIENTS
   ip address virtual 192.168.10.1/24
!
interface Vlan20
   vrf CLIENTS
   ip address virtual 192.168.20.1/24
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 10 vni 1010
   vxlan vlan 20 vni 2020
   vxlan vrf CLIENTS vni 10100
!
ip virtual-router mac-address 00:00:00:00:00:01
!
ip routing
ip routing vrf CLIENTS
!
router bgp 65502
   router-id 10.10.10.12
   timers bgp 3 9
   maximum-paths 2 ecmp 2
   neighbor SPINES peer group
   neighbor SPINES remote-as 65500
   neighbor SPINES bfd
   neighbor SPINES password 7 fDU2u9m4KeL5vrpR0VRCug==
   neighbor SPINES send-community extended
   neighbor 10.10.12.1 peer group SPINES
   neighbor 10.10.12.1 description S1
   neighbor 10.10.22.1 peer group SPINES
   neighbor 10.10.22.1 description S2
   !
   vlan 10
      rd auto
      route-target both 655:1010
      redistribute learned
   !
   vlan 20
      rd auto
      route-target both 655:2020
      redistribute learned
   !
   address-family evpn
      neighbor SPINES activate
      neighbor SPINES encapsulation vxlan 
   !
   address-family ipv4
      neighbor SPINES activate
      network 10.10.10.12/32
   !
   vrf CLIENTS
      rd 65502:10100
      route-target import evpn 655:10100
      route-target export evpn 655:10100
      redistribute connected
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
vrf instance CLIENTS
!
interface Ethernet1
   switchport access vlan 10
!
interface Ethernet2
   switchport access vlan 20
!
interface Ethernet7
   mtu 9214
   no switchport
   ip address 10.10.23.2/30
!
interface Ethernet8
   mtu 9214
   no switchport
   ip address 10.10.13.2/30
!
interface Loopback0
   ip address 10.10.10.13/32
!
interface Vlan10
   vrf CLIENTS
   ip address virtual 192.168.10.1/24
!
interface Vlan20
   vrf CLIENTS
   ip address virtual 192.168.20.1/24
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 10 vni 1010
   vxlan vlan 20 vni 2020
   vxlan vrf CLIENTS vni 10100
!
ip virtual-router mac-address 00:00:00:00:00:01
!
ip routing
ip routing vrf CLIENTS
!
router bgp 65503
   router-id 10.10.10.13
   timers bgp 3 9
   maximum-paths 2 ecmp 2
   neighbor SPINES peer group
   neighbor SPINES remote-as 65500
   neighbor SPINES bfd
   neighbor SPINES password 7 fDU2u9m4KeL5vrpR0VRCug==
   neighbor SPINES send-community extended
   neighbor 10.10.13.1 peer group SPINES
   neighbor 10.10.13.1 description S1
   neighbor 10.10.23.1 peer group SPINES
   neighbor 10.10.23.1 description S2
   !
   vlan 10
      rd auto
      route-target both 655:1010
      redistribute learned
   !
   vlan 20
      rd auto
      route-target both 655:2020
      redistribute learned
   !
   address-family evpn
      neighbor SPINES activate
      neighbor SPINES encapsulation vxlan 
   !
   address-family ipv4
      neighbor SPINES activate
      network 10.10.10.13/32
   !
   vrf CLIENTS
      rd 65503:10100
      route-target import evpn 655:10100
      route-target export evpn 655:10100
      redistribute connected
!
end
```
</details>

## Результаты

Проверим таблицу соседства BGP, таблицу полученных от соседей префиксов ipv4, таблицу маршрутизации BGP, таблицу evpn route-type 3,5. Будем проверять только LEAF, тк на Spine конфигурация осталась прежней с [L2VNI](https://github.com/bersenevns/home_work_otus/blob/main/hw5/README.md).

<details>
<summary><b>Показать результаты на L1</b></summary>

```bash
L1#sh bgp summary 
BGP summary information for VRF default
Router identifier 10.10.10.11, local AS number 65501
Neighbor            AS Session State AFI/SAFI                AFI/SAFI State   NLRI Rcd   NLRI Acc
---------- ----------- ------------- ----------------------- -------------- ---------- ----------
10.10.11.1       65500 Established   IPv4 Unicast            Negotiated              3          3
10.10.11.1       65500 Established   L2VPN EVPN              Negotiated              8          8
10.10.21.1       65500 Established   IPv4 Unicast            Negotiated              2          2
10.10.21.1       65500 Established   L2VPN EVPN              Negotiated              8          8     
   
L1#sh bgp ipv4 unicast 

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      10.10.10.1/32          10.10.11.1            0       -          100     0       65500 i
 * >Ec    10.10.10.12/32         10.10.21.1            0       -          100     0       65500 65502 i
 *  ec    10.10.10.12/32         10.10.11.1            0       -          100     0       65500 65502 i
 * >Ec    10.10.10.13/32         10.10.21.1            0       -          100     0       65500 65503 i
 *  ec    10.10.10.13/32         10.10.11.1            0       -          100     0       65500 65503 i

L1#sh ip route bgp

 B E      10.10.10.1/32 [200/0] via 10.10.11.1, Ethernet8
 B E      10.10.10.12/32 [200/0] via 10.10.21.1, Ethernet7 # Leaf-2
                                 via 10.10.11.1, Ethernet8 # Leaf-2
 B E      10.10.10.13/32 [200/0] via 10.10.21.1, Ethernet7 # Leaf-3
                                 via 10.10.11.1, Ethernet8 # Leaf-3

L1#sh vxlan vtep
Remote VTEPS for Vxlan1:

VTEP              Tunnel Type(s)
----------------- --------------
10.10.10.12       unicast, flood
10.10.10.13       unicast, flood

Total number of remote VTEPS:  2

L1#sh bgp evpn route-type imet 

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 10.10.10.11:10 imet 10.10.10.11
                                 -                     -       -       0       i
 * >      RD: 10.10.10.11:20 imet 10.10.10.11
                                 -                     -       -       0       i
 * >Ec    RD: 10.10.10.12:10 imet 10.10.10.12
                                 10.10.10.12           -       100     0       65500 65502 i
 *  ec    RD: 10.10.10.12:10 imet 10.10.10.12
                                 10.10.10.12           -       100     0       65500 65502 i
 * >Ec    RD: 10.10.10.12:20 imet 10.10.10.12
                                 10.10.10.12           -       100     0       65500 65502 i
 *  ec    RD: 10.10.10.12:20 imet 10.10.10.12
                                 10.10.10.12           -       100     0       65500 65502 i
 * >Ec    RD: 10.10.10.13:10 imet 10.10.10.13
                                 10.10.10.13           -       100     0       65500 65503 i
 *  ec    RD: 10.10.10.13:10 imet 10.10.10.13
                                 10.10.10.13           -       100     0       65500 65503 i
 * >Ec    RD: 10.10.10.13:20 imet 10.10.10.13
                                 10.10.10.13           -       100     0       65500 65503 i
 *  ec    RD: 10.10.10.13:20 imet 10.10.10.13
                                 10.10.10.13           -       100     0       65500 65503 i

L1#sh bgp evpn route-type ip-prefix ipv4

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 65501:10100 ip-prefix 192.168.10.0/24
                                 -                     -       -       0       i
 * >Ec    RD: 65502:10100 ip-prefix 192.168.10.0/24
                                 10.10.10.12           -       100     0       65500 65502 i
 *  ec    RD: 65502:10100 ip-prefix 192.168.10.0/24
                                 10.10.10.12           -       100     0       65500 65502 i
 * >Ec    RD: 65503:10100 ip-prefix 192.168.10.0/24
                                 10.10.10.13           -       100     0       65500 65503 i
 *  ec    RD: 65503:10100 ip-prefix 192.168.10.0/24
                                 10.10.10.13           -       100     0       65500 65503 i
 * >      RD: 65501:10100 ip-prefix 192.168.20.0/24
                                 -                     -       -       0       i
 * >Ec    RD: 65502:10100 ip-prefix 192.168.20.0/24
                                 10.10.10.12           -       100     0       65500 65502 i
 *  ec    RD: 65502:10100 ip-prefix 192.168.20.0/24
                                 10.10.10.12           -       100     0       65500 65502 i
 * >Ec    RD: 65503:10100 ip-prefix 192.168.20.0/24
                                 10.10.10.13           -       100     0       65500 65503 i
 *  ec    RD: 65503:10100 ip-prefix 192.168.20.0/24
                                 10.10.10.13           -       100     0       65500 65503 i
```
</details>

<details>
<summary><b>Показать результаты на L2</b></summary>

```bash
L2#sh bgp summary 
BGP summary information for VRF default
Router identifier 10.10.10.12, local AS number 65502
Neighbor            AS Session State AFI/SAFI                AFI/SAFI State   NLRI Rcd   NLRI Acc
---------- ----------- ------------- ----------------------- -------------- ---------- ----------
10.10.12.1       65500 Established   IPv4 Unicast            Negotiated              2          2
10.10.12.1       65500 Established   L2VPN EVPN              Negotiated              4          4
10.10.22.1       65500 Established   IPv4 Unicast            Negotiated              1          1
10.10.22.1       65500 Established   L2VPN EVPN              Negotiated              4          4

L2#sh bgp ipv4 unicast

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      10.10.10.1/32          10.10.12.1            0       -          100     0       65500 i
 * >Ec    10.10.10.11/32         10.10.12.1            0       -          100     0       65500 65501 i
 *  ec    10.10.10.11/32         10.10.22.1            0       -          100     0       65500 65501 i
 * >      10.10.10.12/32         -                     -       -          -       0       i
 * >Ec    10.10.10.13/32         10.10.12.1            0       -          100     0       65500 65503 i
 *  ec    10.10.10.13/32         10.10.22.1            0       -          100     0       65500 65503 i

L2#sh ip route bgp

 B E      10.10.10.1/32 [200/0] via 10.10.12.1, Ethernet8
 B E      10.10.10.11/32 [200/0] via 10.10.22.1, Ethernet7 # Leaf-1
                                 via 10.10.12.1, Ethernet8 # Leaf-1
 B E      10.10.10.13/32 [200/0] via 10.10.22.1, Ethernet7 # Leaf-3
                                 via 10.10.12.1, Ethernet8 # Leaf-3

L2#sh vxlan vtep
Remote VTEPS for Vxlan1:

VTEP              Tunnel Type(s)
----------------- --------------
10.10.10.11       flood, unicast
10.10.10.13       flood, unicast

Total number of remote VTEPS:  2

L2#sh bgp evpn route-type imet 

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 10.10.10.11:10 imet 10.10.10.11
                                 10.10.10.11           -       100     0       65500 65501 i
 *  ec    RD: 10.10.10.11:10 imet 10.10.10.11
                                 10.10.10.11           -       100     0       65500 65501 i
 * >Ec    RD: 10.10.10.11:20 imet 10.10.10.11
                                 10.10.10.11           -       100     0       65500 65501 i
 *  ec    RD: 10.10.10.11:20 imet 10.10.10.11
                                 10.10.10.11           -       100     0       65500 65501 i
 * >      RD: 10.10.10.12:10 imet 10.10.10.12
                                 -                     -       -       0       i
 * >      RD: 10.10.10.12:20 imet 10.10.10.12
                                 -                     -       -       0       i
 * >Ec    RD: 10.10.10.13:10 imet 10.10.10.13
                                 10.10.10.13           -       100     0       65500 65503 i
 *  ec    RD: 10.10.10.13:10 imet 10.10.10.13
                                 10.10.10.13           -       100     0       65500 65503 i
 * >Ec    RD: 10.10.10.13:20 imet 10.10.10.13
                                 10.10.10.13           -       100     0       65500 65503 i
 *  ec    RD: 10.10.10.13:20 imet 10.10.10.13
                                 10.10.10.13           -       100     0       65500 65503 i

L2#sh bgp evpn route-type ip-prefix ipv4

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 65501:10100 ip-prefix 192.168.10.0/24
                                 10.10.10.11           -       100     0       65500 65501 i
 *  ec    RD: 65501:10100 ip-prefix 192.168.10.0/24
                                 10.10.10.11           -       100     0       65500 65501 i
 * >      RD: 65502:10100 ip-prefix 192.168.10.0/24
                                 -                     -       -       0       i
 * >Ec    RD: 65503:10100 ip-prefix 192.168.10.0/24
                                 10.10.10.13           -       100     0       65500 65503 i
 *  ec    RD: 65503:10100 ip-prefix 192.168.10.0/24
                                 10.10.10.13           -       100     0       65500 65503 i
 * >Ec    RD: 65501:10100 ip-prefix 192.168.20.0/24
                                 10.10.10.11           -       100     0       65500 65501 i
 *  ec    RD: 65501:10100 ip-prefix 192.168.20.0/24
                                 10.10.10.11           -       100     0       65500 65501 i
 * >      RD: 65502:10100 ip-prefix 192.168.20.0/24
                                 -                     -       -       0       i
 * >Ec    RD: 65503:10100 ip-prefix 192.168.20.0/24
                                 10.10.10.13           -       100     0       65500 65503 i
 *  ec    RD: 65503:10100 ip-prefix 192.168.20.0/24
                                 10.10.10.13           -       100     0       65500 65503 i
```
</details>

<details>
<summary><b>Показать результаты на L3</b></summary>

```bash
L3#sh bgp summary 
BGP summary information for VRF default
Router identifier 10.10.10.13, local AS number 65503
Neighbor            AS Session State AFI/SAFI                AFI/SAFI State   NLRI Rcd   NLRI Acc
---------- ----------- ------------- ----------------------- -------------- ---------- ----------
10.10.13.1       65500 Established   IPv4 Unicast            Negotiated              3          3
10.10.13.1       65500 Established   L2VPN EVPN              Negotiated              8          8
10.10.23.1       65500 Established   IPv4 Unicast            Negotiated              2          2
10.10.23.1       65500 Established   L2VPN EVPN              Negotiated              8          8

L3#sh bgp ipv4 unicast 

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      10.10.10.1/32          10.10.13.1            0       -          100     0       65500 i
 * >Ec    10.10.10.11/32         10.10.23.1            0       -          100     0       65500 65501 i
 *  ec    10.10.10.11/32         10.10.13.1            0       -          100     0       65500 65501 i
 * >Ec    10.10.10.12/32         10.10.13.1            0       -          100     0       65500 65502 i
 *  ec    10.10.10.12/32         10.10.23.1            0       -          100     0       65500 65502 i
 * >      10.10.10.13/32         -                     -       -          -       0       i

L3#sh ip route bgp

 B E      10.10.10.1/32 [200/0] via 10.10.13.1, Ethernet8 
 B E      10.10.10.11/32 [200/0] via 10.10.23.1, Ethernet7 # Leaf-1
                                 via 10.10.13.1, Ethernet8 # Leaf-1
 B E      10.10.10.12/32 [200/0] via 10.10.23.1, Ethernet7 # Leaf-2
                                 via 10.10.13.1, Ethernet8 # Leaf-2

L3#sh vxlan vtep 
Remote VTEPS for Vxlan1:

VTEP              Tunnel Type(s)
----------------- --------------
10.10.10.11       flood, unicast
10.10.10.12       flood, unicast

Total number of remote VTEPS:  2

L3#sh bgp evpn route-type imet 

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 10.10.10.11:10 imet 10.10.10.11
                                 10.10.10.11           -       100     0       65500 65501 i
 *  ec    RD: 10.10.10.11:10 imet 10.10.10.11
                                 10.10.10.11           -       100     0       65500 65501 i
 * >Ec    RD: 10.10.10.11:20 imet 10.10.10.11
                                 10.10.10.11           -       100     0       65500 65501 i
 *  ec    RD: 10.10.10.11:20 imet 10.10.10.11
                                 10.10.10.11           -       100     0       65500 65501 i
 * >Ec    RD: 10.10.10.12:10 imet 10.10.10.12
                                 10.10.10.12           -       100     0       65500 65502 i
 *  ec    RD: 10.10.10.12:10 imet 10.10.10.12
                                 10.10.10.12           -       100     0       65500 65502 i
 * >Ec    RD: 10.10.10.12:20 imet 10.10.10.12
                                 10.10.10.12           -       100     0       65500 65502 i
 *  ec    RD: 10.10.10.12:20 imet 10.10.10.12
                                 10.10.10.12           -       100     0       65500 65502 i
 * >      RD: 10.10.10.13:10 imet 10.10.10.13
                                 -                     -       -       0       i
 * >      RD: 10.10.10.13:20 imet 10.10.10.13
                                 -                     -       -       0       i

L3#sh bgp evpn route-type ip-prefix ipv4

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 65501:10100 ip-prefix 192.168.10.0/24
                                 10.10.10.11           -       100     0       65500 65501 i
 *  ec    RD: 65501:10100 ip-prefix 192.168.10.0/24
                                 10.10.10.11           -       100     0       65500 65501 i
 * >Ec    RD: 65502:10100 ip-prefix 192.168.10.0/24
                                 10.10.10.12           -       100     0       65500 65502 i
 *  ec    RD: 65502:10100 ip-prefix 192.168.10.0/24
                                 10.10.10.12           -       100     0       65500 65502 i
 * >      RD: 65503:10100 ip-prefix 192.168.10.0/24
                                 -                     -       -       0       i
 * >Ec    RD: 65501:10100 ip-prefix 192.168.20.0/24
                                 10.10.10.11           -       100     0       65500 65501 i
 *  ec    RD: 65501:10100 ip-prefix 192.168.20.0/24
                                 10.10.10.11           -       100     0       65500 65501 i
 * >Ec    RD: 65502:10100 ip-prefix 192.168.20.0/24
                                 10.10.10.12           -       100     0       65500 65502 i
 *  ec    RD: 65502:10100 ip-prefix 192.168.20.0/24
                                 10.10.10.12           -       100     0       65500 65502 i
 * >      RD: 65503:10100 ip-prefix 192.168.20.0/24
                                 -                     -       -       0       i
```
</details>

Все необходимые loopback известны. Сессии evpn также установлены, VTEPs видны. Мы видим маршруты type-3 (IMET) и type-5 (ip-prefix). Маршруты type-2 (mac-ip) пока неизвестны, потому что еще не было трафика от хостов. 

Теперь попробуем с хоста 192.168.10.11 (Leaf-1) сделать ping до всех остальных хостов, подключенных ко всем Leaf.

<details>
<summary><b>Показать результаты на 192.168.10.11</b></summary>

```bash
VPCS> sh ip

NAME        : VPCS[1]
IP/MASK     : 192.168.10.11/24
GATEWAY     : 192.168.10.1
DNS         : 
MAC         : 00:50:79:66:68:17
LPORT       : 20000
RHOST:PORT  : 127.0.0.1:30000
MTU         : 1500

VPCS> ping 192.168.10.22

192.168.10.22 icmp_seq=1 timeout
84 bytes from 192.168.10.22 icmp_seq=2 ttl=64 time=11.101 ms
84 bytes from 192.168.10.22 icmp_seq=3 ttl=64 time=9.726 ms
84 bytes from 192.168.10.22 icmp_seq=4 ttl=64 time=9.446 ms
84 bytes from 192.168.10.22 icmp_seq=5 ttl=64 time=11.508 ms

VPCS> ping 192.168.10.33

192.168.10.33 icmp_seq=1 timeout
84 bytes from 192.168.10.33 icmp_seq=2 ttl=64 time=10.764 ms
84 bytes from 192.168.10.33 icmp_seq=3 ttl=64 time=9.968 ms
84 bytes from 192.168.10.33 icmp_seq=4 ttl=64 time=10.896 ms
84 bytes from 192.168.10.33 icmp_seq=5 ttl=64 time=10.138 ms

```
</details>

<details>
<summary><b>Показать результаты на 192.168.20.11</b></summary>

```bash
VPCS> sh ip            

NAME        : VPCS[1]
IP/MASK     : 192.168.10.11/24
GATEWAY     : 192.168.10.1
DNS         : 
MAC         : 00:50:79:66:68:22
LPORT       : 20000
RHOST:PORT  : 127.0.0.1:30000
MTU         : 1500

VPCS> ping 192.168.10.1 # Anycast Gateway

84 bytes from 192.168.10.1 icmp_seq=1 ttl=64 time=2.424 ms
84 bytes from 192.168.10.1 icmp_seq=2 ttl=64 time=3.246 ms
84 bytes from 192.168.10.1 icmp_seq=3 ttl=64 time=3.213 ms
84 bytes from 192.168.10.1 icmp_seq=4 ttl=64 time=2.535 ms
84 bytes from 192.168.10.1 icmp_seq=5 ttl=64 time=2.370 ms

VPCS> ping 192.168.10.22 # host Leaf-2

84 bytes from 192.168.10.22 icmp_seq=1 ttl=64 time=13.278 ms
84 bytes from 192.168.10.22 icmp_seq=2 ttl=64 time=12.075 ms
84 bytes from 192.168.10.22 icmp_seq=3 ttl=64 time=12.030 ms
84 bytes from 192.168.10.22 icmp_seq=4 ttl=64 time=12.022 ms
84 bytes from 192.168.10.22 icmp_seq=5 ttl=64 time=13.777 ms

VPCS> ping 192.168.10.33 # host Leaf-3

84 bytes from 192.168.10.33 icmp_seq=1 ttl=64 time=15.765 ms
84 bytes from 192.168.10.33 icmp_seq=2 ttl=64 time=12.267 ms
84 bytes from 192.168.10.33 icmp_seq=3 ttl=64 time=12.189 ms
84 bytes from 192.168.10.33 icmp_seq=4 ttl=64 time=10.988 ms
84 bytes from 192.168.10.33 icmp_seq=5 ttl=64 time=11.494 ms

VPCS> ping 192.168.20.11 # host Leaf-1

84 bytes from 192.168.20.11 icmp_seq=1 ttl=63 time=5.277 ms
84 bytes from 192.168.20.11 icmp_seq=2 ttl=63 time=4.994 ms
84 bytes from 192.168.20.11 icmp_seq=3 ttl=63 time=4.404 ms
84 bytes from 192.168.20.11 icmp_seq=4 ttl=63 time=4.830 ms
84 bytes from 192.168.20.11 icmp_seq=5 ttl=63 time=5.885 ms

VPCS> ping 192.168.20.22 # host Leaf-2

84 bytes from 192.168.20.22 icmp_seq=1 ttl=62 time=23.321 ms
84 bytes from 192.168.20.22 icmp_seq=2 ttl=62 time=12.292 ms
84 bytes from 192.168.20.22 icmp_seq=3 ttl=62 time=12.815 ms
84 bytes from 192.168.20.22 icmp_seq=4 ttl=62 time=12.744 ms
84 bytes from 192.168.20.22 icmp_seq=5 ttl=62 time=14.975 ms

VPCS> ping 192.168.20.33 # host Leaf-3

84 bytes from 192.168.20.33 icmp_seq=1 ttl=62 time=18.840 ms
84 bytes from 192.168.20.33 icmp_seq=2 ttl=62 time=15.128 ms
84 bytes from 192.168.20.33 icmp_seq=3 ttl=62 time=12.172 ms
84 bytes from 192.168.20.33 icmp_seq=4 ttl=62 time=12.146 ms
84 bytes from 192.168.20.33 icmp_seq=5 ttl=62 time=11.575 ms

```
</details>

<details>
<summary><b>Посмотрим, что изменилось на Leaf-1</b></summary>

```bash
L1#sh bgp evpn route-type mac-ip

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 10.10.10.11:20 mac-ip 0050.7966.681d
                                 -                     -       -       0       i
 * >      RD: 10.10.10.11:20 mac-ip 0050.7966.681d 192.168.20.11
                                 -                     -       -       0       i
 * >Ec    RD: 10.10.10.12:10 mac-ip 0050.7966.681e
                                 10.10.10.12           -       100     0       65500 65502 i
 *  ec    RD: 10.10.10.12:10 mac-ip 0050.7966.681e
                                 10.10.10.12           -       100     0       65500 65502 i
 * >Ec    RD: 10.10.10.12:10 mac-ip 0050.7966.681e 192.168.10.22
                                 10.10.10.12           -       100     0       65500 65502 i
 *  ec    RD: 10.10.10.12:10 mac-ip 0050.7966.681e 192.168.10.22
                                 10.10.10.12           -       100     0       65500 65502 i
 * >Ec    RD: 10.10.10.12:20 mac-ip 0050.7966.681f
                                 10.10.10.12           -       100     0       65500 65502 i
 *  ec    RD: 10.10.10.12:20 mac-ip 0050.7966.681f
                                 10.10.10.12           -       100     0       65500 65502 i
 * >Ec    RD: 10.10.10.12:20 mac-ip 0050.7966.681f 192.168.20.22
                                 10.10.10.12           -       100     0       65500 65502 i
 *  ec    RD: 10.10.10.12:20 mac-ip 0050.7966.681f 192.168.20.22
                                 10.10.10.12           -       100     0       65500 65502 i
 * >Ec    RD: 10.10.10.13:10 mac-ip 0050.7966.6820
                                 10.10.10.13           -       100     0       65500 65503 i
 *  ec    RD: 10.10.10.13:10 mac-ip 0050.7966.6820
                                 10.10.10.13           -       100     0       65500 65503 i
 * >Ec    RD: 10.10.10.13:10 mac-ip 0050.7966.6820 192.168.10.33
                                 10.10.10.13           -       100     0       65500 65503 i
 *  ec    RD: 10.10.10.13:10 mac-ip 0050.7966.6820 192.168.10.33
                                 10.10.10.13           -       100     0       65500 65503 i
 * >Ec    RD: 10.10.10.13:20 mac-ip 0050.7966.6821
                                 10.10.10.13           -       100     0       65500 65503 i
 *  ec    RD: 10.10.10.13:20 mac-ip 0050.7966.6821
                                 10.10.10.13           -       100     0       65500 65503 i
 * >Ec    RD: 10.10.10.13:20 mac-ip 0050.7966.6821 192.168.20.33
                                 10.10.10.13           -       100     0       65500 65503 i
 *  ec    RD: 10.10.10.13:20 mac-ip 0050.7966.6821 192.168.20.33
                                 10.10.10.13           -       100     0       65500 65503 i
 * >      RD: 10.10.10.11:10 mac-ip 0050.7966.6822
                                 -                     -       -       0       i
 * >      RD: 10.10.10.11:10 mac-ip 0050.7966.6822 192.168.10.11
                                 -                     -       -       0       i

L1#sh mac address-table 
          Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports      Moves   Last Move
----    -----------       ----        -----      -----   ---------
   1    0000.0000.0001    STATIC      Cpu
  10    0000.0000.0001    STATIC      Cpu
  10    0050.7966.681e    DYNAMIC     Vx1        1       0:05:52 ago
  10    0050.7966.6820    DYNAMIC     Vx1        1       0:05:24 ago
  10    0050.7966.6822    DYNAMIC     Et1        1       0:08:22 ago
  20    0000.0000.0001    STATIC      Cpu
  20    0050.7966.681d    DYNAMIC     Et2        1       0:06:10 ago
  20    0050.7966.681f    DYNAMIC     Vx1        1       0:05:39 ago
  20    0050.7966.6821    DYNAMIC     Vx1        1       0:05:09 ago
4094    0000.0000.0001    STATIC      Cpu
4094    50e6.3de9.14d9    DYNAMIC     Vx1        1       0:27:44 ago
4094    50fa.63ee.8e7e    DYNAMIC     Vx1        1       0:27:44 ago
Total Mac Addresses for this criterion: 12

```
</details>

Мы видим, что на Leaf-1 появились маршруты type-2 (mac-ip), изучены по EVPN BGP, внесены в локальную таблицу mac-адресов.

Ping до всех хостов успешен, L3VNI работает, задача выполнена!
