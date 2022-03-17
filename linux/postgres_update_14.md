# Обновление сервера PostgreSQL с версии 12.10 до версии 14

Итак прочитав мануалы и рекомендации по обновлению сервера PostgreSQL, приступаем к обновлению сервера PostgreSQL. Для этого выполним следующие действия:<br><br>

## Порядок обновления сервера PostgreSQL до последней версии

<br>

### Шаг 1

Останавливаете работу всех систем, использующих базу данных.
<br>Остановливаете работу php-fpm или apache2 (nginx), а также временно отключите задачи в cron.<br><br>

### Шаг 2

Проверьте, какая версия PostgreSql у вас установлена на данный момент:

``` bash
sudo -u postgres psql -d userside -A -c "SELECT VERSION()"
sudo apt list --installed | grep postgresql
```

<br>

### Шаг 3

Обновите текущие версии

``` bash
sudo apt update
sudo apt --only-upgrade install postgresql
```

<br>

### Шаг 4

Установите новую версию PostgreSql не удаляя старой(!).

``` bash
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
echo "deb http://apt.postgresql.org/pub/repos/apt/ $(lsb_release -cs)-pgdg main" | sudo tee /etc/apt/sources.list.d/postgresql-pgdg.list > /dev/null
sudo apt update
sudo apt install postgresql-14 postgresql-client14
```

<br>

### Шаг 5

Выполняем резервное копирование всех баз данных сервера (если еще не делали):

``` bash
sudo -u postgres pg_dumpall > /tmp/dump_psql_before_update.dump
```

<br>

### Шаг 6

Cмотрим, как выглядят ваши кластеры. Сейчас должны быть две версии, обе в состоянии онлайн:

``` bash
pg_lsclusters
# Вы должны увидеть таблицу:
   Ver Cluster Port Status Owner    Data directory
   12 main  5432 online postgres /var/lib/postgresql/12/main
   14 main  5433 online postgres /var/lib/postgresql/14/main
```

<br>

### Шаг 7

При установке PostgreSql, установщик автоматически создает кластер с конфигурацией и базами данных,
   чтобы можно было начинать работу с базой данных сразу же. В случае обновления это лишняя операция и 
   кластер для 11 версии нужно удалить. Мы будем его создавать на основе кластера версии 12:

``` bash
sudo pg_dropcluster 14 main --stop
```

<br>

### Шаг 8

Остановите работу службы

``` bash
sudo systemctl stop postgresql
```

<br>

### Шаг 9

Сделайте запасное резервное копирование текущих каталогов с файлами базы данных и файлами конфигураций.
<br>Будьте осторожны, файлы базы данных могут занимать очень много места, поэтому, возможно, будет лучше выбрать другое место для их размещения. Позже, если обновление пройдет успешно, эти каталоги нужно будет удалить:

```bash
sudo cp -r /var/lib/postgresql/12/main/ /tmp/pg9.6-lib
sudo cp -r /etc/postgresql/12/main/ /tmp/pg9.6-etc
```

<br>

### Шаг 10

Запустите процедуру создания нового кластера на основе старого (обновление версии кластера):

``` bash
sudo pg_upgradecluster -m upgrade 12 main
```

<br>

### Шаг 11

Запустите службу:

``` bash
sudo systemctl start postgresql
```

<br>

### Шаг 12

Посмотрите, как выглядят ваши кластеры:

``` bash
pg_lsclusters
# Теперь вы должны увидеть вот такую таблицу, в которой версия 9.6 находится в состоянии down:
Ver Cluster Port Status Owner    Data directory
12  main  5433 down   postgres /var/lib/postgresql/12/main
14  main  5432 online postgres /var/lib/postgresql/14/main
```

<br>

### Шаг 13

Проверьте версии:

``` bash
sudo -u postgres psql -d userside -A -c "SELECT VERSION()"
```

<br>

## Навигация

[Вернуться в основное меню](../README.md)
<br> [Linux](../linux/README.md)
