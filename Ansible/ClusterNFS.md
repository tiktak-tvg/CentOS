### Кластер хранения (NFS) 
Один узел выступает в роли NFS-сервера (раздает директорию)<br>
Второй — в роли клиента (монтирует ее). Ansible-роль может управлять как серверной частью (/etc/exports), так и клиентской (записи в /etc/fstab)<br>
Роль управляющего хоста -  10.1.100.10<br>
Кластер хранения (NFS) на узлах 10.1.100.21 (Server) и 10.1.100.22 (Client).<br>
#### Шаг 1: Настройка инвентаря и переменных
Добавим новую группу хостов и переменные для NFS.
```bash
# 1. Добавляем группу в инвентарь
cat <<'EOF' >> /etc/ansible/hosts

[nfs_cluster]
10.1.100.21 nfs_role=server
10.1.100.22 nfs_role=client
EOF
```
# 2. Создаем файл переменных
```bash
cat <<'EOF' > /etc/ansible/group_vars/nfs_cluster.yml
---
nfs_export_dir: "/srv/nfs_share"
nfs_mount_dir: "/mnt/nfs_share"
EOF
```
#### Шаг 2: Создание ролей
Создадим структуру для двух новых ролей: nfs_server и nfs_client.
```bash
mkdir -p /etc/ansible/roles/nfs_server/tasks
mkdir -p /etc/ansible/roles/nfs_client/tasks
```
##### 1. Роль nfs_server (Настройка сервера)
```bash
cat <<'EOF' > /etc/ansible/roles/nfs_server/tasks/main.yml
---
- name: Установка NFS Server и libsemanage-python (для SELinux)
  yum:
    name:
      - nfs-utils
      - libsemanage-python
    state: present

- name: Создание директории для экспорта
  file:
    path: "{{ nfs_export_dir }}"
    state: directory
    mode: '0777'
    owner: nobody
    group: nobody

- name: Настройка /etc/exports
  lineinfile:
    path: /etc/exports
    line: "{{ nfs_export_dir }} {{ hostvars[groups['nfs_cluster'][1]].ansible_default_ipv4.address }}(rw,sync,no_subtree_check,no_root_squash)"
    state: present
  notify: restart nfs-server

- name: Включение и запуск службы NFS
  service:
    name: nfs-server
    state: started
    enabled: yes

- name: Настройка Firewalld (NFS, RPC, Mountd)
  firewalld:
    service: "{{ item }}"
    permanent: yes
    state: enabled
    immediate: yes
  loop:
    - nfs
    - rpc-bind
    - mountd

- name: Настройка SELinux для NFS (разрешить экспорт и чтение/запись)
  seboolean:
    name: "{{ item }}"
    state: yes
    persistent: yes
  loop:
    - nfs_export_all_rw
    - nfs_export_all_ro
EOF
```
(Примечание: если папки handlers нет, создайте её: mkdir -p /etc/ansible/roles/nfs_server/handlers)

##### 2. Роль nfs_client (Настройка клиента)
```bash
cat <<'EOF' > /etc/ansible/roles/nfs_client/tasks/main.yml
---
- name: Установка NFS Client (nfs-utils)
  yum:
    name: nfs-utils
    state: present

- name: Создание точки монтирования на клиенте
  file:
    path: "{{ nfs_mount_dir }}"
    state: directory
    mode: '0777'

- name: Монтирование NFS-шары и добавление в /etc/fstab
  mount:
    path: "{{ nfs_mount_dir }}"
    src: "{{ hostvars[groups['nfs_cluster'][0]].ansible_default_ipv4.address }}:{{ nfs_export_dir }}"
    fstype: nfs
    opts: defaults,_netdev
    state: mounted
EOF
```
#### Шаг 3: Главный плейбук
Создадим плейбук, который сначала настраивает сервер, а затем клиент (чтобы клиент сразу мог смонтировать готовую шару).
```bash
cat <<'EOF' > /etc/ansible/nfs_cluster.yml
---
- name: Настройка NFS Сервера
  hosts: nfs_cluster
  become: yes
  gather_facts: yes
  roles:
    - role: nfs_server
      when: nfs_role == 'server'

- name: Настройка NFS Клиента
  hosts: nfs_cluster
  become: yes
  gather_facts: yes
  roles:
    - role: nfs_client
      when: nfs_role == 'client'
EOF
```
#### Шаг 4: Запуск и проверка
Запускаем плейбук:
```bash
ansible-playbook /etc/ansible/nfs_cluster.yml
```
Ожидается: PLAY RECAP с failed=0 на обоих хостах.

