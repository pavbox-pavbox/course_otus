# Домашнее задание №7

![image](https://github.com/user-attachments/assets/2419de20-fc9f-46ec-bffb-1b0d50f95adc)

### Изучение возможностей multihoming на Arista и Huawei

VXLAN фабрика аналогична предыдущей домашней работе, поэтому приводить ее здесь нет смысла. Остановимся на натройках непосредственной multihoming

В качесте конечных устройств будут выступать коммутаторы Ариста и Хуавей. На них настроен Portchannel и Eth-trunk соотвественно. Создан Vlan 20, присвоен ip адрес, разрешен пропуск влана 20 через агрегированный порт

#### Huawei (ip 192.168.20.120)
    
    vlan batch 20
    #
    interface Vlanif20
     ip address 192.168.20.120 255.255.255.0
    #
    interface Eth-Trunk1
     port link-type trunk
     port trunk allow-pass vlan 20
     mode lacp-dynamic
    #
    interface GE1/0/0
     undo shutdown
     eth-trunk 1
    #
    interface GE1/0/1
     undo shutdown
     eth-trunk 1
    #
    ip route-static 0.0.0.0 0.0.0.0 192.168.20.1

#### Arista (ip 192.168.20.130)

    vlan 20
    !
    interface Port-Channel1
       switchport trunk allowed vlan 20
       switchport mode trunk
    !
    interface Ethernet1
       channel-group 1 mode active
    !
    interface Ethernet2
       channel-group 1 mode active
    !
    interface Vlan20
       ip address 192.168.20.130/24
    !
    ip routing
    !
    ip route 0.0.0.0/0 192.168.20.1
    !

Далее настраиваем L2 и L3

Создаем два агрегированных интерфейса. ESI идентификаторы будут разные, маки интерфейсов также разные. В агрегированный интерфейс Po6 добавляем физический интерфейс Et6. Аналогично с Po7 и Et7.

Po6 будет смотреть в сторону коммутатора Arista, Po7 - в сторону Huawei

    interface Port-Channel6
       switchport trunk allowed vlan 20
       switchport mode trunk
       !
       evpn ethernet-segment
          identifier 0011:2222:3333:4444:7777
          route-target import de:ad:be:ef:00:01
       lacp system-id dead.beef.0001
    !
    interface Port-Channel7
       switchport trunk allowed vlan 20
       switchport mode trunk
       !
       evpn ethernet-segment
          identifier 0011:2222:3333:4444:6666
          route-target import de:ad:be:ef:00:00
       lacp system-id dead.beef.0000
    !
    interface Ethernet6
       channel-group 6 mode active
       lacp timer fast
    !
    interface Ethernet7
       channel-group 7 mode active
       lacp timer fast
    !

Виртуальный адрес и его мак был настроен ранее, в предыдущей работе

    interface Vlan20
       vrf VXLAN
       ip address virtual 192.168.20.1/24
    !
    ip virtual-router mac-address 00:aa:aa:aa:aa:aa
    !

#### проверяем, что получилось

На стороне L2 агрегированные линки поднялись (Po6 и Po7)

    L2(config-if-Et6)# sh int statu
    Port       Name   Status       Vlan     Duplex Speed  Type            Flags Encapsulation
    Et1               connected    routed   full   1G     EbraTestPhyPort
    Et2               connected    routed   full   1G     EbraTestPhyPort
    Et3               connected    1        full   1G     EbraTestPhyPort
    Et4               connected    1        full   1G     EbraTestPhyPort
    Et5               connected    1        full   1G     EbraTestPhyPort
    Et6               connected    in Po6   full   1G     EbraTestPhyPort
    Et7               connected    in Po7   full   1G     EbraTestPhyPort
    Et8               connected    20       full   1G     EbraTestPhyPort
    Ma1               connected    routed   a-full a-1G   10/100/1000
    Po6               connected    trunk    full   1G     N/A
    Po7               connected    trunk    full   1G     N/A

на стороне Arist-ы линк поднялся (Po1)

    PortChannel(config)#sh int sta
    Port       Name   Status       Vlan     Duplex Speed  Type            Flags Encapsulation
    Et1               connected    in Po1   full   1G     EbraTestPhyPort
    Et2               connected    in Po1   full   1G     EbraTestPhyPort
    Et3               connected    1        full   1G     EbraTestPhyPort
    Et4               connected    1        full   1G     EbraTestPhyPort
    Et5               connected    1        full   1G     EbraTestPhyPort
    Et6               connected    1        full   1G     EbraTestPhyPort
    Et7               connected    1        full   1G     EbraTestPhyPort
    Et8               connected    1        full   1G     EbraTestPhyPort
    Ma1               connected    routed   a-full a-1G   10/100/1000
    Po1               connected    trunk    full   2G     N/A

на стороне Huawei линк поднялся (ET1)

    [~HUAWEI]dis int bri
    
    InUti/OutUti: input utility rate/output utility rate
    Interface                  PHY      Protocol  InUti OutUti   inErrors  outErrors
    Eth-Trunk1                 up       up           0%  0.01%          0          0
      GE1/0/0                  up       up           0%  0.01%          0          0
      GE1/0/1                  up       up           0%  0.01%          0          0
    GE1/0/2                    *down    down         0%     0%          0          0
    GE1/0/3                    *down    down         0%     0%          0          0
    GE1/0/4                    *down    down         0%     0%          0          0
    GE1/0/5                    *down    down         0%     0%          0          0
    GE1/0/6                    *down    down         0%     0%          0          0
    GE1/0/7                    *down    down         0%     0%          0          0
    GE1/0/8                    *down    down         0%     0%          0          0
    GE1/0/9                    *down    down         0%     0%          0          0
    MEth0/0/0                  up       down         0%     0%          0          0
    NULL0                      up       up(s)        0%     0%          0          0
    Vlanif20                   up       up           --     --          0          0


пробуем пингать

с Аристы шлюз и удаленный хост в другом влане (из предыдущей лабы )

    PortChannel(config)#ping 192.168.20.1
    PING 192.168.20.1 (192.168.20.1) 72(100) bytes of data.
    80 bytes from 192.168.20.1: icmp_seq=1 ttl=64 time=6.11 ms
    80 bytes from 192.168.20.1: icmp_seq=2 ttl=64 time=3.74 ms
    80 bytes from 192.168.20.1: icmp_seq=3 ttl=64 time=2.64 ms
    80 bytes from 192.168.20.1: icmp_seq=4 ttl=64 time=2.52 ms
    80 bytes from 192.168.20.1: icmp_seq=5 ttl=64 time=2.53 ms
    
    --- 192.168.20.1 ping statistics ---
    5 packets transmitted, 5 received, 0% packet loss, time 23ms
    rtt min/avg/max/mdev = 2.523/3.511/6.114/1.379 ms, ipg/ewma 5.770/4.743 ms
    PortChannel(config)#ping 192.168.10.10
    PING 192.168.10.10 (192.168.10.10) 72(100) bytes of data.
    80 bytes from 192.168.10.10: icmp_seq=1 ttl=62 time=11.5 ms
    80 bytes from 192.168.10.10: icmp_seq=2 ttl=62 time=7.83 ms
    80 bytes from 192.168.10.10: icmp_seq=3 ttl=62 time=7.72 ms
    80 bytes from 192.168.10.10: icmp_seq=4 ttl=62 time=8.00 ms
    80 bytes from 192.168.10.10: icmp_seq=5 ttl=62 time=9.29 ms
    
    --- 192.168.10.10 ping statistics ---
    5 packets transmitted, 5 received, 0% packet loss, time 45ms
    rtt min/avg/max/mdev = 7.723/8.887/11.576/1.460 ms, ipg/ewma 11.373/10.218 ms
    PortChannel(config)#

пинги ходят, все получилось

теперь с хуавея:

    [~HUAWEI]ping 192.168.20.1
      PING 192.168.20.1: 56  data bytes, press CTRL_C to break
        Request time out
        Request time out
        Request time out
        Request time out
        Request time out
    
      --- 192.168.20.1 ping statistics ---
        5 packet(s) transmitted
        0 packet(s) received
        100.00% packet loss

не пингается даже шлюзовой адрес. Смотрим дампы.

![image](https://github.com/user-attachments/assets/2a29605d-721d-4601-bbad-25ac81e7d611)

есть запрос ARP, есть reply, но huawei почему-то не пополняет свою ARP таблицу:

    [~HUAWEI]dis arp
    ARP Entry Types: D - Dynamic, S - Static, I - Interface, O - OpenFlow, RD - Redirect
    EXP: Expire-time VLAN:VLAN or Bridge Domain
    
    IP ADDRESS      MAC ADDRESS    EXP(M) TYPE/VLAN       INTERFACE        VPN-INSTANCE
    ----------------------------------------------------------------------------------------
    192.168.20.120  3864-0111-1200        I               Vlanif20
    192.168.20.1    Incomplete        1   D               Vlanif20
    ----------------------------------------------------------------------------------------
    Total:2         Dynamic:1       Static:0    Interface:1    OpenFlow:0
    Redirect:0
    [~HUAWEI]

Очень интересная особенность Хуавея. Что с ней делать - непонятно. В итоге на хувике схема неработоспособна, однако на Аристе - ESI LAG собрался и работоспособен. Проверим скорость реакции при отключении одного порта из ЛАГа:

    PortChannel(config)#ping 192.168.10.10 repeat 1000 interval 1
    PING 192.168.10.10 (192.168.10.10) 72(100) bytes of data.
    80 bytes from 192.168.10.10: icmp_seq=1 ttl=62 time=10.1 ms
    80 bytes from 192.168.10.10: icmp_seq=2 ttl=62 time=17.1 ms
    80 bytes from 192.168.10.10: icmp_seq=3 ttl=62 time=18.6 ms
    80 bytes from 192.168.10.10: icmp_seq=4 ttl=62 time=27.3 ms
    80 bytes from 192.168.10.10: icmp_seq=5 ttl=62 time=15.7 ms
    80 bytes from 192.168.10.10: icmp_seq=6 ttl=62 time=12.4 ms
    80 bytes from 192.168.10.10: icmp_seq=7 ttl=62 time=16.3 ms
    80 bytes from 192.168.10.10: icmp_seq=8 ttl=62 time=10.6 ms
    80 bytes from 192.168.10.10: icmp_seq=9 ttl=62 time=11.7 ms
    80 bytes from 192.168.10.10: icmp_seq=10 ttl=62 time=16.5 ms
    80 bytes from 192.168.10.10: icmp_seq=11 ttl=62 time=12.5 ms
    80 bytes from 192.168.10.10: icmp_seq=12 ttl=62 time=13.8 ms
    80 bytes from 192.168.10.10: icmp_seq=13 ttl=62 time=11.1 ms
    80 bytes from 192.168.10.10: icmp_seq=14 ttl=62 time=13.0 ms
    80 bytes from 192.168.10.10: icmp_seq=15 ttl=62 time=14.9 ms
    80 bytes from 192.168.10.10: icmp_seq=17 ttl=62 time=14.1 ms
    80 bytes from 192.168.10.10: icmp_seq=18 ttl=62 time=9.56 ms
    80 bytes from 192.168.10.10: icmp_seq=19 ttl=62 time=10.8 ms
    80 bytes from 192.168.10.10: icmp_seq=20 ttl=62 time=9.50 ms

отсутствует icmp_seq=16 Результат удовлетворительный.

# Задача решена
