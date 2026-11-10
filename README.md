**Vagrant-стенд c OSPF**

# Цели
- Создать домашнюю сетевую лабораторию. 
- Научится настраивать протокол OSPF в Linux-based системах.

# Описание

1. Разверачиваем 3 виртуальные машины
2. Объединяем их разными vlan
	- настраиваем OSPF между машинами на базе FRR;
	- изображаем ассиметричный роутинг;
	- делаем один из линков "дорогим", но что бы при этом роутинг был симметричным.

# Немного теории

**OSPF** — протокол динамической маршрутизации, использующий концепцию разделения на области в целях масштабирования. 

Административная дистанция OSPF — 110

Основные свойства протокола OSPF:
- Быстрая сходимость
- Масштабируемость (подходит для маленьких и больших сетей)
- Безопасность (поддежка аутентиикации)
- Эффективность (испольование алгоритма поиска кратчайшего пути)

При настроенном OSPF маршрутизатор формирует таблицу топологии с использованием результатов вычислений, основанных на алгоритме кратчайшего пути (SPF) Дейкстры. Алгоритм поиска кратчайшего пути основывается на данных о совокупной стоимости доступа к точке назначения. Стоимость доступа определятся на основе скорости интерфейса. 

Чтобы повысить эффективность и масштабируемость OSPF, протокол поддерживает иерархическую маршрутизацию с помощью областей (area). 

**Область OSPF (area)** — Часть сети, которой ограничивается формирование базы данных о состоянии каналов. Маршрутизаторы, находящиеся в одной и той же области, имеют одну и ту же базу данных о топологии сети. Для определения областей применяются идентификаторы областей.

Протоколы OSPF бвывают 2-х версий: 
    • OSPFv2
    • OSPFv3

Основным отличием протоколов является то, что OSPFv2 работает с IPv4, а OSPFv3 — c IPv6. 
Маршрутизаторы в OSPF клаlссифицируются на основе выполняемой ими функции:
![[Image 1.png]]

- **Internal router (внутренний маршрутизатор)** — маршрутизатор, все интерфейсы которого находятся в одной и той же области. 
- **Backbone router (магистральный маршрутизатор)** — это маршрутизатор, который находится в магистральной зоне (area 0).
- **ABR (пограничный маргрутизатор области)** — маршрутизатор, интерфейсы которого подключены к разным областям.
- **ASBR (Граничный маршрутизатор автономной системы)** — это маршрутизатор, у которого интерфейс подключен к внешней сети.

Также с помощью OSPF можно настроить ассиметричный роутинг.

**Ассиметричная маршрутизация** — возможность пересекать сеть в одном направлении, используя один путь, и возвращаться через другой путь.

# Практическая часть

## Архитектура сети
Разворачиваем 3 виртуальные машины.

Так как мы планируем настроить OSPF, все 3 виртуальные машины должны быть соединены между собой (разными VLAN), а также иметь одну (или несколько) доолнительных сетей, к которым, далее OSPF сформирует маршруты. Исходя из данных требований, мы можем нарисовать топологию сети:
![[Pasted image 20261108225133.png]]

```ad-info
Обратим внимание, сети, указанные на схеме не должны использоваться в Oracle Virtualbox, иначе Vagrant не сможет собрать стенд и зависнет. 
По умолчанию Virtualbox использует сеть `10.0.2.0/24`. 
Если была настроена другая сеть, то проверить её можно в настройках программы: **VirtualBox — File — Preferences — Network** — щёлкаем по созданной сети
```

![[Image 2.png]]

## vagrantfile

Создаём каталог, в котором будут храниться настройки виртуальной машины. В каталоге создаём файл с именем vagrantfile, добавляем в него следующее содержимое:

```ruby
MACHINES = {
  :router1 => {
    :box_name => "ubuntu/focal64",
    :vm_name => "router1",
    :net => [
      {ip: '10.0.10.1', adapter: 2, netmask: "255.255.255.252", virtualbox__intnet: "r1-r2"},
      {ip: '10.0.12.1', adapter: 3, netmask: "255.255.255.252", virtualbox__intnet: "r1-r3"},
      {ip: '192.168.10.1', adapter: 4, netmask: "255.255.255.0", virtualbox__intnet: "net1"},
      {ip: '192.168.50.10', adapter: 5},
    ]
  },

  :router2 => {
    :box_name => "ubuntu/focal64",
    :vm_name => "router2",
    :net => [
      {ip: '10.0.10.2', adapter: 2, netmask: "255.255.255.252", virtualbox__intnet: "r1-r2"},
      {ip: '10.0.11.2', adapter: 3, netmask: "255.255.255.252", virtualbox__intnet: "r2-r3"},
      {ip: '192.168.20.1', adapter: 4, netmask: "255.255.255.0", virtualbox__intnet: "net2"},
      {ip: '192.168.50.11', adapter: 5},
    ]
  },

  :router3 => {
    :box_name => "ubuntu/focal64",
    :vm_name => "router3",
    :net => [
      {ip: '10.0.11.1', adapter: 2, netmask: "255.255.255.252", virtualbox__intnet: "r2-r3"},
      {ip: '10.0.12.2', adapter: 3, netmask: "255.255.255.252", virtualbox__intnet: "r1-r3"},
      {ip: '192.168.30.1', adapter: 4, netmask: "255.255.255.0", virtualbox__intnet: "net3"},
      {ip: '192.168.50.12', adapter: 5},
    ]
  }

}

Vagrant.configure("2") do |config|

  MACHINES.each do |boxname, boxconfig|

    config.vm.define boxname do |box|

      box.vm.box = boxconfig[:box_name]
      box.vm.host_name = boxconfig[:vm_name]

      if boxconfig[:vm_name] == "router3"
        box.vm.provision "ansible" do |ansible|
          ansible.playbook = "ansible/provision.yml"
          ansible.inventory_path = "ansible/hosts"
          ansible.host_key_checking = "false"
          ansible.limit = "all"
        end
      end

      boxconfig[:net].each do |ipconf|
        box.vm.network "private_network", **ipconf
      end

    end
  end
end
```

Результатом выполнения данного vagrantfile будут 3 созданные виртуальные машины, которые соединены между собой сетями (10.0.10.0/30, 10.0.11.0/30 и 10.0.12.0/30).
У каждого роутера есть дополнительная сеть:
- на router1 — 192.168.10.0/24
- на router2 — 192.168.20.0/24
- на router3 — 192.168.30.0/24

```ad-info
Заметим что запуск ansible скрипта начинается после того как закончится конфигурация `router3`. Это прописано в в конце vagrantfile.
```

## ansible

Для настроки хостов с помощью Ansible нам нужно создать несколько файлов и положить их в отдельную папку (в нашем примере имя папки ansible): 

