### Веб-кластер (Nginx + Docker + HAProxy + Keepalived) + логирование (rsyslog) + автоматическое резервное копирование (backup)
### Предварительные условия (руками, до Ansible)
Эти шаги не автоматизируются плейбуком — их надо сделать один раз на каждом хосте.

##### 0.1. Управляющая машина centpca
```bash
# Установить Ansible
yum install -y epel-release
yum ins

# Проверить версию (должна быть 2.9+)
ansible --version
```
##### 0.2. Все три хоста (.12, .14, centpca)
```bash
# Отключить SELinux (или оставить Enforcing — роль deploy_app ставит seboolean)
# Если оставляете Enforcing — ничего не делайте, роль справится.
# Если отключаете:
setenforce 0
sed -i 's/^SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config

# Убедиться, что firewalld запущен
systemctl enable --now firewalld

# Убедиться, что hostname корректный (для шаблона Jinja2)
hostnamectl set-hostname centpc1   # на .12
hostnamectl set-hostname centpc3   # на .14
```
##### 0.3. SSH-ключи с centpca на .12 и .14
```bash
# На centpca
ssh-keygen -t rsa -b 2048 -N "" -f ~/.ssh/id_rsa

# Скопировать ключ на оба хоста
ssh-copy-id root@10.1.100.12
ssh-copy-id root@10.1.100.14

# Проверить, что заходит без пароля
ssh root@10.1.100.12 'hostname'
ssh root@10.1.100.14 'hostname'
```
##### 0.4. Инвентарь /etc/ansible/hosts
```ini
[servers]
10.1.100.12 ansible_user=root
10.1.100.14 ansible_user=root
```
#### 0.5. Проверка связности
```bash
ansible servers -m ping
# Ожидаемо: оба хоста → SUCCESS
```
Если ping не проходит — дальше идти нельзя. Разбирайтесь с SSH/ключами/сетью.

#### 0.6. Удалить системный nginx (если он был)
```bash
ansible gitlab -m shell -a "systemctl stop nginx 2>/dev/null; systemctl disable nginx 2>/dev/null; yum remove -y nginx"
```
#### ЧАСТЬ 1. Структура ролей (создать один раз)
```bash
mkdir -p /etc/ansible/roles/{webserver,backup,security,docker,deploy_app,loadbalancer,syslog}/{tasks,templates,handlers,defaults}
mkdir -p /etc/ansible/roles/webserver/templates
```
#### ЧАСТЬ 2. Конфиги Ansible (по порядку)
##### 2.1. site.yml — главный плейбук
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
...
EOF
```
Порядок критичен: webserver (шаблон) → deploy_app (монтирует его в контейнер) → loadbalancer (VRRP + порты).

##### 2.2. Роль webserver
roles/webserver/tasks/main.yml
```bash
cat <<'EOF' > /etc/ansible/roles/webserver/tasks/main.yml
---
- name: Установка веб-сервера nginx
  yum:
    name: nginx
    state: present

- name: Генерация index.html из шаблона Jinja2
  template:
    src: templates/index.html.j2
    dest: /usr/share/nginx/html/index.html
    mode: '0644'

- name: Запуск и включение автозагрузки nginx
  service:
    name: nginx
    state: started
    enabled: yes
...
EOF
```
roles/webserver/templates/index.html.j2
```bash
cat <<'EOF' > /etc/ansible/roles/webserver/templates/index.html.j2
<html>
<head>
    <title>Ansible Managed Node</title>
</head>
<body style="font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; background-color: #f4f6f9; color: #333; margin: 0; padding: 50px; text-align: center;">
    <div style="background: white; max-width: 600px; margin: auto; padding: 30px; border-radius: 8px; box-shadow: 0 4px 15px rgba(0,0,0,0.1);">
        <h1 style="color: #4b6cb7;">🚀 Нода под управлением Ansible</h1>
        <hr style="border: 0; height: 1px; background: #eee; margin: 20px 0;">
        <p style="font-size: 18px;">Привет! Этот файл сгенерирован автоматически.</p>

        <table style="width: 100%; text-align: left; margin-top: 20px; border-collapse: collapse;">
            <tr style="background: #f8f9fa;"><th style="padding: 10px;">Параметр</th><th style="padding: 10px;">Значение</th></tr>
            <tr><td style="padding: 10px; border-bottom: 1px solid #eee;">Имя хоста:</td><td style="padding: 10px; border-bottom: 1px solid #eee; font-weight: bold; color: #e91e63;">{{ ansible_hostname }}</td></tr>
            <tr><td style="padding: 10px; border-bottom: 1px solid #eee;">IP-адрес:</td><td style="padding: 10px; border-bottom: 1px solid #eee; font-family: monospace;">{{ inventory_hostname }}</td></tr>
            <tr><td style="padding: 10px; border-bottom: 1px solid #eee;">ОС сервера:</td><td style="padding: 10px; border-bottom: 1px solid #eee;">{{ ansible_distribution }} {{ ansible_distribution_version }}</td></tr>
            <tr><td style="padding: 10px; border-bottom: 1px solid #eee;">Ядер CPU:</td><td style="padding: 10px; border-bottom: 1px solid #eee;">{{ ansible_processor_vcpus }}</td></tr>
        </table>

        <p style="margin-top: 30px; font-size: 12px; color: #999;">Время сборки конфигурации: 2026 год</p>
    </div>
