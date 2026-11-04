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

##### Что нужно проверить на managerK3s
```bash
# RAM — минимум 4 ГБ, комфортно 8+ ГБ
free -h

# Диск — минимум 50 ГБ свободно
df -h /

# CPU — 2+ ядра
nproc
```
Если у managerK3s меньше 4 ГБ RAM — ELK туда не влезет (даже с heap: 1g будет тесно). В этом случае лучше оставить на .24 или выделить отдельный узел. Если 8+ ГБ — смело ставьте на 10.1.100.10.

### Этап 1: Подготовка managerK3s
#### 1.1. Установка Java 11 (OpenJDK) — обязательна для Elasticsearch, Logstash, Kibana.
```bash
yum install -y java-11-openjdk-devel
java -version
# Ожидается: openjdk version "11.0.x"
```
#### 1.2. Отключение SELinux (на время теста) — чтобы избежать проблем с правами.
```bash
setenforce 0
getenforce
# Ожидается: Permissive
```
#### 1.3. Настройка firewalld — открываем порты ELK.
```bash
cat > /tmp/elk-firewall.sh <<'EOF'
#!/bin/bash
firewall-cmd --permanent --add-port=9200/tcp
firewall-cmd --permanent --add-port=9300/tcp
firewall-cmd --permanent --add-port=5044/tcp
firewall-cmd --permanent --add-port=5601/tcp
firewall-cmd --reload
EOF

bash /tmp/elk-firewall.sh
firewall-cmd --list-ports

# Ожидается: 9200/tcp 9300/tcp 5044/tcp 5601/tcp
```
#### 1.4. Очистка предыдущих установок (если есть) — чтобы начать с чистого листа.
```bash
# Остановить и удалить пакеты (если установлены)
for pkg in elasticsearch logstash kibana filebeat; do
  systemctl stop $pkg 2>/dev/null
  yum remove -y $pkg 2>/dev/null
done

# Удалить данные и конфиги
rm -rf /var/lib/elasticsearch /var/lib/logstash /var/lib/kibana /var/lib/filebeat
rm -rf /etc/elasticsearch /etc/logstash /etc/kibana /etc/filebeat
```
### Этап 2: Установка Elasticsearch 7.17.29 (RPM с Aliyun)
#### 2.1. Скачивание RPM (официальный репозиторий блокирует запросы, используем зеркало Aliyun).
```bash
wget https://mirrors.aliyun.com/elasticstack/7.x/yum/7.17.29/elasticsearch-7.17.29-x86_64.rpm -O /tmp/elasticsearch-7.17.29.rpm
ls -lh /tmp/elasticsearch-7.17.29.rpm

# Ожидается: ~310 МБ
```
#### 2.2. Установка:
```bash
yum localinstall -y /tmp/elasticsearch-7.17.29.rpm
```
#### 2.3. Настройка JVM heap (обязательно Xms = Xmx, иначе bootstrap check не пройдёт):
```bash
mkdir -p /etc/elasticsearch/jvm.options.d
cat > /etc/elasticsearch/jvm.options.d/heap.options <<'EOF'
-Xms2g
-Xmx2g
EOF
```
Если на узле меньше 8 ГБ RAM, уменьшите до 1g.

#### 2.4. Настройка discovery.type: single-node (для одиночного узла):
```bash
grep -q '^discovery.type:' /etc/elasticsearch/elasticsearch.yml || \
  echo 'discovery.type: single-node' >> /etc/elasticsearch/elasticsearch.yml
```
#### 2.5. Запуск и проверка:
```bash
systemctl daemon-reload
systemctl enable --now elasticsearch
sleep 20
systemctl status elasticsearch --no-pager
curl -s http://127.0.0.1:9200 | jq .cluster_name

# Ожидается: "elasticsearch" (или заданное имя)
```
### Этап 3: Установка Logstash 7.17.29
#### 3.1. Скачивание RPM:
```bash
wget https://mirrors.aliyun.com/elasticstack/7.x/yum/7.17.29/logstash-7.17.29-x86_64.rpm -O /tmp/logstash-7.17.29.rpm
```
#### 3.2. Установка:
```bash
yum localinstall -y /tmp/logstash-7.17.29.rpm
```
#### 3.3. Настройка pipeline (приём Beats → отправка в Elasticsearch):
```bash
cat > /etc/logstash/conf.d/beats.conf <<'EOF'
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
EOF
```
Важно: имя индекса фиксировано (filebeat-7.17.29-...), потому что %{[agent.version]} в Logstash 7.x не раскрывается автоматически.