**Конфигурационный файл: ansible.cfg** — файл описывает базовые настройки для работы Ansible:
```
[defaults]
#Отключение проверки ключа хоста
host_key_checking = false
#Указываем имя файла инвентаризации
inventory = hosts
#Отключаем игнорирование предупреждений
command_warnings= false
```

**Файл инвентаризации hosts** — данный файл хранит информацию о том, как подключиться к хосту:
```
[routers]
router1 ansible_host=192.168.50.10 ansible_user=vagrant ansible_ssh_private_key_file=.vagrant/machines/router1/virtualbox/private_key 
router2 ansible_host=192.168.50.11 ansible_user=vagrant ansible_ssh_private_key_file=.vagrant/machines/router2/virtualbox/private_key 
router3 ansible_host=192.168.50.12 ansible_user=vagrant ansible_ssh_private_key_file=.vagrant/machines/router3/virtualbox/private_key
``` 

`[servers]` - в квадратных скобках указана группа хостов
`router1` — имя нашего хоста (имена хостов и групп не могут быть одинаковые)
`ansible_host` — адрес нашего хоста
`ansible_user` — имя пользователя, с помощью которого Ansible будет подключаться к хосту
`ansible_ssh_private_key` — адрес расположения ssh-ключа

В файл инвентаризации также можно добовлять переменные, которые могут автоматически добавляться в `jinja template`. Добавление переменных рассмотрим далее. 

**Ansible-playbook `provision.yml`** — основной файл, в котором содержатся инструкции (модули) по настройке для Ansible. 

Дополнительно создадим каталоги для темплейтов конфигурационных файлов **`templates`** и файлов с переменными **`defaults`**

### Установка пакетов для тестирования и настройки OSPF

Перед настройкой FRR рекомендуется поставить базовые программы для изменения конфигурационных файлов (vim) и изучения сети (traceroute, tcpdump, net-tools):

  ```bash
apt update
apt install vim traceroute tcpdump net-tools
  ```

В конфигурационном файле `provision.yml` это будет выглядить следующим образом:
```ruby
#Начало файла provision.yml
- name: OSPF
  #Указываем имя хоста или группу, которые будем настраивать
  hosts: all
  #Параметр выполнения модулей от root-пользователя
  become: yes
  #Указание файла с дополнителыми переменными (понадобится при добавлении темплейтов)
  vars_files:
    - defaults/main.yml
  tasks:
    # Обновление пакетов и установка vim, traceroute, tcpdump, net-tools
    - name: install base tools
      apt:
        name:
          - vim
          - traceroute
          - tcpdump
          - net-tools
        state: present
        update_cache: true
```

### Настройка OSPF между машинами на базе FRR.

***Процесс установки FRR и настройки OSPF вручную:***

- **Отключаем файерволл ufw и удаляем его из автозагрузки:**
```bash
systemctl stop ufw 
systemctl disable ufw
```

- **Добавляем gpg ключ:**
```bash
curl -s https://deb.frrouting.org/frr/keys.asc | sudo apt-key add -
```

- **Добавляем репозиторий c пакетом FRR:**
```bash
echo deb https://deb.frrouting.org/frr $(lsb_release -s -c) frr-stable > /etc/apt/sources.list.d/frr.list
```

- **Обновляем пакеты и устанавливаем FRR:**
```bash
sudo apt update
sudo apt install frr frr-pythontools
```

- **Разрешаем (включаем) маршрутизацию транзитных пакетов:**
```bash
sysctl net.ipv4.conf.all.forwarding=1
```

- **Включаем демон ospfd в FRR**
Для этого открываем в редакторе файл `/etc/frr/daemons` и меняем в нём параметры для пакетов `zebra` и `ospfd` на `yes`:

(Показана часть файла в которой необходимо произвести изменения)

```bash
nano /etc/frr/daemons
```

```
zebra=yes
ospfd=yes
bgpd=no
ospf6d=no
ripd=no
ripngd=no
isisd=no
pimd=no
ldpd=no
nhrpd=no
eigrpd=no
babeld=no
sharpd=no
pbrd=no
bfdd=no
fabricd=no
vrrpd=no
pathd=no
```

- **Настройка OSPF**
Для настройки OSPF нам потребуется создать файл `/etc/frr/frr.conf` который будет содержать в себе информацию о требуемых интерфейсах и OSPF. Разберем пример создания файла на хосте router1. 

Для начала нам необходимо узнать имена интерфейсов и их адреса. Сделать это можно с помощью двух способов:

**Посмотреть в linux:** 
```bash
ip a | grep inet
```

Вывод:
```bash
root@router1:~# ip a | grep inet
    inet 127.0.0.1/8 scope host lo
    inet6 ::1/128 scope host 
    inet 10.0.2.15/24 metric 100 brd 10.0.2.255 scope global dynamic enp0s3
    inet6 fd17:625c:f037:2:1a:a2ff:fe1e:9db7/64 scope global dynamic mngtmpaddr noprefixroute 
    inet6 fe80::1a:a2ff:fe1e:9db7/64 scope link 
    inet 10.0.10.1/30 brd 10.0.10.3 scope global enp0s8
    inet6 fe80::a00:27ff:fe5c:a87/64 scope link 
    inet 10.0.12.1/30 brd 10.0.12.3 scope global enp0s9
    inet6 fe80::a00:27ff:fe58:e517/64 scope link 
    inet 192.168.10.1/24 brd 192.168.10.255 scope global enp0s10
    inet6 fe80::a00:27ff:fe3c:c7d7/64 scope link 
    inet 192.168.50.10/24 brd 192.168.50.255 scope global enp0s16
    inet6 fe80::a00:27ff:fe36:cc41/64 scope link 
```

**Зайти в интерфейс FRR и посмотреть информацию об интерфейсах:**
```bash
root@router1:~# vtysh

Hello, this is FRRouting (version 10.7.1).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

router1# show interface brief
Interface       Status  VRF             Addresses
---------       ------  ---             ---------
enp0s3          up      default         10.0.2.15/24
                                        + fd17:625c:f037:2:1a:a2ff:fe1e:9db7/64
enp0s8          up      default         10.0.10.1/30
                                        fe80::a00:27ff:fe5c:a87/64
enp0s9          up      default         10.0.12.1/30
                                        fe80::a00:27ff:fe58:e517/64
enp0s10         up      default         192.168.10.1/24
                                        fe80::a00:27ff:fe3c:c7d7/64
enp0s16         up      default         192.168.50.10/24
                                        fe80::a00:27ff:fe36:cc41/64
lo              up      default         

router1# exit

```

В обоих примерах мы увидем имена сетевых интерфейсов, их ip-адреса и маски подсети. Исходя из схемы мы понимаем, что для настройки OSPF нам достаточно описать интерфейсы `enp0s8`, `enp0s9`, `enp0s10`.

Создаём файл `/etc/frr/frr.conf` и вносим в него следующую информацию:

Пример файла на `router1`:
```bash
!Указание версии FRR
frr version 8.1
frr defaults traditional
!Указываем имя машины
hostname router1
log syslog informational
no ipv6 forwarding
service integrated-vtysh-config
!
!Добавляем информацию об интерфейсе enp0s8
interface enp0s8
 !Указываем имя интерфейса
 description r1-r2
 !Указываем ip-aдрес и маску (эту информацию мы получили в прошлом шаге)
 ip address 10.0.10.1/30
 !Указываем параметр игнорирования MTU
 ip ospf mtu-ignore
 !Если потребуется, можно указать «стоимость» интерфейса
 !ip ospf cost 1000
 !Указываем параметры hello-интервала для OSPF пакетов
 ip ospf hello-interval 10
 !Указываем параметры dead-интервала для OSPF пакетов
 !Должно быть кратно предыдущему значению
 ip ospf dead-interval 30
!
interface enp0s9
 description r1-r3
 ip address 10.0.12.1/30
 ip ospf mtu-ignore
 !ip ospf cost 45
 ip ospf hello-interval 10
 ip ospf dead-interval 30

interface enp0s10
 description net_router1
 ip address 192.168.10.1/24
 ip ospf mtu-ignore
 !ip ospf cost 45
 ip ospf hello-interval 10
 ip ospf dead-interval 30 
!
!Начало настройки OSPF
router ospf
 !Указываем router-id 
 router-id 1.1.1.1
 !Указываем сети, которые хотим анонсировать соседним роутерам
 network 10.0.10.0/30 area 0
 network 10.0.12.0/30 area 0
 network 192.168.10.0/24 area 0 
 !Указываем адреса соседних роутеров
 neighbor 10.0.10.2
 neighbor 10.0.12.2
 default-information originate always

!Указываем адрес log-файла
log file /var/log/frr/frr.log

```

Сохраняем изменения и выходим из данного файла. 

Вместо файла `frr.conf` мы можем задать данные параметры вручную из vtysh. Vtysh использует cisco-like команды.
 
```ad-info
На хостах `router2` и `router3` также потребуется настроить конфигруационные файлы, предварительно поменяв ip -адреса интерфейсов. 
```

В ходе создания файла мы видим несколько OSPF-параметров, которые требуются для настройки:
- **hello-interval** — интервал который указывает через сколько секунд протокол OSPF будет повторно отправлять запросы на другие роутеры. Данный интервал должен быть одинаковый на всех портах и роутерах, между которыми настроен OSPF. 
- **Dead-interval** — если в течении заданного времени роутер не отвечает на запросы, то он считается вышедшим из строя и пакеты уходят на другой роутер (если это возможно). Значение должно быть кратно hello-интервалу. Данный интервал должен быть одинаковый на всех портах и роутерах, между которыми настроен OSPF.
- **router-id** — идентификатор маршрутизатора (необязательный параметр), если данный параметр задан, то роутеры определяют свои роли по  данному параметру. Если данный идентификатор не задан, то роли маршрутизаторов определяются с помощью Loopback-интерфейса или самого большого ip-адреса на роутере.

После создания файлов `/etc/frr/frr.conf` и `/etc/frr/daemons` нужно проверить, что владельцем файла является пользователь frr. Группа файла также должна быть frr. Должны быть установленны следующие права:
у владельца на чтение и запись
у группы только на чтение
```bash
ls -l /etc/frr
```

Если права или владелец файла указан неправильно, то нужно поменять владельца и назначить правильные права, например:
```bash
chown frr:frr /etc/frr/frr.conf 
chmod 640 /etc/frr/frr.conf
``` 

Перезапускаем FRR и добавляем его в автозагрузку
```bash
systemct restart frr 
systemctl enable frr
```

Проверям, что OSPF перезапустился без ошибок:

```bash
root@router1:~# systemctl status frr
● frr.service - FRRouting
     Loaded: loaded (/lib/systemd/system/frr.service; enabled; vendor preset: enabled)
     Active: active (running) since Mon 2026-11-09 08:20:22 UTC; 6h ago
       Docs: https://frrouting.readthedocs.io/en/latest/setup.html
    Process: 5917 ExecStart=/usr/lib/frr/frrinit.sh start (code=exited, status=0/SUCCESS)
   Main PID: 5933 (watchfrr)
     Status: "FRR Operational"
      Tasks: 10 (limit: 1117)
     Memory: 21.7M
     CGroup: /system.slice/frr.service
             ├─5933 /usr/lib/frr/watchfrr -d mgmtd zebra ospfd staticd
             ├─5944 /usr/lib/frr/mgmtd -d -F traditional -A 127.0.0.1
             ├─5947 /usr/lib/frr/zebra -d -F traditional -A 127.0.0.1 -s 90000000
             ├─5952 /usr/lib/frr/ospfd -d -F traditional -A 127.0.0.1
             └─5955 /usr/lib/frr/staticd -d -F traditional -A 127.0.0.1

Nov 09 08:20:22 router1 ospfd[5952]: [VTVCM-Y2NW3] Configuration Read in Took: 00:00:00
Nov 09 08:20:22 router1 frrinit.sh[5960]: [5960|mgmtd] done
Nov 09 08:20:22 router1 frrinit.sh[5964]: [5964|ospfd] done
Nov 09 08:20:22 router1 watchfrr[5933]: [QDG3Y-BY5TN] mgmtd state -> up : connect succeeded
Nov 09 08:20:22 router1 watchfrr[5933]: [QDG3Y-BY5TN] zebra state -> up : connect succeeded
Nov 09 08:20:22 router1 watchfrr[5933]: [QDG3Y-BY5TN] ospfd state -> up : connect succeeded
Nov 09 08:20:22 router1 watchfrr[5933]: [QDG3Y-BY5TN] staticd state -> up : connect succee>
Nov 09 08:20:22 router1 watchfrr[5933]: [KWE5Q-QNGFC] all daemons up, doing startup-comple>
Nov 09 08:20:22 router1 frrinit.sh[5917]:  * Started watchfrr
Nov 09 08:20:22 router1 systemd[1]: Started FRRouting.
```

Если мы правильно настроили OSPF, то с любого хоста нам должны быть доступны сети:
192.168.10.0/24
192.168.20.0/24
192.168.30.0/24
10.0.10.0/30 
10.0.11.0/30
10.0.12.0/30


Проверим доступность сетей с хоста router1:

попробуем сделать ping до ip-адреса 192.168.30.1
```bash
root@router1:~# ping 192.168.30.1
PING 192.168.30.1 (192.168.30.1) 56(84) bytes of data.
64 bytes from 192.168.30.1: icmp_seq=1 ttl=64 time=1.32 ms
64 bytes from 192.168.30.1: icmp_seq=2 ttl=64 time=1.15 ms
^C
--- 192.168.30.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1005ms
rtt min/avg/max/mdev = 1.146/1.234/1.322/0.088 ms
root@router1:~# 
```

