### Проекты для двух машин (Простая отказоустойчивость)
Здесь уже можно отработать базовые сценарии высокой доступности.

Отказоустойчивый кластер БД (PostgreSQL) на узлах 10.1.100.21 (Primary) и 10.1.100.22 (Standby) с потоковой репликацией.

#### Этап 1: Подготовка управляющей машины (10.1.100.10)
Установите Ansible, необходимые коллекции и настройте SSH-доступ.
```bash
# 1. Установка Ansible
yum install -y ansible

# 2. Установка коллекции для работы с SELinux
ansible-galaxy collection install ansible.posix

# 3. Копирование SSH-ключа на целевую машину
ssh-keygen -t rsa -b 4096 -N "" -f ~/.ssh/id_rsa  # если ключа еще нет
ssh-copy-id root@10.1.100.21
ssh-copy-id root@10.1.100.22
```
Проверяем доступ:
```bash
ssh root@10.1.100.21 'hostname'
ssh root@10.1.100.22 'hostname'
```
### Кластер Базы Данных (PostgreSQL с репликацией): Ansible может настроить на первом узле мастер (Primary), а на втором — реплику (Standby), которая будет получать данные в режиме реального времени. Это дает отказоустойчивость и возможность распределять нагрузку на чтение.
#### Этап 2: Подготовка зависимостей
Устанавливаем необходимые коллекции Ansible (включая ansible.posix для модуля firewalld) и утилиту yum-utils.
```bash
# Установка коллекций (ставим в системную папку, чтобы было доступно всем)
ansible-galaxy collection install community.postgresql ansible.posix -p /usr/share/ansible/collections

# Установка yum-utils на целевые хосты
ansible postgres_cluster -m yum -a "name=yum-utils state=present"
```
#### Этап 3: Создание конфигураций (EOF)
##### 1. Инвентарь /etc/ansible/hosts
```bash
cat <<'EOF' > /etc/ansible/hosts
[postgres_cluster]
10.1.100.21 ansible_user=root pg_role=primary
10.1.100.22 ansible_user=root pg_role=standby
EOF
```
Проверка доступности хостов через ping
```bash
ansible postgres_cluster -m ping
```

<img width="1112" height="316" alt="image" src="https://github.com/user-attachments/assets/dc8c639a-fe8e-4f37-82e3-319eaf520b9c" />

Проверка правил firewalld (правильный синтаксис, уже добавленный порт 5432)
```bash
ansible postgres_cluster -m shell -a "firewall-cmd --list-all"
```

<img width="1110" height="491" alt="image" src="https://github.com/user-attachments/assets/f6b2a80b-309d-4518-a01f-caf9ebab0f6a" />

