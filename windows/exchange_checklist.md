# CheckList проверки работоспособности сервера Exchange

## Задача:

<br>Возникают ситуации, когда необходимо выполнить быструю проверку работоспособности почтового сервера Exchange. 
<br><br>Ниже пошаговое руководство по проверке сервера Exchange.
<br><br>

## Пошаговое руководство

<br>

### **Шаг 1** - Проверяем службы самого Exchange

<br>Управление компьютером → Службы → сортируем по имени (если англ. версия) или по запуску если русская<br><br>
Проверяем службы – должны быть запущены все в имени которых есть Microsoft Exchange, кроме Notification Broker (он нужен только при старте системы для проверки Exchange самого себя)<br><br>
Или тоже самое в PowerShell<br><br>

```powershell
Test-ServiceHealth
```

в выводе должно все быть в состоянии TRUE<br><br>

### **Шаг 2** - Проверяем корректность монтирования почтовых баз данных на почтовом сервере

```powershell
 Test-MAPIConnectivity [-Server SRVname] 
```

в колонке Result все строки должны быть SUCCESS<br><br>

### **Шаг 3** - Проверяем состояние индексов почтовых баз

```powershell
Get-MailboxDatabaseCopyStatus * | ft -auto

Name                           Status   CopyQueueLength   ReplayQueueLength    LastInspectedLogTime   ContentIndexState
-------                        -------  ----------------  ------------------   ---------------------  --------------------
MailBoxDatabaseONE\EXCH01      Mounted  0                 0                                           Healthy
MailBoxDatabaseTWO\EXCH01      Mounted  0                 0                                           FailedAndSuspended
MailBoxDatabaseTHREE\EXCH01    Mounted  0                 0                                           Failed

```

Все что не Healthy - это не нормально...
<br><br>

### **Шаг 4** - Просмотр журналов работы 

<br>В первую очередь проверяем основные журналы на предмет замечаний и ошибок

Windows Logs → Application<br>
Windows Logs → System<br>
И более подробно по ошибкам если есть можно посмотреть в<br>
Application and Services Logs → All
<br><br>

### **Шаг 5** - Проверяем установлены ли доверительные отношения с контроллером домена

```powershell
Test-ComputerSecureChannel
```

Если все в порядке, то должен вернуть TRUE<br><br>

### **Шаг 6** - Проверяем применение групповых политик домена

```powershell
gpupdate /force
```

Если в выводе никаких ошибок нет, то все в порядке<br><br>

### **Шаг 7** - Проверка правильности настроек DNS-сервера

<br>Так как Exchange работает только в домене, то в настройке сетевой карты должен быть только сервера DNS нашего домена, которым доверяли бы все участники.<br><br>

```diff
! Не должно быть никаких DNS-серверов от Гугл, Яндекс и т.д. (ТОЛЬКО ДОМЕННЫЕ СЕРВЕРЫ)
```

Утилита nslookup поможет проверить корректность работы dns-серверов.

### **Шаг 8** - Проверка контроллера домена ActiveDirectory

```powershell
dcdiag
```

в выводе все должно быть PASSED<br><br>

Если контроллеров в домене несколько, то обязательно нужно проверить как выполняется репликация между ними.<br>
Репликацию можно выполнить вручную в оснастке<br><br>
Active Directory Sites and Services → NTDS setting → пр. кнопка мыши → Replicate now<br><br>

Дальше нужно глянуть отработала ли репликация<br>

```powershell
repadmin /replsummary
```

```diff
! Возможно обновиться этот чек-лист, но пока хватает быстро проверить все ли в порядке с почтовым сервером
```

<br>

### Навигация

[Вернуться в основное меню](../README.md)
<br> [Windows](../windows/README.md)
