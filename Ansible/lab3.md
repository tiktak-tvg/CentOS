#### Лабораторная работа по этой архитектуре.



#### Новая роль monitoring — установка Prometheus Node Exporter на обе ноды
##### Шаг 1. Создаем структуру директорий роли
Выполните команду на управляющей машине (centpca):

```bash
mkdir -p /etc/ansible/roles/monitoring/{tasks,handlers}
Шаг 2. Создаем файл обработчиков (Handlers)
```
Создаем файл handlers/main.yml, чтобы Ansible мог перезапускать сервис при изменении конфигурации:

```bash
cat <<'EOF' > /etc/ansible/roles/monitoring/handlers/main.yml
---
- name: restart node_exporter
  systemd:
    name: node_exporter
    state: restarted
    daemon_reload: yes
...
EOF
```
##### Шаг 3. Создаем основные задачи (Tasks)
Создаем файл tasks/main.yml со всеми тегами monitoring (это критически важно для работы команды --tags monitoring):
```bash
cat <<'EOF' > /etc/ansible/roles/monitoring/tasks/main.yml
---
# tasks file for monitoring (Prometheus Node Exporter)
- name: Создание системного пользователя node_exporter (без логина)
  user:
    name: node_exporter
    shell: /sbin/nologin
    create_home: no
    system: yes
  tags: monitoring

- name: Скачивание и распаковка Node Exporter (v1.8.2)
  unarchive:
    src: "https://github.com/prometheus/node_exporter/releases/download/v1.8.2/node_exporter-1.8.2.linux-amd64.tar.gz"
    dest: /tmp
    remote_src: yes
    creates: /tmp/node_exporter-1.8.2.linux-amd64/node_exporter
  tags: monitoring

- name: Копирование бинарника в /usr/local/bin
  copy:
    src: /tmp/node_exporter-1.8.2.linux-amd64/node_exporter
    dest: /usr/local/bin/node_exporter
    mode: '0755'
    owner: node_exporter
    group: node_exporter
    remote_src: yes
  tags: monitoring

- name: Создание systemd-юнита node_exporter
  copy:
    dest: /etc/systemd/system/node_exporter.service
    content: |
      [Unit]
      Description=Prometheus Node Exporter
      Documentation=https://github.com/prometheus/node_exporter
      After=network.target

      [Service]
      User=node_exporter
      Group=node_exporter
      Type=simple
      ExecStart=/usr/local/bin/node_exporter
      Restart=on-failure
      RestartSec=5

      [Install]
      WantedBy=multi-user.target
    mode: '0644'
  notify: restart node_exporter
  tags: monitoring

- name: Запуск и включение автозагрузки node_exporter
  systemd:
    name: node_exporter
    daemon_reload: yes
    state: started
    enabled: yes
  tags: monitoring

- name: Открыть порт 9100/tcp в firewalld (метрики Node Exporter)
  firewalld:
    port: 9100/tcp
    permanent: yes
    state: enabled
    immediate: yes
  when: ansible_os_family == "RedHat"
  tags: [monitoring, firewalld]

- name: Проверка, что Node Exporter отвечает на /metrics
  shell: curl -s http://127.0.0.1:9100/metrics | head -1
  args:
    warn: false
  register: node_exporter_check
  changed_when: false
  failed_when: node_exporter_check.rc != 0
  tags: monitoring
...
EOF
```
##### Шаг 4. Добавляем роль в главный плейбук site.yml
Убедитесь, что роль monitoring добавлена в конец списка ролей в файле /etc/ansible/site.yml.

Ваш site.yml должен выглядеть так:
```bash
cat <<'EOF' > /etc/ansible/site.yml
---
- name: Комплексная настройка веб-кластера, безопасности и Docker
  hosts: servers
  become: yes
  roles:
    - webserver
    - backup
    - security
    - docker
    - deploy_app
    - loadbalancer
    - syslog
    - monitoring
...
EOF
```
##### Шаг 5. Запуск роли (с нуля или обновление)
Теперь вы можете установить Node Exporter на обе ноды одной командой:
```bash
ansible-playbook /etc/ansible/site.yml --tags monitoring
```
Ожидаемый результат: Ansible выполнит все задачи (создаст пользователя, скачает и распакует архив, скопирует бинарник, создаст сервис, запустит его и откроет порт). В PLAY RECAP должно быть failed=0.

