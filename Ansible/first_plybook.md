#### Для базовой проверки создайте простейший файл инвентаря (список хостов), чтобы убедиться, что Ansible может управлять локальной или удаленными машинами.
##### 1. Откройте или отредактируйте стандартный файл хостов:
```bash
vi /etc/ansible/hosts
```
##### 2. Добавьте туда ваш текущий сервер (или удаленный IP), например:
```ini
[local]
localhost ansible_connection=local
```
##### 3. Выполните тестовую команду ping через Ansible:
```bash
ansible local -m ping
```

<img width="1121" height="138" alt="image" src="https://github.com/user-attachments/assets/f44516e7-2391-4a68-9a30-dba1db50c86f" />

Если всё настроено верно, Ansible вернет зеленый ответ **"ping": "pong"**

#### Давайте создадим простой сценарий автоматизации (Playbook). Он проверит, установлен ли в системе текстовый редактор vim и запущен ли веб-сервер, который я настраивал ранее (nginx), а при необходимости установит и включит их.
##### 1. Создайте файл сценария:
```bash
nano /etc/ansible/first_playbook.yml
```
##### 2. Вставьте в него следующий YAML-код:
```yaml
---
- name: Проверка и настройка базового ПО
  hosts: local
  become: yes
  tasks:

    - name: Установка последней версии vim
      yum:
        name: vim
        state: latest

    - name: Установка веб-сервера nginx
      yum:
        name: nginx
        state: present

    - name: Запуск и включение автозагрузки nginx
      service:
        name: nginx
        state: started
        enabled: yes
```
##### 3. Запустите сценарий на выполнение:
```bash
ansible-playbook /etc/ansible/first_playbook.yml
```
После выполнения вы увидите подробный отчет о проделанной работе.

<img width="1128" height="611" alt="image" src="https://github.com/user-attachments/assets/70205d14-60c5-4db6-9bb6-1ba808eefdfb" />

#### Чтобы запустить сценарий на удалённых хостах, например 10.1.100.12 и 10.1.100.13, необходимо сделать две вещи: добавить их в файл инвентаря (hosts) и настроить доступ к ним по SSH.

##### Шаг 1. Обновите файл инвентаря hosts
Вместо локального подключения создайте новую группу хостов (например, [servers]) и пропишите туда ваши IP-адреса. Выполните команду, чтобы перезаписать файл:
```bash
cat << 'EOF' > /etc/ansible/hosts
[servers]
10.1.100.12
10.1.100.13
EOF
```
##### Шаг 2. Настройте беспарольный доступ по SSH (Обязательно)
Ansible подключается к удалённым серверам по протоколу SSH. Чтобы он не запрашивал пароль при каждом запуске, настройте авторизацию по ключам:

##### 1. Сгенерируйте SSH-ключ на вашей текущей машине (если не делали этого ранее, на все вопросы нажимайте Enter):
```bash
ssh-keygen -t rsa -b 4048
```

<img width="1138" height="317" alt="image" src="https://github.com/user-attachments/assets/0d54719e-16c1-4e86-827c-d9359fde02fc" />

##### 2. Скопируйте ключ на оба удалённых хоста (система один раз попросит ввести пароль пользователя root от удалённых машин):
```bash
ssh-copy-id root@10.1.100.12
ssh-copy-id root@10.1.100.13
```

<img width="1125" height="448" alt="image" src="https://github.com/user-attachments/assets/574e2b28-0ef9-4661-b99f-78cc326f0b81" />

##### 3. Проверьте доступ с помощью модуля ping. Команда должна вернуть зеленый статус SUCCESS для обоих IP:
```bash
ansible servers -m ping
```

<img width="1021" height="255" alt="image" src="https://github.com/user-attachments/assets/9bc40c47-2784-486b-bfdc-6d9493fa50b9" />

Если всё настроено верно, Ansible вернет зеленый ответ **"ping": "pong"**.

