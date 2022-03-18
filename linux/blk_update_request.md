# blk_update_request: I/O error, dev fd0, sector 0

## **Проблема:**

На виртуальной машине заметил ошибки в логах постоянные ошибки, типа:

```bash
blk_update_request: I/O error, dev fd0, sector 0
hv_vmbus: probe failed for device vmbus_10 (-19)
```

Нужно разобраться и найти решение.

## **Решение:**

Ошибка связана с попытками подключить Floppy дисковод, который в ВМ отсутствует. Решается все довольно просто, путем отключения драйвера Floppy в системе.

Для того чтобы этого не происходило выполняем следующие команды в терминале:

```bash
echo "blacklist floppy" | sudo tee /etc/modprobe.d/blacklist-floppy.conf
sudo rmmod floppy
sudo update-initramfs -u
```

Перегружаем систему и используем дальше. В логах таких ошибок уже не будет.

<br>

## Навигация

[Вернуться в основное меню](../README.md)
<br> [Linux](../linux/README.md)