Запустим трассировку до адреса 192.168.30.1
```bash
root@router1:~# traceroute 192.168.30.1
traceroute to 192.168.30.1 (192.168.30.1), 30 hops max, 60 byte packets
 1  192.168.30.1 (192.168.30.1)  0.997 ms  0.868 ms  0.813 ms
root@router1:~# 
```
Попробуем отключить интерфейс enp0s9 и немного подождем и снова запустим трассировку до ip-адреса 192.168.30.1
```bash
root@router1:~# ifconfig enp0s9 down
root@router1:~# ip a | grep enp0s9 
4: enp0s9: <BROADCAST,MULTICAST> mtu 1500 qdisc fq_codel state DOWN group default qlen 1000
root@router1:~# traceroute 192.168.30.1
traceroute to 192.168.30.1 (192.168.30.1), 30 hops max, 60 byte packets
 1  10.0.10.2 (10.0.10.2)  0.522 ms  0.479 ms  0.460 ms
 2  192.168.30.1 (192.168.30.1)  0.796 ms  0.777 ms  0.644 ms
root@router1:~# 
```

Как мы видим, после отключения интерфейса сеть 192.168.30.0/24 нам остаётся доступна.

Также мы можем проверить из интерфейса vtysh какие маршруты мы видим на данный момент:
```bash
root@router1:~# 
root@router1:~# vtysh

Hello, this is FRRouting (version 8.1).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

router1# show ip route ospf
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, F - PBR,
       f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

O   10.0.10.0/30 [110/1000] is directly connected, enp0s8, weight 1, 02:50:21
O>* 10.0.11.0/30 [110/200] via 10.0.12.2, enp0s9, weight 1, 00:01:00
O   10.0.12.0/30 [110/100] is directly connected, enp0s9, weight 1, 00:01:00
O   192.168.10.0/24 [110/100] is directly connected, enp0s10, weight 1, 02:50:21
O>* 192.168.20.0/24 [110/300] via 10.0.10.2, enp0s9, weight 1, 00:01:00
O>* 192.168.30.0/24 [110/200] via 10.0.12.2, enp0s9, weight 1, 00:01:00
router1# 
```

- **Пример настройки FRR через ansible:**
Часть конфига `provision.yml` в котором будет воспроизводиться то что было описано выше.
```ruby
#Отключаем UFW и удаляем его из автозагрузки
    - name: disable ufw service
      service:
        name: ufw
        state: stopped
        enabled: false

    # Добавляем gpg-key репозитория
    - name: add gpg frrouting.org
      apt_key:
        url: "https://deb.frrouting.org/frr/keys.asc"
        state: present

    # Добавляем репозиторий https://deb.frrouting.org/frr
    - name: add frr repo
      apt_repository:
        repo: 'deb https://deb.frrouting.org/frr {{ ansible_distribution_release }} frr-stable'
        state: present

    # Обновляем пакеты и устанавливаем FRR
    - name: install FRR packages
      apt:
        name:
          - frr
          - frr-pythontools
        state: present
        update_cache: true

    # Включаем маршрутизацию транзитных пакетов
    - name: set up forward packages across routers
      sysctl:
        name: net.ipv4.conf.all.forwarding
        value: '1'
        state: present

    # Отключаем запрет ассиметричного роутинга 
    - name: set up asynchronous routing
      sysctl:
        name: net.ipv4.conf.all.rp_filter
        value: '0'
        state: present


    # Копируем файл daemons на хосты, указываем владельца и права
    - name: base set up OSPF
      template:
        src: daemons
        dest: /etc/frr/daemons
        owner: frr
        group: frr
        mode: 0640

    # Копируем файл frr.conf на хосты, указываем владельца и права
    - name: set up OSPF
      template:
        src: frr.conf.j2
        dest: /etc/frr/frr.conf
        owner: frr
        group: frr
        mode: 0640

    # Перезапускам FRR и добавляем в автозагрузку
    - name: restart FRR
      service:
        name: frr
        state: restarted
        enabled: true
```


Файлы daemons и `frr.conf` должны лежать в каталоге `ansible/template`. 
Давайте подробнее рассмотрим эти файлы. 
```ad-info
Содержимое файла `daemons` одинаково на всех хостах, а вот содержание файла `frr.conf` на всех хостах будет разное.
```

**==Для того, чтобы не создавать 3 похожих файла, можно воспользоваться jinja2 template.==** Jinja2 позволит добавлять в файл уникальные значения для разных серверов. 

Перед тем, как начать настройку хостов Ansible забирает информацию о каждом хосте. Эту информацию можно использовать в jinja2 темплейтах. 

Посмотреть, какую информацию о сервере собрал Ansible можно с помощью команды:
```bash
ansible router1 -i ansible/hosts -m setup -e "host_key_checking = false"
```
Команда выполнятся из каталога проекта (где находится Vagrantfile).

Помимо фактов Jinja2 может брать информацию из файлов hosts и defaults/main.yml (так как мы указали его в начале файла provision.yml)

- **Как мы использовали jinja2 на примере frr.conf**

Создадим файл с названием `./ansible/templates/frr.conf.j2`

Вставим в него следующее содержимое:

```bash
!Указание версии FRR
frr version 8.1
frr defaults traditional
!Указываем имя машины
hostname {{ ansible_hostname }}
log syslog informational
no ipv6 forwarding
service integrated-vtysh-config
!
!Добавляем информацию об интерфейсе enp0s8
interface enp0s8
 !Указываем имя интерфейса
 description r1-r2
 !Указываем ip-aдрес и маску 
 ip address {{ ansible_enp0s8.ipv4.address }}/30
 !Указываем параметр игнорирования MTU
 ip ospf mtu-ignore
 !Если потребуется, можно указать «стоимость» интерфейса
 {% if ansible_hostname == 'router1' %}
 ip ospf cost 1000
 {% elif ansible_hostname == 'router2' and symmetric_routing == true %}
 ip ospf cost 1000
 {% else %}
 !ip ospf cost 450
 {% endif %}
 !Указываем параметры hello-интервала для OSPF пакетов
 ip ospf hello-interval 10
 !Указываем параметры dead-интервала для OSPF пакетов
 !Должно быть кратно предыдущему значению
 ip ospf dead-interval 30
!
interface enp0s9
 description r1-r3
 ip address {{ ansible_enp0s9.ipv4.address }}/30
 ip ospf mtu-ignore
 !ip ospf cost 45
 ip ospf hello-interval 10
 ip ospf dead-interval 30

interface enp0s10
 description net_router1
 ip address {{ ansible_enp0s10.ipv4.address }}/24
 ip ospf mtu-ignore
 !ip ospf cost 45
 ip ospf hello-interval 10
 ip ospf dead-interval 30 
!
!Начало настройки OSPF
router ospf
 !Указываем router-id 
 {% if router_id_enable == false %}!{% endif %}router-id {{ router_id }}
 !Указываем сети, которые хотим анонсировать соседним роутерам
  {% if ansible_hostname == 'router1' %}
! Блок для router1
 network 10.0.10.0/30 area 0
 network 10.0.12.0/30 area 0
 network 192.168.10.0/24 area 0
{% elif ansible_hostname == 'router2' %}
! Блок для router2
 network 10.0.10.0/30 area 0
 network 10.0.11.0/30 area 0
 network 192.168.20.0/24 area 0
{% elif ansible_hostname == 'router3' %}
! Блок для router3
 network 10.0.11.0/30 area 0
 network 10.0.12.0/30 area 0
 network 192.168.30.0/24 area 0
{% else %}
# Значения по умолчанию
{% endif %}
 !Указываем адреса соседних роутеров
 {% if ansible_hostname == 'router1' %}
! Блок для router1
 neighbor 10.0.10.2
 neighbor 10.0.12.2
{% elif ansible_hostname == 'router2' %}
! Блок для router2
 neighbor 10.0.10.1
 neighbor 10.0.11.1
{% elif ansible_hostname == 'router3' %}
! Блок для router3
 neighbor 10.0.12.1
 neighbor 10.0.11.2
{% else %}
# Значения по умолчанию
{% endif %}
 default-information originate always
!Указываем адрес log-файла
log file /var/log/frr/frr.log

```

Объяснение:
Пример того как мы берем информацию из конфигурации создаваемой машины и вставляем ее в нужных местах. Таким образом у нас получается конфиг со значениями для конкретной машины.
```bash
!Указываем имя машины
hostname {{ ansible_hostname }}
```

```bash
!Указываем ip-aдрес и маску 
 ip address {{ ansible_enp0s8.ipv4.address }}/30
```


Допустим мы хотим перед настройкой OSPF выбрать, будем ли мы указывать router-id. Для этого в файле `defaults/main.yml` создаем переменную 
```bash
router_id_enable: false
``` 
По умолчанию назначаем ей значение `false`
Значение `router_id` мы можем задать в файле hosts, дописав его в конец строки наших хостов:
```bash
[routers]
router1 ansible_host=192.168.50.10 ansible_user=vagrant ansible_ssh_private_key_file=.vagrant/machines/router1/virtualbox/private_key router_id=1.1.1.1
router2 ansible_host=192.168.50.11 ansible_user=vagrant ansible_ssh_private_key_file=.vagrant/machines/router2/virtualbox/private_key router_id=2.2.2.2
router3 ansible_host=192.168.50.12 ansible_user=vagrant ansible_ssh_private_key_file=.vagrant/machines/router3/virtualbox/private_key router_id=3.3.3.3
```

```bash
!Указываем router-id 
 {% if router_id_enable == false %}!{% endif %}router-id {{ router_id }}
```

В этом случае ansible подставит в frr.conf значение `router-id` из файла `hosts`

```ad-note
Данное правило в любом случае будет подставлять значение `router-id`, но, если параметр `router_id_enable` будет равен значению `false`, то в начале строки будет ставиться символ `!` (восклицательный знак) и строка не будет учитываться в настройках FRR.
```

- **Настройка ассиметричного роутинга**

Для настройки ассиметричного роутинга нам необходимо выключить блокировку ассиметричной маршрутизации: 
```bash
sysctl net.ipv4.conf.all.rp_filter=0
```

Далее, выбираем один из роутеров, на котором изменим «стоимость интерфейса». Например поменяем стоимость интерфейса `enp0s8` на `router1`:
```bash
root@router1:~# vtysh

Hello, this is FRRouting (version 8.1).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

router1# conf t
router1(config)# int enp0s8 
router1(config-if)# ip ospf cost 1000
router1(config-if)# exit
router1(config)# exit
router1# show ip route ospf
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, F - PBR,
       f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

O   10.0.10.0/30 [110/300] via 10.0.12.2, enp0s9, weight 1, 00:02:24
O>* 10.0.11.0/30 [110/200] via 10.0.12.2, enp0s9, weight 1, 00:02:29
O   10.0.12.0/30 [110/100] is directly connected, enp0s9, weight 1, 00:02:29
O   192.168.10.0/24 [110/100] is directly connected, enp0s10, weight 1, 00:03:04
O>* 192.168.20.0/24 [110/300] via 10.0.12.2, enp0s9, weight 1, 00:02:24
O>* 192.168.30.0/24 [110/200] via 10.0.12.2, enp0s9, weight 1, 00:02:29
router1# 


router2# show ip route ospf 
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, F - PBR,
       f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

O   10.0.10.0/30 [110/100] is directly connected, enp0s8, weight 1, 00:00:09
O   10.0.11.0/30 [110/100] is directly connected, enp0s9, weight 1, 00:34:11
O>* 10.0.12.0/30 [110/200] via 10.0.10.1, enp0s8, weight 1, 00:00:09
  *                        via 10.0.11.1, enp0s9, weight 1, 00:00:09
O>* 192.168.10.0/24 [110/200] via 10.0.10.1, enp0s8, weight 1, 00:00:09
O   192.168.20.0/24 [110/100] is directly connected, enp0s10, weight 1, 00:34:11
O>* 192.168.30.0/24 [110/200] via 10.0.11.1, enp0s9, weight 1, 00:33:36
router2# 
```

После внесения данных настроек, мы видим, что маршрут до сети 192.168.20.0/30  теперь пойдёт через router2, но обратный трафик от router2 пойдёт по другому пути. Давайте это проверим:
- На router1 запускаем пинг от 192.168.10.1 до 192.168.20.1: 
```bash
ping -I 192.168.10.1 192.168.20.1
```

- На `router2` запускаем `tcpdump`, который будет смотреть трафик только на порту `enp0s9`:
```bash
root@router2:~# tcpdump -i enp0s9
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on enp0s9, link-type EN10MB (Ethernet), capture size 262144 bytes
19:03:00.185258 IP 192.168.10.1 > router2: ICMP echo request, id 6, seq 108, length 64
19:03:01.186977 IP 192.168.10.1 > router2: ICMP echo request, id 6, seq 109, length 64
19:03:02.188563 IP 192.168.10.1 > router2: ICMP echo request, id 6, seq 110, length 64
19:03:02.540289 IP router2 > ospf-all.mcast.net: OSPFv2, Hello, length 48
19:03:02.542198 IP 10.0.11.1 > ospf-all.mcast.net: OSPFv2, Hello, length 48
19:03:03.189952 IP 192.168.10.1 > router2: ICMP echo request, id 6, seq 111, length 64
```
Видим что данный порт только получает ICMP-трафик с адреса 192.168.10.1