</body>
</html>
EOF
```
##### 2.3. Роль backup
roles/backup/tasks/main.yml
```bash
cat <<'EOF' > /etc/ansible/roles/backup/tasks/main.yml
---
- name: Создание директории для хранения локальных бэкапов на сервере
  file:
    path: /var/backups/web
    state: directory
    mode: '0755'

- name: Создание bash-скрипта для упаковки архива
  copy:
    dest: /usr/local/bin/backup_web.sh
    mode: '0755'
    content: |
      #!/bin/bash
      DATE=$(date +%Y-%m-%d_%H-%M)
      tar -czf /var/backups/web/web_backup_$DATE.tar.gz /usr/share/nginx/html /etc/nginx
      find /var/backups/web -type f -mtime +7 -delete

- name: Добавление задачи в планировщик Cron (каждую ночь в 03:30)
  cron:
    name: "Backup web content"
    minute: "30"
    hour: "3"
    job: "/usr/local/bin/backup_web.sh"
...
EOF
```
##### 2.4. Роль security
roles/security/tasks/main.yml
```bash
cat <<'EOF' > /etc/ansible/roles/security/tasks/main.yml
---
- name: Отключение авторизации по паролям в SSH (разрешаем только ключи)
  lineinfile:
    path: /etc/ssh/sshd_config
    regexp: '^#?PasswordAuthentication'
    line: 'PasswordAuthentication no'
    state: present
  notify: restart sshd

- name: Отключение прямого удаленного входа для root (опционально)
  lineinfile:
    path: /etc/ssh/sshd_config
    regexp: '^#?PermitRootLogin'
    line: 'PermitRootLogin prohibit-password'
    state: present
  notify: restart sshd
...
EOF
```
roles/security/handlers/main.yml
```bash
cat <<'EOF' > /etc/ansible/roles/security/handlers/main.yml
---
- name: restart sshd
  service:
    name: sshd
    state: restarted
...
EOF
```
##### 2.5. Роль docker
roles/docker/tasks/main.yml
```bash
cat <<'EOF' > /etc/ansible/roles/docker/tasks/main.yml
---
- name: Добавление официального репозитория Docker CE
  yum_repository:
    name: docker-ce-stable
    description: Docker CE Stable
    baseurl: https://download.docker.com/linux/centos/7/$basearch/stable
    gpgcheck: yes
    gpgkey: https://download.docker.com/linux/centos/gpg

- name: Обновление кэша yum
  yum:
    update_cache: yes

- name: Установка Docker CE и зависимостей
  yum:
    name:
      - docker-ce
      - docker-ce-cli
      - containerd.io
    state: present

- name: Запуск службы Docker и добавление в автозагрузку
  service:
    name: docker
    state: started
    enabled: yes

- name: Скачивание и установка Docker Compose
  get_url:
    url: https://github.com/docker/compose/releases/download/v2.24.0/docker-compose-linux-x86_64
    dest: /usr/local/bin/docker-compose
    mode: '0755'
...
EOF
```
##### 2.6. Роль deploy_app — ключевая (все фиксы здесь)
roles/deploy_app/tasks/main.yml
```bash
cat <<'EOF' > /etc/ansible/roles/deploy_app/tasks/main.yml
---
# tasks file for deploy_app
- name: Удаление старых конфликтующих конфигов Nginx
  file:
    path: "{{ item }}"
    state: absent
  loop:
    - /etc/nginx/conf.d/default.conf
    - /etc/nginx/conf.d/app_proxy.conf

