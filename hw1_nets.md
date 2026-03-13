## Часть 1 База: работа бриджей (MAC-learning, flooding)

**Запишите в отчет МАС-адреса своих хостов, результаты пинг, выдержку из
tcpdump с ARP и затем юникаст обменом, содержимое МАС-таблиц обоих бриджей.
Объясните адреса в ARP-фреймах.**

#### MAC-адреса хостов

* MAC-адрес host1: aa:c1:ab:16:f2:0c
* MAC-адрес host2: aa:c1:ab:f6:ab:c2

MAC-адрес — это уникальный идентификатор сетевого интерфейса на канальном уровне. IPv6 генерируются из MAC-адресов. Они нужны для сетевого взаимодействия. Теперь пробуем пинговать

#### Результаты пинг второго хоста на первый
На br1 запустил сниффер трафика на интерфейсе eth1, чтобы увидеть, что вообще происходит
пинганул

```bash
/ # ping 192.168.1.10 -c 5
PING 192.168.1.10 (192.168.1.10): 56 data bytes
64 bytes from 192.168.1.10: seq=0 ttl=64 time=0.418 ms
64 bytes from 192.168.1.10: seq=1 ttl=64 time=0.139 ms
64 bytes from 192.168.1.10: seq=2 ttl=64 time=0.177 ms
64 bytes from 192.168.1.10: seq=3 ttl=64 time=0.135 ms
64 bytes from 192.168.1.10: seq=4 ttl=64 time=0.125 ms

--- 192.168.1.10 ping statistics ---
5 packets transmitted, 5 packets received, 0% packet loss
round-trip min/avg/max = 0.125/0.198/0.418 ms
```

смотрим что в выдержке из tcpdump:

```bash
00:18:53.594902 aa:c1:ab:7f:c8:d4 > 33:33:00:00:00:02, ethertype IPv6 (0x86dd), length 70: (hlim 255, next-header ICMPv6 (58) payload length: 16) fe80::a8c1:abff:fe7f:c8d4 > ff02::2: [icmp6 sum ok] ICMP6, router solicitation, length 16
          source link-address option (1), length 8 (1): aa:c1:ab:7f:c8:d4
            0x0000:  aac1 ab7f c8d4
00:28:10.651534 aa:c1:ab:1f:f4:e3 > 33:33:00:00:00:02, ethertype IPv6 (0x86dd), length 70: (hlim 255, next-header ICMPv6 (58) payload length: 16) fe80::a8c1:abff:fe1f:f4e3 > ff02::2: [icmp6 sum ok] ICMP6, router solicitation, length 16
          source link-address option (1), length 8 (1): aa:c1:ab:1f:f4:e3
            0x0000:  aac1 ab1f f4e3
00:28:35.047225 aa:c1:ab:f6:ab:c2 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Ethernet (len 6), IPv4 (len 4), Request who-has 192.168.1.10 tell 192.168.1.20, length 28
00:28:35.047274 aa:c1:ab:16:f2:0c > aa:c1:ab:f6:ab:c2, ethertype ARP (0x0806), length 42: Ethernet (len 6), IPv4 (len 4), Reply 192.168.1.10 is-at aa:c1:ab:16:f2:0c, length 28
00:28:35.047337 aa:c1:ab:f6:ab:c2 > aa:c1:ab:16:f2:0c, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 64, id 30152, offset 0, flags [DF], proto ICMP (1), length 84)
    192.168.1.20 > 192.168.1.10: ICMP echo request, id 36, seq 0, length 64
00:28:35.047413 aa:c1:ab:16:f2:0c > aa:c1:ab:f6:ab:c2, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 64, id 63422, offset 0, flags [none], proto ICMP (1), length 84)
    192.168.1.10 > 192.168.1.20: ICMP echo reply, id 36, seq 0, length 64
00:28:36.053377 aa:c1:ab:f6:ab:c2 > aa:c1:ab:16:f2:0c, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 64, id 30799, offset 0, flags [DF], proto ICMP (1), length 84)
    192.168.1.20 > 192.168.1.10: ICMP echo request, id 36, seq 1, length 64
00:28:36.053404 aa:c1:ab:16:f2:0c > aa:c1:ab:f6:ab:c2, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 64, id 64100, offset 0, flags [none], proto ICMP (1), length 84)
    192.168.1.10 > 192.168.1.20: ICMP echo reply, id 36, seq 1, length 64
00:28:37.055864 aa:c1:ab:f6:ab:c2 > aa:c1:ab:16:f2:0c, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 64, id 31228, offset 0, flags [DF], proto ICMP (1), length 84)
    192.168.1.20 > 192.168.1.10: ICMP echo request, id 36, seq 2, length 64
00:28:37.055897 aa:c1:ab:16:f2:0c > aa:c1:ab:f6:ab:c2, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 64, id 64683, offset 0, flags [none], proto ICMP (1), length 84)
    192.168.1.10 > 192.168.1.20: ICMP echo reply, id 36, seq 2, length 64
10 packets captured
10 packets received by filter
0 packets dropped by kernel
```

#### Содержимое MAC-таблиц:

##### br1:

```bash
/ # brctl showmacs bridge0
port no mac addr                is local?       ageing timer
  1     aa:c1:ab:1f:f4:e3       yes                0.00
  1     aa:c1:ab:1f:f4:e3       yes                0.00
  2     aa:c1:ab:7f:c8:d4       no               255.10
  2     aa:c1:ab:eb:86:38       yes                0.00
  2     aa:c1:ab:eb:86:38       yes                0.00
  2     aa:c1:ab:f6:ab:c2       no                25.73
```

##### br2:

```bash
/ # brctl showmacs bridge0
port no mac addr                is local?       ageing timer
  1     aa:c1:ab:7f:c8:d4       yes                0.00
  1     aa:c1:ab:7f:c8:d4       yes                0.00
  2     aa:c1:ab:a0:8f:6b       yes                0.00
  2     aa:c1:ab:a0:8f:6b       yes                0.00
  2     aa:c1:ab:eb:86:38       no               284.90
  1     aa:c1:ab:f6:ab:c2       no               186.60
```


Как раз видим сначала идет ARP-запрос: host2 (MAC aa:c1:ab:f6:ab:c2) спрашивает у всех: "У кого IP 192.168.1.10?"
Потом получаем ARP-ответ: host1 (MAC aa:c1:ab:16:f2:0c) отвечает host2: "Это у меня, мой MAC такой"
И дальше после того как адреса узнаны идет обычный ICMP-обмен:
host2 отправляет echo request
host1 отвечает echo reply

##Часть 2 Широковещательный шторм

добавили дополнительный линк между бриджами через eth3, STP отключил на обоих бриджах

Сделал пинг

```bash
/ # ping 192.168.1.20 -c 10
PING 192.168.1.20 (192.168.1.20): 56 data bytes

--- 192.168.1.20 ping statistics ---
10 packets transmitted, 0 packets received, 100% packet loss
```
А в tcpdump:

```bash
/ # tcpdump -i eth1 -enn -vvv -c 50
tcpdump: listening on eth1, link-type EN10MB (Ethernet), snapshot length 262144 bytes
07:43:06.619440 aa:c1:ab:57:77:1c > 33:33:00:00:00:16, ethertype IPv6 (0x86dd), length 110: (hlim 1, next-header Options (0) payload length: 56) fe80::a8c1:abff:fe57:771c > ff02::16: HBH (rtalert: 0x0000) (padn) [icmp6 sum ok] ICMP6, multicast listener report v2, 2 group record(s) [gaddr ff02::1:ff57:771c to_ex { }] [gaddr ff02::6a to_ex { }]
07:43:06.619440 aa:c1:ab:57:77:1c > 33:33:00:00:00:02, ethertype IPv6 (0x86dd), length 70: (hlim 255, next-header ICMPv6 (58) payload length: 16) fe80::a8c1:abff:fe57:771c > ff02::2: [icmp6 sum ok] ICMP6, router solicitation, length 16
          source link-address option (1), length 8 (1): aa:c1:ab:57:77:1c
            0x0000:  aac1 ab57 771c
07:43:06.619440 aa:c1:ab:57:77:1c > 01:00:5e:00:00:16, ethertype IPv4 (0x0800), length 54: (tos 0xc0, ttl 1, id 0, offset 0, flags [DF], proto IGMP (2), length 40, options (RA))
    0.0.0.0 > 224.0.0.22: igmp v3 report, 1 group record(s) [gaddr 224.0.0.106 to_ex { }]
07:43:06.619441 aa:c1:ab:57:77:1c > 33:33:00:00:00:02, ethertype IPv6 (0x86dd), length 70: (hlim 255, next-header ICMPv6 (58) payload length: 16) fe80::a8c1:abff:fe57:771c > ff02::2: [icmp6 sum ok] ICMP6, router solicitation, length 16
          source link-address option (1), length 8 (1): aa:c1:ab:57:77:1c
            0x0000:  aac1 ab57 771c
07:43:06.619441 aa:c1:ab:57:77:1c > 33:33:00:00:00:16, ethertype IPv6 (0x86dd), length 110: (hlim 1, next-header Options (0) payload length: 56) fe80::a8c1:abff:fe57:771c > ff02::16: HBH (rtalert: 0x0000) (padn) [icmp6 sum ok] ICMP6, multicast listener report v2, 2 group record(s) [gaddr ff02::1:ff57:771c to_ex { }] [gaddr ff02::6a to_ex { }]
07:43:06.619442 aa:c1:ab:57:77:1c > 33:33:00:00:00:02, ethertype IPv6 (0x86dd), length 70: (hlim 255, next-header ICMPv6 (58) payload length: 16) fe80::a8c1:abff:fe57:771c > ff02::2: [icmp6 sum ok] ICMP6, router solicitation, length 16
          source link-address option (1), length 8 (1): aa:c1:ab:57:77:1c
            0x0000:  aac1 ab57 771c
07:43:06.619443 aa:c1:ab:57:77:1c > 33:33:00:00:00:16, ethertype IPv6 (0x86dd), length 110: (hlim 1, next-header Options (0) payload length: 56) fe80::a8c1:abff:fe57:771c > ff02::16: HBH (rtalert: 0x0000) (padn) [icmp6 sum ok] ICMP6, multicast listener report v2, 2 group record(s) [gaddr ff02::1:ff57:771c to_ex { }] [gaddr ff02::6a to_ex { }]
07:43:06.619443 aa:c1:ab:57:77:1c > 33:33:00:00:00:02, ethertype IPv6 (0x86dd), length 70: (hlim 255, next-header ICMPv6 (58) payload length: 16) fe80::a8c1:abff:fe57:771c > ff02::2: [icmp6 sum ok] ICMP6, router solicitation, length 16
          source link-address option (1), length 8 (1): aa:c1:ab:57:77:1c
            0x0000:  aac1 ab57 771c
07:43:06.619444 aa:c1:ab:57:77:1c > 33:33:00:00:00:02, ethertype IPv6 (0x86dd), length 70: (hlim 255, next-header ICMPv6 (58) payload length: 16) fe80::a8c1:abff:fe57:771c > ff02::2: [icmp6 sum ok] ICMP6, router solicitation, length 16
          source link-address option (1), length 8 (1): aa:c1:ab:57:77:1c
            0x0000:  aac1 ab57 771c
07:43:06.619445 aa:c1:ab:57:77:1c > 33:33:00:00:00:16, ethertype IPv6 (0x86dd), length 110: (hlim 1, next-header Options (0) payload length: 56) fe80::a8c1:abff:fe57:771c > ff02::16: HBH (rtalert: 0x0000) (padn) [icmp6 sum ok] ICMP6, multicast listener report v2, 2 group record(s) [gaddr ff02::1:ff57:771c to_ex { }] [gaddr ff02::6a to_ex { }]
07:43:06.619445 aa:c1:ab:57:77:1c > 33:33:00:00:00:02, ethertype IPv6 (0x86dd), length 70: (hlim 255, next-header ICMPv6 (58) payload length: 16) fe80::a8c1:abff:fe57:771c > ff02::2: [icmp6 sum ok] ICMP6, router solicitation, length 16
          source link-address option (1), length 8 (1): aa:c1:ab:57:77:1c
            0x0000:  aac1 ab57 771c
07:43:06.619446 aa:c1:ab:57:77:1c > 01:00:5e:00:00:16, ethertype IPv4 (0x0800), length 54: (tos 0xc0, ttl 1, id 0, offset 0, flags [DF], proto IGMP (2), length 40, options (RA))
    0.0.0.0 > 224.0.0.22: igmp v3 report, 1 group record(s) [gaddr 224.0.0.106 to_ex { }]
07:43:06.619446 aa:c1:ab:57:77:1c > 33:33:00:00:00:16, ethertype IPv6 (0x86dd), length 110: (hlim 1, next-header Options (0) payload length: 56) fe80::a8c1:abff:fe57:771c > ff02::16: HBH (rtalert: 0x0000) (padn) [icmp6 sum ok] ICMP6, multicast listener report v2, 2 group record(s) [gaddr ff02::1:ff57:771c to_ex { }] [gaddr ff02::6a to_ex { }]
07:43:06.619447 aa:c1:ab:1e:1e:35 > 33:33:00:00:00:02, ethertype IPv6 (0x86dd), length 70: (hlim 255, next-header ICMPv6 (58) payload length: 16) fe80::a8c1:abff:fe1e:1e35 > ff02::2: [icmp6 sum ok] ICMP6, router solicitation, length 16
          source link-address option (1), length 8 (1): aa:c1:ab:1e:1e:35
            0x0000:  aac1 ab1e 1e35
07:43:06.619447 aa:c1:ab:21:68:a7 > 33:33:00:00:00:02, ethertype IPv6 (0x86dd), length 70: (hlim 255, next-header ICMPv6 (58) payload length: 16) fe80::a8c1:abff:fe21:68a7 > ff02::2: [icmp6 sum ok] ICMP6, router solicitation, length 16
          source link-address option (1), length 8 (1): aa:c1:ab:21:68:a7
            0x0000:  aac1 ab21 68a7
07:43:06.619447 aa:c1:ab:57:77:1c > 33:33:00:00:00:16, ethertype IPv6 (0x86dd), length 110: (hlim 1, next-header Options (0) payload length: 56) fe80::a8c1:abff:fe57:771c > ff02::16: HBH (rtalert: 0x0000) (padn) [icmp6 sum ok] ICMP6, multicast listener report v2, 2 group record(s) [gaddr ff02::1:ff57:771c to_ex { }] [gaddr ff02::6a to_ex { }]
07:43:06.619448 aa:c1:ab:1e:1e:35 > 33:33:00:00:00:02, ethertype IPv6 (0x86dd), length 70: (hlim 255, next-header ICMPv6 (58) payload length: 16) fe80::a8c1:abff:fe1e:1e35 > ff02::2: [icmp6 sum ok] ICMP6, router solicitation, length 16
          source link-address option (1), length 8 (1): aa:c1:ab:1e:1e:35
            0x0000:  aac1 ab1e 1e35
07:43:06.619448 aa:c1:ab:21:68:a7 > 33:33:00:00:00:02, ethertype IPv6 (0x86dd), length 70: (hlim 255, next-header ICMPv6 (58) payload length: 16) fe80::a8c1:abff:fe21:68a7 > ff02::2: [icmp6 sum ok] ICMP6, router solicitation, length 16
          source link-address option (1), length 8 (1): aa:c1:ab:21:68:a7
            0x0000:  aac1 ab21 68a7
07:43:06.619448 aa:c1:ab:57:77:1c > 33:33:00:00:00:02, ethertype IPv6 (0x86dd), length 70: (hlim 255, next-header ICMPv6 (58) payload length: 16) fe80::a8c1:abff:fe57:771c > ff02::2: [icmp6 sum ok] ICMP6, router solicitation, length 16
          source link-address option (1), length 8 (1): aa:c1:ab:57:77:1c
            0x0000:  aac1 ab57 771c
07:43:06.619449 aa:c1:ab:57:77:1c > 33:33:00:00:00:16, ethertype IPv6 (0x86dd), length 110: (hlim 1, next-header Options (0) payload length: 56) :: > ff02::16: HBH (rtalert: 0x0000) (padn) [icmp6 sum ok] ICMP6, multicast listener report v2, 2 group record(s) [gaddr ff02::1:ff57:771c to_ex { }] [gaddr ff02::6a to_ex { }]
07:43:06.619450 aa:c1:ab:57:77:1c > 33:33:00:00:00:16, ethertype IPv6 (0x86dd), length 110: (hlim 1, next-header Options (0) payload length: 56) fe80::a8c1:abff:fe57:771c > ff02::16: HBH (rtalert: 0x0000) (padn) [icmp6 sum ok] ICMP6, multicast listener report v2, 2 group record(s) [gaddr ff02::1:ff57:771c to_ex { }] [gaddr ff02::6a to_ex { }]
07:43:06.619450 aa:c1:ab:57:77:1c > 33:33:00:00:00:02, ethertype IPv6 (0x86dd), length 70: (hlim 255, next-header ICMPv6 (58) payload length: 16) fe80::a8c1:abff:fe57:771c > ff02::2: [icmp6 sum ok] ICMP6, router solicitation, length 16
          source link-address option (1), length 8 (1): aa:c1:ab:57:77:1c
            0x0000:  aac1 ab57 771c
07:43:06.619450 aa:c1:ab:57:77:1c > 01:00:5e:00:00:16, ethertype IPv4 (0x0800), length 54: (tos 0xc0, ttl 1, id 0, offset 0, flags [DF], proto IGMP (2), length 40, options (RA))
    0.0.0.0 > 224.0.0.22: igmp v3 report, 1 group record(s) [gaddr 224.0.0.106 to_ex { }]
07:43:06.619451 aa:c1:ab:57:77:1c > 33:33:00:00:00:02, ethertype IPv6 (0x86dd), length 70: (hlim 255, next-header ICMPv6 (58) payload length: 16) fe80::a8c1:abff:fe57:771c > ff02::2: [icmp6 sum ok] ICMP6, router solicitation, length 16
          source link-address option (1), length 8 (1): aa:c1:ab:57:77:1c
            0x0000:  aac1 ab57 771c
07:43:06.619451 aa:c1:ab:57:77:1c > 33:33:00:00:00:16, ethertype IPv6 (0x86dd), length 110: (hlim 1, next-header Options (0) payload length: 56) fe80::a8c1:abff:fe57:771c > ff02::16: HBH (rtalert: 0x0000) (padn) [icmp6 sum ok] ICMP6, multicast listener report v2, 2 group record(s) [gaddr ff02::1:ff57:771c to_ex { }] [gaddr ff02::6a to_ex { }]
07:43:06.619451 aa:c1:ab:1e:1e:35 > 33:33:00:00:00:02, ethertype IPv6 (0x86dd), length 70: (hlim 255, next-header ICMPv6 (58) payload length: 16) fe80::a8c1:abff:fe1e:1e35 > ff02::2: [icmp6 sum ok] ICMP6, router solicitation, length 16
          source link-address option (1), length 8 (1): aa:c1:ab:1e:1e:35
            0x0000:  aac1 ab1e 1e35
07:43:06.619452 aa:c1:ab:8d:7f:3b > 33:33:00:00:00:02, ethertype IPv6 (0x86dd), length 70: (hlim 255, next-header ICMPv6 (58) payload length: 16) fe80::a8c1:abff:fe8d:7f3b > ff02::2: [icmp6 sum ok] ICMP6, router solicitation, length 16
          source link-address option (1), length 8 (1): aa:c1:ab:8d:7f:3b
            0x0000:  aac1 ab8d 7f3b
07:43:06.619452 aa:c1:ab:57:77:1c > 33:33:00:00:00:16, ethertype IPv6 (0x86dd), length 110: (hlim 1, next-header Options (0) payload length: 56) fe80::a8c1:abff:fe57:771c > ff02::16: HBH (rtalert: 0x0000) (padn) [icmp6 sum ok] ICMP6, multicast listener report v2, 2 group record(s) [gaddr ff02::1:ff57:771c to_ex { }] [gaddr ff02::6a to_ex { }]
07:43:06.619453 aa:c1:ab:1e:1e:35 > 33:33:00:00:00:02, ethertype IPv6 (0x86dd), length 70: (hlim 255, next-header ICMPv6 (58) payload length: 16) fe80::a8c1:abff:fe1e:1e35 > ff02::2: [icmp6 sum ok] ICMP6, router solicitation, length 16
          source link-address option (1), length 8 (1): aa:c1:ab:1e:1e:35
            0x0000:  aac1 ab1e 1e35
07:43:06.619453 aa:c1:ab:57:77:1c > 33:33:00:00:00:02, ethertype IPv6 (0x86dd), length 70: (hlim 255, next-header ICMPv6 (58) payload length: 16) fe80::a8c1:abff:fe57:771c > ff02::2: [icmp6 sum ok] ICMP6, router solicitation, length 16
          source link-address option (1), length 8 (1): aa:c1:ab:57:77:1c
            0x0000:  aac1 ab57 771c
07:43:06.619453 aa:c1:ab:57:77:1c > 01:00:5e:00:00:16, ethertype IPv4 (0x0800), length 54: (tos 0xc0, ttl 1, id 0, offset 0, flags [DF], proto IGMP (2), length 40, options (RA))
    0.0.0.0 > 224.0.0.22: igmp v3 report, 1 group record(s) [gaddr 224.0.0.106 to_ex { }]
07:43:06.619454 aa:c1:ab:57:77:1c > 33:33:00:00:00:16, ethertype IPv6 (0x86dd), length 110: (hlim 1, next-header Options (0) payload length: 56) fe80::a8c1:abff:fe57:771c > ff02::16: HBH (rtalert: 0x0000) (padn) [icmp6 sum ok] ICMP6, multicast listener report v2, 2 group record(s) [gaddr ff02::1:ff57:771c to_ex { }] [gaddr ff02::6a to_ex { }]
07:43:06.619454 aa:c1:ab:57:77:1c > 33:33:00:00:00:16, ethertype IPv6 (0x86dd), length 110: (hlim 1, next-header Options (0) payload length: 56) :: > ff02::16: HBH (rtalert: 0x0000) (padn) [icmp6 sum ok] ICMP6, multicast listener report v2, 2 group record(s) [gaddr ff02::1:ff57:771c to_ex { }] [gaddr ff02::6a to_ex { }]
07:43:06.619455 aa:c1:ab:57:77:1c > 33:33:00:00:00:16, ethertype IPv6 (0x86dd), length 110: (hlim 1, next-header Options (0) payload length: 56) fe80::a8c1:abff:fe57:771c > ff02::16: HBH (rtalert: 0x0000) (padn) [icmp6 sum ok] ICMP6, multicast listener report v2, 2 group record(s) [gaddr ff02::1:ff57:771c to_ex { }] [gaddr ff02::6a to_ex { }]
07:43:06.619455 aa:c1:ab:57:77:1c > 33:33:00:00:00:16, ethertype IPv6 (0x86dd), length 110: (hlim 1, next-header Options (0) payload length: 56) :: > ff02::16: HBH (rtalert: 0x0000) (padn) [icmp6 sum ok] ICMP6, multicast listener report v2, 2 group record(s) [gaddr ff02::1:ff57:771c to_ex { }] [gaddr ff02::6a to_ex { }]
07:43:06.619456 aa:c1:ab:57:77:1c > 33:33:00:00:00:16, ethertype IPv6 (0x86dd), length 110: (hlim 1, next-header Options (0) payload length: 56) fe80::a8c1:abff:fe57:771c > ff02::16: HBH (rtalert: 0x0000) (padn) [icmp6 sum ok] ICMP6, multicast listener report v2, 2 group record(s) [gaddr ff02::1:ff57:771c to_ex { }] [gaddr ff02::6a to_ex { }]
07:43:06.619456 aa:c1:ab:57:77:1c > 33:33:00:00:00:16, ethertype IPv6 (0x86dd), length 110: (hlim 1, next-header Options (0) payload length: 56) fe80::a8c1:abff:fe57:771c > ff02::16: HBH (rtalert: 0x0000) (padn) [icmp6 sum ok] ICMP6, multicast listener report v2, 2 group record(s) [gaddr ff02::1:ff57:771c to_ex { }] [gaddr ff02::6a to_ex { }]
07:43:06.619456 aa:c1:ab:57:77:1c > 33:33:00:00:00:16, ethertype IPv6 (0x86dd), length 110: (hlim 1, next-header Options (0) payload length: 56) :: > ff02::16: HBH (rtalert: 0x0000) (padn) [icmp6 sum ok] ICMP6, multicast listener report v2, 2 group record(s) [gaddr ff02::1:ff57:771c to_ex { }] [gaddr ff02::6a to_ex { }]
07:43:06.619457 aa:c1:ab:57:77:1c > 33:33:00:00:00:16, ethertype IPv6 (0x86dd), length 110: (hlim 1, next-header Options (0) payload length: 56) fe80::a8c1:abff:fe57:771c > ff02::16: HBH (rtalert: 0x0000) (padn) [icmp6 sum ok] ICMP6, multicast listener report v2, 2 group record(s) [gaddr ff02::1:ff57:771c to_ex { }] [gaddr ff02::6a to_ex { }]
07:43:06.619458 aa:c1:ab:57:77:1c > 33:33:00:00:00:16, ethertype IPv6 (0x86dd), length 110: (hlim 1, next-header Options (0) payload length: 56) fe80::a8c1:abff:fe57:771c > ff02::16: HBH (rtalert: 0x0000) (padn) [icmp6 sum ok] ICMP6, multicast listener report v2, 2 group record(s) [gaddr ff02::1:ff57:771c to_ex { }] [gaddr ff02::6a to_ex { }]
07:43:06.619459 aa:c1:ab:93:dd:fd > 33:33:00:00:00:02, ethertype IPv6 (0x86dd), length 70: (hlim 255, next-header ICMPv6 (58) payload length: 16) fe80::a8c1:abff:fe93:ddfd > ff02::2: [icmp6 sum ok] ICMP6, router solicitation, length 16
          source link-address option (1), length 8 (1): aa:c1:ab:93:dd:fd
            0x0000:  aac1 ab93 ddfd
07:43:06.619460 aa:c1:ab:57:77:1c > 01:00:5e:00:00:16, ethertype IPv4 (0x0800), length 54: (tos 0xc0, ttl 1, id 0, offset 0, flags [DF], proto IGMP (2), length 40, options (RA))
    0.0.0.0 > 224.0.0.22: igmp v3 report, 1 group record(s) [gaddr 224.0.0.106 to_ex { }]
07:43:06.619460 aa:c1:ab:57:77:1c > 33:33:00:00:00:16, ethertype IPv6 (0x86dd), length 110: (hlim 1, next-header Options (0) payload length: 56) fe80::a8c1:abff:fe57:771c > ff02::16: HBH (rtalert: 0x0000) (padn) [icmp6 sum ok] ICMP6, multicast listener report v2, 2 group record(s) [gaddr ff02::1:ff57:771c to_ex { }] [gaddr ff02::6a to_ex { }]
07:43:06.619460 aa:c1:ab:57:77:1c > 33:33:00:00:00:16, ethertype IPv6 (0x86dd), length 110: (hlim 1, next-header Options (0) payload length: 56) fe80::a8c1:abff:fe57:771c > ff02::16: HBH (rtalert: 0x0000) (padn) [icmp6 sum ok] ICMP6, multicast listener report v2, 2 group record(s) [gaddr ff02::1:ff57:771c to_ex { }] [gaddr ff02::6a to_ex { }]
07:43:06.619462 aa:c1:ab:57:77:1c > 33:33:00:00:00:16, ethertype IPv6 (0x86dd), length 110: (hlim 1, next-header Options (0) payload length: 56) fe80::a8c1:abff:fe57:771c > ff02::16: HBH (rtalert: 0x0000) (padn) [icmp6 sum ok] ICMP6, multicast listener report v2, 2 group record(s) [gaddr ff02::1:ff57:771c to_ex { }] [gaddr ff02::6a to_ex { }]
07:43:06.619464 aa:c1:ab:57:77:1c > 33:33:00:00:00:16, ethertype IPv6 (0x86dd), length 110: (hlim 1, next-header Options (0) payload length: 56) fe80::a8c1:abff:fe57:771c > ff02::16: HBH (rtalert: 0x0000) (padn) [icmp6 sum ok] ICMP6, multicast listener report v2, 2 group record(s) [gaddr ff02::1:ff57:771c to_ex { }] [gaddr ff02::6a to_ex { }]
07:43:06.619464 aa:c1:ab:1c:06:7d > 33:33:00:00:00:02, ethertype IPv6 (0x86dd), length 70: (hlim 255, next-header ICMPv6 (58) payload length: 16) fe80::a8c1:abff:fe1c:67d > ff02::2: [icmp6 sum ok] ICMP6, router solicitation, length 16
          source link-address option (1), length 8 (1): aa:c1:ab:1c:06:7d
            0x0000:  aac1 ab1c 067d
07:43:06.619465 aa:c1:ab:57:77:1c > 33:33:00:00:00:16, ethertype IPv6 (0x86dd), length 110: (hlim 1, next-header Options (0) payload length: 56) fe80::a8c1:abff:fe57:771c > ff02::16: HBH (rtalert: 0x0000) (padn) [icmp6 sum ok] ICMP6, multicast listener report v2, 2 group record(s) [gaddr ff02::1:ff57:771c to_ex { }] [gaddr ff02::6a to_ex { }]
07:43:06.619466 aa:c1:ab:57:77:1c > 01:00:5e:00:00:16, ethertype IPv4 (0x0800), length 54: (tos 0xc0, ttl 1, id 0, offset 0, flags [DF], proto IGMP (2), length 40, options (RA))
    0.0.0.0 > 224.0.0.22: igmp v3 report, 1 group record(s) [gaddr 224.0.0.106 to_ex { }]
07:43:06.619467 aa:c1:ab:57:77:1c > 33:33:00:00:00:16, ethertype IPv6 (0x86dd), length 110: (hlim 1, next-header Options (0) payload length: 56) fe80::a8c1:abff:fe57:771c > ff02::16: HBH (rtalert: 0x0000) (padn) [icmp6 sum ok] ICMP6, multicast listener report v2, 2 group record(s) [gaddr ff02::1:ff57:771c to_ex { }] [gaddr ff02::6a to_ex { }]
50 packets captured
3152 packets received by filter
0 packets dropped by kernel
```

