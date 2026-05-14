# Ansible Role: vector

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Vector Version](https://img.shields.io/badge/vector-latest-blue.svg)](https://vector.dev)
[![Ansible Galaxy](https://img.shields.io/badge/galaxy-vector-blue.svg)](https://galaxy.ansible.com)

> 🚀 Установка и настройка Vector log agent с поддержкой ClickHouse sink, systemd-юнитами и production-ready конфигурацией.

## 📋 Оглавление

- [Ansible Role: vector](#ansible-role-vector)
  - [📋 Оглавление](#-оглавление)
  - [📋 Описание](#-описание)
  - [⚙️ Требования](#️-требования)
  - [📦 Зависимости](#-зависимости)
  - [🔧 Переменные роли](#-переменные-роли)
    - [Основные параметры установки](#основные-параметры-установки)
    - [Параметры подключения к ClickHouse](#параметры-подключения-к-clickhouse)
    - [Параметры оптимизации отправки](#параметры-оптимизации-отправки)
    - [Дополнительные параметры](#дополнительные-параметры)
    - [Пример настройки переменных](#пример-настройки-переменных)
  - [🗂️ Структура шаблонов](#️-структура-шаблонов)
  - [🚀 Быстрый старт](#-быстрый-старт)
  - [🔍 Отладка и диагностика](#-отладка-и-диагностика)
  - [🤝 Handlers](#-handlers)
  - [🔐 Безопасность](#-безопасность)

## 📋 Описание

Роль `vector` автоматизирует развёртывание [Vector](https://vector.dev) — высокопроизводительного агента для сбора, трансформации и отправки логов. Роль выполняет:

- ✅ Создание системного пользователя и группы `vector` с безопасными настройками
- ✅ Скачивание и валидация бинарного файла Vector (проверка размера и типа архива)
- ✅ Распаковка в настраиваемую директорию установки
- ✅ Генерация конфигурации `vector.toml` из Jinja2-шаблона
- ✅ Развёртывание hardened systemd-юнита с security-ограничениями
- ✅ Валидация конфигурации перед запуском (`vector --validate`)
- ✅ Управление сервисом через systemd с graceful restart

## ⚙️ Требования

- Ansible >= 2.9
- Python 3.6+
- Целевая ОС: Linux (Ubuntu 20.04/22.04, Debian 10/11, CentOS 7/8, RHEL 8+)
- Доступ к интернету для загрузки Vector (или предварительно подготовленный локальный бинарник)
- `curl`, `tar`, `file` — утилиты для распаковки (устанавливаются автоматически)

## 📦 Зависимости

Роль автономна и не требует внешних зависимостей. Все необходимые пакеты устанавливаются через `vector_dependencies`.

## 🔧 Переменные роли

### Основные параметры установки

| Переменная | Значение по умолчанию | Описание |
|------------|----------------------|----------|
| `vector_dependencies` | `['curl', 'tar', 'file']` | Пакеты, необходимые для загрузки и распаковки Vector |
| `vector_version` | `0.34.0` | Версия Vector для установки (используется в `vector_url`) |
| `vector_arch` | `"{{ ansible_architecture \| map_arch }}"` | Архитектура целевой системы (автоматическое определение) |
| `vector_os` | `"{{ ansible_system \| lower }}"` | ОС целевой системы (linux, windows и т.д.) |
| `vector_filename` | `"vector-{{ vector_version }}-{{ vector_arch }}-{{ vector_os }}.tar.gz"` | Имя архива для загрузки |
| `vector_url` | `"https://packages.vector.dev/vector-{{ vector_version }}-{{ vector_arch }}-{{ vector_os }}.tar.gz"` | URL для загрузки бинарника |
| `vector_install_dir` | `/opt/vector` | Директория для установки бинарных файлов |
| `vector_config_dir` | `/etc/vector` | Директория для конфигурационных файлов |
| `vector_log_dir` | `/var/log/vector` | Директория для логов агента |
| `vector_config_file` | `"{{ vector_config_dir }}/vector.toml"` | Полный путь к основному файлу конфигурации |

### Параметры подключения к ClickHouse

| Переменная | Значение по умолчанию | Описание |
|------------|----------------------|----------|
| `vector_clickhouse_endpoint` | `http://localhost:8123` | Endpoint ClickHouse HTTP-интерфейса |
| `vector_clickhouse_database` | `logs` | Имя базы данных для записи логов |
| `vector_clickhouse_table` | `events` | Имя таблицы для вставки данных |
| `vector_clickhouse_user` | `default` | Пользователь для аутентификации в ClickHouse |
| `vector_clickhouse_password` | `""` | Пароль пользователя (рекомендуется использовать Ansible Vault) |

### Параметры оптимизации отправки

| Переменная | Значение по умолчанию | Описание |
|------------|----------------------|----------|
| `vector_batch_max_events` | `1000` | Максимальное количество событий в одном батче отправки |
| `vector_batch_timeout_secs` | `5` | Таймаут отправки батча в секундах (если не набран `max_events`) |

### Дополнительные параметры

| Переменная | Значение по умолчанию | Описание |
|------------|----------------------|----------|
| `vector_output_path` | `"{{ vector_log_dir }}/debug.json"` | Путь для отладочного file-sink (опционально) |
| `vector_service_enabled` | `true` | Включить автозапуск сервиса при загрузке системы |

### Пример настройки переменных

```yaml
# group_vars/log_agents.yml
vector_version: 0.35.0
vector_install_dir: /usr/local/vector
vector_config_dir: /etc/vector
vector_log_dir: /var/log/vector

# ClickHouse подключение
vector_clickhouse_endpoint: https://ch-cluster.prod:8443
vector_clickhouse_database: production_logs
vector_clickhouse_table: app_events
vector_clickhouse_user: vector_writer
vector_clickhouse_password: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          303662643...

# Оптимизация батчинга
vector_batch_max_events: 2000
vector_batch_timeout_secs: 3
```

## 🗂️ Структура шаблонов

Роль использует следующие Jinja2-шаблоны:

```bash
templates/
    ├── vector.toml.j2        # Основная конфигурация Vector (TOML)
    └── vector.service.j2     # Systemd unit file с security hardening
```
- `Особенности vector.toml.j2:`
    - 📥 Источник stdin с JSON-декодированием (готов к расширению)
    - 🔄 Поддержка трансформаций через remap (закомментировано по умолчанию)
    - 📤 ClickHouse sink с настраиваемым маппингом колонок
    - 🔐 Аутентификация через переменные окружения (безопасное хранение паролей)
    - 📦 Батчинг для оптимизации сетевых запросов

- `Особенности vector.service.j2:`
    - 🔐 Security hardening: NoNewPrivileges=true, ProtectSystem=strict, ProtectHome=true
    - 📁 Ограничение прав записи: ReadWritePaths только для необходимых директорий
    - ♻️ Автоматический рестарт при сбоях с экспоненциальной задержкой
    - 🔑 Передача секретов через Environment (совместимо с Ansible Vault)

## 🚀 Быстрый старт

1. Установка роли

```bash
# Через Ansible Galaxy (после публикации)
ansible-galaxy install your_namespace.vector_role

# Или из локального репозитория
git clone https://github.com/xo4ychill/vector-role.git roles/vector-role
```

2. Подготовка секретов (рекомендуется)

```bash
# Шифрование пароля ClickHouse
ansible-vault encrypt_string 'my_secure_password' --name 'vector_clickhouse_password'
```

3. Создание playbook

```yaml
# deploy_vector.yml
---
- name: Deploy Vector log agent
  hosts: log_agents
  become: true
  
  vars_files:
    - vault.yml  # Файл с зашифрованными переменными
  
  roles:
    - role: vector
      tags: ['vector', 'logging']
```

4. Запуск

```bash
# Обычный запуск с запросом пароля vault
ansible-playbook -i inventory.ini deploy_vector.yml --ask-vault-pass

# Dry-run (проверка без изменений)
ansible-playbook -i inventory.ini deploy_vector.yml --check --ask-vault-pass

# Только валидация конфигурации
ansible-playbook -i inventory.ini deploy_vector.yml --tags 'validate' --ask-vault-pass
```

## 🔍 Отладка и диагностика
Роль включает многоуровневую валидацию и отладку:
- Валидация загрузки:
    - ✅ Проверка размера файла (>1 МБ) для исключения загрузки HTML-страниц ошибок
    - ✅ Проверка типа файла через утилиту file (должен быть gzip/xz/tar архив)
- Валидация конфигурации:
    - ✅ Запуск vector --validate --config перед применением изменений
    - ✅ Вывод подробных ошибок валидации в лог Ansible
- Диагностика запуска сервиса:
    - ✅ Проверка наличия systemd unit file
    - ✅ Вывод логов journalctl -u vector при неудачном старте
    - ✅ Проверка доступности бинарника перед рестартом

- Полезные команды:
```bash
# Проверка статуса сервиса на удалённом хосте
ansible log_agents -m systemd -a 'name=vector state=started' --become

# Просмотр логов Vector
ansible log_agents -m command -a 'journalctl -u vector -n 50 --no-pager' --become

# Тест конфигурации вручную
ansible log_agents -m command -a '/opt/vector/bin/vector --validate --config /etc/vector/vector.toml' --become
```

## 🤝 Handlers
Роль определяет следующие handlers:
| Handler          | Действие                                     | Когда вызывается                                           |
|------------------|----------------------------------------------|------------------------------------------------------------|
| `restart vector` | `systemctl restart vector` с `daemon_reload` | При изменении конфигурации или systemd unit                |
| `reload systemd` | `systemctl daemon-reload`                    | При изменении unit file без необходимости полного рестарта |

💡 Handlers выполняются атомарно в конце play. Валидация бинарника перед рестартом предотвращает падение сервиса при битых конфигурациях.

## 🔐 Безопасность
- Рекомендации по защите секретов:
  - Используйте Ansible Vault для хранения паролей:
    ```yaml
    vector_clickhouse_password: !vault |
    ANSIBLE_VAULT;1.1;AES256
    303662643...
    ```
  - Ограничьте права доступа к конфигурации
    ```yaml
    # В defaults/main.yml уже установлено:
    # mode: "0640" для vector.toml
    # owner/group: vector
    ```

  - Используйте переменные окружения для передачи секретов в процесс (реализовано в systemd unit).
  - Регулярно обновляйте версию Vector через vector_version для получения исправлений безопасности.
- Security hardening в systemd:
  - `NoNewPrivileges=true` — запрет повышения привилегий
  - `ProtectSystem=strict` — монтирование ФС только для чтения, кроме указанных путей
  - `ProtectHome=true` — запрет доступа к домашним директориям пользователей
  - `ReadWritePaths` — явное указание разрешённых путей для записи

