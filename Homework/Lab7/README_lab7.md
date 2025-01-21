# VXLAN. Multihoming.

### Цели:
- Настроить отказоустойчивое подключение клиентов с использованием EVPN Multihoming


![Схема сети ](lab5.png)
### Адрес план:

|Device|Interface|IP Address|Subnet Mask|Link type
|---|---|---|---|---|
Spine1|lo1|10.0.1.1|255.255.255.255|lo
Spine1 |Eth1|10.2.1.0|255.255.255.254|p2p Spine1 eth1 to Leaf1 eth1
Spine1 |Eth2|10.2.1.2|255.255.255.254|p2p Spine1-eth2 to Leaf2 eth1
Spine1 |Eth3|10.2.1.4|255.255.255.254|p2p Spine1-eth3 to Leaf3-eth1
Spine2 |lo1|10.0.2.1|255.255.255.255|lo
Spine2 |Eth1|10.2.2.0|255.255.255.254|p2p Spine2-eth1 to Leaf1-eth1
Spine2 |Eth2|10.2.2.2|255.255.255.254|p2p Spine2-eth2 to Leaf2-eth2
Spine2 |Eth3|10.2.2.4|255.255.255.254|p2p Spine2-eth3 to Leaf3-eth2
Leaf1 |lo1|10.0.0.1|255.255.255.255|lo
Leaf1 |Eth1|10.2.1.1|255.255.255.254|p2p leaf1-eth1 to Spine1-eth1
Leaf1 |Eth2|10.2.2.1|255.255.255.254|p2p leaf1-eth2 to Spine2-eth1
Leaf2 |lo1|10.0.0.2|255.255.255.255|lo
Leaf2 |Eth1|10.2.1.3|255.255.255.254|p2p leaf2-eth1 to Spine1-eth2
Leaf2 |Eth2|10.2.2.3|255.255.255.254|p2p leaf2-eth2 to Spine2-eth2
Leaf3 |lo1|10.0.0.3|255.255.255.255|lo
Leaf3 |Eth1|10.2.1.5|255.255.255.254|p2p leaf3-eth1 to Spine1-eth3
Leaf3 |Eth2|10.2.2.5|255.255.255.254|p2p leaf3-eth2 to Spine2-eth3
VPC1  |Eth0|192.168.10.10|255.255.255.0|VPC1_vlan10-eth0 to Leaf1-eth3
VPC2  |Eth0|192.168.20.10|255.255.255.0|VPC2_vlan20-eth0 to Leaf1-eth4
VPC3  |Eth0|192.168.10.11|255.255.255.0|VPC3_vlan10-eth0 to Leaf2-eth3
VPC4  |Eth0|192.168.20.11|255.255.255.0|VPC4_vlan20-eth0 to Leaf2-eth4
VPC5  |Eth0|192.168.10.12|255.255.255.0|VPC5_vlan10-eth0 to Leaf3-eth3
VPC6  |Eth0|192.168.20.12|255.255.255.0|VPC6_vlan20-eth0 to Leaf3-eth4

## Выполнение:
### Подготовка оборудования:
- назначаем адреса интерфейсов коммутаторам, согласно адрес плана, переводим интерфейс в L3 режим.
- включаем маршрутизацию
- включаем протокол динамической маршрутизации BGP, назанчаем номер AS
- объявляем в BGP необходимые интерфейсы при помощи карты маршрутов и префикс листаов, анонсируем loopback интерфейс в bgp
- включаем глобальное использование BFD
- Включаем и настраиваем BFD на интерфейсах.
- создаем интерфейс VxVlan 1
- назначаем адреса всем VPC, сошласно адрес-плана
- Объявляем в BGP созданные vlan, указываем route-distinguisher, route-target 

<details><summary> Конфигурация leaf1 </summary>

~~~
hostname Leaf1
!
spanning-tree mode mstp
!
vlan 10,20
!
interface Ethernet1
   description p2p leaf1-eth1 to Spine1-eth1
   no switchport
   ip address 10.2.1.1/31
   bfd interval 100 min-rx 100 multiplier 3
