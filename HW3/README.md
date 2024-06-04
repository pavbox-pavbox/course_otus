# Домашнее задание №3

![image](https://github.com/pavbox-pavbox/course_otus/assets/97456111/79296ab1-2ee8-4be3-b00f-13ec268670d6)

### Продолжаем ставить эксперименты с сетью Клоза

## Настройка ISIS

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

Настройка ISIS на коммутаторе L1. network-entity формируется особым образом из адреса Lo1. коммутатору задается уровень ISIS L1

    isis 1
     is-level level-1
     bfd all-interfaces enable
     network-entity 49.0101.0000.0001.00

Включение ISIS на интерфесах коммутатора ...

    interface GE1/0/1
     undo portswitch
     undo shutdown
     ip address 10.0.0.1 255.255.255.254
     isis enable 1
     isis bfd enable
    #
    interface GE1/0/2
     undo portswitch
     undo shutdown
     ip address 10.0.1.1 255.255.255.254
     isis enable 1
     isis bfd enable

... и включение ISIS на интерфейсе Lo1

    interface LoopBack1
     ip address 10.100.0.1 255.255.255.255
     isis enable 1

Настройки протокола на примере S1. Без явного указания уровня ISIS huawei назначает уровень L1-L2

    isis 1
     bfd all-interfaces enable
     network-entity 49.0102.0000.0001.00

Один из интерфейсов в сторону коммутаторов уровня Leaf (остальные аналогично)

    interface GE1/0/1
     undo portswitch
     undo shutdown
     ip address 10.0.0.0 255.255.255.254
     isis enable 1
     isis bfd enabl

### Проверяем

связность с пирами isis

    <L1>dis isis peer

    Peer Information for ISIS(1)
    --------------------------------------------------------------------------------

      System ID     Interface       Circuit ID        State HoldTime(s) Type     PRI
    --------------------------------------------------------------------------------
    0102.0000.0001  GE1/0/1         0101.0000.0001.01  Up            30 L1        64
    0102.0000.0002  GE1/0/2         0102.0000.0002.01  Up             8 L1        64

    Total Peer(s): 2

маршруты в таблице маршрутизации

	<L1>dis ip rout protocol isis
	Proto: Protocol        Pre: Preference
	Route Flags: R - relay, D - download to fib, T - to vpn-instance, B - black hole route
	------------------------------------------------------------------------------
	_public_ Routing Table : IS-IS
			 Destinations : 9        Routes : 11

	IS-IS routing table status : <Active>
			 Destinations : 6        Routes : 8

	Destination/Mask    Proto   Pre  Cost        Flags NextHop         Interface

		   10.2.0.0/31  ISIS-L1 15   20            D   10.0.0.0        GE1/0/1
		   10.2.1.0/31  ISIS-L1 15   20            D   10.0.1.0        GE1/0/2
		   10.4.0.0/31  ISIS-L1 15   20            D   10.0.0.0        GE1/0/1
		   10.4.1.0/31  ISIS-L1 15   20            D   10.0.1.0        GE1/0/2
		 10.100.0.2/32  ISIS-L1 15   20            D   10.0.0.0        GE1/0/1
                            ISIS-L1 15   20            D   10.0.1.0        GE1/0/2
		 10.100.0.3/32  ISIS-L1 15   20            D   10.0.0.0        GE1/0/1
                            ISIS-L1 15   20            D   10.0.1.0        GE1/0/2

	IS-IS routing table status : <Inactive>
			 Destinations : 3        Routes : 3

	Destination/Mask    Proto   Pre  Cost        Flags NextHop         Interface

		   10.0.0.0/31  ISIS-L1 15   0                 10.0.0.1        GE1/0/1
		   10.0.1.0/31  ISIS-L1 15   0                 10.0.1.1        GE1/0/2
		 10.100.0.1/32  ISIS-L1 15   0                 10.100.0.1      LoopBack1

наблюдаем два маршрута равной стоимости до каждого leaf-а
### Задача решена
