# PasarGuard Ansible

Ansible playbook для автоматического развертывания PasarGuard Node на серверах.

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
- Запускает PasarGuard Node контейнер
- Генерирует xray ключи и shortId
- Добавляет агент Dozzle

## Caddy stub site на нодах

Playbook `install-caddy-stub.yml` поднимает Caddy с заглушкой-сайтом ("Купи слона") на каждой ноде. Используется для Reality fallback и self-steal TLS fingerprint.

**Переменные** (задаются в `inventory.ini` или через `-e`):

| Переменная | По умолчанию | Описание |
|---|---|---|
| `caddy_stub_domain` | `sw.smart-sausage.de` | Домен заглушки |
| `caddy_stub_port` | `8443` | Порт заглушки |

### Запуск на всех нодах

```bash
ansible-playbook -i inventory.ini install-caddy-stub.yml -v
```

### Запуск с другим доменом/портом

```bash
ansible-playbook -i inventory.ini install-caddy-stub.yml \
  -e "caddy_stub_domain=sw.your-domain.com" \
  -e "caddy_stub_port=8443" -v
```

### Запуск на одной ноде

```bash
ansible-playbook -i inventory.ini install-caddy-stub.yml -l node-nsk-1 -v
```

**Что делает playbook:**
- Создаёт `/opt/caddy-stub` с `Caddyfile` и `srv/index.html`
- Поднимает `caddy-stub` в Docker с `--network host`
- Открывает порт в UFW
- Caddy слушает `:80` (для ACME) и `domain:port` (для заглушки)

---

## Настройка UFW на нодах

Playbook `setup-ufw-node.yml` настраивает UFW на нодах: выставляет политики по умолчанию, открывает нужные порты и ограничивает порты `62050`/`62051` только с IP панели.

**Обязательный параметр:** `panel_ip` — внешний IP сервера панели.

### DEV (панель smart-sausage.de, IP `185.121.134.157`)

```bash
ansible-playbook -i inventory.ini setup-ufw-node.yml -e "panel_ip=185.121.134.157" -v
```

### PROD (панель domoi-panel.ru, IP `159.194.221.58`)

```bash
ansible-playbook -i inventory.ini setup-ufw-node.yml -e "panel_ip=159.194.221.58" -v
```

Запуск на конкретной ноде:

```bash
ansible-playbook -i inventory.ini setup-ufw-node.yml -e "panel_ip=159.194.221.58" -l node-nsk-1 -v
```

---

## Бэкап и восстановление главной панели

Playbook `install-backup.yml` принимает параметр `backup_backend`:

| Значение | Хранилище | Среда |
|----------|-----------|-------|
| `gdrive` (по умолчанию) | Google Drive | DEV |
| `yadisk` | Яндекс.Диск | PROD |

### DEV — Google Drive

```bash
ansible-playbook -i inventory.ini install-backup.yml -l dev-panel -e "backup_backend=gdrive" -v
```

### PROD — Яндекс.Диск

```bash
ansible-playbook -i inventory.ini install-backup.yml -l prod-panel -e "backup_backend=yadisk" -v
```

**После запуска** на сервере настроить rclone:

- **Google Drive:** `rclone authorize 'drive'` локально, вставить токен на сервере через `rclone config`
- **Яндекс.Диск:** `rclone config` на сервере → `yadisk` → `yandex` → авторизация через браузер

---

### Установка бэкапа (детали)

**Требования:**
- В `inventory.ini` должен быть указан хост в группе `[pasarguard]`
- После запуска playbook необходимо настроить rclone (инструкции будут выведены в конце выполнения)

**Что делает playbook:**
- Устанавливает rclone, sqlite3, rsync
- Развертывает скрипт бэкапа `/usr/local/bin/backup-pasarguard.sh`
- Развертывает скрипт восстановления `/usr/local/bin/restore-pasarguard.sh`
- Настраивает cron для ежедневного бэкапа (по умолчанию в 00:00 UTC = 03:00 по UTC+3)
- Хранит бэкапы за последние 7 дней