!
interface Ethernet2
   description p2p leaf1-eth2 to Spine2-eth1
   no switchport
   ip address 10.2.2.1/31
   bfd interval 100 min-rx 100 multiplier 3
!
interface Ethernet3
   description to-pc1
   mtu 9000
   switchport access vlan 10
!
interface Ethernet4
   description to-pc2
   mtu 9000
   switchport access vlan 20
!
interface Ethernet5
!
interface Ethernet6
!
interface Ethernet7
!
interface Ethernet8
!
interface Loopback1
   ip address 10.0.0.1/32
!
interface Management1
!
interface Vlan10
!
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan vlan 20 vni 10020
!
ip routing
!
ip prefix-list PL_POOL
   seq 10 permit 10.0.0.1/32
!
route-map RM_REDIS_CON permit 10
   match ip address prefix-list PL_POOL
!
router bgp 65001
   router-id 10.0.0.1
   maximum-paths 10
   neighbor EVPN peer group
   neighbor EVPN remote-as 65000
   neighbor EVPN update-source Loopback1
   neighbor EVPN ebgp-multihop 5
   neighbor EVPN send-community extended
   neighbor SPINE peer group
   neighbor SPINE remote-as 65000
   neighbor SPINE bfd
   neighbor SPINE timers 3 9
   neighbor 10.0.1.1 peer group EVPN
   neighbor 10.0.2.1 peer group EVPN
   neighbor 10.2.1.0 peer group SPINE
   neighbor 10.2.2.0 peer group SPINE
   redistribute connected route-map RM_REDIS_CON
   !
   vlan 10
      rd 10.0.0.1:10010
      route-target both 10:10010
      redistribute learned
   !
   vlan 20
      rd 10.0.0.1:10020
      route-target both 10:10020
      redistribute learned
   !
   address-family evpn
      neighbor EVPN activate
   !
   address-family ipv4
      neighbor SPINE activate
!
end


~~~
</details>
<details><summary> Конфигурация leaf2 </summary>

~~~
hostname Leaf2
!
spanning-tree mode mstp
!
vlan 10,20
!
interface Ethernet1
   description p2p Leaf2-eth1 to Spine1-eth2
   no switchport
   ip address 10.2.1.3/31
   bfd interval 100 min-rx 100 multiplier 3
!
interface Ethernet2
   description p2p leaf2-eth2 to spine2-eth2
   no switchport
   ip address 10.2.2.3/31
   bfd interval 100 min-rx 100 multiplier 3
!
interface Ethernet3
   description to-pc3
   mtu 9000
   switchport access vlan 10
!
interface Ethernet4
   description to-pc4
   mtu 9000
   switchport access vlan 20
!
interface Ethernet5
!
interface Ethernet6
!
interface Ethernet7
!
interface Ethernet8
!
interface Loopback1
   ip address 10.0.0.2/32
!
interface Management1
!
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan vlan 20 vni 10020
!
ip routing
!
ip prefix-list PL_POOL
   seq 10 permit 10.0.0.2/32
!
route-map RM_REDIS_CON permit 10
   match ip address prefix-list PL_POOL
!
router bgp 65002
   router-id 10.0.0.2
   maximum-paths 10
   neighbor EVPN peer group
   neighbor EVPN remote-as 65000
   neighbor EVPN update-source Loopback1
   neighbor EVPN ebgp-multihop 5
   neighbor EVPN send-community extended
   neighbor SPINE peer group
   neighbor SPINE remote-as 65000
   neighbor SPINE bfd
   neighbor SPINE timers 3 9
   neighbor 10.0.1.1 peer group EVPN
   neighbor 10.0.2.1 peer group EVPN
   neighbor 10.2.1.2 peer group SPINE
   neighbor 10.2.2.2 peer group SPINE
   redistribute connected route-map RM_REDIS_CON
   !
   vlan 10
      rd 10.0.0.2:10010
      route-target both 10:10010
      redistribute learned
   !
   vlan 20
      rd 10.0.0.2:10020
      route-target both 10:10020
      redistribute learned
   !
   address-family evpn
      neighbor EVPN activate
   !
   address-family ipv4
      neighbor SPINE activate
