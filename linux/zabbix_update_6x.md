# Обновление ZABBIX сервера с версии 5.4 до версии 6.0.x

## **Задача:**

Появилась необходимость обновить ZABBIX до новой версии.
Была установлена версия 5.4 на базе PostgreSQL 12.9
диск виртуальной машины - 64Гб
Необходимо обновить до 6.0.1

## **Предисловие:**

Рекомендации к системе указаны на странице - https://www.zabbix.com/documentation/current/en/manual/installation/requirements
<br>Но кто читать Руководства до возникновения проблем - это путь не русский))) 

## **Как решалась задача:**

В официальном описании обновления версии с 5.4 до 6.0 были указаны следующие шаги.

### **Step 1:** Stop Zabbix Server

``` bash
sudo service zabbix-server stop
# или
sudo systemctl stop zabbix-server
```

### **Step 2:** Backup Zabbix components

``` bash
# Create directories for backup files
mkdir -p /opt/zabbix_backup/bin_files /opt/zabbix_backup/conf_files /opt/zabbix_backup/doc_files /opt/zabbix_backup/web_files /opt/zabbix_backup/db_files

# Backup Zabbix binary, doc and conf files
cp -rp /etc/zabbix/zabbix_server.conf /opt/zabbix_backup/conf_files
cp -rp /usr/sbin/zabbix_server /opt/zabbix_backup/bin_files
cp -rp /usr/share/doc/zabbix-* /opt/zabbix_backup/doc_files
cp -rp /etc/httpd/conf.d/zabbix.conf /opt/zabbix_backup/conf_files 2>/dev/null
cp -rp /etc/apache2/conf-enabled/zabbix.conf /opt/zabbix_backup/conf_files 2>/dev/null
cp -rp /etc/zabbix/php-fpm.conf /opt/zabbix_backup/conf_files 2>/dev/null

# Backup Zabbix web files (frontend)
cp -rp /usr/share/zabbix/ /opt/zabbix_backup/web_files

# Backup Zabbix database
sudo -u postgres pg_dumpall > /opt/zabbix_backup/db_files/dump_pgsql_before_update_zabbix.dump
```

### **Step 3:** Upgrade Zabbix Server and Frontend

``` bash
# delete old Zabbix repository
dpkg --purge zabbix-release

# Upgrade Zabbix
wget https://repo.zabbix.com/zabbix/6.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_6.0-1+ubuntu$(lsb_release -rs)_all.deb
sudo dpkg -i zabbix-release_6.0-1+ubuntu$(lsb_release -rs)_all.deb
sudo apt update
sudo apt install --only-upgrade zabbix-server-pgsql zabbix-frontend-php zabbix-agent zabbix-nginx-conf
```

### **Step 4:** Start Zabbix service and database upgrade

``` bash
#
systemctl start zabbix-server

# Database upgrade can last from 1 minute to hours depending on database size. 
# Check the upgrade progress with the command 
sudo cat /var/log/zabbix/zabbix_server.log | grep database
  1794:20200408:200607.700 current database version (mandatory/optional): 04040000/04040002
  1794:20200408:200607.700 starting automatic database upgrade
  1794:20200408:200607.706 completed 1% of database upgrade
  1794:20200408:200608.804 completed 10% of database upgrade
                            .....
  1794:20200408:200613.111 completed 98% of database upgrade
  1794:20200408:200613.123 completed 100% of database upgrade
  1794:20200408:200613.123 database upgrade fully completed
  1794:20200408:200613.136 database could be upgraded to use primary keys in history tables
```

И вот на этом этапе было обнаружено ошибку - Ваша версия сервер базы данных не поддерживается...
Оказалось, что для работы новой версии ZABBIX необходим PostgreSQL не ниже 13 версии

Проверяем на всякий случай свободное место на системном разделе

``` bash
df -h
```

И обнаруживаем, что места на диске осталось совсем немного.<br><br>

### **Исправление 1 - Системный раздел - изменить размер (без LVM)**

Добавляем в параметрах виртуалки размер диска и расширяем диск в системе по следующему документу ([ссылка](ext_part_with_LVM.md))
<br><br>

### **Исправление 2 - PostgreSQL: Обновление сервера до новой версии**

Когда мы все проверили и подготовили, приступаем к обновлению сервера PostgreSQL ([ссылка](postgres_update_14.md))
<br><br>

### **Step 5:** Проверяем запуск ZABBIX

После всех вышеприведенных действий проверяем нормально ли запустился сервер ZABBIX

``` bash
sudo cat /var/log/zabbix/zabbix_server.log | grep database
```

Как видим, все в порядке и сервер теперь запускается без ошибок.<br><br>

## Навигация

[Вернуться в основное меню](../README.md)
<br> [Linux](../linux/README.md)
