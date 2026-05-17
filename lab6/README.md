### VxLAN. L3 VNI

### Цель
- Настроить маршрутизацию в рамках Overlay между клиентами.

### Схема

![topology.png](topology.png)

### Настройка оборудования

#### SPINE-1
```
configure
!
hostname spine-1
!
interface Loopback1
 ip address 10.0.1.0/32
 exit
interface Ethernet1
 description to-leaf-1
 no switchport
 mtu 9214
 ip address 10.2.1.0/31
 exit
interface Ethernet2
 no switchport
 mtu 9214
 ip address 10.2.1.2/31
 description to-leaf-2
 exit
interface Ethernet3
 description to-leaf-3
 no switchport
 mtu 9214
 ip address 10.2.1.4/31
 exit
!
peer-filter PF_LEAFS_AS_RANGE
 match as-range 65001-65003 result accept
!
ip routing
router bgp 65000
 router-id 10.0.1.0
 maximum-paths 4 ecmp 4
 neighbor LEAFS peer group
 neighbor LEAFS bfd
 neighbor LEAFS timers 3 9
 neighbor LEAFS password LAB6KEY
 bgp listen range 10.2.1.0/29 peer-group LEAFS peer-filter PF_LEAFS_AS_RANGE
 neighbor EVPN peer group
 neighbor EVPN update-source Loopback1
 neighbor EVPN send-community extended
 bgp listen range 10.0.1.0/30 peer-group EVPN peer-filter PF_LEAFS_AS_RANGE
 !
 address-family ipv4
  neighbor LEAFS activate
  network 10.0.1.0/32
  exit
 !
 address-family evpn
   neighbor EVPN activate
 !
interface Ethernet 1-3
 bfd interval 100 min_rx 100 multiplier 3
 exit
```

#### SPINE-2
```
configure
!
hostname spine-2
!
interface Loopback1
 ip address 10.0.2.0/32
 exit
interface Ethernet1
 no switchport
 mtu 9214
 ip address 10.2.2.0/31
 description to-leaf-1
 exit
interface Ethernet2
 description to-leaf-2
 no switchport
 mtu 9214
 ip address 10.2.2.2/31
 exit
interface Ethernet3
 description to-leaf-3
 no switchport
 mtu 9214
 ip address 10.2.2.4/31
 exit
!
peer-filter PF_LEAFS_AS_RANGE
 match as-range 65001-65003 result accept
!
ip routing
router bgp 65000
 router-id 10.0.2.0
 maximum-paths 4 ecmp 4
 neighbor LEAFS peer group
 neighbor LEAFS send-community
 neighbor LEAFS bfd
 neighbor LEAFS timers 3 9
 neighbor LEAFS password LAB6KEY
 bgp listen range 10.2.2.0/29 peer-group LEAFS peer-filter PF_LEAFS_AS_RANGE
 neighbor EVPN peer group
 neighbor EVPN update-source Loopback1
 neighbor EVPN send-community extended
 bgp listen range 10.0.1.0/30 peer-group EVPN peer-filter PF_LEAFS_AS_RANGE
 !
 address-family ipv4
  neighbor LEAFS activate
  network 10.0.2.0/32
  exit
 !
 address-family evpn
   neighbor EVPN activate
 !
interface Ethernet 1-3
 bfd interval 100 min_rx 100 multiplier 3
 exit
```

