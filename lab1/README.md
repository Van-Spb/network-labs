### Проектирование адресного пространства

### Цель
- Собрать схему CLOS и распределить адресное пространство для Underlay сети

### Схема с адресацией IPv4

![address_space.png](address_space.png)

### Настройка оборудования

#### SPINE-1
```
configure
hostname spine-1
interface Loopback1
 ip address 10.0.1.0/32
exit
interface Ethernet1
 ip address 10.2.1.0/31
 description to-leaf-1
exit
interface Ethernet2
 ip address 10.2.1.2/31
 description to-leaf-2
exit
interface Ethernet3
 ip address 10.2.1.4/31
 description to-leaf-3
exit
```

#### SPINE-2
```
configure
interface Loopback1
 ip address 10.0.2.0/32
exit
hostname spine-2
interface Ethernet1
 ip address 10.2.2.0/31
 description to-leaf-1
exit
interface Ethernet2
 ip address 10.2.2.2/31
 description to-leaf-2
exit
interface Ethernet3
 ip address 10.2.2.4/31
 description to-leaf-3
exit
```

#### LEAF-1
```
configure
interface Loopback1
 ip address 10.0.1.1/32
exit
interface Loopback2
 ip address 10.1.1.1/32
exit
hostname leaf-1
interface Ethernet1
 ip address 10.2.1.1/31
 description to-spine-1
exit
interface Ethernet2
 ip address 10.2.2.1/31
 description to-spine-2
exit
```

#### LEAF-2
```
configure
interface Loopback1
 ip address 10.0.1.2/32
exit
interface Loopback2
 ip address 10.1.1.2/32
exit
hostname leaf-2
interface Ethernet1
 ip address 10.2.1.3/31
 description to-spine-1
exit
interface Ethernet2
 ip address 10.2.2.3/31
 description to-spine-2
exit
```

#### LEAF-3
```
configure
interface Loopback1
 ip address 10.0.1.3/32
exit
interface Loopback2
 ip address 10.1.1.3/32
exit
hostname leaf-3
interface Ethernet1
 ip address 10.2.1.5/31
 description to-spine-1
exit
interface Ethernet2
 ip address 10.2.2.5/31
 description to-spine-2
exit
```