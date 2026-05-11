
# VXLAN L2VNI

## Цель

Обеспечить L2 связанность между хостами по L3 сети в топологии Spine–Leaf (2 Spine + 3 Leaf).

Для этого:
- настроить BGP peering между Leaf и Spine в AF L2 evpn.
- хосты в одном VLAN, но на разных Leaf должны пинговать друг друга.

---

## Схема

Протокол динамической маршрутизации BGP будет использоваться и в качестве underlay (eBGP), и в качестве overlay (AF evpn), чтобы избежать избыточной конфигурации. Подробнее о настройке eBGP underlay [здесь](https://github.com/bersenevns/home_work_otus/blob/main/hw4/README.md).

К каждому Leaf подключено по 2 хоста:
- хост в VLAN 10 (eth1) попадет в VNI 1010
- хост в VLAN 20 (eth2) попадет в VNI 2020

![схема](scheme_l2vni.png)

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
11) Использование vlan-aware-bundle, чтобы работать не с одним, а сразу с несколькими VLAN
12) redistribute изученных локально mac-адресов
13) Включение отправки extended community (без этого не заработает)

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
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 10 vni 1010
   vxlan vlan 20 vni 2020
!
ip routing
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
   vlan-aware-bundle DC
      rd 655:1
      route-target both 655:100
      redistribute learned
      vlan 10,20
   !
   address-family evpn
      neighbor SPINES activate
      neighbor SPINES encapsulation vxlan
   !
   address-family ipv4
      neighbor SPINES activate
      network 10.10.10.11/32
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
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 10 vni 1010
   vxlan vlan 20 vni 2020
!
ip routing
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
   vlan-aware-bundle DC
      rd 655:2
      route-target both 655:100
      redistribute learned
      vlan 10,20
   !
   address-family evpn
      neighbor SPINES activate
      neighbor SPINES encapsulation vxlan 
   !
   address-family ipv4
      neighbor SPINES activate
      network 10.10.10.12/32
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
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 10 vni 1010
   vxlan vlan 20 vni 2020
!
ip routing
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
   vlan-aware-bundle DC
      rd 655:3
      route-target both 655:100
      redistribute learned
      vlan 10,20
   !
   address-family evpn
      neighbor SPINES activate
      neighbor SPINES encapsulation vxlan 
   !
   address-family ipv4
      neighbor SPINES activate
      network 10.10.10.13/32
!
end
```
</details>

## Результаты

Проверим таблицу соседства BGP, таблицу полученных от соседей префиксов ipv4, таблицу маршрутизации BGP, таблицу evpn route-type 2,3, выполним ping до удаленных loopback.
Возьмем в качестве пример S1 из спайнов и L3 из лифов.

<details>
<summary><b>Показать результаты на S1</b></summary>

```bash
S1#sh bgp summary 
BGP summary information for VRF default
Router identifier 10.10.10.1, local AS number 65500
Neighbor            AS Session State AFI/SAFI                AFI/SAFI State   NLRI Rcd   NLRI Acc
---------- ----------- ------------- ----------------------- -------------- ---------- ----------
10.10.11.2       65501 Established   IPv4 Unicast            Negotiated              1          1
10.10.11.2       65501 Established   L2VPN EVPN              Negotiated              2          2
10.10.12.2       65502 Established   IPv4 Unicast            Negotiated              1          1
10.10.12.2       65502 Established   L2VPN EVPN              Negotiated              2          2
10.10.13.2       65503 Established   IPv4 Unicast            Negotiated              1          1
10.10.13.2       65503 Established   L2VPN EVPN              Negotiated              2          2        
   
S1#sh bgp ipv4 unicast

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      10.10.10.1/32          -                     -       -          -       0       i
 * >      10.10.10.11/32         10.10.11.2            0       -          100     0       65501 i
 * >      10.10.10.12/32         10.10.12.2            0       -          100     0       65502 i
 * >      10.10.10.13/32         10.10.13.2            0       -          100     0       65503 i