!
end


~~~
</details>
<details><summary> Конфигурация leaf3 </summary>

~~~
hostname Leaf3
!
spanning-tree mode mstp
!
vlan 10,20
!
interface Ethernet1
   description p2p leaf3-eth1 to spine1-eth3
   no switchport
   ip address 10.2.1.5/31
   bfd interval 100 min-rx 100 multiplier 3
!
interface Ethernet2
   description p2p leaf3-eth2 to spine2-eth3
   no switchport
   ip address 10.2.2.5/31
   bfd interval 100 min-rx 100 multiplier 3
!
interface Ethernet3
   description to-pc5
   mtu 9000
   switchport access vlan 10
!
interface Ethernet4
   description to-pc6
   mtu 9000
   switchport access vlan 20
!
interface Ethernet5
!
interface Ethernet6
!
interface Ethernet7
!
interface Ethernet8
!
interface Loopback1
   ip address 10.0.0.3/32
!
interface Management1
!
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan vlan 20 vni 10020
!
ip routing
!
ip prefix-list PL_POOL
   seq 10 permit 10.0.0.3/32
!
route-map RM_REDIS_CON permit 10
   match ip address prefix-list PL_POOL
!
router bgp 65003
   router-id 10.0.0.3
   maximum-paths 10
   neighbor EVPN peer group
   neighbor EVPN remote-as 65000
   neighbor EVPN update-source Loopback1
   neighbor EVPN ebgp-multihop 5
   neighbor EVPN send-community extended
   neighbor SPINE peer group
   neighbor SPINE remote-as 65000
   neighbor SPINE bfd
   neighbor SPINE timers 3 9
   neighbor 10.0.1.1 peer group EVPN
   neighbor 10.0.2.1 peer group EVPN
   neighbor 10.2.1.4 peer group SPINE
   neighbor 10.2.2.4 peer group SPINE
   redistribute connected route-map RM_REDIS_CON
   !
   vlan 10
      rd 10.0.0.3:10010
      route-target both 10:10010
      redistribute learned
   !
   vlan 20
      rd 10.0.0.3:10020
      route-target both 10:10020
      redistribute learned
   !
   address-family evpn
      neighbor EVPN activate
   !
   address-family ipv4
      neighbor SPINE activate
!
end


~~~
</details>
<details><summary> Конфигурация spine1 </summary>

~~~
hostname Spine1
!
spanning-tree mode mstp
!
interface Ethernet1
   description p2p Spine1_eth1 to Leaf1_eth1
   no switchport
   ip address 10.2.1.0/31
   bfd interval 100 min-rx 100 multiplier 3
   no ip ospf neighbor bfd
!
interface Ethernet2
   description p2p Spine1-eth2 to Leaf2-eth1
   no switchport
   ip address 10.2.1.2/31
   bfd interval 100 min-rx 100 multiplier 3
   no ip ospf neighbor bfd
!
interface Ethernet3
   description p2p Spine1-eth3 to Leaf3-eth1
   no switchport
   ip address 10.2.1.4/31
   bfd interval 100 min-rx 100 multiplier 3
   no ip ospf neighbor bfd
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
interface Loopback1
   ip address 10.0.1.1/32
!
interface Management1
!
ip routing
!
route-map RM_REDIS_CON permit 10
   match interface Loopback1
!
peer-filter EVPN
   10 match as-range 65001-65003 result accept
!
peer-filter LEAF-AS
   10 match as-range 65001-65003 result accept
!
router bgp 65000
   router-id 10.0.1.1
   maximum-paths 10
   bgp listen range 10.0.0.0/24 peer-group EVPN peer-filter EVPN
   bgp listen range 10.2.1.0/24 peer-group LEAF peer-filter LEAF-AS
   neighbor EVPN peer group
   neighbor EVPN next-hop-unchanged
   neighbor EVPN update-source Loopback1
   neighbor EVPN ebgp-multihop 3
   neighbor EVPN send-community extended
   neighbor LEAF peer group
   redistribute connected route-map RM_REDIS_CON
   !
   address-family evpn
      neighbor EVPN activate
   !
   address-family ipv4
      neighbor LEAF activate