**Что бэкапится:**
- `/var/lib/pasarguard/` — данные PasarGuard (xray-конфиги, сертификаты)
- `/opt/pasarguard/` — конфигурация стека (docker-compose.yml, .env, Caddyfile)

**Ручной запуск бэкапа:**
```bash
/usr/local/bin/backup-pasarguard.sh
```

**Логи бэкапа:**
```bash
tail -f /var/log/backup-pasarguard.log
```

### Настройка rclone

После запуска playbook необходимо настроить rclone. Рекомендуемый вариант — SSH port forwarding.

**Вариант А: SSH port forwarding (рекомендуется)**

Шаг 1: На локальной машине запустите SSH с пробросом порта:
```bash
ssh -L 53682:localhost:53682 root@<IP_ПАНЕЛИ>
```

Шаг 2: На сервере запустите rclone config:
```bash
rclone config
```

Шаг 3: Ответьте на вопросы:
- `No remotes found - make a new one?` → введите `n` (new)
- `name>` → введите `gdrive`
- `Type of storage>` → введите `drive` или выберите номер `17`
- `scope>` → нажмите Enter (по умолчанию)
- `root_folder_id>` → нажмите Enter
- `service_account_file>` → нажмите Enter
- `Use auto config?` → введите `y` (yes)
- **Откроется URL в браузере на вашей локальной машине** — авторизуйтесь в Google
- `Configure this as a team drive?` → введите `n` (no)
- `Use this remote?` → введите `y` (yes)
- `y/n>` → введите `q` (quit)

Шаг 4: Проверьте подключение:
```bash
rclone lsd gdrive:
```

---

**Вариант Б: Через токен (если нет возможности открыть браузер с сервера)**

Шаг 1: На локальной машине установите rclone и получите токен:
```bash
brew install rclone
rclone authorize 'drive'
```
- Откроется браузер — авторизуйтесь в Google
- Скопируйте полученный токен (длинная строка)

Шаг 2: На сервере запустите rclone config:
```bash
rclone config
```

Шаг 3: Ответьте на вопросы:
- `No remotes found - make a new one?` → введите `n` (new)
- `name>` → введите `gdrive`
- `Type of storage>` → введите `drive` или выберите номер `17`
- `scope>` → нажмите Enter
- `root_folder_id>` → нажмите Enter
- `service_account_file>` → нажмите Enter
- `Use auto config?` → введите `n` (no)
- **`Enter a value>` → вставьте скопированный токен**
- `Configure this as a team drive?` → введите `n` (no)
- `Use this remote?` → введите `y` (yes)
- `y/n>` → введите `q` (quit)

Шаг 4: Проверьте подключение:
```bash
rclone lsd gdrive:
```

### Восстановление из бэкапа

**Требования для восстановления:**

Если устанавливали бэкап через `install-backup.yml` — всё уже готово.

Если восстановление на новом сервере, нужно:
- Docker и docker-compose
- rclone с настроенным Google Drive (инструкции выше)
- sqlite3, rsync (устанавливаются playbook)

**Что восстанавливается из бэкапа:**
- `/var/lib/pasarguard/` — данные PasarGuard (xray-конфиги, сертификаты)
- `/opt/pasarguard/` — конфигурация стека (docker-compose.yml, .env, Caddyfile)
- Структура директорий — создаёт `/var/lib/pasarguard` и `/opt/pasarguard` если их нет

**Что НЕ восстанавливается (нужно установить отдельно):**
- Docker и docker-compose
- rclone (если не было до бэкапа)

**Посмотреть список бэкапов:**
```bash
rclone ls gdrive:pasarguard-backups/
```

**Восстановить интерактивно:**
```bash
/usr/local/bin/restore-pasarguard.sh
```

**Восстановить конкретный бэкап:**
```bash
/usr/local/bin/restore-pasarguard.sh pasarguard_backup_20240424_030000.tar.gz
```

**Логи восстановления:**
```bash
tail -f /var/log/restore-pasarguard.log
```

Скрипт восстановления:
- Останавливает docker-compose стек
- Восстанавливает базу данных, данные и конфигурацию
- Запускает стек заново