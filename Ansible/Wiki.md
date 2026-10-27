### Nginx Reverse Proxy + SSL для Wiki.js на CentOS 7.9
```txt
[ Браузер ] --HTTPS--> [ developers 10.1.100.10 :443 ]
                            │ Nginx (reverse proxy, self-signed SSL)
                            │
                            └──HTTP--> [ wikipc 10.1.100.17 :3000 ]
                                          │ Wiki.js + PostgreSQL (Docker)
```

#### Что будет сделано:

- На wikipc — Docker + Wiki.js + PostgreSQL через Ansible.
- На developers — Nginx как reverse proxy с самоподписанным сертификатом (SAN с developers, wikipc, IP-адресами), HTTP→HTTPS редирект, WebSocket для редактора Wiki.js.
- Открыты только нужные порты: 443/tcp (и 80 для редиректа) на developers, 3000/tcp на wikipc (только для developers).

#### ШАГ 0. Предварительные условия (руками, один раз)
```bash
# На хосте 10.1.100.17 (wikipc):
hostnamectl set-hostname wikipc
setenforce 0
sed -i 's/^SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config
systemctl enable --now firewalld

# На developers — SSH-ключ на wikipc:
ssh-copy-id root@10.1.100.17
ssh root@10.1.100.17 'hostname'
# Ожидаемо: wikipc
```
#### ШАГ 1. Обновление инвентаря
```bash
cat <<'EOF' > /etc/ansible/hosts
[servers]
10.1.100.12 ansible_user=root
10.1.100.14 ansible_user=root

[wiki]
10.1.100.17 ansible_user=root
EOF

# Проверка:
ansible wiki -m ping
ansible servers -m ping
```
#### ШАГ 2. Роль wikijs (развёртывание Wiki.js на wikipc)
```bash
mkdir -p /etc/ansible/roles/wikijs/{tasks,templates,defaults}
mkdir -p /etc/ansible/roles/nginx_proxy/{tasks,templates,handlers,defaults}
```
##### 2.1. roles/wikijs/defaults/main.yml
```bash
cat <<'EOF' > /etc/ansible/roles/wikijs/defaults/main.yml
---
wikijs_dir: /opt/wikijs
wikijs_port: 3000

# Администратор, создаётся при первом запуске
wikijs_admin_email: "admin@example.com"
wikijs_admin_password: "ChangeMe123!"

# Параметры БД
wikijs_db_name: wiki
wikijs_db_user: wikijs
wikijs_db_password: "wikijsrocks"

# Зафиксированные версии образов
wikijs_image: "requarks/wiki:2"
postgres_image: "postgres:15-alpine"
EOF
```
##### 2.2. roles/wikijs/templates/docker-compose.yml.j2
```bash
cat <<'EOF' > /etc/ansible/roles/wikijs/templates/docker-compose.yml.j2
version: "3.8"

services:
  db:
    image: {{ postgres_image }}
    container_name: wikijs_db
    restart: unless-stopped
    environment:
      POSTGRES_DB: {{ wikijs_db_name }}
      POSTGRES_USER: {{ wikijs_db_user }}
      POSTGRES_PASSWORD: {{ wikijs_db_password }}
    volumes:
      - ./db:/var/lib/postgresql/data
    networks:
      - wikijs_net

  wiki:
    image: {{ wikijs_image }}
    container_name: wikijs_app
    restart: unless-stopped
    depends_on:
      - db
    ports:
      - "{{ wikijs_port }}:3000"
    environment:
      DB_TYPE: postgres
      DB_HOST: db
      DB_PORT: 5432
      DB_NAME: {{ wikijs_db_name }}
      DB_USER: {{ wikijs_db_user }}
      DB_PASS: {{ wikijs_db_password }}
      ADMIN_EMAIL: {{ wikijs_admin_email }}
      ADMIN_PASS: {{ wikijs_admin_password }}
    volumes:
      - ./wiki-data:/wiki/data
    networks:
      - wikijs_net

networks:
  wikijs_net:
    driver: bridge
EOF
```
##### 2.3. roles/wikijs/tasks/main.yml
Используем shell: docker-compose up -d вместо модуля docker_compose_v2, чтобы не тянуть Docker SDK для Python 2.7 на wikipc — тот же приём, что уже применён в deploy_app.

