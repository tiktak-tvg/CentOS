### Централизованный сбор логов (ELK Stack): 
- Промышленная замена вашему rsyslog 
- Ansible развернет Elasticsearch (хранение и поиск) на одном или нескольких узлах 
- Logstash (обработка) 
- Kibana (визуализация) на других
- Filebeat установит на все хосты для отправки логов.

Структура:

Этап 0: Проверка требований

Этап 1: Подготовка (Java, SELinux, firewalld)

Этап 2: Inventory

Этап 3: Elasticsearch (RPM с Aliyun + настройка)

Этап 4: Logstash (RPM + pipeline)

Этап 5: Kibana (RPM + 0.0.0.0)

Этап 6: Filebeat на 6 хостов (RPM копирование через Ansible)

Этап 7: ILM + шаблон

Этап 8: Проверки

Этап 9: Kibana UI (Index Pattern)

> [!TIP]
> Лучше для ELK использовать управляющий сервер managerK3s (10.1.100.10)

- Узел ELK: managerK3s (10.1.100.10) — на нём будут Elasticsearch, Logstash, Kibana.
- Filebeat-клиенты: 10.1.100.10, 10.1.100.24, 10.1.100.25, 10.1.100.26, 10.1.100.27, 10.1.100.28 (6 хостов).
- Требования к managerK3s: ≥ 8 ГБ RAM (минимум 4 ГБ с heap 1g), ≥ 50 ГБ свободного диска.
- SELinux: на время установки переводится в Permissive, для production — настройка политик.
- Firewalld: открыты порты 9200, 9300, 5044, 5601.

Если у managerK3s меньше 4 ГБ RAM — ELK туда не влезет (даже с heap: 1g будет тесно). В этом случае лучше оставить на .24 или выделить отдельный узел. Если 8+ ГБ — смело ставьте на 10.1.100.10.

### Полное руководство: ELK Stack на managerK3s с Filebeat на 6 хостах
### Этап 0: Проверка требований на managerK3s
```bash
free -h      # ≥ 8 ГБ RAM (минимум 4 ГБ с heap 1g)
df -h /      # ≥ 50 ГБ свободно
nproc        # ≥ 2 ядра
```
Если RAM < 4 ГБ — уменьшите heap.options до -Xms1g/-Xmx1g. Если < 2 ГБ — ELK не влезет, ставьте на .24.

### Этап 1: Подготовка managerK3s
#### 1.1. Java 11 + SELinux + firewalld
```bash
cat > /tmp/elk-prep.sh <<'EOF'
#!/bin/bash
set -e

# Java 11
yum install -y java-11-openjdk-devel
java -version

# SELinux → Permissive (для теста; production — политики через audit2allow)
setenforce 0
getenforce

# Firewalld
firewall-cmd --permanent --add-port=9200/tcp
firewall-cmd --permanent --add-port=9300/tcp
firewall-cmd --permanent --add-port=5044/tcp
firewall-cmd --permanent --add-port=5601/tcp
firewall-cmd --reload
firewall-cmd --list-ports

echo "=== Preparation OK ==="
EOF

bash /tmp/elk-prep.sh
```
#### 1.2. Очистка предыдущих установок (если были)
```bash
cat > /tmp/elk-cleanup.sh <<'EOF'
#!/bin/bash
for pkg in elasticsearch logstash kibana filebeat; do
  systemctl stop $pkg 2>/dev/null || true
  yum remove -y $pkg 2>/dev/null || true
done
rm -rf /var/lib/elasticsearch /var/lib/logstash /var/lib/kibana /var/lib/filebeat
rm -rf /etc/elasticsearch /etc/logstash /etc/kibana /etc/filebeat
echo "=== Cleanup OK ==="
EOF

bash /tmp/elk-cleanup.sh
```
### Этап 2: Inventory
```bash
cat >> /root/k3s-ha/inventory.yml <<'EOF'

    # === ELK Stack (все компоненты на managerK3s) ===
    elk:
      hosts:
        10.1.100.10:
          ansible_connection: local

    # === Filebeat-клиенты (все 6 хостов, включая managerK3s) ===
    filebeat_clients:
      hosts:
        10.1.100.10:
          ansible_connection: local
        10.1.100.24:
        10.1.100.25:
        10.1.100.26:
        10.1.100.27:
        10.1.100.28:
EOF
```
##### Проверка:
```bash
ansible -i /root/k3s-ha/inventory.yml elk --list-hosts
# hosts (1): 10.1.100.10

ansible -i /root/k3s-ha/inventory.yml filebeat_clients --list-hosts
# hosts (6)

ansible -i /root/k3s-ha/inventory.yml elk -m ping
# SUCCESS => "ping": "pong"
```
### Этап 3: Скачивание RPM с Aliyun (GPG + 4 пакета)
```bash
cat > /tmp/elk-predownload.sh <<'EOF'
#!/bin/bash
set -e

# GPG-ключ Elasticsearch
wget -q https://mirrors.aliyun.com/elasticstack/GPG-KEY-elasticsearch -O /tmp/GPG-KEY-elasticsearch
rpm --import /tmp/GPG-KEY-elasticsearch

# 4 RPM
for pkg in elasticsearch logstash kibana filebeat; do
  echo "Downloading ${pkg}..."
  wget -q "https://mirrors.aliyun.com/elasticstack/7.x/yum/7.17.29/${pkg}-7.17.29-x86_64.rpm" \
    -O "/tmp/${pkg}-7.17.29.rpm"
done

ls -lh /tmp/*.rpm /tmp/GPG-KEY-elasticsearch
echo "=== Predownload OK ==="
EOF

bash /tmp/elk-predownload.sh
```
> [!Warning]
> официальный artifacts.elastic.co блокирует запросы из РФ (403). Aliyun — проверенно работает.

