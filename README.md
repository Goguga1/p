hostnamectl set-hostname
Настройка ip-v4 isp:
mcedit /etc/network/interfaces
auto eth0
    iface eth0 inet dhcp
auto eth1
    iface eth1 inet static
    address 172.16.4.1/28
systemctl restart networking
f2 сохранение изменений и f10 выход из файла
//
Настройка ip-v4 hq-rtr:
auto eth0
    iface eth0 inet static
    address 172.16.4.2/28
    gateway 172.16.4.1
auto eth1(vlan должен быть 4095 в esxi)
    iface eth1 inet manual
auto eth1.100
    iface eth1.100 inet static
    address 192.168.1.1/26
    vlan-raw-device eth1
//
Настройка ip-v4 br-srv:
cd /etc/net/ifaces/ens19
mcedit options
TYPE=eth
DISABLED=no
BOOTPROTO=static
NM_CONTROLLED=no
mcedit ipv4address
192.168.4.2/27
mcedit ipv4route
default via 192.168.4.1
systemctl restart network
export EDITOR=mcedit
crontab -e
@reboot /bin/systemctl restart network
//
настроим часовой пояс на всех устройствах 
timedatectl set-timezone Europe/Moscow
//
на всех роутерах убераем комент с net.ipv4.ip_forward=1 в файле /etc/sysctl.conf и после пишем sysctl -p
//
настройка nat на isp
iptables -t nat -A POSTROUTING -s 172.16.4.0/28 -o eth0 -j MASQUERADE
iptables -t nat -A POSTROUTING -s 172.16.5.0/28 -o eth0 -j MASQUERADE
iptables-save > /root/rules
export EDITOR=mcedit
crontab -e
@reboot /sbin/iptables-restore < /root/rules(нужно оставить пустую строку в конце файла)
//
настройка nat на hq-rtr
iptables -t nat -A POSTROUTING -s 192.168.1.0/26 -o eth0 -j MASQUERADE
iptables -t nat -A POSTROUTING -s 192.168.2.0/28 -o eth0 -j MASQUERADE
iptables -t nat -A POSTROUTING -s 192.168.3.0/29 -o eth0 -j MASQUERADE
iptables-save > /root/rules
export EDITOR=mcedit
crontab -e
@reboot /sbin/iptables-restore < /root/rules(нужно оставить пустую строку в конце файла)
//
настройка nat на br-rtr
iptables -t nat -A POSTROUTING -s 192.168.4.0/27 -o eth0 -j MASQUERADE
iptables-save > /root/rules
export EDITOR=mcedit
crontab -e
@reboot /sbin/iptables-restore < /root/rules(нужно оставить пустую строку в конце файла)
//
настройка hq-srv
mkdir /etc/net/ifaces/enp0s3.100
mcedit /etc/net/ifaces/enp0s3.100/options
TYPE=vlan
HOST= enp0s3 (основной интерфейс, но у вас может быть иное название)
VID=100 (id VLAN’а)
DISABLED=no
BOOTPROTO=static
mcedit /etc/net/ifaces/enp0s3.100/ipv4address 
192.168.1.2/26
mcedit /etc/net/ifaces/enp0s3.100/ipv4route
default via 192.168.1.1
systemctl restart network
export EDITOR=mcedit
crontab -e
@reboot /bin/systemctl restart netowrk(нужно оставить пустую строку в конце файла)
//
настройка hq-cli
mkdir /etc/net/ifaces/enp0s3.200
mcedit /etc/net/ifaces/enp0s3.200/options
TYPE=vlan
HOST= enp0s3 (основной интерфейс, но у вас может быть иное название)
VID=200 (id VLAN’а)
DISABLED=no
BOOTPROTO=dhcp
systemctl restart network
EDITOR=mcedit
crontab -e
@reboot /bin/systemctl restart netowrk(нужно оставить пустую строку в конце файла)
//
Настройка IP-туннеля между офисами HQ и BR
hq-rtr
mcedit /etc/network/interfaces
auto gre1
    iface gre1 inet tunnel
    address 10.10.10.1
    netmask 255.255.255.252
    mode gre
    local 172.16.4.2
    endpoint 172.16.5.2
    ttl 255
systemctl restart networking
//
br-rtr
mcedit /etc/network/interfaces
auto gre1
    iface gre1 inet tunnel
    address 10.10.10.2
    netmask 255.255.255.252
    mode gre
    local 172.16.5.2
    endpoint 172.16.4.2
    ttl 255
