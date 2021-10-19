# Тестовый Tomcat - установка и настройка

Потербовался сервер для тестов приложений от программистов.<br>
Поступили требования:

* сервер на Ubuntu-20.04
* NGINX последней версии
* PostgreSQL 12 версии
* PHP-fpm последняя версия
* JDK версия 8
* Tomcat версии 9

## **Настройка сервера** (пока в ручную):

После выполнения базовых настроек сервера.
Начнем настроивать требуемые сервисы

### NGINX + PHP-fpm

Для начала устанавливаем сервер NGINX и PHP-fpm.

```bash
sudo apt install nginx php-fpm
```

По условию задачи никаких особых настроек мы не делаем.
Частично сервер настраивают программисты для конкретных веб-приложений
Поэтому делаем минимальные настройки для работы сервера

```txt
server {

	listen 80 default_server;
	listen [::]:80 default_server;
	server_tokens off;

	root /opt/web;
	index index.php index.html index.htm;

	server_name _;

	location / {
		# First attempt to serve request as file, then
		# as directory, then fall back to displaying a 404.
		try_files $uri $uri/ =404;
	}

	location ~ \.php$ {
                include snippets/fastcgi-php.conf;
                fastcgi_pass unix:/run/php/php-fpm.sock;
        }

	location ~* ^.+\.(css|ogg|ogv|svg|svgz|eot|otf|woff|mp4|ttf|rss|atom|jpg|jpeg|gif|png|ico|zip|tgz|gz|rar|bz2|doc|xls|exe|ppt|tar|mid|midi|wav|bmp|rtf)$ {
                expires max;
        }

        location ~ /\.ht {
                deny all;
        }
}
```

Перезапустим и проверим работу сервера NGINX

```bash
sudo systemctl start nginx
sudo systemctl status nginx
```

При входе на страницу сервера будет страница 403 Forbidden
Это мы исправим позже после установки Tomcat

### PostgreSQL

Для установки воспользуемся официальным мануалом по [адресу](https://www.postgresql.org/download/linux/ubuntu/)

```bash
# Create the file repository configuration:
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'

# Import the repository signing key:
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -

# Update the package lists:
sudo apt update

# Install the latest version of PostgreSQL.
# If you want a specific version, use 'postgresql-12' or similar instead of 'postgresql':
sudo apt install postgresql-12 -y
```

В нашу задачу полная настройка сервера PostgreSQL не входит. Дальше не настраиваем.

### Установка Tomcat + JDK8

JDK устанавливаем по [мануалу](https://www.javahelps.com/2015/03/install-oracle-jdk-in-ubuntu.html)

```bash
# скачать файл с офф.страницы

# создадим каталог для работы
sudo mkdir /usr/lib/jvm
cd /usr/lib/jvm
# распакуем скачанный файл
sudo tar -xvzf ~/jdk-8u291-linux-x64.tar.gz 

# настроим окружение для работы
sudo nano /etc/environment 
sudo update-alternatives --install "/usr/bin/java" "java" "/usr/lib/jvm/jdk1.8.0_291/bin/java" 0
sudo update-alternatives --install "/usr/bin/javac" "javac" "/usr/lib/jvm/jdk1.8.0_291/bin/javac" 0
sudo update-alternatives --set java /usr/lib/jvm/jdk1.8.0_291/bin/java
sudo update-alternatives --set javac /usr/lib/jvm/jdk1.8.0_291/bin/javac

# проверим созданные записи в окружении
update-alternatives --list java
update-alternatives --list javac

# проверим версию java чтобы окончательно убедиться что все работает
java -version
```

В официальной документации по Tomcat [ссылка](https://tomcat.apache.org/)
все описано, но для большей наглядности проще воспользоваться любым мануалом с описанием всех шагов.
Приведу необходимые шаги для установки сервера Tomcat

```bash
# создаем пользователя и группу для Tomcat
sudo groupadd tomcat
sudo useradd -s /bin/false -g tomcat -d /opt/tomcat tomcat

# качаем и распаковываем сервер Tomcat
sudo wget https://dlcdn.apache.org/tomcat/tomcat-9/v9.0.54/bin/apache-tomcat-9.0.54.tar.gz
sudo mkdir /opt/tomcat
sudo tar zxvf apache-tomcat-9.0.54.tar.gz -C /opt/tomcat --strip-components 1

# Установим необходимые разрешения
sudo chgrp -R tomcat /opt/tomcat
cd /opt/tomcat
sudo chmod -R g+r conf
sudo chmod g+x conf
sudo chown -R tomcat webapps/ work/ temp/ logs/
```

Создадим файл для запуска через systemd

```bash
sudo nano /etc/systemd/system/tomcat.service
```

И внесем в него следующее содержимое:

```txt
[Unit]
Description=Apache Tomcat Web Application Container
After=network.target

[Service]
Type=forking

Environment=JAVA_HOME=/usr/lib/jvm/jdk1.8.0_291/
Environment=CATALINA_PID=/opt/tomcat/temp/tomcat.pid
Environment=CATALINA_HOME=/opt/tomcat
Environment=CATALINA_BASE=/opt/tomcat
Environment='CATALINA_OPTS=-Xms512M -Xmx1024M -server -XX:+UseParallelGC'
Environment='JAVA_OPTS=-Djava.awt.headless=true -Djava.security.egd=file:/dev/./urandom'

ExecStart=/opt/tomcat/bin/startup.sh
ExecStop=/opt/tomcat/bin/shutdown.sh

User=tomcat
Group=tomcat
UMask=0007
RestartSec=10
Restart=always

[Install]
WantedBy=multi-user.target
```

Запустим и проверим работу сервиса

```bash
sudo systemctl daemon-reload
sudo systemctl start tomcat.service
sudo systemctl status tomcat.service
```

Убеждаемся что все работает и отдаем доступ для программистов.
Если что-то понадобиться еще дополню.

## 19 октября 2021
Обновление и дополнение

**Необходимо сделать в конфиге переадресацию на приложение, которое работает на Tomcat

```txt
upstream tomcat {
  server 127.0.0.1:8080 fail_timeout=0;
}

server {
  listen 80;
  listen [::]:80;

  server_name xyz.mydomain.com;

  access_log /var/log/nginx/fr.corp.platinka.ru-access.log;
  error_log  /var/log/nginx/fr.corp.platinka.ru-error.log;

  location / {
    include proxy_params;
    proxy_pass http://tomcat/;
  }
}

```

Этого оказалось достаточно для экспериментов. На этом пока все...


```diff
! В планах добавить установку с помощью Ansible
```

### Навигация
[Вернуться в основное меню](../README.md)
<br> [Linux](../linux/README.md)
