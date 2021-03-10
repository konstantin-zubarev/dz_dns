## Стенд для настройки и обслуживание DNS.

Цель:

- взять стенд https://github.com/erlong15/vagrant-bind, добавить сервер client2

завести в зоне dns.lab имена:
- web1 - смотрит на клиент1
- web2 смотрит на клиент2

завести еще одну зону newdns.lab:
- завести в ней запись
- www - смотрит на обоих клиентов

настроить split-dns
- клиент1 - видит обе зоны, но в зоне dns.lab только web1
- клиент2 видит только dns.lab

*) настроить все без выключения selinux

![](topology.jpeg)

### Реализация.

#### 1. Установка и запуск DNS master на сервере ns01
Установим необходимые пакеты для поднятие DNS сервера `bind`, `bind-utils`.
```
[root@ns01 ~]# yum install -y bind bind-utils
```

Создадим каталог `master`, `keys` для DNS сервера `ns01`.
```
[root@ns01 ~]# mkdir -p /var/named/master
[root@ns01 ~]# mkdir -p /var/named/keys
```

Для защиты от искажений и подделок ответов сервера, сгенерируем ключ и сохраним в `/var/named/keys`.
```
[root@ns01 ~]# dnssec-keygen -a HMAC-MD5 -b 128 -n HOST rndc-key | base64
GrtiE9kz16GK+OKKU/qJvQ==
```
Создадим файл rndc.key в каталоги `/var/named/keys`
```
[root@ns01 ~]# vi /var/named/keys/rndc-key.key

key "rndc.key" {
    algorithm hmac-md5;
    secret "GrtiE9kz16GK+OKKU/qJvQ==";
};
```

Анологично сделаим для `cleint1_dns.key`, `cleint2_dns.key`, `default.key`

`cleint1_dns.key`

```
[root@ns01 ~]# dnssec-keygen -a HMAC-MD5 -b 128 -n HOST cleint1_dns.key | base64
S2NsZWludDFfZG5zLmtleS4rMTU3KzI4NjczCg==
[root@ns01 ~]# vi /var/named/keys/cleint1_dns.key

key "cleint1_dns.key" {
    algorithm hmac-md5;
    secret "S2NsZWludDFfZG5zLmtleS4rMTU3KzMxNTYwCg==";
};
```

`cleint2_dns.key`

```
[root@ns01 ~]# dnssec-keygen -a HMAC-MD5 -b 128 -n HOST cleint2_dns.key | base64
S2NsZWludDJfZG5zLmtleS4rMTU3KzExNzA1Cg==
[root@ns01 ~]# vi /var/named/keys/cleint2_dns.key

key "cleint2_dns.key" {
    algorithm hmac-md5;
    secret "S2NsZWludDJfZG5zLmtleS4rMTU3KzExNzA1Cg==";
};
```

`default.key`

```
[root@ns01 ~]# dnssec-keygen -a HMAC-MD5 -b 128 -n HOST default.key | base64
S2RlZmF1bHQua2V5LisxNTcrMDA1MzMK
[root@ns01 ~]# vi /var/named/keys/cleint2_dns.key

key "default.key" {
    algorithm hmac-md5;
    secret "S2RlZmF1bHQua2V5LisxNTcrMDA1MzMK";
};
```

Настроим конфигурационный файл `/etc/named.conf`.
```
[root@ns01 ~]# vi /etc/named.conf
```

<details>
  <summary>named.conf</summary>

