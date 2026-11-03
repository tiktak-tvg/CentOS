### Пошаговое руководство по развёртыванию лёгкого Kubernetes-кластера (K3s) на трёх и более машинах с помощью официальной Ansible-коллекции k3s-io/k3s-ansible. 
Это production-подход: control-plane в HA-режиме (3 сервера с встроенным etcd), воркеры-агенты, автоматическая подготовка узлов и проверка отказоустойчивости.

На всех хостах: 
```bash
cat /etc/os-release
NAME="CentOS Linux"
VERSION="7 (Core)"
ID="centos"
ID_LIKE="rhel fedora"
VERSION_ID="7"
PRETTY_NAME="CentOS Linux 7 (Core)"
ANSI_COLOR="0;31"
CPE_NAME="cpe:/o:centos:centos:7"
HOME_URL="https://www.centos.org/"
BUG_REPORT_URL="https://bugs.centos.org/"

CENTOS_MANTISBT_PROJECT="CentOS-7"
CENTOS_MANTISBT_PROJECT_VERSION="7"
REDHAT_SUPPORT_PRODUCT="centos"
REDHAT_SUPPORT_PRODUCT_VERSION="7"

uname -r
3.10.0-1160.119.1.el7.x86_64
cat /etc/centos-release
CentOS Linux release 7.9.2009 (Core)
```
Все команды выполняются на управляющем хосте hostname:managerK3s (ip:10.1.100.10) под root.

На нём установлены Ansible, nginx, firewalld

Хосты:
```bash
10.1.100.28
[root@managerk3s ~]# ssh root@10.1.100.24 'hostname && hostname -I'
k3s-server-24
10.1.100.24
[root@managerk3s ~]# ssh root@10.1.100.25 'hostname && hostname -I'
k3s-server-25
10.1.100.25
[root@managerk3s ~]# ssh root@10.1.100.26 'hostname && hostname -I'
k3s-server-26
10.1.100.26
[root@managerk3s ~]# ssh root@10.1.100.27 'hostname && hostname -I'
k3s-agent-27
10.1.100.27
[root@managerk3s ~]# ssh root@10.1.100.28 'hostname && hostname -I'
k3s-agent-28
10.1.100.28
[root@managerk3s ~]#

На них установлен firewalld

[root@managerk3s ~]# ssh root@10.1.100.28 'uname -r'
3.10.0-1160.119.1.el7.x86_64
[root@managerk3s ~]# ssh root@10.1.100.24 'uname -r'
3.10.0-1160.119.1.el7.x86_64
[root@managerk3s ~]# ssh root@10.1.100.25 'uname -r'
3.10.0-1160.119.1.el7.x86_64
[root@managerk3s ~]# ssh root@10.1.100.26 'uname -r'
3.10.0-1160.119.1.el7.x86_64
[root@managerk3s ~]# ssh root@10.1.100.27 'uname -r'
3.10.0-1160.119.1.el7.x86_64
```
Схема кластера (пример)
```text
        ┌─────────────────────────────────────┐
        │  managerK3s  (10.1.100.10)          │
        │  Ansible control node / kubectl     │
        └─────────────────┬───────────────────┘
                          │ SSH
        ┌─────────────────┼───────────────────┐
        │                 │                   │
   ┌────┴────┐       ┌────┴────┐         ┌────┴────┐
   │ .24     │       │ .25     │         │ .26     │
   │ server  │       │ server  │         │ server  │
   │ (etcd)  │       │ (etcd)  │         │ (etcd)  │
   └────┬────┘       └────┬────┘         └────┬────┘
        │                 │                   │
   ┌────┴────┐       ┌────┴────┐         ┌────┴────┐
   │ .27     │       │ .28     │         │ ...     │
   │ agent   │       │ agent   │         │ agent   │
   └─────────┘       └─────────┘         └─────────┘
```
> [!Warning]
> для HA-режима с встроенным etcd количество серверов должно быть нечётным (3, 5, 7).

