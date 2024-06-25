# Домашнее задание №5

![image](https://github.com/pavbox-pavbox/course_otus/assets/97456111/e45441b8-704f-4938-a49d-f7e8404b4191)

Из-за неспособности настроить лабу на хуавеях пришлось переехать на Аристы

План распределения IP адресов

Стыковочные сети

|Node|Интерфейс|Адрес|-|Адрес|Интерфейс|Node|
|-|-|-|-|-|-|-|
|S1|Ethernet1|10.0.0.0|-|10.0.0.1|Ethernet1|L1|
|S1|Ethernet2|10.2.0.0|-|10.2.0.1|Ethernet1|L2|
|S1|Ethernet3|10.4.0.0|-|10.4.0.1|Ethernet1|L3|

|Node|Интерфейс|Адрес|-|Адрес|Интерфейс|Node|
|-|-|-|-|-|-|-|
|S2|Ethernet1|10.0.1.0|-|10.0.1.1|Ethernet2|L1|
|S2|Ethernet2|10.2.1.0|-|10.2.1.1|Ethernet2|L2|
|S2|Ethernet3|10.4.1.0|-|10.4.1.1|Ethernet2|L3|

loopback интерфейсы на всех коммутатора уровня Leaf

|node|Интерфейс|Адрес|
|-|-|-|
|L1|Lo1|10.100.0.1|
|L2|Lo1|10.100.0.2|
|L3|Lo1|10.100.0.3|

## Underlay настроен на протоколе OSPF

Пример настройки L1:

    router ospf 1
       router-id 10.100.0.1
       network 10.0.0.0/16 area 0.0.0.0
       network 10.100.0.1/32 area 0.0.0.0
       max-lsa 12000
       maximum-paths 2
    !

Пример настройки S1:

    router ospf 1
       router-id 10.200.0.1
       redistribute connected
       network 10.0.0.0/16 area 0.0.0.0
       network 10.2.0.0/16 area 0.0.0.0
       network 10.4.0.0/16 area 0.0.0.0
       network 10.200.0.1/32 area 0.0.0.0
       max-lsa 12000
       maximum-paths 2

## Далее настраиваем overlay на BGP:

L1:

    router bgp 65001
       router-id 10.100.0.1
       timers bgp 3 9
       maximum-paths 10
       neighbor SPINES peer group
       neighbor SPINES update-source Loopback1
       neighbor SPINES bfd
       neighbor SPINES allowas-in 2
       neighbor SPINES ebgp-multihop 10
       neighbor SPINES send-community extended
       neighbor 10.200.0.1 peer group SPINES
       neighbor 10.200.0.1 remote-as 65901
       neighbor 10.200.0.2 peer group SPINES
       neighbor 10.200.0.2 remote-as 65901
       !
       vlan 10
          rd 10010:10010
          route-target both 10010:10010
          redistribute learned
       !
       address-family evpn
          neighbor SPINES activate
       !
       address-family ipv4
          no neighbor SPINES activate

S1:

    router bgp 65901
       router-id 10.200.0.1
       timers bgp 3 9
       maximum-paths 8 ecmp 16
       neighbor LEAFS peer group
       neighbor LEAFS remote-as 65001
       neighbor LEAFS next-hop-unchanged
       neighbor LEAFS update-source Loopback1
       neighbor LEAFS bfd
       neighbor LEAFS allowas-in 2
       neighbor LEAFS ebgp-multihop 10
       neighbor LEAFS send-community extended
       neighbor 10.100.0.1 peer group LEAFS
       neighbor 10.100.0.2 peer group LEAFS
       neighbor 10.100.0.3 peer group LEAFS
       !
       address-family evpn
          neighbor LEAFS activate
       !
       address-family ipv4
          neighbor LEAFS activate
    !

Как видно в настройках лифа, создан условный влан 10, который маппится в VNI 10010. Настроено это в интерфейсе Vxlan1 (L1, L2, L3))

    interface Vxlan1
       vxlan source-interface Loopback1
       vxlan udp-port 4789
       vxlan vlan 10 vni 10010
       vxlan vlan 20 vni 10020
    !

Также этот влан висит на порту Ethernet8 каждого из лифов, и за этим портом включены хосты с адресами 192.168.0.1 и 192.168.0.2

пробуем пингать с первого хоста второй

    VPC1> ping 192.168.0.2
    
    84 bytes from 192.168.0.2 icmp_seq=1 ttl=64 time=5.746 ms
    84 bytes from 192.168.0.2 icmp_seq=2 ttl=64 time=16.181 ms

