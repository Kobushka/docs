# **Загрузка на сервер файлов больше 1Мб**

## **Описание проблемы:**

Столкнулись вчера с проблемой. Не загружаются файлы больше 1Мб на сервер.<br>
Запросы проходят через сервер обратного прокси, настроенного на NGINX.<br>
Перечитаны все доступные мануалы и сообщения на форумах.<br>
Проверены все конфиги на сервере, где работает приложение (tomcat, php, nginx и т.д.)<br>
Проверены конфиги на сервере прокси.<br>
Но все без изменений.
<br><br>

## **Что сделали чтобы исправить:**

Оказалось, что на сервере прокси необходимо добавить строки в конфиг, чтобы сам прокси начал пропускать файлы большего размера, чем дефолтные значения.
<br><br>
Необходимо внести изменения в конфиг NGINX в двух местах.

<details>
<summary>Файл /etc/nginx/nginx.conf</summary>

```txt

http {

    ...
    client_max_body_size 10m;
    ...

}

```

</details>

<details>
<summary>Файл /etc/nginx/proxy_param</summary>

```txt
# Default
proxy_set_header Host $http_host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;

# Added lines to config
proxy_buffers 16 16k;
proxy_buffer_size 32k;

# prevent regular 504 Gateway Time-out message
proxy_connect_timeout 600;
proxy_send_timeout 600;
proxy_read_timeout 600;
send_timeout 600;

proxy_set_header X-Query-String $request_uri;
proxy_set_header X-Host $host;
proxy_set_header X-Remote-Addr $remote_addr;
proxy_set_header X-Request-Filename $request_filename;
proxy_set_header X-Request-URI $request_uri;
proxy_set_header X-Server-Name $server_name;
proxy_set_header X-Server-Port $server_port;
proxy_set_header X-Server-Protocol $server_protocol;

proxy_intercept_errors on;

# apparently this is how to disable cache?
expires -1;

```

</details><br><br>

После добавления этих строк в конфиг все сайты после прокси получили возможность загружать файлы, больше 1Мб, на сервер.
<br><br>

### Навигация
[Вернуться в основное меню](../README.md)
<br> [Linux](../linux/README.md)