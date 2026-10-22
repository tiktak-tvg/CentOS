### Пошаговое развертывание GitLab CE с нуля (Ansible)
#### Подготовка управляющей машины (10.1.100.9)
Установите Ansible, необходимые коллекции и настройте SSH-доступ.
```bash
# 1. Установка Ansible
yum install -y ansible git

# 2. Установка коллекции для работы с SELinux
ansible-galaxy collection install ansible.posix

# 3. Копирование SSH-ключа на целевую машину
ssh-keygen -t rsa -b 4096 -N "" -f ~/.ssh/id_rsa  # если ключа еще нет
ssh-copy-id root@10.1.100.15
```
Проверяем доступ:
```bash
ssh root@10.1.100.15 'hostname'
```
Проверка правил firewalld (правильный синтаксис)
```bash
ansible gitlab -m shell -a "firewall-cmd --list-all"
```
Что ищем: в секции services должны быть http и https (или в ports — 80/tcp и 443/tcp). Если их нет — откройте порты:
```bash
ansible gitlab -m shell -a "firewall-cmd --permanent --add-service=http --add-service=https && firewall-cmd --reload"
```

<img width="1091" height="294" alt="image" src="https://github.com/user-attachments/assets/e80f725f-009f-4d2a-8f0c-569b200f7f3e" />

#### ЭТАП 1. Инвентарь
Создайте файл инвентаря:
```bash
cat <<'EOF' > /etc/ansible/hosts
[gitlab]
10.1.100.15 ansible_user=root
EOF
```
Проверьте связь: ansible gitlab -m ping.

<img width="1091" height="127" alt="image" src="https://github.com/user-attachments/assets/e6d9d188-3639-4c00-94bd-bc7183743c24" />

#### Удалить системный nginx (если он был)
```bash
ansible gitlab -m shell -a "systemctl stop nginx 2>/dev/null; systemctl disable nginx 2>/dev/null; yum remove -y nginx"
```
#### ЭТАП 2. Роль security (с тегами)
```bash
mkdir -p /etc/ansible/roles/security/{tasks,handlers}
```

Создаём файл **/etc/ansible/roles/security/tasks/main.yml**
```bash
cat <<'EOF' > /etc/ansible/roles/security/tasks/main.yml
---
- name: Отключение авторизации по паролям в SSH (разрешаем только ключи)
  lineinfile:
    path: /etc/ssh/sshd_config
    regexp: '^#?PasswordAuthentication'
    line: 'PasswordAuthentication no'
  notify: Перезапуск SSH
  tags: security

- name: Отключение прямого удаленного входа для root
  lineinfile:
    path: /etc/ssh/sshd_config
    regexp: '^#?PermitRootLogin'
    line: 'PermitRootLogin without-password'
  notify: Перезапуск SSH
  tags: security
...
EOF
```

<img width="1120" height="301" alt="image" src="https://github.com/user-attachments/assets/fcfa725c-85eb-4a1c-a281-2c0e34ec8d1c" />

Создаём файл **/etc/ansible/roles/security/handlers/main.yml**
```bash
cat <<'EOF' > /etc/ansible/roles/security/handlers/main.yml
---
- name: Перезапуск SSH
  service:
    name: sshd
    state: restarted
  tags: security
...
EOF
```

<img width="1132" height="122" alt="image" src="https://github.com/user-attachments/assets/09d39b0d-5959-4bd6-bc1f-4ee8f90e3c2d" />

