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

# Настройка DHCP и TFTP-сервера
Для того, чтобы наш клиент мог получить ip-адрес нам требуется DHCP-сервер, чтобы можно было получить
файл pxelinux.0 нам потребуется TFTP-сервер. Утилита dnsmasq совмещает в себе сразу и DHCP и TFTP-
сервер.

отключаем firewall:
- systemctl stop ufw
- systemctl disable ufw

<img width="734" height="83" alt="image" src="https://github.com/user-attachments/assets/a9adfc8b-7760-46f3-a34c-de2552aab315" />

# обновляем кэш и устанавливаем утилиту dnsmasq

```
sudo apt update
sudo apt install dnsmasq
```
# создаём файл /etc/dnsmasq.d/pxe.conf и добавляем в него следующее содержимое
```
vim /etc/dnsmasq.d/pxe.conf
#Указываем интерфейс в на котором будет работать DHCP/TFTP
interface=enp0s8
bind-interfaces
#Также указаваем интерфейс и range адресов которые будут выдаваться по DHCP
dhcp-range=enp0s8,10.0.0.100,10.0.0.120
#Имя файла, с которого надо начинать загрузку для Legacy boot (этот пример рассматривается в методичке)
dhcp-boot=pxelinux.0
#Имена файлов, для UEFI-загрузки (не обязательно добавлять)
dhcp-match=set:efi-x86_64,option:client-arch,7
dhcp-boot=tag:efi-x86_64,bootx64.efi
#Включаем TFTP-сервер
enable-tftp
#Указываем каталог для TFTP-сервера
tftp-root=/srv/tftp/amd64
```
<img width="1225" height="295" alt="image" src="https://github.com/user-attachments/assets/5f3fe141-d8fe-479f-ad88-09fdce1f8f19" />


# cоздаём каталоги для файлов TFTP-сервера
```
mkdir -p /srv/tftp
```
<img width="626" height="115" alt="image" src="https://github.com/user-attachments/assets/6610e9cc-d163-452a-a834-266537e2c243" />

# скачиваем файлы для сетевой установки Ubuntu и распаковываем их в каталог /srv/tftp
```
wget https://releases.ubuntu.com/noble/ubuntu-24.04.4-netboot-amd64.tar.gz ( ссылка поменялась )
tar -xzvf ubuntu-24.04.4-netboot-amd64.tar.gz -C /srv/tftp
по итогу, в каталоге /srv/tftp/amd64 мы увидем вот такие файлы:
├── bootx64.efi
├── grub
│ └── grub.cfg
├── grubx64.efi
├── initrd
├── ldlinux.c32
├── linux
├── pxelinux.0
└── pxelinux.cfg
└── default
2 directories, 8 files
```
<img width="698" height="268" alt="image" src="https://github.com/user-attachments/assets/45d81f8f-a925-42fa-b78b-1351cb79e3bb" />

- перезапускаем службу dnsmasq
```
systemctl restart dnsmasq
```

# Настройка Web-сервера

Для того, чтобы отдавать файлы по HTTP нам потребуется настроенный веб-сервер.
- устанавливаем Web-сервер apache2
```
sudo apt install apache2
 создаём каталог /srv/images в котором будут храниться iso-образы для установки по сети
mkdir /srv/images
```
- переходим в каталог /srv/images и скачиваем iso-образ ubuntu 24.04
```
cd /srv/images
wget https://releases.ubuntu.com/noble/ubuntu-24.04.4-live-server-amd64.iso
```
<img width="1461" height="210" alt="image" src="https://github.com/user-attachments/assets/b5fc69bd-bd53-413f-a961-2db0cefb864a" />

- создаём файл /etc/apache2/sites-available/ks-server.conf и добавлем в него следующее содержимое
```
vim /etc/apache2/sites-available/ks-server.conf
#Указываем IP-адрес хоста и порт на котором будет работать Web-сервер
<VirtualHost 10.0.0.20:80>
DocumentRoot /
# Указываем директорию /srv/images из которой будет загружаться iso-образ
<Directory /srv/images>
Options Indexes MultiViews
AllowOverride All
Require all granted
</Directory>
</VirtualHost>
```
<img width="920" height="231" alt="image" src="https://github.com/user-attachments/assets/225518ee-2f82-4de9-b492-78718a24d5a3" />

- активируем конфигурацию ks-server в apache
```
sudo a2ensite ks-server.conf
```
- вносим изменения в файл /srv/tftp/amd64/pxelinux.cfg/default
vim /srv/tftp/amd64/pxelinux.cfg/default
DEFAULT install
LABEL install
KERNEL linux
INITRD initrd
APPEND root=/dev/ram0 ramdisk_size=3000000 ip=dhcp iso-url=http://10.0.0.20/srv/images/noble-live-server-
amd64.iso autoinstall
В данном файле мы указываем что файлы linux и initrd будут забираться по tftp, а сам iso-образ ubuntu 24.04
будет скачиваться из нашего веб-сервера http://10.0.0.20/srv/images/noble-live-server-amd64.iso
Из-за того, что образ достаточно большой (2.6G) и он сначала загружается в ОЗУ, необходимо указать
размер ОЗУ до 3 гигабайт (root=/dev/ram0 ramdisk_size=3000000)
- перезагружаем сервер
```
systemctl restart apache2
```
Столкнулся с проблемой загрузки

<img width="1545" height="224" alt="image" src="https://github.com/user-attachments/assets/13d592cf-d8c8-4aba-bf37-0ac679a9be09" />


На этом настройка Web-сервера завершена и на данный момент, если мы запустим ВМ pxeclient, то увидим
загрузку по PXE
и далее увидим загрузку iso-образа
и откроется мастер установки ubuntu Так как на хосте pxeclient используется 2 сетевых карты, загрузка может начаться с неправильного
адреса, тогда мы получим вот такое сообщение
В данной ситуации поможет перезапуск виртуальной машины с помощью VirtualBox, либо её удаление и
повторная инициализация с помощью команды vagrant up

<img width="975" height="526" alt="image" src="https://github.com/user-attachments/assets/4113fdea-d05c-4068-a30a-576f2e63bcb7" />


