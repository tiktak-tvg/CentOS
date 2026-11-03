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

### Шаг 2: chrony на всей группе k3s_cluster
SSH-доступ и уникальные hostname для k3s_cluster
#### 2.1. SSH-доступ без пароля
```bash
ssh-keygen -t rsa -b 4096   # если ключа ещё нет
for ip in 24 25 26 27 28; do
  ssh-copy-id root@10.1.100.$ip
done
```
#### 2.2. Создать inventory.yml
Выполните эти команды на managerK3s:
```bash
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
```
> [!TIP]
> inventory создаётся минимальный — только группа k3s_cluster, k3s_hostname и ansible_user. Все переменные для K3s (token, api_endpoint, k3s_version, extra_server_args) добавим на следующих шагах.

###### Проверка через группу k3s_cluster:
```bash
ansible -i inventory.yml k3s_cluster -m ping
```
<img width="1077" height="568" alt="image" src="https://github.com/user-attachments/assets/5a0b8b4f-8a27-4b86-8aeb-4a3d9673e168" />

###### Проверить структуру групп
```bash
cd /root/k3s-ha
ansible -i inventory.yml k3s_cluster --list-hosts
```
Ожидается 5 хостов.

<img width="1075" height="161" alt="image" src="https://github.com/user-attachments/assets/c1b814e3-86f6-47b4-82d1-d5137e80fa10" />

```bash
ansible -i inventory.yml server --list-hosts

# (ansible-venv) [root@managerk3s k3s-ha]# ansible -i inventory.yml server --list-hosts
#   hosts (3):
#     10.1.100.24
#     10.1.100.25
#     10.1.100.26
# (ansible-venv) [root@managerk3s k3s-ha]#


ansible -i inventory.yml agent --list-hosts
# 2 хоста: .27, .28
```
#### 2.3. Уникальные hostname через k3s_cluster и k3s_hostname
Используем per-host переменную k3s_hostname из inventory — это самый чистый способ задать разные имена для разных узлов через одну команду.
```bash
ansible -i inventory.yml k3s_cluster -m hostname -a "name={{ k3s_hostname }}" -b
```
##### Проверка:
```bash
ansible -i inventory.yml k3s_cluster -m shell -a "hostname"
```
##### Ожидается:
- k3s-server-24, k3s-server-25, k3s-server-26
- k3s-agent-27, k3s-agent-28

> [!Warning]
> Уникальные hostname нужны для etcd — иначе узлы .25/.26 получат duplicate node name found и не смогут присоединиться.

<img width="1073" height="193" alt="image" src="https://github.com/user-attachments/assets/1293af79-8498-424b-9f7b-e0a81263245d" />

#### 2.4. Установка на managerK3s
```bash
yum install -y chrony
systemctl enable --now chronyd
```
#### 2.5. Установка на всех узлах группы k3s_cluster
```bash
ansible -i inventory.yml k3s_cluster -m yum -a "name=chrony state=present" -b
ansible -i inventory.yml k3s_cluster -m systemd -a "name=chronyd state=started enabled=yes" -b
```
<img width="1073" height="1106" alt="image" src="https://github.com/user-attachments/assets/a2c8142c-8086-4a77-9395-a4589671b2c8" />

#### 2.6. Проверка синхронизации
```bash
ansible -i inventory.yml k3s_cluster -m shell -a "chronyc tracking | grep 'System time'"
```
System time должен быть меньше 1 секунды на всех узлах.

```bash
ansible -i inventory.yml k3s_cluster -m shell -a "date -u '+%Y-%m-%d %H:%M:%S'"
```
Все строки различаются не более чем на 2 секунды.

<img width="1076" height="351" alt="image" src="https://github.com/user-attachments/assets/5180ad92-02eb-480a-a692-a5b9aed19253" />

Если chrony не находит NTP-сервер (изолированная сеть) — добавьте в **/etc/chrony.conf** строку server **<внутренний-NTP-IP> iburst** и перезапустите chronyd на всех узлах через k3s_cluster.

### Шаг 3: Firewalld permanent-правила
Правила должны быть permanent — иначе после перезагрузки они исчезнут, и kubectl «ослепнет».

