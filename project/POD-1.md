### Конфигурация оборудования POD-1

#### SPINE-1
```
configure
!
hostname spine-1-dc-1
!
interface Loopback1
 ip address 10.0.1.0/32
 exit
!
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
interface Ethernet4
 description to-border-leaf
 no switchport
 mtu 9214
 ip address 10.2.1.6/31
 exit
!
peer-filter PF_LEAFS_AS_RANGE
 match as-range 65001-65004 result accept
!
ip routing
!
router bgp 65000
 router-id 10.0.1.0
 maximum-paths 4 ecmp 4
 neighbor UNDERLAY peer group
 neighbor UNDERLAY bfd
 neighbor UNDERLAY timers 3 9
 neighbor UNDERLAY password POD1
 bgp listen range 10.2.1.0/29 peer-group UNDERLAY peer-filter PF_LEAFS_AS_RANGE
 neighbor EVPN peer group
 neighbor EVPN update-source Loopback1
 neighbor EVPN ebgp-multihop 3
 neighbor EVPN next-hop-unchanged
 neighbor EVPN send-community extended
 bgp listen range 10.0.1.0/29 peer-group EVPN peer-filter PF_LEAFS_AS_RANGE
 !
 address-family ipv4
  neighbor UNDERLAY activate
  network 10.0.1.0/32
  exit
 !
 address-family evpn
   neighbor EVPN activate
   exit
 !
interface Ethernet 1-3
 bfd interval 100 min_rx 100 multiplier 3
 exit
```

#### SPINE-2
```
configure
!
hostname spine-2-dc-1
!
interface Loopback1
 ip address 10.0.2.0/32
 exit
!
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
interface Ethernet4
 description to-border-leaf
 no switchport
 mtu 9214
 ip address 10.2.2.6/31
 exit
!
peer-filter PF_LEAFS_AS_RANGE
 match as-range 65001-65004 result accept
!
ip routing
!
router bgp 65000
 router-id 10.0.2.0
 maximum-paths 4 ecmp 4
 neighbor UNDERLAY peer group
 neighbor UNDERLAY send-community
 neighbor UNDERLAY bfd
 neighbor UNDERLAY timers 3 9
 neighbor UNDERLAY password POD1
 bgp listen range 10.2.2.0/29 peer-group UNDERLAY peer-filter PF_LEAFS_AS_RANGE
 neighbor EVPN peer group
 neighbor EVPN update-source Loopback1
 neighbor EVPN ebgp-multihop 3
 neighbor EVPN next-hop-unchanged
 neighbor EVPN send-community extended
 bgp listen range 10.0.1.0/29 peer-group EVPN peer-filter PF_LEAFS_AS_RANGE
 !
 address-family ipv4
  neighbor UNDERLAY activate
  network 10.0.2.0/32
  exit
 !
 address-family evpn
   neighbor EVPN activate
   exit
 !
interface Ethernet 1-3
 bfd interval 100 min_rx 100 multiplier 3
 exit
```

---

#### LEAF-1
```
configure
!
hostname leaf-1-dc-1
!
vlan 10,20
!
interface Loopback1
 ip address 10.0.1.1/32
 exit
!
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
interface port-channel 1
 description pc-to-mh-client-1
 switchport mode access
 switchport access vlan 10
 evpn ethernet-segment
 identifier 0000:0000:0000:0000:0001
 route-target import 00:00:00:00:00:01
 lacp system-id 0000.0000.1111
!
interface port-channel 2
 description pc-to-mh-client-2
 switchport mode access
 switchport access vlan 20
 evpn ethernet-segment
 identifier 0000:0000:0000:0000:0002
 route-target import 00:00:00:00:00:02
 lacp system-id 0000.0000.2222
 !
interface Ethernet3
 description to-mh-client-1
 channel-group 1 mode active
 mtu 9214
 exit
!
interface Ethernet4
 description to-mh-client-2
 channel-group 2 mode active
 mtu 9214
 exit
!
vrf instance EVPN_IRB
ip routing vrf EVPN_IRB
!
ip virtual-router mac-address 00:11:22:33:44:55
!
interface vlan 10
 vrf EVPN_IRB
 ip address virtual 192.168.10.254/24
!
interface vlan 20
 vrf EVPN_IRB
 ip address virtual 192.168.20.254/24
!
ip prefix-list PL_VTEP_BACKUP permit 10.0.1.2/32
!
interface vxlan 1
 vxlan source-interface Loopback1
 vxlan udp-port 4789
 vxlan vlan 10 vni 10010
 vxlan vlan 20 vni 10020
 vxlan vrf EVPN_IRB vni 11000
 bfd vtep evpn interval 50 min-rx 50 multiplier 3
 bfd vtep evpn prefix-list PL_VTEP_BACKUP
!
ip routing
!
router bgp 65001
 router-id 10.0.1.1
 maximum-paths 4 ecmp 4
 neighbor UNDERLAY peer group
 neighbor UNDERLAY bfd
 neighbor UNDERLAY timers 3 9
 neighbor UNDERLAY password POD1
 neighbor UNDERLAY allowas-in 1
 neighbor UNDERLAY remote-as 65000
 neighbor 10.2.1.0 peer group UNDERLAY
 neighbor 10.2.2.0 peer group UNDERLAY
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
 !
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
 neighbor UNDERLAY activate
 network 10.0.1.1/32
 exit
 !
 address-family evpn
  neighbor EVPN activate
  exit
 !
link tracking group UPLINK-TRACKING
 links minimum 1
 recovery delay 1
 exit
 !
interface Ethernet 1-2
 bfd interval 100 min_rx 100 multiplier 3
 link tracking group UPLINK_TRACKING upstream
 exit
 !
interface Ethernet 3-4
 link tracking group UPLINK_TRACKING downstream
 exit
```

