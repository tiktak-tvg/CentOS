#### Лабораторная работа на стенде.
Делаем систему по схеме. С этими конфигами ansible-playbook /etc/ansible/site.yml на чистых CentOS 7 развернёт весь стенд с нуля за один прогон — без ручных действий.

<img width="737" height="328" alt="image" src="https://github.com/user-attachments/assets/7d4c44b8-6e49-4c7b-b11d-7efa05d25742" />

---

Слой   	| Компонент	                 |Порт
--------|----------------------------|------------------------
Отказоустойчивость |	Keepalived (VIP 10.1.100.100)	VRRP  |	(112)
Балансировка	 |	HAProxy (roundrobin)	 |	80
Мониторинг	 |	HAProxy stats	 |	8404
Веб-сервер (системный)	 |	nginx	 |	81
Веб-сервер (контейнер)	 |	nginx:1.21 в Docker	 |	8080
Контент	 |	Шаблон Jinja2 (/usr/share/nginx/html) |		—


#### Предварительные условия
Хост	  |IP	                          |Роль
--------|-----------------------------|------------------------
centpca	—	управляющая машина  | 10.1.100.10 | (Ansible)
centpc1	 |10.1.100.12	MASTER  |Keepalived + веб-узел
centpc3	 |10.1.100.14	BACKUP  |Keepalived + веб-узел
VIP	 |10.1.100.100	 |плавающий адрес

На всех трёх: CentOS 7.9, SSH-ключи настроены, centpca может ходить по SSH на .12 и .14.

##### 1. Инвентарь /etc/ansible/hosts
```ini
[servers]
10.1.100.12 ansible_user=root
10.1.100.14 ansible_user=root
```
##### 2. site.yml — главный плейбук
```yaml
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
```
Порядок ролей критичен: **webserver** (генерирует шаблон) → **deploy_app** (монтирует его в контейнер) → **loadbalancer** (VRRP, порты).

##### 3. Роль webserver — /etc/ansible/roles/webserver/tasks/main.yml
```yaml
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
```
Шаблон **/etc/ansible/roles/webserver/templates/index.html.j2**
```html
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
```
##### 4. Роль backup — /etc/ansible/roles/backup/tasks/main.yml
```yaml
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
```
##### 5. Роль security — /etc/ansible/roles/security/tasks/main.yml
```yaml
---
- name: Отключение авторизации по паролям в SSH (разрешаем только ключи)
  lineinfile:
    path: /etc/ssh/sshd_config
    regexp: '^#?PasswordAuthentication'
    line: 'PasswordAuthentication no'
    state: present
  notify: restart sshd

- name: Отключение прямого удаленного входа для root
  lineinfile:
    path: /etc/ssh/sshd_config
    regexp: '^#?PermitRootLogin'
    line: 'PermitRootLogin prohibit-password'
    state: present
  notify: restart sshd
...
```
Хэндлер **/etc/ansible/roles/security/handlers/main.yml**
```yaml
---
- name: restart sshd
  service:
    name: sshd
    state: restarted
...
```
##### 6. Роль docker — /etc/ansible/roles/docker/tasks/main.yml
```yaml
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
```
##### 7. Роль deploy_app — /etc/ansible/roles/deploy_app/tasks/main.yml
Ключевая роль — именно здесь все фиксы про docker-py, volumes и recreate.

```yaml
---
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
```
##### 8. Роль loadbalancer — /etc/ansible/roles/loadbalancer/tasks/main.yml
Вторая ключевая роль — **VRRP, порт 81, порт 8404, psmisc, socat, stats socket**.

