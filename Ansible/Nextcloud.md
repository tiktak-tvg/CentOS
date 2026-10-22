Пошаговое развертывание Nextcloud (Ansible)
- Целевая машина: 10.1.100.15 (чистый CentOS 7)
- Управляющая машина: 10.1.100.9 (Ansible)

#### Подготовка инвентаря
Добавим новый хост в инвентарь. Если вы уже добавляли 10.1.100.15 для GitLab, можно использовать ту же группу или создать новую, чтобы не было конфликтов.

Инструкция: развертывание Nextcloud с нуля

Ниже — полный, проверенный на вашем опыте плейбук. Он учитывает все проблемы, с которыми мы столкнулись:
- Конфликт с GitLab (порт 8080 занят, используем другой 8081)
- PHP 8.3
- MariaDB 10.6 (не 5.5)
- Расширения: sodium, posix, curl, intl, gd, zip, mbstring, xml, bcmath, gmp, apcu, redis6
- Права на сокет PHP-FPM (nginx:nginx)
- Права на директорию сессий
- trusted_domains
- Отключение cookie_secure и overwriteprotocol=http
- Отключение password_policy (конфликт cURL на CentOS 7)
- Memory limit 512M

#### ЭТАП 0. Подготовка управляющей машины (centpc09)
```bash
yum install -y ansible git yum-utils
ansible-galaxy collection install ansible.posix
ssh-copy-id root@10.1.100.15
ansible nextcloud -m ping
```
#### ЭТАП 1. Инвентарь
```bash
cat <<'EOF' > /etc/ansible/hosts
[nextcloud]
10.1.100.15 ansible_user=root
EOF
```
#### ЭТАП 2. Структура роли
```bash
mkdir -p /etc/ansible/roles/nextcloud/{tasks,templates,handlers}
```
#### ЭТАП 3. Финальный tasks/main.yml
```bash
cat <<'EOF' > /etc/ansible/roles/nextcloud/tasks/main.yml
---
# ============ 1. Репозитории ============
- name: Установка EPEL и Remi репозиториев
  yum:
    name:
      - epel-release
      - yum-utils
      - http://rpms.remirepo.net/enterprise/remi-release-7.rpm
    state: present

- name: Включение модуля remi-php83
  shell: yum-config-manager --enable remi-php83
  changed_when: false

# ============ 2. MariaDB 10.6 из архива (yum.mariadb.org не работает) ============
- name: Удаление старой MariaDB 5.5 (если есть)
  yum:
    name:
      - mariadb
      - mariadb-server
      - mariadb-libs
    state: absent
  ignore_errors: yes

- name: Репозиторий MariaDB 10.6 (архив)
  copy:
    dest: /etc/yum.repos.d/mariadb-archive.repo
    content: |
      [mariadb-archive]
      name = MariaDB 10.6 Archive
      baseurl = https://archive.mariadb.org/mariadb-10.6/yum/centos/7/x86_64/
      gpgkey = https://yum.mariadb.org/RPM-GPG-KEY-MariaDB
      gpgcheck = 1
    mode: '0644'

# ============ 3. Пакеты ============
- name: Установка Nginx, MariaDB 10.6, Redis, PHP 8.3 и всех расширений
  yum:
    name:
      - nginx
      - MariaDB-server
      - MariaDB-client
      - redis
      - MySQL-python
      - php83-php-fpm
      - php83-php-cli
      - php83-php-pdo
      - php83-php-mysqlnd
      - php83-php-xml
      - php83-php-mbstring
      - php83-php-json
      - php83-php-curl
      - php83-php-zip
      - php83-php-gd
      - php83-php-intl
      - php83-php-bcmath
      - php83-php-gmp
      - php83-php-posix
      - php83-php-process
      - php83-php-sodium
      - php83-php-pecl-apcu
      - php83-php-pecl-redis6
    state: present

# ============ 4. Директории сокета и сессий ============
- name: Директория для сокета PHP-FPM
  file:
    path: /var/run/php-fpm
    state: directory
    owner: nginx
    group: nginx
    mode: '0755'

- name: Директория для PHP-сессий
  file:
    path: /var/opt/remi/php83/lib/php/session
    state: directory
    owner: nginx
    group: nginx
    mode: '0700'

- name: tmpfiles.d для автосоздания директорий после перезагрузки
  copy:
    dest: /etc/tmpfiles.d/nextcloud.conf
    content: |
      d /var/run/php-fpm 0755 nginx nginx -
      d /var/opt/remi/php83/lib/php/session 0700 nginx nginx -
    mode: '0644'

# ============ 5. PHP memory limit ============
- name: Увеличение memory_limit до 512M
  lineinfile:
    path: /etc/opt/remi/php83/php.ini
    regexp: '^memory_limit ='
    line: 'memory_limit = 512M'

# ============ 6. Запуск сервисов ============
- name: Запуск MariaDB
  systemd:
    name: mariadb
    state: started
    enabled: yes

- name: Инициализация MariaDB (mysql_install_db если нужно)
  shell: |
    if [ ! -d /var/lib/mysql/mysql ]; then
      mysql_install_db --user=mysql --datadir=/var/lib/mysql
      systemctl restart mariadb
    fi
  changed_when: false

- name: Запуск Redis и PHP-FPM 8.3
  systemd:
    name: "{{ item }}"
    state: started
    enabled: yes
  loop:
    - redis
    - php83-php-fpm

# ============ 7. SELinux ============
- name: SELinux booleans
  seboolean:
    name: "{{ item }}"
    state: yes
    persistent: yes
  loop:
    - httpd_can_network_connect
    - httpd_can_network_connect_db
    - httpd_graceful_shutdown

- name: SELinux контекст для Nextcloud
  sefcontext:
    target: '/var/www/nextcloud(/.*)?'
    setype: httpd_sys_rw_content_t
    state: present

# ============ 8. База данных ============
- name: Создание БД nextcloud (utf8mb4)
  mysql_db:
    name: nextcloud
    state: present
    encoding: utf8mb4
    collation: utf8mb4_general_ci
    login_unix_socket: /var/lib/mysql/mysql.sock

- name: Создание пользователя nextcloud
  mysql_user:
    name: nextcloud
    password: "StrongNextcloudPassword123!"
    priv: 'nextcloud.*:ALL'
    state: present
    host: localhost
    login_unix_socket: /var/lib/mysql/mysql.sock

# ============ 9. Распаковка Nextcloud ============
- name: Создание /var/www
  file:
    path: /var/www
    state: directory
    mode: '0755'

- name: Скачивание Nextcloud (с таймаутом 15 мин)
  get_url:
    url: https://download.nextcloud.com/server/releases/latest.zip
    dest: /tmp/nextcloud.zip
    mode: '0644'
    timeout: 900

- name: Распаковка Nextcloud
  unarchive:
    src: /tmp/nextcloud.zip
    dest: /var/www
    remote_src: yes
    creates: /var/www/nextcloud

- name: Права на директорию Nextcloud
  file:
    path: /var/www/nextcloud
    state: directory
    owner: nginx
    group: nginx
    mode: '0755'
    recurse: yes

- name: SELinux контекст на файлы Nextcloud
  command: restorecon -Rv /var/www/nextcloud
  changed_when: false

# ============ 10. PHP-FPM: сокет с правильными правами ============
- name: Настройка PHP-FPM 8.3
  lineinfile:
    path: /etc/opt/remi/php83/php-fpm.d/www.conf
    regexp: "{{ item.regexp }}"
    line: "{{ item.line }}"
  loop:
    - { regexp: '^user = apache', line: 'user = nginx' }
    - { regexp: '^group = apache', line: 'group = nginx' }
    - { regexp: '^listen = /run/php-fpm/www.sock', line: 'listen = /var/run/php-fpm/php83-fpm.sock' }
    - { regexp: '^;?listen.owner', line: 'listen.owner = nginx' }
    - { regexp: '^;?listen.group', line: 'listen.group = nginx' }
    - { regexp: '^;?listen.mode', line: 'listen.mode = 0660' }
  notify: restart php83-fpm

# ============ 11. Nginx на порту 8081 ============
- name: Удаление default.conf
  file:
    path: /etc/nginx/conf.d/default.conf
    state: absent

- name: Конфиг Nginx для Nextcloud (8081)
  template:
    src: nextcloud.conf.j2
    dest: /etc/nginx/conf.d/nextcloud.conf
    mode: '0644'
  notify: restart nginx

- name: Запуск Nginx
  systemd:
    name: nginx
    state: started
    enabled: yes

# ============ 12. Firewalld ============
- name: Открытие порта 8081
  firewalld:
    port: 8081/tcp
    permanent: yes
    state: enabled
    immediate: yes
  when: ansible_os_family == "RedHat"
...
EOF
```
#### ЭТАП 4. Шаблон Nginx
```bash
cat <<'EOF' > /etc/ansible/roles/nextcloud/templates/nextcloud.conf.j2
upstream php-handler {
    server unix:/var/run/php-fpm/php83-fpm.sock;
}

server {
    listen 8081;
    server_name {{ ansible_default_ipv4.address }};

    root /var/www/nextcloud;
    index index.php index.html /index.php$request_uri;

    client_max_body_size 512M;
    fastcgi_buffers 64 4K;
    fastcgi_hide_header X-Powered-By;

    location = /robots.txt { allow all; log_not_found off; access_log off; }

    location ~ ^/(?:index|remote|public|cron|core/ajax/update|status|ocs/v[12]|updater/.+|ocs-provider/.+)\.php(?:$|/) {
        fastcgi_split_path_info ^(.+\.php)(/.*)$;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
        fastcgi_param HTTPS off;
        fastcgi_param modHeadersAvailable true;
        fastcgi_param front_controller_active true;
        fastcgi_pass php-handler;
        fastcgi_intercept_errors on;
        fastcgi_request_buffering off;
    }

    location ~ \.php(?:$|/) {
        fastcgi_split_path_info ^(.+\.php)(/.*)$;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
        fastcgi_param HTTPS off;
        fastcgi_param modHeadersAvailable true;
        fastcgi_param front_controller_active true;
        fastcgi_pass php-handler;
        fastcgi_intercept_errors on;
        fastcgi_request_buffering off;
    }

    location ~ \.(?:css|js|svg|gif|png|jpg|ico|wasm|tflite|map)$ {
        try_files $uri /index.php$request_uri;
        expires 6M;
        access_log off;
    }

    location ~ \.woff2?$ {
        try_files $uri /index.php$request_uri;
        expires 7d;
        access_log off;
    }

    location / {
        try_files $uri $uri/ /index.php$request_uri;
    }

    location ~ ^/(?:\.htaccess|data|config|db_structure\.xml|README) {
        deny all;
    }
}
EOF
```
#### ЭТАП 5. Handlers
```bash
cat <<'EOF' > /etc/ansible/roles/nextcloud/handlers/main.yml
---
- name: restart php83-fpm
  service:
    name: php83-php-fpm
    state: restarted

- name: restart nginx
  service:
    name: nginx
    state: restarted
...
EOF
```
#### ЭТАП 6. Плейбук nextcloud.yml
```bash
cat <<'EOF' > /etc/ansible/nextcloud.yml
---
- name: Развертывание Nextcloud (LEMP + PHP 8.3 + MariaDB 10.6)
  hosts: nextcloud
  become: yes
  roles:
    - nextcloud
...
EOF
```
#### ЭТАП 7. Запуск
```bash
ansible-playbook /etc/ansible/nextcloud.yml
```
Установка займёт 10–15 минут (скачивание ~150 МБ + установка пакетов).

