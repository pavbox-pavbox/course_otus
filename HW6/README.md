# Домашнее задание №6

![image](https://github.com/pavbox-pavbox/course_otus/assets/97456111/60bc5b6a-83af-479c-be4d-f0c5d2280a6b)

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
          rd auto
          route-target both 10:10
          redistribute learned
       !
       address-family evpn
          neighbor SPINES activate
       !
       address-family ipv4
          no neighbor SPINES activate
       !
       vrf VXLAN
          rd 65001:10000
          route-target import evpn 65001:10000
          route-target export evpn 65001:10000
          router-id 10.100.0.1
          redistribute connected

настраиваем VTEP интерфейс на L1
    
    interface Vxlan1
       vxlan source-interface Loopback1
       vxlan udp-port 4789
       vxlan vlan 10 vni 10010
       vxlan vrf VXLAN vni 10000
       vxlan learn-restrict any

также настраиваем anycast интерфейс в vlan10

    interface Vlan10
       vrf VXLAN
       ip address virtual 192.168.10.1/24

### на L2 настраиваем все аналогично, но влан берем 20, и также третий октет в клиентской сети будет также 20

таким образом:

  на L1 vlan10 будет маппиться в vni 10010
  
  на L2 vlan20 будет маппиться в vni 10020
  
  и на обоих спайнах vrf VXLAN соответствует vni 10000

на спайнах включаем маршрутизацию

    ip routing
    ip routing vrf VXLAN

##проверяем, что получилось

пингаем с host1 адрес host2 (он в другой сети)

        VPCS> sh ip
        
        NAME        : VPCS[1]
        IP/MASK     : 192.168.10.10/24
        GATEWAY     : 192.168.10.1
        DNS         : 
        MAC         : 00:50:79:66:68:06
        LPORT       : 20000
        RHOST:PORT  : 127.0.0.1:30000
        MTU         : 1500
        
        VPCS> ping 192.168.20.10
        
        84 bytes from 192.168.20.10 icmp_seq=1 ttl=62 time=18.761 ms
        84 bytes from 192.168.20.10 icmp_seq=2 ttl=62 time=17.735 ms
        84 bytes from 192.168.20.10 icmp_seq=3 ttl=62 time=6.907 ms
        84 bytes from 192.168.20.10 icmp_seq=4 ttl=62 time=9.342 ms
        84 bytes from 192.168.20.10 icmp_seq=5 ttl=62 time=17.359 ms

и наблюдаем, что на лифе появился маршрут до удаленного хоста

    L1(config-router-bgp-vrf-VXLAN)#sh ip route vrf VXLAN

    VRF: VXLAN
    
    Gateway of last resort is not set
    
     C        192.168.10.0/24 is directly connected, Vlan10
     B E      192.168.20.10/32 [200/0] via VTEP 10.100.0.2 VNI 10000 router-mac 50:00:00:03:37:66 local-interface Vxlan1
     B E      192.168.20.0/24 [200/0] via VTEP 10.100.0.2 VNI 10000 router-mac 50:00:00:03:37:66 local-interface Vxlan1

вывод bgp evpn

    L1(config-router-bgp-vrf-VXLAN)#sh bgp evpn
    BGP routing table information for VRF default
    Router identifier 10.100.0.1, local AS number 65001
    Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                        c - Contributing to ECMP, % - Pending BGP convergence
    Origin codes: i - IGP, e - EGP, ? - incomplete
    AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop
    
              Network                Next Hop              Metric  LocPref Weight  Path
     * >      RD: 10.100.0.1:10 mac-ip 0050.7966.6806
                                     -                     -       -       0       i
     * >      RD: 10.100.0.1:10 mac-ip 0050.7966.6806 192.168.10.10
                                     -                     -       -       0       i
     * >Ec    RD: 10.100.0.2:20 mac-ip 0050.7966.6807
                                     10.100.0.2            -       100     0       65901 65001 i
     *  ec    RD: 10.100.0.2:20 mac-ip 0050.7966.6807
                                     10.100.0.2            -       100     0       65901 65001 i
     * >Ec    RD: 10.100.0.2:20 mac-ip 0050.7966.6807 192.168.20.10
                                     10.100.0.2            -       100     0       65901 65001 i
     *  ec    RD: 10.100.0.2:20 mac-ip 0050.7966.6807 192.168.20.10
                                     10.100.0.2            -       100     0       65901 65001 i
     * >      RD: 10.100.0.1:10 imet 10.100.0.1
                                     -                     -       -       0       i
     * >Ec    RD: 10.100.0.2:20 imet 10.100.0.2
                                     10.100.0.2            -       100     0       65901 65001 i
     *  ec    RD: 10.100.0.2:20 imet 10.100.0.2
                                     10.100.0.2            -       100     0       65901 65001 i
     * >Ec    RD: 10010:10010 imet 10.100.0.3
                                     10.100.0.3            -       100     0       65901 65001 i
     *  ec    RD: 10010:10010 imet 10.100.0.3
                                     10.100.0.3            -       100     0       65901 65001 i

# Задача решена

