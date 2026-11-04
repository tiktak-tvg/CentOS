### Пошаговое руководство по развёртыванию лёгкого Kubernetes-кластера (K3s) на трёх и более машинах с помощью официальной Ansible-коллекции k3s-io/k3s-ansible.
Архитектура
```txt
3.10.0-1160.119.1.el7.x86_64
CentOS Linux release 7.9.2009 (Core)

Все команды выполняются на управляющем хосте hostname:managerK3s (ip:10.1.100.10) под root.
На managerK3s  установлены Ansible, nginx, firewalld

Хосты:

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

```
Схема кластера
```text
managerK3s (10.1.100.10) → kubectl → VIP 10.1.100.100:16443 (HAProxy + Keepalived)
                                              │
                    ┌─────────────────────────┼─────────────────────────┐
                    │                         │                         │
              k3s-server-24             k3s-server-25             k3s-server-26
              (etcd + API)              (etcd + API)              (etcd + API)
                    │                         │                         │
              k3s-agent-27              k3s-agent-28
```
*Требования:*
- **Все хосты:** CentOS 7.9, ядро 3.10, firewalld, SELinux Enforcing.
- **Управляющий хост:** managerK3s (10.1.100.10), root, SSH-доступ ко всем узлам без пароля.
- **Узлы кластера:** 3 сервера (.24, .25, .26) + 2 агента (.27, .28), уникальные hostname уже заданы.
- **Версия K3s:** v1.32.5+k3s1 — последняя, стабильно работающая с cgroup v1.

### Этап 1: Предустановка Ansible на managerK3s
Выполняется только на managerK3s от root.

#### 1.1. Сборка Python 3.9.18
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
##### Проверка:
```bash
/opt/python3.9/bin/python3.9 --version
# Ожидается: Python 3.9.18
```
#### 1.2. Виртуальное окружение и ansible-core
```bash
# Убедитесь, что вы НЕ в активированном venv
/opt/python3.9/bin/python3.9 -m venv ~/ansible-venv
source ~/ansible-venv/bin/activate

# Проверка, что venv на Python 3.9
which python
# Ожидается: /root/ansible-venv/bin/python
python --version
# Ожидается: Python 3.9.18

pip install --upgrade pip
pip install ansible-core==2.15.2 netaddr
```
#### 1.3. Служебные коллекции (фиксированные версии)
```bash
ansible-galaxy collection install ansible.utils --force
ansible-galaxy collection install 'ansible.posix:==1.5.4' --force
ansible-galaxy collection install 'community.general:==11.4.5' --force
```
#### 1.4. Финальная проверка
```bash
ansible --version
# Ожидается: ansible [core 2.15.2], python version = 3.9.18

which -a ansible ansible-playbook ansible-galaxy python pip
# Все пять путей должны начинаться с /root/ansible-venv/bin/

python -c "import netaddr; print('netaddr:', netaddr.__version__)"
python -c "import yaml; print('yaml:', yaml.__version__)"

ansible-galaxy collection list | grep -E 'ansible.utils|ansible.posix|community.general'
# Ожидается: 6.1.1, 1.5.4, 11.4.5
```
#### 1.5. Фиксация окружения
```bash
mkdir -p /root/k3s-ha
pip freeze > /root/k3s-ha/requirements-pip.txt
ansible-galaxy collection list > /root/k3s-ha/requirements-collections.txt
```
### Этап 2: Inventory и SSH-доступ
#### 2.1. SSH-ключи без пароля
```bash
ssh-keygen -t rsa -b 4096   # если ключа ещё нет
for ip in 24 25 26 27 28; do
  ssh-copy-id root@10.1.100.$ip
done
```
##### Проверка:
```bash
for ip in 24 25 26 27 28; do
  ssh -o ConnectTimeout=3 root@10.1.100.$ip "hostname"
done
```
#### 2.2. Создание inventory.yml
```bash
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
##### Проверка структуры групп:
```bash
ansible -i inventory.yml k3s_cluster --list-hosts
# 5 хостов
ansible -i inventory.yml server --list-hosts
# 3 хоста
ansible -i inventory.yml agent --list-hosts
# 2 хоста
```
### Этап 3: chrony на всей группе k3s_cluster
```bash
# На managerK3s
yum install -y chrony
systemctl enable --now chronyd