### Этап 4: Elasticsearch 7.17.29
```bash
cat > /tmp/elk-elasticsearch.sh <<'EOF'
#!/bin/bash
set -e

# 1. Установка из локального RPM
yum localinstall -y /tmp/elasticsearch-7.17.29.rpm

# 2. JVM heap (Xms ДОЛЖЕН быть равен Xmx, иначе bootstrap check падает)
mkdir -p /etc/elasticsearch/jvm.options.d
cat > /etc/elasticsearch/jvm.options.d/heap.options <<HEAP
-Xms2g
-Xmx2g
HEAP

# 3. discovery.type: single-node (для одиночного узла)
grep -q '^discovery.type:' /etc/elasticsearch/elasticsearch.yml || \
  echo 'discovery.type: single-node' >> /etc/elasticsearch/elasticsearch.yml

# 4. Запуск
systemctl daemon-reload
systemctl enable --now elasticsearch
sleep 20
systemctl is-active elasticsearch

echo "=== Elasticsearch OK ==="
EOF

bash /tmp/elk-elasticsearch.sh

# Проверка
curl -s http://127.0.0.1:9200 | jq '{cluster_name, version: .version.number, tagline}'

# Ожидается: cluster_name: "elasticsearch", version: "7.17.29"
```
Если heap 2g не влезает (мало RAM) — замените 2g на 1g в /etc/elasticsearch/jvm.options.d/heap.options и systemctl restart elasticsearch.

### Этап 5: Logstash 7.17.29
```bash
cat > /tmp/elk-logstash.sh <<'EOF'
#!/bin/bash
set -e

# 1. Установка
yum localinstall -y /tmp/logstash-7.17.29.rpm

# 2. Pipeline: Beats input → Elasticsearch output
# ВАЖНО: имя индекса фиксировано "filebeat-7.17.29-*" — %{[agent.version]} в Logstash 7.x не раскрывается
cat > /etc/logstash/conf.d/beats.conf <<CONF
input {
  beats {
    port => 5044
  }
}

output {
  elasticsearch {
    hosts => ["http://127.0.0.1:9200"]
    index => "filebeat-7.17.29-%{+YYYY.MM.dd}"
  }
}
CONF

# 3. Проверка конфига (из /tmp — иначе ENOENT /root/k3s-ha)
cd /tmp
sudo -u logstash /usr/share/logstash/bin/logstash \
  --path.settings /etc/logstash -t -f /etc/logstash/conf.d/beats.conf 2>&1 | tail -3

# 4. Запуск
systemctl enable --now logstash
sleep 40
systemctl is-active logstash

echo "=== Logstash OK ==="
EOF

bash /tmp/elk-logstash.sh

# Проверка
ss -tlnp | grep 5044

# LISTEN на *:5044 или [::]:5044
```
Если Logstash не слушает 5044 через 60 секунд — проверьте journalctl -u logstash -n 30. Часто нужно уменьшить heap в /etc/logstash/jvm.options: -Xms512m/-Xmx512m.

