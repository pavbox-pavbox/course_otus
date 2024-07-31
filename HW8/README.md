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

