# 🗺️ map.md — карта проекта

## 1. Репозитории

| Репо | Что внутри |
|---|---|
| `ansible-lab` | плейбуки, `hosts.ini`, `group_vars/`, шаблоны (`.j2`), дэшборды Grafana в `files/` |
| `my-nginx-site` | сайт (`index.html`), бэкенд (`main.py`), `Dockerfile`, `docker-compose.yml` |

---

## 2. Инфраструктура

| | Control | Managed 1 | Managed 2 |
|---|---|---|---|
| Hostname | `devops-study` | `vm2-ubuntu` | `vm3-debian` |
| IP | `192.168.13.130` | `192.168.13.133` | `192.168.13.134` |
| ОС | Ubuntu Server LTS 26.04 | Ubuntu Server LTS 26.04.01 | Debian Minimal 12 |
| Роль | Ansible + мониторинг | сайт + метрики | сайт + метрики |

- Ansible-группы в `hosts.ini`: `ubuntu_servers`, `debian_servers`, `monitoring` (control)
- Доступ: SSH по ключу, `become` через `NOPASSWD` sudoers

---

## 3. Что где запущено

### Control node
| Сервис | Порт | Как |
|---|---|---|
| Prometheus | 9090 | systemd, тарболл в `/usr/local/bin/prometheus` |
| Grafana | 3000 | systemd, тарболл в `/opt/grafana/` |
| Alertmanager | 9093 | systemd, тарболл в `/usr/local/bin/alertmanager/` |
| Node Exporter | 9100 | systemd |
| Blackbox | 9115 | systemd |

### Managed (обе)
| Сервис | Порт | Как |
|---|---|---|
| website | 9090:80 | Docker |
| backend | 8081:8000 | Docker |
| PostgreSQL | 5432 (внутр.) | Docker, volume `pgdata` |
| cAdvisor | 8080:8080 | Docker |
| Node Exporter | 9100 | systemd |

---

## 4. Точка входа и порядок

**Руками — только подготовка control node:**
1. SSH-доступ к managed (ключи, беспарольный sudo)
2. `hosts.ini` с именами/IP машин
3. `~/new_vault_pass` с паролем vault
4. `ansible.cfg` с указанием дефолтного инвентаря

**Дальше всё плейбуками:**

```bash
ansible-playbook install_docker.yml      # сайт (точка входа)
ansible-playbook install_prometheus.yml
ansible-playbook install_grafana.yml
ansible-playbook install_alertmanager.yml
ansible-playbook install_blackbox.yml
ansible-playbook install_node_exporter.yml
ansible-playbook install_cadvisor.yml
```

Порядок после `install_docker.yml` - любой. Каждый плейбук ставит свою службу и **идемпотентен**.

---

## 5. Секреты

- Все секреты (`DB_PASSWORD`, `POSTGRES_PASSWORD`, `telegram_bot_token`, `telegram_chat_id`) зашифрованы **Ansible Vault**
- Файл: `group_vars/all/vault.yml`
- Пароль: `~/new_vault_pass` (обязателен при запуске, иначе Ansible встанет)
- Публичные переменные - `group_vars/all/vars.yml`

```yaml
# vars.yml
project_dir: /var/www/html/my-nginx-site
db_host: db
db_name: site_analytics
db_user: myuser
web_port: 9090
backend_port: 8081
```

---

## 6. Мониторинг - полная цепочка

```
blackbox щупает сайт (module http_2xx)
      │  probe_success == 0
      ▼
Prometheus (job: blackbox → ?target=<сайт>)
      │  правило SiteDown, for: 5m (rules.yml)
      ▼
Alertmanager (localhost:9093)
      ▼
Telegram (send_resolved: true)
```

**Скрейп-джобы Prometheus:** `prometheus` (self), `node_exporter` (all), `cadvisor` (managed), `blackbox` (проверка сайтов).

**Важно:** `scrape_interval` в `prometheus.yml` **и** `timeInterval` в datasource Grafana = `15s`. Должны совпадать — иначе `$__rate_interval` ломается.

---

## 7. Grafana dashboards

- Лежат в `ansible-lab/files/`, отдаются **provisioned** через `provider.yml`
- `allowUiUpdates: false` → источник только файл
- Дэшборды ссылаются на datasource **по реальному uid** (плейсхолдеры `${DS_PROMETHEUS}` заменены)

---

## 8. Ещё не сделано (план)

| # | Задача | Зачем |
|---|---|---|
| 1 | **CI** (GitHub Actions + self-hosted runner) | автодеплой при коммите |
| 2 | **Bootstrap control node** (Vagrant) | убрать ручную подготовку |
| 3 | **Terraform** | IaC |

> CI раньше был, но при сносе системы не восстановлен. Планируется вернуть.

---

## 9. Слабое место - Docker

Надо доучить:
- `Dockerfile`: структура, `healthcheck`, multi-stage

**Linux-база:** базовые команды и права - подтягивать в процессе.
