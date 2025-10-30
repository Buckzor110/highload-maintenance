## Задание 1. Создание Server Pool для backend

### 1.1 Создание директорий и index.html

Создаю две директории для backend серверов и создаю в них HTML файлы:
```bash
mkdir ~/backend1 ~/backend2
echo '<h1>Response from Backend Server 1</h1>' > ~/backend1/index.html
echo '<h2>*** Response from Backend Server 2 ***</h2>' > ~/backend2/index.html
```

Проверяем, что директории и файлы созданы:
```bash
ls ~/backend1 ~/backend2
/home/debian/backend1:
index.html

/home/debian/backend2:
index.html
```

Проверяем содержимое файлов:
```bash
cat ~/backend1/index.html
<h1>Response from Backend Server 1</h1>

cat ~/backend2/index.html
<h2>*** Response from Backend Server 2 ***</h2>
```

### 1.2 Запуск HTTP серверов

Запускаю HTTP серверы на портах 8081 и 8082:
```bash
cd ~/backend1 && python3 -m http.server 8081 
Serving HTTP on 0.0.0.0 port 8081 (http://0.0.0.0:8081/) ...

cd ~/backend2 && python3 -m http.server 8082
Serving HTTP on 0.0.0.0 port 8082 (http://0.0.0.0:8082/) ...
```

Проверяю, что серверы работают:
```bash
curl http://localhost:8081
<h1>Response from Backend Server 1</h1>

curl http://localhost:8082
<h2>*** Response from Backend Server 2 ***</h2>
```

---

## Задание 2. DNS Load Balancing с помощью dnsmasq 

### 2.1 Конфигурация dnsmasq

Открываю файл `dnsmasq.conf` и настраиваю в нем две A-записи для домена `my-awesome-highload-app.local`, указывающие на IP-адреса серверов:
```
nano ~/dnsmasq.conf
```
Файл `~/dnsmasq.conf`:
```
address=/my-awesome-highload-app.local/127.0.0.1
address=/my-awesome-highload-app.local/127.0.0.2
```

### 2.2 Запуск dnsmasq

Запускаю dnsmasq с указанием конфига и порта 5353:
```bash
sudo dnsmasq --no-daemon --no-resolv --conf-file=~/dnsmasq.conf  --port=5353

dnsmasq: started, version 2.90 cachesize 150
dnsmasq: compile time options: IPv6 GNU-getopt DBus no-UBus i18n IDN2 DHCP DHCPv6 no-Lua TFTP conntrack ipset nftset auth cryptohash DNSSEC loop-detect inotify dumpfile
dnsmasq: warning: no upstream servers configured
dnsmasq: read /etc/hosts - 7 names
```

### 2.3 Проверка DNS через dig

```bash
dig @127.0.0.1 -p5353 my-awesome-highload-app.local

; <<>> DiG 9.18.33-1~deb12u2-Debian <<>> @127.0.0.1 -p 5353 my-awesome-highload-app.local
; (1 server found)
;; global options: +cmd
;; Got answer:
;; WARNING: .local is reserved for Multicast DNS
;; You are currently testing what happens when an mDNS query is leaked to DNS
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 55875
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
;; QUESTION SECTION:
;my-awesome-highload-app.local.	IN	A

;; ANSWER SECTION:
my-awesome-highload-app.local. 0 IN	A	127.0.0.2
my-awesome-highload-app.local. 0 IN	A	127.0.0.1

;; Query time: 0 msec
;; SERVER: 127.0.0.1#5353(127.0.0.1) (UDP)
;; WHEN: Thu Oct 30 05:35:01 PDT 2025
;; MSG SIZE  rcvd: 90
```

### 2.4 Анализ

В ответе видно две A-записи для домена `my-awesome-highload-app.local`, указывающие на IP-адреса наших поднятых серверов
Если backend2 (127.0.0.2:8082) выйдет из строя, DNS всё равно будет отдавать две записи. Клиенты/балансировщики смогут попытаться обратиться ко всем, но реально работать будет только тот backend, который доступен