#### LEAF-1
```
configure
!
hostname leaf-1
!
vlan 10
vlan 20
!
vrf instance EVPN_IRB
ip routing vrf EVPN_IRB
!
interface Loopback1
 ip address 10.0.1.1/32
 exit
interface Loopback2
 ip address 10.1.1.1/32
 exit
interface Ethernet1
 description to-spine-1
 no switchport
 mtu 9214
 ip address 10.2.1.1/31
 exit
interface Ethernet2
 description to-spine-2
 no switchport
 mtu 9214
 ip address 10.2.2.1/31
 exit
!
interface Ethernet3
 description to-client-1
 switchport access vlan 10
 mtu 9214
 exit
interface Ethernet4
 description to-client-2
 switchport access vlan 20
 mtu 9214
 exit
!
ip virtual-router mac-address 00:11:22:33:44:55
!
interface vlan 10
 vrf EVPN_IRB
 ip address virtual 192.168.10.254/24
interface vlan 20
 vrf EVPN_IRB
 ip address virtual 192.168.20.254/24
!
interface vxlan 1
 vxlan source-interface Loopback1
 vxlan udp-port 4789
 vxlan vlan 10 vni 10010
 vxlan vlan 20 vni 10020
 vxlan vrf EVPN_IRB vni 11000
!
ip routing
router bgp 65001
 router-id 10.0.1.1
 maximum-paths 4 ecmp 4
 neighbor SPINES peer group
 neighbor SPINES bfd
 neighbor SPINES timers 3 9
 neighbor SPINES password LAB6KEY
 neighbor SPINES allowas-in 1
 neighbor SPINES remote-as 65000
 neighbor 10.2.1.0 peer group SPINES
 neighbor 10.2.2.0 peer group SPINES
 neighbor EVPN peer group
 neighbor EVPN remote-as 65000
 neighbor EVPN update-source Loopback1
 neighbor EVPN ebgp-multihop 3
 neighbor EVPN send-community extended
 neighbor 10.0.1.0 peer group EVPN
 neighbor 10.0.2.0 peer group EVPN
 !
 vlan 10
  rd 65001:10010
  route-target both 10:10010
  redistribute learned
 vlan 20
  rd 65001:10020
  route-target both 20:10020
  redistribute learned
 !
 vrf EVPN_IRB
  rd 65001:11000
  route-target import evpn 1000:11000
  route-target export evpn 1000:11000
  redistribute connected
  exit
 !
 address-family ipv4
 neighbor SPINES activate
 network 10.0.1.1/32
 exit
 !
 address-family evpn
  neighbor EVPN activate
  exit
 !
interface Ethernet 1-2
 bfd interval 100 min_rx 100 multiplier 3
 exit
```

#### LEAF-2
```
configure
!
hostname leaf-2
!
vlan 10
!
vrf instance EVPN_IRB
ip routing vrf EVPN_IRB
!
interface Loopback1
 ip address 10.0.1.2/32
 exit
interface Loopback2
 ip address 10.1.1.2/32
 exit
interface Ethernet1
 description to-spine-1
 no switchport
 mtu 9214
 ip address 10.2.1.3/31
 exit
interface Ethernet2
 description to-spine-2
 no switchport
 mtu 9214
 ip address 10.2.2.3/31
 exit
!
interface Ethernet3
 description to-client-1
 switchport access vlan 10
 mtu 9214
 exit
!
ip virtual-router mac-address 00:11:22:33:44:55
!
interface vlan 10
 vrf EVPN_IRB
 ip address virtual 192.168.10.254/24
!
interface vxlan 1
 vxlan source-interface Loopback1
 vxlan udp-port 4789
 vxlan vlan 10 vni 10010
 vxlan vrf EVPN_IRB vni 11000
!
ip routing
router bgp 65002
 router-id 10.0.1.2
 maximum-paths 4 ecmp 4
 neighbor SPINES peer group
 neighbor SPINES bfd
 neighbor SPINES timers 3 9
 neighbor SPINES password LAB6KEY
 neighbor SPINES allowas-in 1
 neighbor SPINES remote-as 65000
 neighbor 10.2.1.2 peer group SPINES
 neighbor 10.2.2.2 peer group SPINES
 neighbor EVPN peer group
 neighbor EVPN remote-as 65000
 neighbor EVPN update-source Loopback1
 neighbor EVPN ebgp-multihop 3
 neighbor EVPN send-community extended
 neighbor 10.0.1.0 peer group EVPN
 neighbor 10.0.2.0 peer group EVPN
 !
 vlan 10
  rd 65002:10010
  route-target both 10:10010
  redistribute learned
 !
 vrf EVPN_IRB
  rd 65002:11000
  route-target import evpn 1000:11000
  route-target export evpn 1000:11000
  redistribute connected
  exit
 !
 address-family ipv4
  neighbor SPINES activate
  network 10.0.1.2/32
  exit
 !
 address-family evpn
  neighbor EVPN activate
  exit
 !
interface Ethernet 1-2
 bfd interval 100 min_rx 100 multiplier 3
 exit
```