#### 3.4. Проверка конфига и запуск:
```bash
cd /tmp
sudo -u logstash /usr/share/logstash/bin/logstash --path.settings /etc/logstash -t -f /etc/logstash/conf.d/beats.conf 2>&1 | tail -3

# Ожидается: Config Validation Result: OK

systemctl enable --now logstash
sleep 40
systemctl is-active logstash
ss -tlnp | grep 5044

# Ожидается: LISTEN на 0.0.0.0:5044 или [::]:5044
```
### Этап 4: Установка Kibana 7.17.29
#### 4.1. Скачивание RPM:

```bash
wget https://mirrors.aliyun.com/elasticstack/7.x/yum/7.17.29/kibana-7.17.29-x86_64.rpm -O /tmp/kibana-7.17.29.rpm
```
#### 4.2. Установка:

```bash
yum localinstall -y /tmp/kibana-7.17.29.rpm
```
#### 4.3. Настройка server.host: "0.0.0.0" (чтобы Kibana была доступна снаружи):

```bash
sed -i 's/^#\?server\.host:.*/server.host: "0.0.0.0"/' /etc/kibana/kibana.yml
grep '^server.host' /etc/kibana/kibana.yml

# Ожидается: server.host: "0.0.0.0"
```
#### 4.4. Запуск и проверка:

```bash
systemctl enable --now kibana
sleep 20
systemctl is-active kibana
ss -tlnp | grep 5601

# Ожидается: LISTEN на 0.0.0.0:5601

curl -s -o /dev/null -w "%{http_code}\n" http://10.1.100.10:5601

# Ожидается: 302
```
### Этап 5: Установка Filebeat на 6 хостов
#### 5.1. Скачивание RPM на managerK3s:

```bash
wget https://mirrors.aliyun.com/elasticstack/7.x/yum/7.17.29/filebeat-7.17.29-x86_64.rpm -O /tmp/filebeat-7.17.29.rpm
```
#### 5.2. Копирование RPM на все узлы (используем Ansible, группы filebeat_clients):

```bash
cat > /root/k3s-ha/filebeat-install.yml <<'EOF'
---
- name: Install Filebeat RPM on all nodes
  hosts: filebeat_clients
  become: true
  tasks:
    - name: Copy Filebeat RPM
      copy:
        src: /tmp/filebeat-7.17.29.rpm
        dest: /tmp/filebeat-7.17.29.rpm
        mode: '0644'

    - name: Install Filebeat
      yum:
        name: /tmp/filebeat-7.17.29.rpm
        state: present
EOF

ansible-playbook -i /root/k3s-ha/inventory.yml /root/k3s-ha/filebeat-install.yml
```
#### 5.3. Настройка Filebeat (отправка в Logstash на 10.1.100.10:5044):

```bash
cat > /root/k3s-ha/filebeat-config.yml <<'EOF'
---
- name: Configure Filebeat
  hosts: filebeat_clients
  become: true
  tasks:
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
EOF

ansible-playbook -i /root/k3s-ha/inventory.yml /root/k3s-ha/filebeat-config.yml
```
#### 5.4. Запуск Filebeat на всех узлах:

```bash
ansible -i /root/k3s-ha/inventory.yml filebeat_clients -m systemd -a "name=filebeat state=restarted enabled=yes" -b
sleep 10
ansible -i /root/k3s-ha/inventory.yml filebeat_clients -m shell -a "systemctl is-active filebeat"

# Ожидается: active × 6
```
#### 5.5. Проверка связи с Logstash:

```bash
ansible -i /root/k3s-ha/inventory.yml filebeat_clients -m shell -a "filebeat test output"

# Ожидается: logstash: 10.1.100.10:5044... OK
```
### Этап 6: Настройка ILM (7 дней хранения) и Index Template
#### 6.1. Создание ILM-политики (только удаление через 7 дней):

```bash
cat > /tmp/ilm-setup.sh <<'EOF'
#!/bin/bash

# Удалить старую политику (если была)
curl -s -X DELETE "http://127.0.0.1:9200/_ilm/policy/filebeat-policy" >/dev/null

# Создать delete-only политику
curl -s -X PUT "http://127.0.0.1:9200/_ilm/policy/filebeat-policy" -H 'Content-Type: application/json' -d'
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

# Создать index template (replicas=0, lifecycle)
curl -s -X PUT "http://127.0.0.1:9200/_index_template/filebeat" -H 'Content-Type: application/json' -d'
{
  "index_patterns": ["filebeat-*"],
  "template": {
    "settings": {
      "number_of_replicas": 0,
      "index.lifecycle.name": "filebeat-policy"
    }
  }
}'

echo "ILM policy + template созданы"
EOF

bash /tmp/ilm-setup.sh
```
#### 6.2. Применение ILM к уже существующим индексам (если они есть):

```bash
curl -X PUT "http://10.1.100.10:9200/filebeat-*/_settings" -H 'Content-Type: application/json' -d'
{
  "index": {
    "lifecycle": { "name": "filebeat-policy" }
  }
}'
```
#### 6.3. Проверка применения ILM:

```bash
# Проверить настройки индекса
curl -s "http://10.1.100.10:9200/filebeat-7.17.29-*/_settings" \
  | jq 'to_entries[0].value.settings.index | {replicas: .number_of_replicas, lifecycle: .lifecycle.name}'
# Ожидается: replicas: "0", lifecycle: "filebeat-policy"

# Проверить ILM explain
curl -s "http://10.1.100.10:9200/filebeat-7.17.29-*/_ilm/explain" \
  | jq '.["indices"][keys[0]] | {policy: .policy, phase: .phase, step: .step}'
# Ожидается: policy: "filebeat-policy", phase: "new", step: "complete"
```



### Шаг 1: Inventory
Добавьте в /root/k3s-ha/inventory.yml группы elk и filebeat_clients (в конец, внутри children):
```bash
cat >> /root/k3s-ha/inventory.yml <<'EOF'

    # === ELK Stack (все компоненты на managerK3s) ===
    elk:
      hosts:
        10.1.100.10:

    # === Filebeat-клиенты (все 5 узлов K3s; managerK3s уже в elk) ===
    filebeat_clients:
      hosts:
        10.1.100.24:
        10.1.100.25:
        10.1.100.26:
        10.1.100.27:
        10.1.100.28:
EOF
```
Важно: 10.1.100.10 есть только в elk, не в filebeat_clients — иначе роль Filebeat попытается установиться на managerK3s дважды (в рамках elk и filebeat_clients). Впрочем, если вы хотите логи и с managerK3s — добавьте его и в filebeat_clients, роль идемпотентна.

