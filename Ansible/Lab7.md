[root@localhost ~]# ssh-copy-id root@10.1.100.15

/usr/bin/ssh-copy-id: ERROR: failed to open ID file '/root/.pub': Нет такого файла или каталога
        (to install the contents of '/root/.pub' anyway, look at the -f option)
[root@localhost ~]# mkdir -p ~/.ssh && chmod 700 ~/.ssh
[root@localhost ~]# ssh-keygen -t rsa -b 4048
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa):
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /root/.ssh/id_rsa.
Your public key has been saved in /root/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:91nc6sSUHpa+6kfOkYKHxgLB4npQQHR42kbwIqJhou0 root@localhost.localdomain
The key's randomart image is:
+---[RSA 4048]----+
| o=+..           |
|  .o= o          |
|=..B.. .         |
|*+o.+ .      . + |
|o .+   .S..o  O..|
| .. .   ..=.oO+o |
|  E.     o .o=*. |
|             o+. |
|           .ooo  |
+----[SHA256]-----+
[root@localhost ~]# ssh-copy-id root@10.1.100.15
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/root/.ssh/id_rsa.pub"
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
root@10.1.100.15's password:

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh 'root@10.1.100.15'"
and check to make sure that only the key(s) you wanted were added.

[root@localhost ~]# ansible servers -m ping
[WARNING]: Could not match supplied host pattern, ignoring: servers
[WARNING]: No hosts matched, nothing to do
[root@localhost ~]# ansible gitlab -m ping
10.1.100.15 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false,
    "ping": "pong"
}
[root@localhost ~]#
[root@localhost ~]# mkdir -p /etc/ansible/roles/gitlab/{tasks,templates,handlers}
[root@localhost ~]#
cat <<'EOF' > /etc/ansible/roles/gitlab/tasks/main.yml
---
- name: Установка зависимостей для GitLab
  yum:
    name:
      - curl
      - policycoreutils
      - openssh-server
      - postfix
      - openssl
    state: present

- name: Включение и запуск Postfix (для отправки почты)
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

- name: Установка GitLab CE (Omnibus) - может занять несколько минут
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

- name: Генерация самоподписанного SSL-сертификата
  openssl_certificate:
    path: "/etc/gitlab/ssl/{{ ansible_default_ipv4.address }}.crt"
    privatekey_path: "/etc/gitlab/ssl/{{ ansible_default_ipv4.address }}.key"
    provider: selfsigned
    subject:
      commonName: "{{ ansible_default_ipv4.address }}"

- name: Настройка конфигурации GitLab (gitlab.rb)
  template:
    src: gitlab.rb.j2
    dest: /etc/gitlab/gitlab.rb
    owner: root
    group: root
    mode: '0600'
  notify: reconfigure gitlab

- name: Запуск и включение автозагрузки GitLab
  systemd:
    name: gitlab-runsvdir
    state: started
    enabled: yes

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
  ignore_errors: yes
...
EOF

cat <<'EOF' > /etc/ansible/roles/gitlab/templates/gitlab.rb.j2
# Внешний URL
external_url 'https://{{ ansible_default_ipv4.address }}'

# Отключаем Let's Encrypt, используем самоподписанные сертификаты
letsencrypt['enable'] = false
nginx['ssl_certificate'] = "/etc/gitlab/ssl/{{ ansible_default_ipv4.address }}.crt"
nginx['ssl_certificate_key'] = "/etc/gitlab/ssl/{{ ansible_default_ipv4.address }}.key"
nginx['redirect_http_to_https'] = true

# Отключаем ресурсоемкие компоненты мониторинга (для экономии ОЗУ)
prometheus_monitoring['enable'] = false
prometheus['enable'] = false
alertmanager['enable'] = false
grafana['enable'] = false
node_exporter['enable'] = false
redis_exporter['enable'] = false
postgres_exporter['enable'] = false
gitlab_exporter['enable'] = false

# Уменьшаем количество рабочих процессов для экономии памяти
puma['worker_processes'] = 2
EOF
