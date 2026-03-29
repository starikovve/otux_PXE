# otux_PXE
# DHCP, PXE

Описание домашнего задания
1. Настроить загрузку по сети дистрибутива Ubuntu 24
2. Установка должна проходить из HTTP-репозитория.
3. Настроить автоматическую установку c помощью файла user-data
   
*4. Настроить автоматическую загрузку по сети дистрибутива Ubuntu 24 c использованием UEFI

Vagrantfile развернёт хост pxeserver, настроит его через скрипт, и, далее развернёт хост pxeclient,
который попробует загрузиться через pxe.

Vagrant up закончится ошибкой

# 1. Настройка DHCP и TFTP-сервера

Для того, чтобы наш клиент мог получить ip-адрес нам требуется DHCP-сервер, чтобы можно было получить
файл pxelinux.0 нам потребуется TFTP-сервер. Утилита dnsmasq совмещает в себе сразу и DHCP и TFTP-
сервер.

```
Для того, чтобы наш клиент мог получить ip-адрес нам требуется DHCP-сервер, чтобы можно было получить
файл pxelinux.0 нам потребуется TFTP-сервер. Утилита dnsmasq совмещает в себе сразу и DHCP и TFTP-
сервер.
1) отключаем firewall:
systemctl stop firewalld
systemctl disable firewalld
2) устанавливаем утилиту dnsmasq 
yum install -y dnsmasqsudo apt update
sudo apt install dnsmasq
3) создаём файл /etc/dnsmasq.d/pxe.conf и добавляем в него следующее содержимое
nano /etc/dnsmasq.d/pxe.conf #Указываем интерфейс в на котором будет работать DHCP/TFTP
interface=eth1
bind-interfaces
#Также указаваем интерфейс и range адресов которые будут выдаваться по DHCP
dhcp-range=eth1,10.0.0.100,10.0.0.120
#Имя файла, с которого надо начинать загрузку для Legacy boot (этот пример рассматривается в методичке)
dhcp-boot=pxelinux.0
#Имена файлов, для UEFI-загрузки (не обязательно добавлять)
dhcp-match=set:efi-x86_64,option:client-arch,7
dhcp-boot=tag:efi-x86_64,bootx64.efi
#Включаем TFTP-сервер
enable-tftp
#Указываем каталог для TFTP-сервера
tftp-root=/srv/tftp/amd64
4) создаём каталоги для файлов TFTP-сервера
mkdir -p /srv/tftp
5) скачиваем файлы для сетевой установки centos.org/7.9 и распаковываем их в каталог /srv/tftp
wget http://vault.centos.org/7.9.2009/os/x86_64/images/pxeboot/vmlinuz -P /srv/tftp/
wget http://vault.centos.org/7.9.2009/os/x86_64/images/pxeboot/initrd.img -P /srv/tftp/

pxelinux.cfg
mkdir -p /srv/tftp/pxelinux.cfg
nano /srv/tftp/pxelinux.cfg/default
DEFAULT menu.c32
PROMPT 0
TIMEOUT 300
ONTIMEOUT local

MENU TITLE PXE Boot Menu

LABEL local
  MENU LABEL Boot from local disk
  LOCALBOOT 0

LABEL centos7
  MENU LABEL Install CentOS 7
  KERNEL centos7/vmlinuz
  APPEND initrd=centos7/initrd.img inst.repo=http://vault.centos.org/7.9.2009/os/x86_64/ quiet
Итоговая структура
/srv/tftp/
├── pxelinux.0
├── menu.c32
├── pxelinux.cfg/
│   └── default
└── centos7/
    ├── vmlinuz
    └── initrd.img
6) перезапускаем службу dnsmasq
systemctl restart dnsmasq
systemctl enable dnsmasq

```