- name: Установка системного certifi из EPEL (защита от синтаксических ошибок Python 3)
  yum:
    name: python2-certifi
    state: present
  ignore_errors: yes

- name: Удаление конфликтующего старого docker-py (иначе docker_container не работает)
  shell: |
    pip2 uninstall -y docker-py 2>/dev/null || true
    pip2 uninstall -y docker 2>/dev/null || true
    pip uninstall -y docker-py 2>/dev/null || true
    pip uninstall -y docker 2>/dev/null || true
  args:
    warn: false
  changed_when: false

- name: Установка совместимого с Python 2.7 Docker SDK (без docker-py, с фиксированными requests/urllib3)
  pip:
    name:
      - "requests<2.28"
      - "urllib3<1.27"
      - "docker==4.4.4"
    state: present
    extra_args: "--no-cache-dir"
    executable: pip2

- name: Проверка, что Docker SDK импортируется (python2)
  command: python2 -c "import docker; print(docker.__version__)"
  register: docker_sdk_version
  changed_when: false
  failed_when: docker_sdk_version.rc != 0

- name: Разрешить Nginx сетевые подключения (Исправление 502 Bad Gateway из-за SELinux)
  seboolean:
    name: httpd_can_network_connect
    state: yes
    persistent: yes

- name: Создание локальной директории для кастомной страницы приложения с правами 0755
  file:
    path: /usr/share/nginx/html
    state: directory
    mode: '0755'

- name: Удаление старого контейнера с неверным volume (если есть)
  shell: docker rm -f web_app_container 2>/dev/null || true
  args:
    warn: false
  changed_when: false

- name: Запуск Docker-контейнера с веб-приложением (шаблон Jinja2 монтируется внутрь)
  docker_container:
    name: web_app_container
    image: nginx:1.21
    state: started
    recreate: yes
    ports:
      - "8080:80"
    volumes:
      - /usr/share/nginx/html:/usr/share/nginx/html:ro
    restart_policy: always

- name: Создание стабильной конфигурации проксирования через localhost на порт 81
  copy:
    dest: /etc/nginx/conf.d/app_proxy.conf
    content: |
      server {
          listen 81 default_server;
          server_name _;

          location / {
              proxy_pass http://127.0.0.1:8080;
              proxy_set_header Host $host;
              proxy_set_header X-Real-IP $remote_addr;
              proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          }
      }

- name: Проверка синтаксиса конфигурации Nginx перед перезапуск-триггером
  command: nginx -t
  changed_when: false

- name: Включение автозагрузки и чистый перезапуск системного Nginx
  service:
    name: nginx
    state: restarted
    enabled: yes

- name: Проверка, что контейнер отдаёт шаблон Jinja2 (а не дефолтную страницу nginx)
  shell: curl -s http://127.0.0.1:8080 | grep -E 'Имя хоста:|ОС сервера:'
  args:
    warn: false
  register: container_serves_template
  changed_when: false
  failed_when: container_serves_template.rc != 0
...
EOF
```
##### 2.7. Роль loadbalancer — ключевая (VRRP + порты)
roles/loadbalancer/tasks/main.yml
```bash
cat <<'EOF' > /etc/ansible/roles/loadbalancer/tasks/main.yml
---
# tasks file for loadbalancer (HAProxy + Keepalived VIP)
- name: Установка пакетов haproxy, keepalived, psmisc (killall) и socat (stats)
  yum:
    name:
      - haproxy
      - keepalived
      - psmisc
      - socat
    state: present
  tags: loadbalancer

- name: Убедиться, что firewalld установлен (RedHat)
  yum:
    name: firewalld
    state: present
  when: ansible_os_family == "RedHat"
  tags: [loadbalancer, firewalld]

- name: Убедиться, что firewalld запущен и в автозагрузке (RedHat)
  systemd:
    name: firewalld
    state: started
    enabled: yes
  when: ansible_os_family == "RedHat"
  tags: [loadbalancer, firewalld]

- name: Разрешить VRRP в firewalld (иначе split-brain Keepalived)
  firewalld:
    rich_rule: 'rule protocol value="vrrp" accept'
    permanent: yes
    state: enabled
    immediate: yes
  when: ansible_os_family == "RedHat"
  tags: [loadbalancer, firewalld]

- name: Открыть порт 81/tcp в firewalld (nginx :81 для HAProxy healthcheck)
  firewalld:
    port: 81/tcp
    permanent: yes
    state: enabled
    immediate: yes
  when: ansible_os_family == "RedHat"
  tags: [loadbalancer, firewalld]

