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

S1:

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

настраиваем VTEP интерфейс на S1
    
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

### на S2 настраиваем все аналогично, но влан берем 20, и также третий октет в клиентской сети будет также 20

на спайнах включаем маршрутизацию

    ip routing
    ip routing vrf VXLAN

##проверяем, что получилось