#### LEAF-3
```
configure
!
hostname leaf-3
!
vlan 20
!
vrf instance EVPN_IRB
ip routing vrf EVPN_IRB
!
interface Loopback1
 ip address 10.0.1.3/32
 exit
interface Loopback2
 ip address 10.1.1.3/32
 exit
interface Ethernet1
 description to-spine-1
 no switchport
 mtu 9214
 ip address 10.2.1.5/31
 exit
interface Ethernet2
 description to-spine-2
 no switchport
 mtu 9214
 ip address 10.2.2.5/31
 exit
!
interface Ethernet4
 description to-client-2
 switchport access vlan 20
 mtu 9214
 exit
!
ip virtual-router mac-address 00:11:22:33:44:55
!
interface vlan 20
 vrf EVPN_IRB
 ip address virtual 192.168.20.254/24
!
interface vxlan 1
 vxlan source-interface Loopback1
 vxlan udp-port 4789
 vxlan vlan 20 vni 10020
 vxlan vrf EVPN_IRB vni 11000
!
ip routing
router bgp 65003
 router-id 10.0.1.3
 maximum-paths 4 ecmp 4
 neighbor SPINES peer group
 neighbor SPINES bfd
 neighbor SPINES timers 3 9
 neighbor SPINES password LAB6KEY
 neighbor SPINES allowas-in 1
 neighbor SPINES remote-as 65000
 neighbor 10.2.1.4 peer group SPINES
 neighbor 10.2.2.4 peer group SPINES
 neighbor EVPN peer group
 neighbor EVPN remote-as 65000
 neighbor EVPN update-source Loopback1
 neighbor EVPN ebgp-multihop 3
 neighbor EVPN send-community extended
 neighbor 10.0.1.0 peer group EVPN
 neighbor 10.0.2.0 peer group EVPN
 !
 vlan 20
  rd 65003:10020
  route-target both 20:10020
  redistribute learned
 !
 vrf EVPN_IRB
 rd 65003:11000
 route-target import evpn 1000:11000
 route-target export evpn 1000:11000
 redistribute connected
 exit
 !
 address-family ipv4
 neighbor SPINES activate
 network 10.0.1.3/32
 exit
 !
 address-family evpn
  neighbor EVPN activate
  exit
 !
interface Ethernet 1-2
 bfd interval 100 min_rx 100 multiplier 3
 exit
```

### Проверка примененных настроек

#### LEAF-1

