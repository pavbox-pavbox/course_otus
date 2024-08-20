## Проектирование сети ЦОД на базе VXLAN/EVPN для предоставления услуг по передаче трафика в системы хранения содержимого

![image](https://github.com/user-attachments/assets/ab3ba58d-a62a-4c58-b998-30e316da0247)


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

Теперь пришло время из всего этого извлечь пользу. Мы условились, что клиентам будем предоставлять L2VPN. Это позволит нам никак не ограничивать клиентов в выборе IP адресов. Каждый клиент (а под клиентами мы подразумеваем как источники трафика, так и системы обработки и хранения трафика) будет коммуницировать в своем L2 домене. Домены, соответсвтенно, изолированы друг от друга. Для того, чтобы упростить дальнейшую эксплуатацию нашей транспортной сети, согласование vlan-ов и сетей в них переходит в зону ответственности эксплуатации.

Например, пара "генератор трафика - система хранения" для региона Лондон будет коммуницировать между через vlan 1234. Для пары "Москва" влан взаимодействия будет vlan 3210

Чтобы не менять настройки всей нашей фабрики каждый раз, когда в эксплуатацию вводится новый клиент, применим следующее решение:

На лифах создаем небольшой набор VLAN

        !
        vlan 100,200,300

Например, VLAN100 у нас будет предназначен для организации каналов управления. Если бы образы предоставляли возможность traffic-engineering, то мы бы предоставили ему наивысшый приоритет и минимальные задержки. VLAN200 - организация каналов передачи непосредственно ползователького трафика, VLAN300 - зарезервирован для будущих нужд

Далее нам нужно распространить эти VLAN по нашей сети. Создаем Vxlan интерфейс, в котором тунеллируем эти вланы в определенные VNI

        interface Vxlan1
           vxlan source-interface Loopback0
           vxlan udp-port 4789
           vxlan vlan 100 vni 10100
           vxlan vlan 200 vni 10200
           vxlan vlan 300 vni 10300

И далее в настройках BGP распространяем их по нашей сети

        router bgp 65001
        
           !
           vlan 100
              rd 10100:10100
              route-target both 10100:10100
              redistribute learned
           !
           vlan 200
              rd 10200:10200
              route-target both 10200:10200
              redistribute learned
           !
           vlan 300
              rd 10300:10300
              route-target both 10300:10300
              redistribute learned
           !

Этот набор вланов будет меняться редко. Однако клиенты будут меняться довольно часто. Поэтому на клиентской границе мы настроим QinQ. Такое решение избавит от необходимости перенастройки всей фабрики при вводе в эксплуатацию новых клиентов.

Включаем QinQ на интерфейсе и сообщаем, какую верхнюю метку будем навешивать

        interface Port-Channel1
           switchport access vlan 100
           switchport mode dot1q-tunnel
           !
           evpn ethernet-segment
              identifier 0011:2222:3333:4444:5555
              route-target import de:ad:be:ef:00:01
           lacp system-id dead.beef.0001
        !

В данном примере на интерфейсе Port-channel1 настроен QinQ с верхней меткой 100. Здесь есть возможность ошибки, когда разные клиенты возьмут себе один влан, но это мы оставляем на откуп инженерам экплуатации. Кроме этого, в данном примере настроен port-channel с использованием EVPN with All-Active multihoming. Со стороны клиента данный данный порт выглядит как обычный LACP порт, но фактически клиент подключен к двум разным коробкам.

Осталось проверить работоспособность нашей идеи. Все ли работает так, как задумано?

В качестве клиента - генератора трафика выступает коммутатор с настроенным Port-Channel. В port-channel добавлены два физических порта, порт работает в режиме trunk, и пропускает только vlan 1234. Также в данном влане создаем ip интерфейс с тестовым адресом

        interface Port-Channel1
           switchport trunk allowed vlan 1234
           switchport mode trunk
        !
        interface Ethernet1
           channel-group 1 mode active
        !
        interface Ethernet2
           channel-group 1 mode active
        !
        interface Vlan1234
           ip address 192.168.100.1/24
        !

Как мы видели чуть ранее, коммутаторы LSW ничего не знают про клиентский влан.

На дальней стороне делаем аналогичные настройки

        interface Port-Channel1
           switchport trunk allowed vlan 1234
           switchport mode trunk
        !
        interface Ethernet1
           channel-group 1 mode active
        !
        interface Ethernet2
           channel-group 1 mode active
        !
        interface Vlan1234
           ip address 192.168.100.2/24

Проверяем связность - связность есть

        shd1(config)#ping 192.168.100.1
        PING 192.168.100.1 (192.168.100.1) 72(100) bytes of data.
        80 bytes from 192.168.100.1: icmp_seq=1 ttl=64 time=20.1 ms
        80 bytes from 192.168.100.1: icmp_seq=2 ttl=64 time=12.9 ms
        80 bytes from 192.168.100.1: icmp_seq=3 ttl=64 time=12.7 ms
        80 bytes from 192.168.100.1: icmp_seq=4 ttl=64 time=10.5 ms
        80 bytes from 192.168.100.1: icmp_seq=5 ttl=64 time=11.9 ms

Заглянем внутрь кадров на участке, например, SSW-BLSW. 

![image](https://github.com/user-attachments/assets/efd5ce88-09fb-4174-ae5b-bdadb0507c82)

Видим, что непосредственно ICMP пакеты идут с меткой VID 1234, а уровнем выше - заголовки VXLAN  с VNI 10100 (который маппится во влан VID100). Делаем вывод, что все работает так, как нужно. 

### Схема рабочая, можно пользоваться

#### Осталось проверить наши цели.

##### 1. сеть можно развивать в пределах портовой емкости оборудования - IP план позволяет
##### 2. географическое разделение сети не помеха для корректной работы
##### 3. ввиду небольшого набора тунеллируемых VLAN-ов настройки идентичны на всех лифах, что позволяет мигрировать клиентов без перенастройки IP фабрики
##### 4. клиенты находятся каждый в своем влане, поэтому трафик изолирован. Однако необходимо следить за тем, чтобы номера вланов у разных клиентов не совпадали. В случае же необходимости, чтобы у разных клиентов был одинаковый VLAN, можно организовать еще один маппинг вланов в VNI, настроить всю фабрику, и отдать в сторону клиентов уже не влан 200, а условный влан 201. В этом случае пересесения трафика между клиентами в одном влане не будет

# Цели достигнуты, задача решена


<details>
<summary>LSW1</summary>
    
        ! device: LSW1 (vEOS-lab, EOS-4.29.2F)
        !
        ! boot system flash:/vEOS-lab.swi
        !
        no aaa root
        !
        transceiver qsfp default-mode 4x10G
        !
        service routing protocols model multi-agent
        !
        hostname LSW1
        !
        spanning-tree mode mstp
        !
        vlan 100,200,300
        !
        interface Port-Channel1
           switchport access vlan 100
           switchport mode dot1q-tunnel
           !
           evpn ethernet-segment
              identifier 0011:2222:3333:4444:5555
              route-target import de:ad:be:ef:00:01
           lacp system-id dead.beef.0001
        !
        interface Ethernet1
           no switchport
           ip address 10.0.0.0/31
        !
        interface Ethernet2
           no switchport
           ip address 10.1.0.0/31
        !
        interface Ethernet3
           channel-group 1 mode active
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
           switchport access vlan 200
        !
        interface Loopback0
           ip address 10.100.0.1/32
        !
        interface Management1
        !
        interface Vxlan1
           vxlan source-interface Loopback0
           vxlan udp-port 4789
           vxlan vlan 100 vni 10100
           vxlan vlan 200 vni 10200
           vxlan vlan 300 vni 10300
        !
        ip routing
        !
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
           vlan 100
              rd 10100:10100
              route-target both 10100:10100
              redistribute learned
           !
           vlan 200
              rd 10200:10200
              route-target both 10200:10200
              redistribute learned
           !
           vlan 300
              rd 10300:10300
              route-target both 10300:10300
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
           network 10.0.0.0/31 area 0.0.0.0
           network 10.1.0.0/31 area 0.0.0.0
           network 10.100.0.1/32 area 0.0.0.0
           max-lsa 12000
           maximum-paths 2
        !
        end
</details>


<details>
<summary>LSW2</summary>

```
! device: LSW2 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model multi-agent
!
hostname LSW2
!
spanning-tree mode mstp
!
vlan 100,200,300
!
interface Port-Channel1
   evpn ethernet-segment
      identifier 0011:2222:3333:4444:5555
      route-target import de:ad:be:ef:00:01
   lacp system-id dead.beef.0001
!
interface Ethernet1
   no switchport
   ip address 10.2.0.0/31
!
interface Ethernet2
   no switchport
   ip address 10.3.0.0/31
!
interface Ethernet3
   channel-group 1 mode active
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
interface Loopback0
   ip address 10.100.0.2/32
!
interface Management1
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 100 vni 10100
   vxlan vlan 200 vni 10200
   vxlan vlan 300 vni 10300
!
ip routing
!
router bgp 65001
   router-id 10.100.0.2
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
   vlan 100
      rd 10100:10100
      route-target both 10100:10100
      redistribute learned
   !
   vlan 200
      rd 10200:10200
      route-target both 10200:10200
      redistribute learned
   !
   vlan 300
      rd 10300:10300
      route-target both 10300:10300
      redistribute learned
   !
   address-family evpn
      neighbor SPINES activate
   !
   address-family ipv4
      no neighbor SPINES activate
!
router ospf 1
   router-id 10.100.0.2
   network 10.2.0.0/31 area 0.0.0.0
   network 10.3.0.0/31 area 0.0.0.0
   network 10.100.0.2/32 area 0.0.0.0
   max-lsa 12000
   maximum-paths 2
!
end
```
</details>

<details>
<summary>SSW1</summary>

```
! device: SSW1 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model multi-agent
!
hostname SSW1
!
spanning-tree mode mstp
!
interface Ethernet1
   no switchport
   ip address 10.0.0.1/31
!
interface Ethernet2
   no switchport
   ip address 10.2.0.1/31
!
interface Ethernet3
   no switchport
   ip address 10.0.1.0/31
!
interface Ethernet4
   no switchport
   ip address 10.1.1.0/31
!
interface Ethernet5
   no switchport
   ip address 10.4.1.0/31
!
interface Ethernet6
!
interface Ethernet7
!
interface Ethernet8
!
interface Loopback0
   ip address 10.101.0.1/32
!
interface Management1
!
ip routing
!
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
router ospf 1
   router-id 10.101.0.1
   network 10.0.0.0/31 area 0.0.0.0
   network 10.0.1.0/31 area 0.0.0.0
   network 10.1.1.0/31 area 0.0.0.0
   network 10.2.0.0/31 area 0.0.0.0
   network 10.4.1.0/31 area 0.0.0.0
   network 10.101.0.1/32 area 0.0.0.0
   max-lsa 12000
   maximum-paths 2
!
end
```
</details>

<details>
<summary>SSW2</summary>

```
! device: SSW2 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model multi-agent
!
hostname SSW2
!
spanning-tree mode mstp
!
interface Ethernet1
   no switchport
   ip address 10.1.0.1/31
!
interface Ethernet2
   no switchport
   ip address 10.3.0.1/31
!
interface Ethernet3
   no switchport
   ip address 10.2.1.0/31
!
interface Ethernet4
   no switchport
   ip address 10.3.1.0/31
!
interface Ethernet5
   no switchport
   ip address 10.5.1.0/31
!
interface Ethernet6
!
interface Ethernet7
!
interface Ethernet8
!
interface Loopback0
   ip address 10.101.0.2/32
!
interface Management1
!
ip routing
!
router bgp 65100
   router-id 10.101.0.2
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
   neighbor SPINES peer group
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
router ospf 1
   router-id 10.101.0.2
   network 10.1.0.0/31 area 0.0.0.0
   network 10.2.1.0/31 area 0.0.0.0
   network 10.3.0.0/31 area 0.0.0.0
   network 10.3.1.0/31 area 0.0.0.0
   network 10.5.1.0/31 area 0.0.0.0
   network 10.101.0.2/32 area 0.0.0.0
   max-lsa 12000
   maximum-paths 2
!
end
```
    
</details>

<details>
<summary>BLSW1</summary>

```
! device: BLSW1 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model multi-agent
!
hostname BLSW1
!
spanning-tree mode mstp
!
vlan 100,200,300
!
interface Port-Channel1
   switchport access vlan 100
   switchport mode dot1q-tunnel
   !
   evpn ethernet-segment
      identifier 0011:2222:3333:4444:5566
      route-target import de:ad:be:ef:00:02
   lacp system-id dead.beef.0002
!
interface Ethernet1
   no switchport
   ip address 10.0.1.1/31
!
interface Ethernet2
   no switchport
   ip address 10.2.1.1/31
!
interface Ethernet3
   channel-group 1 mode active
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
   switchport access vlan 200
!
interface Loopback0
   ip address 10.102.0.1/32
!
interface Management1
!
interface Vlan200
   ip address 192.168.200.100/24
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 100 vni 10100
   vxlan vlan 200 vni 10200
   vxlan vlan 300 vni 10300
!
ip routing
!
router bgp 65001
   router-id 10.102.0.1
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
   vlan 100
      rd 10100:10100
      route-target both 10100:10100
      redistribute learned
   !
   vlan 200
      rd 10200:10200
      route-target both 10200:10200
      redistribute learned
   !
   vlan 300
      rd 10300:10300
      route-target both 10300:10300
      redistribute learned
   !
   address-family evpn
      neighbor SPINES activate
   !
   address-family ipv4
      no neighbor SPINES activate
!
router ospf 1
   router-id 10.102.0.1
   network 10.0.1.0/31 area 0.0.0.0
   network 10.2.1.0/31 area 0.0.0.0
   network 10.102.0.1/32 area 0.0.0.0
   max-lsa 12000
   maximum-paths 2
!
end
```
</details>

<details>
<summary>BLSW2</summary>

```
! device: BLSW2 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model multi-agent
!
hostname BLSW2
!
spanning-tree mode mstp
!
vlan 100,200,300
!
interface Port-Channel1
   switchport access vlan 100
   switchport mode dot1q-tunnel
   !
   evpn ethernet-segment
      identifier 0011:2222:3333:4444:5566
      route-target import de:ad:be:ef:00:02
   lacp system-id dead.beef.0002
!
interface Ethernet1
   no switchport
   ip address 10.1.1.1/31
!
interface Ethernet2
   no switchport
   ip address 10.3.1.1/31
!
interface Ethernet3
!
interface Ethernet4
   channel-group 1 mode active
!
interface Ethernet5
!
interface Ethernet6
!
interface Ethernet7
!
interface Ethernet8
!
interface Loopback0
   ip address 10.102.0.2/32
!
interface Management1
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 100 vni 10100
   vxlan vlan 200 vni 10200
   vxlan vlan 300 vni 10300
!
ip routing
!
router bgp 65001
   router-id 10.102.0.2
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
   vlan 100
      rd 10100:10100
      route-target both 10100:10100
      redistribute learned
   !
   vlan 200
      rd 10200:10200
      route-target both 10200:10200
      redistribute learned
   !
   vlan 300
      rd 10300:10300
      route-target both 10300:10300
      redistribute learned
   !
   address-family evpn
      neighbor SPINES activate
   !
   address-family ipv4
      no neighbor SPINES activate
!
router ospf 1
   router-id 10.102.0.2
   network 10.1.1.0/31 area 0.0.0.0
   network 10.3.1.0/31 area 0.0.0.0
   network 10.102.0.2/32 area 0.0.0.0
   max-lsa 12000
!
end
```
</details>