---

## Задание 3. Балансировка Layer 4 с помощью IPVS 

### 3.1 Создание dummy-интерфейса

Создаю dummy-интерфейс с адресом 192.168.100.1/32
```bash
sudo ip link add dummy1 type dummy
sudo ip addr add 192.168.100.1/32 dev dummy1
sudo ip link set dummy1 up
```

### 3.2 Настройка IPVS

Создаю виртуальный сервер и добавляю бэкенды:
```bash
sudo ipvsadm -A -t 192.168.100.1:80 -s rr
sudo ipvsadm -a -t 192.168.100.1:80 -r 127.0.0.1:8081 -m
sudo ipvsadm -a -t 192.168.100.1:80 -r 127.0.0.1:8082 -m
```

- `-s rr`: round-robin балансировка

### 3.3 Проверка балансировки

```bash
curl http://192.168.100.1
curl http://192.168.100.1
```

Видим, что ответы приходят поочерёдно от backend1 и backend2:
```
debian@debian:~$ curl http://192.168.100.1
<h2>*** Response from Backend Server 2 ***</h2>
debian@debian:~$ curl http://192.168.100.1
<h1>Response from Backend Server 1</h1>
debian@debian:~$ curl http://192.168.100.1
<h2>*** Response from Backend Server 2 ***</h2>
debian@debian:~$ curl http://192.168.100.1
<h1>Response from Backend Server 1</h1>
```

Счётчики IPVS:

```bash
sudo ipvsadm -L -n
```

```
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  192.168.100.1:80 rr
  -> 127.0.0.1:8081               Masq    1      0          2         
  -> 127.0.0.1:8082               Masq    1      0          2         
```
Счётчики InActConn увеличиваются при запросах, что подтверждает балансировку

---

## Задание 4. Балансировка L7 с помощью NGINX 

### 4.1 Конфигурация nginx (active-backup)

Открываем файл конфигурации nginx и настраиваем upstream с двумя backend серверами, где второй является backup:
```
sudo nano /etc/nginx/nginx.conf
```
Настройка upstream + сервер:
```
http {
  upstream backend_pool {
    server 127.0.0.1:8081 max_fails=7 fail_timeout=10s;
    server 127.0.0.1:8082 backup;
  }

  server {
    listen 127.0.0.1:8888;
    location / {
      proxy_pass http://backend_pool;
      proxy_set_header X-high-load-test 123;
      proxy_next_upstream off;
    }
  }
}
```

- **Балансировка active-backup:** трафик идёт на основной сервер, а если 7 подряд ошибок, то всё идёт на backup

### 4.2 Проверка переключения backup

Делаю запрос к nginx:
```bash
curl http://127.0.0.1:8888
<h1>Response from Backend Server 1</h1>
```

Сейчас балансировка идёт на backend1. Теперь симулирую его недоступность