##### 2. Глобальные переменные /etc/ansible/group_vars/postgres_cluster.yml
(Создаем папку и файл, чтобы переменные были видны всем ролям)
```bash
mkdir -p /etc/ansible/group_vars
cat <<'EOF' > /etc/ansible/group_vars/postgres_cluster.yml
---
postgres_version: 14
postgres_port: 5432
postgres_data_dir: "/var/lib/pgsql/{{ postgres_version }}/data"
postgres_config_dir: "{{ postgres_data_dir }}"
replication_user: "replicator"
replication_password: "StrongReplPass123!"
EOF
```
##### 3. Роль postgres_common
```bash
mkdir -p /etc/ansible/roles/postgres_common/{tasks,handlers}

cat <<'EOF' > /etc/ansible/roles/postgres_common/tasks/main.yml
---
- name: Установка репозитория PGDG
  yum:
    name: "https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm"
    state: present

- name: Отключение устаревших репозиториев (pgdg12, pgdg13)
  shell: "yum-config-manager --disable pgdg12 pgdg13 || true"
  changed_when: false

- name: Установка необходимых пакетов
  yum:
    name:
      - "postgresql{{ postgres_version }}-server"
      - "python-psycopg2"
      - "firewalld"
    state: present

- name: Включение и запуск firewalld
  service:
    name: firewalld
    state: started
    enabled: yes

- name: Открыть порт PostgreSQL (5432/tcp) в firewalld
  firewalld:
    port: "{{ postgres_port }}/tcp"
    permanent: yes
    state: enabled
    immediate: yes

- name: Инициализация базы данных
  shell: "/usr/pgsql-{{ postgres_version }}/bin/postgresql-{{ postgres_version }}-setup initdb"
  args:
    creates: "{{ postgres_data_dir }}/PG_VERSION"

- name: Настройка параметров в postgresql.conf
  lineinfile:
    path: "{{ postgres_config_dir }}/postgresql.conf"
    regexp: "{{ item.regexp }}"
    line: "{{ item.line }}"
  loop:
    - { regexp: '^#?listen_addresses', line: "listen_addresses = '*'" }
    - { regexp: '^#?wal_level', line: "wal_level = replica" }
    - { regexp: '^#?max_wal_senders', line: "max_wal_senders = 10" }
    - { regexp: '^#?wal_keep_size', line: "wal_keep_size = 1GB" }
    - { regexp: '^#?hot_standby', line: "hot_standby = on" }
  notify: restart postgresql

- name: Запуск и включение службы PostgreSQL
  service:
    name: "postgresql-{{ postgres_version }}"
    state: started
    enabled: yes
EOF
```
```bash
cat <<'EOF' > /etc/ansible/roles/postgres_common/handlers/main.yml
---
- name: restart postgresql
  service:
    name: "postgresql-{{ postgres_version }}"
    state: restarted
EOF
```
##### 4. Роль postgres_primary
```bash
mkdir -p /etc/ansible/roles/postgres_primary/{tasks,handlers}

cat <<'EOF' > /etc/ansible/roles/postgres_primary/tasks/main.yml
---
- name: Разрешить подключение реплик в pg_hba.conf
  lineinfile:
    path: "{{ postgres_config_dir }}/pg_hba.conf"
    line: "host replication {{ replication_user }} {{ hostvars[item].ansible_default_ipv4.address }}/32 scram-sha-256"
    state: present
  loop: "{{ groups['postgres_cluster'] | difference([inventory_hostname]) }}"
  notify: reload postgresql

- name: Применение конфигурации pg_hba.conf (принудительно)
  meta: flush_handlers

- name: Создание пользователя для репликации
  community.postgresql.postgresql_user:
    name: "{{ replication_user }}"
    password: "{{ replication_password }}"
    role_attr_flags: "REPLICATION,LOGIN"
    state: present
    login_unix_socket: /var/run/postgresql
  become_user: postgres

- name: Создание слота репликации
  community.postgresql.postgresql_slot:
    name: "{{ item | replace('.', '_') }}"
    slot_type: physical
    state: present
    login_unix_socket: /var/run/postgresql
  become_user: postgres
  loop: "{{ groups['postgres_cluster'] | difference([inventory_hostname]) }}"
EOF
```
```bash
cat <<'EOF' > /etc/ansible/roles/postgres_primary/handlers/main.yml
---
- name: reload postgresql
  service:
    name: "postgresql-{{ postgres_version }}"
    state: reloaded
EOF
```
##### 5. Роль postgres_standby
```bash
mkdir -p /etc/ansible/roles/postgres_standby/tasks

cat <<'EOF' > /etc/ansible/roles/postgres_standby/tasks/main.yml
---
- name: Остановка службы PostgreSQL на реплике
  service:
    name: "postgresql-{{ postgres_version }}"
    state: stopped

- name: Очистка директории с данными
  file:
    path: "{{ postgres_data_dir }}"
    state: absent

- name: Создание пустой директории для данных
  file:
    path: "{{ postgres_data_dir }}"
    state: directory
    owner: postgres
    group: postgres
    mode: '0700'

- name: Принудительная перезагрузка конфигурации на мастере перед копированием
  shell: |
    sudo -u postgres /usr/pgsql-{{ postgres_version }}/bin/pg_ctl reload -D {{ hostvars[groups['postgres_cluster'][0]].postgres_data_dir }}
  delegate_to: "{{ groups['postgres_cluster'][0] }}"
  changed_when: false

- name: Получение базовой копии с мастера (pg_basebackup)
  shell: |
    /usr/pgsql-{{ postgres_version }}/bin/pg_basebackup \
      -h {{ hostvars[groups['postgres_cluster'][0]].ansible_default_ipv4.address }} \
      -D {{ postgres_data_dir }} \
      -U {{ replication_user }} \
      -P -X stream -R -S {{ inventory_hostname | replace('.', '_') }}
  become_user: postgres
  environment:
    PGPASSWORD: "{{ replication_password }}"

- name: Запуск службы PostgreSQL на реплике
  service:
    name: "postgresql-{{ postgres_version }}"
    state: started
    enabled: yes
EOF
```
##### 6. Главный плейбук /etc/ansible/pg_cluster.yml
```bash
cat <<'EOF' > /etc/ansible/pg_cluster.yml
---
- name: Установка и базовая настройка PostgreSQL
  hosts: postgres_cluster
  become: yes
  roles:
    - postgres_common

- name: Настройка Primary (Мастера)
  hosts: postgres_cluster
  become: yes
  gather_facts: yes
  tasks:
    - name: Настройка мастера
      include_role:
        name: postgres_primary
      when: pg_role == 'primary'

- name: Настройка Standby (Реплики)
  hosts: postgres_cluster
  become: yes
  gather_facts: yes
  tasks:
    - name: Настройка реплики
      include_role:
        name: postgres_standby
      when: pg_role == 'standby'
EOF
```
#### Этап 4: Запуск и проверка
Запускаем плейбук. Первый прогон займет пару минут (скачивание пакетов, инициализация базы и копирование).
```bash
ansible-playbook /etc/ansible/pg_cluster.yml
```
Дождитесь PLAY RECAP с failed=0 на обоих хостах.