#### LEAF-2
```
configure
!
hostname leaf-2-dc-1
!
vlan 10,20
!
interface Loopback1
 ip address 10.0.1.2/32
 exit
!
interface Ethernet1
 description to-spine-1
 no switchport
 mtu 9214
 ip address 10.2.1.3/31
 exit
 !
 interface Ethernet2
 description to-spine-2
 no switchport
 mtu 9214
 ip address 10.2.2.3/31
 exit
!
interface port-channel 1
 description pc-to-mh-client-1
 switchport mode access
 switchport access vlan 10
 evpn ethernet-segment
 identifier 0000:0000:0000:0000:0001
 route-target import 00:00:00:00:00:01
 lacp system-id 0000.0000.1111
!
interface port-channel 2
 description pc-to-mh-client-2
 switchport mode access
 switchport access vlan 20
 evpn ethernet-segment
 identifier 0000:0000:0000:0000:0002
 route-target import 00:00:00:00:00:02
 lacp system-id 0000.0000.2222
 !
interface Ethernet3
 description to-mh-client-1
 channel-group 1 mode active
 mtu 9214
 exit
!
interface Ethernet4
 description to-mh-client-2
 channel-group 2 mode active
 mtu 9214
 exit
!
vrf instance EVPN_IRB
ip routing vrf EVPN_IRB
!
ip virtual-router mac-address 00:11:22:33:44:55
!
interface vlan 10
 vrf EVPN_IRB
 ip address virtual 192.168.10.254/24
!
interface vlan 20
 vrf EVPN_IRB
 ip address virtual 192.168.20.254/24
!
ip prefix-list PL_VTEP_BACKUP permit 10.0.1.1/32
!
interface vxlan 1
 vxlan source-interface Loopback1
 vxlan udp-port 4789
 vxlan vlan 10 vni 10010
 vxlan vlan 20 vni 10020
 vxlan vrf EVPN_IRB vni 11000
 bfd vtep evpn interval 50 min-rx 50 multiplier 3
 bfd vtep evpn prefix-list PL_VTEP_BACKUP
!
ip routing
!
router bgp 65002
 router-id 10.0.1.2
 maximum-paths 4 ecmp 4
 neighbor UNDERLAY peer group
 neighbor UNDERLAY bfd
 neighbor UNDERLAY timers 3 9
 neighbor UNDERLAY password POD1
 neighbor UNDERLAY allowas-in 1
 neighbor UNDERLAY remote-as 65000
 neighbor 10.2.1.2 peer group UNDERLAY
 neighbor 10.2.2.2 peer group UNDERLAY
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
 vlan 20
  rd 65002:10020
  route-target both 20:10020
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
  neighbor UNDERLAY activate
  network 10.0.1.2/32
  exit
 !
 address-family evpn
  neighbor EVPN activate
  exit
 !
link tracking group UPLINK-TRACKING
 links minimum 1
 recovery delay 1
 exit
 !
interface Ethernet 1-2
 bfd interval 100 min_rx 100 multiplier 3
 link tracking group UPLINK_TRACKING upstream
 exit
 !
interface Ethernet 3-4
 link tracking group UPLINK_TRACKING downstream
 exit
```