- На router2 запускаем tcpdump, который будет смотреть трафик только на порту enp0s8:

```bash
root@router2:~# tcpdump -i enp0s8
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on enp0s8, link-type EN10MB (Ethernet), capture size 262144 bytes
19:05:24.410547 IP router2 > 192.168.10.1: ICMP echo reply, id 6, seq 248, length 64
19:05:25.461411 IP router2 > 192.168.10.1: ICMP echo reply, id 6, seq 249, length 64
19:05:26.496036 IP router2 > 192.168.10.1: ICMP echo reply, id 6, seq 250, length 64
19:05:27.498524 IP router2 > 192.168.10.1: ICMP echo reply, id 6, seq 251, length 64
```
Видим что данный порт только отправляет ICMP-трафик на адрес 192.168.10.1

Таким образом мы видим ассиметричный роутинг.

Часть где мы включаем ассимитричный роутинг с помошью `ansible`

```ruby
# Отключаем запрет ассиметричного роутинга 
    - name: set up asynchronous routing
      sysctl:
        name: net.ipv4.conf.all.rp_filter
        value: '0'
        state: present


    # Копируем файл daemons на хосты, указываем владельца и права
    - name: base set up OSPF
      template:
        src: daemons
        dest: /etc/frr/daemons
        owner: frr
        group: frr
        mode: 0640

    # Копируем файл frr.conf на хосты, указываем владельца и права
    - name: set up OSPF
      template:
        src: frr.conf.j2
        dest: /etc/frr/frr.conf
        owner: frr
        group: frr
        mode: 0640

    # Перезапускам FRR и добавляем в автозагрузку
    - name: restart FRR
      service:
        name: frr
        state: restarted
        enabled: true
```

Блок в `frr.conf` который будет отвечать за подстановку нужной "стоимости" интерфейса в зависимости от хоста:
```bash
!Если потребуется, можно указать «стоимость» интерфейса
 {% if ansible_hostname == 'router1' %}
 ip ospf cost 1000
 {% elif ansible_hostname == 'router2' and symmetric_routing == true %}
 ip ospf cost 1000
 {% else %}
 !ip ospf cost 450
 {% endif %}
```
Данное условие проверят имя хоста и переменную symmetric_routing и добавляет в темплейт следующие параметры:
- Если имя хоста router1 — то добавляется стоимость интерфейса 1000
- Если имя хоста router2 И значение параметра symmetric_routing true — то добавляется стоимость интерфейса 1000
В остальных случаях добавляется закомментированный параметр

Чтобы было легко переключаться между ассиметричным и симметричным роутингом, мы можем добавить переменную symmetric_routing со значением по умолчанию false в файл defaults/main.yml
symmetric_routing: false

- **Симметричный роутинг вручную:**
Настройка симметичного роутинга

Так как у нас уже есть один «дорогой» интерфейс, нам потребуется добавить ещё один дорогой интерфейс, чтобы у нас перестала работать ассиметричная маршрутизация. 

Так как в прошлом примере мы заметили что router2 будет отправлять обратно трафик через порт enp0s8, мы также должны сделать его дорогим и далее проверить, что теперь используется симметричная маршрутизация:

Поменяем стоимость интерфейса enp0s8 на router2:
```bash
router2# conf t
router2(config)# int enp0s8
router2(config-if)# ip ospf cost 1000
router2(config-if)# exit
router2(config)# exit
router2# 
router2# show ip route ospf
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, F - PBR,
       f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

O   10.0.10.0/30 [110/1000] is directly connected, enp0s8, weight 1, 00:00:13
O   10.0.11.0/30 [110/100] is directly connected, enp0s9, weight 1, 00:29:53
O>* 10.0.12.0/30 [110/200] via 10.0.11.1, enp0s9, weight 1, 00:00:13
O>* 192.168.10.0/24 [110/300] via 10.0.11.1, enp0s9, weight 1, 00:00:13
O   192.168.20.0/24 [110/100] is directly connected, enp0s10, weight 1, 00:29:53
O>* 192.168.30.0/24 [110/200] via 10.0.11.1, enp0s9, weight 1, 00:29:18
router2# 
router2# exit 
root@router2:~# 
```

После внесения данных настроек, мы видим, что маршрут до сети 192.168.10.0/30  пойдёт через router2.

Давайте это проверим:
- На router1 запускаем пинг от 192.168.10.1 до 192.168.20.1: 
```bash
ping -I 192.168.10.1 192.168.20.1
```

- На router2 запускаем tcpdump, который будет смотреть трафик только на порту enp0s9:
```bash
root@router2:~# tcpdump -i enp0s9
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on enp0s9, link-type EN10MB (Ethernet), capture size 262144 bytes
19:30:28.551713 IP 192.168.10.1 > router2: ICMP echo request, id 6, seq 1737, length 64
19:30:28.551801 IP router2 > 192.168.10.1: ICMP echo reply, id 6, seq 1737, length 64
19:30:29.553801 IP 192.168.10.1 > router2: ICMP echo request, id 6, seq 1738, length 64
19:30:29.553927 IP router2 > 192.168.10.1: ICMP echo reply, id 6, seq 1738, length 64
19:30:30.555858 IP 192.168.10.1 > router2: ICMP echo request, id 6, seq 1739, length 64
19:30:30.555930 IP router2 > 192.168.10.1: ICMP echo reply, id 6, seq 1739, length 64
19:30:31.557504 IP 192.168.10.1 > router2: ICMP echo request, id 6, seq 1740, length 64
19:30:31.557573 IP router2 > 192.168.10.1: ICMP echo reply, id 6, seq 1740, length 64
19:30:32.559191 IP 192.168.10.1 > router2: ICMP echo request, id 6, seq 1741, length 64
19:30:32.559260 IP router2 > 192.168.10.1: ICMP echo reply, id 6, seq 1741, length 64

```
Теперь мы видим, что трафик между роутерами ходит симметрично.

Для удобного переключения параметров нам потребуется запускать из ansible-playbook только 2 последних модуля. Чтобы не ждать выполнения всего плейбука, можно добавить тег к модулям:
```ruby
- name: set up OSPF 
    template:
      src: frr.conf.j2
      dest: /etc/frr/frr.conf
      owner: frr
      group: frr
      mode: 0640
    tags:
      - setup_ospf

  - name: restart FRR
    service:
      name: frr
      state: restarted
      enabled: true
    tags:
      - setup_ospf
```

Тогда можно будет запускать playbook не полностью. Пример запуска модулей из ansible-playbook, которые помечены тегами:
```bash
ansible-playbook -i ansible/hosts -l all ansible/provision.yml -t setup_ospf -e "host_key_checking=false"
```

# Готово

В конце продублирую содержимое всех нужных конфигов и файлов.

