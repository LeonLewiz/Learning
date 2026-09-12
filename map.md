## 1. Репозитории
ansible-lab - репозиторий с плейбуками для ансибла.
my-nginx-site - репозиторий с самим сайтом, Dockerfile.

## 2. Точка входа
Сначала руками подготовить control node: настроить доступ Ansible к Managed-nodes, вписать имена в hosts.ini, предоставить беспарольный доступ через SSH и IP-адреса. Потом `ansible-playbook -i hosts.ini install_docker.yml`. Потом можно запускать все остальные плейбуки.

## 3. Порядок
После установки основного сайта можно запускать любой другой плейбук. Каждый отвечает за установку своей службы.

## 4. Порты, расположение служб, IP-адреса, имена машин, OC.
backend - 8081:8000 - Managed
website - 9090:80 - Managed
cAdvisor - 8080:8080 - Managed
Blackbox - 9115 - Control
Prometheus - 9090 - Control
NodeExporter - 9100 - All
Grafana - 3000 - Control
Alertmanager - 9093 - Control

3 виртуальные машины:
Control - 192.168.13.130 - Ubuntu Server LTS 26.04
Managed 1 - 192.168.13.133 - Ubuntu Server LTS 26.04.01
Managed 2 - 192.168.13.134 - Debian Minimal 12


## 5. Секреты
Все секреты (DB_PASSWORD, POSTGRES_PASSWORD) зашифрованы через Ansible-Vault моим личным паролем (.vault_pass) и лежат по пути `group_vars/all/vault.yml`. Файл ~/.vault_pass с паролем должен обязательно быть при запуске install_docker.yml, иначе не сработает.
# vars.yml
```yml
project_dir: /var/www/html/my-nginx-site
db_host: db
db_name: site_analytics
db_user: myuser
web_port: 9090
backend_port: 8081
```
# .env.j2
```yml
# db container
POSTGRES_USER={{ db_user }}
POSTGRES_PASSWORD={{ POSTGRES_PASSWORD }}
POSTGRES_DB={{ db_name }}

# backend
DB_HOST={{ db_host }}
DB_USER={{ db_user }}
DB_PASSWORD={{ DB_PASSWORD }}
DB_NAME={{ db_name }}
```
## 6. CI
CI был настроен через Github Actions и self-hosted runner. Подключены оба репозитория - ansible-lab и my-nginx-site. При Коммите в ansible-lab в этот репозиторий самостоятельно загружаются новые файлы. При Коммите в my-nginx-site CI билдит из новых файлов образ и отправляет в Github Packages и пересоздает контейнеры на Managed-nodes.

## 7. Dashboards
Все дэшборды хранятся в `ansible-lab/files/` в режиме provisioned. В случае чего переживут удаление.
## 8. Не знаю
список всего, что не понял
Плохо разбираюсь с докером.
  -Healthchecks
  -Multi-stage
  -Basic Dockerfile structure
Проблемы с базовыми командами и ключами Linux. Нужно заучить.