#### LEAF-3
```
configure
!
hostname leaf-3-dc-1
!
vlan 10,20
!
interface Loopback1
 ip address 10.0.1.3/32
 exit
!
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
interface Ethernet3
 description to-client-1
 switchport access vlan 10
 mtu 9214
 exit
!
interface Ethernet4
 description to-client-2
 switchport access vlan 20
 mtu 9214
 exit
!
vrf instance EVPN_IRB
ip routing vrf EVPN_IRB
!
ip virtual-router mac-address 00:11:22:33:44:55
!
interface vlan 10
 vrf EVPN_IRB
 ip address virtual 192.168.10.254/24
!
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
!
router bgp 65003
 router-id 10.0.1.3
 maximum-paths 4 ecmp 4
 neighbor UNDERLAY peer group
 neighbor UNDERLAY bfd
 neighbor UNDERLAY timers 3 9
 neighbor UNDERLAY password POD1
 neighbor UNDERLAY allowas-in 1
 neighbor UNDERLAY remote-as 65000
 neighbor 10.2.1.4 peer group UNDERLAY
 neighbor 10.2.2.4 peer group UNDERLAY
 neighbor EVPN peer group
 neighbor EVPN remote-as 65000
 neighbor EVPN update-source Loopback1
 neighbor EVPN ebgp-multihop 3
 neighbor EVPN send-community extended
 neighbor 10.0.1.0 peer group EVPN
 neighbor 10.0.2.0 peer group EVPN
 !
 vlan 10
  rd 65003:10010
  route-target both 10:10010
  redistribute learned
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
 neighbor UNDERLAY activate
 network 10.0.1.3/32
 exit
 !
 address-family evpn
  neighbor EVPN activate
  exit
 !
link tracking group UPLINK-TRACKING
 links minimum 1
 recovery delay 1
 exit
 !
interface Ethernet 1-2
 bfd interval 100 min_rx 100 multiplier 3
 link tracking group UPLINK_TRACKING upstream
 exit
 !
interface Ethernet 3-4
 link tracking group UPLINK_TRACKING downstream
 exit
```

---

#### BORDER-LEAF
```
configure
!
hostname border-leaf-dc-1
!
vlan 10,20
!
interface Loopback1
 ip address 10.0.1.4/32
 exit
!
interface Ethernet1
 description to-spine-1
 no switchport
 mtu 9214
 ip address 10.2.1.7/31
 exit
interface Ethernet2
 description to-spine-2
 no switchport
 mtu 9214
 ip address 10.2.2.7/31
 exit
!
interface Ethernet3
 description to-border-leaf-dc-2
 no switchport
 mtu 9214
 ip address 172.16.0.0/31
 exit
!
vrf instance EVPN_IRB
ip routing vrf EVPN_IRB
!
ip virtual-router mac-address 00:11:22:33:44:55
!
interface vlan 10
 vrf EVPN_IRB
 ip address virtual 192.168.10.254/24
!
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
!
router bgp 65004
 router-id 10.0.1.4
 maximum-paths 4 ecmp 4
 neighbor UNDERLAY peer group
 neighbor UNDERLAY bfd
 neighbor UNDERLAY timers 3 9
 neighbor UNDERLAY password POD1
 neighbor UNDERLAY allowas-in 1
 neighbor UNDERLAY remote-as 65000
 neighbor 10.2.1.6 peer group UNDERLAY
 neighbor 10.2.2.6 peer group UNDERLAY
 neighbor EVPN peer group
 neighbor EVPN remote-as 65000
 neighbor EVPN update-source Loopback1
 neighbor EVPN ebgp-multihop 3
 neighbor EVPN send-community extended
 neighbor 10.0.1.0 peer group EVPN
 neighbor 10.0.2.0 peer group EVPN
 neighbor DCI peer group
 neighbor DCI remote-as 66004
 neighbor DCI password DCI1-2
 neighbor DCI send-community extended
 
 neighbor 172.16.0.1 peer group DCI
 !
 vlan 10
  rd evpn domain all 65004:10010
  route-target import export evpn domain all 10:10010
 !
 vlan 20
  rd evpn domain all 65004:10020
  route-target import export evpn domain all 20:10020
 !
 vrf EVPN_IRB
  rd 65004:11000
  route-target import evpn 1000:11000
  route-target export evpn 1000:11000
  exit
 ! 
 address-family ipv4
 neighbor UNDERLAY activate
 network 10.0.1.4/32
 neighbor DCI activate
 exit
 !
 address-family evpn
  neighbor EVPN activate
  neighbor DCI activate
  neighbor DCI domain remote
  domain identifier 65004:1
  domain identifier 65004:1 remote
  neighbor default next-hop-self received-evpn-routes route-type ip-prefix inter-domain
  exit
 !
interface Ethernet 1-2
 bfd interval 100 min_rx 100 multiplier 3
 exit
```

---

#### SERVER-1
```
configure
!
hostname SERVER-1
!
vlan 10
!
interface port-channel 1
 description mlag-to-leaf-1-2
 switchport mode access
 switchport access vlan 10
!
interface Ethernet1
 description to-leaf-1
 channel-group 1 mode active
 mtu 9214
 exit
!
interface Ethernet2
 description to-leaf-2
 channel-group 1 mode active
 mtu 9214
 exit
!
ip routing
!
interface Vlan10
 ip address 192.168.10.1/24
!
ip route  0.0.0.0/0 192.168.10.254
!
```

#### SERVER-2
```
configure
!
hostname SERVER-2
!
vlan 20
!
interface port-channel 1
 description mlag-to-leaf-1-2
 switchport mode access
 switchport access vlan 20
!
interface Ethernet1
 description to-leaf-1
 channel-group 1 mode active
 mtu 9214
 exit
!
interface Ethernet2
 description to-leaf-2
 channel-group 1 mode active
 mtu 9214
 exit
!
ip routing
!
interface Vlan20
 ip address 192.168.20.1/24
!
ip route  0.0.0.0/0 192.168.20.254
!
```