#### ЭТАП 3. Роль gitlab (с учетом всех фиксов)
```bash
mkdir -p /etc/ansible/roles/gitlab/{tasks,templates,handlers}
```
##### 3.1. Задачи (tasks/main.yml)
Включаем генерацию SSL с SAN (через CSR), правильные зависимости и отключение проблемных параметров.
```bash
cat <<'EOF' > /etc/ansible/roles/gitlab/tasks/main.yml
---
# --- КРИТИЧЕСКИЙ ФИКС №1: Остановка системного Nginx (освобождение портов 80/443) ---
- name: Проверка, установлен ли системный Nginx
  stat:
    path: /usr/sbin/nginx
  register: system_nginx

- name: Остановка и отключение системного Nginx (если он есть)
  systemd:
    name: nginx
    state: stopped
    enabled: no
  when: system_nginx.stat.exists
  ignore_errors: yes

# --- Установка и настройка ---
- name: Установка зависимостей для GitLab и SELinux
  yum:
    name:
      - curl
      - policycoreutils
      - policycoreutils-python
      - openssh-server
      - postfix
      - openssl
      - libselinux-python
    state: present

- name: Включение и запуск Postfix
  systemd:
    name: postfix
    state: started
    enabled: yes

- name: Отключение SELinux (перевод в permissive)
  ansible.posix.selinux:
    state: permissive
    policy: targeted
  ignore_errors: yes

- name: Добавление официального репозитория GitLab CE
  shell: curl -sS https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.rpm.sh | bash
  args:
    creates: /etc/yum.repos.d/gitlab_gitlab-ce.repo

- name: Установка GitLab CE (Omnibus)
  yum:
    name: gitlab-ce
    state: present

- name: Создание директории для SSL-сертификатов
  file:
    path: /etc/gitlab/ssl
    state: directory
    mode: '0755'

# --- КРИТИЧЕСКИЙ ФИКС №2: Генерируем ключ от root:root (группы gitlab-www еще нет) ---
- name: Генерация самоподписанного SSL-ключа (от root)
  openssl_privatekey:
    path: "/etc/gitlab/ssl/{{ ansible_default_ipv4.address }}.key"
    size: 2048

- name: Генерация CSR (с указанием SAN - IP адреса)
  openssl_csr:
    path: "/etc/gitlab/ssl/{{ ansible_default_ipv4.address }}.csr"
    privatekey_path: "/etc/gitlab/ssl/{{ ansible_default_ipv4.address }}.key"
    common_name: "{{ ansible_default_ipv4.address }}"
    subject_alt_name:
      - "IP:{{ ansible_default_ipv4.address }}"

- name: Генерация самоподписанного SSL-сертификата (с SAN)
  openssl_certificate:
    path: "/etc/gitlab/ssl/{{ ansible_default_ipv4.address }}.crt"
    csr_path: "/etc/gitlab/ssl/{{ ansible_default_ipv4.address }}.csr"
    privatekey_path: "/etc/gitlab/ssl/{{ ansible_default_ipv4.address }}.key"
    provider: selfsigned
    selfsigned_not_after: "+365d"

- name: Настройка конфигурации GitLab (gitlab.rb)
  template:
    src: gitlab.rb.j2
    dest: /etc/gitlab/gitlab.rb
    owner: root
    group: root
    mode: '0600'
  notify: reconfigure gitlab

# --- КРИТИЧЕСКИЙ ФИКС №3: Запускаем reconfigure СЕЙЧАС, чтобы создалась группа gitlab-www ---
- name: Применение конфигурации GitLab (создание группы gitlab-www)
  meta: flush_handlers

# --- Выставляем правильные права (теперь группа gitlab-www существует) ---
- name: Установка правильных прав на приватный ключ SSL
  file:
    path: "/etc/gitlab/ssl/{{ ansible_default_ipv4.address }}.key"
    owner: root
    group: gitlab-www
    mode: '0640'

- name: Установка правильных прав на сертификат SSL
  file:
    path: "/etc/gitlab/ssl/{{ ansible_default_ipv4.address }}.crt"
    owner: root
    group: gitlab-www
    mode: '0644'

- name: Перезапуск Nginx для применения прав на сертификаты
  command: gitlab-ctl restart nginx
  changed_when: true

- name: Открытие портов 80 и 443 в firewalld
  firewalld:
    service: "{{ item }}"
    permanent: yes
    state: enabled
    immediate: yes
  loop:
    - http
    - https
  when: ansible_os_family == "RedHat"
...
EOF
```
##### 3.2. Шаблон конфигурации (templates/gitlab.rb.j2)
Критически важные фиксы: нет grafana, принудительный IPv4, только TLSv1.2.
```bash
cat <<'EOF' > /etc/ansible/roles/gitlab/templates/gitlab.rb.j2
# Внешний URL
external_url 'https://{{ ansible_default_ipv4.address }}'

# Настройка HTTPS (самоподписанные сертификаты)
letsencrypt['enable'] = false
nginx['ssl_certificate'] = "/etc/gitlab/ssl/{{ ansible_default_ipv4.address }}.crt"
nginx['ssl_certificate_key'] = "/etc/gitlab/ssl/{{ ansible_default_ipv4.address }}.key"
nginx['redirect_http_to_https'] = true

# КРИТИЧЕСКИЙ ФИКС: Принудительный IPv4 и только TLSv1.2 (для CentOS 7)
nginx['listen_addresses'] = ['0.0.0.0']
nginx['ssl_protocols'] = "TLSv1.2"

# Отключаем ресурсоемкие компоненты мониторинга (экономия ОЗУ)
prometheus_monitoring['enable'] = false
prometheus['enable'] = false
alertmanager['enable'] = false
node_exporter['enable'] = false
redis_exporter['enable'] = false
postgres_exporter['enable'] = false
gitlab_exporter['enable'] = false

# Уменьшаем количество рабочих процессов (для серверов с 4ГБ ОЗУ)
puma['worker_processes'] = 2
EOF
```
##### 3.3. Обработчик (handlers/main.yml)
```bash
cat <<'EOF' > /etc/ansible/roles/gitlab/handlers/main.yml
---
- name: reconfigure gitlab
  command: gitlab-ctl reconfigure
  listen: reconfigure gitlab
...
EOF
```
#### ЭТАП 4. Главный плейбук
```bash
cat <<'EOF' > /etc/ansible/gitlab.yml
---
- name: Развертывание GitLab CE и настройка безопасности
  hosts: gitlab
  become: yes
  roles:
    - gitlab
    - security
...
EOF
```
#### ЭТАП 5. Запуск и финальная настройка
Запустите плейбук:
```bash
ansible-playbook /etc/ansible/gitlab.yml
```
#### Вход в систему
Узнайте пароль администратора:
```bash
ansible gitlab -m shell -a "cat /etc/gitlab/initial_root_password"
```
Откройте браузер: https://10.1.100.15 (примите предупреждение о сертификате).