- name: Открыть порт 8404/tcp в firewalld (HAProxy stats в браузере)
  firewalld:
    port: 8404/tcp
    permanent: yes
    state: enabled
    immediate: yes
  when: ansible_os_family == "RedHat"
  tags: [loadbalancer, firewalld]

- name: Настройка конфигурации HAProxy (Балансировка веб-кластера + stats)
  copy:
    dest: /etc/haproxy/haproxy.cfg
    content: |
      global
          log         127.0.0.1 local2
          chroot      /var/lib/haproxy
          pidfile     /var/run/haproxy.pid
          maxconn     4000
          user        haproxy
          group       haproxy
          daemon
          stats socket /var/run/haproxy.sock mode 660 level admin

      defaults
          mode                    http
          log                     global
          option                  httplog
          option                  dontlognull
          option http-server-close
          option forwardfor       except 127.0.0.1
          option                  redispatch
          retries                 3
          timeout http-request    10s
          timeout queue           1m
          timeout connect         10s
          timeout client          1m
          timeout server          1m
          timeout http-keep-alive 10s
          timeout check           10s
          maxconn                 3000

      frontend main_web_frontend
          bind *:80
          default_backend web_cluster_backend

      backend web_cluster_backend
          balance roundrobin
          option httpchk GET /
          server web_node1 10.1.100.12:81 check
          server web_node2 10.1.100.14:81 check

      listen stats
          bind *:8404
          stats enable
          stats uri /stats
          stats refresh 5s
  tags: loadbalancer

- name: Включение автозагрузки и запуск службы HAProxy
  service:
    name: haproxy
    state: restarted
    enabled: yes
  tags: loadbalancer

- name: Настройка Keepalived для хоста 10.1.100.12 (MASTER)
  copy:
    dest: /etc/keepalived/keepalived.conf
    content: |
      vrrp_script check_haproxy {
          script "killall -0 haproxy"
          interval 2
          weight 2
      }

      vrrp_instance VI_1 {
          state MASTER
          interface ens33
          virtual_router_id 51
          priority 101
          advert_int 1
          authentication {
              auth_type PASS
              auth_pass secret123
          }
          virtual_ipaddress {
              10.1.100.100/24
          }
          track_script {
              check_haproxy
          }
      }
  when: inventory_hostname == '10.1.100.12'
  tags: loadbalancer

- name: Настройка Keepalived для хоста 10.1.100.14 (BACKUP)
  copy:
    dest: /etc/keepalived/keepalived.conf
    content: |
      vrrp_script check_haproxy {
          script "killall -0 haproxy"
          interval 2
          weight 2
      }

      vrrp_instance VI_1 {
          state BACKUP
          interface ens33
          virtual_router_id 51
          priority 100
          advert_int 1
          authentication {
              auth_type PASS
              auth_pass secret123
          }
          virtual_ipaddress {
              10.1.100.100/24
          }
          track_script {
              check_haproxy
          }
      }
  when: inventory_hostname == '10.1.100.14'
  tags: loadbalancer

- name: Включение автозагрузки и запуск службы Keepalived
  service:
    name: keepalived
    state: restarted
    enabled: yes
  tags: loadbalancer
...
EOF
```
##### 2.8. Роль syslog
roles/syslog/tasks/main.yml
```bash
cat <<'EOF' > /etc/ansible/roles/syslog/tasks/main.yml
---
- name: Конфигурация Rsyslog как Центрального Сервера (Приемника)
  copy:
    dest: /etc/rsyslog.d/server.conf
    content: |
      module(load="imudp")
      input(type="imudp" port="514")
      module(load="imtcp")
      input(type="imtcp" port="514")
      $template RemoteLogs,"/var/log/remote/%HOSTNAME%/%PROGRAMNAME%.log"
      *.* ?RemoteLogs
      & stop
  when: inventory_hostname == '10.1.100.12'
  notify: restart rsyslog

- name: Конфигурация Rsyslog как Клиента (Отправителя)
  copy:
    dest: /etc/rsyslog.d/client.conf
    content: |
      *.* @@10.1.100.12:514
  when: inventory_hostname == '10.1.100.14'
  notify: restart rsyslog

- name: Проверка синтаксиса конфигурации Rsyslog
  command: rsyslogd -N1
  changed_when: false

- name: Перезапуск службы Rsyslog для применения настроек
  service:
    name: rsyslog
    state: restarted
    enabled: yes

