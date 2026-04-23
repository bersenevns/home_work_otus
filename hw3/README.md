
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

---

## Конфигурация

1) Включен роутинг
2) На интерфейсах настроен MTU 9000
3) Настроен router-id
4) Настроен BFD
5) Network-type Point-to-point
6) Настроена аутентификация md5 для IS-IS
7) Настроен параметр is-hostname для читаемости соседства

Конфигурация устройств представлена ниже, лишние строки удалены в целях читаемости.

<details>
<summary><b>Показать конфигурацию S1</b></summary>

```bash
hostname S1
!
interface Ethernet1
   mtu 9000
   no switchport
   ip address 10.10.11.1/30
   isis enable UNDERLAY
   isis bfd
   isis network point-to-point
   isis authentication mode md5 level-2
   isis authentication key 7 cvOJP+1oA0lBd9aIu3gTUA== level-2
!
interface Ethernet2
   mtu 9000
   no switchport
   ip address 10.10.12.1/30
   isis enable UNDERLAY
   isis bfd
   isis network point-to-point
   isis authentication mode md5 level-2
   isis authentication key 7 cvOJP+1oA0lBd9aIu3gTUA== level-2
!
interface Ethernet3
   mtu 9000
   no switchport
   ip address 10.10.13.1/30
   isis enable UNDERLAY
   isis bfd
   isis network point-to-point
   isis authentication mode md5 level-2
   isis authentication key 7 cvOJP+1oA0lBd9aIu3gTUA== level-2
!
interface Loopback0
   ip address 10.10.10.1/32
   isis enable UNDERLAY
!
ip routing
!
router isis UNDERLAY
   net 49.0001.0100.1001.0001.00
   is-hostname S1
   router-id ipv4 10.10.10.1
   is-type level-2
   authentication mode md5 level-2
   authentication key 7 cvOJP+1oA0lBd9aIu3gTUA== level-2
   !
   address-family ipv4 unicast
      maximum-paths 2
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
   mtu 9000
   no switchport
   ip address 10.10.21.1/30
   isis enable UNDERLAY
   isis bfd
   isis network point-to-point
   isis authentication mode md5 level-2
   isis authentication key 7 cvOJP+1oA0lBd9aIu3gTUA== level-2
!
interface Ethernet2
   mtu 9000
   no switchport
   ip address 10.10.22.1/30
   isis enable UNDERLAY
   isis bfd
   isis network point-to-point
   isis authentication mode md5 level-2
   isis authentication key 7 cvOJP+1oA0lBd9aIu3gTUA== level-2
!
interface Ethernet3
   mtu 9000
   no switchport
   ip address 10.10.23.1/30
   isis enable UNDERLAY
   isis bfd
   isis network point-to-point
   isis authentication mode md5 level-2
   isis authentication key 7 cvOJP+1oA0lBd9aIu3gTUA== level-2
!
interface Loopback0
   ip address 10.10.10.2/32
   isis enable UNDERLAY
!
ip routing
!
router isis UNDERLAY
   net 49.0001.0100.1001.0002.00
   is-hostname S2
   router-id ipv4 10.10.10.2
   is-type level-2
   authentication mode md5 level-2
   authentication key 7 cvOJP+1oA0lBd9aIu3gTUA== level-2
   !
   address-family ipv4 unicast
      maximum-paths 2
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
   mtu 9000
   no switchport
   ip address 10.10.21.2/30
   isis enable UNDERLAY
   isis bfd
   isis network point-to-point
   isis authentication mode md5 level-2
   isis authentication key 7 cvOJP+1oA0lBd9aIu3gTUA== level-2
!
interface Ethernet8
   mtu 9000
   no switchport
   ip address 10.10.11.2/30
   isis enable UNDERLAY
   isis bfd
   isis network point-to-point
   isis authentication mode md5 level-2
   isis authentication key 7 cvOJP+1oA0lBd9aIu3gTUA== level-2
!
interface Loopback0
   ip address 10.10.10.11/32
   isis enable UNDERLAY
!
ip routing
!
router isis UNDERLAY
   net 49.0001.0100.1001.0011.00
   is-hostname L1
   router-id ipv4 10.10.10.11
   is-type level-2
   authentication mode md5 level-2
   authentication key 7 cvOJP+1oA0lBd9aIu3gTUA== level-2
   !
   address-family ipv4 unicast
      maximum-paths 2
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
   mtu 9000
   no switchport
   ip address 10.10.22.2/30
   isis enable UNDERLAY
   isis bfd
   isis network point-to-point
   isis authentication mode md5 level-2
   isis authentication key 7 cvOJP+1oA0lBd9aIu3gTUA== level-2
!
interface Ethernet8
   mtu 9000
   no switchport
   ip address 10.10.12.2/30
   isis enable UNDERLAY
   isis bfd
   isis network point-to-point
   isis authentication mode md5 level-2
   isis authentication key 7 cvOJP+1oA0lBd9aIu3gTUA== level-2
!
interface Loopback0
   ip address 10.10.10.12/32
   isis enable UNDERLAY
!
ip routing
!
router isis UNDERLAY
   net 49.0001.0100.1001.0012.00
   is-hostname L2
   router-id ipv4 10.10.10.12
   is-type level-2
   authentication mode md5 level-2
   authentication key 7 cvOJP+1oA0lBd9aIu3gTUA== level-2
   !
   address-family ipv4 unicast
      maximum-paths 2
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
   mtu 9000
   no switchport
   ip address 10.10.23.2/30
   isis enable UNDERLAY
   isis bfd
   isis network point-to-point
   isis authentication mode md5 level-2
   isis authentication key 7 cvOJP+1oA0lBd9aIu3gTUA== level-2
!
interface Ethernet8
   mtu 9000
   no switchport
   ip address 10.10.13.2/30
   isis enable UNDERLAY
   isis bfd
   isis network point-to-point
   isis authentication mode md5 level-2
   isis authentication key 7 cvOJP+1oA0lBd9aIu3gTUA== level-2
!
interface Loopback0
   ip address 10.10.10.13/32
   isis enable UNDERLAY
!
ip routing
!
router isis UNDERLAY
   net 49.0001.0100.1001.0013.00
   is-hostname L3
   router-id ipv4 10.10.10.13
   is-type level-2
   authentication mode md5 level-2
   authentication key 7 cvOJP+1oA0lBd9aIu3gTUA== level-2
   !
   address-family ipv4 unicast
      maximum-paths 2
!
end
```
</details>

