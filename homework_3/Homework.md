## Задание 1. Анализ состояний TCP-соединений

### 1.1 Поднятие HTTP сервера
```angular2html
python3 -m http.server 8080
```

Сервер успешно запущен и слушает порт 8080:
```
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
```

### 1.2 Поиск слушающего сокета
```angular2html
ss -tlnp | grep 8080
```

Видим вывод, что сервер слушает на порту 8080:
```angular2html
LISTEN 0      5            0.0.0.0:8080      0.0.0.0:*    users:(("python3",pid=2808,fd=3))
```

### 1.3 Подключение к серверу через curl
Делаем запрос на сервер:
```angular2html
curl http://localhost:8080
```

Получаем ответ:
```html
<!DOCTYPE HTML>
<html lang="en">
<head>
    <meta charset="utf-8">
    <title>Directory listing for /</title>
</head>
<body>
<h1>Directory listing for /</h1>
<hr>
<ul>
    <li><a href=".bash_history">.bash_history</a></li>
    <li><a href=".bash_logout">.bash_logout</a></li>
    <li><a href=".bashrc">.bashrc</a></li>
    <li><a href=".cache/">.cache/</a></li>
    <li><a href=".config/">.config/</a></li>
    <li><a href=".face">.face</a></li>
    <li><a href=".face.icon">.face.icon@</a></li>
    <li><a href=".lesshst">.lesshst</a></li>
    <li><a href=".local/">.local/</a></li>
    <li><a href=".profile">.profile</a></li>
    <li><a href=".sudo_as_admin_successful">.sudo_as_admin_successful</a></li>
    <li><a href=".viminfo">.viminfo</a></li>
    <li><a href="Desktop/">Desktop/</a></li>
    <li><a href="Documents/">Documents/</a></li>
    <li><a href="Downloads/">Downloads/</a></li>
    <li><a href="Music/">Music/</a></li>
    <li><a href="Pictures/">Pictures/</a></li>
    <li><a href="Public/">Public/</a></li>
    <li><a href="Templates/">Templates/</a></li>
    <li><a href="Videos/">Videos/</a></li>
</ul>
<hr>
</body>
```

### 1.4 Анализ состояний TCP-сокетов для 8080
Для анализа используем:
```angular2html
ss -tan | grep 8080
```

**Пример вывода после curl:**
```angular2html
LISTEN    0      5            0.0.0.0:8080      0.0.0.0:*
TIME-WAIT 0      0          127.0.0.1:8080    127.0.0.1:45590  
```

**Объяснение TIME-WAIT:**
- TIME-WAIT — состояние сокета после завершения TCP соединения, обеспечивает корректное завершение обработки запроса 
- Его нельзя вручную удалить — это требование протокола TCP для предотвращения коллизий и повторного использования номера порта до гарантированного завершения старой сессии

**Проблемы при большом количестве TIME-WAIT:**
- Если таких сокетов становится слишком много (например, на высоконагруженных прокси/балансировщиках), может закончиться пул локальных портов, что приведёт к отказу новых соединений
- Также лишний расход ресурсов на хранение состояний соединения

## Задание 2. Динамическая маршрутизация с BIRD 
### 2.1 Создание dummy-интерфейса service_0

Созданим интерфейс и назначим IP:

```angular2html
sudo ip link add service_0 type dummy
sudo ip addr add 192.168.14.88/32 dev service_0
sudo ip link set service_0 up
sudo ip route add 192.168.14.88/32 dev service_0
```
Аналогично создаим остальные интерфейсы:

```angular2html
sudo ip link add service_1 type dummy
sudo ip addr add 192.168.14.1/30 dev service_1
sudo ip link set service_1 up
sudo ip route add 192.168.14.1/30 dev service_1

sudo ip link add service_2 type dummy
sudo ip addr add 192.168.10.4/32 dev service_2
sudo ip link set service_2 up
sudo ip route add 192.168.10.4/32 dev service_1

sudo ip link add srv_1 type dummy
sudo ip addr add 192.168.14.4/32 dev srv_1
sudo ip link set srv_1 up
sudo ip route add 192.168.14.4/32 dev srv_1
```

### 2.3 Конфигурация BIRD для анонса только нужных адресов
Открываю файл конфигурации BIRD `/etc/bird/bird.conf` и добавляю:
```
protocol rip {
    interface "enp0s1" {
        mode multicast;
    };
    network 192.168.14.0/24 {
        import all;
        export filter {
            if net ~ [192.168.14.0/24] && netlen = 32 && (ip ~ [192.168.14.88,192.168.14.XXX]) then accept;
            if net ~ [192.168.14.0/24] && netlen = 32 && iface ~ "service*" then accept;
            reject;
        };
    };
}
```

- Анонсируются только /32 адреса на интерфейсах, имя которых начинается с service_, либо явно указанные

### 2.4 Проверка через tcpdump
Пример фильтра RIP:

```angular2html
sudo tcpdump -i enp0s1 port 520 -n
```
- Видим только объявления нужных адресов; другие (~srv_1, /30) не появляются в RIP объявлениях