Получилось что? очень нагрелся ноут(боялся даже что что-то пойдет не так)
Видим, что не пингуется.
В выдержке видим многократное повторение одних и тех же пакетов, MAC-адреса источников повторяются, 
пакеты multicast (IPv6 и IGMP) зациклились.

Почему?
Ну мы зациклили, теперь в сети образовалась петля из-за второго соединения между бриджами
и теперь пакеты бесконечно циркулируют:

br1 --(eth2)--> br2 --(eth3)--> br1 --(eth2)--> br2 --(eth3)--> br1 ...
И STP был отключен, поэтому петля не была заблокирована и любой широковещательный пакет начал бесконечно циркулировать, вот и шторм.

##Часть 3 Магия STP
Включил на обоих бриджах STP

сделал пинг

```bash
/ # ping 192.168.1.20 -c 10
PING 192.168.1.20 (192.168.1.20): 56 data bytes
64 bytes from 192.168.1.20: seq=0 ttl=64 time=0.120 ms
64 bytes from 192.168.1.20: seq=1 ttl=64 time=0.082 ms
64 bytes from 192.168.1.20: seq=2 ttl=64 time=0.127 ms
64 bytes from 192.168.1.20: seq=3 ttl=64 time=0.068 ms
64 bytes from 192.168.1.20: seq=4 ttl=64 time=0.146 ms
64 bytes from 192.168.1.20: seq=5 ttl=64 time=0.154 ms
64 bytes from 192.168.1.20: seq=6 ttl=64 time=0.151 ms
64 bytes from 192.168.1.20: seq=7 ttl=64 time=0.131 ms
64 bytes from 192.168.1.20: seq=8 ttl=64 time=0.132 ms
64 bytes from 192.168.1.20: seq=9 ttl=64 time=0.136 ms

--- 192.168.1.20 ping statistics ---
10 packets transmitted, 10 packets received, 0% packet loss
round-trip min/avg/max = 0.068/0.124/0.154 ms
```
В выдержке:

```bash
/ # tcpdump -i eth1 -enn -c 10 -vvv
tcpdump: listening on eth1, link-type EN10MB (Ethernet), snapshot length 262144 bytes
07:48:18.074796 aa:c1:ab:73:a9:4f > 01:80:c2:00:00:00, 802.3, length 38: LLC, dsap STP (0x42) Individual, ssap STP (0x42) Command, ctrl 0x03: STP 802.1d, Config, Flags [none], bridge-id 8000.aa:c1:ab:1e:1e:35.8001, length 35
        message-age 0.00s, max-age 20.00s, hello-time 2.00s, forwarding-delay 15.00s
        root-id 8000.aa:c1:ab:1e:1e:35, root-pathcost 0
07:48:20.060649 aa:c1:ab:73:a9:4f > 01:80:c2:00:00:00, 802.3, length 38: LLC, dsap STP (0x42) Individual, ssap STP (0x42) Command, ctrl 0x03: STP 802.1d, Config, Flags [none], bridge-id 8000.aa:c1:ab:1e:1e:35.8001, length 35
        message-age 0.00s, max-age 20.00s, hello-time 2.00s, forwarding-delay 15.00s
        root-id 8000.aa:c1:ab:1e:1e:35, root-pathcost 0
07:48:22.042509 aa:c1:ab:73:a9:4f > 01:80:c2:00:00:00, 802.3, length 38: LLC, dsap STP (0x42) Individual, ssap STP (0x42) Command, ctrl 0x03: STP 802.1d, Config, Flags [none], bridge-id 8000.aa:c1:ab:1e:1e:35.8001, length 35
        message-age 0.00s, max-age 20.00s, hello-time 2.00s, forwarding-delay 15.00s
        root-id 8000.aa:c1:ab:1e:1e:35, root-pathcost 0
07:48:24.030623 aa:c1:ab:73:a9:4f > 01:80:c2:00:00:00, 802.3, length 38: LLC, dsap STP (0x42) Individual, ssap STP (0x42) Command, ctrl 0x03: STP 802.1d, Config, Flags [none], bridge-id 8000.aa:c1:ab:1e:1e:35.8001, length 35
        message-age 0.00s, max-age 20.00s, hello-time 2.00s, forwarding-delay 15.00s
        root-id 8000.aa:c1:ab:1e:1e:35, root-pathcost 0
07:48:26.075052 aa:c1:ab:73:a9:4f > 01:80:c2:00:00:00, 802.3, length 38: LLC, dsap STP (0x42) Individual, ssap STP (0x42) Command, ctrl 0x03: STP 802.1d, Config, Flags [none], bridge-id 8000.aa:c1:ab:1e:1e:35.8001, length 35
        message-age 0.00s, max-age 20.00s, hello-time 2.00s, forwarding-delay 15.00s
        root-id 8000.aa:c1:ab:1e:1e:35, root-pathcost 0
07:48:27.274667 aa:c1:ab:21:68:a7 > aa:c1:ab:1c:06:7d, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 64, id 41032, offset 0, flags [DF], proto ICMP (1), length 84)
    192.168.1.10 > 192.168.1.20: ICMP echo request, id 29, seq 0, length 64
07:48:27.274733 aa:c1:ab:1c:06:7d > aa:c1:ab:21:68:a7, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 64, id 50354, offset 0, flags [none], proto ICMP (1), length 84)
    192.168.1.20 > 192.168.1.10: ICMP echo reply, id 29, seq 0, length 64
07:48:28.060402 aa:c1:ab:73:a9:4f > 01:80:c2:00:00:00, 802.3, length 38: LLC, dsap STP (0x42) Individual, ssap STP (0x42) Command, ctrl 0x03: STP 802.1d, Config, Flags [none], bridge-id 8000.aa:c1:ab:1e:1e:35.8001, length 35
        message-age 0.00s, max-age 20.00s, hello-time 2.00s, forwarding-delay 15.00s
        root-id 8000.aa:c1:ab:1e:1e:35, root-pathcost 0
07:48:28.275482 aa:c1:ab:21:68:a7 > aa:c1:ab:1c:06:7d, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 64, id 41095, offset 0, flags [DF], proto ICMP (1), length 84)
    192.168.1.10 > 192.168.1.20: ICMP echo request, id 29, seq 1, length 64
07:48:28.275518 aa:c1:ab:1c:06:7d > aa:c1:ab:21:68:a7, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 64, id 51212, offset 0, flags [none], proto ICMP (1), length 84)
    192.168.1.20 > 192.168.1.10: ICMP echo reply, id 29, seq 1, length 64
10 packets captured
10 packets received by filter
0 packets dropped by kernel
```