И наблюдаем, что на лифе появились два маршута типа mac-ip равной стоимости через оба спайна

    L1# sh bgp evpn route-type mac-ip detail 
    BGP routing table information for VRF default
    Router identifier 10.100.0.1, local AS number 65001
    BGP routing table entry for mac-ip 0050.7966.6806, Route Distinguisher: 10010:10010
     Paths: 1 available
      Local
        - from - (0.0.0.0)
          Origin IGP, metric -, localpref -, weight 0, tag 0, valid, local, best
          Extended Community: Route-Target-AS:10010:10010 TunnelEncap:tunnelTypeVxlan
          VNI: 10010 ESI: 0000:0000:0000:0000:0000
    BGP routing table entry for mac-ip 0050.7966.6807, Route Distinguisher: 10010:10010
     Paths: 2 available
      65901 65001
        10.100.0.2 from 10.200.0.2 (10.200.0.2)
          Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
          Extended Community: Route-Target-AS:10010:10010 TunnelEncap:tunnelTypeVxlan
          VNI: 10010 ESI: 0000:0000:0000:0000:0000
      65901 65001
        10.100.0.2 from 10.200.0.1 (10.200.0.1)
          Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
          Extended Community: Route-Target-AS:10010:10010 TunnelEncap:tunnelTypeVxlan
          VNI: 10010 ESI: 0000:0000:0000:0000:0000
    L1#

# задача решена

Сохраненные конфиги на всякий случай

<details>
  <summary>L1</summary>

```
L1#sh run
! Command: show running-config
! device: L1 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model multi-agent
!
hostname L1
!
spanning-tree mode mstp
!
vlan 10
!
interface Ethernet1
   no switchport
   ip address 10.0.0.1/31
!
interface Ethernet2
   no switchport
   ip address 10.0.1.1/31
!
interface Ethernet3
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
   switchport access vlan 10
!
interface Loopback1
   ip address 10.100.0.1/32
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
router bgp 65001
   router-id 10.100.0.1
   timers bgp 3 9
   maximum-paths 10
   neighbor SPINES peer group
   neighbor SPINES update-source Loopback1
   neighbor SPINES bfd
   neighbor SPINES allowas-in 2
   neighbor SPINES ebgp-multihop 10
   neighbor SPINES send-community extended
   neighbor 10.200.0.1 peer group SPINES
   neighbor 10.200.0.1 remote-as 65901
   neighbor 10.200.0.2 peer group SPINES
   neighbor 10.200.0.2 remote-as 65901
   !
   vlan 10
      rd 10010:10010
      route-target both 10010:10010
      redistribute learned
   !
   address-family evpn
      neighbor SPINES activate
   !
   address-family ipv4
      no neighbor SPINES activate
!
router ospf 1
   router-id 10.100.0.1
   network 10.0.0.0/16 area 0.0.0.0
   network 10.100.0.1/32 area 0.0.0.0
   max-lsa 12000
   maximum-paths 2
!
end

```

</details>

<details>
  <summary>L2</summary>

```
L2#sh run
! Command: show running-config
! device: L2 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model multi-agent
!
hostname L2
!
spanning-tree mode mstp
!
vlan 10
!
interface Ethernet1
   no switchport
   ip address 10.2.0.1/31
!
interface Ethernet2
   no switchport
   ip address 10.2.1.1/31
!
interface Ethernet3
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
   switchport access vlan 10
!
interface Loopback1
   ip address 10.100.0.2/32
!
interface Loopback2
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
router bgp 65001
   router-id 10.100.0.2
   timers bgp 3 9
   maximum-paths 10
   neighbor SPINES peer group
   neighbor SPINES update-source Loopback1
   neighbor SPINES bfd
   neighbor SPINES allowas-in 2
   neighbor SPINES ebgp-multihop 10
   neighbor SPINES send-community extended
   neighbor 10.200.0.1 peer group SPINES
   neighbor 10.200.0.1 remote-as 65901
   neighbor 10.200.0.2 peer group SPINES
   neighbor 10.200.0.2 remote-as 65901
   !
   vlan 10
      rd 10010:10010
      route-target both 10010:10010
      redistribute learned
   !
   address-family evpn
      neighbor SPINES activate
   !
   address-family ipv4
      no neighbor SPINES activate
!
router ospf 1
   network 10.2.0.0/16 area 0.0.0.0
   network 10.100.0.2/32 area 0.0.0.0
   max-lsa 12000
   maximum-paths 2
!
end

```
</details>

<details>
  <summary>L3</summary>

