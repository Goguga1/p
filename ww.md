ISP:
user → useruser → sudo -i
hostnamectl set-hostname isp; exec bash
mcedit /etc/network/interfaces
auto eth0
iface eth0 inet dhcp
auto eth1
iface eth1 inet static
address 172.16.4.1/28
auto eth2
iface eth2 inet static
address 172.16.5.1/28
(F2 → F10)
systemctl restart networking
timedatectl set-timezone Europe/Moscow → timedatectl status
mcedit /etc/sysctl.conf → net.ipv4.ip_forward=1 → sysctl -p
iptables -t nat -A POSTROUTING –s 172.16.4.0/28 –o eth0 -j MASQUERADE
iptables -t nat -A POSTROUTING –s 172.16.5.0/28 –o eth0 -j MASQUERADE
iptables -t nat -L
iptables-save > /root/rules
export EDITOR=mcedit → crontab -e
@reboot /sbin/iptables-restore < /root/rules
reboot
iptables -t nat -L

HQ-RTR:
user → useruser → sudo -i
hostnamectl set-hostname hq-rtr.au-team.irpo; exec bash
systemctl disable network-manager
mcedit /etc/network/interfaces
auto eth0
iface eth0 inet static
address 172.16.4.2/28
gateway 172.16.4.1
auto eth1
iface eth1 inet static
address 192.168.1.1/26
auto eth2
iface eth2 inet static
address 192.168.2.1/28
auto eth3
iface eth3 inet static
address 192.168.3.1/29
(F2 → F10)
systemctl restart networking
timedatectl set-timezone Europe/Moscow → timedatectl status
mcedit /etc/sysctl.conf → net.ipv4.ip_forward=1 → sysctl -p
iptables -t nat -A POSTROUTING -s 192.168.1.0/26 -o eth0 -j MASQUERADE
iptables -t nat -A POSTROUTING -s 192.168.2.0/28 -o eth0 -j MASQUERADE
iptables -t nat -A POSTROUTING -s 192.168.3.0/29 -o eth0 -j MASQUERADE
iptables -t nat -L
iptables-save > /root/rules
export EDITOR=mcedit → crontab -e
@reboot /sbin/iptables-restore < /root/rules
reboot
iptables -t nat -L
mcedit /etc/network/interfaces
auto gre1
iface gre1 inet tunnel
address 10.10.10.1
netmask 255.255.255.252
mode gre
local 172.16.4.2
endpoint 172.16.5.2
ttl 255
systemctl restart networking → ip a
mcedit /etc/apt/sources.list
#deb https://dl.astralinux.ru/astra/stables/2.12_x86-64/repository/ orel main contrib non-free
deb [trusted=yes] http://deb.debian.org/debian buster main
mcedit /etc/resolv.conf → nameserver 8.8.8.8 → (F2 → F10)
[systemctl restart networking]
apt update → apt install frr
mcedit /etc/frr/daemons → ospfd=yes → (F2 → F10)
systemctl restart frr → vtysh → conf t → router ospf
network 10.10.10.0/30 area 0
network 192.168.1.0/26 area 0
network 192.168.2.0/28 area 0
network 192.168.3.0/29 area 0
do wr mem → ex → int gre1
ip ospf authentication message-digest
ip ospf message-digest-key 1 md5 P@ssw0rd
do wr mem → ex → ex → ex
mcedit /etc/apt/sources.list
deb https://dl.astralinux.ru/astra/stables/2.12_x86-64/repository/ orel main contrib non-free
#deb [trusted=yes] http://deb.debian.org/debian buster main
[vtysh] → [do show ip ospf neighbor]
[mcedit /etc/resolv.conf] → [nameserver 8.8.8.8] → [(F2 → F10)]
apt update → apt install dnsmasq
mcedit /etc/dnsmasq.conf
no-resolv
dhcp-range=192.168.2.2,192.168.2.14,9999h
dhcp-option=3,192.168.2.1
dhcp-option=6,192.168.1.2
interface=eth1.200
systemctl restart dnsmasq → systemctl status dnsmasq
useradd net_admin -m
passwd net_admin → P@$$word
mcedit /etc/sudoers
net_admin	ALL=(ALL:ALL) NOPASSWD: ALL (Добавить в самый конец)
[mcedit /etc/resolv.conf] → [nameserver 8.8.8.8] → [(F2 → F10)]
[systemctl restart networking]
apt update → apt install chrony → systemctl status chrony
timedatectl → mcedit /etc/chrony/chrony.conf
local stratum 5
allow 192.168.1.0/26
allow 192.168.2.0/28
allow 172.16.5.0/28
allow 192.168.4.0/27
#pool 2.debian.pool.ntp.org iburst
#rtcsync
(F2 → F10)
systemctl enable --now chrony → systemctl restart chrony
timedatectl set-ntp 0 → timedatectl
apt update → apt install openssh-server → mcedit /etc/ssh/sshd_config
Port 22
MaxAuthTries 2
AllowUsers net_admin
PermitRootLogin no
Banner /root/banner
(F2 → F10)
echo “Authorized access only” > /root/banner
systemctl enable --now sshd → systemctl restart sshd
iptables -t nat -A PREROUTING -p tcp -d 192.168.1.1 --dport 2024 -j DNAT --to-destination 192.168.1.2:2024
iptables-save > /root/rules
reboot
[echo “nameserver 192.168.1.2” > /etc/resolv.conf] → [apt update]
apt install nginx → mcedit /etc/nginx/sites-available/proxy
server {
        listen 80;
        server_name moodle.au-team.irpo;
        location / {
                proxy_pass http://192.168.1.2:80;
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP  $remote_addr;
                proxy_set_header X-Forwarded-For $remote_addr;
        }
}
server {
        listen 80;
        server_name wiki.au-team.irpo;
        location / {
                proxy_pass http://192.168.4.2:8080;
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP  $remote_addr;
                proxy_set_header X-Forwarded-For $remote_addr;
        }
}
(F2 → F10)
rm -rf /etc/nginx/sites-available/default
rm -rf /etc/nginx/sites-enabled/default
ln -s /etc/nginx/sites-available/proxy /etc/nginx/sites-enabled
ls -la /etc/nginx/sites-enabled
systemctl restart nginx
BR-RTR:
user → useruser → sudo -i
hostnamectl set-hostname br-rtr.au-team.irpo; exec bash
systemctl disable network-manager
mcedit /etc/network/interfaces
auto eth0
iface eth0 inet static
address 172.16.5.2/28
gateway 172.16.5.1
auto eth1
iface eth1 inet static
address 192.168.4.1/27
(F2 → F10)
systemctl restart networking
timedatectl set-timezone Europe/Moscow → timedatectl status
mcedit /etc/sysctl.conf → net.ipv4.ip_forward=1 → sysctl -p
iptables -t nat -A POSTROUTING -s 192.168.4.0/27 -o eth0 -j MASQUERADE
iptables -t nat -L
iptables-save > /root/rules
export EDITOR=mcedit → crontab -e
@reboot /sbin/iptables-restore < /root/rules
reboot
iptables -t nat -L
mcedit /etc/network/interfaces
auto gre1
iface gre1 inet tunnel
address 10.10.10.2
netmask 255.255.255.252
mode gre
local 172.16.5.2
endpoint 172.16.4.2
ttl 255
(F2 → F10)
systemctl restart networking → ip a
ping 10.10.10.1
mcedit /etc/apt/sources.list
#deb https://dl.astralinux.ru/astra/stables/2.12_x86-64/repository/ orel main contrib non-free
deb [trusted=yes] http://deb.debian.org/debian buster main
mcedit /etc/resolv.conf → nameserver 8.8.8.8 → (F2 → F10)
[systemctl restart networking]
apt update → apt install frr
mcedit /etc/frr/daemons → ospfd=yes → (F2 → F10)
systemctl restart frr → vtysh → conf t → router ospf
network 10.10.10.0/30 area 0
network 192.168.4.0/27 area 0
do wr mem → ex → int gre1
ip ospf authentication message-digest
ip ospf message-digest-key 1 md5 P@ssw0rd
do wr mem → ex → ex → ex
mcedit /etc/apt/sources.list
deb https://dl.astralinux.ru/astra/stables/2.12_x86-64/repository/ orel main contrib non-free
#deb [trusted=yes] http://deb.debian.org/debian buster main
[vtysh] → [do show ip ospf neighbor] → [ex]
useradd net_admin -m
passwd net_admin → P@$$word
mcedit /etc/sudoers
net_admin	ALL=(ALL:ALL) NOPASSWD: ALL (Добавить в самый конец)
apt purge ntp → apt purge chrony
mcedit /etc/resolv.conf
nameserver 8.8.8.8
nameserver 192.168.1.2
(F2 → F10)
[systemctl restart networking]
[apt update] → [apt install systemd-timesyncd]
mcedit /etc/systemd/timesyncd.conf → NTP=172.16.4.2 → (F2 → F10)
systemctl enable --now systemd-timesyncd
[apt update] → [apt install openssh-server]
mcedit /etc/ssh/sshd_config
Port 22
MaxAuthTries 2
AllowUsers net_admin
PermitRootLogin no
Banner /root/banner
(F2 → F10)
echo “Authorized access only” > /root/banner
[systemctl enable --now sshd] → systemctl restart sshd
iptables -t nat -A PREROUTING -p tcp -d 192.168.4.1 --dport 80 -j DNAT --to-destination 192.168.4.2:8080
iptables -t nat -A PREROUTING -p tcp -d 192.168.4.1 --dport 2024 -j DNAT --to-destination 192.168.4.2:2024
iptables-save > /root/rules
reboot
HQ-SRV:
user → user → su – → root
hostnamectl hostname hq-srv.au-team.irpo; exec bash
timedatectl set-timezone Europe/Moscow → timedatectl status
ip a → cd /etc/net/ifaces/[ens192]/ → ls → mcedit options
BOOTPROTO=static
TYPE=eth
NM_CONTROLLED=no
DISABLED=no
(F2 → F10)
mcedit ipv4address → 192.168.1.2/26 → (F2 → F10)
mcedit ipv4route → default via 192.168.1.1 → (F2 → F10)
export EDITOR=mcedit → crontab -e
@reboot /bin/systemctl restart network → (F2 → F10)
reboot
[traceroute 192.168.4.2] → systemctl disable --now bind
mcedit /etc/resolv.conf → nameserver 8.8.8.8 → (F2 → F10)
apt-get update → [apt-get install dnsmasq]
systemctl enable --now dnsmasq → systemctl status dnsmasq
mcedit /etc/dnsmasq.conf
no-resolv
domain=au-team.irpo
server=8.8.8.8
interface=*
address=/hq-rtr.au-team.irpo/192.168.1.1
ptr-record=1.1.168.192.in-addr.arpa,hq-rtr.au-team.irpo
cname=moodle.au-team.irpo,hq-rtr.au-team.irpo
cname=wiki.au-team.irpo,hq-rtr.au-team.irpo
address=/br-rtr.au-team.irpo/192.168.4.1
address=/hq-srv.au-team.irpo/192.168.1.2
ptr-record=2.1.168.192.in-addr.arpa,hq-srv.au-team.irpo
address=/hq-cli.au-team.irpo/192.168.2.6 (Смотреть адрес на HQ-CLI, т.к он выдаётся по DHCP)
ptr-record=6.2.168.192.in-addr.arpa,hq-cli.au-team.irpo
address=/br-srv.au-team.irpo/192.168.4.2
(F2 → F10) → mcedit /etc/hosts → 192.168.1.1(tab)hq-rtr.au-team.irpo → (F10)
systemctl restart dnsmasq
mcedit /etc/dnsmasq.conf (После настройки samba)
server=/au-team.irpo/192.168.4.2 (Добавить перед server=8.8.8.8)
(F2 → F10)
systemctl restart dnsmasq
ping google.com
ping hq-rtr.au-team.irpo
useradd sshuser -u 1010 → [id sshuser]
passwd sshuser → P@ssw0rd
mcedit /etc/sudoers
WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL (Раскомментировать)
usermod -aG wheel sshuser → apt-get install openssh-common
mcedit /etc/openssh/sshd_config
Port 2024
MaxAuthTries 2
AllowUsers sshuser
PermitRootLogin no
Banner /root/banner → (F2 → F10) → echo “Authorized access only” > /root/banner
systemctl enable --now sshd → systemctl restart sshd
Добавить 3 диска по 1Гб → lsblk
mdadm --create /dev/md0 --level=5 --raid-devices=3 /dev/sd[b-d]
cat /proc/mdstat
mdadm --detail -scan --verbose > /etc/mdadm.conf
fdisk /dev/md0
n → Enter ... Enter → w
mkfs.ext4 /dev/md0p1
mcedit /etc/fstab
/dev/md0p1	/raid5		ext4	defaults	0	0
(F2 → F10)
mkdir /raid5 → mount -a
apt-get update → apt-get install nfs-server
mkdir /raid5/nfs → chown 99:99 /raid5/nfs → chmod 777 /raid5/nfs
mcedit /etc/exports
/raid5/nfs 192.168.2.0/28(rw,sync,no_subtree_check)
(F2 → F10)
exportfs -a → exportfs -v
systemctl enable nfs → systemctl restart nfs → ls /raid5/nfs
systemctl disable --now chronyd → systemctl status chronyd
apt-get update → apt-get install systemd-timesyncd
mcedit /etc/systemd/timesyncd.conf → NTP=192.168.1.1 → (F2 → F10)
systemctl enable --now systemd-timesyncd → timedatectl timesync-status
reboot
apt-get update && apt-get install apache2 php8.2 apache2-mod_php8.2 mariadb-server php8.2-opcache php8.2-curl php8.2-gd php8.2-intl php8.2-mysqli php8.2-xml php8.2-xmlrpc php8.2-ldap php8.2-zip php8.2-soap php8.2-mbstring php8.2-json php8.2-xmlreader php8.2-fileinfo php8.2-sodium
systemctl enable -–now httpd2 mysqld
mysql_secure_installation → Enter → Y → Y → 123qweR% → Enter → Y → Y → Y → Y
mariadb -u root -p → 123qweR%
CREATE DATABASE moodledb;
CREATE USER moodle IDENTIFIED BY ‘P@ssw0rd’;
GRANT ALL PRIVILEGES ON moodledb.* TO moodle;
FLUSH PRIVILEGES; → exit
curl -L https://github.com/moodle/moodle/archive/refs/tags/v4.5.0.zip > /root/moodle.zip
[ls] → unzip /root/moodle.zip -d /var/www/html
mv /var/www/html/moodle-4.5.0/* /var/www/html/
ls /var/www/html → mkdir /var/www/moodledata
chown apache2:apache2 /var/www/html
chown apache2:apache2 /var/www/moodledata
mcedit /etc/php/8.2/apache2-mod_php/php.ini
(F7 → max_input_vars → Enter)
max_input_vars = 5000 (раскомментировать и изменить знач.) → (F2 → F10)
cd /var/www/html → ls → rm index.html → y → Enter → systemctl restart httpd2
mcedit /var/www/html/config.php
$CFG->wwwroot = ‘http://moodle.au-team.irpo’; → (F2 → F10)
HQ-CLI:
user → user → su – → root
hostnamectl hostname hq-cli.au-team.irpo; exec bash
timedatectl set-timezone Europe/Moscow → timedatectl status
ip a → cd /etc/net/ifaces/[ens192]/ → ls → mcedit options
BOOTPROTO=dhcp
TYPE=eth
NM_CONTROLLED=no
DISABLED=no
systemctl restart network → export EDITOR=mcedit → crontab -e
@reboot /bin/systemctl restart network
reboot
ping google.com
ping hq-rtr.au-team.irpo
dig moodle.au-team.irpo
dig wiki.au-team.irpo
Пуск → Центр управления системой → root → Аутентификация → Домен Active Directory → Домен: AU-TEAM.IRPO → Рабочая группа: AU-TEAM → Имя компьютера: hq-cli → Галочка восстановить файлы ... → Применить → Да → Administrator → 123qweR% → OK → OK
reboot
user → user → su – → root
apt-get update → apt-get install admc
kinit administrator → 123qweR% → admc
Настройки → Дополнительные возможности (галочку вкл.)
au-team.irpo → sudoers → rules_hq (double click) → атрибуты → sudoOption (double click) → Изменить → Добавить → !authenticate → OK → OK → sudoCommand (double click) → Изменить → Добавить → /bin/grep → OK → Добавить → /usr/bin/id → OK → OK → OK
apt-get update → apt-get install sudo libsss_sudo → control sudo public
mcedit /etc/sssd/sssd.conf
services = nss, pam, sudo
sudo_provider = ad (Добавить перед auth_provider=ad)
(F2 → F10)
mcedit /etc/nsswitch.conf
sudoers: files sss (Добавить после gshadow: files)
(F2 → F10)
reboot
user → user → su – → root
rm -rf /var/lib/sss/db/*
sss_cache -E → systemctl restart sssd
sudo -l -U user1.hq
logout
user1.hq → 123qweR%
sudo cat /etc/passwd | sudo grep root && sudo id root
apt-get update → apt-get install nfs-clients
mkdir -p /mnt/nfs → mcedit /etc/fstab
192.168.1.2:/raid5/nfs	/mnt/nfs	nfs	intr,soft,_netdev,x-systemd.automount 0 0
(F2 → F10)
mount -a → mount -v → touch /mnt/nfs/file
systemctl disable --now chronyd → systemctl status chronyd
apt-get update → apt-get install systemd-timesyncd
mcedit /etc/systemd/timesyncd.conf → NTP=192.168.1.1 → (F2 → F10)
systemctl enable --now systemd-timesyncd → timedatectl timesync-status
reboot
[ctrl+alt+F2 → root → root] → [startx] → [reboot] (если сломалось GUI)
useradd sshuser -u 1010 → [id sshuser] → passwd sshuser → P@ssw0rd
mcedit /etc/sudoers
WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL → (F2 → F10)
usermod -aG wheel sshuser
apt-get install openssh-common → mcedit /etc/openssh/sshd_config
Port 2024
MaxAuthTries 2
AllowUsers sshuser
PermitRootLogin no
Banner /root/banner
(F2 → F10)
echo “Authorized access only” > /root/banner
systemctl enable --now sshd → systemctl restart sshd
Пуск → Mozilla Firefox → 192.168.4.2:8080 → set up the wiki → Далее → Далее → Хост базы данных: mariadb → Имя базы данных (без дефисов): mediawiki → Имя пользователя базы данных: wiki → Пароль базы данных: WikiP@ssw0rd → Далее → Далее → Название вики: wiki (можно своё название) → Ваше имя участника: wiki → Пароль: WikiP@ssw0rd → убрать галочку → переключить радиокнопку → Далее → Далее → Далее
scp -P 2024 /home/user(текущий пользователь)/Загрузки/LocalSettings.php sshuser@192.168.4.2:/home/sshuser/ → P@ssw0rd
Пуск → Mozilla Firefox → 192.168.4.2:8080
ssh -p 2024 sshuser@192.168.4.1 → P@ssw0rd
Пуск → Mozilla Firefox → http://192.168.1.2/install.php → Русский (ru) → Далее → Далее → MariaDB (“родной”/mariadb) → Далее → Название базы данных: moodledb; Пользователь базы данных: moodle; Пароль: P@ssw0rd → Далее → Продолжить → Продолжить (ждем долго) → Продолжить
Логин: admin; Новый пароль: P@ssw0rd; Имя: Администратор (можно любое); Фамилия: Пользователь (можно любое); Адрес электронной почты: test.test@mail.ru (можно любое) → Обновить профиль
Полное название сайта: moodle (можно любое); Краткое название сайта: 11 (согласно вашему рабочему месту); Настройки местоположения: Европа/Москва (согласно вашему региону); Контакты службы поддержки: test.test@mail.ru (можно любое) → Сохранить изменения
Пуск → Mozilla Firefox → http://moodle.au-team.irpo
Пуск → Mozilla Firefox → http://wiki.au-team.irpo
apt-get update → apt-get install yandex-browser-stable
Пуск → Yandex Browser
BR-SRV:
user → user → su – → root
hostnamectl hostname br-srv.au-team.irpo; exec bash
ip a → cd /etc/net/ifaces/[ens192]/ → ls → mcedit options
BOOTPROTO=static
TYPE=eth
NM_CONTROLLED=no
DISABLED=no
(F2 → F10)
mcedit ipv4address → 192.168.4.2/27 → (F2 → F10)
mcedit ipv4route → default via 192.168.4.1
export EDITOR=mcedit → crontab -e
@reboot /bin/systemctl restart network → (F2 → F10) → reboot
timedatectl set-timezone Europe/Moscow → timedatectl status
useradd sshuser -u 1010 → [id sshuser]
passwd sshuser → P@ssw0rd
mcedit /etc/sudoers
WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL (Раскомментировать)
usermod -aG wheel sshuser
[apt-get install openssh-common]
mcedit /etc/openssh/sshd_config
Port 2024
MaxAuthTries 2
AllowUsers sshuser
PermitRootLogin no
Banner /root/banner
systemctl enable --now sshd
systemctl restart sshd
[apt-get remove bind]
mcedit /etc/resolv.conf → (F8 → F8 → F8 → F8) → nameserver 192.168.1.2
apt-get update → apt-get install task-samba-dc
rm -rf /etc/samba/smb.conf
[hostname -f]→[hostnamectl set-hostname br-srv.au-team.irpo; exec bash]
mcedit /etc/hosts → 192.168.4.2 br-srv.au-team.irpo → (F2 → F10)
samba-tool domain provision
[AU-TEAM.IRPO] (Enter)
[AU-TEAM] (Enter)
[dc] (Enter)
[SAMBA_INTERNAL] (Enter)
[192.168.1.2 (Возможно вводим вручную)] (Enter)
123qweR% (Enter)
mv -f /var/lib/samba/private/krb5.conf /etc/krb5.conf
systemctl enable samba
export EDITOR=mcedit → сrontab -e
[@reboot /bin/systemctl restart network]
@reboot /bin/systemctl restart samba
Пустая строка → (F2 → F10)
reboot → user → user → su – → root
samba-tool domain info 127.0.0.1
samba-tool user add user1.hq 123qweR%
samba-tool user add user2.hq 123qweR%
samba-tool user add user3.hq 123qweR%
samba-tool user add user4.hq 123qweR%
samba-tool user add user5.hq 123qweR%
samba-tool group add hq
samba-tool group addmembers hq user1.hq,user2.hq,user3.hq,user4.hq,user5.hq
apt-repo add rpm http://altrepo.ru/local-p10 noarch local-p10
apt-get update → apt-get install sudo-samba-schema
sudo-schema-apply → Yes [Да] → (Enter)
Administrator → 123qweR% → 123qweR% (Enter) → (Enter)
create-sudo-rule
Имя правила : rules_hq
sudoCommand : /bin/cat
sudoUser  : %hq
(Tab → OK)
[curl -L https://bit.ly/3C1nEYz > /root/users.zip]
[unzip users.zip –d /opt]
ls /opt → mcedit import
#!/bin/bash
csv_file=”/opt/Users.csv”
while IFS=”;” read -r firstName lastName role phone ou street zip city country password; do
    if [ “$firstName” == “First Name” ]; then
        continue
    fi
    username=”${firstName,,}.${lastName,,}”
    sudo samba-tool user add “$username” 123qweR%
done < “$csv_file”
(F2 → F10)
chmod +x /root/import → bash /root/import
systemctl disable --now chronyd → systemctl status chronyd
apt-get update → apt-get install systemd-timesyncd
mcedit /etc/systemd/timesyncd.conf → NTP=172.16.4.2 → (F2 → F10)
systemctl enable --now systemd-timesyncd → timedatectl timesync-status
reboot → user → user → su – → root
apt-get update → apt-get install ansible
[mkdir /etc/ansible] → mcedit /etc/ansible/hosts
hq-srv ansible_host=sshuser@192.168.1.2 ansible_port=2024
hq-cli ansible_host=sshuser@192.168.2.6 ansible_port=2024 (может быть другой адрес)
hq-rtr ansible_host=net_admin@192.168.1.1 ansible_port=22
br-rtr ansible_host=net_admin@192.168.4.1 ansible_port=22
(F2 → F10)
mcedit /etc/ansible/ansible.cfg
ansible_python_interpreter=/usr/bin/python3 (под [defaults])
(F2 → F10)
ssh-keygen -t rsa → Enter → Enter → Enter
ssh-copy-id -p 22 net_admin@192.168.4.1 → yes → P@$$word
ssh-copy-id -p 2024 sshuser@192.168.2.6 (другой IP, т.к. адрес он получает по DHCP) → yes → P@ssw0rd
ssh-copy-id -p 2024 sshuser@192.168.1.2 → yes → P@ssw0rd
ssh-copy-id -p 22 net_admin@192.168.1.1 → yes → P@$$word
ansible all -m ping
apt-get update → apt-get install docker-engine docker-compose
systemctl enable --now docker → systemctl status docker
docker pull mediawiki → docker pull mariadb
mcedit /home/user/wiki.yml
services:
  mariadb:
    image: mariadb
    container_name: mariadb
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: 123qweR%
      MYSQL_DATABASE: mediawiki
      MYSQL_USER: wiki
      MYSQL_PASSWORD: WikiP@ssw0rd
    volumes: [ mariadb_data:/var/lib/mysql ]
  wiki:
    image: mediawiki
    container_name: wiki
    restart: always
    environment:
      MEDIAWIKI_DB_HOST: mariadb
      MEDIAWIKI_DB_USER: wiki
      MEDIAWIKI_DB_PASSWORD: WikiP@ssw0rd
      MEDIAWIKI_DB_NAME: mediawiki
    ports:
      - "8080:80"
    #volumes: [ /home/user/mediawiki/LocalSettings.php:/var/www/html/LocalSettings.php ]
volumes:
  mariadb_data:
(F2 → F10)
docker compose -f /home/user/wiki.yml up -d
→ Настроить через HQ-CLI
[rm -rf /home/user/LocalSettings.php]
mkdir /home/user/mediawiki
mv /home/sshuser/LocalSettings.php /home/user/mediawiki/
ls /home/user/mediawiki/
mcedit /home/user/wiki.yml
→ Раскомментировать volumes
(F2 → F10)
docker compose -f /home/user/wiki.yml up -d