```
leaf-1(config)#show ip bgp summary 
BGP summary information for VRF default
Router identifier 10.0.1.1, local AS number 65001
Neighbor Status Codes: m - Under maintenance
  Neighbor         V  AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  10.0.1.0         4  65000            680       689    0    0 09:05:44 Estab   3      3
  10.0.2.0         4  65000            673       700    0    0 09:05:44 Estab   3      3
  10.2.1.0         4  65000          12818     12821    0    0 09:05:45 Estab   3      3
  10.2.2.0         4  65000          12808     12814    0    0 09:05:45 Estab   3      3
leaf-1(config)#
leaf-1(config)#
leaf-1(config)#show bgp evpn route-type mac-ip 
BGP routing table information for VRF default
Router identifier 10.0.1.1, local AS number 65001
Route status codes: s - suppressed, * - valid, > - active, # - not installed, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >     RD: 65001:10010 mac-ip 0050.7966.685f
                                -                     -       -       0       i
 * >     RD: 65001:10010 mac-ip 0050.7966.685f 192.168.10.1
                                -                     -       -       0       i
 * >Ec   RD: 65003:10020 mac-ip 0050.7966.6860
                                10.0.1.3              -       100     0       65000 65003 i
 *  ec   RD: 65003:10020 mac-ip 0050.7966.6860
                                10.0.1.3              -       100     0       65000 65003 i
 * >Ec   RD: 65003:10020 mac-ip 0050.7966.6860 192.168.20.2
                                10.0.1.3              -       100     0       65000 65003 i
 *  ec   RD: 65003:10020 mac-ip 0050.7966.6860 192.168.20.2
                                10.0.1.3              -       100     0       65000 65003 i
 * >Ec   RD: 65002:10010 mac-ip 0050.7966.6861
                                10.0.1.2              -       100     0       65000 65002 i
 *  ec   RD: 65002:10010 mac-ip 0050.7966.6861
                                10.0.1.2              -       100     0       65000 65002 i
 * >Ec   RD: 65002:10010 mac-ip 0050.7966.6861 192.168.10.2
                                10.0.1.2              -       100     0       65000 65002 i
 *  ec   RD: 65002:10010 mac-ip 0050.7966.6861 192.168.10.2
                                10.0.1.2              -       100     0       65000 65002 i
 * >     RD: 65001:10020 mac-ip 0050.7966.6869
                                -                     -       -       0       i
 * >     RD: 65001:10020 mac-ip 0050.7966.6869 192.168.20.1
                                -                     -       -       0       i
leaf-1(config)#
leaf-1(config)#show bgp evpn route-type ip-prefix ipv4
BGP routing table information for VRF default
Router identifier 10.0.1.1, local AS number 65001
Route status codes: s - suppressed, * - valid, > - active, # - not installed, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >     RD: 65001:11000 ip-prefix 192.168.10.0/24
                                -                     -       -       0       i
 * >Ec   RD: 65002:11000 ip-prefix 192.168.10.0/24
                                10.0.1.2              -       100     0       65000 65002 i
 *  ec   RD: 65002:11000 ip-prefix 192.168.10.0/24
                                10.0.1.2              -       100     0       65000 65002 i
 * >     RD: 65001:11000 ip-prefix 192.168.20.0/24
                                -                     -       -       0       i
 * >Ec   RD: 65003:11000 ip-prefix 192.168.20.0/24
                                10.0.1.3              -       100     0       65000 65003 i
 *  ec   RD: 65003:11000 ip-prefix 192.168.20.0/24
                                10.0.1.3              -       100     0       65000 65003 i
leaf-1(config)#
leaf-1(config)#show mac address-table 
          Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports      Moves   Last Move
----    -----------       ----        -----      -----   ---------
  10    0050.7966.685f    DYNAMIC     Et3        1       0:21:12 ago
  10    0050.7966.6861    DYNAMIC     Vx1        1       0:40:24 ago
  20    0050.7966.6860    DYNAMIC     Vx1        1       0:40:04 ago
  20    0050.7966.6869    DYNAMIC     Et4        1       0:06:30 ago
4094    5022.047a.ee09    DYNAMIC     Vx1        1       0:02:37 ago
4094    50b1.7132.2125    DYNAMIC     Vx1        1       9:05:26 ago
Total Mac Addresses for this criterion: 6

          Multicast Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports
----    -----------       ----        -----
Total Mac Addresses for this criterion: 0
leaf-1(config)#
leaf-1(config)#show ip route vrf EVPN_IRB

VRF: EVPN_IRB
Codes: C - connected, S - static, K - kernel, 
       O - OSPF, IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type2, B - BGP, B I - iBGP, B E - eBGP,
       R - RIP, I L1 - IS-IS level 1, I L2 - IS-IS level 2,
       O3 - OSPFv3, A B - BGP Aggregate, A O - OSPF Summary,
       NG - Nexthop Group Static Route, V - VXLAN Control Service,
       DH - DHCP client installed default route, M - Martian,
       DP - Dynamic Policy Route, L - VRF Leaked

Gateway of last resort is not set

 B E      192.168.10.2/32 [200/0] via VTEP 10.0.1.2 VNI 11000 router-mac 50:22:04:7a:ee:09
 C        192.168.10.0/24 is directly connected, Vlan10
 B E      192.168.20.2/32 [200/0] via VTEP 10.0.1.3 VNI 11000 router-mac 50:b1:71:32:21:25
 C        192.168.20.0/24 is directly connected, Vlan20
```