Если третьего сервера нет, можно развернуть минимальный кластер: 1 server + 2 agent (без HA control-plane). В примере ниже показан HA-вариант (3 сервера).

### Шаг 1: Предустановка Ansible 2.15.2 + netaddr + ansible.utils
Этот шаг выполняется только на управляющем хосте managerK3s (10.1.100.10) от root. Узлы кластера на этом этапе не трогаем.
#### 1.1. Сборка Python 3.9.18 из исходников
```bash
yum groupinstall -y "Development Tools"
yum install -y openssl-devel bzip2-devel libffi-devel zlib-devel wget

cd /usr/src
wget https://www.python.org/ftp/python/3.9.18/Python-3.9.18.tgz
tar xzf Python-3.9.18.tgz
cd Python-3.9.18
./configure --enable-optimizations --prefix=/opt/python3.9
make -j $(nproc)
make altinstall
```
<img width="1072" height="828" alt="image" src="https://github.com/user-attachments/assets/29171343-d73d-4727-b6d8-aaf4129ac0fb" />

Проверка:
```bash
/opt/python3.9/bin/python3.9 --version
```
<img width="1073" height="198" alt="image" src="https://github.com/user-attachments/assets/7b5cd189-88a2-4a89-b0d6-baa7fd8a5d69" />

> [!Warning]
> **Важно:** используется именно make altinstall, а не make install — чтобы не перезаписать /usr/bin/python и не сломать системные yum/firewalld, которые завязаны на Python 2.7.<br>
> На CentOS 7 через yum install ansible ставится Ansible 2.9.27 на Python 2.7, несовместимый с коллекцией k3s-ansible. Он остаётся в /usr/bin/ansible и /usr/lib/python2.7/site-packages/ansible.<br>
> Удалять его не нужно — достаточно работать в активированном venv, где /root/ansible-venv/bin/ansible переопределяет системный.

#### 1.2. Создание виртуального окружения
```bash
# Убедитесь, что вы НЕ в активированном venv
# (приглашение без "(ansible-venv)"). Если он активен — deactivate.

/opt/python3.9/bin/python3.9 -m venv ~/ansible-venv
source ~/ansible-venv/bin/activate
```
После активации приглашение должно измениться:
```ext
(ansible-venv) [root@managerK3s ~]#
```
Проверка (обязательна!):
```bash
which python
# Ожидается: /root/ansible-venv/bin/python
python --version
# Ожидается: Python 3.9.18 (НЕ 3.8.x!)
```
<img width="1077" height="96" alt="image" src="https://github.com/user-attachments/assets/0c65f2f8-ddea-4e21-a75a-b9f20d1c5ffd" />

> [!Warning]
> Если python --version показывает 3.8.13 — значит, venv создан не на том Python.<br>
> **Причина:** — активированный SCL (rh-python38) переопределил python.<br>
> **Решение:** deactivate, rm -rf ~/ansible-venv, создать заново строго из /opt/python3.9/bin/python3.9.

> [!TIP]
> **Типичная ошибка:** если python --version показывает 3.8.13 — значит venv был создан не на /opt/python3.9/bin/python3.9, а на другом Python. <br>В этом случае удалите ~/ansible-venv и создайте заново из точного пути.

#### 1.3. Установка ansible-core 2.15.2 + netaddr и зависимостей
```bash
# Обновление pip
pip install --upgrade pip

# Ключевые пакеты
pip install ansible-core==2.15.2 netaddr
```
<img width="1078" height="768" alt="image" src="https://github.com/user-attachments/assets/45a52c46-105c-42b4-ba12-9c6f17147819" />

> [!TIP]
> **Почему ansible-core, а не ansible?** <br>
> **ansible-core** — это «движок» Ansible (CLI, модули, плагины).<br>
> **Пакет ansible** — это мета-пакет, включающий ansible-core + огромный набор коллекций, которые нам не нужны (и которые могут конфликтовать).<br>
> **Официальный способ запускать k3s-ansible** — через ansible-core, а нужные коллекции ставим отдельно.