<img width="1109" height="58" alt="image" src="https://github.com/user-attachments/assets/a31ea7c5-cbf0-4b89-93f5-891d210f60b4" />

##### Проверка 1: Статус репликации (на мастере)
```bash
ssh root@10.1.100.21 "sudo -u postgres psql -c 'SELECT client_addr, state, sync_state FROM pg_stat_replication;'"
```
Ожидаемо: 10.1.100.22 | streaming | async.

##### Проверка 2: Режим восстановления (на реплике)
```bash
ssh root@10.1.100.22 "sudo -u postgres psql -c 'SELECT pg_is_in_recovery();'"
```
Ожидаемо: t.

<img width="1112" height="212" alt="image" src="https://github.com/user-attachments/assets/3fabe936-199c-4f25-8b63-6812211d3640" />

##### Проверка 3: Создаем таблицу на мастере (10.1.100.21) и создаем данные на мастере:
```bash
ssh root@10.1.100.21 "sudo -u postgres psql -c \"CREATE TABLE test_repl (id int, msg text); INSERT INTO test_repl VALUES (1, 'Clean Install Success');\""
```
##### Проверка 4: Читаем данные на реплике (10.1.100.22):
```bash
# Читаем данные на реплике
ssh root@10.1.100.22 "sudo -u postgres psql -c 'SELECT * FROM test_repl;'"
```
Ожидаемый результат:
```text
 id |         msg         
----+---------------------
  1 | Clean Install Success
(1 row)
```

<img width="1110" height="178" alt="image" src="https://github.com/user-attachments/assets/6e5b4f49-5700-4290-8b13-3e2dba936c21" />

---

> [!TIP]
> Если вы хотите переустановить всё заново, то сделаем полную очистку всего установленного.

Удалим всё, что связано с PostgreSQL, данные и старые конфигурации Ansible.

```bash
# 1. Останавливаем PostgreSQL на обоих хостах
ansible postgres_cluster -m service -a "name=postgresql-14 state=stopped" || true

# 2. Удаляем пакеты PostgreSQL с их конфигурациями
ansible postgres_cluster -m yum -a "name=postgresql14-server state=absent autoremove=yes"

# 3. Полностью удаляем директорию с данными (чтобы initdb отработал заново)
ansible postgres_cluster -m file -a "path=/var/lib/pgsql/14 state=absent"

# 4. Удаляем старые файлы Ansible
rm -rf /etc/ansible/roles/postgres_common
rm -rf /etc/ansible/roles/postgres_primary
rm -rf /etc/ansible/roles/postgres_standby
rm -f /etc/ansible/pg_cluster.yml
rm -f /etc/ansible/group_vars/postgres_cluster.yml
```

### Балансировка PostgreSQL (Разделение чтения/записи)
Настроить HAProxy так, чтобы он принимал SQL-запросы на порт 5432 и перенаправлял их:

- Все запросы на INSERT/UPDATE — на Primary (10.1.100.21).
- Все запросы на SELECT — распределял между Primary и Standby (10.1.100.22).
- Это классический паттерн для повышения производительности баз данных.

