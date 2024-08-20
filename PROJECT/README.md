## Проектирование сети ЦОД на базе VXLAN/EVPN для предоставления услуг по передаче трафика в системы хранения содержимого


### Цель проектирования: спроектировать сеть передачи трафика в системы хранения содержимого. Предусмотреть:
#### 1. предусмотреть возможность дальнейшего расширения сети
#### 2. предусмотреть связь географически разнесенных мест размещения оборудования
#### 3. предусмотреть возможность миграции как источников контента, так и устройств хранения контента между различными автозалами
#### 4. предусмотреть возможность роста сети путем добавления пар "источник - хранилище" с изоляцией трафика от ранее установленных пар

В качестве основы нашей сети будем использовать топологию на основе сетей Клоза. Источники контента мы будем подключать к левой части сети, к коммутаторам LSW, хранилище контента будет справа, коммутаторы BLSW. Основное оборудование подключается к паре коммутаторов, третий коммутатор BLSW будет демонстрировать возможность миграции оборудования между автозалами.

Начинаем нашу сеть с плана IP адресации

Сначала организуем стык LSW со спайнами SSW. Сети используем с маской /31, не забываем про возможную суммаризацию сетей. Сети с нулевым третьим октетом выбраны для связи LSW-SSW

|Node|Интерфейс|Адрес|-|Адрес|Интерфейс|Node|
|-|-|-|-|-|-|-|
|LSW1|Ethernet1|10.0.0.0|-|10.0.0.1|Ethernet1|SSW1|
|LSW1|Ethernet2|10.1.0.0|-|10.1.0.1|Ethernet1|SSW2|
|LSW2|Ethernet1|10.2.0.0|-|10.2.0.1|Ethernet1|SSW1|
|LSW2|Ethernet2|10.3.0.0|-|10.3.0.1|Ethernet1|SSW2|

Далее - SSW с BLSW. Для их связи выбраны сети с третьим октетом, раным единице

|Node|Интерфейс|Адрес|-|Адрес|Интерфейс|Node|
|-|-|-|-|-|-|-|
|SSW1|Ethernet1|10.0.1.0|-|10.0.1.1|Ethernet1|BLSW1|
|SSW1|Ethernet2|10.1.1.0|-|10.1.1.1|Ethernet1|BLSW2|
|SSW2|Ethernet1|10.2.1.0|-|10.2.1.1|Ethernet1|BLSW1|
|SSW2|Ethernet2|10.3.1.0|-|10.3.1.1|Ethernet1|BLSW2|

Стык с BLSW3 (условный отдельный автозал)

|Node|Интерфейс|Адрес|-|Адрес|Интерфейс|Node|
|-|-|-|-|-|-|-|
|SSW1|Ethernet1|10.4.1.0|-|10.4.1.1|Ethernet1|BLSW3|
|SSW2|Ethernet1|10.5.1.0|-|10.5.1.1|Ethernet1|BLSW3|

Интерфейсы объявляются как тока-точка. Настройка портов будет выглядеть так:

    interface Ethernet1
       no switchport
       ip address 10.0.0.0/31
       ip ospf network point-to-point

Далее - очередь loopback интерфейсов, на которых мы будем строить underlay сеть и overlay сеть. Здесь суммаризация не требуется, поэтому делаем просто запоминающиеся адреса. Роли коммутаторов будут отличаться вторым октетом адреса. Адреса используем с маской /32

|hostname|ip|	|hostname|ip|	|hostname|ip|
|-|-|-|-|-|-|-|-|
|LSW1|10.100.0.1|	|SSW1|10.101.0.1|	|BLSW1|10.102.0.1|
|LSW2|10.100.0.2|	|SSW1|10.101.0.2|	|BLSW1|10.102.0.2|
| | |	| | |	|BLSW3|10.102.0.3|

Сети выделены, первый шаг пройден.

Настраиваем ospf. Настройки показаны на примере LSW1

Не забыть включить роутинг глобально:

        ip routing
        !
        router ospf 1
           router-id 10.100.0.1
           network 10.0.0.0/31 area 0.0.0.0
           network 10.1.0.0/31 area 0.0.0.0
           network 10.100.0.1/32 area 0.0.0.0
           max-lsa 12000
           maximum-paths 2
        !
Анонсируем сеть loopback интерфейса и стыковые сети. Можно будет посмотреть трейсами, как ходит пакет