Выключаю backend1:
```
debian@debian:~$ cd ~/backend1 && python3 -m http.server 8081
Serving HTTP on 0.0.0.0 port 8081 (http://0.0.0.0:8081/) ...
192.168.100.1 - - [30/Oct/2025 05:49:57] "GET / HTTP/1.1" 200 -
192.168.100.1 - - [30/Oct/2025 05:49:59] "GET / HTTP/1.1" 200 -
192.168.100.1 - - [30/Oct/2025 05:51:50] "GET / HTTP/1.1" 200 -
^C
Keyboard interrupt received, exiting.
```
Делаю 8 запросов к nginx:
```bash
debian@debian:~$ curl http://127.0.0.1:8888
<html>
<head><title>502 Bad Gateway</title></head>
<body>
<center><h1>502 Bad Gateway</h1></center>
<hr><center>nginx/1.22.1</center>
</body>
</html>
debian@debian:~$ curl http://127.0.0.1:8888
<html>
<head><title>502 Bad Gateway</title></head>
<body>
<center><h1>502 Bad Gateway</h1></center>
<hr><center>nginx/1.22.1</center>
</body>
</html>
debian@debian:~$ curl http://127.0.0.1:8888
<html>
<head><title>502 Bad Gateway</title></head>
<body>
<center><h1>502 Bad Gateway</h1></center>
<hr><center>nginx/1.22.1</center>
</body>
</html>
debian@debian:~$ curl http://127.0.0.1:8888
<html>
<head><title>502 Bad Gateway</title></head>
<body>
<center><h1>502 Bad Gateway</h1></center>
<hr><center>nginx/1.22.1</center>
</body>
</html>
debian@debian:~$ curl http://127.0.0.1:8888
<html>
<head><title>502 Bad Gateway</title></head>
<body>
<center><h1>502 Bad Gateway</h1></center>
<hr><center>nginx/1.22.1</center>
</body>
</html>
debian@debian:~$ curl http://127.0.0.1:8888
<html>
<head><title>502 Bad Gateway</title></head>
<body>
<center><h1>502 Bad Gateway</h1></center>
<hr><center>nginx/1.22.1</center>
</body>
</html>
debian@debian:~$ curl http://127.0.0.1:8888
<html>
<head><title>502 Bad Gateway</title></head>
<body>
<center><h1>502 Bad Gateway</h1></center>
<hr><center>nginx/1.22.1</center>
</body>
</html>
debian@debian:~$ curl http://127.0.0.1:8888
<html>
<head><title>502 Bad Gateway</title></head>
<body>
<center><h1>502 Bad Gateway</h1></center>
<hr><center>nginx/1.22.1</center>
</body>
</html>
debian@debian:~$ curl http://127.0.0.1:8888
<h2>*** Response from Backend Server 2 ***</h2>
```

Видим успешное переключение на backend2 после ряда ошибок, правда почему-то не после 7, а после 8 запросов


### 4.3 Проверка заголовка

