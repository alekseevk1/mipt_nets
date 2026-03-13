# ДЗ 2 IPv6 и SLAAC

## Часть 1 Базовая конфигурация
Установил все нужное на хосте2
```bash
/ # apk add radvd tcpdump iproute2
( 1/12) Installing libcap2 (2.77-r0)
( 2/12) Installing zstd-libs (1.5.7-r2)
( 3/12) Installing libelf (0.194-r0)
( 4/12) Installing libmnl (1.0.5-r2)
( 5/12) Installing iproute2-minimal (6.17.0-r0)
( 6/12) Installing libxtables (1.8.11-r1)
( 7/12) Installing iproute2-tc (6.17.0-r0)
( 8/12) Installing iproute2-ss (6.17.0-r0)
( 9/12) Installing iproute2 (6.17.0-r0)
  Executing iproute2-6.17.0-r0.post-install
(10/12) Installing radvd (2.20-r0)
  Executing radvd-2.20-r0.pre-install
(11/12) Installing libpcap (1.10.5-r1)
(12/12) Installing tcpdump (4.99.5-r1)
Executing busybox-1.37.0-r30.trigger
OK: 12.7 MiB in 28 packages
```

### МАС-адреса Host1/2.

На 1 хосте:

```bash
31: eth1@if32: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 9500 qdisc noqueue state UP 
    link/ether aa:c1:ab:40:55:5a brd ff:ff:ff:ff:ff:ff
```

На 2 хосте:
```bash
33: eth1@if34: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9500 qdisc noqueue state UP mode DEFAULT group default 
    link/ether aa:c1:ab:88:c5:0d brd ff:ff:ff:ff:ff:ff link-netnsid 1
```

Создал конфиг 
```bash
interface eth1
> {
>   AdvSendAdvert on;
>   AdvManagedFlag off;
>   AdvOtherConfigFlag off;
>   prefix 2001:db8:1::/64
>   {
>     AdvOnLink on;
>     AdvAutonomous on;
>     AdvValidLifetime 86400;
>     AdvPreferredLifetime 14400;
>   };
> };
> RA_CONFIG
```

и запустил демона для рассылки Router Advertisement.

Делаю захват RA-сообщений:

```bash
/ # tcpdump -i eth1 -enn -c 5 -v 'icmp6 and ip6[40] == 134'
tcpdump: listening on eth1, link-type EN10MB (Ethernet), snapshot length 262144 bytes
19:15:32.477999 aa:c1:ab:88:c5:0d > 33:33:00:00:00:01, ethertype IPv6 (0x86dd), length 110: (flowlabel 0x9d470, hlim 255, next-header ICMPv6 (58) payload length: 56) fe80::a8c1:abff:fe88:c50d > ff02::1: [icmp6 sum ok] ICMP6, router advertisement, length 56
        hop limit 64, Flags [none], pref medium, router lifetime 1800s, reachable time 0ms, retrans timer 0ms
          prefix info option (3), length 32 (4): 2001:db8:1::/64, Flags [onlink, auto], valid time 86400s, pref. time 14400s
          source link-address option (1), length 8 (1): aa:c1:ab:88:c5:0d
```
Что произошло?
С помощью команды tcpdump был захвачен трафик на интерфейсе eth1 маршрутизатора (Host2). Фильтр icmp6 and ip6[40] == 134 отобрал только Router Advertisement (RA) сообщения. 
В выводе видно:

Кто отправил: Host2 с MAC-адресом aa:c1:ab:88:c5:0d

Кому отправил: multicast-адрес 33:33:00:00:00:01 (все IPv6-узлы в сегменте)

Какой префикс раздается: 2001:db8:1::/64

Параметры префикса:

onlink — префикс считаем локальным

auto — разрешена автоматическая конфигурация адресов (SLAAC)

Время жизни: valid 86400s (1 день), preferred 14400s (4 часа)

Идем дальше

### IPv6-адреса на eth1 Host 2/1.

host1:

```bash
31: eth1@if32: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 9500 qdisc noqueue state UP 
    link/ether aa:c1:ab:40:55:5a brd ff:ff:ff:ff:ff:ff
    inet6 2001:db8:1:0:a8c1:abff:fe40:555a/64 scope global dynamic flags 100 
       valid_lft 86003sec preferred_lft 14003sec
    inet6 fe80::a8c1:abff:fe40:555a/64 scope link 
       valid_lft forever preferred_lft forever
```

host2:

```bash
33: eth1@if34: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9500 qdisc noqueue state UP group default 
    link/ether aa:c1:ab:88:c5:0d brd ff:ff:ff:ff:ff:ff link-netnsid 1
    inet6 2001:db8:1::1/64 scope global 
       valid_lft forever preferred_lft forever
    inet6 fe80::a8c1:abff:fe88:c50d/64 scope link proto kernel_ll 
       valid_lft forever preferred_lft forever
```

На обоих хостах есть Global IPv6 адрес 2001 :db8...  и  Link-local адрес fe80::...