#### LEAF-2
```
leaf-2(config)#show ip bgp summary 
BGP summary information for VRF default
Router identifier 10.0.1.2, local AS number 65002
Neighbor Status Codes: m - Under maintenance
  Neighbor         V  AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  10.0.1.0         4  65000            686       671    0    0 09:06:12 Estab   3      3
  10.0.2.0         4  65000            690       691    0    0 09:06:11 Estab   3      3
  10.2.1.2         4  65000          12832     12829    0    0 09:06:13 Estab   3      3
  10.2.2.2         4  65000          12868     12834    0    0 09:06:13 Estab   3      3
leaf-2(config)#
leaf-2(config)#show bgp evpn route-type mac-ip 
BGP routing table information for VRF default
Router identifier 10.0.1.2, local AS number 65002
Route status codes: s - suppressed, * - valid, > - active, # - not installed, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec   RD: 65001:10010 mac-ip 0050.7966.685f
                                10.0.1.1              -       100     0       65000 65001 i
 *  ec   RD: 65001:10010 mac-ip 0050.7966.685f
                                10.0.1.1              -       100     0       65000 65001 i
 * >Ec   RD: 65001:10010 mac-ip 0050.7966.685f 192.168.10.1
                                10.0.1.1              -       100     0       65000 65001 i
 *  ec   RD: 65001:10010 mac-ip 0050.7966.685f 192.168.10.1
                                10.0.1.1              -       100     0       65000 65001 i
 * >Ec   RD: 65003:10020 mac-ip 0050.7966.6860
                                10.0.1.3              -       100     0       65000 65003 i
 *  ec   RD: 65003:10020 mac-ip 0050.7966.6860
                                10.0.1.3              -       100     0       65000 65003 i
 * >Ec   RD: 65003:10020 mac-ip 0050.7966.6860 192.168.20.2
                                10.0.1.3              -       100     0       65000 65003 i
 *  ec   RD: 65003:10020 mac-ip 0050.7966.6860 192.168.20.2
                                10.0.1.3              -       100     0       65000 65003 i
 * >     RD: 65002:10010 mac-ip 0050.7966.6861
                                -                     -       -       0       i
 * >     RD: 65002:10010 mac-ip 0050.7966.6861 192.168.10.2
                                -                     -       -       0       i
 * >Ec   RD: 65001:10020 mac-ip 0050.7966.6869
                                10.0.1.1              -       100     0       65000 65001 i
 *  ec   RD: 65001:10020 mac-ip 0050.7966.6869
                                10.0.1.1              -       100     0       65000 65001 i
 * >Ec   RD: 65001:10020 mac-ip 0050.7966.6869 192.168.20.1
                                10.0.1.1              -       100     0       65000 65001 i
 *  ec   RD: 65001:10020 mac-ip 0050.7966.6869 192.168.20.1
                                10.0.1.1              -       100     0       65000 65001 i
leaf-2(config)#
leaf-2(config)#show bgp evpn route-type ip-prefix ipv4
BGP routing table information for VRF default
Router identifier 10.0.1.2, local AS number 65002
Route status codes: s - suppressed, * - valid, > - active, # - not installed, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec   RD: 65001:11000 ip-prefix 192.168.10.0/24
                                10.0.1.1              -       100     0       65000 65001 i
 *  ec   RD: 65001:11000 ip-prefix 192.168.10.0/24
                                10.0.1.1              -       100     0       65000 65001 i
 * >     RD: 65002:11000 ip-prefix 192.168.10.0/24
                                -                     -       -       0       i
 * >Ec   RD: 65001:11000 ip-prefix 192.168.20.0/24
                                10.0.1.1              -       100     0       65000 65001 i
 *  ec   RD: 65001:11000 ip-prefix 192.168.20.0/24
                                10.0.1.1              -       100     0       65000 65001 i
 * >Ec   RD: 65003:11000 ip-prefix 192.168.20.0/24
                                10.0.1.3              -       100     0       65000 65003 i
 *  ec   RD: 65003:11000 ip-prefix 192.168.20.0/24
                                10.0.1.3              -       100     0       65000 65003 i
leaf-2(config)#
leaf-2(config)#show mac address-table 
          Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports      Moves   Last Move
----    -----------       ----        -----      -----   ---------
  10    0050.7966.685f    DYNAMIC     Vx1        1       0:22:50 ago
  10    0050.7966.6861    DYNAMIC     Et3        1       0:42:01 ago
4094    5066.bacb.0a09    DYNAMIC     Vx1        1       9:06:33 ago
4094    50b1.7132.2125    DYNAMIC     Vx1        1       9:06:33 ago
Total Mac Addresses for this criterion: 4

          Multicast Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports
----    -----------       ----        -----
Total Mac Addresses for this criterion: 0
leaf-2(config)#
leaf-2(config)#show ip route vrf EVPN_IRB

VRF: EVPN_IRB
Codes: C - connected, S - static, K - kernel, 
       O - OSPF, IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type2, B - BGP, B I - iBGP, B E - eBGP,
       R - RIP, I L1 - IS-IS level 1, I L2 - IS-IS level 2,
       O3 - OSPFv3, A B - BGP Aggregate, A O - OSPF Summary,
       NG - Nexthop Group Static Route, V - VXLAN Control Service,
       DH - DHCP client installed default route, M - Martian,
       DP - Dynamic Policy Route, L - VRF Leaked

Gateway of last resort is not set

 B E      192.168.10.1/32 [200/0] via VTEP 10.0.1.1 VNI 11000 router-mac 50:66:ba:cb:0a:09
 C        192.168.10.0/24 is directly connected, Vlan10
 B E      192.168.20.1/32 [200/0] via VTEP 10.0.1.1 VNI 11000 router-mac 50:66:ba:cb:0a:09
 B E      192.168.20.2/32 [200/0] via VTEP 10.0.1.3 VNI 11000 router-mac 50:b1:71:32:21:25
 B E      192.168.20.0/24 [200/0] via VTEP 10.0.1.3 VNI 11000 router-mac 50:b1:71:32:21:25
                                  via VTEP 10.0.1.1 VNI 11000 router-mac 50:66:ba:cb:0a:09
```

