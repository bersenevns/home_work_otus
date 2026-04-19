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

## ⚙️ Конфигурация

1) На интерфейсах настроен MTU 9214
2) Настроен router-id
3) По умолчанию все интерфейсы в OSPF являются пассивными
4) Network-type Point-to-point

Конфигурация устройст представлена ниже, лишние строки удалены в целях читаемости.

### S1

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