Ожидаемый результат: PLAY RECAP с failed=0.

#### ЭТАП 8. Пост-установка через браузер
Откройте http://10.1.100.15:8081 в режиме инкогнито.
- Заполните форму:
- Имя администратора: nextcloud_admin
- Пароль администратора: MySuperCloud2026!
- Пользователь БД: nextcloud
- Пароль БД: StrongNextcloudPassword123!
- Имя БД: nextcloud
- Хост БД: localhost
- Нажмите «Установить».

#### ЭТАП 9. Обязательные фиксы после установки (иначе вход не будет работать)
После того как установка завершилась (появилась страница входа), выполните на centpc09:
```bash
# 1. Отключить password_policy (конфликт с cURL на CentOS 7)
ansible nextcloud -m shell -a "sudo -u nginx /opt/remi/php83/root/usr/bin/php /var/www/nextcloud/occ app:disable password_policy"

# 2. Установить протокол HTTP и отключить secure cookie
ansible nextcloud -m shell -a "sudo -u nginx /opt/remi/php83/root/usr/bin/php /var/www/nextcloud/occ config:system:set overwriteprotocol --value='http'"
ansible nextcloud -m shell -a "sudo -u nginx /opt/remi/php83/root/usr/bin/php /var/www/nextcloud/occ config:system:set cookie_secure --value='false' --type=boolean"

# 3. Перезапустить PHP-FPM
ansible nextcloud -m shell -a "systemctl restart php83-php-fpm"
```
#### ЭТАП 10. Вход
В режиме инкогнито откройте http://10.1.100.15:8081/login.

