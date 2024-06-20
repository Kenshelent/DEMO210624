# Модуль 1

<details>

<summary>Подготовка</summary>

### Настройка машин

Создаем виртуальные машины. Все устройства Ubuntu Server, исключая SRV устройства, которые являются Ubuntu Desktop.

Распределяем сетевые адаптеры (везде сетевые мосты + адаптеры, смотрящие на ближайших соседей). 
 ![Адаптеры](https://github.com/Kenshelent/DEMO210624/blob/main/%D0%A1%D0%B5%D1%82%D0%B5%D0%B2%D1%8B%D0%B5%20%D0%B0%D0%B4%D0%B0%D0%BF%D1%82%D0%B5%D1%80%D1%8B.png) <br>
Присваиваем имена хостов, имена устройств в соответствии с условиями.
 

</details>

<details>

<summary>Задание 1. NAT</summary>

### #natnanate

Присваиваем IP-адреса, маски и шлюзы адаптерам в соответствии с таблицей (заполняя таблицу, указываем имя адаптера на данном устройстве и 4 последних символа MAC):

![Таблица IP](https://github.com/Kenshelent/DEMO210624/blob/main/%D0%A2%D0%B0%D0%B1%D0%BB%D0%B8%D1%86%D0%B0%20%D0%BC%D0%B0%D1%80%D1%88%D1%80%D1%83%D1%82%D0%B8%D0%B7%D0%B0%D1%86%D0%B8%D0%B8.png)
 

### Данные действия выполняются на устройствах ISP, HQ-R, BR-R. 
Делаем проброс портов. Для этого переходим в файл командой
```
sudo nano /etc/sysctl.conf
```
в данном файле убираем # возле строк
```
net.ipv4.ip_forward=1 			для IPv4
net.ipv6.conf.all.forwarding=1 		для IPv6
```
Сохраняем файл и выходим и пишем.
```
sudo sysctl -p
```
Есть команда для временного решения. Работает до перезагрузки. Чтобы все работало настраиваем frr.
```
sudo sysctl -w net.ipv4.ip_forward=1
```

### NETPLAN
Проверяем видят ли машины (HQ-R-ISP и BR-R-ISP) друг друга командой ping <br>
если не работает, то редактируем настройки конфигурации адаптеров в файле
```
Sudo nano /etc/netplan/название файла(оно разное)
 ```
![NETPLAN](https://github.com/Kenshelent/DEMO210624/blob/main/%D0%A4%D0%B0%D0%B9%D0%BB%20netplan.png)


### Настраиваем NAT на ISP
```
sudo iptables -t nat -A POSTROUTING -o <Интерфейс, смотрящий в интернет> -j MASQUERADE
```

Требуется сохранить настройки NAT на ISP. Для этого устанавливаем iptables persistent
```
sudo apt-get install iptables-persistent Спросит сохранить ли, нажимаем два раза <y>
```
Ручное сохранение 
```
sudo netfilter-persistent save
```
### Проблема с DNS
На HQ-R и BR-R переводим интерфейсы, смотрящие в интернет, в состояние DOWN
```
sudo ip link set <интерфейс> down
```
Проверяем, пингуется ли 8.8.8.8 с них. Скорее всего, ping 8.8.8.8 сработает, но ping ya.ru покажет ошибку в разрешении имен. Для решения данной проблемы нам требуется перейти в файл
```
sudo nano /etc/systemd/resolved.conf
```
где убираем # в строке DNS= и приводим ее к виду DNS=8.8.8.8
 
Теперь перезагружаем службу resolved.service

```
sudo systemctl restart systemd-resolved.service

sudo systemctl start systemd-resolved

sudo systemctl enable systemd-resolved

sudo reboot

```
Проверяем, пингуется ли 8.8.8.8 или ya.ru с HQ-R и BR-R. Если всё успешно, то можем окончательно убрать сетевые мосты c HQ-R и BR-R из адаптеров VirtualBox.

Для SRV устройств просто указываем IP, gateway и DNS 8.8.8.8. Делать это желательно в nmtui. Сервера не получат доступ в интернет, пока не настроен OSPF. Если что-то пошло не так, удаляем файлы конфигов
```
sudo rm /etc/netplan/*
```
</details>

<details>

<summary>Задание 2. Настройка FRR</summary>

### Настройка FRR (OSPF) делается на **ISP, HQ-R и BR-R**. 
Установите FRR на каждом маршрутизаторе
```
sudo apt-get update
sudo apt-get upgrade -y
sudo apt-get install frr
```

Включите необходимые демоны (OSPF). <br> Отредактируйте файл /etc/frr/daemons и убедитесь, что следующие строки активны (уберите символ #):
```
ospfd=yes
```
Перезапустите FRR для применения изменений:
```
sudo systemctl restart frr
```
Настройте OSPF на каждом маршрутизаторе. Для этого запустите vtysh: 
```
sudo vtysh 
```

### FRR MOMENT HQ-R, BR-R
FRR в своем конфигурационном файле изначально хранит команду no ip forwarding / no ipv6 forwarding, что приводит к конфликту с настройками ОС (sysctl.conf), чтобы это изменить, необходимо в режиме глобальной конфигурации указать данные команды без преписки no. 
```
frr -> ip forwarding
```

# Настройка FFR для ISP: 
```
enable 
configure terminal 
router-id 1.1.1.1
router ospf
network 1.1.1.0/30 area 0 
network 2.2.2.0/30 area 0
network 3.3.3.0/30 area 0
end 
write
```
# Настройка FFR для HQ-R: 
```
enable 
configure terminal
ip forwarding 
router-id 1.1.1.2
router ospf
network 1.1.1.0/30 area 0
network 172.16.100.0/26 area 0
network 4.4.4.0/30 area 0
end
write 
```
# Настройка FFR для BR-R:
``` 
enable 
configure terminal
ip forwarding 
router-id 2.2.2.2
router ospf
network 2.2.2.0/30 area 0 
network 192.168.100.0/28 area 0 
end 
write
```
</details>

<details>

<summary>Задание 3. DHCP</summary>

### Задание выполняется на HQ-R. 
Для этого будем использовать пакет isc-dhcp-server. <br>
Следующие шаги помогут вам настроить DHCP сервер, включая резервирование IP-адреса для определенного устройства. <br>
Для установки DHCP сервера откройте терминал и выполните следующую команду: <br>
```
sudo apt-get install isc-dhcp-server 
```
После установки, необходимо настроить конфигурационный файл DHCP сервера.  <br>
Откройте файл /etc/dhcp/dhcpd.conf для редактирования: 
```
sudo nano /etc/dhcp/dhcpd.conf
```
### Основные настройки:
```
option domain-name "hq.work"; 
option domain-name-servers 172.16.100.2, 8.8.8.8; 
Пул IP-адресов 
subnet 172.16.100.0 netmask 255.255.255.192 { 
range 172.16.100.5 172.16.100.20;
option broadcast-address 172.16.100.63;
option routers 172.16.100.1; 
} 
Резервация IP-адреса
host hq-srv { 
hardware ethernet <xx:xx:xx:xx:xx:xx>; mac на hq-srv
fixed-address 172.16.100.2; 
} 
```

Перейти в файл sudo nano /etc/default/isc-dhcp-server

```
INTERFACESv4="Интерфейс на локальную сеть  HQ-SRV"
INTERFACESv6=""
```

После внесения изменений перезапустите DHCP сервер для применения новых настроек: 
```
sudo systemctl restart isc-dhcp-server
```

</details>

<details>

<summary>Задание 4. Создание учетных записей</summary>

### Учетные записи 

Войдите в каждое устройство, указанное в задании и создайте учетные записи с соответствующими именами и паролями. <br>
Настройка sudo привилегий: Если учетные записи должны иметь привилегии суперпользователя, добавьте их в группу sudo. <br>
### Создание учетной записи Admin: 
```
sudo useradd admin
sudo passwd admin
```
### Создание учетной записи Branch admin:
```
sudo useradd branch_admin
sudo passwd branch_admin
```
### Создание учетной записи Network admin:
```
sudo useradd network_admin
sudo passwd network_admin
```
### Добавление учетной записи Admin в группу sudo:
```
sudo usermod -aG sudo admin
```
### Добавление учетной записи Branch admin в группу sudo:
```
sudo usermod -aG sudo branch_admin
```
### Добавление учетной записи Network admin в группу sudo:
```
sudo usermod -aG sudo network_admin
```

</details>

<details>

<summary>Задание 5. Пропускная способность</summary>

### Выполнять на ISP, HQ-R
```
apt-get install iperf3 -y. Во время установки нажимаем no
```
### Выполнять на ISP
```
iperf3 -s
```
### Выполнять на HQ-R
```
iperf3 -c 1.1.1.1
```

![Пример](https://github.com/Kenshelent/DEMO210624/blob/main/%D0%92%D1%8B%D0%BF%D0%BE%D0%BB%D0%BD%D0%B5%D0%BD%D0%B8%D0%B5%20iperf3.png)

Скриншот демонстрирует результаты теста пропускной способности сети с использованием утилиты iperf3. <br>
Тест проводился между двумя хостами с IP-адресами 1.1.1.1 и 1.1.1.2. <br>
На левой части экрана запущен сервер iperf3, на правой - клиент. 

Основные параметры: <br>

Интервал тестирования: 10 секунд. <br>
Общий объем переданных данных: 5.16 ГБайт.<br>
Средняя пропускная способность: 4.44 Гбит/сек.<br>

Более детально:<br>

В каждом односекундном интервале пропускная способность варьируется от 3.92 Гбит/сек до 4.75 Гбит/сек на передаче данных.<br>
На стороне сервера фиксируются объемы данных и скорость передачи за каждый интервал. <br>
На стороне клиента дополнительно фиксируется количество повторных отправок пакетов (Retr). <br>

Пиковая пропускная способность:<br>

Максимальная: 4.75 Гбит/сек.<br>
Минимальная: 3.92 Гбит/сек.<br>

Эти данные показывают высокую производительность и стабильность сети на протяжении всего теста, с незначительными колебаниями в пропускной способности.<br>


</details>

<details>

<summary>Задание 6. Бекап Скрипты</summary>

# Скриптики

Создание backup скрипта на Ubuntu Server для автоматизации процесса копирования файлов <br>
Вот пример простого bash-скрипта для выполнения резервного копирования: <br>
# Создайте директорию для резервного копирования
```
sudo mkdir -p "/etc/backup"
```
Перейти в эту папку.
```
cd /etc/backup
```
Создание скрипта:
Создайте новый файл скрипта. 
```
sudo nano backup.sh
```
Редактирование скрипта:
```
#!/bin/bash
# Копирование файлов и директорий
cp -r /etc/frr/frr.conf /etc/backup/frr.conf
# Вывод сообщения об успешном завершении
echo "OK"
```
Сохранение и закрытие файла: <br>
Придание скрипту права на выполнение:
```
sudo chmod +x backup.sh
```
Запуск скрипта:
```
sudo ./backup.sh
```
Выходим из директории
```
cd
```
### Автоматизация через cron:
Чтобы автоматизировать выполнение скрипта, вы можете добавить его в cron. Откройте cron для редактирования:
```
crontab -e
```
Добавьте строку для выполнения скрипта в нужное время. Например, для ежедневного выполнения в полночь:
```
0 0 * * */etc/backup/backup.sh
```
Теперь ваш скрипт будет выполняться автоматически в соответствии с расписанием cron, делая резервные копии ваших файлов и директорий.

</details>

<details>


<summary>Задание 7. SSH</summary>

### IPTABLES

```
HQ-R$ iptables -t nat -A PREROUTING -i <ИНТЕРФЕЙС СМОТРЯЩИЙ В ISP> -j DNAT -p tcp --dport 2222 --to-destination <IP HQ-SRV>:22
```

Пример сценария:
Внешний пользователь пытается подключиться к вашему серверу по IP-адресу 172.16.100.2 (IP вашего HQ-SRV) и порту 2222. <br>
Пакет поступает на интерфейс смотрящего на тот интерфейс от которого идет запрос. <br>
Правило в таблице nat обнаруживает, что пакет предназначен для порта 2222.<br>
Пакет пересылается к внутреннему серверу с IP 172.16.100.2 на порт 22.<br>

### НА HQ-SRV
```
sudo apt-get install openssh-server
nano /etc/ssh/sshd_config
```
Перезапуск служб
```
hq-srv$ systemctl enable ssh
hq-srv$ systemctl restart ssh
```
### ПРОВЕРКА
```
ssh hq-srv@<IP-HQ-SRV> -p 2222 # Проверка с устройства. Вы должны согласиться с ключом и зайти в hq-srv
```

</details>

<details>

<summary>Задание 8. CLI BLOCK</summary>

### НА ISP
```
iptables -A FORWARD -s 3.3.3.0/30 -p tcp --dport 2222 -j DROP 
```
### НА HQ-R
```
iptables -A FORWARD -s 4.4.4.0/30 -p tcp --dport 2222 -j DROP 
```
</details>

# Модуль 2
<details>

<summary>Задание 1. DNS-сервера</summary>


### Делать на HQ-SRV 

Установите пакет bind9
```
sudo apt install bind9
```
Настройте файл конфигурации BIND:
```
sudo nano /etc/bind/named.conf.local
```
Добавьте конфигурацию для зоны hq.work и обратной зоны:
```

zone "hq.work" {
    type master;
    file "/etc/bind/db.hq.work";
};

zone "16.172.in-addr.arpa" {
    type master;
    file "/etc/bind/db.172.16";
};

zone "branch.work" {
    type master;
    file "/etc/bind/db.branch.work";
};

zone "168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.192.168";
};

```

Создайте файлы зоны для hq.work:
```
sudo cp /etc/bind/db.local /etc/bind/db.hq.work
sudo nano /etc/bind/db.hq.work
```
Измените содержимое файла на следующее:
```
$TTL    604800
@       IN      SOA     ns.hq.work. admin.hq.work. (
                         2         ; Serial
                     604800         ; Refresh
                      86400         ; Retry
                    2419200         ; Expire
                     604800 )       ; Negative Cache TTL
;
@       IN      NS      ns.hq.work.
ns      IN      A       172.16.100.2
hq-r    IN      A       172.16.100.1
hq-srv  IN      A       172.16.100.2
```
Создайте файлы обратной зоны для hq.work:
```
sudo cp /etc/bind/db.127 /etc/bind/db.172.16
sudo nano /etc/bind/db.172.16
```
Измените содержимое файла на следующее:
```
$TTL    604800
@       IN      SOA     ns.hq.work. admin.hq.work. (
                         2         ; Serial
                     604800         ; Refresh
                      86400         ; Retry
                    2419200         ; Expire
                     604800 )       ; Negative Cache TTL
;
@       IN      NS      ns.hq.work.
1.100   IN      PTR     hq-r.hq.work.
2.100   IN      PTR     hq-srv.hq.work.
```
Создайте файлы зоны для branch.work:
```
sudo cp /etc/bind/db.local /etc/bind/db.branch.work
sudo nano /etc/bind/db.branch.work
```
Измените содержимое файла на следующее:
```
$TTL    604800
@       IN      SOA     ns.branch.work. admin.branch.work. (
                         2         ; Serial
                     604800         ; Refresh
                      86400         ; Retry
                    2419200         ; Expire
                     604800 )       ; Negative Cache TTL
;
@       IN      NS      ns.branch.work.
ns      IN      A       192.168.100.2
br-r    IN      A       192.168.100.1
br-srv  IN      A       192.168.100.2
```

Создайте файлы обратной зоны для branch.work:
```
sudo cp /etc/bind/db.127 /etc/bind/db.192.168
sudo nano /etc/bind/db.192.168
```
Измените содержимое файла на следующее:
```
$TTL    604800
@       IN      SOA     ns.branch.work. admin.branch.work. (
                         2         ; Serial
                     604800         ; Refresh
                      86400         ; Retry
                    2419200         ; Expire
                     604800 )       ; Negative Cache TTL
;
@       IN      NS      ns.branch.work.
1.100   IN      PTR     br-r.branch.work.
2.100   IN      PTR     br-srv.branch.work.
```

Перезапустите сервис BIND для применения изменений:
```
sudo systemctl restart bind9
```
Проверьте конфигурацию:
Убедитесь, что конфигурация BIND не содержит ошибок:
```
sudo named-checkconf
```
Проверьте файлы зоны:
```
sudo named-checkzone hq.work /etc/bind/db.hq.work
sudo named-checkzone 16.172.in-addr.arpa /etc/bind/db.172.16
sudo named-checkzone branch.work /etc/bind/db.branch.work
sudo named-checkzone 168.192.in-addr.arpa /etc/bind/db.192.168
```

С помощью DNS можно обращаться к серверам и устройствам по именам (например, hq-r.hq.work) вместо сложных для запоминания IP-адресов (например, 172.16.100.1).
```
ping hq-r.hq.work (Соответствующий IP-адрес: 172.16.100.1)
nslookup 172.16.100.1(Соответствующее доменное имя: hq-r.hq.work)
ping br-r.branch.work (Соответствующий IP-адрес: 192.168.100.1)
nslookup 192.168.100.1 (Соответствующее доменное имя: br-r.branch.work)
```

### Настройка DNS-форвардинга на BIND

Редактирование конфигурации BIND для включения форвардинга:

Откройте файл /etc/bind/named.conf.options для редактирования:
```
sudo nano /etc/bind/named.conf.options
```
Добавьте форвардинг DNS-запросов к внешним DNS-серверам:

Найдите секцию options и добавьте следующие строки:
```
options {
    directory "/var/cache/bind";

    // For security purposes, BIND should run as a non-root user.
    // Uncomment the following line and specify the user and group.
    // user bind;
    // group bind;

    // If there is a firewall between you and nameservers you want
    // to talk to, you might need to uncomment the query-source
    // directive below.  Previous versions of BIND always asked
    // questions using port 53, but BIND 8.1 uses an unprivileged
    // port by default.
    // query-source address * port 53;

    // If your name server is behind a firewall, you might need to specify the ports
    // that the server can use to make queries. See the "open-ended" list of options
    // for more information.
    // query-source-v6 address * port 53;

    // forwarders for external DNS requests
    forwarders {
        8.8.8.8;  // Google DNS
        8.8.4.4;  // Google DNS
    };

    // Disable recursion for external queries
    allow-recursion {
        any;
    };

    // If you want to allow only specific clients to query your nameserver,
    // you can specify their IP addresses here.
    // allow-query { 192.168.0.0/24; };

    dnssec-validation auto;

    auth-nxdomain no;    # conform to RFC1035
    listen-on-v6 { any; };
};
```
В этом примере мы настроили DNS-сервер для перенаправления запросов к внешним DNS-серверам Google (8.8.8.8 и 8.8.4.4) для тех имен, которые он не может разрешить локально.

Перезапустите сервис BIND для применения изменений:

```
sudo systemctl restart bind9
```
</details>


<details>

<summary>Задание 2. NTP</summary>

### на HQ-R
```
sudo apt-get install chrony -y
sudo nano /etc/chrony/chrony.conf
```
Пишем в файл
```
server 127.0.0.1 iburst prefer:| server 127.0.0.1
```
указывает Chrony использовать локальный сервер времени с IP-адресом 127.0.0.1 (localhost).
iburst: этот параметр заставляет Chrony посылать несколько (обычно четыре) запросов на начальной стадии синхронизации времени, чтобы ускорить процесс получения времени от сервера.
prefer: указывает, что этот сервер должен быть предпочтительным, если доступно несколько серверов.
```
hwtimestamp *
```
Эта директива включает аппаратные временные метки на всех интерфейсах (* обозначает все интерфейсы). Аппаратные временные метки позволяют Chrony более точно определять время передачи и получения пакетов, что улучшает точность синхронизации.
```
local stratum 5
```
local: указывает Chrony работать как локальный сервер времени.
stratum 5: задает стратиум (уровень) локального сервера времени. Стратиум определяет, насколько далеко сервер находится от эталонного источника времени. Чем ниже значение стратиума, тем ближе сервер к эталонному источнику времени. Значение 5 означает, что сервер времени не является эталонным и должен использоваться как временный источник при отсутствии других источников.
```
allow all
```
Эта директива позволяет всем сетям и устройствам синхронизировать время с данным Chrony сервером. По умолчанию Chrony может блокировать запросы от некоторых сетей, но эта настройка снимает все ограничения.


</details>

<details>

<summary>Задание 5. Apache</summary>

### на BR-SRV

Для настройки веб-сервера Apache на сервере BR-SRV для LMS с использованием базы данных MySQL, выполните следующие шаги:
1. Установка Apache и MySQL
Сначала установим необходимые пакеты на сервере BR-SRV.
```
sudo apt install apache2 mysql-server -y
```
Затем войдем в MySQL и создадим базу данных и пользователей.
```
sudo mysql -u root -p
```
Внутри MySQL выполните следующие команды:
```
CREATE DATABASE lms_db;
CREATE USER 'admin'@'localhost' IDENTIFIED BY 'P@ssw0rd';
CREATE USER 'manager1'@'localhost' IDENTIFIED BY 'P@ssw0rd';
CREATE USER 'manager2'@'localhost' IDENTIFIED BY 'P@ssw0rd';
CREATE USER 'manager3'@'localhost' IDENTIFIED BY 'P@ssw0rd';
CREATE USER 'user1'@'localhost' IDENTIFIED BY 'P@ssw0rd';
CREATE USER 'user2'@'localhost' IDENTIFIED BY 'P@ssw0rd';
CREATE USER 'user3'@'localhost' IDENTIFIED BY 'P@ssw0rd';
CREATE USER 'user4'@'localhost' IDENTIFIED BY 'P@ssw0rd';
CREATE USER 'user5'@'localhost' IDENTIFIED BY 'P@ssw0rd';
CREATE USER 'user6'@'localhost' IDENTIFIED BY 'P@ssw0rd';
CREATE USER 'user7'@'localhost' IDENTIFIED BY 'P@ssw0rd';
CREATE USER 'user2'@'localhost' IDENTIFIED BY 'P@ssw0rd';

-- Раздаем права
GRANT ALL PRIVILEGES ON lms_db.* TO 'admin'@'localhost';
GRANT ALL PRIVILEGES ON lms_db.* TO 'manager1'@'localhost';
GRANT ALL PRIVILEGES ON lms_db.* TO 'manager2'@'localhost';
GRANT ALL PRIVILEGES ON lms_db.* TO 'manager3'@'localhost';
GRANT ALL PRIVILEGES ON lms_db.* TO 'user1'@'localhost';
GRANT ALL PRIVILEGES ON lms_db.* TO 'user2'@'localhost';
GRANT ALL PRIVILEGES ON lms_db.* TO 'user3'@'localhost';
GRANT ALL PRIVILEGES ON lms_db.* TO 'user4'@'localhost';
GRANT ALL PRIVILEGES ON lms_db.* TO 'user5'@'localhost';
GRANT ALL PRIVILEGES ON lms_db.* TO 'user6'@'localhost';
GRANT ALL PRIVILEGES ON lms_db.* TO 'user7'@'localhost';


-- Создаём роли
GRANT 'Admin' TO 'admin'@'localhost';
GRANT ‘Manager’ TO 'manager1'@'localhost';
GRANT ‘Manager’ TO 'manager2'@'localhost';
GRANT ‘Manager’ TO 'manager3'@'localhost';
GRANT 'WS' TO 'user1'@'localhost';
GRANT 'WS' TO 'user2'@'localhost';
GRANT 'WS' TO 'user3'@'localhost';
GRANT 'WS' TO 'user4'@'localhost';
GRANT 'TEAM' TO 'user5'@'localhost';
GRANT 'TEAM' TO 'user6'@'localhost';
GRANT 'TEAM' TO 'user7'@'localhost';


-- Активируем роли для пользователей
SET DEFAULT ROLE 'Admin' FOR 'admin'@'localhost';
SET DEFAULT ROLE ‘Manager’ FOR 'manager1'@'localhost';
SET DEFAULT ROLE ‘Manager’ FOR 'manager2'@'localhost';
SET DEFAULT ROLE ‘Manager’ FOR 'manager3'@'localhost';
SET DEFAULT ROLE 'WS' FOR 'user1'@'localhost';
SET DEFAULT ROLE 'WS' FOR 'user2'@'localhost';
SET DEFAULT ROLE 'WS' FOR 'user3'@'localhost';
SET DEFAULT ROLE 'WS' FOR 'user4'@'localhost';
SET DEFAULT ROLE 'TEAM' FOR 'user5'@'localhost';
SET DEFAULT ROLE 'TEAM' FOR 'user6'@'localhost';
SET DEFAULT ROLE 'TEAM' FOR 'user7'@'localhost';

FLUSH PRIVILEGES;
EXIT;
```
### Настройка Apache
Создадим конфигурационный файл для вашего сайта в Apache.
```
sudo nano /etc/apache2/sites-available/lms.conf
```
Добавьте в файл следующие строки:
```
<VirtualHost *:80>
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/html/lms
    ServerName br-srv

    <Directory /var/www/html/lms>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```
Активируем сайт и перезапускаем Apache:
```
sudo a2ensite lms.conf
sudo systemctl reload apache2
```
### Создание главной страницы
Создадим директорию для вашего сайта и добавим главную страницу с номером места.
```
sudo mkdir -p /var/www/html/lms
sudo nano /var/www/html/lms/index.html
```
Добавьте в файл index.html следующий код:
```
<h1>НОМЕР 1</h1>
```
Так же можно открыть файл /var/html/index.html
```
В котором ищем текст любой со страницы localhost. и меняем его на свой номер!
```
### Проверка работы
Откройте браузер и перейдите по адресу вашего сервера (например, http://br-srv или http://localhost/lms). Вы должны увидеть номер места на главной странице.
Эти шаги помогут вам настроить веб-сервер Apache и базу данных MySQL для вашего LMS
</details>


<details>

<summary>Задание 1. Rsyslog</summary>

### На клиентской машине (в самом задании на всех):


Установка Rsyslog: Выполните команду для установки rsyslog:
```
sudo apt-get install rsyslog
```
Настройка Rsyslog для прослушивания сетевых сообщений: Откройте файл конфигурации rsyslog для редактирования:
```
sudo nano /etc/rsyslog.conf
```
Раскомментируйте строки для включения модулей UDP и TCP:
```
module(load="imudp")
input(type="imudp" port="514")
module(load="imtcp")
input(type="imtcp" port="514")
```
Настройка отправки логов на сервер hq-srv: В том же файле /etc/rsyslog.conf добавьте следующую строку:
```
*.* @<ip hq srv>:514
```
Перезапуск службы rsyslog: Перезапустите службу rsyslog для применения изменений:
```
sudo systemctl restart rsyslog
```
Проверка отправки логов: Убедитесь, что логи отправляются корректно, с помощью команды:
```
sudo tail -f /var/log/syslog
```

### На сервере (hq-srv):
Установка Rsyslog: Установите rsyslog, если он еще не установлен:
```
sudo apt-get install rsyslog
```
Настройка Rsyslog для приема сетевых сообщений: Откройте файл конфигурации rsyslog для редактирования:
```
sudo nano /etc/rsyslog.conf
```
Раскомментируйте строки для включения модулей UDP и TCP:
```
module(load="imudp")
input(type="imudp" port="514")
module(load="imtcp")
input(type="imtcp" port="514")
```
Перезапуск службы rsyslog: Перезапустите службу rsyslog для применения изменений:
```
sudo systemctl restart rsyslog
```
Проверка получения логов: Проверьте, что сервер получает логи от клиента, с помощью команды:
```
sudo tail -f /var/log/syslog
```
Проверка работы: Отправьте тестовое сообщение с удаленного хоста: 
```
logger "Test message from client"
```

</details>

<details>

<summary>Задание 2. Центр Сертификации</summary>

###  на HQ-SRV
Установите Openssl на каждом устройстве
```
sudo apt-get update
sudo apt-get upgrade -y
sudo apt-get install openssh (уже вероятно установлен)
sudo apt-get install openssh-server (для того, чтобы при передаче файлов подключаться по ssh к устройствам)
```
### Настройки на HQ-SRV
Создаем корневой сертификат
```
sudo openssl genrsa -des3 -out myCA.key 2048
```
Вводим парольную фразу, простую т.к. ее писать придется много раз !НО МИНИМУМ 4 СИМВОЛА!*
```
sudo openssl req -x509 -new -nodes -key myCA.key -sha256 -days 1825 -out myCA.pem
```
Вводим парольную фразу и заполняем данные
Создаем самоподписный сертификат, устанавливаем срок его действия на 1024 дня. 
```
sudo ssh-keygen -t rsa -b 2048 -f ssh_host_rsa_key
```
Далее создаем webserver.key - Приватный ключ RSA длиной 4096 бит, который будет использоваться веб-сервером для шифрования и дешифрования трафика. А также файл webserver.csr: Запрос на подпись сертификата, который содержит публичную часть ключа и информацию о сервере (например, доменное имя, организация, страна и т.д.). Этот файл используется для получения сертификата от центра сертификации (CA).
```
sudo openssl req -newkey rsa:2048 -nodes -keyout webserver.key -out webserver.csr
```
Создаем самоподписанный сертификат, используя указанный приватный ключ.
```
sudo openssl req -x509 -new -nodes -key myCA.key -sha256 -days 1024 -out myCA.pem
```
Создаем сертификат
```
sudo openssl x509 -req -in webserver.csr -CA myCA.pem -CAkey myCA.key -CAcreateserial -out webserver.crt -days 365 -sha256
```
### ВАЖНО!

Копируем ключ.pub c сервера на котором находится центр сертификации во все хосты (HQ-SRV).
```
sudo scp ssh_host_rsa_key.pub br-srv@192.168.100.10:~
```
br-srv@192.168.100.10:~ - копирует на название@адрес машины:~( этот символ = домашняя директория)

где мы указываем логин_на_машине@ip_машины:~


</details>


<details>

<summary>Задание 3. SSH 2</summary>

Установка SSH-сервера (на сервере)
Установка OpenSSH:
```
sudo apt install ssh
```
Проверка состояния SSH-сервера:
```
sudo systemctl status ssh
```
Создать banner
```
echo "Authorized access only!" | sudo tee /etc/ssh/banner
```
Переходим в файл sshd_config
```
sudo nano /etc/ssh/sshd_confing
```
Переведите на нестандартный порт:
```
Port 2222
```
Установите предел времени аутентификации до 5 минут: 
```
LoginGraceTime 5m 
```
Установите запрет на доступ root:
```
PermitRootLogin no 
```
Ограничьте ввод попыток до 4:
```
MaxAuthTries 4 
```
Для авторизации по сертификату:
```
PubkeyAuthentication yes
AuthorizedKeysFile /home/hq-srv/ssh_host_rsa_key.pub
```
Если это не десктоп версия, то из корневого каталога пишем ls и вероятно, данный файл будет там один
Отключите аутентификацию по паролю:
```
PasswordAuthentication no 
```
Отключите пустые пароли:
```
PermitEmptyPasswords no 
```
Banner:
```
Banner /etc/ssh/banner
```
После внесения всех изменений, перезапустите SSH службу, чтобы применить новые настройки:
```
sudo systemctl restart ssh
```
Подключение с указанием ключа:
```
sudo ssh -i /путь user@ip_remote_host
```
ОБЯЗАТЕЛЬНО SUDO | Путь вида /home/hq-srv/ssh_host_rsa_key.pub


</details>

<details>

<summary>Задание 5. Ограничения</summary>

Настройте систему управления трафиком на роутере BR-R для контроля входящего трафика. ВСЕ ВВОДИМ ПО ПОРЯДКУ.

# Разрешение подключений к портам DNS, HTTP и HTTPS
```
sudo iptables -A INPUT -p tcp --dport 53 -j ACCEPT
sudo iptables -A INPUT -p udp --dport 53 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT
```
# Разрешение работы протоколов ICMP
```
sudo iptables -A INPUT -p icmp -j ACCEPT
```
# Разрешение работы протокола SSH
```
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 2222 -j ACCEPT
```
# Разрешение работы gre (на всякий случай)
```
sudo iptables -A INPUT -p gre -j ACCEPT
```
# Разрешение уже установленных подключений и трафика на loopback интерфейсе (Чтобы ничего не сломалось)
```
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A INPUT -i lo -j ACCEPT
```
# Запрет всех прочих подключений
```
sudo iptables -P INPUT DROP
sudo iptables -P FORWARD DROP
sudo iptables -P OUTPUT ACCEPT
sudo netfilter-persistent save
```

</details>

<details>

<summary>Задание 6. Принтер </summary>

### LOCALHOST:631
ADMINISTRATION -> LPD PRINTER -> lpd://localhost/Virtual_Printer

Укажите название принтера, описание и месторасположение (по вашему усмотрению).
Выберите драйвер для принтера.
</details>

<details>

<summary>Задание 7. GRE туннель </summary>

### Настройка GRE туннеля на HQ-R: 
```
sudo ip tunnel add gre1 mode gre local 1.1.1.2 remote 2.2.2.2 ttl 255
sudo ip addr add 10.0.0.1/30 dev gre1
sudo ip link set gre1 up
```
### Настройка GRE туннеля на BR-R: 
```
sudo ip tunnel add gre1 mode gre local 2.2.2.2 remote 1.1.1.2 ttl 255
sudo ip addr add 10.0.0.2/30 dev gre1
sudo ip link set gre1 up
```
Добавление маршрутов для использования GRE туннеля:
На HQ-R добавьте маршрут к внутренней сети BR-R через GRE туннель.
```
sudo ip route add 192.168.100.0/28 dev gre1
```
На BR-R добавьте маршрут к внутренней сети HQ-R через GRE туннель.
```
sudo ip route add 172.16.100.0/26 dev gre1
```
Проверка подключения:
Проверьте туннель
### На HQ-R
```
ping 10.0.0.2
```
### На BR-R
```
ping 10.0.0.1
```

</details>
<details>

<summary>Задание 8. Настройка скриптов мониторинга</summary>

### Настройка скриптов мониторинга
Создайте скрипт для проверки нагрузки процессора, оперативной памяти и заполненности диска.

```
sudo nano /usr/local/bin/system_monitor.sh
```

```
#!/bin/bash

CPU_THRESHOLD=70
MEM_THRESHOLD=80
DISK_THRESHOLD=85

# Check CPU load
CPU_LOAD=$(top -bn1 | grep "load average:" | awk '{print $10}' | sed 's/,//')
CPU_LOAD_INT=${CPU_LOAD%.*}

if [ "$CPU_LOAD_INT" -ge "$CPU_THRESHOLD" ]; then
  logger -p user.warning "CPU load is $CPU_LOAD%"
fi

# Check Memory usage
MEM_USAGE=$(free | grep Mem | awk '{print $3/$2 * 100.0}')
MEM_USAGE_INT=${MEM_USAGE%.*}

if [ "$MEM_USAGE_INT" -ge "$MEM_THRESHOLD" ]; then
  logger -p user.warning "Memory usage is $MEM_USAGE%"
fi

# Check Disk usage
DISK_USAGE=$(df -h / | grep / | awk '{ print $5 }' | sed 's/%//g')

if [ "$DISK_USAGE" -ge "$DISK_THRESHOLD" ]; then
  logger -p user.warning "Disk usage is $DISK_USAGE%"
fi
```
Сделайте скрипт исполняемым:
```
sudo chmod +x /usr/local/bin/system_monitor.sh
```
Настройка cron для периодического выполнения скрипта
Откройте crontab для редактирования:
```
sudo crontab -e
```
Добавьте следующую строку для запуска скрипта каждую минуту:
```
* * * * * /usr/local/bin/system_monitor.sh
```
Настройка rsyslog для отправки уведомлений
Откройте файл конфигурации rsyslog:
```
sudo nano /etc/rsyslog.conf
```
Добавьте следующую строку, чтобы включить отправку предупреждений на syslog:
```
user.warning    /var/log/system_monitor.log
```
Перезапустите службу rsyslog для применения изменений:
```
sudo systemctl restart rsyslog
```
</details>

<details>

<summary>Задание 9. RAID</summary>


Для настройки программного RAID 5 из дисков по 1 Гб на машине с Ubuntu Linux (BR-SRV), выполните следующие шаги:
Предварительно нужно добавить диски на выключенной машине!

Установите необходимые пакеты:
```
sudo apt-get update
sudo apt-get install mdadm
```
Инициализация дисков: Предположим, что ваши диски распознаны как /dev/sdb, /dev/sdc, и /dev/sdd. Убедитесь, что эти диски не содержат важных данных и что они подготовлены для использования в RAID.
Создайте RAID 5:
```
sudo mdadm --create --verbose /dev/md0 --level=5 --raid-devices=3 /dev/sdb /dev/sdc /dev/sdd
```
Проверьте статус RAID:
```
sudo mdadm --detail /dev/md0
```
Создайте файловую систему:
```
sudo mkfs.ext4 /dev/md0
```
Создайте точку монтирования и смонтируйте RAID:
```
sudo mkdir -p /mnt/raid5
sudo mount /dev/md0 /mnt/raid5
```
Обновите /etc/fstab для автоматического монтирования: 
Чтобы ваш RAID автоматически монтировался при загрузке системы, добавьте строку в файл /etc/fstab:
```
echo '/dev/md0 /mnt/raid5 ext4 defaults,nofail,discard 0 0' | sudo tee -a /etc/fstab
```
Настройте mdadm конфигурацию: 
Обновите конфигурацию mdadm для сохранения настроек RAID массива:
```
sudo mdadm --detail --scan | sudo tee -a /etc/mdadm/mdadm.conf
sudo update-initramfs -u
```


</details>
