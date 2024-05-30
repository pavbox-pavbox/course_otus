
# Домашнее задание №1
![image](https://github.com/pavbox-pavbox/course_otus/assets/97456111/ebf2c347-9931-415e-a814-469fbf5218d6)
### На рисунке представлена сеть имени Чарльза Клоза
В качестве роутеров в данном задании использованы CE12800

Интерфейсы между лифами и спайнами переведены в режим `undo portswitch`, на интерфейсах назначены адреса из таблицы ниже

Сети между лифом и спайном выбраны с маской /31

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

Для обеспечения связности Leaf-Leaf на коммутаторах включен ospf
#### S1:    

    ospf 1 router-id 10.0.0.0
     enable traffic-adjustment
     area 0.0.0.0
      network 10.0.0.0 0.0.255.255
     area 0.0.0.1
      network 10.2.0.0 0.0.255.255
     area 0.0.0.2
      network 10.4.0.0 0.0.255.255
На S2 настройки аналогичные, за исключением router-id

#### L1
    ospf 1 router-id 10.0.0.1
     area 0.0.0.0
      network 10.0.0.0 0.0.255.255
настройки ospf на L2 и L3 пытливому читателю предлагается разработать самостоятельно

    <HUAWEI>dis ip rou
    Proto: Protocol        Pre: Preference
    Route Flags: R - relay, D - download to fib, T - to vpn-instance, B - black hole route
    ------------------------------------------------------------------------------
    Routing Table : _public_
             Destinations : 12       Routes : 12        

    Destination/Mask    Proto   Pre  Cost        Flags NextHop         Interface
    
       10.0.0.0/31  Direct  0    0             D   10.0.0.1        GE1/0/1
       10.0.0.1/32  Direct  0    0             D   127.0.0.1       GE1/0/1
       10.0.1.0/31  Direct  0    0             D   10.0.1.1        GE1/0/2
       10.0.1.1/32  Direct  0    0             D   127.0.0.1       GE1/0/2
       10.2.0.0/31  OSPF    10   2             D   10.0.0.0        GE1/0/1
       10.2.1.0/31  OSPF    10   2             D   10.0.1.0        GE1/0/2
       10.4.0.0/31  OSPF    10   2             D   10.0.0.0        GE1/0/1
       10.4.1.0/31  OSPF    10   2             D   10.0.1.0        GE1/0/2
      127.0.0.0/8   Direct  0    0             D   127.0.0.1       InLoopBack0
      127.0.0.1/32  Direct  0    0             D   127.0.0.1       InLoopBack0
    127.255.255.255/32  Direct  0    0             D   127.0.0.1       InLoopBack0
    255.255.255.255/32  Direct  0    0             D   127.0.0.1       InLoopBack0
    <HUAWEI>ping 10.4.1.1
      PING 10.4.1.1: 56  data bytes, press CTRL_C to break
    Reply from 10.4.1.1: bytes=56 Sequence=1 ttl=254 time=37 ms
    Reply from 10.4.1.1: bytes=56 Sequence=2 ttl=254 time=6 ms
    Reply from 10.4.1.1: bytes=56 Sequence=3 ttl=254 time=7 ms

Связность есть