<img width="1108" height="63" alt="image" src="https://github.com/user-attachments/assets/4e8f7795-4078-491a-9bed-8eb923046fb0" />

##### Проверка работы NFS:
Проверим, что шара смонтирована на клиенте:
```bash
ssh root@10.1.100.22 "df -h | grep nfs"
```
Ожидаемо: увидите строку 10.1.100.21:/srv/nfs_share с размером диска.

Проверим запись в /etc/fstab (чтобы сохранялось после перезагрузки):
```bash
ssh root@10.1.100.22 "tail -n 1 /etc/fstab"
```
Ожидаемо: 10.1.100.21:/srv/nfs_share /mnt/nfs_share nfs defaults,_netdev 0 0

##### Проверим передачу данных:
Создадим файл на клиенте и проверим его на сервере.
```bash
# Создаем файл на клиенте (10.1.100.22)
ssh root@10.1.100.22 "echo 'Hello NFS from Client' > /mnt/nfs_share/test_nfs.txt"

# Проверяем наличие файла на сервере (10.1.100.21)
ssh root@10.1.100.21 "cat /srv/nfs_share/test_nfs.txt"
Ожидаемо: Hello NFS from Client.
```

<img width="1109" height="169" alt="image" src="https://github.com/user-attachments/assets/a9436c1f-5774-4b23-bb77-cb187369dc99" />

##### Какие порты открываются и зачем:
**nfs (порт 2049/tcp)** — это основной порт, по которому передаются сами файлы и данные.

**rpc-bind (порт 111/tcp и udp)** — это «Portmapper». Когда клиент хочет смонтировать шару, он сначала стучится на порт 111 и спрашивает: «На каком порту у тебя работает служба монтирования?» Без этого порта клиент не знает, куда подключаться дальше.

**mountd (порт 20048/tcp)** — это служба, которая непосредственно обрабатывает запросы на монтирование (mount). Она проверяет файл /etc/exports и говорит клиенту «да» или «нет».

> [!NOTE]
> Важный момент: Служба mountd по умолчанию может выбирать динамический порт при каждом запуске. Но встроенные сервисы в firewalld (которые мы и включили) автоматически отслеживают эти изменения, поэтому нам не пришлось жестко прописывать порты вручную.

<img width="1114" height="715" alt="image" src="https://github.com/user-attachments/assets/5667621a-6990-453d-9d1f-ddbe9687f90b" />

##### Анализ вывода firewall-cmd --list-all:
```txt
- 10.1.100.21 (Сервер):
services: ... mountd nfs rpc-bind ... — открыты порты для NFS-сервера (2049, 20048, 111).
ports: 5432/tcp — открыт порт для PostgreSQL (Primary).

- 10.1.100.22 (Клиент):
services: ... ssh ... — нет служб NFS.
ports: 5432/tcp — открыт порт для PostgreSQL (Standby).
```
> [!TIP]
> Когда клиент .22 стучится на сервер .21, фаервол на .22 запоминает это соединение. Когда сервер отвечает, фаервол понимает, что это ответ на запрос клиента, и автоматически пропускает его обратно. Открывать входящие порты на клиенте не нужно.

### DRBD + Keepalived для NFS — сделаем файловое хранилище отказоустойчивым (если сервер 10.1.100.21 упадёт, шара продолжит работать на 10.1.100.22)
Полное развёртывание отказоустойчивого NFS (DRBD + Keepalived) на CentOS 7
Схема:
```text
   Клиенты (10.1.100.10 и др.)
             │ mount -t nfs 10.1.100.23:/srv/nfs_share
             ▼
     VIP 10.1.100.23 (Keepalived VRRP)
             │
   ┌─────────┴─────────┐
   │  10.1.100.21      │  10.1.100.22
   │  centpc21         │  centpc22
   │  MASTER (prio 100)│  BACKUP (prio 90)
   │  NFS-server       │  NFS-server
   │  DRBD Primary     │  DRBD Secondary
   │  /dev/drbd0←sdb   │  /dev/drbd0←sdb
   └───────────────────┘
            ▲ DRBD-репликация по 7788/tcp
```
Все команды ниже выполняются на управляющем хосте manager2122 (10.1.100.10) под root.
#### Архитектура решения
**DRBD**: Обеспечивает синхронную репликацию данных между дисками узлов 10.1.100.21 (Primary) и 10.1.100.22 (Secondary) в режиме реального времени.

