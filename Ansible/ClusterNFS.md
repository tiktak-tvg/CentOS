### Кластер хранения (NFS): Один узел выступает в роли NFS-сервера (раздает директорию), а второй — в роли клиента (монтирует ее). Ansible-роль может управлять как серверной частью (/etc/exports), так и клиентской (записи в /etc/fstab).
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
- name: Установка NFS Server
  yum:
    name: nfs-utils
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
```bash
cat <<'EOF' > /etc/ansible/roles/nfs_server/handlers/main.yml
---
- name: restart nfs-server
  service:
    name: nfs-server
    state: restarted
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