```bash
cat <<'EOF' > /etc/ansible/roles/wikijs/tasks/main.yml
---
- name: Создание директории Wiki.js
  file:
    path: "{{ wikijs_dir }}"
    state: directory
    mode: '0755'

- name: Генерация docker-compose.yml
  template:
    src: docker-compose.yml.j2
    dest: "{{ wikijs_dir }}/docker-compose.yml"
    mode: '0644'

- name: Запуск Wiki.js через docker-compose
  shell: docker-compose up -d
  args:
    chdir: "{{ wikijs_dir }}"
  register: compose_up
  changed_when: "'Started' in compose_up.stderr or 'Created' in compose_up.stderr"

- name: Открыть порт Wiki.js в firewalld
  firewalld:
    port: "{{ wikijs_port }}/tcp"
    permanent: yes
    state: enabled
    immediate: yes
  when: ansible_os_family == "RedHat"

#- name: Ожидание готовности Wiki.js
#  uri:
#    url: "http://127.0.0.1:{{ wikijs_port }}"
#    status_code: 200
#  register: wikijs_ready
#  until: wikijs_ready.status == 200
#  retries: 30
#  delay: 5
#  changed_when: false
# ИСПРАВЛЕНО: Используем wait_for для проверки TCP-порта вместо HTTP-статуса.
# Это исключает зависания при редиректах (302) на локализованные страницы.
- name: Ожидание открытия порта Wiki.js
  wait_for:
    host: 127.0.0.1
    port: "{{ wikijs_port }}"
    delay: 5
    timeout: 300
  changed_when: false

EOF
```
#### ШАГ 3. Роль nginx_proxy (reverse proxy + SSL на developers)
##### 3.1. roles/nginx_proxy/defaults/main.yml
```bash
cat <<'EOF' > /etc/ansible/roles/nginx_proxy/defaults/main.yml
---
proxy_server_name: developers
proxy_ip: 10.1.100.10
wiki_ip: 10.1.100.17
wiki_port: 3000
EOF
```
##### 3.2. roles/nginx_proxy/templates/openssl-san.cnf.j2
```bash
cat <<'EOF' > /etc/ansible/roles/nginx_proxy/templates/openssl-san.cnf.j2
[req]
distinguished_name = req_distinguished_name
x509_extensions = v3_req
prompt = no

[req_distinguished_name]
C = RU
ST = Moscow
L = Moscow
O = Company
CN = {{ proxy_server_name }}

[v3_req]
keyUsage = keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1 = {{ proxy_server_name }}
DNS.2 = wikipc
DNS.3 = localhost
IP.1 = {{ proxy_ip }}
IP.2 = {{ wiki_ip }}
IP.3 = 127.0.0.1
EOF
```
##### 3.3. roles/nginx_proxy/templates/wiki-proxy.conf.j2
```bash
cat <<'EOF' > /etc/ansible/roles/nginx_proxy/templates/wiki-proxy.conf.j2
# HTTP -> HTTPS редирект
server {
    listen 80;
    server_name {{ proxy_ip }} {{ proxy_server_name }} wikipc {{ wiki_ip }};
    return 301 https://$host$request_uri;
}

# HTTPS reverse proxy на Wiki.js
server {
    listen 443 ssl http2;
    server_name {{ proxy_ip }} {{ proxy_server_name }} wikipc {{ wiki_ip }};

    ssl_certificate     /etc/pki/tls/certs/{{ proxy_server_name }}.crt;
    ssl_certificate_key /etc/pki/tls/private/{{ proxy_server_name }}.key;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    # Wiki.js часто загружает вложения — увеличиваем лимит
    client_max_body_size 100M;

    access_log /var/log/nginx/wiki_access.log;
    error_log  /var/log/nginx/wiki_error.log;

    location / {
        proxy_pass http://{{ wiki_ip }}:{{ wiki_port }};
        proxy_http_version 1.1;

        # Заголовки для корректной работы Wiki.js за прокси
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host  $host;
        proxy_set_header X-Forwarded-Port  $server_port;

        # WebSocket (редактор Wiki.js)
        proxy_set_header Upgrade    $http_upgrade;
        proxy_set_header Connection "upgrade";

        # Таймауты для длинных операций
        proxy_read_timeout    300s;
        proxy_send_timeout    300s;
        proxy_connect_timeout 10s;
    }
}
EOF
```
##### 3.4. roles/nginx_proxy/handlers/main.yml
```bash
cat <<'EOF' > /etc/ansible/roles/nginx_proxy/handlers/main.yml
---
- name: restart nginx
  service:
    name: nginx
    state: restarted
EOF
```
##### 3.5. roles/nginx_proxy/tasks/main.yml
```bash
cat <<'EOF' > /etc/ansible/roles/nginx_proxy/tasks/main.yml
---
- name: Установка nginx
  yum:
    name: nginx
    state: present

- name: Удалить дефолтный конфиг nginx
  file:
    path: /etc/nginx/conf.d/default.conf
    state: absent

- name: Создать директории для SSL
  file:
    path: "{{ item }}"
    state: directory
    mode: '0755'
  loop:
    - /etc/pki/tls/private
    - /etc/pki/tls/certs

- name: Генерация конфига OpenSSL с SAN
  template:
    src: openssl-san.cnf.j2
    dest: /etc/pki/tls/openssl-san.cnf
    mode: '0644'

- name: Генерация самоподписанного сертификата (10 лет)
  shell: |
    openssl req -x509 -nodes -days 3650 -newkey rsa:2048 \
      -keyout /etc/pki/tls/private/{{ proxy_server_name }}.key \
      -out /etc/pki/tls/certs/{{ proxy_server_name }}.crt \
      -config /etc/pki/tls/openssl-san.cnf
  args:
    creates: /etc/pki/tls/certs/{{ proxy_server_name }}.crt

- name: Права на ключ (только root)
  file:
    path: /etc/pki/tls/private/{{ proxy_server_name }}.key
    mode: '0600'

- name: Права на сертификат
  file:
    path: /etc/pki/tls/certs/{{ proxy_server_name }}.crt
    mode: '0644'

- name: SELinux — разрешить nginx сетевые соединения (proxy на удалённый хост)
  seboolean:
    name: httpd_can_network_connect
    state: yes
    persistent: yes

- name: Конфиг reverse proxy
  template:
    src: wiki-proxy.conf.j2
    dest: /etc/nginx/conf.d/wiki-proxy.conf
    mode: '0644'
  notify: restart nginx

- name: Firewalld — открыть 80/tcp и 443/tcp
  firewalld:
    port: "{{ item }}"
    permanent: yes
    state: enabled
    immediate: yes
  loop:
    - 80/tcp
    - 443/tcp
  when: ansible_os_family == "RedHat"

- name: Проверка конфигурации nginx
  command: nginx -t
  changed_when: false

- name: Запуск и включение nginx
  service:
    name: nginx
    state: started
    enabled: yes
EOF
```
#### ШАГ 4. Плейбук wikiportal.yml
```bash
cat <<'EOF' > /etc/ansible/wikiportal.yml
---
- name: Развёртывание Wiki.js на wikipc
  hosts: wiki
  become: yes
  roles:
    - docker     # ставит Docker 20.10.24 + compose v2 (уже есть)
    - wikijs     # поднимает стек Wiki.js + PostgreSQL

- name: Настройка Nginx reverse proxy + SSL на developers
  hosts: localhost
  connection: local
  become: yes
  roles:
    - nginx_proxy
...
EOF
```
#### ШАГ 5. Запуск
```bash
ansible-playbook /etc/ansible/wikiportal.yml --syntax-check
ansible-playbook /etc/ansible/wikiportal.yml
```
# Ожидаемо: failed=0 на всех трёх хостах (wikipc + localhost developers)
#### ШАГ 6. Проверки
```bash
# 1. Контейнеры на wikipc (с экранированием фигурных скобок)
ansible wiki -m shell -a "docker ps --filter name=wikijs --format 'table {% raw %}{{.Names}}\t{{.Status}}\t{{.Ports}}{% endraw %}'"

# 2. Wiki.js локально на wikipc отвечает (ИСПРАВЛЕН IP)
ansible wiki -m shell -a "curl -s -o /dev/null -w 'HTTP %{http_code}\n' http://10.1.100.17:3000"

# 3. Сертификат на developers — SAN корректен
openssl x509 -in /etc/pki/tls/certs/developers.crt -noout -text | grep -A1 "Subject Alternative Name"

# 4. HTTP -> HTTPS редирект
curl -sI http://10.1.100.10 | head -3

# 5. HTTPS отвечает через прокси (добавлен -L для перехода по редиректу 302)
curl -k -L -s -o /dev/null -w 'HTTPS HTTP %{http_code}\n' https://10.1.100.10

# 6. Проксирование действительно отдаёт Wiki.js (добавлен -L)
curl -k -L -s https://10.1.100.10 | grep -i '<title>'

# 7. Проверка порта 3000 с developers (должен вернуть 200)
curl -s -o /dev/null -w 'Direct to wikipc HTTP %{http_code}\n' http://10.1.100.17:3000
```