### Корректность работы SLLAC алгоритма, проверьте правильно ли посчитался IPv6 адрес для eth1 Host1 (EUI-64).

1. Mac-адрес хоста 1 aa:c1:ab:40:55:5a делим пополам и вставляем между ff:fe -> aa:c1:ab:ff:fe:40:55:5a
2. Инвертируем 7-й бит aa -> a8 - a8:c1:ab:ff:fe:40:55:5a
3. Переводим в IPv6 нотацию: a8c1:abff:fe40:555a
4. Добавляем префикс маршрутизатора 2001:db8:1:: -> 2001:db8:1:0:a8c1:abff:fe40:555a/64

А посмотрим на реальный адрес:
```bash
# global slaac адрес
inet6 2001:db8:1:0:a8c1:abff:fe40:555a/64 scope global dynamic flags 100 
       valid_lft 86003sec preferred_lft 14003sec
```

Они совпали, значит адрес сформирован правильно в соответствии с алгоритмом EUI-64. Это подтверждает, что SLAAC работает корректно.

### Проверка IPv6 связности для двух типов IPv6 адресов (global, LL) между Host 1 и 2.

Попробуем проверить:

от хост1 до хост2

```bash
/ # ping6 -c 3 2001:db8:1::1
PING 2001:db8:1::1 (2001:db8:1::1): 56 data bytes
64 bytes from 2001:db8:1::1: seq=0 ttl=64 time=0.104 ms
64 bytes from 2001:db8:1::1: seq=1 ttl=64 time=0.142 ms
64 bytes from 2001:db8:1::1: seq=2 ttl=64 time=0.148 ms

--- 2001:db8:1::1 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 0.104/0.131/0.148 ms
/ # ping6 -c 3 fe80::a8c1:abff:fe88:c50d%eth1
PING fe80::a8c1:abff:fe88:c50d%eth1 (fe80::a8c1:abff:fe88:c50d%31): 56 data bytes
64 bytes from fe80::a8c1:abff:fe88:c50d: seq=0 ttl=64 time=0.100 ms
64 bytes from fe80::a8c1:abff:fe88:c50d: seq=1 ttl=64 time=0.120 ms
64 bytes from fe80::a8c1:abff:fe88:c50d: seq=2 ttl=64 time=0.101 ms

--- fe80::a8c1:abff:fe88:c50d%eth1 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 0.100/0.107/0.120 ms
```
 Видно, что все пингуется, связь есть

Смотрим наоборот:

```bash
/ # ping6 -c 3 2001:db8:1:0:a8c1:abff:fe40:555a
PING 2001:db8:1:0:a8c1:abff:fe40:555a (2001:db8:1:0:a8c1:abff:fe40:555a): 56 data bytes
64 bytes from 2001:db8:1:0:a8c1:abff:fe40:555a: seq=0 ttl=64 time=0.128 ms
64 bytes from 2001:db8:1:0:a8c1:abff:fe40:555a: seq=1 ttl=64 time=0.167 ms
64 bytes from 2001:db8:1:0:a8c1:abff:fe40:555a: seq=2 ttl=64 time=0.159 ms

--- 2001:db8:1:0:a8c1:abff:fe40:555a ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 0.128/0.151/0.167 ms
/ # ping6 -c 3 fe80::a8c1:abff:fe40:555a%eth1
PING fe80::a8c1:abff:fe40:555a%eth1 (fe80::a8c1:abff:fe40:555a%33): 56 data bytes
64 bytes from fe80::a8c1:abff:fe40:555a: seq=0 ttl=64 time=0.103 ms
64 bytes from fe80::a8c1:abff:fe40:555a: seq=1 ttl=64 time=0.144 ms
64 bytes from fe80::a8c1:abff:fe40:555a: seq=2 ttl=64 time=0.179 ms

--- fe80::a8c1:abff:fe40:555a%eth1 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 0.103/0.142/0.179 ms
```

Аналогично все корректно.

### Содержимое IPv6 маршрутных и NDP-таблиц с обоих хостов.

На хосте 1:
```bash
/ # ip -6 route show
2001:db8:1::/64 dev eth1  metric 256  expires 8591sec
3fff:172:20:20::/64 dev eth0  metric 256 
fe80::/64 dev eth0  metric 256 
fe80::/64 dev eth1  metric 256 
default via 3fff:172:20:20::1 dev eth0  metric 1024 
/ # ip -6 neigh show dev eth1
fe80::a8c1:abff:febd:bb60  used 50/56/50 probes 6 FAILED
fe80::a8c1:abff:fe88:c50d lladdr aa:c1:ab:88:c5:0d used 16/16/13 probes 1 STALE
2001:db8:1::1 lladdr aa:c1:ab:88:c5:0d used 16/16/13 probes 1 STALE
```