S1#sh ip route bgp

 B E      10.10.10.11/32 [200/0] via 10.10.11.2, Ethernet1
 B E      10.10.10.12/32 [200/0] via 10.10.12.2, Ethernet2
 B E      10.10.10.13/32 [200/0] via 10.10.13.2, Ethernet3

S1#sh bgp evpn

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 655:3 mac-ip 2020 f68a.41ac.2c2f
                                 10.10.10.13           -       100     0       65503 i
 * >      RD: 655:1 imet 1010 10.10.10.11
                                 10.10.10.11           -       100     0       65501 i
 * >      RD: 655:2 imet 1010 10.10.10.12
                                 10.10.10.12           -       100     0       65502 i
 * >      RD: 655:3 imet 1010 10.10.10.13
                                 10.10.10.13           -       100     0       65503 i
 * >      RD: 655:1 imet 2020 10.10.10.11
                                 10.10.10.11           -       100     0       65501 i
 * >      RD: 655:2 imet 2020 10.10.10.12
                                 10.10.10.12           -       100     0       65502 i
 * >      RD: 655:3 imet 2020 10.10.10.13
                                 10.10.10.13           -       100     0       65503 i

S1#ping 10.10.10.11
PING 10.10.10.11 (10.10.10.11) 72(100) bytes of data.
80 bytes from 10.10.10.11: icmp_seq=1 ttl=64 time=4.03 ms
80 bytes from 10.10.10.11: icmp_seq=2 ttl=64 time=2.48 ms
80 bytes from 10.10.10.11: icmp_seq=3 ttl=64 time=2.36 ms
80 bytes from 10.10.10.11: icmp_seq=4 ttl=64 time=2.43 ms
80 bytes from 10.10.10.11: icmp_seq=5 ttl=64 time=2.88 ms

S1#ping 10.10.10.12
PING 10.10.10.12 (10.10.10.12) 72(100) bytes of data.
80 bytes from 10.10.10.12: icmp_seq=1 ttl=64 time=5.63 ms
80 bytes from 10.10.10.12: icmp_seq=2 ttl=64 time=3.98 ms
80 bytes from 10.10.10.12: icmp_seq=3 ttl=64 time=4.58 ms
80 bytes from 10.10.10.12: icmp_seq=4 ttl=64 time=2.54 ms
80 bytes from 10.10.10.12: icmp_seq=5 ttl=64 time=2.98 ms

S1#ping 10.10.10.13
PING 10.10.10.13 (10.10.10.13) 72(100) bytes of data.
80 bytes from 10.10.10.13: icmp_seq=1 ttl=64 time=4.10 ms
80 bytes from 10.10.10.13: icmp_seq=2 ttl=64 time=2.57 ms
80 bytes from 10.10.10.13: icmp_seq=3 ttl=64 time=2.41 ms
80 bytes from 10.10.10.13: icmp_seq=4 ttl=64 time=2.59 ms
80 bytes from 10.10.10.13: icmp_seq=5 ttl=64 time=2.77 ms

```
</details>

На Spine 2 все аналогично, поэтому показывать здесь не буду. Перейдем к Leaf

<details>
<summary><b>Показать результаты на L3</b></summary>

```bash
L3#sh bgp summary 
BGP summary information for VRF default
Router identifier 10.10.10.13, local AS number 65503
Neighbor            AS Session State AFI/SAFI                AFI/SAFI State   NLRI Rcd   NLRI Acc
---------- ----------- ------------- ----------------------- -------------- ---------- ----------
10.10.13.1       65500 Established   IPv4 Unicast            Negotiated              3          3
10.10.13.1       65500 Established   L2VPN EVPN              Negotiated              4          4
10.10.23.1       65500 Established   IPv4 Unicast            Negotiated              3          3
10.10.23.1       65500 Established   L2VPN EVPN              Negotiated              4          4