- Логин: nextcloud_admin
- Пароль: MySuperCloud2026!

Нажмите «Войти» → попадёте в файловый менеджер.

#### Итоговая архитектура (без конфликтов)
Сервис	| Порт	| Где
------- | ----- | ------------------------
GitLab (Omnibus)	 | 80, 443	 | 10.1.100.15
Nextcloud (Nginx + PHP 8.3 + MariaDB 10.6 + Redis)	 | 8081	 | 10.1.100.15

Оба сервиса работают параллельно и не мешают друг другу.

Ключевые "таблетки от грабель" в этом плейбуке

Проблема	 | Решение
---------- | ------------
PHP 7.4 → Nextcloud  | требует 8.3	remi-php83 + пакеты php83-*
MariaDB 5.5 → ошибка 1071 Index too long	 | MariaDB 10.6 из архива
yum.mariadb.org 404	 | archive.mariadb.org
posix_getuid() undefined	 | php83-php-posix + php83-php-process
PASSWORD_ARGON2ID undefined	 | php83-php-sodium
CURL_HTTP_VERSION_2TLS undefined	 | php83-php-curl + app:disable password_policy
session_start(): Permission denied	 | Директория сессий nginx:nginx 700
Host violates local access rules	 | trusted_domains через occ
Cookie secure блокирует вход по HTTP	 | overwriteprotocol=http + cookie_secure=false
Сокет root:root недоступен Nginx	 | listen.owner = nginx в www.conf
Порт 8080 занят GitLab Puma	 | Nginx на 8081

