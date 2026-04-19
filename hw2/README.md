# OSPF Underlay (Spine–Leaf)

## Цель

Настроить протокол динамической маршрутизации OSPF в качестве underlay-сети для топологии Spine–Leaf (2 Spine + 3 Leaf).

Цель:
- обеспечить доступность всех loopback-интерфейсов
- обеспечить доступность всех underlay-сетей
- убедиться, что маршруты распространяются через OSPF

---

## Схема

![схема](scheme.png)

---

## 🌐 Адресация

### Loopback-интерфейсы

| Устройство | Loopback0      |
|------------|----------------|
| S1         | 10.10.10.1/32  |
| S2         | 10.10.10.2/32  |
| L1         | 10.10.10.11/32 |
| L2         | 10.10.10.12/32 |
| L3         | 10.10.10.13/32 |

---

### Underlay (p2p-сети)

| Линк     | Сеть          |
|----------|---------------|
| S1 — L1  | 10.10.11.0/30 |
| S1 — L2  | 10.10.12.0/30 |
| S1 — L3  | 10.10.13.0/30 |
| S2 — L1  | 10.10.21.0/30 |
| S2 — L2  | 10.10.22.0/30 |
| S2 — L3  | 10.10.23.0/30 |

---

## ⚙️ Конфигурация

### S1

```bash
S1#sh run
! Command: show running-config
! device: S1 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model ribd
!
hostname S1
!
spanning-tree mode mstp
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
interface Ethernet4
!
interface Ethernet5
!
interface Ethernet6
!
interface Ethernet7
!
interface Ethernet8
!
interface Loopback0
   ip address 10.10.10.1/32
   ip ospf area 0.0.0.0
!
interface Management1
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
