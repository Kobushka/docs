# Настройки, которые необходимы на сетевых устройствах ROS

Минимально необходимо добавить следующие настройки на устройства Mikrotik

```bash
/snmp community 
set [ find default=yes ] addresses=127.0.0.1/32 authentication-password=YoureStrongPassword encryption-password=YoureStrongPassword
/snmp community 
add addresses=local_net/CIDR authentication-password=YoureStrongPassword_AuthPass encryption-password=YoureStrongPassword_EncPass name=snmp3_zabbix security=private
/snmp set 
enabled=yes engine-id=UniqueID trap-community=snmp3_name trap-interfaces=bridge1 trap-version=3
```

Что получаем в итоге.
Доступ к профилю SNMP по умолчанию будет ограничен только локальным устройством.
Новый, созданный профиль будет доступен для опроса сетевого устройства из локальной сети.

Zabbix в настройках сетевого устройства, необходимо выбрать (укажу здесь описание ручных исправлений):
Заходим в Configuration -->> Hosts и либо заводим новый хост или выбираем существующий.
Далее, в Host -->> Interfaces необходимо настроить SNMP
выбираем SNMP version -->> SNMPv3 и заполняем следующие поля
Context name -->> snmpv3_zabbix (имя указывали на маршрутизаторах)
Security name -->> snmpv3_zabbix (у меня сработало имя одинаковое с Контекстом)
Security level -->> authPriv (обязательно)
Authentication protocol -->> MD5
Authentication passphrase -->> YoureStrongPassword_AuthPass (см. профиль SNMP на маршрутизаторе)
Privacy protocol -->> DES
Privacy passphrase -->> YoureStrongPassword_EncPass

После внесения всех изменений сетевое устройство Mikrotik начнет мониторится и в Zabbix начнут поступать данные с устройства.