Полный обзор машины 10.1.100.15 (centpc15)

На этой машине работают два независимых сервиса, которые мы развернули: GitLab CE и Nextcloud. Они не конфликтуют, потому что используют разные веб-серверы, разные БД и разные порты.

##### Что работает на 10.1.100.15
Сервис	| Порт	| Веб-сервер	| БД	| Статус
---------- | ------------ | ------ | -------- | ---------
GitLab CE		| 443 (HTTPS)	| Omnibus Nginx (/opt/gitlab/embedded/sbin/nginx)	PostgreSQL (встроенная)		| Работает
GitLab CE		| 80 (HTTP)		| Omnibus Nginx (редирект на 443)		| —		| Работает
Nextcloud		| 8081 (HTTP)		| Системный Nginx (/usr/sbin/nginx)		| MariaDB 10.6		| Работает
MariaDB		| 3306 (только localhost)	| 	—		| БД nextcloud		| Работает
Redis		| 6379 (только localhost)		| —		| Кэш для Nextcloud		| Работает
PHP-FPM 8.3	Unix-сокет /var/run/php-fpm/php83-fpm.sock	| —	| — | Работает

##### Порты и адреса
```txt
┌──────────────────────────────────────────────────────────────┐
│  10.1.100.15 (centpc15)                                      │
│                                                              │
│  :80   ─┐                                                    │
│         ├─→ GitLab Omnibus Nginx ─→ Puma/Workhorse ─→ GitLab │
│  :443  ─┘   (SSL, самоподписанный)                           │
│                                                              │
│  :8081 ───→ Системный Nginx ─→ PHP-FPM 8.3 ─→ Nextcloud      │
│                                              │               │
│                                              ├─→ MariaDB 10.6│
│                                              │   (localhost) │
│                                              └─→ Redis       │
│                                                  (localhost) │
└──────────────────────────────────────────────────────────────┘
```
##### Доступ извне:

- GitLab: https://10.1.100.15 (с самоподписанным сертификатом)
- Nextcloud: http://10.1.100.15:8081 (HTTP, без шифрования — для тестов)

##### Файловая структура
**GitLab (Omnibus)**
Путь		| Назначение
---------- | ------------
/etc/gitlab/gitlab.rb		| Главный конфиг GitLab
/etc/gitlab/ssl/		| SSL-сертификаты (10.1.100.15.crt, .key)
/var/opt/gitlab/		| Данные GitLab (репозитории, БД, логи)
/var/log/gitlab/		| Логи всех компонентов GitLab
/opt/gitlab/		| Бинарники GitLab Omnibus
/etc/gitlab/initial_root_password		| Пароль root (удаляется через 24 часа)

**Nextcloud**
Путь		| Назначение
---------- | ------------
/var/www/nextcloud/		| Файлы Nextcloud (ядро, приложения, конфиги)
/var/www/nextcloud/config/config.php		| Главный конфиг Nextcloud
/var/www/nextcloud/data/		| Данные пользователей (файлы, логи)
/etc/nginx/conf.d/nextcloud.conf		| Конфиг Nginx для Nextcloud
/etc/opt/remi/php83/php-fpm.d/www.conf		| Конфиг PHP-FPM
/etc/opt/remi/php83/php.ini		| Конфиг PHP (memory_limit 512M)
/var/log/nginx/		| Логи системного Nginx
/var/opt/remi/php83/lib/php/session/		| PHP-сессии

**MariaDB (для Nextcloud)**
Путь		| Назначение
---------- | ------------
/var/lib/mysql/		| Данные всех БД
/etc/my.cnf.d/		| Настройки MariaDB
/var/log/mariadb/		| Логи MariaDB

##### Пользователи и группы (системные)
Пользователь		| Группа		| Для чего
---------- | ------------ | ----------
git		| git		| GitLab: доступ к репозиториям
gitlab-www		| gitlab-www		| GitLab: Nginx worker
gitlab-psql		| gitlab-psql		| GitLab: PostgreSQL
gitlab-redis	| gitlab-redis		| GitLab: Redis
nginx	nginx		| Системный Nginx (Nextcloud)
mysql		      | mysql		| MariaDB
redis		      | redis		| Redis (системный)
apache		    | apache		| PHP-FPM pool (по умолчанию)

##### Базы данных
БД		| Владелец		| Где		| Для чего
---------- | ------------ | ------- | --------
gitlabhq_production		| gitlab		| PostgreSQL (встроенная в GitLab)		| GitLab
nextcloud		| nextcloud		| MariaDB 10.6		| Nextcloud

> Важно: GitLab использует PostgreSQL, а Nextcloud — MariaDB. Они не пересекаются.

##### Учётные записи (для входа)
**GitLab**
- URL: https://10.1.100.15
- Логин: root
- Пароль: был в /etc/gitlab/initial_root_password (или тот, что вы сменили после первого входа)

**Nextcloud**
- URL: http://10.1.100.15:8081
- Логин: nextcloud_admin
- Пароль: MySuperCloud2026!
- БД (для конфига): пользователь nextcloud, пароль StrongNextcloudPassword123!