#### 3.1. Серверы (группа server)
```bash
ansible -i inventory.yml server -m shell -a "firewall-cmd --permanent --add-port=6443/tcp; firewall-cmd --permanent --add-port=2379/tcp; firewall-cmd --permanent --add-port=2380/tcp; firewall-cmd --permanent --add-port=10250/tcp; firewall-cmd --permanent --add-port=8472/udp; firewall-cmd --permanent --add-port=51820/udp; firewall-cmd --permanent --add-port=51821/udp; firewall-cmd --permanent --add-port=8404/tcp; firewall-cmd --permanent --zone=trusted --add-source=10.42.0.0/16; firewall-cmd --permanent --zone=trusted --add-source=10.43.0.0/16; firewall-cmd --reload" -b
```
#### 3.2. Агенты (группа agent)
```bash
ansible -i inventory.yml agent -m shell -a "firewall-cmd --permanent --add-port=10250/tcp; firewall-cmd --permanent --add-port=8472/udp; firewall-cmd --permanent --add-port=51820/udp; firewall-cmd --permanent --add-port=51821/udp; firewall-cmd --permanent --zone=trusted --add-source=10.42.0.0/16; firewall-cmd --permanent --zone=trusted --add-source=10.43.0.0/16; firewall-cmd --reload" -b
```
#### 3.3. Проверка через k3s_cluster
```bash
ansible -i inventory.yml k3s_cluster -m shell -a "firewall-cmd --list-ports; firewall-cmd --zone=trusted --list-sources"
```
На серверах: 6443/tcp 2379/tcp 2380/tcp 10250/tcp 8472/udp 51820/udp 51821/udp 8404/tcp и 10.42.0.0/16 10.43.0.0/16.<br>
На агентах: 10250/tcp 8472/udp 51820/udp 51821/udp и 10.42.0.0/16 10.43.0.0/16.

<img width="1074" height="279" alt="image" src="https://github.com/user-attachments/assets/dd911220-3807-4d5a-a635-5cc2c564dbbe" />

### Шаг 4: Коллекция k3s-ansible с обходом багов
#### 4.1. Установка из коммита с фиксом airgap
```bash
ansible-galaxy collection install 'git+https://github.com/k3s-io/k3s-ansible.git,99fa632acb713758c8ee376e2a6cc9d03404914c' --force
```
#### 4.2. Обнуление роли airgap
Версия 1.0.0 падает с No file was found when using first_found на airgap-задачах даже при онлайн-установке.
```bash
cat > /root/.ansible/collections/ansible_collections/k3s/orchestration/roles/airgap/tasks/main.yml <<'EOF'
---
# Airgap role disabled: online installation
EOF
```
#### 4.3. Удаление ложной проверки версии Ansible
```bash
cd /root/.ansible/collections/ansible_collections/k3s/orchestration/roles/prereq/tasks/

python <<'PYEOF'
import re
with open('main.yml') as f:
    content = f.read()
pattern = re.compile(r'\n[ ]*- name: Enforce minimum Ansible version\n.*?(?=\n[ ]*- name:|\Z)', re.DOTALL)
new_content, n = pattern.subn('', content)
with open('main.yml', 'w') as f:
    f.write(new_content)
print(f"Removed {n} block(s)")
PYEOF
```
Проверка:
```bash
grep -r 'Enforce minimum Ansible version' /root/.ansible/collections/ansible_collections/k3s/orchestration/roles/prereq/tasks/ 
# Ничего не должно вывести

grep -c 'name:' /root/.ansible/collections/ansible_collections/k3s/orchestration/roles/airgap/tasks/main.yml
# Ожидается: 0
```
<img width="1077" height="101" alt="image" src="https://github.com/user-attachments/assets/9e600dac-313f-4140-ba20-a9d59304e672" />

