# Ansible-роль: Traefik + step-ca

Разворачивает на хосте с Docker reverse proxy **Traefik** и, опционально, локальный центр сертификации **step-ca**. Сертификаты выпускаются автоматически по ACME: либо у локального step-ca, либо у Let's Encrypt.

## Содержание

- [Ansible-роль: Traefik + step-ca](#ansible-роль-traefik--step-ca)
  - [Содержание](#содержание)
  - [Возможности](#возможности)
  - [Требования](#требования)
  - [Быстрый старт](#быстрый-старт)
    - [Вариант 1: локальный step-ca](#вариант-1-локальный-step-ca)
    - [Вариант 2: Let's Encrypt](#вариант-2-lets-encrypt)
    - [Playbook](#playbook)
    - [Хеш пароля для dashboard](#хеш-пароля-для-dashboard)
  - [Переменные](#переменные)
    - [Docker и сеть](#docker-и-сеть)
    - [Traefik](#traefik)
    - [Dashboard](#dashboard)
    - [step-ca](#step-ca)
    - [Let's Encrypt](#lets-encrypt)
  - [Как это работает](#как-это-работает)
    - [Зачем фиксированный IP и алиасы](#зачем-фиксированный-ip-и-алиасы)
    - [Порты и bind-адрес](#порты-и-bind-адрес)
  - [DNS при использовании step-ca](#dns-при-использовании-step-ca)
    - [Выбор домена](#выбор-домена)
    - [Вариант 1: docker-алиасы (мало сервисов)](#вариант-1-docker-алиасы-мало-сервисов)
    - [Вариант 2: внутренний DNS (рекомендуется)](#вариант-2-внутренний-dns-рекомендуется)
    - [Доступ клиентов к самому CA](#доступ-клиентов-к-самому-ca)
    - [Проверка DNS](#проверка-dns)
    - [Срок жизни сертификатов](#срок-жизни-сертификатов)
  - [Подключение сервисов](#подключение-сервисов)
  - [Доверие к корневому сертификату](#доверие-к-корневому-сертификату)
  - [Что создаётся на хосте](#что-создаётся-на-хосте)
  - [Проверка](#проверка)
  - [Тесты Molecule](#тесты-molecule)
  - [Ограничения](#ограничения)
  - [Ссылки](#ссылки)

## Возможности

- Traefik в Docker Compose с фиксированным IP в отдельной docker-сети.
- step-ca с включённым ACME-провижионером (`DOCKER_STEPCA_INIT_ACME`).
- Два certificate resolver в одном конфиге: `stepca` и `le` (Let's Encrypt, по умолчанию staging).
- Редирект HTTP → HTTPS, метрики Prometheus на `:8082`, опциональный dashboard с basic auth.
- Валидация переменных и compose-файлов (`docker compose config -q`) до запуска.
- Healthcheck у контейнеров, ожидание готовности step-ca перед запуском Traefik.
- Идемпотентность: `acme.json` создаётся только если его нет.

## Требования

- Ansible 2.14+ и коллекция `community.docker` (см. `requirements.yaml`):

  ```bash
  ansible-galaxy collection install -r requirements.yaml
  ```

- На целевом хосте установлены Docker Engine и плагин `docker compose` (роль их **не** ставит).
- Переменная `ansible_user` задана: от его имени создаются каталоги и файлы.
- Для step-ca нужен DNS, который резолвит имена ваших сервисов в IP Traefik (см. [Как это работает](#как-это-работает)).

## Быстрый старт

### Вариант 1: локальный step-ca

```yaml
# host_vars/srv-1.yaml
ansible_user: "vlad"

stepca_enabled: true
letsencrypt_enabled: false
stepca_url: "ca.int.example.com"

traefik_ip_address: "172.23.27.254"
traefik_enable_dashboard: true
traefik_dashboard_url: "traefik.int.example.com"
traefik_dashboard_user: "admin"
traefik_dashboard_password_hash: "{{ vault_traefik_dashboard_hash }}"  # обычный хеш $apr1$...
```

### Вариант 2: Let's Encrypt

```yaml
# host_vars/srv-1.yaml
ansible_user: "vlad"

stepca_enabled: false
letsencrypt_enabled: true
letsencrypt_staging_enabled: false   # true = тестовый CA без лимитов, но без доверенного сертификата
acme_email: "you@example.com"

traefik_enable_dashboard: true
traefik_dashboard_url: "traefik.example.com"
traefik_dashboard_user: "admin"
traefik_dashboard_password_hash: "{{ vault_traefik_dashboard_hash }}"
```

### Playbook

```yaml
# playbook.yaml
- name: Deploy Traefik + step-ca
  hosts: all
  become: true
  gather_facts: false
  roles:
    - traefik_stepca
```

```bash
ansible-playbook -i inventory.ini playbook.yaml
```

### Хеш пароля для dashboard

Передавайте хеш в обычном виде, экранирование `$` для compose роль делает сама:

```bash
openssl passwd -apr1 'your-password'
# или
htpasswd -nb admin 'your-password'
```

Храните собственный хеш в Ansible Vault и не используйте тестовые учётные данные в production. В defaults задан тестовый пользователь `admin` с паролем `admin`; при включённом dashboard пустой хеш остановит роль на валидации.

## Переменные

### Docker и сеть

| Переменная | По умолчанию | Описание |
|---|---|---|
| `docker_network_name` | `traefik` | Имя docker-сети |
| `docker_network_subnet` | `172.23.27.0/24` | Подсеть |
| `docker_network_gateway` | `172.23.27.1` | Шлюз |
| `restart_policy` | `unless-stopped` | Политика перезапуска контейнеров |

### Traefik

| Переменная | По умолчанию | Описание |
|---|---|---|
| `traefik_container_name` | `traefik` | Имя контейнера |
| `traefik_image` / `traefik_version` | `traefik` / `latest` | Образ и тег. Тег лучше зафиксировать (v2 и v3 несовместимы) |
| `traefik_ip_address` | `172.23.27.254` | Фиксированный IP в docker-сети |
| `traefik_bind_address` | `0.0.0.0` | Адрес хоста, на котором публикуются порты 80 и 443. IPv6 пишется в скобках: `[::]` |
| `traefik_log_level` | `INFO` | Уровень логов |
| `traefik_docker_dir` | `/home/<user>/traefik` | Каталог проекта |
| `traefik_config_dir` | `<traefik_docker_dir>/config` | Каталог статического конфига |
| `acme_config_dir` | `<traefik_docker_dir>/acme` | Каталог с хранилищами ACME |

### Dashboard

| Переменная | По умолчанию | Описание |
|---|---|---|
| `traefik_enable_dashboard` | `false` | Включить dashboard |
| `traefik_dashboard_url` | `traefik.home.local` | Имя хоста dashboard |
| `traefik_dashboard_user` | `admin` | Пользователь |
| `traefik_dashboard_password_hash` | тестовый хеш `admin:admin` | Хеш пароля; замените в production |

### step-ca

| Переменная | По умолчанию | Описание |
|---|---|---|
| `stepca_enabled` | `false` | Развернуть step-ca и включить resolver `stepca` |
| `stepca_url` | `ca.home.local` | DNS-имя CA (алиас в docker-сети и SAN сертификата) |
| `stepca_init_name` | `local-ca` | Имя CA при инициализации |
| `stepca_image` / `stepca_version` | `smallstep/step-ca` / `latest` | Образ и тег |
| `stepca_container_name` | `stepca` | Имя контейнера |
| `stepca_volume_name` | `step-ca-data` | Volume с ключами и конфигом CA |
| `stepca_docker_dir` | `/home/<user>/stepca` | Каталог compose-проекта |
| `stepca_certificates_duration_hours` | `2160` | Срок жизни сертификата (90 дней) для расчёта продления в Traefik |
| `caserver_url` | `https://<stepca_url>:9000/acme/acme/directory` | ACME-endpoint step-ca |

### Let's Encrypt

| Переменная | По умолчанию | Описание |
|---|---|---|
| `letsencrypt_enabled` | `false` | Включить resolver `le` |
| `letsencrypt_staging_enabled` | `true` | Использовать staging-CA |
| `acme_email` | _пусто_ | E-mail для ACME (обязателен при включённом LE) |
| `letsencrypt_url` / `letsencrypt_staging_url` | адреса Let's Encrypt | Production и staging directory |
| `acme_storage_le` / `acme_storage_stepca` | `acme/acme-le.json` / `acme/acme-stepca.json` | Файлы хранилищ |

## Как это работает

Порядок действий роли:

1. Проверяет переменные и наличие Docker CLI (`validation.yaml`).
2. Создаёт docker-сеть с заданными subnet и gateway.
3. Если `stepca_enabled`: создаёт каталог, рендерит compose и запускает step-ca, затем ждёт статуса `healthy` (до 15 попыток по 5 секунд).
4. Создаёт каталоги Traefik и пустые хранилища ACME с правами `0600`.
5. Рендерит `traefik.yml` и compose, проверяет compose через `docker compose config -q`.
6. Запускает Traefik. При изменении любого из файлов контейнер пересоздаётся принудительно, чтобы подхватить новый статический конфиг.

### Зачем фиксированный IP и алиасы

Для HTTP-01 step-ca сам подключается к домену, для которого выпускается сертификат. Поэтому из контейнера step-ca имя сервиса должно резолвиться в Traefik. Роль даёт Traefik постоянный IP и добавляет сетевые алиасы для `stepca_url` (у step-ca) и `traefik_dashboard_url` (у Traefik).

Для остальных сервисов алиасы по одному не масштабируются. Правильное решение: внутренний DNS (например, dnsmasq с зоной вида `*.int.example.com`, указывающей на Traefik) и `dns:` на этот резолвер у контейнера step-ca.

### Порты и bind-адрес

`traefik_bind_address` влияет только на публикацию портов 80/443 на хосте. Если step-ca резолвит имена сервисов через docker-алиасы, HTTP-01 идёт внутри docker-сети напрямую в контейнер Traefik, и привязка к конкретному адресу выпуск не ломает. Если же step-ca резолвит имена через внешний DNS в IP хоста (см. ниже), он ходит на опубликованный порт 80, и `127.0.0.1` в качестве bind-адреса работать не будет.

## DNS при использовании step-ca

При ACME-выпуске участвуют два независимых направления, и каждому нужно своё разрешение имён:

| Кто | Куда ходит | Что должно резолвиться |
|---|---|---|
| Клиенты (браузер, curl) | на сервисы | `app.int.example.com` → **IP хоста** (LAN-адрес), порт 443 |
| step-ca (контейнер) | на домен, для которого выпускается сертификат (проверка HTTP-01) | то же имя → Traefik: через docker-алиас (IP `traefik_ip_address`) или через внешний DNS (IP хоста, порт 80) |
| Traefik (контейнер) | на ACME-endpoint step-ca | `stepca_url` → контейнер step-ca (обеспечивает алиас в docker-сети) |

Если имя не резолвится у step-ca или порт 80 недоступен, выпуск завершится ошибкой проверки. Причину смотрите в логах Traefik и step-ca.

### Выбор домена

- Не используйте `.local`: зона зарезервирована под mDNS, на многих системах (systemd-resolved, macOS) запросы уйдут мимо вашего DNS.
- Используйте поддомен домена, которым вы владеете (например, `int.example.com`), либо специальную зону `home.arpa`.
- Для каждого имени нужен отдельный сертификат: HTTP-01 не поддерживает wildcard. Wildcard-сертификаты потребуют DNS-01 и в этой роли не реализованы.

### Вариант 1: docker-алиасы (мало сервисов)

Роль сама добавляет алиасы для `stepca_url` и `traefik_dashboard_url`. Для каждого нового сервиса алиас придётся добавлять вручную, поэтому вариант подходит только для нескольких имён. Клиентам имена всё равно нужно прописать в DNS или `/etc/hosts` на IP хоста.

### Вариант 2: внутренний DNS (рекомендуется)

Поднимите DNS-сервер (dnsmasq, CoreDNS, Pi-hole, роутер) и направьте всю зону на хост с Traefik:

```ini
# dnsmasq: все имена зоны, включая ca.int.example.com, ведут на хост
address=/int.example.com/192.168.1.10
```

Затем:

1. Раздайте этот DNS клиентам (DHCP или настройки сетевого интерфейса).
2. Контейнеру step-ca задайте этот DNS-сервер, иначе он использует встроенный docker-резолвер и не увидит зону:

   ```yaml
   # docker-compose-stepca.yaml.j2
   services:
     step-ca:
       dns:
         - 192.168.1.2   # адрес вашего внутреннего DNS
   ```

   Роль DNS-настройки step-ca не делает, это нужно добавить в шаблон вручную.
3. Убедитесь, что `traefik_bind_address` доступен из docker-сети (не `127.0.0.1`), а файрвол хоста пропускает порт 80 из подсети `docker_network_subnet`.

С такой схемой алиасы в docker-сети не нужны, а новые сервисы получают имена без изменений в роли.

### Доступ клиентов к самому CA

Порт 9000 step-ca роль на хост **не публикует**: CA доступен только внутри docker-сети. Если клиентам нужен `step ca bootstrap`, `certbot` или `acme.sh` напрямую к вашему CA, опубликуйте порт и заведите DNS-запись для `stepca_url` на IP хоста. Для работы Traefik это не требуется.

### Проверка DNS

```bash
# с клиента: имя ведёт на хост
dig +short app.int.example.com

# изнутри сети step-ca: то же имя резолвится
docker exec stepca getent hosts app.int.example.com

# и порт 80 хоста достижим из контейнера
docker exec stepca wget -qO- --timeout=3 http://app.int.example.com/ -S 2>&1 | head -n 3
```

### Срок жизни сертификатов

Параметр `certificatesDuration` в Traefik только определяет, когда продлевать сертификат. Фактический срок задаёт step-ca, и по умолчанию это **24 часа**. Чтобы схема работала, в claims ACME-провижионера step-ca нужно выставить `defaultTLSCertDuration` и `maxTLSCertDuration` равными `2160h`. Иначе Traefik будет считать выданный сертификат почти истёкшим и перевыпускать его постоянно. Команда (флаги проверьте через `--help` для своей версии):

```bash
docker exec -it stepca step ca provisioner update acme \
  --x509-default-dur 2160h --x509-max-dur 2160h
```

При включённом `REMOTE_MANAGEMENT` провижионеры хранятся в БД, и команде нужна admin-аутентификация.

## Подключение сервисов

Добавьте контейнеру сервиса лейблы и подключите его к сети роли:

```yaml
services:
  app:
    image: example/app
    networks: [traefik]
    labels:
      - traefik.enable=true
      - traefik.http.routers.app.rule=Host(`app.int.example.com`)
      - traefik.http.routers.app.entrypoints=websecure
      - traefik.http.routers.app.tls.certresolver=stepca   # или le
      - traefik.http.services.app.loadbalancer.server.port=8080

networks:
  traefik:
    external: true
```

`exposedByDefault: false`: сервис без `traefik.enable=true` Traefik не видит.

## Доверие к корневому сертификату

**Traefik → step-ca.** Traefik ходит в ACME-endpoint по HTTPS и должен доверять корню CA. Роль копирует публичный `root_ca.crt` из контейнера step-ca в `traefik_config_dir` и указывает его в `caCertificates`. Весь volume step-ca в Traefik **не монтируется**: в нём лежат приватные ключи CA.

**Клиенты → сервисы.** Браузеры и системы не будут доверять выпущенным сертификатам, пока вы не установите корень. Файл лежит на хосте: `{{ traefik_config_dir }}/root_ca.crt`. Отпечаток для `step ca bootstrap`:

```bash
docker exec stepca step certificate fingerprint /home/step/certs/root_ca.crt
```

## Что создаётся на хосте

```text
{{ traefik_docker_dir }}/
├── docker-compose.yaml
├── config/
│   ├── traefik.yml
│   └── root_ca.crt          # если включён step-ca
└── acme/
    ├── acme-le.json         # если включён Let's Encrypt
    └── acme-stepca.json     # если включён step-ca

{{ stepca_docker_dir }}/
└── docker-compose.yaml      # если включён step-ca
```

Данные step-ca (ключи, `ca.json`, БД) хранятся в docker volume `step-ca-data`.

## Проверка

```bash
docker ps
docker inspect --format '{{.State.Health.Status}}' stepca traefik
docker compose -f /home/vlad/traefik/docker-compose.yaml config -q

# сертификат dashboard
curl --cacert /home/vlad/traefik/config/root_ca.crt https://traefik.int.example.com
```

## Тесты Molecule

Оба сценария создают две Docker-машины: Ubuntu 22.04 и Rocky Linux 9. В каждой запускается systemd, устанавливаются Docker Engine и Compose plugin, после чего роль разворачивается внутри гостевой системы. Для Docker-in-Docker контейнеры тестовых ОС запускаются в `privileged` режиме; используйте эти сценарии только на доверенном тестовом Docker-хосте.

На хосте нужны Docker Engine, Ansible, коллекция `community.docker` и Molecule с Docker-драйвером:

```bash
python -m pip install molecule "molecule-plugins[docker]"
ansible-galaxy collection install -r requirements.yaml
molecule test -s stepca
```

Сценарий `stepca` проверяет на обеих ОС статусы healthcheck контейнеров step-ca и Traefik, выпуск сертификата для dashboard и цепочку до корневого сертификата step-ca.

Сценарий `letsencrypt-staging` разворачивает роль на обеих ОС, проверяет healthcheck Traefik и staging ACME endpoint. Он **не** запрашивает сертификат: для реального Let's Encrypt staging-выпуска нужны публичный домен и доступный извне HTTP-01 порт 80. Это позволяет запускать тест без DNS-настройки и без открытия портов тестовых контейнеров наружу.

Оба сценария удаляют созданные гостевые контейнеры и тестовые данные при завершении.

## Ограничения

- Одновременно можно включить **только один** из `stepca_enabled` и `letsencrypt_enabled`: лейблы dashboard выбирают один resolver.
- Если оба режима выключены, dashboard работает без TLS на `web`, а редирект HTTP → HTTPS включён всегда, поэтому в этом режиме dashboard недоступен. Включайте один из resolver.
- Переменные `DOCKER_STEPCA_INIT_*` применяются только при инициализации пустого volume. Смена `stepca_url` или `stepca_init_name` на уже работающем CA ничего не изменит. Для этого нужно пересоздать volume (все выданные сертификаты и корень будут утеряны) или править `ca.json` вручную.
- Роль не устанавливает Docker и не управляет DNS.
- Docker socket монтируется в Traefik. Для более строгой изоляции используйте socket-proxy.

## Ссылки

- Traefik: <https://doc.traefik.io/traefik/>
- step-ca: <https://smallstep.com/docs/step-ca/>