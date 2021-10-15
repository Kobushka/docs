# Oxidized - установка и настройка

Настраивал по [инструкции](https://github.com/ytti/oxidized#debian-and-ubuntu)
Приведу основные моменты здесь

```bash
sudo add-apt-repository universe
sudo apt install ruby ruby-dev libsqlite3-dev libssl-dev pkg-config cmake libssh2-1-dev libicu-dev zlib1g-dev g++

sudo gem install oxidized
sudo gem install oxidized-script oxidized-web
```

Заводим пользователя oxidized для работы сервиса и конфигурируем oxidized

```bash
useradd oxidized
mkdir /home/oxidized
chown oxidized:oxidized /home/oxidized
```

Переходим на созданного пользователя

```bash
su - oxidized
oxidized
```

После выполнения команды выше будут созданы все папки и конфиги, необходимые для работы oxidized.

На маршрутизаторах Mikrotik заведем пользователя для сбора конфигов

```bash
/user add name=oxidized group=read password=YourStrongPassword address=IP_server
```

Правим конфиг oxidized для сбора конфигураций с маршрутизаторов

<details>
<summary>Файл /home/oxidized/.config/oxidized/config</summary>

```txt
---
username: oxidized
password: YourStrongPassword
// чтобы не указывать для каждого роутера вендора, можно указать тут, но лучше конечно указать для каждого роутера, если вендоров больше чем один
model: routeros
resolve_dns: true

//Периодичность снятия бэкапа в секундах
interval: 3600
use_syslog: false
debug: false
threads: 30

//таймаут сессии - 20 сек. на выгрузку конфигурации. Иногда приходится увеличивать это значение, если маршрутизатор выгружает конфиг дольше
timeout: 20

// Устанавливаем 3 попытки снять бэкап с каждого устройства, после чего считаем, что бэкап сделать не удалось
retries: 3
prompt: !ruby/regexp /^([\w.@-]+[#>]\s?)$/

//IP адрес и порт, на котором будет работать REST API (веб интерфейс по простому)
rest: 127.0.0.1:8888
next_adds_job: false

vars: {}
// Исключаем из бэкапа критичную информацию
remove_secret: true
groups: {}
models: {}

pid: "/home/oxidized/.config/oxidized/pid"

crash:
  directory: "/home/oxidized/.config/oxidized/crashes"
  hostnames: false

stats:
  history_size: 10

// установим тип подключения к управляемым устройствам - SSH
input:
  default: ssh
  debug: false
  ssh:
    secure: false

output:
  default: git
  git:
    user: oxidized
    email: oxidized@email.ru
    repo: "/home/oxidized/.config/oxidized"

// настройки конфига с устройствами, которые будем бекапить
source:
  default: csv
  csv:
    file: "/home/oxidized/.config/oxidized/mikrotik/router.db"
    delimiter: !ruby/regexp /:/
    map:
      name: 0
      model: 1
      ip: 2

// добавляем сюда устройства Mikrotik
model_map:
  juniper: junos
  cisco: ios
  mikrotik: routeros
```

</details>

<br>Вносим устройства, которые бекапим в конфиг

<details>
<summary>Файл /home/oxidized/.config/oxidized/mikrotik/router.db</summary>

```txt
# core of the network
CCR-1036:routeros:192.168.0.1
CCR-2004:routeros:192.168.0.2

# hAP
hAP-office-1:routeros:192.168.0.3
hAP-office-2:routeros:192.168.0.4

# cAP
cAP-5:routeros:192.168.0.5
cAP-6:routeros:192.168.0.6
```

</details>

<br> Как можно обратить внимание, в конфиге указан запуск только 127.0.0.1 с портом 8888.
Это сделано специально, так как установка в Oxidized паролей для пользователя (на момент написания этого мануала) не предусмотрено.
Для того, чтобы обеспечить требуемый безопасный вход, мы настроим NGINX для проверки пароля.<br>
Разработчики пока не планировали исправлять этот момент, поэтому обойдем этот недостаток с помощью Reverse-proxy.

Установим nginx:

```bash
sudo apt install nginx
```

Далее нам необходимо настроить работу NGINX как Reverse proxy. 
Для этого или правим конфиг по умолчанию, если сервер только для этого будет использован, либо добавим и настроим дополнительный конфиг для сервера.
В нашем случае мы будем править конфиг по умолчанию:

```bash
sudo nano /etc/nginx/sites-available/default
```

<details>
<summary>Файл /etc/nginx/sites-available/default</summary>

```txt
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name _;
    auth_basic “Username and Password Required”;
    auth_basic_user_file /etc/nginx/.htpasswd;
    
    location / {
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass http://127.0.0.1:8888;
    }
}

```

</details>

<br>Затем создаем пользователя и пароль:

```bash
sudo htpasswd /etc/nginx/.htpasswd username
```

<br>Разрешаем работу NGINX в firewall

```bash
sudo ufw app list
# в выводе будут список профилей приложений

Available applications:
  Nginx Full
  Nginx HTTP
  Nginx HTTPS
  OpenSSH

```

<br>Как показал вывод, есть три профиля, доступных для Nginx:

* Nginx Full: этот профиль открывает порт 80 (обычный веб-трафик без шифрования) и порт 443 (трафик с шифрованием TLS/SSL)
* Nginx HTTP: этот профиль открывает только порт 80 (обычный веб-трафик без шифрования)
* Nginx HTTPS: этот профиль открывает только порт 443 (трафик с шифрованием TLS/SSL)

Рекомендуется применять самый ограничивающий профиль, который будет разрешать заданный трафик. Сейчас нам нужно будет разрешить трафик на порту 80.

Для активации можно ввести следующую команду:

```bash
sudo ufw allow 'Nginx HTTP'
```

<br>Для проверки изменений введите:

```bash
sudo ufw status
# Вывод укажет, какой трафик HTTP разрешен:

Output
Status: active

To                         Action      From
--                         ------      ----
OpenSSH                    ALLOW       Anywhere                  
Nginx HTTP                 ALLOW       Anywhere                  
OpenSSH (v6)               ALLOW       Anywhere (v6)             
Nginx HTTP (v6)            ALLOW       Anywhere (v6)

```

<br>Теперь nginx будет работать на стандартном порту (или на том, который вы укажете в его настройках), при обращении к нему будет происходить авторизация и пользователь будет перенаправлен на адрес proxy_pass (127.0.0.1:8888 в нашем случае).<br><br><br>


### Навигация
[Вернуться в основное меню](../README.md)
<br> [Linux](../linux/README.md)