```
L3#sh run
! Command: show running-config
! device: L3 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model multi-agent
!
hostname L3
!
spanning-tree mode mstp
!
vlan 10
!
interface Ethernet1
   no switchport
   ip address 10.4.0.1/31
!
interface Ethernet2
   no switchport
   ip address 10.4.1.1/31
!
interface Ethernet3
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
   switchport access vlan 10
!
interface Loopback1
   ip address 10.100.0.3/32
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
router bgp 65001
   router-id 10.100.0.3
   timers bgp 3 9
   maximum-paths 10
   neighbor SPINES peer group
   neighbor SPINES remote-as 65901
   neighbor SPINES update-source Loopback1
   neighbor SPINES bfd
   neighbor SPINES allowas-in 2
   neighbor SPINES ebgp-multihop 10
   neighbor SPINES send-community extended
   neighbor 10.200.0.1 peer group SPINES
   neighbor 10.200.0.2 peer group SPINES
   !
   vlan 10
      rd 10010:10010
      route-target both 10010:10010
      redistribute learned
   !
   address-family evpn
      neighbor SPINES activate
   !
   address-family ipv4
      no neighbor SPINES activate
!
router ospf 1
   router-id 10.100.0.3
   network 10.4.0.0/16 area 0.0.0.0
   network 10.100.0.3/32 area 0.0.0.0
   max-lsa 12000
   maximum-paths 2
!
end
```
</details>

<details>
  <summary>S1</summary>

```
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
service routing protocols model multi-agent
!
hostname S1
!
spanning-tree mode mstp
!
interface Ethernet1
   no switchport
   ip address 10.0.0.0/31
!
interface Ethernet2
   no switchport
   ip address 10.2.0.0/31
!
interface Ethernet3
   no switchport
   ip address 10.4.0.0/31
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
   ip address 10.200.0.1/32
!
interface Management1
!
ip routing
!
router bgp 65901
   router-id 10.200.0.1
   timers bgp 3 9
   maximum-paths 8 ecmp 16
   neighbor LEAFS peer group
   neighbor LEAFS remote-as 65001
   neighbor LEAFS next-hop-unchanged
   neighbor LEAFS update-source Loopback1
   neighbor LEAFS bfd
   neighbor LEAFS allowas-in 2
   neighbor LEAFS ebgp-multihop 10
   neighbor LEAFS send-community extended
   neighbor 10.100.0.1 peer group LEAFS
   neighbor 10.100.0.2 peer group LEAFS
   neighbor 10.100.0.3 peer group LEAFS
   !
   address-family evpn
      neighbor LEAFS activate
   !
   address-family ipv4
      neighbor LEAFS activate
!
router ospf 1
   router-id 10.200.0.1
   redistribute connected
   network 10.0.0.0/16 area 0.0.0.0
   network 10.2.0.0/16 area 0.0.0.0
   network 10.4.0.0/16 area 0.0.0.0
   network 10.200.0.1/32 area 0.0.0.0
   max-lsa 12000
   maximum-paths 2
!
end
````
</details>

<details>
  <summary>S2</summary>

```
S2#sh run
! Command: show running-config
! device: S2 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model multi-agent
!
hostname S2
!
spanning-tree mode mstp
!
interface Ethernet1
   no switchport
   ip address 10.0.1.0/31
!
interface Ethernet2
   no switchport
   ip address 10.2.1.0/31
!
interface Ethernet3
   no switchport
   ip address 10.4.1.0/31
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
   ip address 10.200.0.2/32
!
interface Management1
!
ip routing
!
router bgp 65901
   router-id 10.200.0.2
   timers bgp 3 9
   maximum-paths 8 ecmp 16
   neighbor LEAFS peer group
   neighbor LEAFS remote-as 65001
   neighbor LEAFS next-hop-unchanged
   neighbor LEAFS bfd
   neighbor LEAFS allowas-in 2
   neighbor LEAFS ebgp-multihop 10
   neighbor LEAFS send-community extended
   neighbor 10.100.0.1 peer group LEAFS
   neighbor 10.100.0.2 peer group LEAFS
   neighbor 10.100.0.3 peer group LEAFS
   !
   address-family evpn
      neighbor LEAFS activate
   !
   address-family ipv4
      neighbor LEAFS activate
!
router ospf 1
   router-id 10.200.0.2
   network 10.0.0.0/16 area 0.0.0.0
   network 10.2.0.0/16 area 0.0.0.0
   network 10.4.0.0/16 area 0.0.0.0
   network 10.200.0.2/32 area 0.0.0.0
   max-lsa 12000
   maximum-paths 2
!
end
```
