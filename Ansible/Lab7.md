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
roles/security/handlers/main.yml
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
#### ЭТАП 3. Роль gitlab (с учетом всех фиксов)
```bash
mkdir -p /etc/ansible/roles/gitlab/{tasks,templates,handlers}
```
##### 3.1. Задачи (tasks/main.yml)
Включаем генерацию SSL с SAN (через CSR), правильные зависимости и отключение проблемных параметров.
```bash
cat <<'EOF' > /etc/ansible/roles/gitlab/tasks/main.yml
---
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

- name: Генерация самоподписанного SSL-ключа
  openssl_privatekey:
    path: "/etc/gitlab/ssl/{{ ansible_default_ipv4.address }}.key"
    size: 2048
    owner: root
    group: gitlab-www
    mode: '0640'

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

- name: Запуск всех сервисов GitLab
  command: gitlab-ctl start
  args:
    creates: /var/opt/gitlab/.gitlab_started_marker

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

# Принудительный IPv4 и только TLS 1.2 (для совместимости с OpenSSL 1.0.2 на CentOS 7)
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

# Уменьшаем количество рабочих процессов
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
(Установка займет 5–15 минут. Ansible будет "молчать" на этапах скачивания пакета и выполнения reconfigure).

После завершения (failed=0) выполните команду для гарантии прав на SSL-ключ (чтобы nginx мог его прочитать):
```bash
ansible gitlab -m shell -a "chown root:gitlab-www /etc/gitlab/ssl/10.1.100.15.key && chmod 640 /etc/gitlab/ssl/10.1.100.15.key && gitlab-ctl restart nginx"
```
Вход в систему

Узнайте пароль администратора:
```bash
ansible gitlab -m shell -a "cat /etc/gitlab/initial_root_password"
```
Откройте браузер: https://10.1.100.15 (примите предупреждение о сертификате).

Логин: root, Пароль: из вывода команды.

#### ЭТАП 6. (Опционально) Настройка SSH-доступа для Git
Чтобы делать git clone без пароля, добавьте SSH-ключ вашей управляющей машины в GitLab:

Скопируйте публичный ключ: cat ~/.ssh/id_rsa.pub.

В GitLab: Аватар → Preferences → SSH Keys → Add new key.

Вставьте ключ и сохраните.

Теперь клонирование будет работать мгновенно:

```bash
git clone git@10.1.100.15:root/test-project.git
```