```bash
[root@centpca ~]# cat << 'EOF' > /etc/ansible/first_playbook.yml
> ---
> - name: Подготовка репозиториев и настройка базового ПО
>   hosts: servers
>   become: yes
>   tasks:
>
>     - name: Удаление всех старых файлов репозиториев
>       file:
>         path: "{{ item }}"
>         state: absent
>       loop:
>         - /etc/yum.repos.d/CentOS-Base.repo
>         - /etc/yum.repos.d/CentOS-Vault.repo
>         - /etc/yum.repos.d/CentOS-CR.repo
>         - /etc/yum.repos.d/CentOS-Debuginfo.repo
>         - /etc/yum.repos.d/CentOS-Media.repo
>         - /etc/yum.repos.d/CentOS-Sources.repo
>         - /etc/yum.repos.d/CentOS-fasttrack.repo
>         - /etc/yum.repos.d/epel.repo
>         - /etc/yum.repos.d/epel-testing.repo
>         - /etc/yum.repos.d/myrepo.repo
>
>     - name: Создание рабочего CentOS-Base.repo (HTTP-архив Alibaba Cloud)
>       copy:
>         dest: /etc/yum.repos.d/CentOS-Base.repo
>         content: |
>           [base]
>           name=CentOS-7.9.2009 - Base
>           baseurl=http://mirrors.aliyun.com/centos-vault/7.9.2009/os/$basearch/
>           gpgcheck=0
>           enabled=1
>
>           [updates]
>           name=CentOS-7.9.2009 - Updates
>           baseurl=http://mirrors.aliyun.com/centos-vault/7.9.2009/updates/$basearch/
>           gpgcheck=0
>           enabled=1
>
>           [extras]
>           name=CentOS-7.9.2009 - Extras
>           baseurl=http://mirrors.aliyun.com/centos-vault/7.9.2009/extras/$basearch/
>           gpgcheck=0
>           enabled=1
>
>     - name: Создание репозитория EPEL (HTTP-архив Alibaba Cloud)
>       copy:
>         dest: /etc/yum.repos.d/epel.repo
>         content: |
>           [epel]
>           name=Extra Packages for Enterprise Linux 7 - $basearch
>           baseurl=http://mirrors.aliyun.com/epel-archive/7/$basearch/
>           enabled=1
>           gpgcheck=0
>
>     - name: Очистка кэша yum
>       command: yum clean all
>
>     - name: Обновление кэша yum
>       command: yum makecache
>
>     - name: Обновление криптографических библиотек для поддержки HTTPS
>       yum:
>         name:
>           - nss
>           - nss-util
>           - nss-sysinit
>           - nss-tools
>           - curl
>           - ca-certificates
>         state: latest
>
>     - name: Установка последней версии vim
>       yum:
>         name: vim
>         state: latest
>
>     - name: Установка веб-сервера nginx
>       yum:
>         name: nginx
>         state: present
>
>     - name: Запуск и включение автозагрузки nginx
>       service:
>         name: nginx
>         state: started
>         enabled: yes
> ...
> EOF
[root@centpca ~]# ansible-playbook /etc/ansible/first_playbook.yml
```

<img width="1068" height="1051" alt="image" src="https://github.com/user-attachments/assets/d21bb88d-7ec0-48b8-b206-2c85b8a8c32d" />


#### Как проверить работу Nginx?

Можно быстро убедиться, что веб-серверы отдают дефолтную страницу. Выполните команду проверки HTTP-ответа с управляющего сервера:
```bash
curl -I http://10.1.100.12
curl -I http://10.1.100.13
```
Должны увидеть в первой строке ответа ``HTTP/1.1 200 OK и строку Server: nginx/....``

<img width="1093" height="214" alt="image" src="https://github.com/user-attachments/assets/e06d0dfa-ca0b-476b-833b-ec11d0245c4c" />

Проверка первого хоста (10.1.100.12) прошла отлично — мы видим успешный ответ ``HTTP/1.1 200 OK от веб-сервера Nginx/1.26.1.``

Ошибка curl: (7) Failed connect to 10.1.100.13:80; Нет маршрута до узла (No route to host) на втором хосте указывает на то, что сетевой пакет не может дойти до веб-сервера.<br> 
Поскольку сам Ansible успешно подключается к этой машине по SSH для выполнения плейбука, сетевой уровень работает.<br>
В CentOS причиной этой ошибки почти всегда является активный встроенный межсетевой экран (Firewalld), который по умолчанию блокирует весь входящий веб-трафик на порт 80 (HTTP).<br>
Чтобы решить эту проблему, нужно открыть порт в фаерволе на удаленном хосте. Мы можем сделать это централизованно прямо через Ansible.<br>

#### Открытие портов через Ansible
##### 1. Создайте файл сценария для настройки фаервола:
```bash
cat << 'EOF' > /etc/ansible/fix_firewall.yml
---
- name: Открытие портов для веб-сервера
  hosts: servers
  become: yes
  tasks:

    - name: Разрешить входящий трафик HTTP (порт 80)
      firewalld:
        service: http
        permanent: yes
        state: enabled

    - name: Разрешить входящий трафик HTTPS (порт 443, на будущее)
      firewalld:
        service: https
        permanent: yes
        state: enabled

    - name: Перезапустить службу firewalld для применения правил
      service:
        name: firewalld
        state: restarted
...
EOF
```
##### 2. Запустите этот сценарий:
```bash
ansible-playbook /etc/ansible/fix_firewall.yml
```
##### 3. Повторная проверка
После того как Ansible применит правила межсетевого экрана, выполняем проверку еще раз:
```bash
curl -I http://10.1.100.13
```