!
end


~~~
</details>
<details><summary> Конфигурация spine2 </summary>

~~~
hostname Spine2
!
spanning-tree mode mstp
!
interface Ethernet1
   description p2p Spine2-eth1 to Leaf1-eth2
   no switchport
   ip address 10.2.2.0/31
   bfd interval 100 min-rx 100 multiplier 3
!
interface Ethernet2
   description p2p Spine2-eth2 to Leaf2-eth2
   no switchport
   ip address 10.2.2.2/31
   bfd interval 100 min-rx 100 multiplier 3
!
interface Ethernet3
   description p2p Spine2-eth3 to Leaf3-eth2
   no switchport
   ip address 10.2.2.4/31
   bfd interval 100 min-rx 100 multiplier 3
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
interface Loopback1
   ip address 10.0.2.1/32
!
interface Management1
!
ip routing
!
route-map RM_REDIS_CON permit 10
   match interface Loopback1
!
peer-filter EVPN
   10 match as-range 65001-65003 result accept
!
peer-filter LEAF-AS
   10 match as-range 65001-65003 result accept
!
router bgp 65000
   router-id 10.0.2.1
   maximum-paths 10
   bgp listen range 10.0.0.0/24 peer-group EVPN peer-filter EVPN
   bgp listen range 10.2.2.0/24 peer-group LEAF peer-filter LEAF-AS
   neighbor EVPN peer group
   neighbor EVPN next-hop-unchanged
   neighbor EVPN update-source Loopback1
   neighbor EVPN ebgp-multihop 3
   neighbor EVPN send-community extended
   neighbor LEAF peer group
   redistribute connected route-map RM_REDIS_CON
   !
   address-family evpn
      neighbor EVPN activate
   !
   address-family ipv4
      neighbor LEAF activate
!
end


~~~
</details>

### Проверка связности устройств:
Spine1
~~~
Spine1#sh bgp evpn summary
BGP summary information for VRF default
Router identifier 10.0.1.1, local AS number 65000
Neighbor Status Codes: m - Under maintenance
  Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  10.0.0.1 4 65001         197282    375734    0    0    2d18h Estab   2      2
  10.0.0.2 4 65002          34036     66073    0    0 12:43:47 Estab   2      2
  10.0.0.3 4 65003          24618     55194    0    0 08:41:30 Estab   2      2

~~~
Spine2
~~~
Spine2#sh bgp evpn summary 
BGP summary information for VRF default
Router identifier 10.0.2.1, local AS number 65000
Neighbor Status Codes: m - Under maintenance
  Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  10.0.0.1 4 65001           9589     18362    0    0 04:07:07 Estab   2      2
  10.0.0.2 4 65002          46046     90984    0    0 16:53:58 Estab   2      2
  10.0.0.3 4 65003           9236     21089    0    0 03:43:54 Estab   2      2

~~~
Leaf1
~~~
Leaf1#
Leaf1#show vxlan address-table
          Vxlan Mac Address Table
----------------------------------------------------------------------

VLAN  Mac Address     Type      Prt  VTEP             Moves   Last Move
----  -----------     ----      ---  ----             -----   ---------
  10  0050.7966.680a  EVPN      Vx1  10.0.0.3         1       0:00:44 ago
  20  0050.7966.680b  EVPN      Vx1  10.0.0.3         1       0:00:05 ago
Total Remote Mac Addresses for this criterion: 2
Leaf1#show bgp evpn route-type mac-ip
BGP routing table information for VRF default
Router identifier 10.0.0.1, local AS number 65001
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 10.0.0.1:10010 mac-ip 0050.7966.6806
                                 -                     -       -       0       i
 * >      RD: 10.0.0.1:10020 mac-ip 0050.7966.6807
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.0.3:10010 mac-ip 0050.7966.680a
                                 10.0.0.3              -       100     0       65000 65003 i
 *  ec    RD: 10.0.0.3:10010 mac-ip 0050.7966.680a
                                 10.0.0.3              -       100     0       65000 65003 i
 * >Ec    RD: 10.0.0.3:10020 mac-ip 0050.7966.680b
                                 10.0.0.3              -       100     0       65000 65003 i
 *  ec    RD: 10.0.0.3:10020 mac-ip 0050.7966.680b
                                 10.0.0.3              -       100     0       65000 65003 i