```yaml
---
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
```
##### 9. Роль syslog — /etc/ansible/roles/syslog/tasks/main.yml
```yaml
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
```
Хэндлер **/etc/ansible/roles/syslog/handlers/main.yml**
```yaml
---
- name: restart rsyslog
  service:
    name: rsyslog
    state: restarted
...
```
##### 10. Порядок запуска с нуля
```bash
# 1. Проверить инвентарь
ansible servers -m ping

# 2. Проверить синтаксис плейбука
ansible-playbook /etc/ansible/site.yml --syntax-check

# 3. Запустить всё
ansible-playbook /etc/ansible/site.yml
# Ожидаемо:
[root@centpca ~]# ansible-playbook /etc/ansible/site.yml

PLAY [Комплексная настройка веб-кластера, безопасности и Docker] ***********************************************************************************************

TASK [Gathering Facts] *****************************************************************************************************************************************
ok: [10.1.100.14]
ok: [10.1.100.12]

TASK [webserver : Установка веб-сервера nginx] *****************************************************************************************************************
ok: [10.1.100.12]
ok: [10.1.100.14]

TASK [webserver : Генерация index.html из шаблона Jinja2] ******************************************************************************************************
ok: [10.1.100.12]
ok: [10.1.100.14]

TASK [webserver : Запуск и включение автозагрузки nginx] *******************************************************************************************************
ok: [10.1.100.14]
ok: [10.1.100.12]

TASK [backup : Создание директории для хранения локальных бэкапов на сервере] **********************************************************************************
ok: [10.1.100.14]
ok: [10.1.100.12]

TASK [backup : Создание bash-скрипта для упаковки архива] ******************************************************************************************************
ok: [10.1.100.12]
ok: [10.1.100.14]

TASK [backup : Добавление задачи в планировщик Cron (каждую ночь в 03:30)] *************************************************************************************
ok: [10.1.100.14]
ok: [10.1.100.12]

TASK [security : Отключение авторизации по паролям в SSH (разрешаем только ключи)] *****************************************************************************
ok: [10.1.100.12]
ok: [10.1.100.14]

TASK [security : Отключение прямого удаленного входа для root (опционально, для безопасности)] *****************************************************************
ok: [10.1.100.12]
ok: [10.1.100.14]

TASK [docker : Добавление официального репозитория Docker CE] **************************************************************************************************
ok: [10.1.100.12]
ok: [10.1.100.14]

TASK [docker : Обновление кэша yum] ****************************************************************************************************************************
ok: [10.1.100.12]
ok: [10.1.100.14]

TASK [docker : Установка Docker CE и зависимостей] *************************************************************************************************************
ok: [10.1.100.12]
ok: [10.1.100.14]

TASK [docker : Запуск службы Docker и добавление в автозагрузку] ***********************************************************************************************
ok: [10.1.100.14]
ok: [10.1.100.12]

TASK [docker : Скачивание и установка Docker Compose] **********************************************************************************************************
changed: [10.1.100.12]
changed: [10.1.100.14]

TASK [deploy_app : Удаление старых конфликтующих конфигов Nginx] ***********************************************************************************************
ok: [10.1.100.12] => (item=/etc/nginx/conf.d/default.conf)
ok: [10.1.100.14] => (item=/etc/nginx/conf.d/default.conf)
changed: [10.1.100.12] => (item=/etc/nginx/conf.d/app_proxy.conf)
changed: [10.1.100.14] => (item=/etc/nginx/conf.d/app_proxy.conf)

TASK [deploy_app : Установка системного certifi из EPEL (защита от синтаксических ошибок Python 3)] ************************************************************
ok: [10.1.100.12]
ok: [10.1.100.14]

TASK [deploy_app : Удаление конфликтующего старого docker-py (иначе docker_container не работает)] *************************************************************
ok: [10.1.100.12]
ok: [10.1.100.14]

TASK [deploy_app : Установка совместимого с Python 2.7 Docker SDK (без docker-py, с фиксированными requests/urllib3)] ******************************************
changed: [10.1.100.12]
changed: [10.1.100.14]

TASK [deploy_app : Проверка, что Docker SDK импортируется (python2)] *******************************************************************************************
ok: [10.1.100.12]
ok: [10.1.100.14]

TASK [deploy_app : Разрешить Nginx сетевые подключения (Исправление 502 Bad Gateway из-за SELinux)] ************************************************************
ok: [10.1.100.12]
ok: [10.1.100.14]

TASK [deploy_app : Создание локальной директории для кастомной страницы приложения с правами 0755] *************************************************************
ok: [10.1.100.14]
ok: [10.1.100.12]

TASK [deploy_app : Удаление старого контейнера с неверным volume (если есть)] **********************************************************************************
ok: [10.1.100.12]
ok: [10.1.100.14]

TASK [deploy_app : Запуск Docker-контейнера с веб-приложением (шаблон Jinja2 монтируется внутрь)] **************************************************************
changed: [10.1.100.12]
changed: [10.1.100.14]

TASK [deploy_app : Создание стабильной конфигурации проксирования через localhost на порт 81] ******************************************************************
changed: [10.1.100.12]
changed: [10.1.100.14]

TASK [deploy_app : Проверка синтаксиса конфигурации Nginx перед перезапуск-триггером] **************************************************************************
ok: [10.1.100.14]
ok: [10.1.100.12]

TASK [deploy_app : Включение автозагрузки и чистый перезапуск системного Nginx] ********************************************************************************
changed: [10.1.100.12]
changed: [10.1.100.14]

TASK [deploy_app : Проверка, что контейнер отдаёт шаблон Jinja2 (а не дефолтную страницу nginx)] ***************************************************************
ok: [10.1.100.14]
ok: [10.1.100.12]

TASK [loadbalancer : Установка пакетов haproxy, keepalived, psmisc (killall) и socat (stats)] ******************************************************************
ok: [10.1.100.12]
ok: [10.1.100.14]

TASK [loadbalancer : Убедиться, что firewalld установлен (RedHat)] *********************************************************************************************
ok: [10.1.100.12]
ok: [10.1.100.14]

TASK [loadbalancer : Убедиться, что firewalld запущен и в автозагрузке (RedHat)] *******************************************************************************
ok: [10.1.100.12]
ok: [10.1.100.14]

TASK [loadbalancer : Разрешить VRRP в firewalld (иначе split-brain Keepalived)] ********************************************************************************
ok: [10.1.100.14]
ok: [10.1.100.12]

TASK [loadbalancer : Открыть порт 81/tcp в firewalld (nginx :81 для HAProxy healthcheck)] **********************************************************************
ok: [10.1.100.12]
ok: [10.1.100.14]

TASK [loadbalancer : Открыть порт 8404/tcp в firewalld (HAProxy stats в браузере)] *****************************************************************************
ok: [10.1.100.14]
ok: [10.1.100.12]

TASK [loadbalancer : Настройка конфигурации HAProxy (Балансировка веб-кластера + stats)] ***********************************************************************
ok: [10.1.100.14]
ok: [10.1.100.12]

TASK [loadbalancer : Включение автозагрузки и запуск службы HAProxy] *******************************************************************************************
changed: [10.1.100.14]
changed: [10.1.100.12]

TASK [loadbalancer : Настройка Keepalived для хоста 10.1.100.12 (MASTER)] **************************************************************************************
skipping: [10.1.100.14]
ok: [10.1.100.12]

TASK [loadbalancer : Настройка Keepalived для хоста 10.1.100.14 (BACKUP)] **************************************************************************************
skipping: [10.1.100.12]
ok: [10.1.100.14]

TASK [loadbalancer : Включение автозагрузки и запуск службы Keepalived] ****************************************************************************************
changed: [10.1.100.12]
changed: [10.1.100.14]

TASK [Конфигурация Rsyslog как Центрального Сервера (Приемника)] ***********************************************************************************************
skipping: [10.1.100.14]
ok: [10.1.100.12]

TASK [Конфигурация Rsyslog как Клиента (Отправителя)] **********************************************************************************************************
skipping: [10.1.100.12]
ok: [10.1.100.14]

TASK [Проверка синтаксиса конфигурации Rsyslog] ****************************************************************************************************************
changed: [10.1.100.12]
changed: [10.1.100.14]

TASK [Перезапуск службы Rsyslog для применения настроек] *******************************************************************************************************
changed: [10.1.100.12]
changed: [10.1.100.14]

TASK [syslog : Настройка логирования HAProxy в Syslog (проверка файла конфигурации)] ***************************************************************************
changed: [10.1.100.12]
changed: [10.1.100.14]

TASK [syslog : Перезапуск HAProxy при обновлении настроек логирования] *****************************************************************************************
changed: [10.1.100.12]
changed: [10.1.100.14]

PLAY RECAP *****************************************************************************************************************************************************
10.1.100.12                : ok=42   changed=12   unreachable=0    failed=0    skipped=2    rescued=0    ignored=0
10.1.100.14                : ok=42   changed=12   unreachable=0    failed=0    skipped=2    rescued=0    ignored=0

```