## Результаты

Проверим таблицу соседства IS-IS, таблицу маршрутизации, работоспособность BFD, выполним ping до удаленного loopback.
Возьмем в качестве пример S1 из спайнов и L3 из лифов.

<details>
<summary><b>Показать результаты на S1</b></summary>

```bash
S1#sh isis neighbors 
 
Instance  VRF      System Id        Type Interface          SNPA              State Hold time   Circuit Id       
   
UNDERLAY  default  L1               L2   Ethernet1          P2P               UP    23          15               
   
UNDERLAY  default  L2               L2   Ethernet2          P2P               UP    24          14               
   
UNDERLAY  default  L3               L2   Ethernet3          P2P               UP    27          14               
   
S1#sh ip route
 C        10.10.10.1/32 is directly connected, Loopback0 #свой loopback
 I L2     10.10.10.2/32 [115/30] via 10.10.11.2, Ethernet1 #loopback S2
                                 via 10.10.12.2, Ethernet2 #loopback S2
 I L2     10.10.10.11/32 [115/20] via 10.10.11.2, Ethernet1 #loopback L1
 I L2     10.10.10.12/32 [115/20] via 10.10.12.2, Ethernet2 #loopback L2
 I L2     10.10.10.13/32 [115/20] via 10.10.13.2, Ethernet3 #loopback L3
 C        10.10.11.0/30 is directly connected, Ethernet1 #Link S1-L1
 C        10.10.12.0/30 is directly connected, Ethernet2 #Link S1-L2
 C        10.10.13.0/30 is directly connected, Ethernet3 #Link S1-L3
 I L2     10.10.21.0/30 [115/20] via 10.10.11.2, Ethernet1 #Link S2-L1
 I L2     10.10.22.0/30 [115/20] via 10.10.12.2, Ethernet2 #Link S2-L2
 I L2     10.10.23.0/30 [115/20] via 10.10.13.2, Ethernet3 #Link S2-L3

S1#sh bfd peers 
DstAddr        MyDisc    YourDisc  Interface/Transport    Type          LastUp 
---------- ----------- ----------- -------------------- ------- ---------------
10.10.11.2 3506383235  1114058778        Ethernet1(13)  normal  04/23/26 08:36 
10.10.12.2 2797557802   514908787        Ethernet2(14)  normal  04/23/26 08:32 
10.10.13.2 1698202898  2306285918        Ethernet3(15)  normal  04/23/26 08:37 

   LastDown            LastDiag    State
-------------- ------------------- -----
         NA       No Diagnostic       Up
         NA       No Diagnostic       Up
         NA       No Diagnostic       Up

S1#ping 10.10.10.13
PING 10.10.10.13 (10.10.10.13) 72(100) bytes of data.
80 bytes from 10.10.10.13: icmp_seq=1 ttl=64 time=4.40 ms
80 bytes from 10.10.10.13: icmp_seq=2 ttl=64 time=2.01 ms
80 bytes from 10.10.10.13: icmp_seq=3 ttl=64 time=2.22 ms
80 bytes from 10.10.10.13: icmp_seq=4 ttl=64 time=2.31 ms
80 bytes from 10.10.10.13: icmp_seq=5 ttl=64 time=2.27 ms

--- 10.10.10.13 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 19ms
```
</details>