Делаем аналогичные настройки на остальных коммутаторах и проверяем, что получилось

        LSW1(config-if-Et2)#sh ip route ospf
        
        VRF: default
        
        
         O        10.0.1.0/31 [110/20] via 10.0.0.1, Ethernet1
         O        10.1.1.0/31 [110/20] via 10.0.0.1, Ethernet1
         O        10.2.0.0/31 [110/20] via 10.0.0.1, Ethernet1
         O        10.2.1.0/31 [110/20] via 10.1.0.1, Ethernet2
         O        10.3.0.0/31 [110/20] via 10.1.0.1, Ethernet2
         O        10.3.1.0/31 [110/20] via 10.1.0.1, Ethernet2
         O        10.4.1.0/31 [110/20] via 10.0.0.1, Ethernet1
         O        10.5.1.0/31 [110/20] via 10.1.0.1, Ethernet2
         O        10.100.0.2/32 [110/30] via 10.0.0.1, Ethernet1
                                         via 10.1.0.1, Ethernet2
         O        10.101.0.1/32 [110/20] via 10.0.0.1, Ethernet1
         O        10.101.0.2/32 [110/20] via 10.1.0.1, Ethernet2
         O        10.102.0.1/32 [110/30] via 10.0.0.1, Ethernet1
                                         via 10.1.0.1, Ethernet2
         O        10.102.0.2/32 [110/30] via 10.0.0.1, Ethernet1
                                         via 10.1.0.1, Ethernet2
         O        10.102.0.3/32 [110/30] via 10.0.0.1, Ethernet1
                                         via 10.1.0.1, Ethernet2

Наблюдаем маршруты до loopback интерфейсов нашей сети, и видим их через два интерфеса. Что и требовалось. Второй шаг считаем завершенным.

Наступает время настройки overlay сети. Overlay будем строить на eBGP. Все лифы будут в ASN65001, все спайны - в ASN65100

Настройки лифов и спайнов будут различаться, поэтому приведем их последовательно. Сначала спайн. 

Для удобства дальнейшего развития сети настройки лифов на спайне выделяем в отдельную группу, настраиваем необходимым образом, и далее объявляем соседа членом этой группы. Не забываем уточнить, что тут мы строим EVPN, а не IP связность

        router bgp 65100
           router-id 10.101.0.1
           timers bgp 3 9
           maximum-paths 8 ecmp 16
           neighbor LEAFS peer group
           neighbor LEAFS remote-as 65001
           neighbor LEAFS next-hop-unchanged
           neighbor LEAFS update-source Loopback0
           neighbor LEAFS bfd
           neighbor LEAFS allowas-in 2
           neighbor LEAFS ebgp-multihop 10
           neighbor LEAFS send-community extended
           neighbor 10.100.0.1 peer group LEAFS
           neighbor 10.100.0.2 peer group LEAFS
           neighbor 10.102.0.1 peer group LEAFS
           neighbor 10.102.0.2 peer group LEAFS
           neighbor 10.102.0.3 peer group LEAFS
           !
           address-family evpn
              neighbor LEAFS activate
           !
           address-family ipv4
              no neighbor LEAFS activate
        !

Далее - лифы. Пример - LSW1. Также делаем группу, и также EVPN

        router bgp 65001
           router-id 10.100.0.1
           timers bgp 3 9
           maximum-paths 10
           neighbor SPINES peer group
           neighbor SPINES remote-as 65100
           neighbor SPINES update-source Loopback0
           neighbor SPINES bfd
           neighbor SPINES allowas-in 2
           neighbor SPINES ebgp-multihop 10
           neighbor SPINES send-community extended
           neighbor 10.101.0.1 peer group SPINES
           neighbor 10.101.0.2 peer group SPINES
           !
           address-family evpn
              neighbor SPINES activate
           !
           address-family ipv4
              no neighbor SPINES activate
        !

Проверяем, что получилось на одном из спайнов

        SSW1(config-if-Et2)# sh bgp summary
        BGP summary information for VRF default
        Router identifier 10.101.0.1, local AS number 65100
        Neighbor            AS Session State AFI/SAFI                AFI/SAFI State   NLRI Rcd   NLRI Acc
        ---------- ----------- ------------- ----------------------- -------------- ---------- ----------
        10.100.0.1       65001 Established   L2VPN EVPN              Negotiated              9          9
        10.100.0.2       65001 Established   L2VPN EVPN              Negotiated              3          3
        10.102.0.1       65001 Established   L2VPN EVPN              Negotiated              9          9
        10.102.0.2       65001 Established   L2VPN EVPN              Negotiated              6          6
        10.102.0.3       65001 Established   L2VPN EVPN              Negotiated             15         15

Сессии со всеми соседями в Established, значит, все получилось. Третий шаг пройден.