~~~
Leaf2
~~~
Leaf2#show vxlan address-table
          Vxlan Mac Address Table
----------------------------------------------------------------------

VLAN  Mac Address     Type      Prt  VTEP             Moves   Last Move
----  -----------     ----      ---  ----             -----   ---------
  10  0050.7966.6806  EVPN      Vx1  10.0.0.1         1       0:01:34 ago
  10  0050.7966.680a  EVPN      Vx1  10.0.0.3         1       0:01:33 ago
  20  0050.7966.6807  EVPN      Vx1  10.0.0.1         1       0:00:54 ago
  20  0050.7966.680b  EVPN      Vx1  10.0.0.3         1       0:00:54 ago
Total Remote Mac Addresses for this criterion: 4
Leaf2#show bgp evpn route-type mac-ip
BGP routing table information for VRF default
Router identifier 10.0.0.2, local AS number 65002
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 10.0.0.1:10010 mac-ip 0050.7966.6806
                                 10.0.0.1              -       100     0       65000 65001 i
 *  ec    RD: 10.0.0.1:10010 mac-ip 0050.7966.6806
                                 10.0.0.1              -       100     0       65000 65001 i
 * >Ec    RD: 10.0.0.1:10020 mac-ip 0050.7966.6807
                                 10.0.0.1              -       100     0       65000 65001 i
 *  ec    RD: 10.0.0.1:10020 mac-ip 0050.7966.6807
                                 10.0.0.1              -       100     0       65000 65001 i
 * >Ec    RD: 10.0.0.3:10010 mac-ip 0050.7966.680a
                                 10.0.0.3              -       100     0       65000 65003 i
 *  ec    RD: 10.0.0.3:10010 mac-ip 0050.7966.680a
                                 10.0.0.3              -       100     0       65000 65003 i
 * >Ec    RD: 10.0.0.3:10020 mac-ip 0050.7966.680b
                                 10.0.0.3              -       100     0       65000 65003 i
 *  ec    RD: 10.0.0.3:10020 mac-ip 0050.7966.680b
                                 10.0.0.3              -       100     0       65000 65003 i

~~~
Leaf3
~~~
Leaf3#show vxlan address-table
          Vxlan Mac Address Table
----------------------------------------------------------------------

VLAN  Mac Address     Type      Prt  VTEP             Moves   Last Move
----  -----------     ----      ---  ----             -----   ---------
  10  0050.7966.6806  EVPN      Vx1  10.0.0.1         1       0:02:10 ago
  20  0050.7966.6807  EVPN      Vx1  10.0.0.1         1       0:01:31 ago
Total Remote Mac Addresses for this criterion: 2
Leaf3#show bgp evpn route-type mac-ip
BGP routing table information for VRF default
Router identifier 10.0.0.3, local AS number 65003
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 10.0.0.1:10010 mac-ip 0050.7966.6806
                                 10.0.0.1              -       100     0       65000 65001 i
 *  ec    RD: 10.0.0.1:10010 mac-ip 0050.7966.6806
                                 10.0.0.1              -       100     0       65000 65001 i
 * >Ec    RD: 10.0.0.1:10020 mac-ip 0050.7966.6807
                                 10.0.0.1              -       100     0       65000 65001 i
 *  ec    RD: 10.0.0.1:10020 mac-ip 0050.7966.6807
                                 10.0.0.1              -       100     0       65000 65001 i
 * >      RD: 10.0.0.3:10010 mac-ip 0050.7966.680a
                                 -                     -       -       0       i
 * >      RD: 10.0.0.3:10020 mac-ip 0050.7966.680b
                                 -                     -       -       0       i

~~~