<img width="1147" height="718" alt="image" src="https://github.com/user-attachments/assets/a6e20936-8595-46b3-a2ae-0f52b6450db6" />

#### Давайте превратим эти два пустых веб-сервера в настоящую отказоустойчивую систему

##### 1. Обновим страницы index.html на обоих серверах, чтобы они стали уникальными. Это поможет нам наглядно увидеть, как работает балансировщик.
##### 2. Настроим балансировку трафика на текущей управляющей машине (centpca), чтобы она распределяла запросы между 10.1.100.12 и 10.1.100.13.

##### Этап 1: Установка уникальных веб-страниц

Мы напишем плейбук, который автоматически сгенерирует для каждого сервера свою страницу, используя встроенную переменную Ansible с именем хоста (ansible_hostname) и его IP-адресом.
##### 1. Создайте сценарий для обновления страниц:
```bash
cat << 'EOF' > /etc/ansible/update_web.yml
---
- name: Обновление контента веб-страниц
  hosts: servers
  become: yes
  tasks:

    - name: Создание уникального index.html для каждого сервера
      copy:
        dest: /usr/share/nginx/html/index.html
        content: |
          <html>
          <body>
            <h1>Привет от Ansible!</h1>
            <h2>Сервер: {{ ansible_hostname }}</h2>
            <p>Управляемый IP-адрес: {{ inventory_hostname }}</p>
          </body>
          </html>
...
EOF
```
##### 2. Запустите его:
```bash
ansible-playbook /etc/ansible/update_web.yml
```

<img width="1071" height="241" alt="image" src="https://github.com/user-attachments/assets/12752f16-3a33-45af-92ef-ab01153a38e1" />


##### Этап 2: Настройка балансировщика трафика на centpca
Так как на вашей управляющей машине centpca уже подключен работающий репозиторий nginx, мы установим и настроим Nginx прямо на ней. Он будет выступать в роли Reverse Proxy (Load Balancer) по алгоритму Round-Robin (запросы будут уходить по очереди то на один сервер, то на другой).

##### 1. Установите Nginx на управляющей машине (centpca), если он еще не установлен (у меня установлен):
```bash
yum install nginx -y
```
##### 2. Создайте конфигурационный файл балансировщика:
Замените содержимое главного конфигурационного файла Nginx на centpca следующим блоком. Он объединит ваши серверы в пул под названием backend_servers:
```bash
cat << 'EOF' > /etc/nginx/nginx.conf
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

include /usr/share/nginx/modules/*.conf;

events {
    worker_connections 1024;
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 4096;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    # Пул удаленных серверов для распределения нагрузки
    upstream backend_servers {
        server 10.1.100.12:80 weight=1;
        server 10.1.100.13:80 weight=1;
    }

    server {
        listen       80 default_server;
        server_name  _;

        # Проксирование трафика на пул
        location / {
            proxy_pass http://backend_servers;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }
}
EOF
```
##### 3. Запустите и включите Nginx на машине centpca:
```bash
systemctl start nginx
systemctl enable nginx
```
##### 4. Откройте порт 80 на самом балансировщике centpca (у меня открыт):
```bash
firewall-cmd --permanent --add-service=http
firewall-cmd --reload
```
<img width="1074" height="108" alt="image" src="https://github.com/user-attachments/assets/2935ff3d-b5d9-4aff-bdfb-e01375e56ba4" />

> - По умолчанию система безопасности SELinux запрещает Nginx осуществлять сетевые запросы и проксировать трафик на другие IP-адреса. Фаервол порты открыл, а SELinux блокирует сам процесс изнутри.

*Разрешение проксирования в SELinux*

Выполните команду на машине centpca, чтобы разрешить Nginx подключаться к удаленным серверам:
```bash
setsebool -P httpd_can_network_connect 1
```
*Очистка дефолтных страниц Nginx*
```bash
mv /etc/nginx/conf.d/default.conf /etc/nginx/conf.d/default.conf.bak 2>/dev/null
```
*Перезапуск балансировщика*

Перезапустите веб-сервер, чтобы применить настройки:
```bash
systemctl restart nginx
```
После выполнения этих шагов локальная заглушка уйдет, и балансировщик начнет опрашивать удаленные хосты. Проверьте чередование серверов командами:
```bash
curl -s http://localhost | grep "IP-адрес:"
curl -s http://localhost | grep "IP-адрес:"
curl -s http://localhost | grep "IP-адрес:"
```

<img width="1115" height="545" alt="image" src="https://github.com/user-attachments/assets/2eafc8d5-8bc3-4f12-8e58-adb0a626994a" />

#### Проверка отказоустойчивости

