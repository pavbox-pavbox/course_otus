# Домашнее задание №4

![image](https://github.com/pavbox-pavbox/course_otus/assets/97456111/c34ba967-5549-4b73-838a-d6ee435de5cd)

### Продолжаем ставить эксперименты с сетью Клоза

## Настройка BGP

# К сожалению, образ CE12800 в eve-ng не позволяет настроить iBGP, поэтому будем делать EBGP

План распределения IP адресов

Стыковочные сети

|Node|Интерфейс|Адрес|-|Адрес|Интерфейс|Node|
|-|-|-|-|-|-|-|
|S1|GE1/0/1|10.0.0.0|-|10.0.0.1|GE1/0/1|L1|
|S1|GE1/0/2|10.2.0.0|-|10.2.0.1|GE1/0/1|L2|
|S1|GE1/0/3|10.4.0.0|-|10.4.0.1|GE1/0/1|L3|

|Node|Интерфейс|Адрес|-|Адрес|Интерфейс|Node|
|-|-|-|-|-|-|-|
|S2|GE1/0/1|10.0.1.0|-|10.0.1.1|GE1/0/2|L1|
|S2|GE1/0/2|10.2.1.0|-|10.2.1.1|GE1/0/2|L2|
|S2|GE1/0/3|10.4.1.0|-|10.4.1.1|GE1/0/2|L3|

loopback интерфейсы на всех коммутатора уровня Leaf

|node|Интерфейс|Адрес|
|-|-|-|
|L1|Lo1|10.100.0.1|
|L2|Lo1|10.100.0.2|
|L3|Lo1|10.100.0.3|

Настройка BGP

Для начала распределим AS-NUMBER

|node|AS-NUMBER|
|-|-|
|L1|65001|
|L2|65002|
|L3|65003|
|S1|65101|
|S2|65102|

Настраиваем спайн S1. На S2 настройки похожие

    bgp 65101
      router-id 10.200.0.1
      timer keepalive 3 hold 9
      peer 10.0.0.1 as-number 65001
      peer 10.2.0.1 as-number 65002
      peer 10.4.0.1 as-number 65003
      ipv4-family unicast
        import-route direct
        peer 10.0.0.1 enable
        peer 10.2.0.1 enable
        peer 10.4.0.1 enable

Настраиваем лиф L1. Остальные лифы аналогичным образом

    bgp 65001
     router-id 10.100.0.1
     timer keepalive 3 hold 9
     peer 10.0.0.0 as-number 65101
     peer 10.0.1.0 as-number 65102
     ipv4-family unicast
       network 10.100.0.1 255.255.255.255
       maximum load-balancing ebgp 2 ecmp-nexthop-changed  
       load-balancing as-path-ignore
       peer 10.0.0.0 enable
       peer 10.0.1.0 enable

Таймеры сделаны покороче, настроен load-balancing. Проверяем на L1:

    <L1>dis bgp all summary 
     
     BGP local router ID : 10.100.0.1
     Local AS number : 65001
    
     Address Family:Ipv4 Unicast
     --------------------------------------------------------------------------------------------
     Total number of peers : 2                 Peers in established state : 2
    
      Peer                     AS  MsgRcvd  MsgSent  OutQ  Up/Down       State    RtRcv    RtAdv
      10.0.0.0              65101     1610     1592     0 00:53:38 Established        7       10
      10.0.1.0              65102     1164     1170     0 00:38:58 Established        6       10

BGP в состоянии established (обе). Проверяем маршруты до лупбека L2

    <L1>dis ip routing-table 10.100.0.2
    Proto: Protocol        Pre: Preference
    Route Flags: R - relay, D - download to fib, T - to vpn-instance, B - black hole route
    ------------------------------------------------------------------------------
    Routing Table : _public_
    Summary Count : 2
    
    Destination/Mask    Proto   Pre  Cost        Flags NextHop         Interface
    
         10.100.0.2/32  EBGP    255  0             RD  10.0.0.0        GE1/0/1
                        EBGP    255  0             RD  10.0.1.0        GE1/0/2

Ну и проверяем load-balancing

    <L1>tracert 10.100.0.2
     traceroute to 10.100.0.2(10.100.0.2), max hops: 30, packet length: 40, press CTRL_C to break
     1 10.0.0.0 7 ms  4 ms  4 ms 
     2 10.100.0.2 9 ms  2 ms  3 ms 
    <L1>tracert 10.100.0.2
     traceroute to 10.100.0.2(10.100.0.2), max hops: 30, packet length: 40, press CTRL_C to break
     1 10.0.1.0 3 ms  2 ms  4 ms 
     2 10.100.0.2 13 ms  1 ms  6 ms 

Каждый новый трейс идет своим маршрутом.

# Задача решена