Структура стенда:
```
.
├── ansible
│   ├── ansible.cfg
│   ├── defaults
│   │   └── main.yml
│   ├── hosts
│   ├── provision.yml
│   └── templates
│       ├── daemons
│       └── frr.conf.j2
└── vagrantfile

4 directories, 7 files
```

**`./vagrantfile`**

```ruby
MACHINES = {
  :router1 => {
    :box_name => "ubuntu/focal64",
    :vm_name => "router1",
    :net => [
      {ip: '10.0.10.1', adapter: 2, netmask: "255.255.255.252", virtualbox__intnet: "r1-r2"},
      {ip: '10.0.12.1', adapter: 3, netmask: "255.255.255.252", virtualbox__intnet: "r1-r3"},
      {ip: '192.168.10.1', adapter: 4, netmask: "255.255.255.0", virtualbox__intnet: "net1"},
      {ip: '192.168.50.10', adapter: 5},
    ]
  },

  :router2 => {
    :box_name => "ubuntu/focal64",
    :vm_name => "router2",
    :net => [
      {ip: '10.0.10.2', adapter: 2, netmask: "255.255.255.252", virtualbox__intnet: "r1-r2"},
      {ip: '10.0.11.2', adapter: 3, netmask: "255.255.255.252", virtualbox__intnet: "r2-r3"},
      {ip: '192.168.20.1', adapter: 4, netmask: "255.255.255.0", virtualbox__intnet: "net2"},
      {ip: '192.168.50.11', adapter: 5},
    ]
  },

  :router3 => {
    :box_name => "ubuntu/focal64",
    :vm_name => "router3",
    :net => [
      {ip: '10.0.11.1', adapter: 2, netmask: "255.255.255.252", virtualbox__intnet: "r2-r3"},
      {ip: '10.0.12.2', adapter: 3, netmask: "255.255.255.252", virtualbox__intnet: "r1-r3"},
      {ip: '192.168.30.1', adapter: 4, netmask: "255.255.255.0", virtualbox__intnet: "net3"},
      {ip: '192.168.50.12', adapter: 5},
    ]
  }

}

Vagrant.configure("2") do |config|

  MACHINES.each do |boxname, boxconfig|

    config.vm.define boxname do |box|

      box.vm.box = boxconfig[:box_name]
      box.vm.host_name = boxconfig[:vm_name]

      if boxconfig[:vm_name] == "router3"
        box.vm.provision "ansible" do |ansible|
          ansible.playbook = "ansible/provision.yml"
          ansible.inventory_path = "ansible/hosts"
          ansible.host_key_checking = "false"
          ansible.limit = "all"
        end
      end

      boxconfig[:net].each do |ipconf|
        box.vm.network "private_network", **ipconf
      end

    end
  end
end
```

**`./ansible/ansible.cfg`**

```bash
[defaults]
#Отключение проверки ключа хоста
host_key_checking = false
#Указываем имя файла инвентаризации
inventory = hosts
#Отключаем игнорирование предупреждений
command_warnings= false
```

**`./ansible/defaults/main.yml`**

```bash
router_id_enable: true
symmetric_routing: false
```

**`./ansible/hsots`**

```bash
[routers]
router1 ansible_host=192.168.50.10 ansible_user=vagrant ansible_ssh_private_key_file=.vagrant/machines/router1/virtualbox/private_key router_id=1.1.1.1 
router2 ansible_host=192.168.50.11 ansible_user=vagrant ansible_ssh_private_key_file=.vagrant/machines/router2/virtualbox/private_key router_id=2.2.2.2
router3 ansible_host=192.168.50.12 ansible_user=vagrant ansible_ssh_private_key_file=.vagrant/machines/router3/virtualbox/private_key router_id=3.3.3.3
```

**`./ansible/provision.yml`**

```ruby
#Начало файла provision.yml
- name: OSPF
  #Указываем имя хоста или группу, которые будем настраивать
  hosts: all
  #Параметр выполнения модулей от root-пользователя
  become: yes
  #Указание файла с дополнителыми переменными (понадобится при добавлении темплейтов)
  vars_files:
    - defaults/main.yml
  tasks:
    # Обновление пакетов и установка vim, traceroute, tcpdump, net-tools
    - name: install base tools
      apt:
        name:
          - vim
          - traceroute
          - tcpdump
          - net-tools
        state: present
        update_cache: true

    #Отключаем UFW и удаляем его из автозагрузки
    - name: disable ufw service
      service:
        name: ufw
        state: stopped
        enabled: false

    # Добавляем gpg-key репозитория
    - name: add gpg frrouting.org
      apt_key:
        url: "https://deb.frrouting.org/frr/keys.asc"
        state: present

    # Добавляем репозиторий https://deb.frrouting.org/frr
    - name: add frr repo
      apt_repository:
        repo: 'deb https://deb.frrouting.org/frr {{ ansible_distribution_release }} frr-stable'
        state: present

    # Обновляем пакеты и устанавливаем FRR
    - name: install FRR packages
      apt:
        name:
          - frr
          - frr-pythontools
        state: present
        update_cache: true

    # Включаем маршрутизацию транзитных пакетов
    - name: set up forward packages across routers
      sysctl:
        name: net.ipv4.conf.all.forwarding
        value: '1'
        state: present

    # Отключаем запрет ассиметричного роутинга 
    - name: set up asynchronous routing
      sysctl:
        name: net.ipv4.conf.all.rp_filter
        value: '0'
        state: present


    # Копируем файл daemons на хосты, указываем владельца и права
    - name: base set up OSPF
      template:
        src: daemons
        dest: /etc/frr/daemons
        owner: frr
        group: frr
        mode: 0640

    # Копируем файл frr.conf на хосты, указываем владельца и права
    - name: set up OSPF
      template:
        src: frr.conf.j2
        dest: /etc/frr/frr.conf
        owner: frr
        group: frr
        mode: 0640

    # Перезапускам FRR и добавляем в автозагрузку
    - name: restart FRR
      service:
        name: frr
        state: restarted
        enabled: true
```

**`./ansible/templates/daemons`**