# На всех узлах
ansible -i inventory.yml k3s_cluster -m yum -a "name=chrony state=present" -b
ansible -i inventory.yml k3s_cluster -m systemd -a "name=chronyd state=started enabled=yes" -b
```
##### Проверка синхронизации:
```bash
ansible -i inventory.yml k3s_cluster -m shell -a "chronyc tracking | grep 'System time'"
# System time должен быть < 1 секунды на всех узлах

ansible -i inventory.yml k3s_cluster -m shell -a "date -u '+%Y-%m-%d %H:%M:%S'"
# Все строки различаются не более чем на 2 секунды
```
### Этап 4: Firewalld permanent-правила
#### 4.1. Серверы (группа server)
```bash
ansible -i inventory.yml server -m shell -a "firewall-cmd --permanent --add-port=6443/tcp; firewall-cmd --permanent --add-port=16443/tcp; firewall-cmd --permanent --add-port=2379/tcp; firewall-cmd --permanent --add-port=2380/tcp; firewall-cmd --permanent --add-port=10250/tcp; firewall-cmd --permanent --add-port=8472/udp; firewall-cmd --permanent --add-port=51820/udp; firewall-cmd --permanent --add-port=51821/udp; firewall-cmd --permanent --add-port=8404/tcp; firewall-cmd --permanent --zone=trusted --add-source=10.42.0.0/16; firewall-cmd --permanent --zone=trusted --add-source=10.43.0.0/16; firewall-cmd --permanent --zone=trusted --add-source=10.1.100.10; firewall-cmd --reload" -b
```
#### 4.2. Агенты (группа agent)
```bash
ansible -i inventory.yml agent -m shell -a "firewall-cmd --permanent --add-port=10250/tcp; firewall-cmd --permanent --add-port=8472/udp; firewall-cmd --permanent --add-port=51820/udp; firewall-cmd --permanent --add-port=51821/udp; firewall-cmd --permanent --zone=trusted --add-source=10.42.0.0/16; firewall-cmd --permanent --zone=trusted --add-source=10.43.0.0/16; firewall-cmd --permanent --zone=trusted --add-source=10.1.100.10; firewall-cmd --reload" -b
```
#### 4.3. Разрешить VRRP для Keepalived (на серверах)
```bash
ansible -i inventory.yml server -m shell -a "firewall-cmd --permanent --add-rich-rule='rule protocol value=\"vrrp\" accept'; firewall-cmd --reload" -b
```
#### 4.4. Проверка
```bash
ansible -i inventory.yml k3s_cluster -m shell -a "firewall-cmd --list-ports; firewall-cmd --zone=trusted --list-sources"

# Ожидается:
# Серверы: 6443/tcp 16443/tcp 2379/tcp 2380/tcp 10250/tcp 8472/udp 51820/udp 51821/udp 8404/tcp + 10.42.0.0/16 10.43.0.0/16 10.1.100.10
# Агенты: 10250/tcp 8472/udp 51820/udp 51821/udp + 10.42.0.0/16 10.43.0.0/16 10.1.100.10
```
### Этап 5: SELinux — порты и boolean
#### 5.1. Добавить порты в http_port_t
```bash
# Установить semanage, если ещё нет
ansible -i inventory.yml server -m yum -a "name=policycoreutils-python state=present" -b

# Порты 6443, 16443, 8404 (для HAProxy)
ansible -i inventory.yml server -m shell -a "semanage port -a -t http_port_t -p tcp 6443" -b
ansible -i inventory.yml server -m shell -a "semanage port -a -t http_port_t -p tcp 16443" -b
ansible -i inventory.yml server -m shell -a "semanage port -a -t http_port_t -p tcp 8404" -b