<img width="1122" height="590" alt="image" src="https://github.com/user-attachments/assets/badf66b9-a611-465d-acec-64691401bd47" />

##### Что произойдёт по порядку:

**webserver** — ставит nginx, генерирует /usr/share/nginx/html/index.html из Jinja2-шаблона

**backup** — создаёт /var/backups/web, скрипт, cron в 03:30

**security** — SSH: только ключи, root не логинится по паролю

**docker** — ставит Docker CE, запускает

**deploy_app** — удаляет docker-py, ставит docker==4.4.4, запускает контейнер с монтированием /usr/share/nginx/html:ro, настраивает nginx :81 → :8080

**loadbalancer** — VRRP в firewalld, порт 81, порт 8404, HAProxy roundrobin, Keepalived MASTER/BACKUP

**syslog** — rsyslog сервер на .12, клиент на .14, логи HAProxy в /var/log/haproxy.log

##### 11. Финальные проверки
```bash
# 1. Оба backend'а UP
ansible servers -m shell -a "echo 'show stat' | socat stdio /var/run/haproxy.sock | cut -d, -f1,2,18"
# Ожидаемо: web_node1,UP / web_node2,UP
```

<img width="1115" height="278" alt="image" src="https://github.com/user-attachments/assets/e283a9e7-5348-4f8e-9dd4-e2df041862dc" />

