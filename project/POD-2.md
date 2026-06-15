### Конфигурация оборудования POD-2

#### SPINE-1
```
configure
!
hostname spine-1-dc-2
!
interface Loopback1
 ip address 11.0.1.0/32
 exit
!
interface Ethernet1
 description to-leaf-1
 no switchport
 mtu 9214
 ip address 11.2.1.0/31
 exit
interface Ethernet2
 no switchport
 mtu 9214
 ip address 11.2.1.2/31
 description to-leaf-2
 exit
interface Ethernet4
 description to-border-leaf
 no switchport
 mtu 9214
 ip address 11.2.1.6/31
 exit
!
peer-filter PF_LEAFS_AS_RANGE
 match as-range 66001-66004 result accept
!
ip routing
!
router bgp 66000
 router-id 11.0.1.0
 maximum-paths 4 ecmp 4
 neighbor UNDERLAY peer group
 neighbor UNDERLAY bfd
 neighbor UNDERLAY timers 3 9
 neighbor UNDERLAY password POD2
 bgp listen range 11.2.1.0/29 peer-group UNDERLAY peer-filter PF_LEAFS_AS_RANGE
 neighbor EVPN peer group
 neighbor EVPN update-source Loopback1
 neighbor EVPN ebgp-multihop 3
 neighbor EVPN next-hop-unchanged
 neighbor EVPN send-community extended
 bgp listen range 11.0.1.0/29 peer-group EVPN peer-filter PF_LEAFS_AS_RANGE
 !
 address-family ipv4
  neighbor UNDERLAY activate
  network 11.0.1.0/32
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
hostname spine-2-dc-2
!
interface Loopback1
 ip address 11.0.2.0/32
 exit
!
interface Ethernet1
 no switchport
 mtu 9214
 ip address 11.2.2.0/31
 description to-leaf-1
 exit
interface Ethernet2
 description to-leaf-2
 no switchport
 mtu 9214
 ip address 11.2.2.2/31
 exit
interface Ethernet4
 description to-border-leaf
 no switchport
 mtu 9214
 ip address 11.2.2.6/31
 exit
!
peer-filter PF_LEAFS_AS_RANGE
 match as-range 66001-66004 result accept
!
ip routing
!
router bgp 66000
 router-id 11.0.2.0
 maximum-paths 4 ecmp 4
 neighbor UNDERLAY peer group
 neighbor UNDERLAY send-community
 neighbor UNDERLAY bfd
 neighbor UNDERLAY timers 3 9
 neighbor UNDERLAY password POD2
 bgp listen range 11.2.2.0/29 peer-group UNDERLAY peer-filter PF_LEAFS_AS_RANGE
 neighbor EVPN peer group
 neighbor EVPN update-source Loopback1
 neighbor EVPN ebgp-multihop 3
 neighbor EVPN next-hop-unchanged
 neighbor EVPN send-community extended
 bgp listen range 11.0.1.0/29 peer-group EVPN peer-filter PF_LEAFS_AS_RANGE
 !
 address-family ipv4
  neighbor UNDERLAY activate
  network 11.0.2.0/32
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
hostname leaf-1-dc-2
!
vlan 10,20
!
interface Loopback1
 ip address 11.0.1.1/32
 exit
!
interface Ethernet1
 description to-spine-1
 no switchport
 mtu 9214
 ip address 11.2.1.1/31
 exit
interface Ethernet2
 description to-spine-2
 no switchport
 mtu 9214
 ip address 11.2.2.1/31
 exit
!
interface Ethernet3
 description to-mh-client-1
 switchport access vlan 10
 mtu 9214
 exit
!
interface Ethernet4
 description to-mh-client-2
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
 bfd vtep evpn interval 50 min-rx 50 multiplier 3
!
ip routing
!
router bgp 66001
 router-id 11.0.1.1
 maximum-paths 4 ecmp 4
 neighbor UNDERLAY peer group
 neighbor UNDERLAY bfd
 neighbor UNDERLAY timers 3 9
 neighbor UNDERLAY password POD2
 neighbor UNDERLAY allowas-in 1
 neighbor UNDERLAY remote-as 66000
 neighbor 11.2.1.0 peer group UNDERLAY
 neighbor 11.2.2.0 peer group UNDERLAY
 neighbor EVPN peer group
 neighbor EVPN remote-as 66000
 neighbor EVPN update-source Loopback1
 neighbor EVPN ebgp-multihop 3
 neighbor EVPN send-community extended
 neighbor 11.0.1.0 peer group EVPN
 neighbor 11.0.2.0 peer group EVPN
 !
 vlan 10
  rd 66001:10010
  route-target both 10:10010
  redistribute learned
 !
 vlan 20
  rd 66001:10020
  route-target both 20:10020
  redistribute learned
 !
 vrf EVPN_IRB
  rd 66001:11000
  route-target import evpn 1000:11000
  route-target export evpn 1000:11000
  redistribute connected
  exit
 !
 address-family ipv4
 neighbor UNDERLAY activate
 network 11.0.1.1/32
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
hostname leaf-2-dc-2
!
vlan 10,20
!
interface Loopback1
 ip address 11.0.1.2/32
 exit
!
interface Ethernet1
 description to-spine-1
 no switchport
 mtu 9214
 ip address 11.2.1.3/31
 exit
 !
 interface Ethernet2
 description to-spine-2
 no switchport
 mtu 9214
 ip address 11.2.2.3/31
 exit
!
interface Ethernet3
 description to-mh-client-1
 switchport access vlan 10
 mtu 9214
 exit
!
interface Ethernet4
 description to-mh-client-2
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
 bfd vtep evpn interval 50 min-rx 50 multiplier 3
!
ip routing
!
router bgp 66002
 router-id 11.0.1.2
 maximum-paths 4 ecmp 4
 neighbor UNDERLAY peer group
 neighbor UNDERLAY bfd
 neighbor UNDERLAY timers 3 9
 neighbor UNDERLAY password POD2
 neighbor UNDERLAY allowas-in 1
 neighbor UNDERLAY remote-as 66000
 neighbor 11.2.1.2 peer group UNDERLAY
 neighbor 11.2.2.2 peer group UNDERLAY
 neighbor EVPN peer group
 neighbor EVPN remote-as 66000
 neighbor EVPN update-source Loopback1
 neighbor EVPN ebgp-multihop 3
 neighbor EVPN send-community extended
 neighbor 11.0.1.0 peer group EVPN
 neighbor 11.0.2.0 peer group EVPN
 !
 vlan 10
  rd 66002:10010
  route-target both 10:10010
  redistribute learned
 !
 vlan 20
  rd 66002:10020
  route-target both 20:10020
  redistribute learned
 !
 vrf EVPN_IRB
  rd 66002:11000
  route-target import evpn 1000:11000
  route-target export evpn 1000:11000
  redistribute connected
  exit
 !
 address-family ipv4
  neighbor UNDERLAY activate
  network 11.0.1.2/32
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

---

#### BORDER-LEAF
```
configure
!
hostname border-leaf-dc-2
!
vlan 10,20
!
interface Loopback1
 ip address 11.0.1.4/32
 exit