**NFS**: Раздает файловую систему, размещенную на DRBD-устройстве.

**Keepalived**: Управляет виртуальным IP-адресом (10.1.100.23). Этот VIP будет привязан к узлу, который в данный момент является Primary для DRBD. При отказе узла VIP автоматически переместится на второй сервер.

#### ЭТАП 0. Предварительные требования на узлах
На виртуальных машинах .21 и .22 должен быть второй диск одинакового размера (например, 20 ГБ), видимый как /dev/sdb.

Если диск ещё не добавлен — добавьте его через гипервизор и пересканируйте SCSI-шину:
```bash
cat <<'EOF' > /root/rescan.sh
#!/bin/bash
for host in /sys/class/scsi_host/host*/scan; do
  echo "- - -" > "$host"
done
lsblk
EOF
chmod +x /root/rescan.sh
/root/rescan.sh
```
Проверка (на обоих узлах должен быть /dev/sdb одинакового размера):
```bash
ansible nfs_cluster -i /etc/ansible/hosts -m shell -a "lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT"
```
#### ЭТАП 1. Подготовка управляющего хоста
##### 1.1. Установить Ansible и NFS-клиент (для тестов)
```bash
yum install -y epel-release ansible nfs-utils
```
##### 1.2. Создать inventory
```bash
cat <<'EOF' > /etc/ansible/hosts
[postgres_cluster]
10.1.100.21 ansible_user=root pg_role=primary
10.1.100.22 ansible_user=root pg_role=standby

[nfs_cluster]
10.1.100.21 nfs_role=server
10.1.100.22 nfs_role=client

[nfs_cluster:vars]
ansible_python_interpreter=/usr/bin/python
EOF
```
##### 1.3. Настроить SSH-доступ по ключу (без пароля)
```bash
ssh-keygen -t rsa -N '' -f /root/.ssh/id_rsa <<< y >/dev/null 2>&1 || true
ssh-copy-id -o StrictHostKeyChecking=no root@10.1.100.21
ssh-copy-id -o StrictHostKeyChecking=no root@10.1.100.22
```
##### 1.4. Проверить связность
```bash
ansible nfs_cluster -i /etc/ansible/hosts -m ping
```
- Ожидаемо: оба узла SUCCESS → pong.

#### ЭТАП 2. Починка репозиториев CentOS 7 (EOL)
CentOS 7 снят с поддержки, зеркала mirror.centos.org отдают 403. Заменяем их на архив mirror.nsc.liu.se.
```bash
ansible nfs_cluster -i /etc/ansible/hosts -m shell -a "\
sed -i 's|mirror.centos.org|mirror.nsc.liu.se/centos-store|g' /etc/yum.repos.d/CentOS-*.repo && \
sed -i 's|vault.centos.org|mirror.nsc.liu.se/centos-store|g' /etc/yum.repos.d/CentOS-*.repo && \
sed -i 's|^#.*baseurl=http|baseurl=http|g' /etc/yum.repos.d/CentOS-*.repo && \
sed -i 's|^mirrorlist=http|#mirrorlist=http|g' /etc/yum.repos.d/CentOS-*.repo && \
yum clean all && yum makecache"
```
Проверка:
```bash
ansible nfs_cluster -i /etc/ansible/hosts -m shell -a "yum repolist | tail -3"
```
- Ожидаемо: repolist: 33000+.

#### ЭТАП 3. Установка DRBD 9 (проверенные RPM-пакеты)
Пакеты kmod-drbd90 и drbd90-utils в стандартном ELRepo для EL7 уже недоступны, скачаем проверенные версии с зеркал fnal.gov и coreix.net.

