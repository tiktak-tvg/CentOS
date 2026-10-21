### Как проверить работу логирования (rsyslog) и резервного копирования (бэкапов)
#### ЧАСТЬ 1. Как проверить логи rsyslog (Централизованный сбор логов)
Архитектура: нода .14 (centpc3) отправляет логи на ноду .12 (centpc1) по порту 514.

##### Шаг 1: Отправляем тестовый лог с клиента (.14)
Заходим на centpca и генерируем сообщение на клиенте:
```bash
ansible 10.1.100.14 -m shell -a "logger 'TEST-RSYSLOG-CHECK'"
```
##### Шаг 2: Ищем сообщение на сервере (.12)
Сервер сохраняет удаленные логи в директорию /var/log/remote/. Ищем там:
```bash
ansible 10.1.100.12 -m shell -a "grep -r 'TEST-RSYSLOG-CHECK' /var/log/remote/"
```
Ожидаемый результат: Вы должны увидеть путь к файлу (например, /var/log/remote/centpc3/root.log) и само сообщение. Это доказывает, что цепочка «клиент → сервер» работает.

##### Шаг 3: Если логов нет, проверяем диагностику
Служба активна?
```bash
ansible servers -m shell -a "systemctl is-active rsyslog"
```
Порт 514 слушается на сервере?
```bash
ansible 10.1.100.12 -m shell -a "ss -ulpn | grep :514; ss -tlpn | grep :514"
```
Порт открыт в firewalld?
```bash
ansible 10.1.100.12 -m shell -a "firewall-cmd --list-ports"
```
#### ЧАСТЬ 2. Как проверить бэкапы
Бэкапы настроены через cron (запуск каждую ночь в 03:30) и bash-скрипт, который пакует конфиги Nginx и сайт в /var/backups/web/.

##### Шаг 1: Проверяем, что задача есть в cron
```bash
ansible servers -m shell -a "crontab -l | grep backup"
```
Ожидаемый результат: строка вида 30 3 * * * /usr/local/bin/backup_web.sh.

##### Шаг 2: Запускаем бэкап вручную (чтобы не ждать ночи)
```bash
ansible servers -m shell -a "/usr/local/bin/backup_web.sh && echo 'BACKUP OK'"
```
Ш##### аг 3: Проверяем, что архив создался
```bash
ansible servers -m shell -a "ls -lh /var/backups/web/"
```
Ожидаемый результат: файл вида web_backup_2026-10-21_12-00.tar.gz.
(Скрипт также настроен на автоудаление архивов старше 7 дней с помощью find ... -mtime +7 -delete).

##### Шаг 4: Проверяем лог выполнения cron (исторический)
```bash
ansible servers -m shell -a "grep 'backup_web' /var/log/cron | tail -5"
```
Итоговые конфиги и файлы:

##### 1. Файл roles/syslog/handlers/main.yml (Создан с нуля):
```yaml
---
- name: restart rsyslog
  service:
    name: rsyslog
    state: restarted
...
```
##### 2. Файл roles/syslog/tasks/main.yml (Переписан):

```yaml
---
- name: Конфигурация Rsyslog как Центрального Сервера (Приемника)
  copy:
    dest: /etc/rsyslog.d/server.conf
    content: |
      module(load="imudp")
      input(type="imudp" port="514")
      module(load="imtcp")
      input(type="imtcp" port="514")
      template(name="RemoteLogs" type="string" string="/var/log/remote/%HOSTNAME%/%PROGRAMNAME%.log")
      *.* ?RemoteLogs
      & stop
  when: inventory_hostname == '10.1.100.12'
  notify: restart rsyslog

- name: Конфигурация Rsyslog как Клиента (Отправителя)
  copy:
    dest: /etc/rsyslog.d/client.conf
    content: |
      *.* @@10.1.100.12:514
  when: inventory_hostname == '10.1.100.14'
  notify: restart rsyslog

- name: Создание директории для удаленных логов (на сервере)
  file:
    path: /var/log/remote
    state: directory
    mode: '0755'
  when: inventory_hostname == '10.1.100.12'

- name: Открыть порты 514/tcp и 514/udp в firewalld (на сервере)
  firewalld:
    port: "{{ item }}"
    permanent: yes
    state: enabled
    immediate: yes
  loop:
    - 514/tcp
    - 514/udp
  when: 
    - inventory_hostname == '10.1.100.12'
    - ansible_os_family == "RedHat"
  notify: restart rsyslog

- name: Проверка синтаксиса конфигурации Rsyslog
  command: rsyslogd -N1
  changed_when: false
  failed_when: false

- name: Перезапуск службы Rsyslog для применения настроек
  service:
    name: rsyslog
    state: restarted
    enabled: yes

- name: Настройка логирования HAProxy в Syslog
  lineinfile:
    path: /etc/rsyslog.d/haproxy.conf
    line: 'local2.*    /var/log/haproxy.log'
    create: yes
  notify: restart rsyslog

- name: Перезапуск HAProxy при обновлении настроек логирования
  service:
    name: haproxy
    state: restarted
...
```
##### 3.2. Роль backup (Бэкапы)
Эта роль изначально работала хорошо, но для полной картины вот её итоговый код.

Файл **roles/backup/tasks/main.yml:**
```yaml
---
- name: Создание директории для хранения локальных бэкапов на сервере
  file:
    path: /var/backups/web
    state: directory
    mode: '0755'

- name: Создание bash-скрипта для упаковки архива
  copy:
    dest: /usr/local/bin/backup_web.sh
    mode: '0755'
    content: |
      #!/bin/bash
      DATE=$(date +%Y-%m-%d_%H-%M)
      tar -czf /var/backups/web/web_backup_$DATE.tar.gz /usr/share/nginx/html /etc/nginx
      find /var/backups/web -type f -mtime +7 -delete

- name: Добавление задачи в планировщик Cron (каждую ночь в 03:30)
  cron:
    name: "Backup web content"
    minute: "30"
    hour: "3"
    job: "/usr/local/bin/backup_web.sh"
...
```
##### 3.3. Краткий список остальных глобальных фиксов (для контекста)
Keepalived: Открыли протокол vrrp в firewalld (устранили split-brain, когда VIP висел на двух нодах одновременно).

HAProxy: Открыли порт 81/tcp и установили socat (без этого healthcheck на web_node2 не проходил, и балансировка не работала). Добавили stats socket и порт 8404 для веб-статистики.

Docker (deploy_app):

Удалили конфликтующий старый docker-py.

Зафиксировали версии requests<2.28 и urllib3<1.27 (чтобы не падало на Python 2.7).

Добавили volumes для монтирования шаблона Jinja2 внутрь контейнера и recreate: yes.

Мониторинг: Добавили теги monitoring в задачи, чтобы работал запуск --tags monitoring. Настроили Prometheus на порт 9091, чтобы избежать конфликта с Cockpit (порт 9090).

Теперь система полностью автоматизирована. При следующем запуске ansible-playbook /etc/ansible/site.yml все эти конфиги применятся с нуля без ручного вмешательства.
