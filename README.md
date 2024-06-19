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
Если метод не сработал можно попробовать 
```
sudo sysctl -w net.ipv4.ip_forward=1
```

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
Sudo systemctl restart systemd-resolved
```

Перезагружаем устройство

Проверяем, пингуется ли 8.8.8.8 или ya.ru с HQ-R и BR-R. Если всё успешно, то можем окончательно убрать сетевые мосты c HQ-R и BR-R из адаптеров VirtualBox.

Для SRV устройств просто указываем IP, gateway и DNS 8.8.8.8. Делать это желательно в nmtui. Если что-то пошло не так, удаляем файлы конфигов
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
iperf3 -c 1.1.1.2
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
sudo cd /etc/backup
```
Создание скрипта:
Создайте новый файл скрипта. 
```
nano backup.sh
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
Автоматизация через cron:
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
HQ-R$ iptables -t nat -A PREROUTING -i <ИНТЕРФЕЙС> -j DNAT -p tcp --dport 2222 --to-destination <IP HQ-R>:22
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
Изменить
```
#PermitRootLogin yes
```
Перезапуск служб
```
hq-srv$ systemctl enable sshd
hq-srv$ systemctl restart sshd
```
### ПРОВЕРКА
```
ssh hq-srv@<IP-BR-R> -p 2222 # Проверка с устройства. Вы должны согласиться с ключом и зайти в hq-srv
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