##### 3.1. Скачать RPM на manager
```bash
cd /root
rm -f kmod-drbd90-*.rpm drbd90-utils-*.rpm
curl -LO https://linux-mirrors.fnal.gov/linux/elrepo/testing/el7/x86_64/RPMS/kmod-drbd90-9.1.21-1.el7_9.elrepo.x86_64.rpm
curl -LO http://mirrors.coreix.net/elrepo/elrepo/el7/x86_64/RPMS/drbd90-utils-9.27.0-1.el7.elrepo.x86_64.rpm
```
Проверка (ОБЯЗАТЕЛЬНО!):
```bash
ls -lh kmod-drbd90-*.rpm drbd90-utils-*.rpm
file *.rpm
```
Должно быть:<br>
- kmod-drbd90-9.1.21...rpm — ~316K, RPM v3.0
- drbd90-utils-9.27.0...rpm — ~925K, RPM v3.0

Если размер снова 146/196/557 байт — ссылки снова сломались, найдите альтернативу на pkgs.org или https://mirrors.coreix.net/elrepo/elrepo/el7/x86_64/RPMS/.

##### 3.2. Скопировать пакеты на узлы
```bash
ansible nfs_cluster -i /etc/ansible/hosts -m copy \
  -a "src={{ item }} dest=/tmp/{{ item }}" \
  -e "item=kmod-drbd90-9.1.21-1.el7_9.elrepo.x86_64.rpm"

ansible nfs_cluster -i /etc/ansible/hosts -m copy \
  -a "src={{ item }} dest=/tmp/{{ item }}" \
  -e "item=drbd90-utils-9.27.0-1.el7.elrepo.x86_64.rpm"
  ```
##### 3.3. Удалить старый drbd-utils (если был установлен ранее)
```bash
ansible nfs_cluster -i /etc/ansible/hosts -m yum -a "name=drbd-utils state=absent"
```
##### 3.4. Установить пакеты
```bash
ansible nfs_cluster -i /etc/ansible/hosts -m yum \
  -a "name=/tmp/kmod-drbd90-9.1.21-1.el7_9.elrepo.x86_64.rpm,/tmp/drbd90-utils-9.27.0-1.el7.elrepo.x86_64.rpm state=present"
```
##### 3.5. Загрузить модуль ядра
```bash
ansible nfs_cluster -i /etc/ansible/hosts -m shell -a "modprobe drbd && lsmod | grep drbd"
```
- Ожидаемо: строка вида drbd 655700 0.

#### ЭТАП 4. SELinux
Для лабораторной переводим SELinux в Permissive (в продакшне нужны semanage fcontext и restorecon, здесь — упрощённо):

```bash
ansible nfs_cluster -i /etc/ansible/hosts -m shell -a "setenforce 0 && sed -i 's/^SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config && getenforce"
```
- Ожидаемо: Permissive на обоих.

Также убедимся, что на manager есть mount.nfs (иначе клиент не смонтирует шару — мы уже поставили nfs-utils в Этапе 1.1, проверим):

```bash
which mount.nfs
```
- Ожидаемо: /usr/sbin/mount.nfs.