<details>
<summary><b>Показать результаты на L3</b></summary>

```bash
L3#sh isis neighbors 
 
Instance  VRF      System Id        Type Interface          SNPA              State Hold time   Circuit Id       
   
UNDERLAY  default  S1               L2   Ethernet8          P2P               UP    27          0F               
   
UNDERLAY  default  S2               L2   Ethernet7          P2P               UP    24          0F

L3#sh ip route

 I L2     10.10.10.1/32 [115/20] via 10.10.13.1, Ethernet8 #looback S1
 I L2     10.10.10.2/32 [115/20] via 10.10.23.1, Ethernet7 #looback S2
 I L2     10.10.10.11/32 [115/30] via 10.10.23.1, Ethernet7 #looback L1
                                  via 10.10.13.1, Ethernet8 #looback L1
 I L2     10.10.10.12/32 [115/30] via 10.10.23.1, Ethernet7 #looback L2
                                  via 10.10.13.1, Ethernet8 #looback L2
 C        10.10.10.13/32 is directly connected, Loopback0 #свой loopback
 I L2     10.10.11.0/30 [115/20] via 10.10.13.1, Ethernet8 #Link S1-L1
 I L2     10.10.12.0/30 [115/20] via 10.10.13.1, Ethernet8 #Link S1-L2
 C        10.10.13.0/30 is directly connected, Ethernet8 #Link S1-L3
 I L2     10.10.21.0/30 [115/20] via 10.10.23.1, Ethernet7 #Link S2-L1
 I L2     10.10.22.0/30 [115/20] via 10.10.23.1, Ethernet7 #Link S2-L2
 C        10.10.23.0/30 is directly connected, Ethernet7 #Link S2-L3

L3#sh bfd peers 
DstAddr        MyDisc    YourDisc  Interface/Transport    Type          LastUp 
---------- ----------- ----------- -------------------- ------- ---------------
10.10.13.1 2306285918  1698202898        Ethernet8(20)  normal  04/23/26 08:37 
10.10.23.1 3367167881   224527217        Ethernet7(19)  normal  04/23/26 08:37 

   LastDown            LastDiag    State
-------------- ------------------- -----
         NA       No Diagnostic       Up
         NA       No Diagnostic       Up

L3#ping 10.10.10.11
PING 10.10.10.11 (10.10.10.11) 72(100) bytes of data.
80 bytes from 10.10.10.11: icmp_seq=1 ttl=63 time=8.41 ms
80 bytes from 10.10.10.11: icmp_seq=2 ttl=63 time=4.56 ms
80 bytes from 10.10.10.11: icmp_seq=3 ttl=63 time=4.59 ms
80 bytes from 10.10.10.11: icmp_seq=4 ttl=63 time=4.52 ms
80 bytes from 10.10.10.11: icmp_seq=5 ttl=63 time=3.72 ms

--- 10.10.10.11 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 33ms
```
</details>

Все P2P сети известны, все loopback известны. На Spine 1 известно два равноценных маршрута до Loopback S2 (настройка maximum-path 2), а на Leaf3 известно по 2 маршрута до Loopback L1 и L2 (по числу Spine).

Ping успешен. Задача выполнена!