# Порты 2379, 2380 (для etcd)
ansible -i inventory.yml server -m shell -a "semanage port -a -t http_port_t -p tcp 2379" -b
ansible -i inventory.yml server -m shell -a "semanage port -a -t http_port_t -p tcp 2380" -b
```
Если semanage port -a возвращает ValueError: Port tcp/XXXX already defined, используйте -m вместо -a:
```bash
ansible -i inventory.yml server -m shell -a "semanage port -m -t http_port_t -p tcp XXXX" -b
```
#### 5.2. Разрешить HAProxy подключаться к любым портам
```bash
ansible -i inventory.yml server -m shell -a "setsebool -P haproxy_connect_any 1" -b
```
#### 5.3. Разрешить bind на нелокальные адреса (для VIP)
```bash
ansible -i inventory.yml server -m shell -a "sysctl -w net.ipv4.ip_nonlocal_bind=1" -b
ansible -i inventory.yml server -m lineinfile -a "path=/etc/sysctl.conf line='net.ipv4.ip_nonlocal_bind=1' state=present" -b
ansible -i inventory.yml server -m shell -a "sysctl -p" -b
```
#### 5.4. Проверка портов в SELinux
```bash
ansible -i inventory.yml server -m shell -a "semanage port -l | grep -E '6443|16443|8404|2379|2380'" -b
# Все пять портов должны быть в списке http_port_t
```
### Этап 6: VIP — HAProxy + Keepalived
#### 6.1. Установка пакетов
```bash
ansible -i inventory.yml server -m yum -a "name=haproxy,keepalived state=present" -b
```
#### 6.2. HAProxy — конфиг (порт 16443)
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
    bind *:16443
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

ansible -i inventory.yml server -m copy -a "src=/tmp/haproxy.cfg dest=/etc/haproxy/haproxy.cfg owner=root group=root mode=0644" -b
```
#### 6.3. Keepalived — три конфига с разными приоритетами
```bash
# Проверка имени интерфейса (ожидается ens33)
ansible -i inventory.yml server -m shell -a "ip -o link show | awk -F': ' '{print \$2}' | grep -v lo"
```
##### Шаблон для .24 (MASTER, priority 100):
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
##### Генерация конфигов для .25 и .26:
```bash
sed 's/K3S_VIP_24/K3S_VIP_25/; s/state MASTER/state BACKUP/; s/priority 100/priority 90/' /tmp/keepalived-24.conf > /tmp/keepalived-25.conf
sed 's/K3S_VIP_24/K3S_VIP_26/; s/state MASTER/state BACKUP/; s/priority 100/priority 80/' /tmp/keepalived-24.conf > /tmp/keepalived-26.conf
```
##### Копирование по узлам:
```bash
scp /tmp/keepalived-24.conf root@10.1.100.24:/etc/keepalived/keepalived.conf
scp /tmp/keepalived-25.conf root@10.1.100.25:/etc/keepalived/keepalived.conf
scp /tmp/keepalived-26.conf root@10.1.100.26:/etc/keepalived/keepalived.conf
```
#### 6.4. Запуск и проверка
```bash
ansible -i inventory.yml server -m systemd -a "name=haproxy state=started enabled=yes" -b
ansible -i inventory.yml server -m systemd -a "name=keepalived state=started enabled=yes" -b
```
##### Проверка VIP и HAProxy:
```bash
# VIP на .24
ssh root@10.1.100.24 "ip a show ens33 | grep 10.1.100.100"
# Ожидается: inet 10.1.100.100/24 scope global secondary ens33

# HAProxy слушает 16443 и 8404
ansible -i inventory.yml server -m shell -a "ss -tlnp | grep -E '16443|8404'"
# Ожидается: LISTEN на *:16443 и *:8404 для haproxy

# HAProxy stats (backend'ы пока DOWN — K3s не установлен)
curl -s http://10.1.100.24:8404/stats | grep -E 'k3s-server|DOWN'
```
### Этап 7: Коллекция k3s-ansible с патчами
#### 7.1. Установка из коммита с фиксом airgap
```bash
ansible-galaxy collection install 'git+https://github.com/k3s-io/k3s-ansible.git,99fa632acb713758c8ee376e2a6cc9d03404914c' --force
```
#### 7.2. Обнуление роли airgap
```bash
cat > /root/.ansible/collections/ansible_collections/k3s/orchestration/roles/airgap/tasks/main.yml <<'EOF'
---
# Airgap role disabled: online installation
EOF
```
#### 7.3. Удаление проверки версии Ansible
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
##### Проверка:
```bash
grep -r 'Enforce minimum Ansible version' /root/.ansible/collections/ansible_collections/k3s/orchestration/roles/prereq/tasks/
# Ничего не должно вывести

grep -c 'name:' /root/.ansible/collections/ansible_collections/k3s/orchestration/roles/airgap/tasks/main.yml
# Ожидается: 0
```
### Этап 8: Финальный inventory с TLS SAN и api_port
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
##### Ключевые переменные:

api_endpoint: 10.1.100.100 — VIP, без порта.

api_port: 16443 — порт HAProxy. Вместе формируют https://10.1.100.100:16443.

extra_server_args — включает --tls-san=10.1.100.100 (критично для сертификата) и снапшоты etcd.