L3#sh bgp ipv4 unicast

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      10.10.10.1/32          10.10.13.1            0       -          100     0       65500 i
 * >      10.10.10.2/32          10.10.23.1            0       -          100     0       65500 i
 * >Ec    10.10.10.11/32         10.10.13.1            0       -          100     0       65500 65501 i
 *  ec    10.10.10.11/32         10.10.23.1            0       -          100     0       65500 65501 i
 * >Ec    10.10.10.12/32         10.10.13.1            0       -          100     0       65500 65502 i
 *  ec    10.10.10.12/32         10.10.23.1            0       -          100     0       65500 65502 i
 * >      10.10.10.13/32         -                     -       -          -       0       i

L3#sh ip route bgp

 B E      10.10.10.1/32 [200/0] via 10.10.13.1, Ethernet8 # Spine-1
 B E      10.10.10.2/32 [200/0] via 10.10.23.1, Ethernet7 # Spine-2
 B E      10.10.10.11/32 [200/0] via 10.10.23.1, Ethernet7 # Leaf-1
                                 via 10.10.13.1, Ethernet8 # Leaf-1
 B E      10.10.10.12/32 [200/0] via 10.10.23.1, Ethernet7 # Leaf-2
                                 via 10.10.13.1, Ethernet8 # Leaf-2

L3#sh bgp evpn

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 655:1 imet 1010 10.10.10.11
                                 10.10.10.11           -       100     0       65500 65501 i
 *  ec    RD: 655:1 imet 1010 10.10.10.11
                                 10.10.10.11           -       100     0       65500 65501 i
 * >Ec    RD: 655:2 imet 1010 10.10.10.12
                                 10.10.10.12           -       100     0       65500 65502 i
 *  ec    RD: 655:2 imet 1010 10.10.10.12
                                 10.10.10.12           -       100     0       65500 65502 i
 * >      RD: 655:3 imet 1010 10.10.10.13
                                 -                     -       -       0       i
 * >Ec    RD: 655:1 imet 2020 10.10.10.11
                                 10.10.10.11           -       100     0       65500 65501 i
 *  ec    RD: 655:1 imet 2020 10.10.10.11
                                 10.10.10.11           -       100     0       65500 65501 i
 * >Ec    RD: 655:2 imet 2020 10.10.10.12
                                 10.10.10.12           -       100     0       65500 65502 i
 *  ec    RD: 655:2 imet 2020 10.10.10.12
                                 10.10.10.12           -       100     0       65500 65502 i
 * >      RD: 655:3 imet 2020 10.10.10.13
                                 -                     -       -       0       i

L3#ping 10.10.10.11 source loopback 0
PING 10.10.10.11 (10.10.10.11) from 10.10.10.13 : 72(100) bytes of data.
80 bytes from 10.10.10.11: icmp_seq=1 ttl=63 time=6.68 ms
80 bytes from 10.10.10.11: icmp_seq=2 ttl=63 time=5.12 ms
80 bytes from 10.10.10.11: icmp_seq=3 ttl=63 time=4.80 ms
80 bytes from 10.10.10.11: icmp_seq=4 ttl=63 time=4.73 ms
80 bytes from 10.10.10.11: icmp_seq=5 ttl=63 time=4.77 ms

L3#ping 10.10.10.12 source loopback 0
PING 10.10.10.12 (10.10.10.12) from 10.10.10.13 : 72(100) bytes of data.
80 bytes from 10.10.10.12: icmp_seq=1 ttl=63 time=6.54 ms
80 bytes from 10.10.10.12: icmp_seq=2 ttl=63 time=6.34 ms
80 bytes from 10.10.10.12: icmp_seq=3 ttl=63 time=5.82 ms
80 bytes from 10.10.10.12: icmp_seq=4 ttl=63 time=5.47 ms
80 bytes from 10.10.10.12: icmp_seq=5 ttl=63 time=6.23 ms

```
</details>

Все необходимые loopback известны и доступны по сети. Сессии evpn также установлены и мы видим маршруты type-3 (IMET). Теперь попробуем с хоста 192.168.10.11 (Leaf-1) сделать ping до 192.168.10.22 (Leaf-2) и 192.168.10.22 (Leaf-3).

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

Ping успешен. Задача выполнена!
