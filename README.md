# Домашнее задание к занятию 3 «Использование Ansible»

## Описание

Playbook `site.yml` разворачивает два сервиса на двух хостах в Yandex Cloud:

| Play | Хост | Что делает |
|---|---|---|
| Install ClickHouse | clickhouse-01 | Устанавливает ClickHouse из официального репозитория `packages.clickhouse.com`, запускает и включает сервис `clickhouse-server` |
| Install and configure LightHouse | lighthouse-01 | Устанавливает Nginx и unzip, скачивает статику LightHouse с GitHub, распаковывает, настраивает Nginx через шаблон Jinja2, включает сайт и запускает веб-сервер |

## Ограничения по хосту Vector

По заданию требовалось развернуть три сервиса на трёх хостах: **ClickHouse**, **Vector**, **LightHouse**.

Два хоста из трёх отработали штатно:
- `clickhouse-01` — ClickHouse установлен и работает
- `lighthouse-01` — LightHouse развёрнут, Nginx настроен, подключение к ClickHouse проверено

На третьем хосте, `vector-01`, развернуть Vector **не удалось**. Причина — у виртуальной машины отсутствовал исходящий доступ в интернет:

- `ping 8.8.8.8` — 100% потерь;
- `apt-get update` — не мог достучаться ни до `mirror.yandex.ru`, ни до `security.ubuntu.com` (`Network is unreachable`);
- `curl https://packages.timber.io` — DNS-ошибка `Could not resolve host`.

Установить Vector ни через репозиторий Timber, ни через прямую загрузку бинарника невозможно — оба способа требуют выхода в интернет.

### Что было предпринято для решения

1. **Проверена security group** `ansible-sg` — добавлено правило egress `direction=egress, from-port=0, to-port=65535, protocol=tcp, v4-cidrs=0.0.0.0/0`. На `clickhouse-01` и `lighthouse-01` после этого интернет заработал, на `vector-01` — нет.

2. **Настроен NAT-шлюз** `ansible-nat` и таблица маршрутизации `ansible-nat-rt` с маршрутом `0.0.0.0/0 → NAT-шлюз`. Таблица привязана к подсети `ansible-subnet-a`.

3. **ВМ останавливалась и запускалась заново** — не помогло, IP менялся, но исходящий трафик так и не появился.

4. **ВМ удалялась и создавалась с нуля** с теми же параметрами (тот же образ, та же подсеть, та же security group) — проблема воспроизводится.

### Вывод

Проблема **инфраструктурная**, связана с нестабильностью NAT в Yandex Cloud для конкретной ВМ. Логика playbook при этом корректна — `clickhouse-01` и `lighthouse-01` разворачиваются без ошибок, идемпотентность подтверждена повторным прогоном (`changed=0`).

Play `Install Vector` был **исключён из playbook**, чтобы успешно завершить развёртывание остальных сервисов. Playbook с этим play можно посмотреть в истории коммитов — он был написан и отработан, но на завершающем этапе убран.

## Структура проекта

```
ansible-dz3/
├── ansible.cfg               # локальный конфиг Ansible
├── prod.yml                  # inventory: хосты в Yandex Cloud
├── site.yml                  # основной playbook
├── .ansible-lint             # конфиг ansible-lint (skip_list)
├── templates/
│   └── nginx_lighthouse.j2   # шаблон конфига Nginx
├── screenshots/
│   ├── lint.png              # вывод ansible-lint
│   ├── first-run.png         # PLAY RECAP первого прогона
│   ├── second-run.png        # PLAY RECAP второго прогона (идемпотентность)
│   └── lighthouse.png        # LightHouse в браузере
└── README.md
```

## Переменные

### Параметры LightHouse (в `site.yml`, play `Install and configure LightHouse`)

| Переменная | Значение по умолчанию | Описание |
|---|---|---|
| `lighthouse_download_url` | `https://github.com/VKCOM/lighthouse/archive/refs/heads/master.zip` | URL со статикой LightHouse |
| `lighthouse_archive` | `/tmp/lighthouse.zip` | Путь для скачивания архива |
| `lighthouse_dir` | `/var/www/lighthouse` | Директория для распаковки статики |
| `nginx_user` | `www-data` | Владелец файлов LightHouse |

### Параметры inventory (`prod.yml`)

| Переменная | Значение | Описание |
|---|---|---|
| `ansible_user` | `yc-user` | Пользователь для SSH на всех хостах |
| `ansible_ssh_private_key_file` | `~/.ssh/ansible-key` | Приватный SSH-ключ |
| `ansible_ssh_common_args` | `-o StrictHostKeyChecking=no` | Отключение проверки host key |

## Теги

Теги в playbook не используются. Playbook запускается целиком.

## Запуск

```bash
# Проверка синтаксиса и правил
ansible-lint site.yml

# Сухой прогон (без изменений на хостах)
ansible-playbook site.yml --check

# Реальный прогон с показом диффов изменений
ansible-playbook site.yml --diff

# Повторный прогон для проверки идемпотентности
ansible-playbook site.yml --diff
```

## Идемпотентность

**Первый прогон:** `changed > 0` на всех хостах — устанавливаются пакеты, создаются файлы, запускаются сервисы.

**Второй прогон:** `changed=0` на всех хостах — playbook идемпотентен, повторный запуск не вносит изменений.

`PLAY RECAP` второго прогона:

```
PLAY RECAP *********************************************************************
clickhouse-01              : ok=6    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
lighthouse-01              : ok=9    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

## Скриншоты

### ansible-lint

![ansible-lint](https://github.com/user-attachments/assets/2825bd12-a08b-4b72-8837-fe67a9ced537)

![ansible-lint](https://github.com/user-attachments/assets/06e211ac-4225-4e65-9f6e-a3b70d94c39f)

### Первый прогон `--diff`

![Первый прогон](https://github.com/user-attachments/assets/0405f71b-8449-4c89-8e4e-714b1d2814ab)

### Второй прогон `--diff` (идемпотентность)

![Второй прогон](https://github.com/user-attachments/assets/9ea233a5-f7cb-4b3c-bdcd-a352d3bf72b4)

### LightHouse в браузере

![LightHouse](https://github.com/user-attachments/assets/14da4aa1-e00d-4d1e-b981-21224c556757)

![LightHouse](https://github.com/user-attachments/assets/5817250c-db7f-4e0f-8b3a-afc29db53b99)

URL: http://89.169.154.43/

## Окружение

- **Control node:** macOS (Apple M1), Ansible 2.21.1, ansible-lint 26.9.0
- **Managed nodes:** две ВМ в Yandex Cloud, Ubuntu 22.04 LTS
- **Сеть:** VPC `ansible-net`, подсеть `ansible-subnet-a` (192.168.10.0/24), NAT-шлюз `ansible-nat`, security group `ansible-sg` с ingress (22, 80, 8123, 9000) и egress (TCP 0-65535)



## Хосты

| Хост | Роль | Внешний IP | Статус |
|---|---|---|---|
| clickhouse-01 | ClickHouse | 93.77.188.64 | работает |
| vector-01 | Vector | 51.250.10.224 | не развёрнут (нет NAT) |
| lighthouse-01 | LightHouse + Nginx | 89.169.154.43 | работает |