### Этап 6: Kibana 7.17.29
```bash
cat > /tmp/elk-kibana.sh <<'EOF'
#!/bin/bash
set -e

# 1. Установка
yum localinstall -y /tmp/kibana-7.17.29.rpm

# 2. server.host: 0.0.0.0 — иначе слушает только localhost
sed -i 's/^#\?server\.host:.*/server.host: "0.0.0.0"/' /etc/kibana/kibana.yml
grep '^server.host' /etc/kibana/kibana.yml

# 3. Явно указать ES
grep -q '^elasticsearch.hosts:' /etc/kibana/kibana.yml || \
  echo 'elasticsearch.hosts: ["http://127.0.0.1:9200"]' >> /etc/kibana/kibana.yml

# 4. Запуск
systemctl enable --now kibana
sleep 20
systemctl is-active kibana

echo "=== Kibana OK ==="
EOF

bash /tmp/elk-kibana.sh

# Проверка
ss -tlnp | grep 5601

# LISTEN на 0.0.0.0:5601

curl -s -o /dev/null -w "%{http_code}\n" http://10.1.100.10:5601

# 302
```
### Этап 7: Filebeat на 6 хостах
```bash
cat > /root/k3s-ha/filebeat-deploy.yml <<'EOF'
---
- name: Deploy Filebeat to all nodes
  hosts: filebeat_clients
  become: true
  tasks:
    - name: Copy Filebeat RPM
      copy:
        src: /tmp/filebeat-7.17.29.rpm
        dest: /tmp/filebeat-7.17.29.rpm
        mode: '0644'

    - name: Install Filebeat from local RPM
      yum:
        name: /tmp/filebeat-7.17.29.rpm
        state: present

    - name: Deploy filebeat.yml
      copy:
        dest: /etc/filebeat/filebeat.yml
        mode: '0600'
        content: |
          filebeat.inputs:
            - type: log
              enabled: true
              paths:
                - /var/log/messages
                - /var/log/secure
                - /var/log/*.log
            - type: container
              enabled: true
              paths:
                - /var/log/containers/*.log
              processors:
                - add_kubernetes_metadata: ~

          output.logstash:
            hosts: ["10.1.100.10:5044"]

          logging.level: info
          logging.to_files: true
          logging.files:
            path: /var/log/filebeat
            name: filebeat
            keepfiles: 7
            permissions: 0644

    - name: Restart Filebeat
      systemd:
        name: filebeat
        state: restarted
        enabled: true

    - name: Wait for Filebeat to start
      wait_for:
        timeout: 5
EOF

ansible-playbook -i /root/k3s-ha/inventory.yml /root/k3s-ha/filebeat-deploy.yml
```
##### Проверка:
```bash
ansible -i /root/k3s-ha/inventory.yml filebeat_clients -m shell -a "systemctl is-active filebeat"
# active × 6

ansible -i /root/k3s-ha/inventory.yml filebeat_clients -m shell -a "filebeat test output"
# logstash: 10.1.100.10:5044... OK
```
### Этап 8: ILM (7 дней) + Index Template + проверка
```bash
cat > /tmp/ilm-setup.sh <<'EOF'
#!/bin/bash
set -e

# 1. Удалить старую политику (если была)
curl -s -X DELETE "http://127.0.0.1:9200/_ilm/policy/filebeat-policy" >/dev/null || true

# 2. Создать delete-only политику (без rollover — для daily-индексов)
curl -s -X PUT "http://127.0.0.1:9200/_ilm/policy/filebeat-policy" \
  -H 'Content-Type: application/json' -d'
{
  "policy": {
    "phases": {
      "delete": {
        "min_age": "7d",
        "actions": { "delete": {} }
      }
    }
  }
}'

# 3. Index template для новых filebeat-* индексов
curl -s -X PUT "http://127.0.0.1:9200/_index_template/filebeat" \
  -H 'Content-Type: application/json' -d'
{
  "index_patterns": ["filebeat-*"],
  "template": {
    "settings": {
      "number_of_replicas": 0,
      "index.lifecycle.name": "filebeat-policy"
    }
  }
}'

# 4. Применить ILM к уже существующим индексам
curl -s -X PUT "http://127.0.0.1:9200/filebeat-*/_settings" \
  -H 'Content-Type: application/json' -d'
{
  "index": {
    "lifecycle": { "name": "filebeat-policy" },
    "number_of_replicas": 0
  }
}'

echo "=== ILM + Template OK ==="
EOF

bash /tmp/ilm-setup.sh
```
##### Проверка:
```bash
# ILM policy
curl -s http://10.1.100.10:9200/_ilm/policy/filebeat-policy \
  | jq '.["filebeat-policy"].policy.phases'
# {"delete": {"min_age": "7d", "actions": {"delete": {}}}}

# Index template
curl -s http://10.1.100.10:9200/_index_template/filebeat \
  | jq '.index_templates[0].index_template.template.settings'
# {"index": {"lifecycle": {"name": "filebeat-policy"}, "number_of_replicas": "0"}}

# ILM explain (для существующего индекса)
curl -s "http://10.1.100.10:9200/filebeat-7.17.29-*/_ilm/explain" \
  | jq '.["indices"][keys[0]] | {policy: .policy, phase: .phase, step: .step}'
# {"policy": "filebeat-policy", "phase": "new", "step": "complete"}
```
### Этап 9: Финальная проверка + Kibana UI
#### 9.1. Проверка сервисов и индексов
```bash
cat > /tmp/elk-final-check.sh <<'EOF'
#!/bin/bash
echo "=== Elasticsearch ==="
curl -s http://10.1.100.10:9200 | jq '{cluster_name, version: .version.number}'

echo -e "\n=== Logstash (5044) ==="
ss -tlnp | grep 5044 || echo "NOT LISTENING"

echo -e "\n=== Kibana (5601) ==="
ss -tlnp | grep 5601 || echo "NOT LISTENING"

echo -e "\n=== Indices ==="
curl -s "http://10.1.100.10:9200/_cat/indices?v" | grep filebeat

echo -e "\n=== Filebeat status on all nodes ==="
EOF

bash /tmp/elk-final-check.sh

ansible -i /root/k3s-ha/inventory.yml filebeat_clients -m shell -a "systemctl is-active filebeat"

# Ожидается:
# Elasticsearch: cluster_name: "elasticsearch", version: "7.17.29"
# Logstash: LISTEN на 5044
# Kibana: LISTEN на 5601
# Индекс: green open filebeat-7.17.29-2026.11.04 ...
# Filebeat: active × 6
```
#### 9.2. Kibana Index Pattern (в браузере)
Откройте http://10.1.100.10:5601

