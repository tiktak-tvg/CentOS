### Система мониторинга (Zabbix): 
- Более мощная альтернатива Prometheus+Grafana 
- Ansible автоматизирует установку Zabbix Server, веб-интерфейса, базы данных 
- Развертывание Zabbix Agent на всех контролируемых хостах.

**рабочее решение для Zabbix 5.0 LTS (последняя LTS-ветка с официальной поддержкой CentOS 7). Все компоненты (Server, Web, БД) разместим на managerK3s (10.1.100.10), а Zabbix Agent — на всех 6 хостах.**

Архитектура
```text
managerK3s (10.1.100.10)
├── Zabbix Server (порт 10051)
├── Zabbix Web (Apache/Nginx, порт 80)
└── MariaDB 10.5 (БД для Zabbix)

Все хосты (.10, .24, .25, .26, .27, .28)
└── Zabbix Agent (порт 10050) → отправляет метрики на .10:10051
```
Требования к managerK3s: ≥ 4 ГБ RAM, ≥ 20 ГБ диска. MariaDB 10.5 понадобится, потому что стандартная MariaDB 5.5 в CentOS 7 не подходит для Zabbix 5.0.

### Этап 1: Фиксация версий коллекций Ansible
На managerK3s (в активированном venv) установите конкретные версии коллекций, совместимые с CentOS 7 и Zabbix 5.0.

```bash
source /root/ansible-venv/bin/activate

# Коллекция Zabbix версии 2.x (последняя с поддержкой CentOS 7)
ansible-galaxy collection install 'community.zabbix:==2.2.0' --force

# Зависимости для работы с MySQL/MariaDB и PostgreSQL
ansible-galaxy collection install community.mysql --force
ansible-galaxy collection install community.postgresql --force
ansible-galaxy collection install ansible.posix --force
ansible-galaxy collection install community.general --force
```
Проверка:

```bash
ansible-galaxy collection list | grep -E 'community.zabbix|community.mysql|ansible.posix'
# Ожидается: community.zabbix 2.2.0
```
### Этап 2: MariaDB 10.5 на managerK3s
Zabbix 5.0 требует MariaDB 10.5+. Установим её из официального репозитория MariaDB.

```bash
cat > /tmp/install-mariadb.sh <<'EOF'
#!/bin/bash
# Добавляем репозиторий MariaDB 10.5 для CentOS 7
curl -sS https://downloads.mariadb.com/MariaDB/mariadb_repo_setup | bash -s -- --mariadb-server-version=10.5

# Устанавливаем MariaDB
yum install -y MariaDB-server MariaDB-client

# Запускаем и включаем автозапуск
systemctl enable --now mariadb

# Базовая безопасность (без интерактивного ввода)
mysql -e "DELETE FROM mysql.user WHERE User='';"
mysql -e "DROP DATABASE IF EXISTS test;"
mysql -e "FLUSH PRIVILEGES;"

# Создаём БД и пользователя для Zabbix
mysql -e "CREATE DATABASE zabbix CHARACTER SET utf8 COLLATE utf8_bin;"
mysql -e "CREATE USER 'zabbix'@'localhost' IDENTIFIED BY 'ZabbixDBPass123!';"
mysql -e "GRANT ALL PRIVILEGES ON zabbix.* TO 'zabbix'@'localhost';"
mysql -e "FLUSH PRIVILEGES;"

echo "MariaDB 10.5 installed and configured for Zabbix"
EOF

ansible -i /root/k3s-ha/inventory.yml elk -m script -a "/tmp/install-mariadb.sh" -b
```
Проверка: mysql -e "SELECT VERSION();" → должно показать 10.5.x.

### Этап 3: Firewalld на managerK3s
Откройте порты: 10051 (Zabbix Server), 80 (Web), 10050 (Agent, для локальной проверки).

```bash
cat > /tmp/zabbix-firewall.sh <<'EOF'
#!/bin/bash
firewall-cmd --permanent --add-port=10051/tcp
firewall-cmd --permanent --add-port=10050/tcp
firewall-cmd --permanent --add-service=http
firewall-cmd --reload
EOF

ansible -i /root/k3s-ha/inventory.yml elk -m script -a "/tmp/zabbix-firewall.sh" -b
```
SELinux: на время установки можно перевести в Permissive, чтобы избежать проблем с правами:

```bash
ansible -i /root/k3s-ha/inventory.yml elk -m shell -a "setenforce 0" -b
```
Для production вернуть setenforce 1 и настроить политики через audit2allow.

### Этап 4: Inventory для Zabbix
Добавьте в /root/k3s-ha/inventory.yml группы для Zabbix (по аналогии с ELK):