##### Шаг 6. Проверка работоспособности
После завершения плейбука проверьте, что агент работает на обеих нодах:
```bash
# 1. Проверяем, что служба активна
ansible servers -m shell -a "systemctl is-active node_exporter"
# Ожидаемо: "active" на обоих хостах

# 2. Проверяем, что метрики отдаются локально
ansible servers -m shell -a "curl -s http://127.0.0.1:9100/metrics | head -3"
# Ожидаемо: строки "# HELP go_gc_duration_seconds..."

# 3. Проверяем, что порт открыт в firewalld
ansible servers -m shell -a "firewall-cmd --list-ports"
# Ожидаемо: наличие "9100/tcp" в списке

# 4. Проверяем доступность метрик снаружи (с centpca)
curl -s http://10.1.100.12:9100/metrics | head -3
curl -s http://10.1.100.14:9100/metrics | head -3
```

<img width="1150" height="463" alt="image" src="https://github.com/user-attachments/assets/51f2d11e-eeb6-499d-9f64-a49fbb8d32d5" />

---

<img width="909" height="879" alt="image" src="https://github.com/user-attachments/assets/b90c02b2-74c8-4094-a951-d9b4a86cae96" />

---

<img width="1199" height="569" alt="45" src="https://github.com/user-attachments/assets/1f95a2e4-6fed-4ee1-9289-81d4827c406e" />

---

<img width="1373" height="553" alt="47" src="https://github.com/user-attachments/assets/4c72d60a-7cc5-4a61-9efa-77726d803599" />

---
**
---

<img width="1366" height="639" alt="31" src="https://github.com/user-attachments/assets/5bc1ca23-288d-49bb-8e04-fd7fa70ccc33" />

Это идеальная картина — именно то, что должно быть:

Endpoint	|State	Labels  	|Last Scrape
--------- | --------------- | ----------------
http://10.1.100.12:9100/metrics	  | UP	instance="10.1.100.12:9100", job="node_exporter"	 | 6.453s ago
http://10.1.100.14:9100/metrics	  | UP	instance="10.1.100.14:9100", job="node_exporter"	 | 6.570s ago
http://localhost:9091/metrics	    | UP	instance="localhost:9091", job="prometheus"	       | 1.285s ago

Все три target'а UP, Error пустой, Scrape Duration ~12–15ms. Prometheus успешно собирает метрики с обеих нод и с себя. Мониторинг работает.

- Prometheus Server запущен и слушает на :9091 — раз /metrics отвечает.
- Firewalld пропускает 9091 снаружи — раз вы открыли 10.1.100.10:9091 из браузера с другой машины (не с localhost).
- Self-monitoring работает — Prometheus забирает эти же метрики через job_name: 'prometheus' (target localhost:9091 — UP).

##### Как посмотреть метрики с нод, а не только сервера
Endpoint /metrics на :9091 — это метрики самого Prometheus. А метрики с нод (centpc1, centpc3) хранятся в TSDB и доступны через:

Веб-интерфейс (самый наглядный способ)

Откройте http://10.1.100.10:9091 (без /metrics) — это UI Prometheus:
```txt
Status → Targets — увидите три зелёных target'а (prometheus, node_exporter × 2).

Graph — введите запрос, например:

node_cpu_seconds_total — CPU с обеих нод

node_memory_MemAvailable_bytes — свободная память

node_filesystem_avail_bytes — свободное место на дисках

node_load1 — load average

node_network_receive_bytes_total — сеть

up — статус всех target'ов (1 = UP, 0 = DOWN)

Нажмите Execute, затем вкладку Graph — увидите графики. Под графиком будет список instance: 10.1.100.12:9100 и 10.1.100.14:9100 — метрики с обеих нод.
```