##### Управление сервисами
**GitLab**
```bash
gitlab-ctl status              # Статус всех компонентов
gitlab-ctl restart             # Перезапуск
gitlab-ctl reconfigure         # Применить изменения из gitlab.rb
gitlab-ctl stop                # Остановить
gitlab-ctl tail nginx          # Логи nginx в реальном времени
```
Компоненты GitLab (все через runit): gitaly, gitlab-kas, gitlab-workhorse, logrotate, nginx, postgresql, puma, redis, sidekiq.

**Nextcloud**
```bash
systemctl status nginx             # Системный Nginx
systemctl status php83-php-fpm     # PHP-FPM 8.3
systemctl status mariadb           # MariaDB 10.6
systemctl status redis             # Redis

systemctl restart nginx            # Перезапуск Nginx
systemctl restart php83-php-fpm    # Перезапуск PHP-FPM
```
##### Логи
**GitLab**

Лог		| Путь
---------- | ------------
Nginx (GitLab)		| /var/log/gitlab/nginx/error.log
Puma		| /var/log/gitlab/puma/
Sidekiq		| /var/log/gitlab/sidekiq/
Gitaly		| /var/log/gitlab/gitaly/
PostgreSQL		| /var/log/gitlab/postgresql/

**Nextcloud**
Лог		| Путь
---------- | ------------
Nginx (системный)		| /var/log/nginx/error.log
Nextcloud		| /var/www/nextcloud/data/nextcloud.log
PHP-FPM		| /var/opt/remi/php83/var/log/php-fpm/
MariaDB	| /var/log/mariadb/mariadb.log

##### Firewalld (открытые порты)
```bash
firewall-cmd --list-all
Порт/Сервис	Для чего
ssh (22)	SSH-доступ
http (80)	GitLab (редирект на HTTPS)
https (443)	GitLab (HTTPS)
8081/tcp	Nextcloud (HTTP)
```
##### Быстрая проверка «всё ли живо»
```bash
# 1. Порты
ss -tlnp | grep -E ':(80|443|8081|3306|6379)'

# 2. GitLab
curl -k -I https://10.1.100.15 | head -3
# Ожидаемо: HTTP/1.1 302 Found → /users/sign_in

# 3. Nextcloud
curl -I http://10.1.100.15:8081 | head -3
# Ожидаемо: HTTP/1.1 200 OK (или 302)

# 4. MariaDB
mysql -e "SELECT user,host FROM mysql.user WHERE user='nextcloud';"

# 5. GitLab статус
gitlab-ctl status

# 6. Nextcloud БД через occ
sudo -u nginx /opt/remi/php83/root/usr/bin/php /var/www/nextcloud/occ status
```
##### Возможные конфликты (и почему их нет)
Что могло бы конфликтовать	Почему не конфликтует
- Порт 80/443 — GitLab и Nginx Nextcloud	Nextcloud на 8081, GitLab на 80/443
- Nginx — два экземпляра	GitLab использует /opt/gitlab/embedded/sbin/nginx, Nextcloud — /usr/sbin/nginx
- PHP-FPM — два пула	Только один пул (PHP 8.3), обслуживает Nextcloud
- База данных — две БД	GitLab → PostgreSQL (встроенный), Nextcloud → MariaDB 10.6
- Пользователи — nginx vs gitlab-www	Разные пользователи, разные файлы, разные группы
##### Резюме по 10.1.100.15
Что	Где	Как зайти
- GitLab (Git-сервер)	https://10.1.100.15	Логин root, пароль из /etc/gitlab/initial_root_password
- Nextcloud (Файловое облако)	http://10.1.100.15:8081	Логин nextcloud_admin, пароль MySuperCloud2026!
- БД Nextcloud	MariaDB, nextcloud	пользователь nextcloud / StrongNextcloudPassword123!
- База GitLab	PostgreSQL (встроенная)	Управляется автоматически через gitlab-ctl
- Оба сервиса полностью независимы и работают параллельно. Машина 10.1.100.15 успешно выполняет роль двух «серверов» в одном — Git-сервера и файлового облака.