- name: Настройка логирования HAProxy в Syslog (проверка файла конфигурации)
  lineinfile:
    path: /etc/rsyslog.d/haproxy.conf
    line: 'local2.*    /var/log/haproxy.log'
    create: yes
  notify: restart rsyslog

- name: Перезапуск HAProxy при обновлении настроек логирования
  service:
    name: haproxy
    state: restarted
...
EOF
roles/syslog/handlers/main.yml
bash
cat <<'EOF' > /etc/ansible/roles/syslog/handlers/main.yml
---
- name: restart rsyslog
  service:
    name: rsyslog
    state: restarted
...
EOF
```
#### ЧАСТЬ 3. Запуск
```bash
# 1. Проверить синтаксис
ansible-playbook /etc/ansible/site.yml --syntax-check

# 2. Запустить всё с нуля
ansible-playbook /etc/ansible/site.yml
```
Ожидаемо в PLAY RECAP: failed=0 на обоих хостах. Первый прогон будет changed=много (всё создаётся с нуля). Второй прогон — changed=мало (идемпотентность).

#### ЧАСТЬ 4. Финальные проверки
```bash
# 1. Оба backend'а UP
ansible servers -m shell -a "echo 'show stat' | socat stdio /var/run/haproxy.sock | cut -d, -f1,2,18"
# Ожидаемо: web_node1,UP / web_node2,UP

# 2. VIP на MASTER
ansible servers -m shell -a "ip addr show ens33 | grep 10.1.100.100 || echo NO_VIP"
# Ожидаемо: на .12 — VIP, на .14 — NO_VIP

# 3. Балансировка через VIP
for i in 1 2 3 4 5 6; do curl -s http://10.1.100.100 | grep -oP 'Имя хоста: <b>\K[^<]+'; done
# Ожидаемо: centpc1 / centpc3 / centpc1 / centpc3 / ...

# 4. Stats в браузере
curl -s http://10.1.100.12:8404/stats | grep web_node
# Ожидаемо: оба UP

# 5. Firewalld
ansible servers -m shell -a "firewall-cmd --list-ports; firewall-cmd --list-rich-rules"
# Ожидаемо: 81/tcp 8404/tcp + rule protocol value="vrrp" accept

# 6. Шаблон через контейнер
ansible servers -m shell -a "curl -s http://127.0.0.1:8080 | grep 'Имя хоста:'"
# Ожидаемо: centpc1 / centpc3
```
Сводка: что делает каждая роль
Роль	| Что делает	| Ключевые фиксы
------|-------------|-----------------
webserver | Ставит nginx, генерирует index.html из Jinja2 | 	Шаблон в /usr/share/nginx/html
backup	 | /var/backups/web, скрипт, cron 03:30	 | —
security	 | SSH: только ключи, root не логинится	 | —
docker	 | Docker CE, Compose | 	—
deploy_app	 | Контейнер nginx:1.21 с монтированием шаблона,  | nginx :81 → :8080	volumes, recreate, удаление docker-py, requests<2.28
loadbalancer	 | HAProxy roundrobin, Keepalived VIP, stats	VRRP в firewalld, порт 81, порт 8404, psmisc, socat, stats socket
syslog	 | rsyslog сервер/клиент, логи HAProxy	—

Что не автоматизировано (руками, один раз)
- SSH-ключи с centpca на .12 и .14 — ssh-copy-id
- Ansible установлен на centpca
- Инвентарь /etc/ansible/hosts создан
- Hostname на хостах (centpc1, centpc3) — для шаблона Jinja2
- firewalld запущен на хостах (роль его запустит, но если он не установлен — yum install firewalld вручную)

Всё остальное — ansible-playbook /etc/ansible/site.yml с нуля за один прогон.

Порядок действий «с нуля» (кратко)
```text
1. yum install -y epel-release ansible           (на centpca)
2. ssh-keygen + ssh-copy-id на .12 и .14        (на centpca)
3. hostnamectl set-hostname centpc1/centpc3     (на хостах)
4. Создать /etc/ansible/hosts                   (на centpca)
5. ansible servers -m ping                      (проверка)
6. Создать все файлы ролей (cat <<'EOF' выше)   (на centpca)
7. ansible-playbook /etc/ansible/site.yml       (запуск)
8. Проверки (curl, socat, ip addr)              (на centpca)
```
После этого стенд работает: балансировка, отказоустойчивость, stats, шаблон Jinja2 через VIP.
