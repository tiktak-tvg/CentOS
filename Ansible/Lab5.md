### Автоматизация Prometheus и Grafana (плейбук monitoring_server.yml)
Чтобы стенд разворачивался полностью с нуля одной командой, нам нужно автоматизировать установку Prometheus и Grafana.

Поскольку Prometheus и Grafana ставятся на управляющую машину (centpca), мы создадим отдельный плейбук monitoring_server.yml, который будет запускаться локально (hosts: localhost).

Вот полная инструкция, как это сделать:

Так как Prometheus и Grafana ставятся на управляющую машину (centpca), мы создадим отдельный плейбук, который будет запускаться локально (``hosts: localhost``).

#### 1.1. Создаем роль monitoring_server
```bash
mkdir -p /etc/ansible/roles/monitoring_server/tasks
```
#### 1.2. Записываем задачи роли
```bash
cat <<'EOF' > /etc/ansible/roles/monitoring_server/tasks/main.yml
---
# tasks file for monitoring_server (Prometheus + Grafana)
- name: Установка утилит wget и tar
  yum:
    name:
      - wget
      - tar
    state: present

# --- PROMETHEUS ---
- name: Создание системного пользователя prometheus
  user:
    name: prometheus
    shell: /bin/false
    create_home: no
    system: yes

- name: Создание директорий для Prometheus
  file:
    path: "{{ item }}"
    state: directory
    owner: prometheus
    group: prometheus
    mode: '0755'
  loop:
    - /etc/prometheus
    - /var/lib/prometheus

- name: Скачивание и распаковка Prometheus
  unarchive:
    src: "https://github.com/prometheus/prometheus/releases/download/v2.53.0/prometheus-2.53.0.linux-amd64.tar.gz"
    dest: /tmp
    remote_src: yes
    creates: /tmp/prometheus-2.53.0.linux-amd64

- name: Копирование бинарников и консолей Prometheus
  shell: |
    cp /tmp/prometheus-2.53.0.linux-amd64/prometheus /usr/local/bin/
    cp /tmp/prometheus-2.53.0.linux-amd64/promtool /usr/local/bin/
    cp -r /tmp/prometheus-2.53.0.linux-amd64/consoles /etc/prometheus/
    cp -r /tmp/prometheus-2.53.0.linux-amd64/console_libraries /etc/prometheus/
    chown prometheus:prometheus /usr/local/bin/prometheus /usr/local/bin/promtool
    chown -R prometheus:prometheus /etc/prometheus/consoles /etc/prometheus/console_libraries
  args:
    creates: /usr/local/bin/prometheus

- name: Настройка конфигурации prometheus.yml (порт 9091)
  copy:
    dest: /etc/prometheus/prometheus.yml
    owner: prometheus
    group: prometheus
    content: |
      global:
        scrape_interval: 15s
        evaluation_interval: 15s
      scrape_configs:
        - job_name: 'prometheus'
          static_configs:
            - targets: ['localhost:9091']
        - job_name: 'node_exporter'
          static_configs:
            - targets: ['10.1.100.12:9100', '10.1.100.14:9100']

- name: Создание systemd-юнита Prometheus
  copy:
    dest: /etc/systemd/system/prometheus.service
    content: |
      [Unit]
      Description=Prometheus
      Wants=network-online.target
      After=network-online.target
      [Service]
      User=prometheus
      Group=prometheus
      Type=simple
      ExecStart=/usr/local/bin/prometheus \
        --config.file=/etc/prometheus/prometheus.yml \
        --storage.tsdb.path=/var/lib/prometheus/ \
        --web.console.templates=/etc/prometheus/consoles \
        --web.console.libraries=/etc/prometheus/console_libraries \
        --web.listen-address=:9091
      [Install]
      WantedBy=multi-user.target

- name: Запуск и автозагрузка Prometheus
  systemd:
    name: prometheus
    daemon_reload: yes
    state: started
    enabled: yes

# --- GRAFANA ---
- name: Скачивание пакета Grafana
  get_url:
    url: https://dl.grafana.com/oss/release/grafana-10.4.2-1.x86_64.rpm
    dest: /tmp/grafana-10.4.2-1.x86_64.rpm
    mode: '0644'

- name: Установка Grafana
  yum:
    name: /tmp/grafana-10.4.2-1.x86_64.rpm
    state: present
    disable_gpg_check: yes

- name: Запуск и автозагрузка Grafana
  systemd:
    name: grafana-server
    state: started
    enabled: yes

# --- FIREWALLD ---
- name: Открытие портов 9091 (Prometheus) и 3000 (Grafana)
  firewalld:
    port: "{{ item }}"
    permanent: yes
    state: enabled
    immediate: yes
  loop:
    - 9091/tcp
    - 3000/tcp
...
EOF
```
#### 1.3. Создаем плейбук monitoring_server.yml
```bash
cat <<'EOF' > /etc/ansible/monitoring_server.yml
---
- name: Установка Prometheus и Grafana на управляющей машине
  hosts: localhost
  connection: local
  become: yes
  roles:
    - monitoring_server
...
EOF
```
#### 1.4. Как теперь разворачивать стенд с нуля
```bash
# 1. Ставим агенты и веб-кластер на ноды
ansible-playbook /etc/ansible/site.yml

# 2. Ставим сервер мониторинга на управляющую машину
ansible-playbook /etc/ansible/monitoring_server.yml
```
(Единственное, что останется сделать вручную — один раз зайти в Grafana, добавить Datasource Prometheus (URL: http://localhost:9091) и импортировать дашборд ID 11074. Данные Grafana хранит в своей базе SQLite, поэтому после импорта они не пропадут).

### ЧАСТЬ 2. ИТОГОВЫЙ ОТЧЁТ ПО СТЕНДУ
#### 2.1. Архитектура решения
``` text
                          [ Клиент ]
                              │
                              ▼
                 ┌─────────────────────────┐
                 │  VIP 10.1.100.100       │
                 │  (Keepalived)           │
                 └────────────┬────────────┘
                              │
              ┌───────────────┴───────────────┐
              │                               │
       [ MASTER centpc1 ]             [ BACKUP centpc3 ]
        (10.1.100.12)                   (10.1.100.14)
              │                               │
              ▼                               ▼
        ┌───────────┐                   ┌───────────┐
        │  HAProxy  │                   │  HAProxy  │
        │   :80     │                   │   :80     │
        └─────┬─────┘                   └─────┬─────┘
              │                               │
      ┌───────┴───────┐               ┌───────┴───────┐
      │               │               │               │
      ▼               ▼               ▼               ▼
 [ nginx:81 ]   [ nginx:81 ]    [ nginx:81 ]   [ nginx:81 ]
      │               │               │               │
      ▼               ▼               ▼               ▼
 [ docker:8080 ] [ docker:8080 ] [ docker:8080 ] [ docker:8080 ]
      │               │               │               │
      ▼               ▼               ▼               ▼
  [ Шаблон ]       [ Шаблон ]      [ Шаблон ]      [ Шаблон ]
  (centpc1)       (centpc3)       (centpc1)       (centpc3)

  ─────────────────────────────────────────────────────────

  [ Управляющая машина centpca ]
      │
      ├── Prometheus :9091  ──(scrape)──► .12:9100 (node_exporter)
      │                                   .14:9100 (node_exporter)
      │
      ├── Grafana :3000 ──(визуализация дашбордов)
      │
      ├── Ansible (управление конфигурацией)
      │
      └── Централизованный rsyslog (принимает логи от .14)
```
#### 2.2. Таблица компонентов
Слой	Технология	Порт	Где развернуто
Отказоустойчивость	Keepalived (VRRP)	112	.12 (MASTER), .14 (BACKUP)
Балансировка	HAProxy	80	.12, .14
Статистика HAProxy	HAProxy Stats	8404	.12, .14
Веб-сервер (системный)	Nginx	81	.12, .14
Веб-сервер (контейнер)	Nginx 1.21 (Docker)	8080	.12, .14
Мониторинг	Node Exporter	9100	.12, .14
Сбор метрик	Prometheus Server	9091	centpca
Визуализация	Grafana	3000	centpca
Централизованные логи	rsyslog	514	.12 (сервер), .14 (клиент)
Бэкапы	Bash + Cron	—	.12, .14
Управление	Ansible	—	centpca
#### 2.3. Как развернуть стенд с нуля (пошагово)
##### Шаг 1: Предварительные условия
Настроены SSH-ключи с centpca на .12 и .14.

Установлен Ansible (yum install -y ansible).

Создан инвентарь /etc/ansible/hosts.

Созданы все роли (webserver, backup, security, docker, deploy_app, loadbalancer, syslog, monitoring, monitoring_server).

Настроены site.yml и monitoring_server.yml.

##### Шаг 2: Запуск
```bash
# 1. Основной плейбук (веб-кластер, безопасность, Docker, HAProxy, Keepalived, rsyslog, Node Exporter)
ansible-playbook /etc/ansible/site.yml

# 2. Сервер мониторинга (Prometheus + Grafana)
ansible-playbook /etc/ansible/monitoring_server.yml
```
##### Шаг 3: Настройка Grafana (один раз)
Открыть http://10.1.100.10:3000 (admin/admin).

<img width="1055" height="976" alt="image" src="https://github.com/user-attachments/assets/bc3d7d76-3553-48ba-b38d-93a31581a8d9" />

Добавить Data Source: Prometheus, URL: http://localhost:9091.

Импортировать дашборд по ID: 11074 (Node Exporter Dashboard EN).

#### 2.4. Проверки (что должно быть выполнено)
Проверка балансировки и отказоустойчивости
```bash
# 1. Балансировка (должно чередоваться centpc1 / centpc3)
for i in {1..6}; do curl -s http://10.1.100.100 | grep -oP 'Имя хоста: <b>\K[^<]+'; done

# 2. HAProxy видит оба backend'а (должно быть UP)
ansible servers -m shell -a "echo 'show stat' | socat stdio /var/run/haproxy.sock | grep web_node"

# 3. VIP на MASTER (должен быть на .12)
ansible servers -m shell -a "ip addr show ens33 | grep 10.1.100.100 || echo NO_VIP"

# 4. Keepalived роли (MASTER на .12, BACKUP на .14)
ansible servers -m shell -a "journalctl -u keepalived --no-pager -n 20 | grep -E 'Entering (MASTER|BACKUP)'"
Проверка мониторинга
bash
# 1. Node Exporter активен на обеих нодах
ansible servers -m shell -a "systemctl is-active node_exporter"

# 2. Prometheus видит все три target'а (должно быть up)
curl -s http://10.1.100.10:9091/api/v1/targets | python -m json.tool | grep -E '"health"|"scrapeUrl"'
```
Проверка логов
```bash
# 1. Отправить тестовый лог с клиента
ansible 10.1.100.14 -m shell -a "logger 'FINAL-TEST-RSYSLOG'"

# 2. Найти его на сервере
sleep 2
ansible 10.1.100.12 -m shell -a "grep -r 'FINAL-TEST-RSYSLOG' /var/log/remote/"
Проверка бэкапов
bash
# 1. Задача есть в cron
ansible servers -m shell -a "crontab -l | grep backup"

# 2. Ручной запуск (создаст архив)
ansible servers -m shell -a "/usr/local/bin/backup_web.sh && echo OK"

# 3. Проверить, что архив создался
ansible servers -m shell -a "ls -lh /var/backups/web/"
```
#### 2.5. Итоговый результат
В результате выполнения работы был развернут отказоустойчивый веб-кластер с полным набором инфраструктурных сервисов:

Высокая доступность: Keepalived с VIP 10.1.100.100 (переезд при отказе за ~5 секунд).

Балансировка нагрузки: HAProxy (roundrobin) с healthcheck'ами, оба backend'а в статусе UP.

Контейнеризация: Docker + Nginx, шаблон Jinja2 монтируется в контейнер через volume.

Мониторинг: Prometheus + Grafana + Node Exporter (сбор метрик CPU/RAM/Disk/Network с обеих нод).

Централизованное логирование: rsyslog (клиент .14 → сервер .12, запись в /var/log/remote/).

Резервное копирование: ежедневный cron-скрипт (03:30) в /var/backups/web/.

Безопасность: SSH только по ключам, root запрещен.

<img width="747" height="903" alt="image" src="https://github.com/user-attachments/assets/973dd3ba-067c-4501-aa1b-196971fb1083" />
Инфраструктура как код (IaC): всё разворачивается одной командой ansible-playbook site.yml + monitoring_server.yml.