Логин: root, Пароль: из вывода команды.

<img width="1226" height="521" alt="image" src="https://github.com/user-attachments/assets/e038a313-4f77-4a9c-bf02-81211fc53efa" />

#### ЭТАП 6. (Опционально) Настройка SSH-доступа для Git
Чтобы делать git clone без пароля, добавьте SSH-ключ вашей управляющей машины в GitLab:

Скопируйте публичный ключ: cat ~/.ssh/id_rsa.pub.

В GitLab: Аватар → Preferences → SSH Keys → Add new key.

<img width="1097" height="340" alt="image" src="https://github.com/user-attachments/assets/d7d1233d-f1c9-4e28-b872-b193eb148672" />

<img width="1337" height="1001" alt="image" src="https://github.com/user-attachments/assets/8fd6ede9-44be-4487-b2b9-356d6be48051" />

Вставьте ключ и сохраните.

#### Создайте первый проект:

Нажмите кнопку Create a project (или + в верхней панели) -> Create blank project. Дайте ему имя (например, test-project), выберите уровень 
видимости (Private) и нажмите Create project.

<img width="1168" height="717" alt="image" src="https://github.com/user-attachments/assets/05f0cae8-20af-41c0-a3c1-3e41d099166f" />

Теперь клонирование:

```bash
git clone git@10.1.100.15:root/test-project.git

# git clone git@10.1.100.15:root/test-project.git
Cloning into 'test-project'...
remote: Enumerating objects: 3, done.
remote: Counting objects: 100% (3/3), done.
remote: Compressing objects: 100% (2/2), done.
remote: Total 3 (delta 0), reused 0 (delta 0), pack-reused 0 (from 0)
Receiving objects: 100% (3/3), done.
# ls
anaconda-ks.cfg  MYDATES  test-project  
```
#### Выполните команды по порядку:
##### 1. Создайте файл правильно
```bash
cd test-project/
echo 'Hello from my Ansible-managed GitLab!' > test.txt
(Обратите внимание на одинарные кавычки '...').
```
##### 2. Убедитесь, что файл создан
```bash
ls -l test.txt
Должна появиться строка с test.txt.
# ls
anaconda-ks.cfg  MYDATES  test-project  test.txt
```
##### 3. Добавьте файл в Git и сделайте коммит
```bash
git add test.txt
git commit -m "Add first test file via CLI"
Теперь Git должен показать, что создан новый файл: 1 file changed, 1 insertion(+).
```
##### 4. Отправьте изменения на сервер
``bash
git push origin main
Должно появиться сообщение вроде To 10.1.100.15:root/test-project.git ... main -> main.
```
##### 5. Проверьте в браузере
Обновите страницу https://10.1.100.15/root/test-project — вы должны увидеть файл test.txt в списке.

<img width="1213" height="313" alt="image" src="https://github.com/user-attachments/assets/bcddfcb8-c367-49db-ac68-56bd536c13f5" />