На хосте2:
```bash
/ # ip -6 route show
2001:db8:1::/64 dev eth1 proto kernel metric 256 pref medium
3fff:172:20:20::/64 dev eth0 proto kernel metric 256 pref medium
fe80::/64 dev eth0 proto kernel metric 256 pref medium
fe80::/64 dev eth1 proto kernel metric 256 pref medium
default via 3fff:172:20:20::1 dev eth0 metric 1024 pref medium
/ # ip -6 neigh show dev eth1
2001:db8:1:0:a8c1:abff:fe40:555a lladdr aa:c1:ab:40:55:5a STALE 
fe80::a8c1:abff:fe1b:795d lladdr aa:c1:ab:1b:79:5d STALE 
fe80::a8c1:abff:fe44:4e53 FAILED 
fe80::a8c1:abff:fe40:555a lladdr aa:c1:ab:40:55:5a STALE 
```

Что мы видим?
В NDP-таблице Host1 виден маршрутизатор Host2 с MAC-адресом aa:c1:ab:88:c5:0d. В NDP-таблице Host2 виден клиент Host1 с MAC-адресом aa:c1:ab:40:55:5a. Маршрутные таблицы содержат прямой маршрут к сети 2001:db8:1::/64 на обоих хостах. FAILED записи являются устаревшими и не влияют на связность. Все правильно, везде оба адреса привязаны к правильному MAC.


### Попробуйте изменить назначаемый IPv6-адрес на eth1 Host 2 на /127 и проверьте работу SLAAC.

Попробуем изменить в конфиге RADVD префикс с /64 на /127 и проверить, будет ли работать SLAAC.

```bash
/ # tcpdump -i eth1 -enn -c 3 -v 'icmp6 and ip6[40] == 134'
tcpdump: listening on eth1, link-type EN10MB (Ethernet), snapshot length 262144 bytes
20:17:40.846883 aa:c1:ab:88:c5:0d > 33:33:00:00:00:01, ethertype IPv6 (0x86dd), length 110: (flowlabel 0x9d470, hlim 255, next-header ICMPv6 (58) payload length: 56) fe80::a8c1:abff:fe88:c50d > ff02::1: [icmp6 sum ok] ICMP6, router advertisement, length 56
        hop limit 64, Flags [none], pref medium, router lifetime 1800s, reachable time 0ms, retrans timer 0ms
          prefix info option (3), length 32 (4): 2001:db8:1::/127, Flags [onlink, auto], valid time 86400s, pref. time 14400s
          source link-address option (1), length 8 (1): aa:c1:ab:88:c5:0d
20:17:56.867047 aa:c1:ab:88:c5:0d > 33:33:00:00:00:01, ethertype IPv6 (0x86dd), length 110: (flowlabel 0x9d470, hlim 255, next-header ICMPv6 (58) payload length: 56) fe80::a8c1:abff:fe88:c50d > ff02::1: [icmp6 sum ok] ICMP6, router advertisement, length 56
        hop limit 64, Flags [none], pref medium, router lifetime 1800s, reachable time 0ms, retrans timer 0ms
          prefix info option (3), length 32 (4): 2001:db8:1::/127, Flags [onlink, auto], valid time 86400s, pref. time 14400s
          source link-address option (1), length 8 (1): aa:c1:ab:88:c5:0d
exit
20:18:12.889095 aa:c1:ab:88:c5:0d > 33:33:00:00:00:01, ethertype IPv6 (0x86dd), length 110: (flowlabel 0x9d470, hlim 255, next-header ICMPv6 (58) payload length: 56) fe80::a8c1:abff:fe88:c50d > ff02::1: [icmp6 sum ok] ICMP6, router advertisement, length 56
        hop limit 64, Flags [none], pref medium, router lifetime 1800s, reachable time 0ms, retrans timer 0ms
          prefix info option (3), length 32 (4): 2001:db8:1::/127, Flags [onlink, auto], valid time 86400s, pref. time 14400s
          source link-address option (1), length 8 (1): aa:c1:ab:88:c5:0d
3 packets captured
3 packets received by filter
0 packets dropped by kernel
```

А на хосте1:
```bash
/ # ip addr show eth1
31: eth1@if32: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 9500 qdisc noqueue state UP 
    link/ether aa:c1:ab:40:55:5a brd ff:ff:ff:ff:ff:ff
    inet6 2001:db8:1:0:a8c1:abff:fe40:555a/64 scope global dynamic flags 100 
       valid_lft 86346sec preferred_lft 14346sec
    inet6 fe80::a8c1:abff:fe40:555a/64 scope link 
       valid_lft forever preferred_lft forever
/ # ping6 -c 3 2001:db8:1::1
PING 2001:db8:1::1 (2001:db8:1::1): 56 data bytes
64 bytes from 2001:db8:1::1: seq=0 ttl=64 time=0.150 ms
64 bytes from 2001:db8:1::1: seq=1 ttl=64 time=0.140 ms
64 bytes from 2001:db8:1::1: seq=2 ttl=64 time=0.144 ms

--- 2001:db8:1::1 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 0.140/0.144/0.150 ms
```
Адрес остался прежним (с /64)
Что произошло?
SLAAC работает только с префиксом /64. При изменении на /127 клиенты игнорируют RA и не создают новые адреса. Старые адреса продолжают работать до истечения времени жизни
