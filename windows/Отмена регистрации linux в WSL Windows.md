# Дистрибутивы Linux в WSL Windows

Мне понадобилось переустановить дистрибутив Linux для очередных экспериментов.
Оказалось, что это сделать очень просто.
Запускаем PowerShell и далее в нем в две команды отменяем регистрацию необходимого дистрибутива.

```powershell
# получим список установленных дситрибутивов
# или второй вариант wsl -l -v
PS C:\Users\UserName> wsl --list
Распределения подсистемы Windows для Linux:
Ubuntu-20.04 (по умолчанию)

# Отменим регистрацию дистрибутива
PS C:\Users\UserName> wsl --unregister Ubuntu-20.04
Отмена регистрации...

# Еще раз проверим что действительно регистрация дистрибутива отменена
PS C:\Users\UserName> wsl --list
Нет установленных дистрибутивов подсистемы Windows для Linux.
Дистрибутивы можно установить из Microsoft Store:
https://aka.ms/wslstore

```

Для того чтобы заново получить чистый дистрибутив, скачаем и установим дистрибутив из Интернет-магазина Microsoft

```powershell
# Получим список доступных для установки дистрибутивов из Интернет-магазина Microsoft
PS C:\Users\UserName> wsl --list --online
Ниже приведен список допустимых распределений, которые можно установить.
Установите с помощью команды wsl --install -d <Distro>.

NAME            FRIENDLY NAME
Ubuntu          Ubuntu
Debian          Debian GNU/Linux
kali-linux      Kali Linux Rolling
openSUSE-42     openSUSE Leap 42
SLES-12         SUSE Linux Enterprise Server v12
Ubuntu-16.04    Ubuntu 16.04 LTS
Ubuntu-18.04    Ubuntu 18.04 LTS
Ubuntu-20.04    Ubuntu 20.04 LTS

# Установим конкретный дистрибутив, который нам необходим для работы
PS C:\Users\e.kobushka> wsl --install -d Ubuntu-20.04
Распределение Ubuntu 20.04 LTS уже установлено.
Запуск Ubuntu 20.04 LTS...

```

В открывшемся окне уже делаем все настройки системы.

```diff
! Следует помнить, что все файлы старого дистрибутива будут удалены.
! Обязательно сохраните все необходимое
```

### Навигация
[Вернуться в основное меню](../README.md)
<br> [Mikrotik](../mikrotik/README.md)
<br> [Linux](../linux/README.md)
<br> [Mikrotik](../mikrotik/README.md)