### Этап 9: Установка K3s
```bash
cd /root/k3s-ha
nohup ansible-playbook /root/.ansible/collections/ansible_collections/k3s/orchestration/playbooks/site.yml -i inventory.yml > playbook.log 2>&1 &
echo "PID: $!"
tail -f /root/k3s-ha/playbook.log
```
Что важно помнить:

На задаче Enable and check K3s service для .24 будет пауза 30–90 секунд — инициализация etcd. Не нажимайте Ctrl+C.

В логах .24 появятся строки Serving HTTP bootstrap from datastore for 10.1.100.25/26 — признак, что etcd готов.

Задача Verify that all server nodes joined может временно упасть с certificate signed by unknown authority — это некритично, K3s уже сформировал кластер. Playbook продолжит работу, PLAY RECAP покажет failed=0.

### Этап 10: Проверки после установки
#### 10.1. Кластер
```bash
kubectl config use-context k3s-ansible
kubectl get nodes -o wide
kubectl get --raw='/healthz/etcd'
kubectl get pods -A
Ожидается: 5 узлов Ready, /healthz/etcd → ok, все поды в kube-system — Running.
```
#### 10.2. VIP и HAProxy
```bash
# Проверка API через VIP
curl -k https://10.1.100.100:16443/version
# Ожидается: HTTP 401 Unauthorized (нормально, без токена)

# HAProxy stats: все три backend'а UP
curl -s http://10.1.100.24:8404/stats | grep -E 'k3s-server|UP|DOWN'
```
#### 10.3. etcd member list
```bash
ssh root@10.1.100.24 "k3s etcdctl member list -w table"
# Ожидается: 3 участника (k3s-server-24/25/26), все started
```
#### 10.4. Бэкапы etcd
```bash
# Ручной снапшот для проверки
ssh root@10.1.100.24 "k3s etcd-snapshot save --name manual-$(date +%Y%m%d-%H%M)"
ssh root@10.1.100.24 "ls -lh /var/lib/rancher/k3s/server/db/snapshots/"

# Проверка, что cron настроен
ssh root@10.1.100.24 "grep -A1 'ExecStart' /etc/systemd/system/k3s.service"
# В строке ExecStart должны быть --etcd-snapshot-schedule-cron и --etcd-snapshot-retention
```
### Этап 11: Failover-тест
##### Терминал 1 — мониторинг через VIP:
```bash
watch -n 2 'kubectl get nodes -o wide; echo "---"; kubectl get --raw="/healthz/etcd"'
```
##### Терминал 2 — отключаем .24:
```bash
ssh root@10.1.100.24 "systemctl stop k3s"
```
##### Наблюдайте:
- 10.1.100.24 → NotReady через 30–60 сек.
- kubectl продолжает работать через VIP (HAProxy на .25/.26).
- /healthz/etcd → ok (кворум 2 из 3).

##### Проверка записи в API (Терминал 3):
```bash
kubectl create deployment test-ha --image=nginx
kubectl get pods -l app=test-ha
```
##### Возврат .24:
```bash
ssh root@10.1.100.24 "systemctl start k3s"
sleep 45
kubectl get nodes
```
##### Очистка:
```bash
kubectl delete deployment test-ha
```
### Финальный чек-лист production-готовности
Критерий	|Проверка	|Ожидание
---------- | ------- | -------------------------
HA control-plane (3 сервера)	 |kubectl get nodes	 |3 × Ready, роли control-plane,etcd,master
Worker-узлы (2 агента)	 |kubectl get nodes	 |2 × Ready, роль <none>
etcd кворум	 |kubectl get --raw='/healthz/etcd' |	ok
VIP доступен	 |curl -k https://10.1.100.100:16443/version	 |HTTP 401
HAProxy backend'ы	 |curl -s http://10.1.100.24:8404/stats	 |3 × UP
kubectl через VIP	 |kubectl get nodes	 |Работает
Failover .24	kubectl после  |systemctl stop k3s	 |Продолжает работать
firewalld permanent	 |firewall-cmd --list-ports	 |6443, 16443, 8404, 2379, 2380, 10250, 8472/udp, 51820/udp, 51821/udp
SELinux порты	 |semanage port -l l grep http_port_t	 |6443, 16443, 8404, 2379, 2380
ip_nonlocal_bind=1	 |sysctl net.ipv4.ip_nonlocal_bind	 |1
Бэкапы etcd	 |ls /var/lib/rancher/k3s/server/db/snapshots/	 |Файл manual-*