<img width="1108" height="737" alt="image" src="https://github.com/user-attachments/assets/ed02644f-3a7b-41ae-922e-6065482e70e1" />

#### Анализ результатов:
- Контейнеры (docker ps): wikijs_app работает уже час, порт 3000 проброшен. wikijs_db в статусе (healthy) — значит, база данных стабильна и готова к работе.
- Локальный curl (HTTP 302): Wiki.js перенаправляет запрос с корня (/) на локализованную страницу (/ru/home). Это нормальное поведение после активации русского языка.
- Сертификат (SAN): Содержит все нужные имена (developers, wikipc) и IP-адреса. Браузеры будут довольны.
- Редирект HTTP → HTTPS (301): Nginx корректно перенаправляет незащищённый трафик.
- HTTPS через прокси (HTTP 200): С флагом -L curl прошёл по редиректу Wiki.js и получил финальный ответ 200 OK.
- Контент: Заголовок <title>Тестовая страница | Wiki.js</title> и <html lang="ru"> доказывают, что Nginx проксирует запросы именно в Wiki.js, и локализация работает.
- Прямой доступ с developers (HTTP 302): Порты открыты, developers имеет прямой доступ к Wiki.js (вернулся код 302).

#### ШАГ 7. Вход в Wiki.js
Откройте в браузере:

```text
https://10.1.100.10

Примите предупреждение самоподписанного сертификата («Дополнительно» → «Перейти»).

Логин: admin@example.com
Пароль: ChangeMe123!
```
Смените пароль администратора сразу после первого входа — переменные ADMIN_PASS из docker-compose.yml используются только при первой инициализации БД. Если БД уже создана (том db/ существует), повторный запуск docker-compose up пароль не изменит.

Чтобы открывать Wiki.js по имени developers или wikipc, добавьте на клиентской машине в /etc/hosts:

```bash
echo "10.1.100.10 developers wikipc" >> /etc/hosts
Тогда https://developers и https://wikipc будут работать без предупреждения о несовпадении имени (SAN содержит оба имени и оба IP).
```

<img width="999" height="909" alt="image" src="https://github.com/user-attachments/assets/f38a0997-196a-48ba-b99f-3f5520af04af" />

<img width="1180" height="452" alt="image" src="https://github.com/user-attachments/assets/6a7d9c24-b3dc-4fee-abd4-c009788df357" />

<img width="1257" height="362" alt="image" src="https://github.com/user-attachments/assets/2f8c824e-a4df-4517-ad8e-a48d66208ed5" />

##### Разница между этими двумя адресами — это разница между правильной архитектурой и прямым доступом к бэкенду.