сеть исцелилась.

На 1 бридже:

```bash
/ # brctl showstp bridge0
bridge0
 bridge id              8000.aac1ab1e1e35
 designated root        8000.aac1ab1e1e35
 root port                 0                    path cost                  0
 max age                  20.00                 bridge max age            20.00
 hello time                2.00                 bridge hello time          2.00
 forward delay            15.00                 bridge forward delay      15.00
 ageing time             300.00
 hello timer               0.00                 tcn timer                  0.00
 topology change timer     0.00                 gc timer                  50.51
 flags


eth1 (1)
 port id                8001                    state                forwarding
 designated root        8000.aac1ab1e1e35       path cost                  2
 designated bridge      8000.aac1ab1e1e35       message age timer          0.00
 designated port        8001                    forward delay timer        0.00
 designated cost           0                    hold timer                 0.00
 flags

eth2 (2)
 port id                8002                    state                forwarding
 designated root        8000.aac1ab1e1e35       path cost                  2
 designated bridge      8000.aac1ab1e1e35       message age timer          0.00
 designated port        8002                    forward delay timer        0.00
 designated cost           0                    hold timer                 0.00
 flags

eth3 (3)
 port id                8003                    state                forwarding
 designated root        8000.aac1ab1e1e35       path cost                  2
 designated bridge      8000.aac1ab1e1e35       message age timer          0.00
 designated port        8003                    forward delay timer        0.00
 designated cost           0                    hold timer                 0.00
 flags
```

