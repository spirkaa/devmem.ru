---
title: "WireGuard VPN туннель между двумя серверами на Debian"
aliases:
- /2019/06/14/wireguard-vpn-tunnel-mezhdu-dvumya-serverami-na-debian/
date: 2019-06-14T08:29:35+00:00
toc: true
categories:
  - Network
tags:
  - Debian
  - VPN
  - VPS
  - WireGuard
---

## Задача

Организовать доступ через интернет к серверу, который находится за NAT на неподконтрольном маршрутизаторе с динамическим серым внешним IP, соответственно проброс портов с этого маршрутизатора невозможен. Перефразируя в реальный пример: нужно открывать сайт, размещенный на сервере в здании московской школы, через интернет. Интернет в здании предоставляется по госконтракту, оборудование сети школой не контролируется.

Решить эту задачу можно с помощью проброса портов через VPN-туннель между самым дешевым VPS в интернете с публичным IP-адресом и сервером/маршрутизатором внутри локальной сети. Туннель будет на основе [WireGuard](https://www.wireguard.com/) - простого, современного и быстрого VPN-протокола с минимумом настроек без возни с сертификатами и шифрами (привет, OpenVPN).

В статье используются виртуальные машины на Debian 9:

  1. **wg-lab-01**, 192.168.1.101 - сервер WireGuard «VPS в интернете»
  2. **wg-lab-02**, 192.168.1.102 - клиент WireGuard «роутер за NAT»
  3. **wg-lab-03**, 192.168.1.103 - веб-сервер

Пусть ВМ wg-lab-01 имеет публичный IP-адрес и будет использоваться как сервер Wireguard (к нему будут подключаться клиенты) и маршрутизатор для проброса портов на ВМ wg-lab-02, используемую как клиент Wireguard (будет инициализировать подключение к серверу) и маршрутизатор. С ВМ wg-lab-02 те же порты будут перенаправляться на ВМ wg-lab-03, на которой запущен веб-сервер nginx, а в настройках сетевого подключения IP-адрес ВМ wg-lab-02 указан в качестве шлюза по умолчанию (Default gateway).

Трафик должен проходить следующим образом:

  > посетитель сайта &lt;-&gt; интернет &lt;-&gt; wg-lab-01 &lt;-&gt; [туннель wireguard] &lt;-&gt; wg-lab-02 &lt;-&gt; wg-lab-03

Для упрощения конфигурации, весь трафик wg-lab-02 идет через туннель, а также все ВМ запущены внутри одного VLAN.

## Настройка туннеля и маршрутизации

Все настройки выполняются через SSH от имени пользователя root.

### 1. На wg-lab-01 и wg-lab-02

``` shell
# Установить wireguard
echo "deb http://deb.debian.org/debian/ unstable main" > /etc/apt/sources.list.d/unstable-wireguard.list;
printf 'Package: *\nPin: release a=unstable\nPin-Priority: 90\n' > /etc/apt/preferences.d/limit-unstable;
apt update && apt list --upgradable;
apt full-upgrade;
apt install wireguard tcpdump iptables-save;
# Включить маршрутизацию
sed -i 's/^#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/' /etc/sysctl.conf;
sysctl -p;
reboot;
# Сгенерировать ключи wireguard
(umask 077 && printf "[Interface]\nPrivateKey = " | tee /etc/wireguard/wg0.conf > /dev/null);
wg genkey | tee -a /etc/wireguard/wg0.conf | wg pubkey | tee /etc/wireguard/publickey;
```

### 2. На wg-lab-01

Отредактировать файл конфигурации `nano /etc/wireguard/wg0.conf`

``` ini
[Interface]
Address = 10.255.255.1/24
SaveConfig = true
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
ListenPort = 51820
PrivateKey = В ФАЙЛЕ КЛЮЧ УЖЕ ДОЛЖЕН БЫТЬ

[Peer]
PublicKey = публичный ключ с wg-lab-02 # см. в файле /etc/wireguard/publickey
AllowedIPs = 10.255.255.2/32
PersistentKeepalive = 10
```

### 3. На wg-lab-02

Отредактировать файл конфигурации `nano /etc/wireguard/wg0.conf`

``` ini
[Interface]
Address = 10.255.255.2/24
SaveConfig = true
PostUp = iptables -A FORWARD -i eth0 -j ACCEPT; iptables -t nat -A POSTROUTING -o wg0 -j MASQUERADE
PostDown = iptables -D FORWARD -i eth0 -j ACCEPT; iptables -t nat -D POSTROUTING -o wg0 -j MASQUERADE
PrivateKey = В ФАЙЛЕ КЛЮЧ УЖЕ ДОЛЖЕН БЫТЬ

[Peer]
Endpoint = 192.168.1.101:51820
PublicKey = публичный ключ с wg-lab-01 # см. в файле /etc/wireguard/publickey
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 10
```

### 4. На wg-lab-01 и wg-lab-02

Включить автозапуск интерфейса wg0

``` shell
systemctl enable wg-quick@wg0;
wg-quick up wg0
```

### Проверить, что туннель и маршрутизация работают (на wg-lab-02)

* В выводе команды `wg` присутствует строка _latest handshake_
* Адрес интерфейса wg0 на сервере wg-lab-01 пингуется `ping 10.255.255.1`
* В выводе команды `traceroute ya.ru` первый прыжок на 10.255.255.1
* Если бы настраивали через реальный VPS в интернете, то команда `host myip.opendns.com resolver1.opendns.com` показывала бы IP-адрес VPS, а не текущего подключения к интернету

## Настройка веб-сервера

### На wg-lab-03

1. Установить и запустить веб-сервер nginx командой `apt install nginx`, или, если установлен Docker, командой `docker run –name nginx -d -p 80:80 –restart unless-stopped nginx:alpine`
2. Проверить доступность веб-сервера по текущему IP-адресу, полученному через DHCP (см. `ip a`)
3. Настроить статический IP-адрес и указать шлюз по умолчанию IP сервера wg-lab-02, перезагрузить для применения настроек

## Проброс портов

### На wg-lab-01

``` shell
iptables -t nat -A PREROUTING -d 192.168.1.101/32 -p tcp -m multiport --dports 80,443 -j DNAT --to-destination 10.255.255.2
```

### На wg-lab-02

``` shell
iptables -t nat -A PREROUTING -d 10.255.255.2/32 -p tcp -m multiport --dports 80,443 -j DNAT --to-destination 192.168.1.103
```

Настроенные таким образом правила iptables работают до перезагрузки. Чтобы сделать правила постоянными, нужно либо сохранить их с помощью команды `iptables-save > /etc/iptables/rules.v4`, либо соответствующим образом дополнить строки PostUp и PostDown в файлах /etc/wireguard/wg0.conf на обоих ВМ с Wireguard, чтобы правила применялись при активации интерфейса wg0.

Проверить активные правила можно командой `iptables -t nat -L -n -v`

### Проверить, что проброс портов работает

В браузере открыть IP-адрес wg-lab-01 (192.168.1.101) и убедиться, что открывается страница с веб-сервера на wg-lab-03.

## Ссылки

* <https://angristan.xyz/how-to-setup-vpn-server-wireguard-nat-ipv6/>
* <https://nfalcone.net/blog/wireguard-vpn-debian-server/>
* <https://wiki.debian.org/WireGuard>
* <https://wiki.debian.org/iptables>
* <http://linux-ip.net/html/nat-dnat.html>
* <https://www.revsys.com/writings/quicktips/nat.html>
* <https://wiki.yobi.be/index.php/Wireguard>
* <https://www.ericlight.com/wireguard-part-two-vpn-routing.html>
* <https://www.ckn.io/blog/2017/11/14/wireguard-vpn-typical-setup/>
* <https://bneijt.nl/blog/post/wireguard-vpn-using-an-edgerouter-x/>
* <https://andrew.dunn.dev/posts/wireguard-from-your-isp/>