!
interface Ethernet1
 description to-spine-1
 no switchport
 mtu 9214
 ip address 11.2.1.7/31
 exit
interface Ethernet2
 description to-spine-2
 no switchport
 mtu 9214
 ip address 11.2.2.7/31
 exit
!
interface Ethernet3
 description to-border-leaf-dc-1
 no switchport
 mtu 9214
 ip address 172.16.0.1/31
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
router bgp 66004
 router-id 11.0.1.4
 maximum-paths 4 ecmp 4
 neighbor UNDERLAY peer group
 neighbor UNDERLAY bfd
 neighbor UNDERLAY timers 3 9
 neighbor UNDERLAY password POD2
 neighbor UNDERLAY allowas-in 1
 neighbor UNDERLAY remote-as 66000
 neighbor 11.2.1.6 peer group UNDERLAY
 neighbor 11.2.2.6 peer group UNDERLAY
 neighbor EVPN peer group
 neighbor EVPN remote-as 66000
 neighbor EVPN update-source Loopback1
 neighbor EVPN ebgp-multihop 3
 neighbor EVPN send-community extended
 neighbor 11.0.1.0 peer group EVPN
 neighbor 11.0.2.0 peer group EVPN
 neighbor DCI peer group
 neighbor DCI remote-as 65004
 neighbor DCI send-community extended
 neighbor DCI password DCI1-2
 neighbor 172.16.0.0 peer group DCI
 !
 vlan 10
  rd evpn domain all 66004:10010
  route-target import export evpn domain all 10:10010
 !
 vlan 20
  rd evpn domain all 66004:10020
  route-target import export evpn domain all 20:10020
 !
 vrf EVPN_IRB
  rd 66004:11000
  route-target import evpn 1000:11000
  route-target export evpn 1000:11000
  exit
 ! 
 address-family ipv4
 neighbor UNDERLAY activate
 network 11.0.1.4/32
 neighbor DCI activate
 exit
 !
 address-family evpn
  neighbor EVPN activate
  neighbor DCI activate
  neighbor DCI domain remote
  domain identifier 66004:1
  neighbor default next-hop-self received-evpn-routes route-type ip-prefix inter-domain
  exit
 !
interface Ethernet 1-2
 bfd interval 100 min_rx 100 multiplier 3
 exit
```