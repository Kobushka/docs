# Настройка CapsMan для дома

## Задача.
Сосед попросил настроить ему беспроводную сеть для коттеджа.
Нужно сделать гостевую сеть, отдельно от основной домашней сети.

## Исходные данные

**Уже были куплены:**
* [Mikrotik hAP ac²](https://mikrotik.com/product/hap_ac2) - 1 шт.
* [Mikrotik сAP ac](https://mikrotik.com/product/cap_ac) - 2 шт.

**Адресация в локальной сети:**
* адрес маршрутизатора - 192.168.88.1
* адреса точек доступа - 192.168.88.[2, 3]

<br>Подключение к провайдеру уже настроено. Нужно только настроить беспроводную сеть с помощью Capsman

На шлюзовом маршрутизаторе настраиваем Capsman:

```bash
# Добавим бридж для гостей
/interface bridge
    add arp=proxy-arp fast-forward=no name=bridge-guest
# Настроим каналы для беспроводной сети
/caps-man channel
    add band=2ghz-g/n frequency=2412,2437,2462 name=channel_24 tx-power=20
    add band=5ghz-n/ac frequency=5180,5220,5260 name=channel_5 tx-power=20
# Настроим безопасность для подключения клиентов
/caps-man security
    add authentication-types=wpa2-psk encryption=aes-ccm,tkip name=guest passphrase=KeyGuest
    add authentication-types=wpa2-psk encryption=aes-ccm,tkip name=home passphrase=KeyHome
# Настроим куда отправляем клиентов из разных сетей и могут ли клиенты видеть друг друга
/caps-man datapath
    add bridge=bridge-guest name=datapath_guest
    add bridge=bridge-home client-to-client-forwarding=yes name=datapath_home
# Настравиваем конфигурации для провижеров
/caps-man configuration
    add channel=channel_5 country=russia2 datapath=datapath_guest datapath.bridge=bridge-guest hw-protection-mode=rts-cts max-sta-count=15 mode=ap name=cfg_guest_5 security=guest ssid=Guest-5
    add channel=channel_24 country=russia2 datapath=datapath_guest datapath.bridge=bridge-guest hw-protection-mode=rts-cts max-sta-count=15 mode=ap name=cfg_guest_24 security=guest ssid=Guest-2
    add channel=channel_5 datapath=datapath_home datapath.bridge=bridge-home hw-protection-mode=rts-cts max-sta-count=25 mode=ap name=cfg_home_5 rx-chains=0,1,2 security=home ssid=Home-5 tx-chains=0,1,2
    add channel=channel_24 datapath=datapath_home datapath.bridge=bridge-home hw-protection-mode=rts-cts max-sta-count=25 mode=ap name=cfg_home_24 rx-chains=0,1,2 security=home ssid=Home-2 tx-chains=0,1,2
# Куда из Capsman будет лететь весь трафик
/caps-man manager interface
    add disabled=no interface=bridge-home
# Настраиваем профиженеры для точек доступа
/caps-man provisioning
    add action=create-dynamic-enabled hw-supported-modes=gn master-configuration=cfg_home_24 name-prefix=24 slave-configurations=cfg_guest_24
    add action=create-dynamic-enabled hw-supported-modes=ac master-configuration=cfg_home_5 name-format=prefix-identity name-prefix=5 slave-configurations=cfg_guest_5
# Настраиваем аксес-листы для отключения клиентов если они выходят из зоны доступа точек
# Устройства сами не отключаются даже если есть более лучшая точка для выхода в сеть
# С этими настройками получим аналог бесшовной сети для клиентов
/caps-man access-list
    add action=accept allow-signal-out-of-range=10s disabled=no signal-range=-79..120 ssid-regexp=*
    add action=reject allow-signal-out-of-range=10s disabled=no signal-range=-120..-80 ssid-regexp=*
# Разрешаем работу Capsman
/caps-man manager
    set enabled=yes
# Настраиваем DHCP сервер для гостевой сети
/ip pool
add name=dhcp_pool_guest ranges=192.168.100.20-192.168.100.254
/ip dhcp-server
add add-arp=yes address-pool=dhcp_pool_guest disabled=no interface=bridge-guest lease-time=1h name=dhcp-guest

```

<br>На точках доступа нужно минимально сделать следующее:

```bash
/interface wireless cap 
    set caps-man-addresses=192.168.88.1 enabled=yes interfaces=wlan1,wlan2
```
<br>

```diff
+ Нужно помнить, что если мы добавим в локальный бридж точек доступа беспроводные интерфейсы, 
+ то получим отвалы клиентов от подключения, по причине не возможности выдачи второго адреса из пула DHCP. 
! Адреса должен выдавать только Capsman.
```

### Навигация
[Вернуться в основное меню](../README.md)
<br> [Mikrotik](../mikrotik/README.md)
