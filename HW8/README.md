# Домашнее задание №8

![image](https://github.com/user-attachments/assets/c32b97e0-c3aa-4c14-a3e1-7ba212eb58f0)

### Изучение возможностей маршрутизации в нашей сети и оптимизация маршрутной информации

VXLAN фабрика не меняется, добавляем клиентов снизу и роутер сбоку

|клиент|адрес|маска|GW|vlan|
|-|-|-|-|-|
|Host_CL1N1|192.168.10.10|25|192.168.10.1|10|
|Host_CL1N2|192.168.10.210|25|192.168.10.129|110|
|Host_CL2N1|192.168.20.10|25|192.168.20.1|20|

Два клиента первого "кластера" подключены к L1. Создаем два виртуальных шлюза в каждом из клиентских вланов, и помещаем их в один vrf

    interface Vlan10
       vrf VRF_RED
       ip address virtual 192.168.10.1/25
    !
    interface Vlan110
       vrf VRF_RED
       ip address virtual 192.168.10.129/25
    !

Далее этот vrf ассоциируем с vni 10000 в интерфейсе vxlan1

    interface Vxlan1
       vxlan source-interface Loopback1
       vxlan udp-port 4789
       vxlan vrf VRF_RED vni 10000

Сети клиентов выбраны с маской /25 для дальнейшей демонстрации суммаризации маршрутов

На L2 выполняем похожие настройки, меняя клиентскую сеть, влан, vni, и название vrf на коммутаторе

    interface Vlan20
       vrf VRF_BLUE
       ip address virtual 192.168.20.1/24
    !
    interface Vxlan1
       vxlan source-interface Loopback1
       vxlan udp-port 4789
       vxlan vlan 20 vni 10020
       vxlan vrf VRF_BLUE vni 20000

Коммутатор L3 в нашей работе будет выполнять роль border-leaf. На нем настроены оба клиентских vrf. Маршрутизации между клиентами пока нет, так как vrf изолированы друг от друга. 

    interface Vxlan1
       vxlan source-interface Loopback1
       vxlan udp-port 4789
       vxlan vrf VRF_BLUE vni 20000
       vxlan vrf VRF_RED vni 10000
    !
    ip routing
    ip routing vrf VRF_BLUE
    ip routing vrf VRF_RED

По условиям задачи непосредственно маршрутизацию будет осуществлять отдельный роутер. Он расположен справа на схеме. С ним организован стык двумя интерфейсами. 

|устройство|интерфейс|адрес|-|адрес|интерфейс|устройство|
|-|-|-|-|-|-|-|
|L3|Eth3|10.250.0.0/31|-|10.250.0.1/31|Eth1|Router|
|L3|Eth4|10.250.1.0/31|-|10.250.1.1/31|Eth2|Router|

На стороне L3 каждый интерфейс помещен в свой VRF и организован BGP с внешним роутером

    interface Ethernet3
       no switchport
       vrf VRF_RED
       ip address 10.250.0.0/31
    !
    interface Ethernet4
       no switchport
       vrf VRF_BLUE
       ip address 10.250.1.0/31
    !

Настройки BGP на L3

        router bgp 65001
          vrf VRF_BLUE
              rd 65001:20000
              router-id 10.250.1.0
              neighbor 10.250.1.1 remote-as 65101
              neighbor 10.250.1.1 bfd
              neighbor 10.250.1.1 timers 3 9
              !
              address-family ipv4
                 neighbor 10.250.1.1 activate
           !
           vrf VRF_RED
              rd 65001:10000
              router-id 10.250.0.0
              neighbor 10.250.0.1 remote-as 65101
              neighbor 10.250.0.1 bfd
              neighbor 10.250.0.1 timers 3 9
              aggregate-address 192.168.10.0/24 summary-only
              !
              address-family ipv4
                 neighbor 10.250.0.1 activate
        !

И в настройках VRF_RED происходит непосредственно суммаризация маршрутов. Если vrf изучил маршруты, которые лежат внутри указанного 192.168.10.0/24, то данный VRF будет распространять только суммарный маршрут, а не маршруты /25 как мы придумали чуть ранее

Далее переходим к настройкам нашего роутера. В нем настроена маршрутизация в глобальной таблице маршрутизации, поэтому и обеспечивается связность клиентов из VRF_RED и VRF_BLUE.

        router bgp 65101
           neighbor 10.250.0.0 remote-as 65001
           neighbor 10.250.0.0 route-map AS-PATH in
           neighbor 10.250.1.0 remote-as 65001
           neighbor 10.250.1.0 route-map AS-PATH in
           redistribute bgp leaked
           !
           address-family ipv4
              neighbor 10.250.0.0 activate
              neighbor 10.250.1.0 activate
        !

Однако, импортируя маршруты, необходимо очистить AS-PATH, так как коммутатор L3, увидев маршруты из другого vrf со своим номером AS, будет их отбрасывать. Для этого мы создаем route-map

        route-map AS-PATH permit 10
           set as-path match all replacement none
        !
Здесь же можно настроить фильтры для избирательного импорта маршрутов и получить что-то типа фаерволла, но это уже потом. Пока же нужно обеспечить связность.

### Проверяем.

таблица маршрутизации на Router

        Router(config)#sh ip route
        
         C        10.250.0.0/31 is directly connected, Ethernet1
         C        10.250.1.0/31 is directly connected, Ethernet2
         B E      192.168.10.0/24 [200/0] via 10.250.0.0, Ethernet1
         B E      192.168.20.0/24 [200/0] via 10.250.1.0, Ethernet2

Видим что со стороны 10.250.0.0 импортировался суммаризованый маршрут с маской /24

в то же время на L3 присутствуют как маршруты с маской /25, так и суммарный маршрут /24

        L3(config-router-bgp-vrf-VRF_RED)#sh ip route vrf VRF_RED
        
        VRF: VRF_RED
        
         C        10.250.0.0/31 is directly connected, Ethernet3
         B E      192.168.10.0/25 [200/0] via VTEP 10.100.0.1 VNI 10000 router-mac 50:00:00:d5:5d:c0 local-interface Vxlan1
         B E      192.168.10.128/25 [200/0] via VTEP 10.100.0.1 VNI 10000 router-mac 50:00:00:d5:5d:c0 local-interface Vxlan1
         A B      192.168.10.0/24 is directly connected, Null0
         B E      192.168.20.0/24 [200/0] via 10.250.0.1, Ethernet3
        

Проверяем связность между клиентами в разных сетях

        VPCS> sh ip
        
        NAME        : VPCS[1]
        IP/MASK     : 192.168.10.210/25
        GATEWAY     : 192.168.10.129
        DNS         :
        MAC         : 00:50:79:66:68:0a
        LPORT       : 20000
        RHOST:PORT  : 127.0.0.1:30000
        MTU         : 1500
        
        VPCS> ping 192.168.20.10
        
        84 bytes from 192.168.20.10 icmp_seq=1 ttl=59 time=28.074 ms
        84 bytes from 192.168.20.10 icmp_seq=2 ttl=59 time=27.894 ms
        
Проверяем трейс

        VPCS> trace 192.168.20.10
        trace to 192.168.20.10, 8 hops max, press Ctrl+C to stop
         1   192.168.10.129   3.088 ms  1.991 ms  1.630 ms
         2   10.250.0.0   6.196 ms  5.372 ms  6.391 ms
         3   10.250.0.1   8.796 ms  8.998 ms  7.464 ms
         4   10.250.1.0   9.219 ms  8.578 ms  11.272 ms
         5   192.168.20.1   15.394 ms  14.380 ms  13.791 ms
         6   *192.168.20.10   15.480 ms (ICMP type:3, code:3, Destination port unreachable)

Хоть и пишет, что порт назначения недостижим, маршрут показывает верный, через наш внешний роутер

## задача решена

