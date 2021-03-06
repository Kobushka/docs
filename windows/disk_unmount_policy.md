# Диск Windows отключен в соответствии с установленной администратором политикой

## **Проблема:**

На одном из серверов с Windows Server 2016 после каждой перезагрузки сервера отключался дополнительный диск (не системный), подключенный в виде LUN с SAN хранилища по FC. Если открыть консоль управления дисками diskmgmt.msc, можно увидеть, что данный диск находится в автономном режиме Offline.
<br><br>При наведении курсора мыши в консоли управления diskmgmt.msc на отключенный диск появляется всплывающая надпись - ***Offline (The disk is offline because of policy set by an administrator).***
<br><br>Чтобы сделать этот диск доступным в Windows нужно щелкнуть по нему ПКМ и перевести в режим Online. Это придется делать при каждой перезагрузке сервера. Сомнительная перспектива.

## **Решение:**

Как оказалось, такая проблема может наблюдаться в кластерах или на виртуальных машинах с Windows, на которых общие диски могут быть доступны нескольким операционным системам. Это связано с наличием специальной политики SAN Policy, которая впервые появилась в Windows Server 2008. Эта политика управляет автоматическим монтированием внешних дисков и используется для защиты общих дисков, которые доступны нескольким серверам одновременно. По умолчанию в Windows Server для всех SAN дисков, кроме загрузочного, используется политика Offline Shared (VDS_SP_OFFLINE_SHARED). Вы можете изменить SAN Policy на OnlineAll с помощью Diskpart.

Отройте командную строку с правами администратора и выполните команду ***diskpart***. В контексте diskpart выведите текущую политику SAN:

```powershell
# Проверяем текущую политику
DISKPART>san
SAN Policy : Offline Shared

# Измените политику SAN Policy:
DISKPART> san policy=OnlineAll
DiskPart successfully changed the SAN policy for the current operating system.

# Еще раз проверим текущую политику:
DISKPART>san
SAN Policy : Online All

# Выберите ваш диск (в нашем примере индекс диска 2):
DISKPART>select disk 2

# Можете проверить его атрибуты:
DISKPART>attributes disk

# Проверьте, не включен ли атрибут Read-Only, если да, снимите его
# иначе при записи на диск будет появляться надпись The disk is write protected:
DISKPART>attributes disk clear readonly

# Переведите диск в online режим:
DISKPART>online disk
DiskPart successfully onlined the selected disk
```

Вы можете управлять дисками не только из Diskpart, но и с помощью встроенного PowerShell модуля Storage. Например, чтобы перевести диск в онлайн нужно выполнить команду:

``` powershell
Set-Disk 2 -IsOffline 0
```

Закройте diskpart, перезагрузите сервер и проверьте, что диск доступен после загрузки.
<br><br>Как оказалась, проблема с недоступностью подключенных дисков характерна не только для Windows Server, но и для десктопных версий Windows. Например, в Windows 10 при подключении внешнего диска по USB или SSD диска в диспетчере устройства вы также можете видеть такой же статус диска.
<br><br>В Windows 10 проблема с отключающийся Offline дисками исправляется аналогично: изменением политики SAN policy. Если диск новый, возможно понадобится инициализировать его и создать на нем разделы с файловой системой.
<br><br>

### Навигация

[Вернуться в основное меню](../README.md)
<br> [Windows](../windows/README.md)