#### ЭТАП 5. Создание Ansible-роли nfs_ha
##### 5.1. Структура каталогов
```bash
mkdir -p /etc/ansible/roles/nfs_ha/tasks
mkdir -p /etc/ansible/group_vars
```
##### 5.2. group_vars/nfs_cluster.yml
```bash
cat <<'EOF' > /etc/ansible/group_vars/nfs_cluster.yml
---
# Сетевые настройки
vip: "10.1.100.23"
network_interface: "ens33"

# Хранилище
nfs_dir: "/srv/nfs_share"
drbd_dev: "/dev/drbd0"
disk_dev: "/dev/sdb"
shared_secret: "my_super_secret_key_2026"

# Подсеть экспорта
nfs_network: "10.1.100.0/24"
EOF
```
##### 5.3. tasks/main.yml
```bash
cat <<'EOF' > /etc/ansible/roles/nfs_ha/tasks/main.yml
---
- import_tasks: prerequisites.yml
- import_tasks: drbd.yml
- import_tasks: nfs.yml
- import_tasks: keepalived.yml
EOF
```
##### 5.4. tasks/prerequisites.yml
```bash
cat <<'EOF' > /etc/ansible/roles/nfs_ha/tasks/prerequisites.yml
---
- name: Установка пакетов (shell обходит баг модуля yum в CentOS 7)
  shell: yum install -y keepalived nfs-utils rpcbind
  args:
    warn: false

- name: Открытие портов в Firewalld
  firewalld:
    port: "{{ item }}"
    permanent: yes
    state: enabled
    immediate: yes
  loop:
    - 2049/tcp    # NFS
    - 111/tcp     # rpcbind
    - 111/udp     # rpcbind
    - 20048/tcp   # mountd
    - 7788/tcp    # DRBD

- name: Разрешить VRRP (Keepalived)
  firewalld:
    rich_rule: 'rule family="ipv4" source address="{{ nfs_network }}" protocol value="vrrp" accept'
    permanent: yes
    state: enabled
    immediate: yes

- name: Фиксация портов NFS в /etc/nfs.conf
  copy:
    dest: /etc/nfs.conf
    content: |
      [mountd]
      port=20048
      [lockd]
      port=32803
      udp-port=32769
      [statd]
      port=32765
      [nfsd]
      vers3=y
      vers4=y
EOF
```
##### 5.5. tasks/drbd.yml
Ключевые моменты:
- В секциях on используются hostname (centpc21/centpc22), а не IP — иначе DRBD не сможет сопоставить узел.
- Перед create-md всегда делается drbdadm down с failed_when: false — иначе на повторном запуске будет Device is configured.
- create-md идёт с --force и yes | — чтобы не зависать на интерактивных вопросах.
```bash
cat <<'EOF' > /etc/ansible/roles/nfs_ha/tasks/drbd.yml
---
- name: Создание конфигурации ресурса DRBD (/etc/drbd.d/r0.res)
  copy:
    dest: /etc/drbd.d/r0.res
    content: |
      resource r0 {
        protocol C;
        startup {
          wfc-timeout  15;
          degr-wfc-timeout 60;
        }
        net {
          cram-hmac-alg sha1;
          shared-secret "{{ shared_secret }}";
        }
        on centpc21 {
          device    {{ drbd_dev }};
          disk      {{ disk_dev }};
          address   10.1.100.21:7788;
          meta-disk internal;
        }
        on centpc22 {
          device    {{ drbd_dev }};
          disk      {{ disk_dev }};
          address   10.1.100.22:7788;
          meta-disk internal;
        }
      }

- name: Остановка DRBD (если запущен) перед инициализацией
  command: drbdadm down r0
  failed_when: false
  changed_when: false

- name: Инициализация метаданных DRBD (create-md --force)
  shell: yes | drbdadm create-md r0 --force
  args:
    warn: false

- name: Запуск службы DRBD
  systemd:
    name: drbd
    state: started
    enabled: yes

- name: Поднятие ресурса DRBD
  command: drbdadm up r0
  changed_when: false

- name: Промоут первичного узла в Primary (только server)
  command: drbdadm primary --force r0
  when: nfs_role == 'server'
  changed_when: false

- name: Форматирование DRBD в XFS (только server)
  filesystem:
    fstype: xfs
    dev: "{{ drbd_dev }}"
  when: nfs_role == 'server'

- name: Демоут первичного узла в Secondary
  command: drbdadm secondary r0
  when: nfs_role == 'server'
  changed_when: false
EOF
```
##### 5.6. tasks/nfs.yml
```bash
cat <<'EOF' > /etc/ansible/roles/nfs_ha/tasks/nfs.yml
---
- name: Создание точки монтирования
  file:
    path: "{{ nfs_dir }}"
    state: directory
    mode: '0755'

- name: Настройка /etc/exports
  copy:
    dest: /etc/exports
    content: |
      {{ nfs_dir }} {{ nfs_network }}(rw,sync,no_root_squash,no_subtree_check)

- name: Отключение автозапуска NFS-сервера (им управляет Keepalived)
  systemd:
    name: nfs-server
    enabled: no
    state: stopped
  ignore_errors: yes
EOF
```
##### 5.7. tasks/keepalived.yml
Ключевые моменты:
- global_defs { enable_script_security; script_user root } — без этого Keepalived не запускает скрипты (в логах будет SECURITY VIOLATION).
- Скрипт nfs_ha.sh ждёт, пока peer перейдёт в UpToDate, прежде чем промоутить локальный DRBD.
- notify_stop и notify_fault вызывают nfs_ha.sh backup — это позволяет корректно демоутить DRBD при systemctl stop keepalived.
- state: restarted (не started) — чтобы конфиг применялся при каждом прогоне.

