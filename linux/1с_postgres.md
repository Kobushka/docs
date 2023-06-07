# Развертывание сервера 1С + PostgreSQL Pro

Для начала нужен сервер под Linux. У меня развернут Ubuntu 22.04 на облачной платформе.

## **Предварительная настройка Ubuntu**

---

Меняем локализацию сервера, чтобы потом не пришлось шаманить:

```bash
sudo dpkg-reconfigure locales
```

Выбираем `ru_RU.UTF-8 UTF-8`

Далее для корректной работы приложений необходимо установить шрифты из состава Microsoft Core Fonts:

```bash
sudo apt install ttf-mscorefonts-installer fontconfig
```

Устанавливаем дополнительные утилиты для дальнейшей работы
**(без них будут возникать ошибки в работе)**

```bash
sudo apt update
sudo apt install curl gnupg2 imagemagick unixodbc libgsf-1-14
```

В Интернетах и на форумах пишут, что imagemagick для работы 1С сервера нужна.

## **Устанавливаем PostgreSQL Pro**

---

Для сервера 1С существует специальные сборки PostgreSQL, которые можно получить с сайта - <https://1c.postgres.ru>

Выбираем нашу ОС, версию PostgreSQL и заполняем форму с указанием адреса электронной почты. На почту приходит письмо с инструкциями. Мне пришло письмо вот такого содержания:

```text
Вы получили это письмо, поскольку запрашивали инструкции по установке postgreSQL для 1с на сайте [1c.postgres.ru/](https://1c.postgres.ru/).

Используйте инструкции для установки postgreSQL для 1с. Обратите внимание, что команды должны выполняться от пользователя с правами суперпользователя.

wget https://repo.postgrespro.ru/1c-15/keys/pgpro-repo-add.sh
sh pgpro-repo-add.sh

Если наш продукт единственный Postgres на вашей машине и вы хотите сразу получить готовую к употреблению базу:

apt-get install postgrespro-1c-15

Если у вас уже установлен другой Postgres и вы хотите чтобы он продолжал работать параллельно (в том числе и для апгрейда с более старой major-версии):

apt-get install postgrespro-1c-15-contrib
/opt/pgpro/1c-15/bin/pg-setup initdb
/opt/pgpro/1c-15/bin/pg-setup service enable
/opt/pgpro/1c-15/bin/pg-setup service start
```

Как указано выше в письме, выполняем команды:

```bash
wget https://repo.postgrespro.ru/1c-15/keys/pgpro-repo-add.sh
```

 и запускаем на исполнение

```bash
sh pgpro-repo-add.sh
```

Устанавливаем сам сервер PostgreSQL

```bash
apt install postgrespro-1c-15
```

После окончания установки проверяем состояние сервера Postgre

```bash
sudo systemctl status postgrespro-1c-15.service
```

## **Устанавливаем PostgreSQL Pro**

Разрешаем автозагрузку службы

```bash
sudo systemctl enable postgrespro-1c-15.service
```

Входим на сервер под пользователем **postgres**

```bash
sudo -u postgres psql
```

Сразу задаем новый пароль этому пользователю:

```
\password postgres
```

Смотрим расположение **hba_file**:

```
SHOW hba_file;

-- Выдал ответ: ---------------------------------------
/var/lib/pgpro/1c-15/data/pg_hba.conf
(1 row)
```

Настраиваем удаленное подключение к базе данных через пароль.
Можно использовать любой редактор. В данном случае я использую **nano**

```bash
sudo nano /var/lib/pgpro/1c-15/data/pg_hba.conf
```

Листаем до конца файла и добавляем строку:

```text
host   all   all   all   password
```

После этого открываем файл **postgresql.conf**
Ищем параметр `listen_addresses` и меняем значение `localhost` на `*`

Выполняем рестарт службы сервера и проверяем статус после запуска.

```bash
sudo systemctl restart postgrespro-1c-15.service
sudo systemctl status postgrespro-1c-15.service
```

## **Открываем порты для удаленного доступа к серверу**

---

Открываем следующие порты в **iptables**

```bash
iptables -A INPUT -i eth0 -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -i eth0 -p udp --dport 80 -j ACCEPT
iptables -A INPUT -i eth0 -p tcp --dport 5432 -j ACCEPT
iptables -A INPUT -i eth0 -p udp --dport 5432 -j ACCEPT
iptables -A INPUT -i eth0 -p tcp --dport 443 -j ACCEPT
iptables -A INPUT -i eth0 -p udp --dport 443 -j ACCEPT
iptables -I INPUT 1 -p tcp --dport 1540:1541 -j ACCEPT
iptables -I INPUT 1 -p tcp --dport 1560 -j ACCEPT
```

Либо, если UFW

```bash
sudo ufw allow 80
sudo ufw allow 443
sudo ufw allow 1540:1541/tcp
sudo ufw allow 1560/tcp
```

## **Установка сервера 1С**

---

Теперь нам необходимо скачать дистрибутив сервера с портала 1С. Для этого логинимся под действующей учетной записью на [https://releases.1c.ru](https://releases.1c.ru/) и скачиваем файл
**Технологическая платформа 1С:Предприятия (64-bit) для Linux**.

Далее есть разные варианты, но чище получиться с созданием директории для распаковки файлов и дальнейшей работы

```
mkdir ~/1c
cd ~/1c
wget https://'ссылка_из_личного_кабинета_1С'/deb64_8_3_23_1688.tar.gz
tar xzvf deb64_8_3_23_1688.tar.gz
```

Далее необходимо установить пакеты распакованные выше.

По рекомендации с форума 1C, лучше каждый файл ставить по отдельности, так как существует чёткая последовательность: **common**, **crs**, **server**, **ws.**
Это позволит избежать редко возникающих ошибок при установке пакетов на различных системах.
Но, так как многие администраторы ленивы, то чаще без ошибок устанавливаются все пакеты по маске

```
sudo apt install ./*deb
```

Добавляем службу сервера платформы 1С и запускаем ее:

```bash
sudo ln /opt/1cv8/x86_64/8.3.23.1437/srv1cv8-8.3.23.1437@{,default}.service
sudo systemctl link /opt/1cv8/x86_64/8.3.23.1437/srv1cv8-8.3.23.1437@default.service
systemctl enable srv1cv8-8.3.23.1437@default.service
systemctl start srv1cv8-8.3.23.1437@default.service
```

Проверяем работу службы сервера 1С:Предприятие командой

```
systemctl status srv1cv8-8.3.23.1437@default.service
```

При корректной настройке среди выведенных строк должно быть `Active: active (running)`

Все...

<br>

## Навигация

[Вернуться в основное меню](../README.md)
<br> [Linux](../linux/README.md)

