# Домашнее задание №2
![image](https://github.com/pavbox-pavbox/course_otus/assets/97456111/5170ea9b-cf6f-4000-8e9d-d4950382919b)

### Ранее созданная сеть продолжает развиваться

Созданы loopback интерфейсы на всех коммутатора уровня Leaf

|node|Интерфейс|Адрес|
|-|-|-|
|L1|Lo1|10.100.0.1|
|L2|Lo1|10.100.0.2|
|L3|Lo1|10.100.0.3|

На коммутаторах адрес loopback внесен в сети ospf

    ospf 1 router-id 10.0.0.1
     area 0.0.0.0
      network 10.0.0.0 0.0.255.255
      network 10.100.0.1 0.0.0.0

Соседство лифа со спайнами установилось:

    [~HUAWEI]dis ospf peer brief
    OSPF Process 1 with Router ID 10.0.0.1
                       Peer Statistic Information
    Total number of peer(s): 2
     Peer(s) in full state: 2
    -----------------------------------------------------------------------------
     Area Id         Interface                  Neighbor id          State
     0.0.0.0         GE1/0/1                    10.0.0.0             Full
     0.0.0.0         GE1/0/2                    10.0.1.0             Full
    -----------------------------------------------------------------------------

Между коммутаторами L1 и L3 появилась связность двумя маршрутами равной стоимости

    [~HUAWEI]dis ip routing-table 10.100.0.3
    Proto: Protocol        Pre: Preference
    Route Flags: R - relay, D - download to fib, T - to vpn-instance, B - black hole route
    ------------------------------------------------------------------------------
    Routing Table : _public_
    Summary Count : 2
    
    Destination/Mask    Proto   Pre  Cost        Flags NextHop         Interface
    
         10.100.0.3/32  OSPF    10   2             D   10.0.1.0        GE1/0/2
                        OSPF    10   2             D   10.0.0.0        GE1/0/1
## готово!

