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
ansible-playbook -i inventory.ini playbook.yml -v
```

Где:
- `-i inventory.ini` - указывает файл с инвентарем хостов
- `playbook.yml` - основной playbook для развертывания
- `-v` - выводит минимальный лог

## Что делает playbook

- Устанавливает Docker на сервер
- Создает необходимые директории
- Генерирует SSL сертификаты для узла
- Устанавливает клиентский сертификат панели
- Запускает Marzban Node контейнер
- Генерирует xray ключи и shortId
- Добавляет агент Dozzle