#### LEAF-3
```
leaf-3(config)#show ip bgp summary 
BGP summary information for VRF default
Router identifier 10.0.1.3, local AS number 65003
Neighbor Status Codes: m - Under maintenance
  Neighbor         V  AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  10.0.1.0         4  65000            686       693    0    0 09:08:06 Estab   3      3
  10.0.2.0         4  65000            692       699    0    0 09:08:05 Estab   3      3
  10.2.1.4         4  65000          12901     12894    0    0 09:08:07 Estab   3      3
  10.2.2.4         4  65000          12881     12867    0    0 09:08:07 Estab   3      3
leaf-3(config)#
leaf-3(config)#show bgp evpn route-type mac-ip 
BGP routing table information for VRF default
Router identifier 10.0.1.3, local AS number 65003
Route status codes: s - suppressed, * - valid, > - active, # - not installed, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec   RD: 65001:10010 mac-ip 0050.7966.685f
                                10.0.1.1              -       100     0       65000 65001 i
 *  ec   RD: 65001:10010 mac-ip 0050.7966.685f
                                10.0.1.1              -       100     0       65000 65001 i
 * >Ec   RD: 65001:10010 mac-ip 0050.7966.685f 192.168.10.1
                                10.0.1.1              -       100     0       65000 65001 i
 *  ec   RD: 65001:10010 mac-ip 0050.7966.685f 192.168.10.1
                                10.0.1.1              -       100     0       65000 65001 i
 * >Ec   RD: 65002:10010 mac-ip 0050.7966.6861
                                10.0.1.2              -       100     0       65000 65002 i
 *  ec   RD: 65002:10010 mac-ip 0050.7966.6861
                                10.0.1.2              -       100     0       65000 65002 i
 * >Ec   RD: 65002:10010 mac-ip 0050.7966.6861 192.168.10.2
                                10.0.1.2              -       100     0       65000 65002 i
 *  ec   RD: 65002:10010 mac-ip 0050.7966.6861 192.168.10.2
                                10.0.1.2              -       100     0       65000 65002 i
 * >Ec   RD: 65001:10020 mac-ip 0050.7966.6869
                                10.0.1.1              -       100     0       65000 65001 i
 *  ec   RD: 65001:10020 mac-ip 0050.7966.6869
                                10.0.1.1              -       100     0       65000 65001 i
 * >Ec   RD: 65001:10020 mac-ip 0050.7966.6869 192.168.20.1
                                10.0.1.1              -       100     0       65000 65001 i
 *  ec   RD: 65001:10020 mac-ip 0050.7966.6869 192.168.20.1
                                10.0.1.1              -       100     0       65000 65001 i
leaf-3(config)#
leaf-3(config)#show bgp evpn route-type ip-prefix ipv4
BGP routing table information for VRF default
Router identifier 10.0.1.3, local AS number 65003
Route status codes: s - suppressed, * - valid, > - active, # - not installed, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec   RD: 65001:11000 ip-prefix 192.168.10.0/24
                                10.0.1.1              -       100     0       65000 65001 i
 *  ec   RD: 65001:11000 ip-prefix 192.168.10.0/24
                                10.0.1.1              -       100     0       65000 65001 i
 * >Ec   RD: 65002:11000 ip-prefix 192.168.10.0/24
                                10.0.1.2              -       100     0       65000 65002 i
 *  ec   RD: 65002:11000 ip-prefix 192.168.10.0/24
                                10.0.1.2              -       100     0       65000 65002 i
 * >Ec   RD: 65001:11000 ip-prefix 192.168.20.0/24
                                10.0.1.1              -       100     0       65000 65001 i
 *  ec   RD: 65001:11000 ip-prefix 192.168.20.0/24
                                10.0.1.1              -       100     0       65000 65001 i
 * >     RD: 65003:11000 ip-prefix 192.168.20.0/24
                                -                     -       -       0       i
leaf-3(config)#
leaf-3(config)#show ip route vrf EVPN_IRB

VRF: EVPN_IRB
Codes: C - connected, S - static, K - kernel, 
       O - OSPF, IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type2, B - BGP, B I - iBGP, B E - eBGP,
       R - RIP, I L1 - IS-IS level 1, I L2 - IS-IS level 2,
       O3 - OSPFv3, A B - BGP Aggregate, A O - OSPF Summary,
       NG - Nexthop Group Static Route, V - VXLAN Control Service,
       DH - DHCP client installed default route, M - Martian,
       DP - Dynamic Policy Route, L - VRF Leaked

Gateway of last resort is not set

 B E      192.168.10.1/32 [200/0] via VTEP 10.0.1.1 VNI 11000 router-mac 50:66:ba:cb:0a:09
 B E      192.168.10.0/24 [200/0] via VTEP 10.0.1.1 VNI 11000 router-mac 50:66:ba:cb:0a:09
                                  via VTEP 10.0.1.2 VNI 11000 router-mac 50:22:04:7a:ee:09
 B E      192.168.20.1/32 [200/0] via VTEP 10.0.1.1 VNI 11000 router-mac 50:66:ba:cb:0a:09
 C        192.168.20.0/24 is directly connected, Vlan20
```