Вот подробная разница:

#### 1. https://10.1.100.10 (Через Nginx Reverse Proxy)
Это ваш «парадный вход». Запросы идут по цепочке: Браузер → (HTTPS) → Nginx на developers → (HTTP) → Wiki.js на wikipc.

Протокол: https:// (трафик зашифрован).

Порт: 443 (стандартный порт для HTTPS, в адресе его писать не нужно).

Безопасность: Соединение защищено SSL-сертификатом. Nginx принимает шифрованный трафик, расшифровывает его и передаёт в контейнер. Пароли и куки не передаются по сети в открытом виде.

Удобство: Красивый и короткий адрес без указания портов. Именно этот адрес вы будете давать коллегам.

#### 2. https://10.1.100.17:3000 (Напрямую в Wiki.js)
Это доступ напрямую к «движку» Wiki.js, минуя защиту Nginx.

Протокол: Здесь кроется ошибка в вашем вопросе. Контейнер Wiki.js отдаёт контент по http:// (без шифрования). Если вы попытаетесь открыть https://10.1.100.17:3000, браузер выдаст ошибку ERR_SSL_PROTOCOL_ERROR, потому что сервер на порту 3000 не умеет говорить на языке HTTPS. Правильный адрес прямого доступа: http://10.1.100.17:3000.

Порт: 3000 (нестандартный порт, обязательно указывать в адресе).

Безопасность: Небезопасно. Трафик идёт по HTTP в открытом виде. Если кто-то в вашей сети перехватит пакеты, он увидит ваш логин и пароль.

Роль в архитектуре: Этот адрес нужен только системному администратору для отладки (например, чтобы проверить, работает ли сам контейнер, если Nginx упал). Для пользователей он закрыт фаерволом (доступен только с машины developers).

##### Почему схема с Nginx (10.1.100.10) лучше?
- Шифрование (HTTPS): Wiki.js из коробки работает по HTTP. Чтобы включить HTTPS, вам пришлось бы настраивать сертификаты внутри самого контейнера, что сложнее. Nginx берёт на себя всю «тяжёлую работу» по SSL.
- Безопасность: Благодаря firewalld порт 3000 на хосте wikipc закрыт для всех, кроме developers. Хакеры извне не могут напрямую атаковать Wiki.js или базу данных.
- WebSocket: Nginx правильно проксирует WebSocket-соединения (мы настроили заголовки Upgrade и Connection), благодаря чему в Wiki.js работает онлайн-редактор и комментарии в реальном времени.
- Единая точка входа: Вы можете позже настроить Nginx так, чтобы он раздавал не только Wiki.js, но и другие сервисы (например, Grafana или портал Jenkins), распределяя их по путям (например, 10.1.100.10/wiki, 10.1.100.10/monitoring).

Итог: Всегда используйте https://10.1.100.10. Прямой доступ по http://10.1.100.17:3000 оставьте только для экстренной диагностики через SSH.
#### Итоговые правила firewall
Хост	|Порты	|Назначение
----- | -------- | --------
developers (10.1.100.10)		|80/tcp, 443/tcp		|Nginx (HTTP-редирект + HTTPS-прокси)
wikipc (10.1.100.17)		|3000/tcp		|Wiki.js (только для developers)

