# Модуль 1
**Подготовка:**
1.	Создаем виртуальные машины. Все устройства Ubuntu Server, исключая SRV устройства, которые являются Ubuntu Desktop.
2.	Распределяем сетевые адаптеры (везде сетевые мосты + адаптеры, смотрящие на ближайших соседей). 
 
3.	Присваиваем имена хостов, имена устройств в соответствии с условиями:
 

**Задание 1** 
1.	Присваиваем IP-адреса, маски и шлюзы адаптерам в соответствии с таблицей (заполняя таблицу, указываем имя адаптера на данном устройстве и 4 последних символа MAC):

 

2.	Загрузившись в операционную среду, приступаем к настройке. Данные действия выполняются на устройствах ISP, HQ-R, BR-R. 
Делаем проброс портов. Для этого переходим в файл командой
```
sudo nano /etc/sysctl.conf
```
в данном файле убираем # возле строк
```
net.ipv4.ip_forward=1 			для IPv4
net.ipv6.conf.all.forwarding=1 		для IPv6
```
Сохраняем файл и выходим. Перезагружаем устройства командой
sudo reboot
После перезагрузки проверяем, видят ли машины (HQ-R-ISP и BR-R-ISP) друг друга командой ping
если не работает, то редактируем настройки конфигурации адаптеров в файле
```
Sudo nano /etc/netplan/<tab>
 ```

Настраиваем NAT на ISP
```
sudo iptables -t nat -A POSTROUTING -o <Интерфейс, смотрящий в интернет> -j MASQUERADE
```
На HQ-R и BR-R, в VirtualBox удаляем сетевые мосты и проверяем, пингуется ли 8.8.8.8 с них. Скорее всего, ping 8.8.8.8 сработает, но ping ya.ru покажет ошибку в разрешении имен. Для решения данной проблемы нам требуется перейти в файл
```
sudo nano /etc/systemd/resolved.conf
```
где убираем # в строке DNS= и приводим ее к виду DNS=8.8.8.8
 
Теперь перезагружаем службу resolved.service
```
Sudo systemctl restart systemd-resolved
```
Теперь настраиваем NAT аналогичным образом на HQ-R и BR-R