```bash
cat <<'EOF' > /etc/ansible/roles/nfs_ha/tasks/keepalived.yml
---
- name: Скрипт переключения (/usr/local/bin/nfs_ha.sh)
  copy:
    dest: /usr/local/bin/nfs_ha.sh
    mode: '0755'
    content: |
      #!/bin/bash
      case "$1" in
        master)
          echo "Switching to MASTER"
          for i in {1..30}; do
            if drbdadm status r0 | grep -q "peer-disk:UpToDate"; then break; fi
            sleep 1
          done
          drbdadm primary --force r0 || exit 1
          sleep 2
          mount {{ drbd_dev }} {{ nfs_dir }} || exit 1
          systemctl start rpcbind
          systemctl start nfs-server
          ;;
        backup)
          echo "Switching to BACKUP"
          systemctl stop nfs-server
          systemctl stop rpcbind
          umount {{ nfs_dir }} || true
          drbdadm secondary r0
          ;;
      esac

- name: Конфигурация Keepalived (/etc/keepalived/keepalived.conf)
  copy:
    dest: /etc/keepalived/keepalived.conf
    content: |
      global_defs {
          enable_script_security
          script_user root
      }

      vrrp_script chk_nfs {
          script "/usr/bin/pgrep nfsd"
          interval 2
          weight -20
      }

      vrrp_instance VI_1 {
          state {{ 'MASTER' if nfs_role == 'server' else 'BACKUP' }}
          interface {{ network_interface }}
          virtual_router_id 51
          priority {{ 100 if nfs_role == 'server' else 90 }}
          advert_int 1
          authentication {
              auth_type PASS
              auth_pass 1111
          }
          virtual_ipaddress {
              {{ vip }}/24
          }

          notify_master "/usr/local/bin/nfs_ha.sh master"
          notify_backup "/usr/local/bin/nfs_ha.sh backup"
          notify_fault  "/usr/local/bin/nfs_ha.sh backup"
          notify_stop   "/usr/local/bin/nfs_ha.sh backup"

          track_script {
              chk_nfs
          }
      }

- name: Запуск и включение Keepalived
  systemd:
    name: keepalived
    state: restarted
    enabled: yes
EOF
```
#### ЭТАП 6. Основной плейбук
```bash
cat <<'EOF' > /etc/ansible/nfs_cluster.yml
---
- name: Развертывание отказоустойчивого NFS кластера (DRBD + Keepalived)
  hosts: nfs_cluster
  become: yes
  roles:
    - nfs_ha
EOF
```
#### ЭТАП 7. Запуск и проверка
##### 7.1. Проверить синтаксис и запустить
```bash
ansible-playbook -i /etc/ansible/hosts /etc/ansible/nfs_cluster.yml --syntax-check
ansible-playbook -i /etc/ansible/hosts /etc/ansible/nfs_cluster.yml
```
- Должно быть failed=0 на обоих узлах.

##### 7.2. Дождаться окончания синхронизации DRBD
Синхронизация 20 ГБ по LAN обычно занимает 1–3 минуты. Проверяйте до появления replication:Established и UpToDate с обеих сторон:
```bash
ansible nfs_cluster -i /etc/ansible/hosts -m shell -a "drbdadm status r0"
```
- Ожидаемый финальный вывод:
```text
10.1.100.21: r0 role:Primary,   disk:UpToDate,  centpc22 role:Secondary peer-disk:UpToDate
10.1.100.22: r0 role:Secondary, disk:UpToDate,  centpc21 role:Primary   peer-disk:UpToDate
```
> [!WARNING]
> Пока идёт синк (SyncSource/SyncTarget), НЕ перезапускайте Keepalived — это прервёт синхронизацию.