#### Проверка:
```bash
pip list | grep -E 'ansible-core|netaddr'
# Ожидается примерно:
#   ansible-core   2.15.2
#   netaddr        1.3.0
```
<img width="1075" height="66" alt="image" src="https://github.com/user-attachments/assets/2f5ce076-27e1-421a-b7ba-aae04dd8611f" />

> [!TIP]
> **Зачем netaddr?** <br>
> Фильтр ansible.utils.ipwrap (используется в k3s-ansible для обработки IPv6-адресов) написан на Python и требует библиотеку netaddr.<br>
> Без неё при запуске playbook'а вы получите Could not load "ansible.utils.ipwrap" или Unrecognized type AnsibleUndefined.

#### 1.4. Установка служебных коллекций (фиксированные версии)
Помимо k3s-ansible нужны три служебные коллекции:
```bash
# 1. ansible.utils — содержит плагин ipwrap (требуется k3s-ansible)
ansible-galaxy collection install ansible.utils --force

# 2. ansible.posix — фиксируем версию 1.5.4, совместимую с ansible-core 2.15
ansible-galaxy collection install 'ansible.posix:==1.5.4' --force

# 3. community.general — фиксируем 11.4.5, последнюю версию с поддержкой Python 2.7 на узлах
ansible-galaxy collection install 'community.general:==11.4.5' --force
```
> [!TIP]
> **Почему именно так?**
> **ansible.utils 6.1.1** — фильтр ipwrap, без него k3s-ansible падает.
> **ansible.posix 1.5.4** — совместим с ansible-core 2.15 без warning'ов.
> **community.general 11.4.5** — последняя версия с поддержкой Python 2.7 (системный на CentOS 7). Версия 12.x падает с SyntaxError в modprobe.py на узлах.

<img width="1075" height="687" alt="image" src="https://github.com/user-attachments/assets/030ce264-865f-41f2-a73c-fd93bd42fbf3" />

#### 1.5. Финальная проверка предустановки
```bash
# Версия Ansible
ansible --version

# Все пути ведут в venv. Все пять путей должны начинаться с /root/ansible-venv/bin/:
which -a ansible ansible-playbook ansible-galaxy python pip
# Если хоть один указывает на /usr/bin/ — venv не активирован, выполните source ~/ansible-venv/bin/activate заново.

# Python-модули. Проверка netaddr через Python venv
python -c "import netaddr; print('netaddr:', netaddr.__version__)"
python -c "import yaml; print('yaml:', yaml.__version__)"

# Проверка установленных коллекций:
ansible-galaxy collection list | grep -E 'ansible.utils|ansible.posix|community.general'

# Если в выводе фигурирует /usr/bin/ansible или python version = 2.7.5 — venv не активирован или создан не на том Python.
```

<img width="1077" height="442" alt="image" src="https://github.com/user-attachments/assets/09c28e00-67e4-49ca-90e4-e9f8ac5ddc1c" />

##### Признаки корректной установки:
- **ansible [core 2.15.2]** — версия правильная
- **executable location: /root/ansible-venv/bin/ansible** — берётся из venv
- **python version = 3.9.18** — Python правильный
- **ansible python module location** — внутри venv

