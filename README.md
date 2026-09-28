# Домашнее задание к занятию 3 «Использование Ansible»

## Описание

Playbook `site.yml` разворачивает два сервиса на двух хостах в Yandex Cloud:

| Play | Хост | Что делает |
|---|---|---|
| Install ClickHouse | clickhouse-01 | Устанавливает ClickHouse из официального репозитория `packages.clickhouse.com`, запускает и включает сервис `clickhouse-server` |
| Install and configure LightHouse | lighthouse-01 | Устанавливает Nginx и unzip, скачивает статику LightHouse с GitHub, распаковывает, настраивает Nginx через шаблон Jinja2, включает сайт и запускает веб-сервер |

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

![ansible-lint](screenshots/lint.png)

### Первый прогон `--diff`

![Первый прогон](screenshots/first-run.png)

### Второй прогон `--diff` (идемпотентность)

![Второй прогон](screenshots/second-run.png)

### LightHouse в браузере

![LightHouse](screenshots/lighthouse.png)

URL: http://89.169.154.43/

## Окружение

- **Control node:** macOS (Apple M1), Ansible 2.21.1, ansible-lint 26.9.0
- **Managed nodes:** две ВМ в Yandex Cloud, Ubuntu 22.04 LTS
- **Сеть:** VPC `ansible-net`, подсеть `ansible-subnet-a` (192.168.10.0/24), NAT-шлюз `ansible-nat`, security group `ansible-sg` с ingress (22, 80, 8123, 9000) и egress (TCP 0-65535)

## Хосты

| Хост | Роль | Внешний IP |
|---|---|---|
| clickhouse-01 | ClickHouse | 93.77.188.64 |
| lighthouse-01 | LightHouse + Nginx | 89.169.154.43 |