##### 7.3. Перезапустить Keepalived на .21 после синка
Плейбук уже перезапускал Keepalived, но если к моменту его старта peer ещё был Inconsistent, notify_master не смог промоутить DRBD. После окончания синка:
```bash
ansible 10.1.100.21 -i /etc/ansible/hosts -m systemd -a "name=keepalived state=restarted"
sleep 8
```
##### 7.4. Финальная проверка состояния
```bash
# VIP на .21
ansible nfs_cluster -i /etc/ansible/hosts -m shell -a "ip a show ens33 | grep 10.1.100.23"

# DRBD: Primary на .21
ansible nfs_cluster -i /etc/ansible/hosts -m shell -a "drbdadm status r0"

# NFS на .21 активен, диск примонтирован
ansible 10.1.100.21 -i /etc/ansible/hosts -m shell -a "systemctl is-active nfs-server; mount | grep nfs_share"

# NFS на .22 выключен
ansible 10.1.100.22 -i /etc/ansible/hosts -m shell -a "systemctl is-active nfs-server"

# Логи keepalived: нет SECURITY VIOLATION, скрипт запускается
ansible 10.1.100.21 -i /etc/ansible/hosts -m shell -a "journalctl -u keepalived -n 30 --no-pager | grep -iE 'script|notify|security|violation'"
```
Ожидаемо:

VIP 10.1.100.23 на .21

.21: r0 role:Primary, disk:UpToDate

NFS на .21: active, /dev/drbd0 on /srv/nfs_share type xfs

NFS на .22: inactive

В логах keepalived: Opening script file /usr/local/bin/nfs_ha.sh и отсутствие SECURITY VIOLATION

##### 7.5. Тест с клиента (manager)
```bash
umount /mnt/nfs_test 2>/dev/null || true
rm -rf /mnt/nfs_test
mkdir -p /mnt/nfs_test

mount -t nfs 10.1.100.23:/srv/nfs_share /mnt/nfs_test
mount | grep nfs_test        # должно явно показать 10.1.100.23:/srv/nfs_share
echo "Hello HA NFS $(date)" > /mnt/nfs_test/test.txt
cat /mnt/nfs_test/test.txt
```
> [!WARNING]
> Ключевая проверка — mount | grep nfs_test. Если вывод пуст, mount не сработал (проверьте nfs-utils на manager), и cat прочитает локальный файл, а не с NFS.

#### ЭТАП 8. Тест failover
```bash
# 1. Размонтировать на клиенте (чтобы смонтировать заново после переезда)
umount /mnt/nfs_test

# 2. Остановить Keepalived ТОЛЬКО на MASTER (.21)
ansible 10.1.100.21 -i /etc/ansible/hosts -m systemd -a "name=keepalived state=stopped"
sleep 8

# 3. Проверить переезд
ansible nfs_cluster -i /etc/ansible/hosts -m shell -a "ip a show ens33 | grep 10.1.100.23"
#   → VIP должен быть на .22
ansible nfs_cluster -i /etc/ansible/hosts -m shell -a "drbdadm status r0"
#   → .21 Secondary, .22 Primary
ansible 10.1.100.22 -i /etc/ansible/hosts -m shell -a "systemctl is-active nfs-server; mount | grep nfs_share"
#   → active + /dev/drbd0 on /srv/nfs_share

# 4. Смонтировать заново через VIP
mount -t nfs 10.1.100.23:/srv/nfs_share /mnt/nfs_test
mount | grep nfs_test
cat /mnt/nfs_test/test.txt
#   → данные сохранены благодаря DRBD-репликации
Возврат в исходное состояние
bash
umount /mnt/nfs_test
ansible 10.1.100.21 -i /etc/ansible/hosts -m systemd -a "name=keepalived state=started"
sleep 2
ansible 10.1.100.22 -i /etc/ansible/hosts -m systemd -a "name=keepalived state=restarted"
sleep 5

# Проверка: VIP снова на .21, DRBD Primary на .21
ansible nfs_cluster -i /etc/ansible/hosts -m shell -a "ip a show ens33 | grep 10.1.100.23"
ansible nfs_cluster -i /etc/ansible/hosts -m shell -a "drbdadm status r0"
```





