<img width="1379" height="743" alt="34" src="https://github.com/user-attachments/assets/dba47d65-68c0-4c25-a4c2-9745f987d68b" />

##### Как вывести красивые графики для каждой из метрик, которые вы перечислили.

##### 1. node_cpu_seconds_total — CPU с обеих нод
Сырое значение — это просто счётчик секунд, накопленных за всё время. Чтобы увидеть реальную нагрузку в процентах, нужно использовать функцию rate.
Введите в поле запроса:
```promql
rate(node_cpu_seconds_total{mode!="idle"}[5m])
```
{mode!="idle"} — исключаем время простоя (иначе графики будут показывать 99%).

rate(...[5m]) — считаем скорость изменения за последние 5 минут.

Нажмите Execute и переключитесь на вкладку Graph. Вы увидите график с линиями для каждого ядра на каждой ноде.

##### 2. node_memory_MemAvailable_bytes — свободная память
Чтобы график был читаемым (в гигабайтах, а не в миллионах байт), используйте деление:

```promql
node_memory_MemAvailable_bytes / 1024 / 1024 / 1024
```
Нажмите Execute → вкладка Graph. Увидите две линии (для .12 и .14), показывающие свободную память в ГБ.

##### 3. node_filesystem_avail_bytes — свободное место на дисках
Здесь важно отфильтровать виртуальные файловые системы (tmpfs, overlay и т.д.), иначе график будет забит мусором:

```promql
node_filesystem_avail_bytes{fstype!="tmpfs", fstype!="overlay"} / 1024 / 1024 / 1024
```
Нажмите Execute → вкладка Graph. Увидите свободное место в ГБ для реальных дисков на обеих нодах.

##### 4. node_load1 — load average
Здесь сырые данные уже вполне информативны, можно смотреть прямо так:

```promql
node_load1
```
Нажмите Execute → вкладка Graph. Увидите две линии — среднюю нагрузку на обеих нодах за 1 минуту.

##### 5. node_network_receive_bytes_total — сеть
Сырое значение — это счётчик принятых байт. Чтобы увидеть скорость (байт/сек), используйте rate:

```promql
rate(node_network_receive_bytes_total{device!="lo"}[5m])
{device!="lo"} — исключаем loopback-интерфейс.
```
Нажмите Execute → вкладка Graph. Увидите график входящего трафика по всем сетевым картам (включая ens33 и docker0).

##### 6. up — статус всех target'ов
Чтобы убедиться, что всё работает:
```promql
up
```
Нажмите Execute → вкладка Graph (или Table). Значение 1 означает, что узел доступен (UP), 0 — недоступен (DOWN). У вас сейчас должно быть три линии со значением 1 (две ноды и сам Prometheus).

#### Через API (для проверки из консоли)
```bash
# Метрики CPU с обеих нод
curl -s 'http://10.1.100.10:9091/api/v1/query?query=node_cpu_seconds_total' | python -m json.tool | grep -E '"instance"' | sort -u
# Ожидаемо: 10.1.100.12:9100 и 10.1.100.14:9100

# Статус всех target'ов (1 = UP)
curl -s 'http://10.1.100.10:9091/api/v1/query?query=up' | python -m json.tool | grep -E '"instance"|"value"'
# Ожидаемо: три записи с "value": [..., "1"]

# Свободная память на нодах
curl -s 'http://10.1.100.10:9091/api/v1/query?query=node_memory_MemAvailable_bytes' | python -m json.tool | grep -E '"instance"|"value"'
```

<img width="1239" height="248" alt="image" src="https://github.com/user-attachments/assets/67ac42a1-f1a8-4085-8991-90f2695137cf" />

---
**
---

#### Итоговая схема

<img width="753" height="415" alt="image" src="https://github.com/user-attachments/assets/ec6c70aa-173b-42e6-94a4-936d60e3c753" />

---
