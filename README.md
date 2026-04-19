# Marzban Ansible

Ansible playbook для автоматического развертывания Marzban Node на серверах.

## Установка сертификата

Сертификат клиента панели нужно вставить в файл `group_vars/marzban_nodes.yml` в переменную `panel_client_cert`.

Пример:
```yaml
panel_client_cert: |
  -----BEGIN CERTIFICATE-----
  MIIDxTCCAq2gAwIBAgIQAqxcJmoLQJuPC3nyrkYldzANBgkqhkiG9w0BAQUFADBs
  ...
  -----END CERTIFICATE-----
```

Скопируйте файл `group_vars/marzban_nodes.yml.example` и заполните его своими данными.

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

## TODO
- Добавлять агенты Dozzle и Netdata