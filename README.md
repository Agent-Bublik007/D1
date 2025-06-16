# D1
########################## Proxmox ##########################
Настройка происходит в веб интерфесе стенда, не в консоле
stend -> Система -> Сеть -> vmbr3(HQ-Net) -> Редактировать
Ставим галочку "Поддержка виртуальной ЛС" -> Ок -> Применить конфигурацию

HQ-SRV -> Оборудование -> Сетевое устройство (net0) -> Редактировать -> Тег виртуальной ЛС: 10 -> Ок
HQ-CLI -> Оборудование -> Сетевое устройство (net0) -> Редактировать -> Тег виртуальной ЛС: 20 -> Ок
stend -> Система -> Сеть -> Перезагрузить -> Да

########################## На всех устройствах ##########################
hostnamectl set-hostname isp	--- на ISP
			 hq-rtr.au-team.irpo	--- на HQ-RTR
			 hq-srv.au-team.irpo	--- на HQ-SRV
			 hq-cli.au-team.irpo	--- на HQ-CLI
			 br-rtr.au-team.irpo	--- на BR-RTR
			 br-srv.au-team.irpo	--- на BR-SRV
exec bash

>>>>>>2 ПУНКТ<<<<<<<<
########################## ISP ##########################
cd /etc/net/ifaces
mkdir enp6s{19,20}
cp enp6s18/options enp6s19/
cp enp6s18/options enp6s20/
vim enp6s19/options

BOOTPROTO=static
TYPE=eth

esc
:wq

vim enp6s20/options

BOOTPROTO=static
TYPE=eth

esc
:wq
	
echo 172.16.40.1/28 > enp6s19/ipv4address
echo 172.16.50.1/28 > enp6s20/ipv4address
vim /etc/net/sysctl.conf

net.ipv4.ip_forward = 1     --- меняем 0 на 1

esc
:wq

systemctl restart network
ip -c a
sysctl -a | grep ip_forward   --- убедится что наша строчка поменялась с 0 на 1

apt-get update
apt-get install iptables -y
iptables -t nat -A POSTROUTING -o enp6s18 -j MASQUERADE
iptables-save > /etc/sysconfig/iptables
systemctl enable --now iptables

>>>>>>>>>> 4 ПУНКТ <<<<<<<<<<<
########################## HQ-RTR ##########################
iptables -t nat -A POSTROUTING -o enp6s18 -j MASQUERADE
iptables-save > /etc/sysconfig/iptables
systemctl enable --now iptables
cd /etc/net/ifaces
mkdir enp6s19
mkdir enp6s19.10
mkdir enp6s19.20
mkdir enp6s19.99
mkdir tun0
cp enp6s18/options enp6s19/
echo 172.16.40.2/28 > enp6s18/ipv4address
echo default via 172.16.40.1 > enp6s18/ipv4route
echo nameserver 77.88.8.8 > enp6s18/resolv.conf
echo 192.168.10.1/27 > enp6s19.10/ipv4address
echo 192.168.20.1/28 > enp6s19.20/ipv4address
echo 192.168.99.1/29 > enp6s19.99/ipv4address
echo 172.16.30.1/30 > tun0/ipv4address
vim enp6s19.10/options

TYPE=vlan
HOST=enp6s19
VID=10
BOOTPROTO=static

esc
:wq

vim enp6s19.20/options

TYPE=vlan
HOST=enp6s19
VID=20
BOOTPROTO=static

esc
:wq

vim enp6s19.99/options

TYPE=vlan
HOST=enp6s19
VID=99
BOOTPROTO=static

esc
:wq

vim tun0/options

TYPE=iptun
TUNTYPE=gre
TUNLOCAL=172.16.40.2
TUNREMOTE=172.16.50.2
TUNTTL=64
TUNMTU=1400
TUNOPTIONS='ttl 64'

esc
:wq

vim /etc/net/sysctl.conf

net.ipv4.ip_forward = 1     --- меняем 0 на 1

esc
:wq

systemctl restart network

apt-get update
apt-get install frr -y
vim /etc/frr/daemons

osfpd=yes     --- меняем no на yes

esc
:wq

systemctl enable --now frr
vtysh
configure
interface tun0
ip ospf authentication message-digest
ip ospf message-digest-key 1 md5 P@ssw0rd
no ip ospf passive
exit
router ospf
passive-interface default
network 192.168.10.0/27 area 0
network 192.168.20.0/28 area 0
network 192.168.99.0/29 area 0
network 172.16.30.0/30 area 0
do wr
exit
exit
exit