```
options {

    // network 
	listen-on port 53 { 192.168.50.10; };
	listen-on-v6 port 53 { ::1; };

    // data
	directory 	"/var/named";
	dump-file 	"/var/named/data/cache_dump.db";
	statistics-file "/var/named/data/named_stats.txt";
	memstatistics-file "/var/named/data/named_mem_stats.txt";

    // server
    recursion yes;
    allow-query     { 192.168.50.0/24; };
    allow-transfer { 192.168.50.11; };
    
    // dnssec
	dnssec-enable yes;
	dnssec-validation yes;

    // others
	bindkeys-file "/etc/named.iscdlv.key";
	managed-keys-directory "/var/named/dynamic";
	pid-file "/run/named/named.pid";
	session-keyfile "/run/named/session.key";
};

logging {
    channel default_debug {
        file "data/named.run";
        severity dynamic;
    };
};

// RNDC Control for client

controls {
        inet 192.168.50.10 allow { 192.168.50.15; } keys { "rndc-key"; }; 
};

acl "client1" { key "cleint1_dns.key"; 192.168.50.15/32; };

acl "client2" { key "cleint2_dns.key"; 192.168.50.20/32; };

// ZONE TRANSFER WITH TSIG
include "keys/rndc-key.key";
include "keys/cleint1_dns.key";
include "keys/cleint2_dns.key";
include "keys/default.key";

view "client1" {
    match-clients { "client1"; };

    allow-transfer { key "cleint1_dns.key"; };

    server 192.168.50.11 {
        keys { "cleint1_dns.key"; };
    };

    // root zone
    zone "." IN {
        type hint;
        file "named.ca";
    };

    // zones like localhost
    include "/etc/named.rfc1912.zones";

    // roots DNSKEY
    include "/etc/named.root.key";

    // dns.lab zone
    zone "dns.lab" {
        type master;
        file "master/named.client1-dns.lab";
        also-notify { 192.168.50.11; };
    };

    // newdns.lab zone
    zone "newdns.lab" {
        type master;
        file "master/named.newdns.lab";
        also-notify { 192.168.50.11; };
    };

    // dns.lab zone reverse
    zone "50.168.192.in-addr.arpa" {
        type master;
        file "master/named.client1-dns.lab.rev";
        also-notify { 192.168.50.11; };
    };

    // ddns.lab zone
    zone "ddns.lab" {
        type master;
        file "dynamic/named.ddns.lab";
        also-notify { 192.168.50.11; };
        allow-update { key "cleint1_dns.key"; };
    };
};

view "client2" {
    match-clients { "client2"; };

    allow-transfer { key "cleint2_dns.key"; };

    server 192.168.50.11 {
        keys { "cleint2_dns.key"; };
    };

    // root zone
    zone "." IN {
        type hint;
        file "named.ca";
    };

    // zones like localhost
    include "/etc/named.rfc1912.zones";

    // roots DNSKEY
    include "/etc/named.root.key";

    // dns.lab zone
    zone "dns.lab" {
        type master;
        file "master/named.client2-dns.lab";
        also-notify { 192.168.50.11; };
    };

    // dns.lab zone reverse
    zone "50.168.192.in-addr.arpa" {
        type master;
        file "master/named.client2-dns.lab.rev";
        also-notify { 192.168.50.11; };
    };
};

view "default" {
    match-clients { "any"; };

    allow-transfer { key "default.key"; };

    server 192.168.50.11 {
        keys { "default.key"; };
    };

    // root zone
    zone "." IN {
        type hint;
        file "named.ca";
    };

    // zones like localhost
    include "/etc/named.rfc1912.zones";

    // roots DNSKEY
    include "/etc/named.root.key";

    // dns.lab zone
    zone "dns.lab" {
        type master;
        file "master/named.dns.lab";
        also-notify { 192.168.50.11; };
    };

    // newdns.lab zone
    zone "newdns.lab" {
        type master;
        file "master/named.newdns.lab";
        also-notify { 192.168.50.11; };
    };

    // dns.lab zone reverse
    zone "50.168.192.in-addr.arpa" {
        type master;
        file "master/named.dns.lab.rev";
        also-notify { 192.168.50.11; };
    };
};
```
</details>

В каталоге `/var/named/master/` разместим зоны для DNS сервера `ns01`.

Основная зона:

<details>
  <summary>dns.lab</summary>