Видим выводL
```angular2html
RIP v2, Response, length: 36
AFI IPv4, Route Tag 0, 192.168.14.88/32, metric: 1
```

## Задание 3. Настройка фаервола (Host Firewalling)
### 3.1 Блокировка входящих на порт 8080 с помощью nftables

Создадим таблицу и правило:
```angular2html
sudo nft add table inet filter
sudo nft 'add chain inet filter input { type filter hook input priority 0 ; }'
sudo nft add rule inet filter input tcp dport 8080 drop
```

Проверяем созданные правила:
```angular2html
sudo nft list ruleset
```
Вывод:
```
table inet filter {
	chain input {
		type filter hook input priority filter; policy accept;
		tcp dport 8080 drop
	}
}

```

### 3.2 Запуск веб-сервера и демонстрация работы firewall
Запустим сервер:
```angular2html
python3 -m http.server 8080
```
Пытаемся подключиться:
```angular2html
curl http://localhost:8080
```
- Запрос не проходит, так как порт 8080 заблокирован

### 3.3 tcpdump демонстрация
Запустим tcpdump:
```angular2html
sudo tcpdump -i lo port 8080 -n
```
- Видим входящие SYN-пакеты, но сервер не отвечает (нет SYN-ACK) — firewall отсекает соединение на входе; curl получает timeout

## Задание 4. Аппаратное ускорение (offloading) сетевого трафика
### 4.1 Проверка offload возможностей адаптера

```angular2html
sudo ethtool -k eth0
``` 
Получаю вывод:
```
Features for enp0s1:
rx-checksumming: on [fixed]
tx-checksumming: on
	tx-checksum-ipv4: off [fixed]
	tx-checksum-ip-generic: on
	tx-checksum-ipv6: off [fixed]
	tx-checksum-fcoe-crc: off [fixed]
	tx-checksum-sctp: off [fixed]
scatter-gather: on
	tx-scatter-gather: on
	tx-scatter-gather-fraglist: off [fixed]
tcp-segmentation-offload: on
	tx-tcp-segmentation: on
	tx-tcp-ecn-segmentation: off [fixed]
	tx-tcp-mangleid-segmentation: off
	tx-tcp6-segmentation: on
generic-segmentation-offload: on
generic-receive-offload: on
large-receive-offload: off [fixed]
rx-vlan-offload: off [fixed]
tx-vlan-offload: off [fixed]
ntuple-filters: off [fixed]
receive-hashing: off [fixed]
highdma: on [fixed]
rx-vlan-filter: off [fixed]
vlan-challenged: off [fixed]
tx-lockless: off [fixed]
netns-local: off [fixed]
tx-gso-robust: on [fixed]
tx-fcoe-segmentation: off [fixed]
tx-gre-segmentation: off [fixed]
tx-gre-csum-segmentation: off [fixed]
tx-ipxip4-segmentation: off [fixed]
tx-ipxip6-segmentation: off [fixed]
tx-udp_tnl-segmentation: off [fixed]
tx-udp_tnl-csum-segmentation: off [fixed]
tx-gso-partial: off [fixed]
tx-tunnel-remcsum-segmentation: off [fixed]
tx-sctp-segmentation: off [fixed]
tx-esp-segmentation: off [fixed]
tx-udp-segmentation: off [fixed]
tx-gso-list: off [fixed]
fcoe-mtu: off [fixed]
tx-nocache-copy: off
loopback: off [fixed]
rx-fcs: off [fixed]
rx-all: off [fixed]
tx-vlan-stag-hw-insert: off [fixed]
rx-vlan-stag-hw-parse: off [fixed]
rx-vlan-stag-filter: off [fixed]
l2-fwd-offload: off [fixed]
hw-tc-offload: off [fixed]
esp-hw-offload: off [fixed]
esp-tx-csum-hw-offload: off [fixed]
rx-udp_tunnel-port-offload: off [fixed]
tls-hw-tx-offload: off [fixed]
tls-hw-rx-offload: off [fixed]
rx-gro-hw: on [fixed]
tls-hw-record: off [fixed]
rx-gro-list: off
macsec-hw-offload: off [fixed]
rx-udp-gro-forwarding: off
hsr-tag-ins-offload: off [fixed]
hsr-tag-rm-offload: off [fixed]
hsr-fwd-offload: off [fixed]
hsr-dup-offload: off [fixed]

```
Видим блок segmentation offload:
```angular2html
tcp-segmentation-offload: on
    tx-tcp-segmentation: on
    tx-tcp-ecn-segmentation: off [fixed]
    tx-tcp-mangleid-segmentation: off
    tx-tcp6-segmentation: on
```

### 4.2 Объяснение TCP segmentation offload

- TCP segmentation offload (TSO) переносит задачу сегментации больших TCP-сообщений на уровне ядра с хоста на сетевой адаптер
- Это уменьшает загрузку CPU за счёт аппаратной обработки пакетов (NIC разрезает TCP payload на MSS-size сегменты и выставляет нужные поля TCP/IP)
- Позволяет повысить производительность при передаче крупных файлов/соединений