### Шаг 5: VIP — HAProxy + Keepalived
#### 5.1. Установка пакетов на группу server
```bash
ansible -i inventory.yml server -m yum -a "name=haproxy,keepalived state=present" -b
```
#### 5.2. HAProxy — общий конфиг для группы server
```bash
cat > /tmp/haproxy.cfg <<'EOF'
global
    log /dev/log local0
    log /dev/log local1 notice
    chroot /var/lib/haproxy
    stats socket /var/run/haproxy.admin level admin
    stats timeout 30s
    user haproxy
    group haproxy
    daemon
    maxconn 2000

defaults
    log global
    mode tcp
    option tcplog
    option dontlognull
    timeout connect 5000
    timeout client  50000
    timeout server  50000

listen stats
    bind *:8404
    mode http
    stats enable
    stats uri /stats
    stats refresh 10s

frontend k3s-api
    bind *:6443
    mode tcp
    default_backend k3s-servers

backend k3s-servers
    mode tcp
    balance roundrobin
    option tcp-check
    server k3s-server-24 10.1.100.24:6443 check fall 3 rise 2
    server k3s-server-25 10.1.100.25:6443 check fall 3 rise 2
    server k3s-server-26 10.1.100.26:6443 check fall 3 rise 2
EOF
```
```bash
ansible -i inventory.yml server -m copy -a "src=/tmp/haproxy.cfg dest=/etc/haproxy/haproxy.cfg owner=root group=root mode=0644" -b
```
#### 5.3. Keepalived — три конфига с разными приоритетами
```bash
# Проверьте имя интерфейса
ansible -i inventory.yml server -m shell -a "ip -o link show | awk -F': ' '{print \$2}' | grep -v lo"

# Ожидается: ens33 (на всех серверах одинаково)
```
Шаблон базового конфига (.24 — MASTER, priority 100):
```bash
cat > /tmp/keepalived-24.conf <<'EOF'
global_defs {
    router_id K3S_VIP_24
    script_user root
    enable_script_security
}

vrrp_script chk_haproxy {
    script "/usr/bin/killall -0 haproxy"
    interval 2
    weight -30
    fall 3
    rise 2
}

vrrp_instance VI_K3S {
    state MASTER
    interface ens33
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass K3sVIP123
    }
    virtual_ipaddress {
        10.1.100.100/24
    }
    track_script {
        chk_haproxy
    }
}
EOF
```
Варианты для .25 и .26:
```bash
sed 's/K3S_VIP_24/K3S_VIP_25/; s/state MASTER/state BACKUP/; s/priority 100/priority 90/' /tmp/keepalived-24.conf > /tmp/keepalived-25.conf
sed 's/K3S_VIP_24/K3S_VIP_26/; s/state MASTER/state BACKUP/; s/priority 100/priority 80/' /tmp/keepalived-24.conf > /tmp/keepalived-26.conf
```
Распространение по узлам (через группу server, но с разными файлами):
```bash
scp /tmp/keepalived-24.conf root@10.1.100.24:/etc/keepalived/keepalived.conf
scp /tmp/keepalived-25.conf root@10.1.100.25:/etc/keepalived/keepalived.conf
scp /tmp/keepalived-26.conf root@10.1.100.26:/etc/keepalived/keepalived.conf
```
#### 5.4. Запуск сервисов на группе server
```bash
ansible -i inventory.yml server -m systemd -a "name=haproxy state=started enabled=yes" -b
ansible -i inventory.yml server -m systemd -a "name=keepalived state=started enabled=yes" -b
```
#### 5.5. Проверка VIP
```bash
# VIP должен быть на .24 (MASTER)
ssh root@10.1.100.24 "ip a show ens33 | grep 10.1.100.100"
# Ожидается: inet 10.1.100.100/24 scope global secondary ens33

# HAProxy stats
curl -s http://10.1.100.24:8404/stats | grep -E 'k3s-server|UP|DOWN'
```
До установки K3s backend'ы будут DOWN — это нормально. После установки все три станут UP.

### Шаг 6: Cменить порт HAProxy на 16443
Это стандартная практика для HA-кластеров K3s. HAProxy будет слушать VIP:16443, балансируя на реальные API-серверы IP:6443. K3s будет спокойно занимать 6443 на своём IP.

#### 6.1. Заменить bind *:6443 на bind *:16443 в конфиге HAProxy на всех серверах:
```bash
ansible -i inventory.yml server -m lineinfile -a "path=/etc/haproxy/haproxy.cfg regexp='^    bind \*:6443$' line='    bind *:16443'" -b
```
#### 6.2. Добавить порт 16443 в SELinux (иначе снова получите cannot bind socket):
```bash
ansible -i inventory.yml server -m shell -a "semanage port -a -t http_port_t -p tcp 16443" -b
```
Если semanage port -a вернёт ValueError: Port tcp/16443 already defined — используйте -m вместо -a:
```bash
ansible -i inventory.yml server -m shell -a "semanage port -m -t http_port_t -p tcp 16443" -b
```
#### 6.3. Перезапустить HAProxy (именно stop + start, а не restart — важно для пересоздания сокетов):
```bash
ansible -i inventory.yml server -m systemd -a "name=haproxy state=stopped" -b
ansible -i inventory.yml server -m systemd -a "name=haproxy state=started" -b
```
#### 6.4. Проверить, что HAProxy слушает 16443, а 6443 свободен:
```bash
ansible -i inventory.yml server -m shell -a "systemctl is-active haproxy; ss -tlnp | grep -E '16443|8404|6443'"
```
Ожидается для каждого сервера:

```text
active
LISTEN 0 128 *:16443 *:* users:(("haproxy",...))
LISTEN 0 128 *:8404  *:* users:(("haproxy",...))
```
Строки с 6443 не должно быть — порт теперь свободен для K3s.

### Шаг 7: Финальный inventory с группой k3s_cluster
Перезаписываем inventory.yml, добавляя все переменные, обязательные для k3s-ansible.
```bash
cd /root/k3s-ha
TOKEN=$(openssl rand -base64 64 | tr -d '\n')

cat > inventory.yml <<EOF
all:
  vars:
    ansible_user: root
    ansible_port: 22

  children:
    k3s_cluster:
      vars:
        k3s_version: v1.32.5+k3s1
        kubeconfig: /root/.kube/config
        api_endpoint: 10.1.100.100
        api_port: 16443
        token: "${TOKEN}"
        extra_server_args: "--tls-san=10.1.100.100 --etcd-snapshot-schedule-cron='0 */6 * * *' --etcd-snapshot-retention=28"
        extra_agent_args: ""
        user_kubectl: true
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
```
Проверка готовности к установке
Перед ``Шаг 8:`` проверьте всё разом:
```bash
# 1. HAProxy слушает 16443 и 8404, 6443 свободен
ansible -i inventory.yml server -m shell -a "ss -tlnp | grep -E '6443|16443|8404'"

# 2. VIP на .24
ssh root@10.1.100.24 "ip a | grep 10.1.100.100"

# 3. VIP доступен на 16443 с manager
nc -zv 10.1.100.100 16443
# или curl -k https://10.1.100.100:16443/version

# 4. HAProxy stats: все три backend'а DOWN (K3s ещё не установлен)
curl -s http://10.1.100.24:8404/stats | grep -E 'k3s-server|DOWN'
```

### Шаг 8: Установка K3s через group k3s_cluster
```bash
cd /root/k3s-ha
nohup ansible-playbook /root/.ansible/collections/ansible_collections/k3s/orchestration/playbooks/site.yml -i inventory.yml > playbook.log 2>&1 &
echo "PID: $!"

tail -f /root/k3s-ha/playbook.log
```
Что важно помнить:

На задаче Enable and check K3s service для .24 будет пауза 30–90 секунд — инициализация etcd. Не нажимайте Ctrl+C.

В логах .24 (ssh root@10.1.100.24 "journalctl -u k3s -f") появятся строки Serving HTTP bootstrap from datastore for 10.1.100.25/26 — признак, что etcd готов принимать остальные узлы.

Задача Verify that all server nodes joined может временно упасть с certificate signed by unknown authority — это некритично, K3s уже сформировал кластер. Playbook продолжит работу, PLAY RECAP покажет failed=0.

### Шаг 9: Проверки через группу k3s_cluster
#### 9.1. Кластер
```bash
kubectl config use-context k3s-ansible
kubectl get nodes -o wide
kubectl get --raw='/healthz/etcd'
kubectl get pods -A
# Ожидается: 5 узлов Ready, etcd ok, в kube-system все поды Running.
```
#### 9.2. VIP
```bash
nc -zv 10.1.100.100 6443
# Ожидается: succeeded

curl -s http://10.1.100.24:8404/stats | grep -E 'k3s-server|UP|DOWN'
# Все три backend'а — UP
```
#### 9.3. etcd member list (через группу server)
```bash
ssh root@10.1.100.24 "etcdctl --endpoints https://127.0.0.1:2379 \
  --cacert /var/lib/rancher/k3s/server/tls/etcd/server-ca.crt \
  --cert /var/lib/rancher/k3s/server/tls/etcd/server-client.crt \
  --key /var/lib/rancher/k3s/server/tls/etcd/server-client.key \
  member list -w table"
# Ожидается: 3 участника (k3s-server-24/25/26), все started.
```
#### 9.4. Бэкапы etcd — первая проверка
```bash
# Ручной снапшот для немедленной проверки
ssh root@10.1.100.24 "k3s etcd-snapshot save --name manual-$(date +%Y%m%d-%H%M)"

# Список снапшотов
ssh root@10.1.100.24 "ls -lh /var/lib/rancher/k3s/server/db/snapshots/"

# Ожидается файл manual-YYYYMMDD-HHMM размером ~1–2 MB.
```
Проверка, что cron настроен:
```bash
ssh root@10.1.100.24 "grep -A1 'ExecStart' /etc/systemd/system/k3s.service"

# В строке ExecStart должны быть --etcd-snapshot-schedule-cron и --etcd-snapshot-retention
# Автоматические снапшоты начнут появляться после первого запуска cron (0 */6 * * * — в 00:00, 06:00, 12:00, 18:00).
```
### Шаг 10: Failover-тест группы k3s_cluster
Терминал 1 — мониторинг через VIP:
```bash
watch -n 2 'kubectl get nodes -o wide; echo "---"; kubectl get --raw="/healthz/etcd"'
```
Терминал 2 — отключаем .24:
```bash
ssh root@10.1.100.24 "systemctl stop k3s"
```
Наблюдайте:

Время	Ожидаемое поведение
0–30 сек	.24 → NotReady, kubectl продолжает отвечать (через VIP на .25/.26)
30–60 сек	Keepalived переключает VIP на .25 (или .26), HAProxy исключает backend .24
Всё время	.25, .26, .27, .28 — Ready, /healthz/etcd → ok

Проверка записи в API через VIP:
```bash
kubectl create deployment test-ha --image=nginx
kubectl get deployment test-ha
kubectl scale deployment test-ha --replicas=2
kubectl get pods -l app=test-ha
```
Возврат .24:
```bash
ssh root@10.1.100.24 "systemctl start k3s"
sleep 45

# VIP вернулся на MASTER
ssh root@10.1.100.24 "ip a show ens33 | grep 10.1.100.100"

# Узел снова Ready
kubectl get nodes
```
Очистка теста:
```bash
kubectl delete deployment test-ha
```
### Финальный чек-лист
Проверка	Команда	Ожидание
Все узлы k3s_cluster Ready	kubectl get nodes	5 × Ready
etcd кворум 3/3	kubectl get --raw='/healthz/etcd'	ok
VIP доступен	nc -zv 10.1.100.100 6443	succeeded
HAProxy backend'ы	curl -s http://10.1.100.24:8404/stats	3 × UP
chrony синхронизирован	chronyc tracking на каждом узле	System time < 1s
firewalld permanent	firewall-cmd --list-ports	все нужные порты
Бэкапы etcd	ls /var/lib/rancher/k3s/server/db/snapshots/	файлы есть
Failover	Отключить .24 → kubectl работает	да
### Учтённые ошибки прошлых итераций
Ошибка	Решение в руководстве
first_found в airgap падает	Роль airgap обнулена (Этап 5.2)
Enforce minimum Ansible version ложно срабатывает	Задача удалена (Этап 5.3)
ansible_user is undefined	Задана в inventory на уровне all.vars (Этапы 0.1, 7)
token/extra_server_args undefined	Заданы в k3s_cluster.vars (Этап 7)
ipwrap не работает	netaddr + ansible.utils установлены (Этап 1)
SyntaxError в modprobe.py на CentOS 7	community.general 11.4.5 (Этап 1.4)
duplicate node name found	Уникальные hostname через k3s_hostname (Этап 3.2) + cleanup (Этап 0)
Сертификат «из будущего»	chrony на k3s_cluster (Этап 2) + cleanup (Этап 0)
no route to host на 6443	permanent-правила firewalld (Этап 4)
kubectl падает при отказе .24	VIP через HAProxy + Keepalived (Этап 6)
venv создан не на том Python	Явная проверка python --version (Этап 1.2)
Старые пропатченные коллекции конфликтуют	rm -rf /root/.ansible/collections/ansible_collections/ (Этап 0.4)
Старый kubeconfig с мёртвым IP	rm -rf /root/.kube/ (Этап 0.6)
### После успешного развёртывания
Git-репозиторий — сохраните inventory.yml, cleanup.yml, requirements-pip.txt, requirements-collections.txt и это руководство.

Ansible Vault — вынесите token из inventory.yml в group_vars/all/vault.yml (ansible-vault encrypt).

Внешние бэкапы — настройте rsync/S3 для /var/lib/rancher/k3s/server/db/snapshots/.

Мониторинг — kube-prometheus-stack через Helm.

Ingress и TLS — cert-manager + Let's Encrypt для Traefik.

Миграция на AlmaLinux/Rocky 9 — CentOS 7 + ядро 3.10 рабочие, но K3s 1.33+ потребует cgroup v2. Зафиксируйте v1.32.5+k3s1 и планируйте переезд.

Всё руководство использует группу k3s_cluster как единую точку приложения команд (с точечным разделением на server/agent там, где это действительно нужно — firewalld, keepalived). Это делает управление кластером единообразным и упрощает будущие операции (обновления, патчи, аудит).










### Полное описание конфигов по развёртыванию HA-кластера K3s.
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

#### Установка коллекции k3s-ansible
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