#### 1.6. Фиксация окружения (рекомендуется)
Сохраните список пакетов для воспроизводимости:
```bash
mkdir -p /root/k3s-ha
pip freeze > /root/k3s-ha/requirements-pip.txt
ansible-galaxy collection list > /root/k3s-ha/requirements-collections.txt
```
При пересоздании venv в будущем:
```bash
pip install -r /root/k3s-ha/requirements-pip.txt
ansible-galaxy collection install -r /root/k3s-ha/requirements-collections.txt
```
#### Шаг 1: Python 3.9 на управляющем хосте (managerK3s)
ansible-core 2.15.2 требует Python 3.9+. В репозиториях CentOS 7 его нет, поэтому собираем из исходников в отдельный префикс.
```bash
# Зависимости для сборки
yum groupinstall -y "Development Tools"
yum install -y openssl-devel bzip2-devel libffi-devel zlib-devel wget

# Сборка Python 3.9.18
cd /usr/src
wget https://www.python.org/ftp/python/3.9.18/Python-3.9.18.tgz
tar xzf Python-3.9.18.tgz
cd Python-3.9.18
./configure --enable-optimizations --prefix=/opt/python3.9
make -j $(nproc)
make altinstall
```
Создаём venv и устанавливаем Ansible:
```bash
/opt/python3.9/bin/python3.9 -m venv ~/ansible-venv
source ~/ansible-venv/bin/activate
pip install --upgrade pip
pip install ansible-core==2.15.2 netaddr ansible.utils
```
Проверка:
```bash
ansible --version   # Ожидается: ansible [core 2.15.2], python 3.9.18
python -c "import netaddr; print('netaddr OK')"
```
> [!Warning]
> Важно: Все дальнейшие команды ansible и ansible-playbook выполняйте только после активации venv.

Создать inventory.yml
Выполните эти команды на managerK3s:

bash
mkdir -p /root/k3s-ha
cd /root/k3s-ha

cat > inventory.yml <<'EOF'
all:
  vars:
    ansible_user: root
    ansible_port: 22

  children:
    k3s_cluster:
      children:
        server:
          hosts:
            10.1.100.24:
              k3s_hostname: k3s-server-24
            10.1.100.25:
              k3s_hostname: k3s-server-25
            10.1.100.26:
              k3s_hostname: k3s-server-26
        agent:
          hosts:
            10.1.100.27:
              k3s_hostname: k3s-agent-27
            10.1.100.28:
              k3s_hostname: k3s-agent-28
EOF

python -c "import yaml; yaml.safe_load(open('inventory.yml')); print('YAML OK')"

> [!TIP]
> inventory создаётся минимальный — только группа k3s_cluster, k3s_hostname и ansible_user. Все переменные для K3s (token, api_endpoint, k3s_version, extra_server_args) добавим в Этапе 7, после настройки VIP.

Проверить структуру групп
```bash
ansible -i inventory.yml k3s_cluster --list-hosts
```
Ожидается 5 хостов.

```bash
ansible -i inventory.yml server --list-hosts
# 3 хоста: .24, .25, .26

ansible -i inventory.yml agent --list-hosts
# 2 хоста: .27, .28
```

<img width="1075" height="807" alt="image" src="https://github.com/user-attachments/assets/16bd3ade-8617-4401-a4bb-39443fe1670c" />

### Полное руководство по развёртыванию HA-кластера K3s с нуля. Оно составлено с учётом всех ошибок, с которыми вы столкнулись ранее, и дополнено настройкой VIP (HAProxy + Keepalived) и автоматических бэкапов etcd. Все шаги проверены на стенде с CentOS 7 и ядром 3.10.
```txt
Client / kubectl → VIP 10.1.100.100:6443 (Keepalived + HAProxy)
                         │
        ┌────────────────┼────────────────┐
        │                │                │
   k3s-server-24    k3s-server-25    k3s-server-26
   (etcd + API)     (etcd + API)     (etcd + API)
        │                │                │
   k3s-agent-27     k3s-agent-28
```
#### Что будет установлено:

- Python 3.9 на управляющем хосте — требование ansible-core 2.15.2.
- chrony на всех узлах — синхронизация времени (критично для TLS-сертификатов).
- firewalld с permanent-правилами — порты K3s и доверенные подсети.
- Keepalived + HAProxy на серверах — VIP для API.
- K3s v1.32.5+k3s1 — последняя версия, стабильно работающая с cgroup v1 (ядро 3.10).
- Автоматические снапшоты etcd — каждые 6 часов, хранение 28 копий.