Для реализации разделения чтения и записи мы настроим HAProxy так, чтобы он слушал два разных порта: на 5432 будут приниматься запросы на запись (всегда перенаправляясь на Primary), а на 5433 — запросы на чтение (распределяясь между Primary и Standby по алгоритму round-robin). Это стандартный и безопасный паттерн для PostgreSQL.

#### Шаг 1: Создание пользователя для Healthcheck
HAProxy должен уметь проверять, жив ли тот или иной сервер PostgreSQL. Для этого создадим специального пользователя и разрешим ему подключаться с хоста, где будет работать HAProxy.

Выполните на Primary (10.1.100.21):
```bash
ansible 10.1.100.21 -m shell -a "sudo -u postgres psql -c 'CREATE USER haproxy_check WITH LOGIN;'"
```
Теперь разрешим этому пользователю подключаться с IP-адреса вашей машины, где будет установлен HAProxy (например, 10.1.100.10). Обратите внимание: замените 10.1.100.10 на реальный IP вашего HAProxy-хоста.

Настраиваем pg_hba.conf на ОБОИХ серверах (Primary и Standby):

Разрешаем подключение с IP-адреса HAProxy (10.1.100.10) для пользователя haproxy_check (для проверок) и для всех остальных пользователей, например postgres (для реального трафика). Используем trust для простоты тестового стенда (чтобы psql не спрашивал пароль).
```bash
# Добавляем правила в pg_hba.conf
ansible postgres_cluster -m shell -a "echo 'host all haproxy_check 10.1.100.10/32 trust' >> /var/lib/pgsql/14/data/pg_hba.conf"
ansible postgres_cluster -m shell -a "echo 'host all all 10.1.100.10/32 trust' >> /var/lib/pgsql/14/data/pg_hba.conf"

# Перезагружаем конфигурацию и применяем изменения
ansible postgres_cluster -m shell -a "sudo -u postgres /usr/pgsql-14/bin/pg_ctl reload -D /var/lib/pgsql/14/data"
```
Мы используем trust для простоты тестового стенда. В production-среде лучше использовать md5 и задать пароль.

#### Шаг 2: Настройка HAProxy (на 10.1.100.10)
Установите HAProxy на выбранный хост (например, на управляющую машину manager2122 или любой свободный сервер). Если HAProxy ещё не установлен:
```bash
yum install -y haproxy

# На хосте с HAProxy откройте порты для трафика и статистики: 5432 (запись), 5433 (чтение), 8404 (статистика)
firewall-cmd --permanent --add-port=5432/tcp
firewall-cmd --permanent --add-port=5433/tcp
firewall-cmd --permanent --add-port=8404/tcp
firewall-cmd --reload

# SELinux: разрешаем HAProxy ходить по сети и устанавливать сетевые соединения:
setsebool -P haproxy_connect_any on
```
Создайте конфигурационный файл /etc/haproxy/haproxy.cfg со следующим содержимым:
```haproxy
cat <<'EOF' > /etc/haproxy/haproxy.cfg
global
    log /dev/log local0
    maxconn 4096
    user haproxy
    group haproxy
    daemon
    stats socket /var/lib/haproxy/stats mode 660 level admin

defaults
    log global
    mode tcp
    retries 3
    timeout connect 5s
    timeout client 60s
    timeout server 60s

# --- Веб-интерфейс статистики ---
listen stats
    bind *:8404
    mode http
    stats enable
    stats uri /stats
    stats refresh 5s
    stats auth admin:SecurePass123

# --- Трафик на ЗАПИСЬ (порт 5432) ---
# Всегда перенаправляет на Primary
listen pgsql_write
    bind *:5432
    mode tcp
    option tcplog
    option pgsql-check user haproxy_check
    server pg-primary 10.1.100.21:5432 check inter 5s fall 3 rise 2

# --- Трафик на ЧТЕНИЕ (порт 5433) ---
# Распределяет по кругу (roundrobin) между Primary и Standby
listen pgsql_read
    bind *:5433
    mode tcp
    option tcplog
    balance roundrobin
    option pgsql-check user haproxy_check
    server pg-primary 10.1.100.21:5432 check inter 5s fall 3 rise 2
    server pg-standby 10.1.100.22:5432 check inter 5s fall 3 rise 2
EOF
```
В конфигурации выше Primary (10.1.100.21) указан первым в списке на чтение. Если вы хотите снизить нагрузку на мастер, вы можете либо убрать его из блока pgsql_read, либо пометить как backup, чтобы он использовался только если реплика недоступна.