```bash
# This file tells the frr package which daemons to start.
#
# Sample configurations for these daemons can be found in
# /usr/share/doc/frr/examples/.
#
# ATTENTION:
#
# When activating a daemon for the first time, a config file, even if it is
# empty, has to be present *and* be owned by the user and group "frr", else
# the daemon will not be started by /etc/init.d/frr. The permissions should
# be u=rw,g=r,o=.
# When using "vtysh" such a config file is also needed. It should be owned by
# group "frrvty" and set to ug=rw,o= though. Check /etc/pam.d/frr, too.
#
# The watchfrr, zebra and staticd daemons are always started.
#
zebra=yes
bgpd=no
ospfd=yes
ospf6d=no
ripd=no
ripngd=no
isisd=no
pimd=no
pim6d=no
ldpd=no
nhrpd=no
eigrpd=no
babeld=no
sharpd=no
pbrd=no
bfdd=no
fabricd=no
vrrpd=no
pathd=no

#
# If this option is set the /etc/init.d/frr script automatically loads
# the config via "vtysh -b" when the servers are started.
# Check /etc/pam.d/frr if you intend to use "vtysh"!
#
vtysh_enable=yes
zebra_options="  -A 127.0.0.1 -s 90000000"
mgmtd_options="  -A 127.0.0.1"
bgpd_options="   -A 127.0.0.1"
ospfd_options="  -A 127.0.0.1"
ospf6d_options=" -A ::1"
ripd_options="   -A 127.0.0.1"
ripngd_options=" -A ::1"
isisd_options="  -A 127.0.0.1"
pimd_options="   -A 127.0.0.1"
pim6d_options="  -A ::1"
ldpd_options="   -A 127.0.0.1"
nhrpd_options="  -A 127.0.0.1"
eigrpd_options=" -A 127.0.0.1"
babeld_options=" -A 127.0.0.1"
sharpd_options=" -A 127.0.0.1"
pbrd_options="   -A 127.0.0.1"
staticd_options="-A 127.0.0.1"
bfdd_options="   -A 127.0.0.1"
fabricd_options="-A 127.0.0.1"
vrrpd_options="  -A 127.0.0.1"
pathd_options="  -A 127.0.0.1"


# If you want to pass a common option to all daemons, you can use the
# "frr_global_options" variable.
#
#frr_global_options=""


# The list of daemons to watch is automatically generated by the init script.
# This variable can be used to pass options to watchfrr that will be passed
# prior to the daemon list.
#
# To make watchfrr create/join the specified netns, add the the "--netns"
# option here. It will only have an effect in /etc/frr/<somename>/daemons, and
# you need to start FRR with "/usr/lib/frr/frrinit.sh start <somename>".
#
#watchfrr_options=""


# configuration profile
#
#frr_profile="traditional"
#frr_profile="datacenter"

# This is the maximum number of FD's that will be available.  Upon startup this
# is read by the control files and ulimit is called.  Uncomment and use a
# reasonable value for your setup if you are expecting a large number of peers
# in say BGP.
#
#MAX_FDS=1024

# Uncomment this option if you want to run FRR as a non-root user. Note that
# you should know what you are doing since most of the daemons need root
# to work. This could be useful if you want to run FRR in a container
# for instance.
# FRR_NO_ROOT="yes"

# For any daemon, you can specify a "wrap" command to start instead of starting
# the daemon directly. This will simply be prepended to the daemon invocation.
# These variables have the form daemon_wrap, where 'daemon' is the name of the
# daemon (the same pattern as the daemon_options variables).
#
# Note that when daemons are started, they are told to daemonize with the `-d`
# option. This has several implications. For one, the init script expects that
# when it invokes a daemon, the invocation returns immediately. If you add a
# wrap command here, it must comply with this expectation and daemonize as
# well, or the init script will never return. Furthermore, because daemons are
# themselves daemonized with -d, you must ensure that your wrapper command is
# capable of following child processes after a fork() if you need it to do so.
#
# If your desired wrapper does not support daemonization, you can wrap it with
# a utility program that daemonizes programs, such as 'daemonize'. An example
# of this might look like:
#
# bgpd_wrap="/usr/bin/daemonize /usr/bin/mywrapper"
#
# This is particularly useful for programs which record processes but lack
# daemonization options, such as perf and rr.
#
# If you wish to wrap all daemons in the same way, you may set the "all_wrap"
# variable.
#
#all_wrap=""

```

**`./ansible/templates/frr.conf.j2`**

```bash
!Указание версии FRR
frr version 8.1
frr defaults traditional
!Указываем имя машины
hostname {{ ansible_hostname }}
log syslog informational
no ipv6 forwarding
service integrated-vtysh-config
!
!Добавляем информацию об интерфейсе enp0s8
interface enp0s8
 !Указываем имя интерфейса
 description r1-r2
 !Указываем ip-aдрес и маску (эту информацию мы получили в прошлом шаге)
 ip address {{ ansible_enp0s8.ipv4.address }}/30
 !Указываем параметр игнорирования MTU
 ip ospf mtu-ignore
 !Если потребуется, можно указать «стоимость» интерфейса
 {% if ansible_hostname == 'router1' %}
 ip ospf cost 1000
 {% elif ansible_hostname == 'router2' and symmetric_routing == true %}
 ip ospf cost 1000
 {% else %}
 !ip ospf cost 450
 {% endif %}
 !Указываем параметры hello-интервала для OSPF пакетов
 ip ospf hello-interval 10
 !Указываем параметры dead-интервала для OSPF пакетов
 !Должно быть кратно предыдущему значению
 ip ospf dead-interval 30
!
interface enp0s9
 description r1-r3
 ip address {{ ansible_enp0s9.ipv4.address }}/30
 ip ospf mtu-ignore
 !ip ospf cost 45
 ip ospf hello-interval 10
 ip ospf dead-interval 30

interface enp0s10
 description net_router1
 ip address {{ ansible_enp0s10.ipv4.address }}/24
 ip ospf mtu-ignore
 !ip ospf cost 45
 ip ospf hello-interval 10
 ip ospf dead-interval 30 
!
!Начало настройки OSPF
router ospf
 !Указываем router-id 
 {% if router_id_enable == false %}!{% endif %}router-id {{ router_id }}
 !Указываем сети, которые хотим анонсировать соседним роутерам
  {% if ansible_hostname == 'router1' %}
! Блок для router1
 network 10.0.10.0/30 area 0
 network 10.0.12.0/30 area 0
 network 192.168.10.0/24 area 0
{% elif ansible_hostname == 'router2' %}
! Блок для router2
 network 10.0.10.0/30 area 0
 network 10.0.11.0/30 area 0
 network 192.168.20.0/24 area 0
{% elif ansible_hostname == 'router3' %}
! Блок для router3
 network 10.0.11.0/30 area 0
 network 10.0.12.0/30 area 0
 network 192.168.30.0/24 area 0
{% else %}
# Значения по умолчанию
{% endif %}
 !Указываем адреса соседних роутеров
 {% if ansible_hostname == 'router1' %}
! Блок для router1
 neighbor 10.0.10.2
 neighbor 10.0.12.2
{% elif ansible_hostname == 'router2' %}
! Блок для router2
 neighbor 10.0.10.1
 neighbor 10.0.11.1
{% elif ansible_hostname == 'router3' %}
! Блок для router3
 neighbor 10.0.12.1
 neighbor 10.0.11.2
{% else %}
# Значения по умолчанию
{% endif %}
 default-information originate always
!Указываем адрес log-файла
log file /var/log/frr/frr.log

```

