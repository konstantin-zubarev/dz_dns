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
[root@ns01 ~]# dnssec-keygen -a HMAC-MD5 -b 128 -n HOST rndc.key | base64
S3JuZGMua2V5LisxNTcrMjcwNDAK
```
Создадим файл rndc.key в каталоги `/var/named/keys`
```
[root@ns01 ~]# vi /var/named/keys/rndc.key

key "rndc.key" {
    algorithm hmac-md5;
    secret "S3JuZGMua2V5LisxNTcrMjcwNDAK";
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
S2NsZWludDFfZG5zLmtleS4rMTU3KzI4NjczCg==
[root@ns01 ~]# vi /var/named/keys/cleint2_dns.key

key "cleint2_dns.key" {
    algorithm hmac-md5;
    secret "S2NsZWludDJfZG5zLmtleS4rMTU3KzExNzA1Cg==";
};
```

`default.key`

```
[root@ns01 ~]# dnssec-keygen -a HMAC-MD5 -b 128 -n HOST default.key | base64
S2NsZWludDFfZG5zLmtleS4rMTU3KzI4NjczCg==
[root@ns01 ~]# vi /var/named/keys/cleint2_dns.key

key "default.key" {
    algorithm hmac-md5;
    secret "S2NsZWludDJfZG5zLmtleS4rMTU3KzExNzA1Cg==";
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
key "rndc-key" {
    algorithm hmac-md5;
    secret "GrtiE9kz16GK+OKKU/qJvQ==";
};
controls {
    inet 192.168.50.10 allow { 192.168.50.15; } keys { "rndc-key"; }; 
};

acl "client1" {
    192.168.50.15/32; // client1
};

acl "client2" {
    192.168.50.20/32; // client2
};

// ZONE TRANSFER WITH TSIG
//include "/var/named/keys/named.zonetransfer.key"; 

key "zonetransfer.key" {
    algorithm hmac-md5;
    secret "GrtiE9kz16GK+OKKU/qJvQ==";
};
server 192.168.50.11 {
    keys { "zonetransfer.key"; };
};

view "client1" {
    match-clients { "client1"; };

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
        allow-transfer { key "zonetransfer.key"; };
    };

    // newdns.lab zone
    zone "newdns.lab" {
        type master;
        file "master/named.newdns.lab";
        allow-transfer { key "zonetransfer.key"; };
    };

    // dns.lab zone reverse
    zone "50.168.192.in-addr.arpa" {
        type master;
        file "master/named.client1-dns.lab.rev";
        allow-transfer { key "zonetransfer.key"; };
    };

    // ddns.lab zone
    zone "ddns.lab" {
        type master;
        file "dynamic/named.ddns.lab";
        allow-transfer { key "zonetransfer.key"; };
        allow-update { key "zonetransfer.key"; };
    };
};

view "client2" {
    match-clients { "client2"; };

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
        allow-transfer { key "zonetransfer.key"; };
    };

    // dns.lab zone reverse
    zone "50.168.192.in-addr.arpa" {
        type master;
        file "master/named.client2-dns.lab.rev";
        allow-transfer { key "zonetransfer.key"; };
    };
};

view "default" {
    match-clients { "any"; };

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
        allow-transfer { key "zonetransfer.key"; };
    };

    // newdns.lab zone
    zone "newdns.lab" {
        type master;
        file "master/named.newdns.lab";
        allow-transfer { key "zonetransfer.key"; };
    };

    // dns.lab zone reverse
    zone "50.168.192.in-addr.arpa" {
        type master;
        file "master/named.dns.lab.rev";
        allow-transfer { key "zonetransfer.key"; };
    };

    // ddns.lab zone
    zone "ddns.lab" {
        type master;
        file "dynamic/named.ddns.lab";
        allow-transfer { key "zonetransfer.key"; };
        allow-update { key "zonetransfer.key"; };
    };
};
```
</details>


Создадим зоны для DNS сервера `ns01`.

<details>
  <summary>dns.lab</summary>

```
dns.lab
```
</details>


<details>
  <summary>newdns.lab</summary>

```

```
</details>


<details>
  <summary>ddns.lab</summary>

```

```
</details>


Создадим обратную зону для DNS сервера `ns01`.

<details>
  <summary>dns.lab</summary>

```

```
</details>

#### 2. Настройка DNS slave на сервере ns02



#### 3. Настройка DNS master на сервере ns01

