Базовая настройка системы
Изменение имени хоста
hostnamectl hostname "имя_машины"
exec bash
systemctl disable firewalld --now

Создание пользователя net_admin (HQ-RTR и BR-RTR)
adduser net_admin
passwd net_admin
nano /etc/sudoers
net_admin ALL=(ALL:ALL)NOPASSWD:ALL

Создание пользователя sshuser (на HQ-SRV и BR-SRV)
useradd -m -u 1010 sshuser
passwd sshuser
nano /etc/sudoers
sshuser ALL=(ALL:ALL)NOPASSWD:ALL

Настройка SSH (на HQ-SRV и BR-SRV)
nano /etc/mybanner
Authorized access only
nano /etc/ssh/sshd_config
Port 2025
Banner /etc/mybanner
MaxAuthTries 2
AllowUsers sshuser
systemctl restart sshd.service

Настройка маршрутизации
nano /etc/sysctl.conf
net.ipv4.ip_forward = 1
reboot

Настройка GRE туннелей

На BR-RTR
Profile name: tun1
Device: tun1
Mode: GRE
Parent: ens34
Local IP: 172.16.5.2
Remote IP: 172.16.4.2
IPv4 Configuration: Manual
IP: 192.168.0.2/24
Gateway: 192.168.0.1

На HQ-RTR
Profile name: tun1
Device: tun1
Mode: GRE
Parent: ens34
Local IP: 172.16.4.2
Remote IP: 172.16.5.2
IPv4 Configuration: Manual
IP: 192.168.0.1/24
Gateway: 192.168.0.2

Если GRE туннель не работает:

проверить пинг HQ-R ↔ BR-R

перезагрузить ISP

отключить временно другие интерфейсы (hqin и brin)

Настройка OSPF маршрутизации

На HQ-R и BR-R
nano /etc/frr/daemons
ospfd=yes
systemctl enable --now frr

На HQ-RTR
vtysh
conf t
router ospf
passive-interface default
network 192.168.0.0/24 area 0
network 172.16.0.0/26 area 0
exit
interface tun1
no ip ospf network broadcast
no ip ospf passive
exit
do wr
exit
exit

nmcli connection edit tun1
set ip-tunnel.ttl 64
save
quit
systemctl restart frr

На BR-RTR
vtysh
conf t
router ospf
passive-interface default
network 192.168.0.0/24 area 0
network 172.16.6.0/27 area 0
exit
interface tun1
no ip ospf network broadcast
no ip ospf passive
exit
do wr
exit
exit

nmcli connection edit tun1
set ip-tunnel.ttl 64
save
quit
systemctl restart frr

Перезагрузить HQ-R и BR-R

Проверка OSPF
vtysh
show ip ospf neighbor
exit

Настройка DHCP сервера (на HQ-RTR)
nano /etc/sysconfig/dhcpd
DHCPARGS=ens35

nano /etc/dhcp/dhcpd.conf
option domain-name "au-team.irpo";
option domain-name-servers 172.16.0.2;
default-lease-time 6000;
max-lease-time 72000;
authoritative;

subnet 172.16.0.0 netmask 255.255.255.192 {
range 172.16.0.3 172.16.0.8;
option routers 172.16.0.1;
}

systemctl enable --now dhcpd

RAID 5 на HQ-SRV
lsblk
sudo mdadm --zero-superblock /dev/sdb
sudo mdadm --zero-superblock /dev/sdc
sudo mdadm --zero-superblock /dev/sdd
sudo mdadm --create --verbose /dev/md0 --level=5 --raid-devices=3 /dev/sdb /dev/sdc /dev/sdd
mkfs -t ext4 /dev/md0
mkdir /etc/mdadm
echo "DEVICE partitions" > /etc/mdadm/mdadm.conf
mdadm --detail --scan | awk '/ARRAY/ {print}' >> /etc/mdadm/mdadm.conf
mkdir /mnt/raid5
nano /etc/fstab
/dev/md0 /mnt/raid5 ext4 defaults 0 0
mount -a
df -h | grep /mnt/raid5

Настройка NFS SERVER на HQ-SRV
mkdir /mnt/raid5/nfs
chmod 766 /mnt/raid5/nfs
nano /etc/exports
/mnt/raid5/nfs 172.16.0.0/28(rw,no_root_squash)
exportfs -arv
systemctl enable --now nfs-server

Настройка NFS CLIENT
mkdir /mnt/nfs
chmod 777 /mnt/nfs
nano /etc/fstab
172.16.0.2:/mnt/raid5/nfs /mnt/nfs nfs defaults 0 0
mount -a
df -h | grep /mnt/nfs

Настройка chrony на HQ-SRV
nano /etc/chrony.conf

server 127.0.0.1 iburst prefer
hwtimestamp *
local stratum 5
allow 0/0

systemctl enable --now chronyd
chronyc sources
chronyc tracking | grep Stratum
