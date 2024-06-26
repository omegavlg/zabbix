# Домашнее задание к занятию "`Система мониторинга Zabbix`" - `Дедюрин Денис`
---

## Задание 1

### 1. Перед установкой zabbix-server выполняем обновление системы.
```
yum update -y
```
<img src = "img/yum_update.png" width = 100%>

### 2. Устанавливаем PosgreSQL, используя официальную документацию.

#### Устанавливаем репозиторий:
```
sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-8-x86_64/pgdg-redhat-repo-latest.noarch.rpm
```
#### Disable the built-in PostgreSQL module:
```
sudo dnf -qy module disable postgresql
```
<img src = "img/installPG01.png" width = 100%>

#### Install PostgreSQL:
```
sudo dnf install -y postgresql14-server
```
<img src = "img/installPG02.png" width = 100%>

#### Иницализируем БД и добавляем в автозапуск:
```
sudo /usr/pgsql-14/bin/postgresql-14-setup initdb
sudo systemctl enable postgresql-14
sudo systemctl start postgresql-14
```
<img src = "img/installPG03.png" width = 100%>

### 3. Устанавливаем репозиторй Zabbix.
```
rpm -Uvh https://repo.zabbix.com/zabbix/6.0/rhel/8/x86_64/zabbix-release-6.0-4.el8.noarch.rpm
dnf clean all
```
<img src = "img/installZBX01.png" width = 100%>

### 4. Устанавливаем Zabbix сервер и веб-интерфейс.
```
dnf install zabbix-server-pgsql zabbix-web-pgsql zabbix-apache-conf zabbix-sql-scripts zabbix-selinux-policy
```
<img src = "img/installZBX02.png" width = 100%>

### 5. Создаем БД Zabbix и пользователя.
```
su - postgres -c "psql --command \"CREATE USER zabbix WITH PASSWORD '123456789';\""
su - postgres -c "psql --command \"CREATE DATABASE zabbix OWNER zabbix;\""
```
<img src = "img/installZBX03.png" width = 100%>

### 6. Импортируем начальную схему и данные.
```
zcat /usr/share/zabbix-sql-scripts/postgresql/server.sql.gz | sudo -u zabbix psql zabbix
```
<img src = "img/installZBX04.png" width = 100%>

### 7. Редактируем файл: /etc/zabbix/zabbix_server.conf с помощью команды sed
```
sed -i 's/# DBPassword=/DBPassword=123456789/g' /etc/zabbix/zabbix_server.conf
```
<img src = "img/installZBX05.png" width = 100%>

### 8. Запускаем Zabbix-сервер, web-server. Добавляем в автозагрузку.
```
systemctl restart zabbix-server httpd php-fpm
systemctl enable zabbix-server httpd php-fpm
```
<img src = "img/installZBX06.png" width = 100%>

### 9. Заходим в web-интерфейс для первоначальной настройки Zabbix.

Адрес: http://192.168.1.87/zabbix

<img src = "img/web01.png" width = 100%>
<img src = "img/web02.png" width = 100%>
<img src = "img/web03.png" width = 100%>
<img src = "img/web04.png" width = 100%>
<img src = "img/web05.png" width = 100%>
<img src = "img/web06.png" width = 100%>
<img src = "img/web07.png" width = 100%>
<img src = "img/web08.png" width = 100%>

---

## Задание 2

### 1. Устанавливаем zabbix-agent на 2-х ВМ. Один агент будет установлен на туже ВМ где и zabbix-server
```
dnf install zabbix-agent
```
<img src = "img/installZBX07.png" width = 100%>

```
rpm -Uvh https://repo.zabbix.com/zabbix/6.0/rhel/8/x86_64/zabbix-release-6.0-4.el8.noarch.rpm
dnf install zabbix-agent
```
<img src = "img/installZBX08.png" width = 100%>

### 2. Запускаем zabbix-agent на ВМ.
```
systemctl start zabbix-agent
systemctl enable zabbix-agent
```
<img src = "img/installZBX09.png" width = 100%>
<img src = "img/installZBX10.png" width = 100%>

### 3. Добавляем узел через web-интерфейс.

<img src = "img/addagent01.png" width = 100%>
<img src = "img/addagent02.png" width = 100%>
<img src = "img/addagent03.png" width = 100%>

Видим, что в колнке "Доступность", у добавленного узла появился красный статус "zbx" Это говорит о том, что zabbix-agent доверяет только входящему соединению от zabbix-server, если и агент и сервер установлены на одной машине. В случае, если zabbix-agent находится в другом месте, то в его кофигурационный файл необходимо внести изменения.

Для начала можем заглянуть в лог файл и посмотреть на что ругается агент.
```
cat /var/log/zabbix/zabbix_agentd.log
```
<img src = "img/addagent04.png" width = 100%>

Видим, что действительно он не пропускает входящие соединения от удалённого сервера:
```
2050:20240603:153418.885 failed to accept an incoming connection: connection from "192.168.1.87" rejected, allowed hosts: "127.0.0.1"
2050:20240603:153433.959 failed to accept an incoming connection: connection from "192.168.1.87" rejected, allowed hosts: "127.0.0.1"
```
Чтобы это исправить, добавляем ip-адрес zabbix-server в конфигурационный файл /etc/zabbix/zabbix_agentd.conf. Для этого используем команду sed.
```
sed -i 's/Server=127.0.0.1/Server=192.168.1.87/g' /etc/zabbix/zabbix_agentd.conf
```
И перезапускаем сам агент.
```
systemctl start zabbix-agent
```
<img src = "img/addagent05.png" width = 100%>

После этого можно видеть, что статус "zbx" у добавленного узла, сменился на зеленый

<img src = "img/addagent06.png" width = 100%>

### 4. Переходим в меню "Мониторинг -> Последние данные", чтобы убедиться, что информация начала поступать на сервер.

<img src = "img/zbx_finish.png" width = 100%>

### 5. Команды, которые использовались для Git.
```
git add .
git commit -m "edit Readme.MD"
git push origin main
```

---

## Задание 3

### 1. Скачиваем zabbix-agent для Windows с официального сайта.

<img src = "img/zbx_win01.png" width = 100%>

### 2. Запускаем мастер установки и прописываем адрес сервера.

<img src = "img/zbx_win03.png" width = 100%>

### 4. Добавляем узел через web-интерфейс.
Указываем для узла группу "Windows by Zabbix agent".

<img src = "img/zbx_win04.png" width = 100%>

### 5. Видим, что узел успешно добавился и стал доступным.

<img src = "img/zbx_win05.png" width = 100%>

### 6. Для того чтобы zabbix отображал свободное место на диске C:, необходимо добавить новый элемент данных (item). Для этого заходим "Настройка - Узлы сети", выбираем наш узел и переходим в "Элементы данных".
<img src = "img/zbx_win06.png" width = 100%>

Нажимаем на кнопку "Создать элемент данных" и заполняем поля:
<img src = "img/zbx_win07.png" width = 100%>

После ввода можем протестировать введенные данные нажав на кнопку "Тест"
<img src = "img/zbx_win08.png" width = 100%>

В открывшемся окне нажимаем "Получить значение и протестировать" 
Если ошибок не возникнет, поле "Значение" заполнится данными, полученными от агента.

После проверки, нажимаем кнопку "Добавить", чтобы сохранить созданный элемент.

Возвращаемся в меню "Мониторинг -> Последние данные", чтобы увидеть, что данный элемент добавился начали поступать данные.

<img src = "img/zbx_win09.png" width = 100%>