Чтобы вживую увидеть, как Nginx справляется с авариями, вы можете провести простой эксперимент:
##### 1. Остановите веб-сервер на одном из хостов с помощью Ansible прямо из терминала:
```bash
ansible 10.1.100.13 -m service -a "name=nginx state=stopped" --become
```
<img width="1112" height="228" alt="image" src="https://github.com/user-attachments/assets/2a60c879-7bb8-43dc-80f7-099d5164e5bc" />

Вы заметите, что балансировщик на секунду задумается (пытаясь достучаться до упавшего сервера), поймет, что тот недоступен, и выдаст вам страницу с живого хоста 10.1.100.12. Ошибки 502 или 404 пользователь не увидит — балансировщик автоматически спасет ситуацию!

После этого сервер можно будет вернуть в строй командой:
```bash
ansible 10.1.100.13 -m service -a "name=nginx state=started" --become
```

<img width="1133" height="374" alt="image" src="https://github.com/user-attachments/assets/365ea64c-32c8-460a-903b-374c6d9df93a" />

#### Давайте реализуем продвинутую проверку доступности (Health Checks) и переведем наш плейбук на профессиональный инструмент автоматизации — динамические шаблоны Jinja2 (template).
Это позволит сделать систему по-настоящему отказоустойчивой, чтобы балансировщик мгновенно исключал упавшие серверы из пула, а файлы конфигурации генерировались автоматически.


##### Этап 1. Перевод балансировщика на Health Checks

Стандартный Nginx умеет проверять статус серверов прямо "из коробки" с помощью параметров max_fails (сколько ошибок допускается) и fail_timeout (время, на которое сервер исключается из пула).

Давайте перезапишем конфигурацию Nginx на управляющей машине centpca с этими параметрами:
```bash
cat << 'EOF' > /etc/nginx/nginx.conf
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

include /usr/share/nginx/modules/*.conf;

events {
    worker_connections 1024;
}

http {
    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    sendfile            on;
    keepalive_timeout   65;

    # Умный пул балансировки с проверкой доступности
    upstream backend_servers {
        server 10.1.100.12:80 weight=1 max_fails=2 fail_timeout=10s;
        server 10.1.100.13:80 weight=1 max_fails=2 fail_timeout=10s;
    }

    server {
        listen       80 default_server;
        server_name  _;

        location / {
            proxy_pass http://backend_servers;
            proxy_next_upstream error timeout invalid_header http_502 http_503 http_504;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
    }
}
EOF
```
Примените настройки:
```bash
systemctl restart nginx
```
теперь, если один из серверов вернет ошибку 502 или timeout хотя бы 2 раза, Nginx на 10 секунд полностью перестанет слать туда запросы, отдавая трафик только живому серверу.

##### Этап 2. Создание динамического шаблона Jinja2 в Ansible

Вместо того чтобы жестко прописывать текст веб-страницы внутри плейбука (как мы делали в модуле copy), профессионалы используют модуль template и файлы с расширением .j2. Это позволяет использовать циклы, условия и любые переменные Linux.
##### 1. Создадим красивый шаблон будущей страницы:
```bash
mkdir -p /etc/ansible/templates
cat << 'EOF' > /etc/ansible/templates/index.html.j2
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
##### 2. Создадим новый плейбук, использующий этот шаблон:
```bash
cat << 'EOF' > /etc/ansible/deploy_template.yml
---
- name: Деплой динамического веб-контента через шаблоны
  hosts: servers
  become: yes
  tasks:

    - name: Генерация index.html из шаблона Jinja2
      template:
        src: templates/index.html.j2
        dest: /usr/share/nginx/html/index.html
...
EOF
```
##### 3. Запустите деплой:
```bash
ansible-playbook /etc/ansible/deploy_template.yml
```

##### Тестирование новой конфигурации

Когда плейбук завершится, сделайте запрос к балансировщику:
```bash
curl -s http://localhost | grep -E "Имя хоста:|ОС сервера:"
```
Вы увидите, что Ansible не просто скопировал текст, а собрал с каждого сервера его индивидуальные технические характеристики (версию ОС, количество ядер) и зашил их прямо в код страницы.

<img width="1229" height="331" alt="image" src="https://github.com/user-attachments/assets/e1b9197a-709b-49a2-a5df-e7d11b5e333d" />

В выводе четко видно, что Ansible зашел на удаленный сервер, собрал его системные факты (имя хоста centpc1 и операционную систему CentOS 7.9) и на лету подставил их в HTML-структуру. 

Балансировщик Nginx принял этот трафик и корректно отдал его вам.

Вы создали полноценную, автоматизированную инфраструктуру с балансировкой нагрузки, динамическим деплоем контента и умными проверками доступности.