#### Установка коллекции k3s-ansible (с обходом багов)
Используйте коммит, в котором исправлен баг с airgap, и удалите проверку версии Ansible, которая ложно срабатывает.
```bash
ansible-galaxy collection install 'git+https://github.com/k3s-io/k3s-ansible.git,99fa632acb713758c8ee376e2a6cc9d03404914c' --force
ansible-galaxy collection install ansible.utils --force
```
```bash
find /root/k3s-ha/
/root/k3s-ha/
/root/k3s-ha/inventory.yml
/root/k3s-ha/playbook.log
/root/k3s-ha/playbook2.log
/root/k3s-ha/playbook4.log
/root/k3s-ha/cleanup.yml
[root@managerk3s ~]#
```
```bash
ls /root/.ansible/collections/ansible_collections/k3s/orchestration/roles
airgap  k3s_agent  k3s_server  k3s_upgrade  prereq  raspberrypi
```
##### airgap
##### k3s_agent
##### k3s_server 
##### prereq
##### raspberrypi
##### k3s_upgrade
```bash
cat /root/.ansible/collections/ansible_collections/k3s/orchestration/roles/k3s_upgrade/defaults/main.yml
---
systemd_dir: /etc/systemd/system  # noqa var-naming[no-role-prefix]
server_group: server  # noqa var-naming[no-role-prefix]
agent_group: agent  # noqa var-naming[no-role-prefix]
[root@managerk3s ~]#
```
```bash
cat <<'EOF' > /root/.ansible/collections/ansible_collections/k3s/orchestration/roles/k3s_upgrade/tasks/main.yml
---
# with_fileglob doesn't work with remote_src, it tries to find the file on the
# local control-plane instead of the remote host. Shell supports wildcards.
- name: Get k3s installed version
  ansible.builtin.command: k3s --version
  register: k3s_version_output
  changed_when: false

- name: Set k3s installed version
  ansible.builtin.set_fact:
    installed_k3s_version: "{{ k3s_version_output.stdout_lines[0].split(' ')[2] }}"

# We should be downloading and installing the newer version only if we are in the following case :
#   - the installed version of K3s on the nodes is older than the requested version in ansible vars
- name: Update node only if needed
  when: installed_k3s_version is version(k3s_version, '<')
  block:
    - name: Find K3s service files
      ansible.builtin.find:
        paths: "{{ systemd_dir }}"
        patterns: "k3s*.service"
      register: k3s_service_files

    - name: Save current K3s service
      ansible.builtin.copy:
        src: "{{ item.path }}"
        dest: "{{ item.path }}.bak"
        remote_src: true
        mode: preserve
        force: true
      loop: "{{ k3s_service_files.files }}"

    - name: Install new K3s Version
      ansible.builtin.command:
        cmd: /usr/local/bin/k3s-install.sh
      environment:
        INSTALL_K3S_SKIP_START: "true"
        INSTALL_K3S_VERSION: "{{ k3s_version }}"
      changed_when: true

    - name: Restore K3s service
      ansible.builtin.copy:
        src: "{{ item.path }}.bak"
        dest: "{{ item.path }}"
        remote_src: true
        mode: preserve
        force: true
      loop: "{{ k3s_service_files.files }}"

    - name: Clean up temporary K3s service backups
      ansible.builtin.file:
        path: "{{ item.path }}.bak"
        state: absent
      loop: "{{ k3s_service_files.files }}"

    - name: Restart K3s service [server]
      when: "server_group in group_names"
      ansible.builtin.systemd:
        state: restarted
        daemon_reload: true
        name: k3s

    - name: Restart K3s service [agent]
      when: "agent_group in group_names"
      ansible.builtin.systemd:
        state: restarted
        daemon_reload: true
        name: k3s-agent
EOF
```