```bash
sudo tshark -i lo -f "tcp port 8081 or tcp port 8082" -Y "http.request" -V
```
Вывод:
```
Running as user "root" and group "root". This could be dangerous.
Capturing on 'Loopback: lo'
 ** (tshark:2735) 06:33:26.366179 [Main MESSAGE] -- Capture started.
 ** (tshark:2735) 06:33:26.366204 [Main MESSAGE] -- File: "/tmp/wireshark_loHSGDF3.pcapng"
Frame 6: 184 bytes on wire (1472 bits), 184 bytes captured (1472 bits) on interface lo, id 0
    Section number: 1
    Interface id: 0 (lo)
        Interface name: lo
    Encapsulation type: Ethernet (1)
    Arrival Time: Oct 30, 2025 06:33:31.273074669 PDT
    [Time shift for this packet: 0.000000000 seconds]
    Epoch Time: 1761831211.273074669 seconds
    [Time delta from previous captured frame: 0.000059459 seconds]
    [Time delta from previous displayed frame: 0.000000000 seconds]
    [Time since reference or first frame: 1.272833328 seconds]
    Frame Number: 6
    Frame Length: 184 bytes (1472 bits)
    Capture Length: 184 bytes (1472 bits)
    [Frame is marked: False]
    [Frame is ignored: False]
    [Protocols in frame: eth:ethertype:ip:tcp:http]
Ethernet II, Src: 00:00:00_00:00:00 (00:00:00:00:00:00), Dst: 00:00:00_00:00:00 (00:00:00:00:00:00)
    Destination: 00:00:00_00:00:00 (00:00:00:00:00:00)
        Address: 00:00:00_00:00:00 (00:00:00:00:00:00)
        .... ..0. .... .... .... .... = LG bit: Globally unique address (factory default)
        .... ...0 .... .... .... .... = IG bit: Individual address (unicast)
    Source: 00:00:00_00:00:00 (00:00:00:00:00:00)
        Address: 00:00:00_00:00:00 (00:00:00:00:00:00)
        .... ..0. .... .... .... .... = LG bit: Globally unique address (factory default)
        .... ...0 .... .... .... .... = IG bit: Individual address (unicast)
    Type: IPv4 (0x0800)
Internet Protocol Version 4, Src: 127.0.0.1, Dst: 127.0.0.1
    0100 .... = Version: 4
    .... 0101 = Header Length: 20 bytes (5)
    Differentiated Services Field: 0x00 (DSCP: CS0, ECN: Not-ECT)
        0000 00.. = Differentiated Services Codepoint: Default (0)
        .... ..00 = Explicit Congestion Notification: Not ECN-Capable Transport (0)
    Total Length: 170
    Identification: 0x4cac (19628)
    010. .... = Flags: 0x2, Don't fragment
        0... .... = Reserved bit: Not set
        .1.. .... = Don't fragment: Set
        ..0. .... = More fragments: Not set
    ...0 0000 0000 0000 = Fragment Offset: 0
    Time to Live: 64
    Protocol: TCP (6)
    Header Checksum: 0xef9f [validation disabled]
    [Header checksum status: Unverified]
    Source Address: 127.0.0.1
    Destination Address: 127.0.0.1
Transmission Control Protocol, Src Port: 39376, Dst Port: 8082, Seq: 1, Ack: 1, Len: 118
    Source Port: 39376
    Destination Port: 8082
    [Stream index: 1]
    [Conversation completeness: Incomplete, ESTABLISHED (7)]
    [TCP Segment Len: 118]
    Sequence Number: 1    (relative sequence number)
    Sequence Number (raw): 3596271079
    [Next Sequence Number: 119    (relative sequence number)]
    Acknowledgment Number: 1    (relative ack number)
    Acknowledgment number (raw): 3233835792
    1000 .... = Header Length: 32 bytes (8)
    Flags: 0x018 (PSH, ACK)
        000. .... .... = Reserved: Not set
        ...0 .... .... = Accurate ECN: Not set
        .... 0... .... = Congestion Window Reduced: Not set
        .... .0.. .... = ECN-Echo: Not set
        .... ..0. .... = Urgent: Not set
        .... ...1 .... = Acknowledgment: Set
        .... .... 1... = Push: Set
        .... .... .0.. = Reset: Not set
        .... .... ..0. = Syn: Not set
        .... .... ...0 = Fin: Not set
        [TCP Flags: �������AP���]
    Window: 512
    [Calculated window size: 65536]
    [Window size scaling factor: 128]
    Checksum: 0xfe9e [unverified]
    [Checksum Status: Unverified]
    Urgent Pointer: 0
    Options: (12 bytes), No-Operation (NOP), No-Operation (NOP), Timestamps
        TCP Option - No-Operation (NOP)
            Kind: No-Operation (1)
        TCP Option - No-Operation (NOP)
            Kind: No-Operation (1)
        TCP Option - Timestamps: TSval 1625649998, TSecr 1625649998
            Kind: Time Stamp Option (8)
            Length: 10
            Timestamp value: 1625649998
            Timestamp echo reply: 1625649998
    [Timestamps]
        [Time since first frame in this TCP stream: 0.000122042 seconds]
        [Time since previous frame in this TCP stream: 0.000059459 seconds]
    [SEQ/ACK analysis]
        [iRTT: 0.000062583 seconds]
        [Bytes in flight: 118]
        [Bytes sent since last PSH flag: 118]
    TCP payload (118 bytes)
Hypertext Transfer Protocol
    GET / HTTP/1.0\r\n
        [Expert Info (Chat/Sequence): GET / HTTP/1.0\r\n]
            [GET / HTTP/1.0\r\n]
            [Severity level: Chat]
            [Group: Sequence]
        Request Method: GET
        Request URI: /
        Request Version: HTTP/1.0
    X-high-load-test: 123\r\n
    Host: backend_pool\r\n
    Connection: close\r\n
    User-Agent: curl/7.88.1\r\n
    Accept: */*\r\n
    \r\n
    [Full request URI: http://backend_pool/]
    [HTTP request 1/1]
```

В заголовках запроса видим наш кастомный заголовок `X-high-load-test: 123`
