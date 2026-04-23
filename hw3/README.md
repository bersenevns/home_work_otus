
# IS-IS Underlay

## Цель

Настроить протокол динамической маршрутизации IS-IS в качестве underlay-сети для топологии Spine–Leaf (2 Spine + 3 Leaf).

Цель:
- обеспечить доступность всех loopback-интерфейсов
- обеспечить доступность всех point-to-point сетей

---

## Схема

![схема](scheme.png)

---

## Адресация

Логика адресации и коммутация представлены [здесь](https://github.com/bersenevns/home_work_otus/blob/main/hw1/README.md). Добавлены только loopback.

### Loopback-интерфейсы

| Устройство | Loopback0      |
|------------|----------------|
| S1         | 10.10.10.1/32  |
| S2         | 10.10.10.2/32  |
| L1         | 10.10.10.11/32 |
| L2         | 10.10.10.12/32 |
| L3         | 10.10.10.13/32 |

---

## Конфигурация

1) Включен роутинг
2) На интерфейсах настроен MTU 9214
3) Настроен router-id
4) По умолчанию все интерфейсы в OSPF являются пассивными
5) Network-type Point-to-point

Конфигурация устройст представлена ниже, лишние строки удалены в целях читаемости.

<details>
<summary><b>Показать конфигурацию S1</b></summary>

```bash
hostname S1
!
interface Ethernet1
   description TO_LEAF_1
   mtu 9214
   no switchport
   ip address 10.10.11.1/30
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
!
interface Ethernet2
   description TO_LEAF_2
   mtu 9214
   no switchport
   ip address 10.10.12.1/30
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
!
interface Ethernet3
   description TO_LEAF_3
   mtu 9214
   no switchport
   ip address 10.10.13.1/30
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
!
interface Loopback0
   ip address 10.10.10.1/32
   ip ospf area 0.0.0.0
!
ip routing
!
router ospf 1
   router-id 10.10.10.1
   passive-interface default
   no passive-interface Ethernet1
   no passive-interface Ethernet2
   no passive-interface Ethernet3
   max-lsa 12000
!
end
```
</details>

<details>
<summary><b>Показать конфигурацию S2</b></summary>

```bash
hostname S2
!
interface Ethernet1
   description TO_LEAF_1
   mtu 9214
   no switchport
   ip address 10.10.21.1/30
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
!
interface Ethernet2
   description TO_LEAF_2
   mtu 9214
   no switchport
   ip address 10.10.22.1/30
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
!
interface Ethernet3
   description TO_LEAF_3
   mtu 9214
   no switchport
   ip address 10.10.23.1/30
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
!
interface Loopback0
   ip address 10.10.10.2/32
   ip ospf area 0.0.0.0
!
ip routing
!
router ospf 1
   router-id 10.10.10.2
   passive-interface default
   no passive-interface Ethernet1
   no passive-interface Ethernet2
   no passive-interface Ethernet3
   max-lsa 12000
!
end
```
</details>

<details>
<summary><b>Показать конфигурацию L1</b></summary>

```bash
hostname L1
!
interface Ethernet7
   description TO_SPINE_2
   mtu 9214
   no switchport
   ip address 10.10.21.2/30
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
!
interface Ethernet8
   description TO_SPINE_1
   mtu 9214
   no switchport
   ip address 10.10.11.2/30
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
!
interface Loopback0
   ip address 10.10.10.11/32
   ip ospf area 0.0.0.0
!
interface Management1
!
ip routing
!
router ospf 1
   router-id 10.10.10.11
   passive-interface default
   no passive-interface Ethernet7
   no passive-interface Ethernet8
   max-lsa 12000
!
end
```
</details>

<details>
<summary><b>Показать конфигурацию L2</b></summary>

```bash
hostname L2
!
interface Ethernet7
   description TO_SPINE_2
   mtu 9214
   no switchport
   ip address 10.10.22.2/30
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
!
interface Ethernet8
   description TO_SPINE_1
   mtu 9214
   no switchport
   ip address 10.10.12.2/30
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
!
interface Loopback0
   ip address 10.10.10.12/32
   ip ospf area 0.0.0.0
!
interface Management1
!
ip routing
!
router ospf 1
   router-id 10.10.10.12
   passive-interface default
   no passive-interface Ethernet7
   no passive-interface Ethernet8
   max-lsa 12000
!
end
```
</details>

