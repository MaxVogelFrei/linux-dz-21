# DNS
## Задание
настраиваем split-dns  
взять стенд https://github.com/erlong15/vagrant-bind  
добавить еще один сервер client2  
завести в зоне dns.lab  
имена  
web1 - смотрит на клиент1  
web2 смотрит на клиент2  
  
завести еще одну зону newdns.lab  
завести в ней запись  
www - смотрит на обоих клиентов  
  
настроить split-dns  
клиент1 - видит обе зоны, но в зоне dns.lab только web1  
  
клиент2 видит только dns.lab  
  
*) настроить все без выключения selinux  
## Решение
Split dns реализуется добавлением view1 и view2 с листами доступа и разными ключами для трансфера зон в разных view  
```bash
acl "client1" { !key rndc-key; key zonetransfer.key; 192.168.50.15/32; };
acl "client2" { !key zonetransfer.key; key rndc-key; 192.168.50.25/32; };
```
```bash
view "view1" {
    match-clients {"client1";};
...

    // lab's zone
    zone "dns.lab" {
        type master;
        allow-transfer { key zonetransfer.key; };
        file "named.dns.lab.cl1";
    };
...
```
```bash
view "view2" {
    match-clients { "client2"; };

    server 192.168.50.11 {
      keys { rndc-key; };
    };

    // lab's zone
    zone "dns.lab" {
        type master;
        allow-transfer { 192.168.50.11; key rndc-key; };
        file "named.dns.lab";
    };
...
```


### Проверка client1
в зоне dns.lab видно только web1  
```bash
[vagrant@client ~]$ dig @192.168.50.10 web1.dns.lab +short
192.168.50.15
[vagrant@client ~]$ dig @192.168.50.11 web1.dns.lab +short
192.168.50.15
[vagrant@client ~]$ dig @192.168.50.10 web2.dns.lab +short
[vagrant@client ~]$ dig @192.168.50.11 web2.dns.lab +short
```
зону newdns.lab видно и в ней www указывает на 2 адреса  
```bash
[vagrant@client ~]$ dig @192.168.50.10 www.newdns.lab +short
192.168.50.15
192.168.50.25
[vagrant@client ~]$ dig @192.168.50.11 www.newdns.lab +short
192.168.50.25
192.168.50.15
```

### Проверка client2
в зоне dns.lab видно и web1 и web2
```bash
[vagrant@client2 ~]$ dig @192.168.50.10 web1.dns.lab +short
192.168.50.15
[vagrant@client2 ~]$ dig @192.168.50.10 web2.dns.lab +short
192.168.50.25
[vagrant@client2 ~]$ dig @192.168.50.11 web1.dns.lab +short
192.168.50.15
[vagrant@client2 ~]$ dig @192.168.50.11 web2.dns.lab +short
192.168.50.25
```
зону newdns.lab не видно
```bash
[vagrant@client2 ~]$ dig @192.168.50.10 www.newdns.lab +short
[vagrant@client2 ~]$ dig @192.168.50.11 www.newdns.lab +short
```

## Задание *
файлы зон помещаю в /var/named/ , а не в /etc/  
+  
политика selinux named_write_master_zones = 1  
```bash
[root@ns01 ~]# sestatus
SELinux status:                 enabled
SELinuxfs mount:                /sys/fs/selinux
SELinux root directory:         /etc/selinux
Loaded policy name:             targeted
Current mode:                   enforcing
Mode from config file:          enforcing
Policy MLS status:              enabled
Policy deny_unknown status:     allowed
Max kernel policy version:      31
```
```bash
[root@ns02 ~]# sestatus
SELinux status:                 enabled
SELinuxfs mount:                /sys/fs/selinux
SELinux root directory:         /etc/selinux
Loaded policy name:             targeted
Current mode:                   enforcing
Mode from config file:          enforcing
Policy MLS status:              enabled
Policy deny_unknown status:     allowed
Max kernel policy version:      31
```