На 2:

```bash
/ # brctl showstp bridge0
bridge0
 bridge id              8000.aac1ab57771c
 designated root        8000.aac1ab1e1e35
 root port                 2                    path cost                  2
 max age                  20.00                 bridge max age            20.00
 hello time                2.00                 bridge hello time          2.00
 forward delay            15.00                 bridge forward delay      15.00
 ageing time             300.00
 hello timer               0.00                 tcn timer                  0.00
 topology change timer     0.00                 gc timer                 121.85
 flags


eth1 (1)
 port id                8001                    state                forwarding
 designated root        8000.aac1ab1e1e35       path cost                  2
 designated bridge      8000.aac1ab57771c       message age timer          0.00
 designated port        8001                    forward delay timer        0.00
 designated cost           2                    hold timer                 0.79
 flags

eth2 (2)
 port id                8002                    state                forwarding
 designated root        8000.aac1ab1e1e35       path cost                  2
 designated bridge      8000.aac1ab1e1e35       message age timer         19.84
 designated port        8002                    forward delay timer        0.00
 designated cost           0                    hold timer                 0.00
 flags

eth3 (3)
 port id                8003                    state                  blocking
 designated root        8000.aac1ab1e1e35       path cost                  2
 designated bridge      8000.aac1ab1e1e35       message age timer         19.84
 designated port        8003                    forward delay timer        0.00
 designated cost           0                    hold timer                 0.00
 flags
```

бридж с MAC aa:c1:ab:1e:1e:35 является корневым, потому что bridge id = 8000.aac1ab1e1e35 — это ID текущего бриджа и
designated root = 8000.aac1ab1e1e35 — корневой бридж совпадает с текущим
выбор произошел по наименьшему MAC-адресу

Какой порт был заблокирован на обоих бриджах?
На первом бридже — ни один порт не заблокирован!
Чекаем:
eth1 (1) state = forwarding — порт 1 пересылает трафик

eth2 (2) state = forwarding — порт 2 пересылает трафик

eth3 (3) state = forwarding — порт 3 пересылает трафик

Это норм для корневого бриджа.
А вот на втором заблокан eth3 
Это один из двух линков между бриджами, теперь понятно что произошло вообще.
STP оставил один линк (eth2) в forwarding, а второй (eth3) заблокировал
Это разрывает петлю, но сохраняет связность
трафик теперь от host2 к host1 идет:
host2 → br2:eth1 → br2:eth2 → br1:eth2 → br1:eth1 → host1
И петли нет и шторма нет