<details>
<summary><b>Показать конфигурацию L3</b></summary>

```bash
hostname L3
!
interface Ethernet7
   description TO_SPINE_2
   mtu 9214
   no switchport
   ip address 10.10.23.2/30
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
!
interface Ethernet8
   description TO_SPINE_1
   mtu 9214
   no switchport
   ip address 10.10.13.2/30
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
!
interface Ethernet9
!
interface Loopback0
   ip address 10.10.10.13/32
   ip ospf area 0.0.0.0
!
interface Management1
!
ip routing
!
router ospf 1
   router-id 10.10.10.13
   passive-interface default
   no passive-interface Ethernet7
   no passive-interface Ethernet8
   max-lsa 12000
!
end
```
</details>

## Результаты

Возьмем для проверки L2 и проверим у него таблицу маршрутизации, также попробуем достучаться до какого-нибудь Leaf.

```bash
L2#sh ip route

VRF: default
Codes: C - connected, S - static, K - kernel, 
       O - OSPF, IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type2, B - Other BGP Routes,
       B I - iBGP, B E - eBGP, R - RIP, I L1 - IS-IS level 1,
       I L2 - IS-IS level 2, O3 - OSPFv3, A B - BGP Aggregate,
       A O - OSPF Summary, NG - Nexthop Group Static Route,
       V - VXLAN Control Service, M - Martian,
       DH - DHCP client installed default route,
       DP - Dynamic Policy Route, L - VRF Leaked,
       G  - gRIBI, RC - Route Cache Route

Gateway of last resort is not set

 O        10.10.10.1/32 [110/20] via 10.10.12.1, Ethernet8 # loopback Spine1
 O        10.10.10.2/32 [110/20] via 10.10.22.1, Ethernet7 # loopback Spine2
 O        10.10.10.11/32 [110/30] via 10.10.22.1, Ethernet7 # loopback Leaf1
                                  via 10.10.12.1, Ethernet8 # loopback Leaf1
 C        10.10.10.12/32 is directly connected, Loopback0 # свой loopback
 O        10.10.10.13/32 [110/30] via 10.10.22.1, Ethernet7 # loopback Leaf3
                                  via 10.10.12.1, Ethernet8 # loopback Leaf3
 O        10.10.11.0/30 [110/20] via 10.10.12.1, Ethernet8 # link S1-L1
 C        10.10.12.0/30 is directly connected, Ethernet8 # link S1-L2
 O        10.10.13.0/30 [110/20] via 10.10.12.1, Ethernet8 # link S1-L3
 O        10.10.21.0/30 [110/20] via 10.10.22.1, Ethernet7 # link S2-L1
 C        10.10.22.0/30 is directly connected, Ethernet7 # link S2-L2
 O        10.10.23.0/30 [110/20] via 10.10.22.1, Ethernet7 # link S2-L3

L2#ping 10.10.10.11
PING 10.10.10.11 (10.10.10.11) 72(100) bytes of data.
80 bytes from 10.10.10.11: icmp_seq=1 ttl=63 time=7.11 ms
80 bytes from 10.10.10.11: icmp_seq=2 ttl=63 time=4.74 ms
80 bytes from 10.10.10.11: icmp_seq=3 ttl=63 time=4.53 ms
80 bytes from 10.10.10.11: icmp_seq=4 ttl=63 time=4.43 ms
80 bytes from 10.10.10.11: icmp_seq=5 ttl=63 time=4.02 ms

--- 10.10.10.11 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 29ms
rtt min/avg/max/mdev = 4.024/4.969/7.110/1.096 ms, ipg/ewma 7.297/5.987 ms
```

Все P2P сети известны, все loopback известны. До Loopback Leaf1 и Leaf3 известно по 2 маршрута, по числу Spine.

Ping успешен. Задача выполнена!