Проверка:
```bash
ansible -i /root/k3s-ha/inventory.yml elk --list-hosts
# 10.1.100.10
ansible -i /root/k3s-ha/inventory.yml filebeat_clients --list-hosts
# 5 хостов
```
### Шаг 2: Firewalld на managerK3s
На managerK3s тоже есть firewalld — откройте порты:
```bash
cat > /tmp/elk-firewall.sh <<'EOF'
#!/bin/bash
firewall-cmd --permanent --add-port=9200/tcp
firewall-cmd --permanent --add-port=9300/tcp
firewall-cmd --permanent --add-port=5044/tcp
firewall-cmd --permanent --add-port=5601/tcp
firewall-cmd --reload
EOF

ansible -i /root/k3s-ha/inventory.yml elk -m script -a "/tmp/elk-firewall.sh" -b
```
SELinux (на время теста, для production см. замечание ниже):
```bash
ansible -i /root/k3s-ha/inventory.yml elk -m shell -a "setenforce 0" -b
```
### Шаг 3: Playbook ELK (всё на managerK3s)
Логика та же, что и раньше, но теперь kibana_elasticsearch_url и logstash указывают на 127.0.0.1 — потому что всё локально.
```bash
cat > /root/k3s-ha/elk-single.yml <<'EOF'
---
# Elasticsearch
- name: Deploy Elasticsearch
  hosts: elk
  become: true
  vars:
    elasticsearch_version: "7.x"
    elasticsearch_cluster_name: "k3s-logs"
    elasticsearch_network_host: "0.0.0.0"
    elasticsearch_http_port: 9200
    elasticsearch_heap_size: "2g"
    elasticsearch_config:
      cluster.name: "k3s-logs"
      network.host: "0.0.0.0"
      http.port: 9200
      discovery.type: single-node
      bootstrap.memory_lock: true
  roles:
    - geerlingguy.java
    - geerlingguy.elasticsearch

# Logstash
- name: Deploy Logstash
  hosts: elk
  become: true
  vars:
    logstash_version: "7.x"
    logstash_config: |
      input {
        beats {
          port => 5044
        }
      }
      output {
        elasticsearch {
          hosts => ["http://127.0.0.1:9200"]
          index => "filebeat-%{[agent.version]}-%{+YYYY.MM.dd}"
        }
      }
  roles:
    - geerlingguy.java
    - geerlingguy.logstash

# Kibana
- name: Deploy Kibana
  hosts: elk
  become: true
  vars:
    kibana_version: "7.x"
    kibana_server_host: "0.0.0.0"
    kibana_server_port: 5601
    kibana_elasticsearch_url: "http://127.0.0.1:9200"
  roles:
    - geerlingguy.kibana
EOF
```
Запуск:
```bash
cd /root/k3s-ha
ansible-playbook elk-single.yml -i inventory.yml
```
Проверка:
```bash
curl -s http://10.1.100.10:9200 | jq .cluster_name
# "k3s-logs"

ss -tlnp | grep -E '5044|5601'
# LISTEN на 5044 и 5601
```
### Шаг 4: Filebeat на 5 узлов K3s
Указываем Logstash на managerK3s (10.1.100.10:5044):
```bash
cat > /root/k3s-ha/filebeat-all.yml <<'EOF'
---
- name: Deploy Filebeat to all K3s nodes
  hosts: filebeat_clients
  become: true
  vars:
    filebeat_version: "7.x"
    filebeat_output_logstash_enabled: true
    filebeat_output_logstash_hosts:
      - "10.1.100.10:5044"
    filebeat_inputs:
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
  roles:
    - geerlingguy.filebeat
EOF

cd /root/k3s-ha
ansible-playbook filebeat-all.yml -i inventory.yml
```
Отдельно: если хотите собирать логи и с managerK3s (самого себя), добавьте 10.1.100.10 в filebeat_clients в inventory и перезапустите filebeat-all.yml. Роль просто доустановит Filebeat на .10 поверх уже установленного Java (это конфликтов не вызовет).

### Шаг 5: Проверка
```bash
sleep 30
curl -s "http://10.1.100.10:9200/_cat/indices?v" | grep filebeat
# Ожидается строка filebeat-7.x-YYYY.MM.DD
Kibana: http://10.1.100.10:5601 — открывается UI.
```
### Шаг 6: ILM политика (7 дней хранения)
```bash
cat > /tmp/ilm-setup.sh <<'EOF'
#!/bin/bash
curl -s -X PUT "http://127.0.0.1:9200/_ilm/policy/filebeat-policy" -H 'Content-Type: application/json' -d'
{
  "policy": {
    "phases": {
      "hot":    { "actions": { "rollover": { "max_size": "5gb", "max_age": "1d" } } },
      "delete": { "min_age": "7d", "actions": { "delete": {} } }
    }
  }
}'
EOF

ansible -i /root/k3s-ha/inventory.yml elk -m script -a "/tmp/ilm-setup.sh"
```