```bash
cat >> /root/k3s-ha/inventory.yml <<'EOF'

    # === Zabbix Server (всё на managerK3s) ===
    zabbix_server:
      hosts:
        10.1.100.10:

    # === Zabbix Agents (все хосты) ===
    zabbix_agents:
      hosts:
        10.1.100.10:
        10.1.100.24:
        10.1.100.25:
        10.1.100.26:
        10.1.100.27:
        10.1.100.28:
EOF
```
Проверка:

```bash
ansible -i /root/k3s-ha/inventory.yml zabbix_server --list-hosts
# 1 хост
ansible -i /root/k3s-ha/inventory.yml zabbix_agents --list-hosts
# 6 хостов

### Этап 5: Playbook установки Zabbix Server + Web
bash
cat > /root/k3s-ha/zabbix-server.yml <<'EOF'
---
- name: Deploy Zabbix Server + Web on managerK3s
  hosts: zabbix_server
  become: true
  vars:
    zabbix_server_version: "5.0"
    zabbix_server_dbhost: "localhost"
    zabbix_server_dbname: "zabbix"
    zabbix_server_dbuser: "zabbix"
    zabbix_server_dbpassword: "ZabbixDBPass123!"
    zabbix_server_dbhost_run_install: true
    zabbix_server_manage_repo: true
    zabbix_web_version: "5.0"
    zabbix_web_httpd: "apache"
    zabbix_web_php_version: "7.4"
    zabbix_web_server_name: "zabbix.example.com"
  roles:
    - community.zabbix.zabbix_server
    - community.zabbix.zabbix_web
EOF
```
Запуск:

```bash
cd /root/k3s-ha
ansible-playbook zabbix-server.yml -i inventory.yml
```
Проверка после установки:

```bash
# Zabbix Server слушает 10051
ss -tlnp | grep 10051

# Web доступен
curl -s -o /dev/null -w "%{http_code}" http://localhost/
# Ожидается: 200 или 302
```
Примечание: роль zabbix_web автоматически установит PHP 7.4 из SCL (centos-release-scl), так как стандартный PHP 5.4 в CentOS 7 не подходит для Zabbix 5.0.

### Этап 6: Zabbix Agent на все хосты
```bash
cat > /root/k3s-ha/zabbix-agents.yml <<'EOF'
---
- name: Deploy Zabbix Agent to all hosts
  hosts: zabbix_agents
  become: true
  vars:
    zabbix_agent_version: "5.0"
    zabbix_agent_server: "10.1.100.10"
    zabbix_agent_serveractive: "10.1.100.10"
    zabbix_agent_manage_repo: true
  roles:
    - community.zabbix.zabbix_agent
EOF
```
Запуск:

```bash
cd /root/k3s-ha
ansible-playbook zabbix-agents.yml -i inventory.yml
```
Проверка на одном из узлов:

```bash
ssh root@10.1.100.27 "systemctl is-active zabbix-agent"

# Ожидается: active
```
### Этап 7: Проверка в веб-интерфейсе
Откройте в браузере: http://10.1.100.10/ (или http://10.1.100.10/zabbix — зависит от конфигурации Apache).

Мастер настройки: Если установка прошла успешно, откроется страница приветствия Zabbix. Пройдите мастер, указав подключение к БД (localhost, zabbix, пароль ZabbixDBPass123!).

Логин: После завершения мастера войдите в систему. Логин по умолчанию — Admin, пароль — zabbix.

Добавление хостов: В разделе Configuration → Hosts вы должны увидеть хосты, которые автоматически добавились (если в ролях включено автообнаружение). Если нет — добавьте вручную, указав IP-адрес агента (порт 10050).

Проверка данных: В разделе Monitoring → Latest data выберите хост и убедитесь, что метрики (CPU, RAM, диск) поступают.

### Итоговый чек-лист
Компонент	|Проверка	|Ожидание
--------- | ------- | -------------
MariaDB	 |mysql -e "SELECT VERSION();"	 |10.5.x
Zabbix Server	 |ss -tlnp | grep 10051	 |LISTEN на 0.0.0.0:10051
Zabbix Web	 |curl -I http://10.1.100.10/	HTTP/1.1  |200 или 302
Zabbix Agent (6 хостов)	 |ansible zabbix_agents -m shell -a "systemctl is-active zabbix-agent"	 |active × 6
Веб-интерфейс	 |Открыть http://10.1.100.10/	 |Мастер настройки или логин
Данные с агентов	 |Monitoring → Latest data	 |Метрики поступают