########################## BR-RTR ##########################
iptables -t nat -A POSTROUTING -o enp6s18 -j MASQUERADE
iptables-save > /etc/sysconfig/iptables
systemctl enable --now iptables
cd /etc/net/ifaces
mkdir enp6s19
mkdir tun0
cp enp6s18/options enp6s19/
echo 172.16.50.2/28 > enp6s18/ipv4address
echo default via 172.16.50.1 > enp6s18/ipv4route
echo nameserver 77.88.8.8 > enp6s18/resolv.conf
echo 192.168.0.1/28 > enp6s19/ipv4address
echo 172.16.30.2/30 > tun0/ipv4address
vim tun0/options

TYPE=iptun
TUNTYPE=gre
TUNLOCAL=172.16.50.2
TUNREMOTE=172.16.40.2
TUNTTL=64
TUNMTU=1400
TUNOPTIONS='ttl 64'

esc
:wq

vim /etc/net/sysctl.conf
net.ipv4.ip_forward = 1     --- меняем 0 на 1

esc
:wq

systemctl restart network

apt-get update
apt-get install frr -y
vim /etc/frr/daemons

osfpd=yes     --- меняем no на yes

esc
:wq

systemctl enable --now frr
vtysh
configure
interface tun0
ip ospf authentication message-digest
ip ospf message-digest-key 1 md5 P@ssw0rd
no ip ospf passive
exit
router ospf
passive-interface default
network 192.168.0.0/28 area 0
network 172.16.30.0/30 area 0
do wr

########################## HQ-SRV,BR-SRV ##########################
cd /etc/net/ifaces

echo 192.168.10.2/27 > enp6s18/ipv4address		|
echo default via 192.168.10.1 > enp6s18/ipv4route	|  --- для HQ-SRV
echo nameserver 77.88.8.8 > enp6s18/resolv.conf		|
systemctl restart network

echo 192.168.0.2/28 > enp6s18/ipv4address		|
echo default via 192.168.0.1 > enp6s18/ipv4route	|  --- для BR-SRV
echo nameserver 192.168.10.2 > enp6s18/resolv.conf	|
systemctl restart network

>>>>>>>>3 ПУНКТ<<<<<<<<<<<
useradd sshuser -u 1015
passwd sshuser
P@ssw0rd
P@ssw0rd

usermod -aG wheel sshuser
vim /etc/sudoers

#WHEEL_USERS ALL=(ALL:ALL) ALL			|  Находим эти строчки и раскомментируем
#WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL		|
sshuser ALL=(ALL:ALL) NOPASSWD: ALL

esc
:wq!

Проверяем доступ, логинимся под sshuser с паролем P@ssw0rd
sudo -i    --- если не запросил пароль, то всё верно

>>>>>>>>>>5 ПУНКТ<<<<<<<<<<
vim /etc/openssh/sshd_config
Далее необходимо найти строчки
Port 22    --- меняем с 22 на 3015, раскомментируем
MsxAuthTries 6  --- меняем с 6 на 2, раскомментируем

#no default banner path  			| Находим #no default banner path, изменяем строчку ниже, добавляя путь к файлу с данными по баннеру
Banner /etc/mybanner				|

#AllowGroups wheel users			| Находим #AllowGroups wheel users, добавляем строчку ниже указывая нашего пользователя sshuser
AllowUsers sshuser				|

esc
:wq

echo Authorized access only > /etc/mybanner
systemctl restart sshd

Проверяем работу с HQ-RTR
ssh sshuser@192.168.10.2   --- Должно выдать "Connection refused"
ssh sshuser@192.168.10.2 -p 3015
yes
P@ssw0rd    --- пароль от пользователя sshuser
Если зашли - значит работает 

>>>>>>>>>>3 ПУНКТ<<<<<<<<<<<
########################## HQ-RTR,BR-RTR ##########################
useradd net_admin
passwd net_admin
P@$$word
P@$$word

usermod -aG wheel net_admin
vim /etc/sudoers

#WHEEL_USERS ALL=(ALL:ALL) ALL			|  Находим эти строчки и раскомментируем
#WHEEL_USERS ALL=(ALL;ALL) NOPASSWD: ALL		|
net_admin ALL=(ALL;ALL) NOPASSWD: ALL

