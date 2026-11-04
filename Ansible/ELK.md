### Централизованный сбор логов (ELK Stack): 
- Промышленная замена вашему rsyslog 
- Ansible развернет Elasticsearch (хранение и поиск) на одном или нескольких узлах 
- Logstash (обработка) 
- Kibana (визуализация) на других
- Filebeat установит на все хосты для отправки логов.

> [!TOP]
> Лучше для ELK использовать управляющий сервер managerK3s (10.1.100.10)

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
### Итоговое сравнение
Критерий	ELK на .24	ELK на .10 (managerK3s)
Изоляция от кластера K3s	❌ Конкурирует с etcd за I/O	✅ Кластер не затронут
Стабильность etcd	⚠️ Риск перевыбора лидера	✅ Не влияет
RAM/CPU на .24	Отнимается у control-plane	—
RAM/CPU на .10	—	Отнимается у ansible/kubectl
Если узел упадёт	Потеря логи + деградация кластера	Потеря только логов
Требования	≥ 8 ГБ RAM на .24	≥ 8 ГБ RAM на .10
Типичный случай	Маленький стенд, один сервер	Production с выделенным managed-узлом
