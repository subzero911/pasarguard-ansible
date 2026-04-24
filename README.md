# Marzban Ansible

Ansible playbook для автоматического развертывания Marzban Node на серверах.

## Конфигурация

Перечислите свои ноды в файле `inventory.ini`.

## Установка сертификатов

Сертификаты клиента панели нужно вставить в файлы `host_vars/{имя_ноды}.yml`, в переменную `panel_client_cert`.

Пример:
```yaml
panel_client_cert: |
  -----BEGIN CERTIFICATE-----
  MIIDxTCCAq2gAwIBAgIQAqxcJmoLQJuPC3nyrkYldzANBgkqhkiG9w0BAQUFADBs
  ...
  -----END CERTIFICATE-----
```

## Запуск Ansible

Для запуска playbook выполните команду:

```bash
ansible-playbook -i inventory.ini install-node.yml -v
```

Где:
- `-i inventory.ini` - указывает файл с инвентарем хостов
- `install-node.yml` - основной playbook для развертывания
- `-v` - выводит минимальный лог

## Что делает playbook

- Устанавливает Docker на сервер
- Создает необходимые директории
- Генерирует SSL сертификаты для узла
- Устанавливает клиентский сертификат панели
- Запускает Marzban Node контейнер
- Генерирует xray ключи и shortId
- Добавляет агент Dozzle

---

## Обновление Xray

Для обновления Xray core на всех нодах:

```bash
ansible-playbook -i inventory.ini update-xray.yml -v
```

Playbook:
- Скачивает указанную версию Xray
- Обновляет бинарный файл
- Настраивает путь к Xray в docker-compose.yml
- Перезапускает контейнер marzban-node

Версию Xray можно изменить в переменной `xray_version` в начале playbook.

---

## Бэкап и восстановление главной панели

### Установка бэкапа на Google Drive

Для настройки автоматического бэкапа главной панели Marzban на Google Drive:

```bash
ansible-playbook -i inventory.ini install-backup.yml -v
```

**Требования:**
- В `inventory.ini` должен быть указан хост в группе `[marzban_main]`
- После запуска playbook необходимо настроить rclone с Google Drive (инструкции будут выведены в конце выполнения)

**Что делает playbook:**
- Устанавливает rclone, sqlite3, rsync
- Развертывает скрипт бэкапа `/usr/local/bin/backup-marzban.sh`
- Развертывает скрипт восстановления `/usr/local/bin/restore-marzban.sh`
- Настраивает cron для ежедневного бэкапа (по умолчанию в 00:00 UTC = 03:00 по UTC+3)
- Хранит бэкапы за последние 7 дней

**Что бэкапится:**
- `/var/lib/marzban/db.sqlite3` — база данных (online-backup без остановки)
- `/var/lib/marzban/` — данные Marzban (xray-конфиги, сертификаты)
- `/opt/marzban-stack/` — конфигурация стека (docker-compose.yml, .env, Caddyfile)

**Ручной запуск бэкапа:**
```bash
/usr/local/bin/backup-marzban.sh
```

**Логи бэкапа:**
```bash
tail -f /var/log/backup-marzban.log
```

### Восстановление из бэкапа

**Посмотреть список бэкапов:**
```bash
rclone ls gdrive:marzban-backups/
```

**Восстановить интерактивно:**
```bash
/usr/local/bin/restore-marzban.sh
```

**Восстановить конкретный бэкап:**
```bash
/usr/local/bin/restore-marzban.sh marzban_backup_20240424_030000.tar.gz
```

**Логи восстановления:**
```bash
tail -f /var/log/restore-marzban.log
```

Скрипт восстановления:
- Останавливает docker-compose стек
- Восстанавливает базу данных, данные и конфигурацию
- Запускает стек заново

### Настройка rclone с Google Drive

После запуска playbook необходимо настроить rclone. Два варианта:

**Вариант А: SSH port forwarding (рекомендуется)**
```bash
ssh -L 53682:localhost:53682 root@<IP_ПАНЕЛИ>
# На сервере:
rclone config
# → n → gdrive → drive → при авторизации выбрать 'y'
# Откройте URL в браузере на локальной машине
```

**Вариант Б: Через локальный rclone**
```bash
# На локальной машине:
brew install rclone
rclone authorize 'drive'
# Авторизуйтесь в браузере и скопируйте токен

# На сервере:
rclone config
# → n → gdrive → drive → вставьте токен при запросе
```

После настройки проверьте подключение:
```bash
rclone lsd gdrive:
```