systemctl restart networking
//
Настройка динамической маршрутизации с помощью link-state протокола OSPF.
hq-rtr
Нужно закомментировать в /etc/apt/sources.list первую строку с репозиторием АСТРЫ, т.к. он не имеет пакета frr даже после обновлений репозиториев, вместо него мы будем использовать debian репозиторий.
ниже пишем deb [trusted=yes] http://deb.debian.org/debian buster main
mcedit /etc/resolv.conf
nameserver 8.8.8.8
apt update
apt install frr
Затем нам нужно включить настройку ospf через конфигурационный файл /etc/frr/daemons:
mcedit /etc/frr/daemons
Находим в нём следующую строку и приводим её к такому виду:
ospfd=yes
systemctl restart frr
vtysh (зайти в режим настройки)
conf t (режим конфигурации, ВСПОМИНАЕМ ЦИСКО, РЕБЯТКИ!)
router ospf
network 10.10.10.0/30 area 0
network 192.168.1.0/26 area 0
network 192.168.2.0/28 area 0
network 192.168.3.0/29 area 0
do wr mem
Теперь настроим парольную защиту на нашем GRE туннеле через frr:
vtysh
conf t
int gre1
ip ospf authentication message-digest
ip ospf message-digest-key 1 md5 P@ssw0rd
do wr mem
ПОСЛЕ ПРОДЕЛАННОЙ РАБОТЫ, РАСКОММЕНТИРУЙТЕ РЕПОЗИТОРИЙ АСТРЫ И ЗАКОММЕНТИРУЙТЕ РЕПОЗИТОРИЙ DEBIAN!!! ВОТ ТАК
//
br-rtr 
начало тоже самое 
vtysh
conf t 
router ospf
network 10.10.10.0/30 area 0
network 192.168.4.0/27 area 0
do wr mem
Теперь настроим парольную защиту на нашем GRE туннеле через frr на второй стороне тоже:
vtysh
conf t
int gre1
ip ospf authentication message-digest
ip ospf message-digest-key 1 md5 P@ssw0rd
do wr mem
ПОСЛЕ ПРОДЕЛАННОЙ РАБОТЫ, РАСКОММЕНТИРУЙТЕ РЕПОЗИТОРИЙ АСТРЫ И ЗАКОММЕНТИРУЙТЕ РЕПОЗИТОРИЙ DEBIAN!!! ВОТ ТАК
//
Настройка  протокола динамической конфигурации хостов (DHCP)
на hq-rtr
mcedit /etc/resolv.conf
nameserver 8.8.8.8
apt update
apt install dnsmasq
mcedit /etc/dnsmasq.conf
И внесем в него следующие строки (можно прямо в начало файла):
no-resolv
dhcp-range=192.168.2.2,192.168.2.14,9999h
dhcp-option=3,192.168.2.1
dhcp-option=6,192.168.1.2
interface=eth1.200
systemctl restart dnsmasq
systemctl status dnsmasq
//
Проверим работу службы на HQ-CLI, перезапускаем службу network на нём и посмотрим, выдался ли нам адрес:
systemctl restart network
ip a
//
Настройка DNS для офисов HQ и BR
Для начала необходимо отключить несовместимую службу bind если она есть, командой
systemctl disable --now bind
Для работы DNS есть служба dnsmasq (она же и для DHCP)
Установим её на наш сервер HQ-SRV (если есть, как у нас, то переходите к следующему шагу).
Ещё нам нужно добавить в resolv.conf сервер Google, иначе мы не сможем обновить репозитории, поэтому идём его редактировать следующей командой:
mcedit /etc/resolv.conf
И добавляем следующую строку в него:
nameserver 8.8.8.8
Обновим пакеты и установим её командами:
apt-get update
apt-get install dnsmasq (Установка пакета dnsmasq)
systemctl enable --now dnsmasq (Добавление службы в автозапуск)
Затем откроем файл для редактирования конфигурации нашего DNS-сервера:
mcedit /etc/dnsmasq.conf
И добавляем в неё строки (для удобства прям с первой строки файла):
no-resolv (не будет использовать /etc/resolv.conf)
domain=au-team.irpo
server=8.8.8.8 (адрес общедоступного DNS-сервера)
interface=* (на каком интерфейсе будет работать служба)
address=/hq-rtr.au-team.irpo/192.168.1.1
ptr-record=1.1.168.192.in-addr.arpa,hq-rtr.au-team.irpo
cname=moodle.au-team.irpo,hq-rtr.au-team.irpo
cname=wiki.au-team.irpo,hq-rtr.au-team.irpo

address=/br-rtr.au-team.irpo/192.168.4.1

address=/hq-srv.au-team.irpo/192.168.1.2
ptr-record=2.1.168.192.in-addr.arpa,hq-srv.au-team.irpo

address=/hq-cli.au-team.irpo/192.168.2.2 (Смотрите адрес на HQ-CLI, т.к он выдаётся по DHCP)
ptr-record=2.2.168.192.in-addr.arpa,hq-cli.au-team.irpo

address=/br-srv.au-team.irpo/192.168.4.2

Теперь необходимо добавить строку 192.168.1.1<--->hq-rtr.au-team.irpo в файл /etc/hosts:
mcedit /etc/hosts
systemctl restart dnsmasq
Проверим пинг сначала с HQ-SRV на google.com и hq-rtr.au-team.irpo:
ping google.com
ping hq-rtr.au-team.irpo
Теперь проверим пинг с HQ-CLI:
ping google.com
ping hq-rtr.au-team.irpo
//
Создание локальных учетных записей