Проверка связности клиентов:

- ping со стороны CLIENT-1-10:
```
CLIENT-1-10> ping 192.168.10.2          

84 bytes from 192.168.10.2 icmp_seq=1 ttl=64 time=9.614 ms
84 bytes from 192.168.10.2 icmp_seq=2 ttl=64 time=8.183 ms
84 bytes from 192.168.10.2 icmp_seq=3 ttl=64 time=8.359 ms
84 bytes from 192.168.10.2 icmp_seq=4 ttl=64 time=8.801 ms
84 bytes from 192.168.10.2 icmp_seq=5 ttl=64 time=7.790 ms

CLIENT-1-10> ping 192.168.20.1

84 bytes from 192.168.20.1 icmp_seq=1 ttl=63 time=7.408 ms
84 bytes from 192.168.20.1 icmp_seq=2 ttl=63 time=4.890 ms
84 bytes from 192.168.20.1 icmp_seq=3 ttl=63 time=4.589 ms
84 bytes from 192.168.20.1 icmp_seq=4 ttl=63 time=4.452 ms
84 bytes from 192.168.20.1 icmp_seq=5 ttl=63 time=4.313 ms

CLIENT-1-10> ping 192.168.20.2

84 bytes from 192.168.20.2 icmp_seq=1 ttl=62 time=11.009 ms
84 bytes from 192.168.20.2 icmp_seq=2 ttl=62 time=12.061 ms
84 bytes from 192.168.20.2 icmp_seq=3 ttl=62 time=11.249 ms
84 bytes from 192.168.20.2 icmp_seq=4 ttl=62 time=12.438 ms
84 bytes from 192.168.20.2 icmp_seq=5 ttl=62 time=11.349 ms
```