esc
:wq!

Проверяем доступ, логинимся под net_admin с паролем P@$$word
sudo -i    --- если не запросил пароль, то всё верно

>>>>>>>>> 9 ПУНКТ <<<<<<<<<<<
########################## HQ-RTR ##########################
apt-get install dhcp-server -y
systemctl enable --now dhcpd
cp /etc/dhcp/dhcpd.conf.example /etc/dhcp/dhcpd.conf
vim /etc/dhcp/dhcpd.conf

Находим строчку # A slightly different configuration for an internal subnet и ниже меняем на свои данные
subnet 192.168.20.0 netmask 255.255.255.240 {
  range 192.168.20.1 192.168.20.14;
  option domain-name-server 192.168.10.2;
  option domain-name "au-team.irpo";
  option routers 192.168.20.1;
  option broadcast-address 192.168.20.15;
  default-lease-time 600;
  max-lease-time 7200;
}

esc
:wq

vim /etc/sysconfig/dhcpd

DHCPDARGS=enp6s19.20

esc
:wq

systemctl restart dhcpd
После чего включаем HQ-CLI, логинимся и он сразу должен получить айпи адрес 


>>>>>>>>>>>>>>> 11 ПУНКТ <<<<<<<<<<<<
########################## ISP ##########################
apt-get install tzdata 
timedatectl set-timezone Europe/Moscow

########################## На всех остальных устройствах ##########################
timedatectl set-timezone Europe/Moscow

>>>>>>>>>> 10 ПУНКТ <<<<<<<<<<<<<
########################## HQ-SRV ##########################
apt-get update
apt-get install bind bind-utils
echo nameserver 127.0.0.1 > /etc/net/ifaces/enp6s18/resolv.conf
systemctl restart network
vim /etc/bind/options.conf

...
listen-on { any; };
listen-on-v6 { none; };
...
forward first;
forwarders { 77.88.8.8; };
...
allow-query { any; };
...
allow-query-cache { any; };
...
allow-recursion { any; };
...

esc
:wq

systemctl enable --now bind

host ya.ru 127.0.0.1		--- Проверяем работоспособность перенаправляющего DNS сервера

cp /etc/bind/zone/localhost  /etc/bind/zone/au-team.irpo
vim /etc/bind/zone/au-team.irpo

Меняем все localhost на au-team.irpo

	      IN	NS   au-team.irpo.
	      IN	A	   192.168.10.2
hq-rtr	IN	A	   192.168.10.1
	      IN	A	   192.168.20.1
	      IN	A	   192.168.99.1
br-rtr	IN	A	   192.168.0.1
hq-srv	IN	A	   192.168.10.2
hq-cli	IN	A	   192.168.20.2
br-srv	IN	A	   192.168.0.2
moodle	IN	A	   172.16.40.2
wiki	  IN	A	   172.16.50.2

esc
:wq

chown named:named /etc/bind/zone/au-team.irpo
vim /etc/bind/rfc1912.conf

…
zone "au-team.irpo" {
	type master;
	file "au-team.irpo";
};

esc
:wq

systemctl restart bind

host hq-srv.au-team.irpo 127.0.0.1		| Проверяем работоспособность прямой зоны DNS
host br-srv.au-team.irpo 127.0.0.1		|
cp /etc/bind/zone/127.in-addr.arpa  /etc/bind/zone/168.192.in-addr.arpa
vim /etc/bind/zone/168.192.in-addr.arpa

Меняем все localhost на au-team.irpo

	    IN	NS	  au-team.irpo.
1.10	IN	PTR	  hq-rtr.au-team.irpo.
1.20	IN	PTR	  hq-rtr.au.team.irpo.
1.99	IN	PTR	  hq-rtr.au.team.irpo.
2.10	IN	PTR	  hq-srv.au.team.irpo.
2.20	IN	PTR	  hq-cli.au.team.irpo.

esc
:wq

chown named:named /etc/bind/zone/168.192.in-addr.arpa
vim /etc/bind/rfc1912.conf

…
zone "168.192.in-addr.arpa" {
	type master;
	file "168.192.in-addr.arpa";
};

esc
:wq

systemctl restart bind

host 192.168.10.2 127.0.0.1		| Проверяем работоспособность обратной зоны DNS
host 192.168.20.1 127.0.0.1		|
