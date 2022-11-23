Vagrant-стенд для обновления ядра и создания образа системы

Цель домашнего задания
Научиться обновлять ядро в ОС Linux. Получение навыков работы с Vagrant, Packer и публикацией готовых образов в Vagrant Cloud. 

Описание домашнего задания
1) Обновить ядро ОС из репозитория ELRepo
2) Создать Vagrant box c помощью Packer
3) Загрузить Vagrant box в Vagrant Cloud

Описание команд и их вывод:
Создадм папку packer: mkdir packer
Перейдём в эту папку: cd packer
Создадим файл centos.json:
#Основная секция, в ней указываются характеристики нашей ВМ
{
"builders": [
    {
    #Указываем ссылку на файл автоматической конфигурации 
      "boot_command": [
        "<tab> inst.text inst.ks=http://{{ .HTTPIP }}:{{ .HTTPPort }}/ks.cfg<enter><wait>"
      ],
      "boot_wait": "10s",
      #Указываем размер диска для ВМ
      "disk_size": "10240",
      "export_opts": [
        "--manifest",
        "--vsys",
        "0",
        "--description",
        "{{user `artifact_description`}}",
        "--version",
        "{{user `artifact_version`}}"
      ],
      #Указываем семейство ОС нашей ВМ
      "guest_os_type": "RedHat_64",
      #Указываем каталог, из которого возьмём файл автоматической конфигурации
      "http_directory": "http",
      #Контрольная сумма ISO-файла
      #Проверяется после скачивания файла
      "iso_checksum": "4c2424925e6da0e02d4b5db22b2ed89c4068c40040ee8cd042c6a0fbb9e4aac1",
      #Ссылка на дистрибутив из которого будет разворачиваться наша ВМ
      "iso_url": "http://mirror.linux-ia64.org/centos/8-stream/isos/x86_64/CentOS-Stream-8-x86_64-20221027-boot.iso",
      #Hostname нашей ВМ
      #Имя ВМ будет взято из переменной image_name
      "name": "{{user `image_name`}}",
      "output_directory": "builds",
      "shutdown_command": "sudo -S /sbin/halt -h -p",
      "shutdown_timeout": "5m",
      #Пароль пользователя
      "ssh_password": "vagrant",
      #Номер ssh-порта
      "ssh_port": 22,
      "ssh_pty": true,
      #Тайм-аут подключения по SSH
      #Если через 220 минут не получается подключиться, то сборка отменяется
      "ssh_timeout": "220m",
      #Имя пользователя
      "ssh_username": "vagrant",
      #Тип созданного образа (Для VirtualBox)
      "type": "virtualbox-iso",
      #Параметры ВМ
      #2 CPU и 1Гб ОЗУ
      "vboxmanage": [
        [
          "modifyvm",
          "{{.Name}}",
          "--memory",
          "1024"
        ],
        [
          "modifyvm",
          "{{.Name}}",
          "--cpus",
          "2"
        ]
      ],
      #Имя ВМ в VirtualBox
      "vm_name": "packer-centos-vm"
    }
  ],
  "post-processors": [
    {
    #Уровень сжатия
      "compression_level": "7",
      #Указание пути для сохранения образа
      #Будет сохранён в каталог packer
      "output": "centos-{{user `artifact_version`}}-kernel-5-x86_64-Minimal.box",
      "type": "vagrant"
    }
  ],
  #Настройка ВМ после установки
  "provisioners": [
    {
      "execute_command": "{{.Vars}} sudo -S -E bash '{{.Path}}'",
      "expect_disconnect": true,
      "override": {
        "{{user `image_name`}}": {
        #Скрипты, которые будут запущены после установки ОС
        #Скрипты выполняются в указанном порядке
          "scripts": [
            "scripts/stage-1-kernel-update.sh",
            "scripts/stage-2-clean.sh"
          ]
        }
      },
      #Тайм-аут запуска скриптов, после того, как подключились по SSH
      "pause_before": "20s",
      "start_retry_timeout": "1m",
      "type": "shell"
    }
  ],
  #Указываем переменные
  #К переменным можно обращаться внутри данного JSON-файла
  "variables": {
    "artifact_description": "CentOS Stream 8 with kernel 5.x",
    "artifact_version": "8",
    "image_name": "centos-8"
  }
}
Создадим в каталоге packer каталог http: mkdir http
Перейдём в каталог http: cd http
Создадим внутри каталога http файл ks.cfg со следующим содержимым:

#Указываем тип установки
 %packages
@^minimal-environment
@standard
%end
# Подтверждаем лицензионное соглашение
eula --agreed
# Указываем язык нашей ОС
lang en_US.UTF-8
# Раскладка клавиутуры
keyboard us
# Указываем часовой пояс
timezone UTC+3
# Включаем сетевой интерфейс и получаем ip-адрес по DHCP
network --bootproto=dhcp --device=link --activate
# Задаём hostname otus-c8
network --hostname=otus-c8
# Указываем пароль root пользователя
rootpw vagrant
authconfig --enableshadow --passalgo=sha512
# Создаём пользователя vagrant, добавляем его в группу Wheel
user --groups=wheel --name=vagrant --password=vagrant --gecos="vagrant"
# Включаем SELinux в режиме enforcing
selinux --enforcing
# Выключаем штатный межсетевой экран
firewall --disabled
firstboot --disable
# Выбираем установку в режиме командной строки
text
# Указываем адрес, с которого установщик возьмёт недостающие компоненты
url --url="http://mirror.centos.org/centos/8-stream/BaseOS/x86_64/os/"
# System bootloader configuration
bootloader --location=mbr --append="ipv6.disable=1 crashkernel=auto"
skipx
logging --level=info
zerombr
clearpart --all --initlabel
# Автоматически размечаем диск, создаём LVM
autopart --type=lvm
# Перезагрузка после установки
reboot

#убираем пароль для пользоватея vagrant
%post
sed -i "s/^.*requiretty/#Defaults requiretty/" /etc/sudoers
echo "vagrant        ALL=(ALL)       NOPASSWD: ALL" >> /etc/sudoers.d/vagrant
chmod 0440 /etc/sudoers.d/vagrant
%end

Вернёмся в каталог packer из каталога http: cd ..
Создадим в каталоге packer каталог scripts: mkdir scripts
Перейдём в каталог scripts: cd scripts
В первом файле stage-1-kernel-update.sh будут содаржаться команды по обновлению ядра, которые мы разобрали в предыдущей части:

#!/bin/bash

# Установка репозитория elrepo
sudo yum install -y https://www.elrepo.org/elrepo-release-8.el8.elrepo.noarch.rpm 
# Установка нового ядра из репозитория elrepo-kernel
yum --enablerepo elrepo-kernel install kernel-ml -y

# Обновление параметров GRUB
grub2-mkconfig -o /boot/grub2/grub.cfg
grub2-set-default 0
echo "Grub update done."
# Перезагрузка ВМ
shutdown -r now
Второй файл stage-2-clean.sh просто очистит ненужные файлы из нашей ОС и добавит ssh-ключ пользователя vagrant (в данной практической работе его можно просто скопировать, разбирать его мы не будем):

#!/bin/bash

# Обновление и очистка всех ненужных пакетов
yum update -y
yum clean all


# Добавление ssh-ключа для пользователя vagrant
mkdir -pm 700 /home/vagrant/.ssh
curl -sL https://raw.githubusercontent.com/mitchellh/vagrant/master/keys/vagrant.pub -o /home/vagrant/.ssh/authorized_keys
chmod 0600 /home/vagrant/.ssh/authorized_keys
chown -R vagrant:vagrant /home/vagrant/.ssh


# Удаление временных файлов
rm -rf /tmp/*
rm  -f /var/log/wtmp /var/log/btmp
rm -rf /var/cache/* /usr/share/doc/*
rm -rf /var/cache/yum
rm -rf /vagrant/home/*.iso
rm  -f ~/.bash_history
history -c

rm -rf /run/log/journal/*
sync
grub2-set-default 0
echo "###   Hi from second stage" >> /boot/grub2/grub.cfg

Переходим в каталог packer: cd ..
Cоздадим образ системы: packer build centos.json
После успешного создания образа в каталоге packer появился файл сentos-8-kernel-5-x86_64-Minimal.box