- ping со стороны CLIENT-2-10:
```
client-2-10> ping 192.168.10.1

84 bytes from 192.168.10.1 icmp_seq=1 ttl=64 time=7.981 ms
84 bytes from 192.168.10.1 icmp_seq=2 ttl=64 time=8.652 ms
84 bytes from 192.168.10.1 icmp_seq=3 ttl=64 time=7.877 ms
84 bytes from 192.168.10.1 icmp_seq=4 ttl=64 time=8.321 ms
84 bytes from 192.168.10.1 icmp_seq=5 ttl=64 time=8.654 ms

client-2-10> ping 192.168.20.1

84 bytes from 192.168.20.1 icmp_seq=1 ttl=62 time=14.350 ms
84 bytes from 192.168.20.1 icmp_seq=2 ttl=62 time=12.363 ms
84 bytes from 192.168.20.1 icmp_seq=3 ttl=62 time=12.251 ms
84 bytes from 192.168.20.1 icmp_seq=4 ttl=62 time=11.428 ms
84 bytes from 192.168.20.1 icmp_seq=5 ttl=62 time=11.907 ms

client-2-10> ping 192.168.20.2

84 bytes from 192.168.20.2 icmp_seq=1 ttl=62 time=13.862 ms
84 bytes from 192.168.20.2 icmp_seq=2 ttl=62 time=12.746 ms
84 bytes from 192.168.20.2 icmp_seq=3 ttl=62 time=12.728 ms
84 bytes from 192.168.20.2 icmp_seq=4 ttl=62 time=11.585 ms
84 bytes from 192.168.20.2 icmp_seq=5 ttl=62 time=11.456 ms
```