```bash
# 2. VIP на MASTER
ansible servers -m shell -a "ip addr show ens33 | grep 10.1.100.100 || echo NO_VIP"
# Ожидаемо: на .12 — VIP, на .14 — NO_VIP
```

<img width="1091" height="104" alt="image" src="https://github.com/user-attachments/assets/5a54c910-99a6-479d-9099-28d72d132772" />

```bash
# 3. Балансировка через VIP
for i in 1 2 3 4 5 6; do curl -s http://10.1.100.100 | grep -oP 'Имя хоста: <b>\K[^<]+'; done
# Ожидаемо:
[root@centpca ~]# for i in 1 2 3 4 5 6; do curl -s http://10.1.100.100 | grep -oP 'Имя хоста: <b>\K[^<]+'; done
centpc1
centpc3
centpc1
centpc3
centpc1
centpc3
[root@centpca ~]#

# 4. Stats в браузере
curl -s http://10.1.100.12:8404/stats | grep web_node
# Ожидаемо: оба UP
```

<img width="1122" height="141" alt="image" src="https://github.com/user-attachments/assets/22d78987-a5c3-43e2-a17c-1ec1c831da2e" />

--- 

<img width="1143" height="338" alt="image" src="https://github.com/user-attachments/assets/767c2373-3eff-43d8-8c53-d00dbea640fb" />

```bash
# 5. Firewalld
ansible servers -m shell -a "firewall-cmd --list-ports; firewall-cmd --list-rich-rules"
# Ожидаемо: 81/tcp 8404/tcp + rule protocol value="vrrp" accept
Если нет, добавить
ansible servers -m shell -a "firewall-cmd --permanent --add-port=8404/tcp && firewall-cmd --reload"
ansible servers -m shell -a "firewall-cmd --permanent --add-port=81/tcp && firewall-cmd --reload"
ansible servers -m shell -a "firewall-cmd --list-all | grep -E 'ports|services'"
```

<img width="1105" height="196" alt="image" src="https://github.com/user-attachments/assets/4edfa6f4-1992-4574-8b1d-fb40975df9cf" />

```bash
Удаление, если ошиблись
ansible servers -m shell -a "firewall-cmd --permanent --remove-port=8484/tcp && firewall-cmd --reload"
```
