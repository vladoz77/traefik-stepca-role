# Роль Traefik + step-ca

Эта роль разворачивает в Docker:

- Traefik как reverse proxy
- optional `step-ca` для локального ACME/PKI
- optional Let's Encrypt
- сетевой стек с фиксированным IP
- конфигурацию Traefik через шаблон
- валидатор docker compose перед запуском
- healthcheck для контейнеров

## Что делает роль

Роль выполняет следующие действия:

1. Создаёт Docker network с заданным subnet/gateway.
2. Создаёт директории:
   - `{{ traefik_docker_dir }}`
   - `{{ traefik_config_dir }}`
   - `{{ acme_config_dir }}`
3. Создаёт файлы `acme.json` для Let's Encrypt и для step-ca, если соответствующие режимы включены.
4. Генерирует конфигурацию Traefik:
   - `traefik.yml`
5. Генерирует `docker-compose.yaml`.
6. Проверяет корректность compose через:
   - `docker compose -f ... config -q`
7. Запускает стек через `docker compose`.

## Пример host_vars

```yaml
ansible_user: "vlad"

docker_network_name: "traefik"
docker_network_subnet: "172.23.27.0/24"
docker_network_gateway: "172.23.27.1"

stepca_enabled: true
stepca_url: "ca.home.local"

traefik_ip_address: "172.23.27.254"
traefik_enable_dashboard: true
traefik_dashboard_url: "traefik.home.local"
traefik_dashboard_user: "admin"
traefik_dashboard_password_hash: "$${apr1}$$CmiVRFjs$$bPQcSgQ.2HcTzM.HVTvAl1"

letsencrypt_enabled: false
acme_email: "your-email@example.com"
```

## Пример использования

### Вариант 1: локальный `step-ca` + dashboard

```yaml
# host_vars/srv-1.yaml
ansible_user: "vlad"

stepca_enabled: true
stepca_url: "ca.home.local"

letsencrypt_enabled: false

traefik_ip_address: "172.23.27.254"
traefik_enable_dashboard: true
traefik_dashboard_url: "traefik.home.local"
traefik_dashboard_user: "admin"
traefik_dashboard_password_hash: "$${apr1}$$CmiVRFjs$$bPQcSgQ.2HcTzM.HVTvAl1"
```

```yaml
# playbook.yaml
- name: Deploy Traefik + step-ca
  hosts: all
  become: true
  gather_facts: false
  roles:
    - traefik
```

После запуска роль создаст:

- сеть `traefik` с IP `172.23.27.254`
- контейнер `step-ca`
- контейнер `traefik`
- dashboard на `https://traefik.home.local`
- TLS через локальный Step CA

### Вариант 2: Let's Encrypt + dashboard

```yaml
# host_vars/srv-1.yaml
ansible_user: "vlad"

stepca_enabled: false
letsencrypt_enabled: true
acme_email: "your-email@example.com"

traefik_ip_address: "172.23.27.254"
traefik_enable_dashboard: true
traefik_dashboard_url: "traefik.home.local"
traefik_dashboard_user: "admin"
traefik_dashboard_password_hash: "$${apr1}$$CmiVRFjs$$bPQcSgQ.2HcTzM.HVTvAl1"
```

### Проверка после установки

```bash
docker ps

docker compose -f /home/vlad/traefik/docker-compose.yaml config
```

## Запуск роли

```bash
ansible-playbook -i inventory.ini playbook.yaml
```

## Что создаётся на хосте

После применения роли в каталоге `{{ traefik_docker_dir }}` появляются:

- `docker-compose.yaml`
- `config/traefik.yml`
- `acme/acme-le.json` (если включён Let's Encrypt)
- `acme/acme-stepca.json` (если включён step-ca)

## Структура конфигурации Traefik

В шаблоне `traefik.yml.j2` настроены:

- HTTP -> HTTPS redirect
- entrypoints `web` и `websecure`
- metrics endpoint на `:8082`
- Docker provider
- certificates resolvers для:
  - `stepca`
  - `le`

## Ограничения и замечания

- Для Let's Encrypt обязательно указать `acme_email` при `letsencrypt_enabled=true`.
- Для dashboard обязательно указать `traefik_dashboard_password_hash`.
- `acme.json` создаётся только если файл отсутствует, чтобы не происходило постоянного изменения на каждом прогоне Ansible.
- Compose валидируется перед запуском через `docker compose config`.

## Полезные ссылки

- Traefik: https://doc.traefik.io/traefik/
- Smallstep CA: https://smallstep.com/docs/step-ca/