```
$TTL 3600
$ORIGIN dns.lab.
@		IN	SOA	ns01.dns.lab. root.dns.lab. (
                            2302211516 ; serial
                            3600       ; refresh (1 hour)
                            600        ; retry (10 minutes)
                            86400      ; expire (1 day)
                            600        ; minimum (10 minutes)
			)

		IN	NS	ns01.dns.lab.
		IN	NS	ns02.dns.lab.

; DNS Servers
ns01		IN	A	192.168.50.10
ns02		IN	A	192.168.50.11
; other
client1	IN	A	192.168.50.15
client2	IN	A	192.168.50.20
```
</details>

Дополнительная зона для `client1`:

<details>
  <summary>client1-dns.lab</summary>

```
$include "/var/named/master/named.dns.lab"

web1		IN	CNAME	client1.dns.lab.
```
</details>

Дополнительная зона для `client2`:

<details>
  <summary>client1-dns.lab</summary>

```
$include "/var/named/master/named.dns.lab"

web1		IN	CNAME	client1.dns.lab.
web2		IN	CNAME	client2.dns.lab.
```
</details>

Зона `newdns.lab`:

<details>
  <summary>newdns.lab</summary>

```
$TTL 3600
$ORIGIN newdns.lab.
@		IN	SOA	ns01.newdns.lab. root.newdns.lab. (
                            2302211517 ; serial
                            3600       ; refresh (1 hour)
                            600        ; retry (10 minutes)
                            86400      ; expire (1 day)
                            600        ; minimum (10 minutes)
			)

		IN	NS	ns01.newdns.lab.
		IN	NS	ns02.newdns.lab.

; DNS Servers
ns01		IN	A	192.168.50.10
ns02		IN	A	192.168.50.11
; other
www		IN	A	192.168.50.15
www		IN	A	192.168.50.20
```
</details>

Зона `ddns.lab`:

<details>
  <summary>ddns.lab</summary>

```
$TTL 3600
$ORIGIN ddns.lab.
@               IN      SOA     ns01.dns.lab. root.dns.lab. (
                            2302211516 ; serial
                            3600       ; refresh (1 hour)
                            600        ; retry (10 minutes)
                            86400      ; expire (1 day)
                            600        ; minimum (10 minutes)
                        )

                IN      NS      ns01.dns.lab.
                IN      NS      ns02.dns.lab.

; DNS Servers
ns01            IN      A       192.168.50.10
ns02            IN      A       192.168.50.11
```
</details>

В каталоге `/var/named/master/` разместим обратную зоны для DNS сервера `ns01`.

Основная обратная зона:

<details>
  <summary>dns.lab.rev</summary>

```
$TTL 3600
$ORIGIN 50.168.192.in-addr.arpa.
50.168.192.in-addr.arpa.	IN	SOA	ns01.dns.lab. root.dns.lab. (
                            2302211516 ; serial
                            3600       ; refresh (1 hour)
                            600        ; retry (10 minutes)
                            86400      ; expire (1 day)
                            600        ; minimum (10 minutes)
                        )

		IN	NS	ns01.dns.lab.
		IN	NS	ns02.dns.lab.

; DNS Servers
10		IN	PTR	ns01.dns.lab.
11		IN	PTR	ns02.dns.lab.
; other
15		IN	PTR	client1.dns.lab.
20		IN	PTR	client2.dns.lab.
```
</details>

Дополнительная обратная зона для `client1`:

<details>
  <summary>client1-dns.lab.rev</summary>

```
$include "/var/named/master/named.dns.lab.rev"

15		IN	PTR	client1.dns.lab.
```
</details>

Дополнительная обратная зона для `client2`:

<details>
  <summary>client2-dns.lab.rev</summary>

```
$include "/var/named/master/named.dns.lab.rev"

15		IN	PTR	web1.dns.lab.
20		IN	PTR	web2.dns.lab.
```
</details>

#### 2. Настройка DNS slave на сервере ns02



#### 3. Настройка DNS master на сервере ns01