Stack Management → Index Patterns → Create index pattern

Pattern: filebeat-*

Time field: @timestamp

Save

Discover → выберите filebeat-* → Last 15 minutes → видны логи с 6 хостов

### Итоговый чек-лист

Проверка	|Команда	|Ожидание
--------- | ------- | -------------
Java 11	 |java -version	 |11.0.x
SELinux	 |getenforce	 |Permissive
Firewalld	 |firewall-cmd --list-ports	 |9200/tcp 9300/tcp 5044/tcp 5601/tcp
Elasticsearch	 |curl http://10.1.100.10:9200	 |JSON, cluster_name
Logstash	 |ss -tlnp | grep 5044	 |LISTEN
Kibana	 |curl -I http://10.1.100.10:5601	 |302
Filebeat × 6	 |systemctl is-active filebeat	 |active × 6
Filebeat → Logstash	 |filebeat test output	 |OK
Индекс	 |curl .../_cat/indices | grep filebeat	 |green, filebeat-7.17.29-*
ILM policy	 |curl .../_ilm/policy/filebeat-policy	 |delete через 7d
Index template	 |curl .../_index_template/filebeat	 |replicas: 0, lifecycle
ILM explain	 |curl .../_ilm/explain	 |policy: filebeat-policy, phase: new
Kibana pattern	 |UI	 |filebeat-* создан