```
#### Шаг 3: Запуск и проверка
Запустите HAProxy и проверьте его конфигурацию:
```bash
# Проверка синтаксиса
haproxy -c -f /etc/haproxy/haproxy.cfg

# Запуск и добавление в автозагрузку
systemctl restart haproxy
systemctl enable haproxy
```
Теперь проверим, что разделение работает. Выполните на хосте с HAProxy:

##### 4. Проверка записи (порт 5432):
Выполните эти команды на управляющей машине (10.1.100.10), где установлен HAProxy.

*1. Проверка веб-интерфейса (Stats)*

Откройте в браузере: http://10.1.100.10:8404/stats

Логин: **admin, Пароль: SecurePass123**

Убедитесь, что в блоках pgsql_write и pgsql_read все серверы горят зелёным (статус UP).

<img width="1374" height="604" alt="image" src="https://github.com/user-attachments/assets/97756430-3fab-44c1-8839-fe468e4f0b39" />

*2. Проверка ЗАПИСИ (порт 5432)*
```bash
psql -h 127.0.0.1 -p 5432 -U postgres -d postgres -c "SELECT inet_server_addr();"
```
Ожидаемый результат: всегда 10.1.100.21 (Primary).

Проверьте 3 раза, IP не должен меняться.

*3. Проверка ЧТЕНИЯ (порт 5433)*
```bash
for i in {1..6}; do psql -h 127.0.0.1 -p 5433 -U postgres -d postgres -t -c "SELECT inet_server_addr();"; done
```
Ожидаемый результат: Идеальное чередование:
```text
 10.1.100.21
 10.1.100.22
 10.1.100.21
 10.1.100.22
 10.1.100.21
 10.1.100.22
```

<img width="1107" height="309" alt="image" src="https://github.com/user-attachments/assets/1eb69f3b-9b56-4d3a-9f01-3d5e4a9912fe" />

*4. Проверка "Failover" (отказоустойчивость)*

Давайте проверим, что будет, если реплика упадёт.
```bash
# Останавливаем PostgreSQL на Standby (10.1.100.22)
ansible 10.1.100.22 -m service -a "name=postgresql-14 state=stopped"
# Подождите 15 секунд или Sleep
# Проверяем чтение (теперь все запросы должны идти на мастера) 
for i in {1..3}; do psql -h 127.0.0.1 -p 5433 -U postgres -d postgres -t -c "SELECT inet_server_addr();"; done
# Ожидаемо: 10.1.100.21 (три раза подряд)

# Возвращаем реплику в строй
ansible 10.1.100.22 -m service -a "name=postgresql-14 state=started"
```

<img width="1110" height="230" alt="image" src="https://github.com/user-attachments/assets/5ec03e5f-f827-43a5-921a-205feab023e6" />

Поздравляю! Теперь ваш кластер баз данных умеет грамотно распределять нагрузку, а HAProxy автоматически исключает упавшие узлы из балансировки

#### Итог
Теперь ваше приложение может отправлять запросы на запись на порт 5432, а запросы на чтение — на порт 5433. HAProxy будет автоматически отслеживать доступность серверов и распределять нагрузку. 

В случае падения Standby, все запросы на чтение продолжат обслуживаться Primary (если он включён в бэкенд чтения), а если упадёт Primary — запросы на запись будут недоступны до восстановления, но чтение продолжит работать через реплику.

> [!TIP]
> Если вы хотите переустановить всё заново, то сделаем полную очистку всего установленного.

##### Очистка (чтобы начать с нуля)
Выполните это на управляющей машине (10.1.100.10), чтобы удалить старые настройки.
```bash
# 1. Останавливаем HAProxy
systemctl stop haproxy || true

# 2. Удаляем старого пользователя из PostgreSQL (только на мастере, реплика подхватит)
ansible 10.1.100.21 -m shell -a "sudo -u postgres psql -c 'DROP USER IF EXISTS haproxy_check;'" || true

# 3. Очищаем старые правила в pg_hba.conf на обоих серверах
ansible postgres_cluster -m shell -a "sed -i '/10.1.100.10/d' /var/lib/pgsql/14/data/pg_hba.conf"
```






























