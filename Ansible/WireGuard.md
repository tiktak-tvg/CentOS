### Развертывание VPN-сервера WireGuard с помощью Ansible
WireGuard — это современный, быстрый и безопасный VPN-протокол, встроенный в ядро Linux начиная с версии 5.6. На CentOS 7 ядро старое (3.10), поэтому мы установим модуль ядра из репозитория ELRepo — это официально рекомендованный способ.

Инструкция по созданию Ansible-роли, которая:
- Устанавливает WireGuard (модуль ядра + утилиты)
- Генерирует ключи для сервера и клиентов
- Настраивает сетевой интерфейс wg0
- Включает IP-форвардинг и маскарадинг
- Открывает UDP-порт в firewalld
- Создаёт готовые конфигурации для клиентов

#### Предварительные условия (руками, до Ansible)
Эти шаги не автоматизируются плейбуком — их надо сделать один раз на каждом хосте.

##### 0.1. Управляющая машина centpca
```bash
# Установить Ansible
yum install -y epel-release
yum ins

# Проверить версию (должна быть 2.9+)
ansible --version
```
##### 0.2. На хостах (10.1.100.21, centpManager 10.1.100.9)
```bash
# Отключить SELinux (или оставить Enforcing — роль deploy_app ставит seboolean)
# Если оставляете Enforcing — ничего не делайте, роль справится.
# Если отключаете:
setenforce 0
sed -i 's/^SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config

# Убедиться, что firewalld запущен
systemctl enable --now firewalld

# Убедиться, что hostname корректный (для шаблона Jinja2)
hostnamectl set-hostname centp08vpn   # на 10.1.100.21
```
##### 0.3. SSH-ключи с centManager на 10.1.100.9
```bash
# На centManager
ssh-keygen -t rsa -b 2048 -N "" -f ~/.ssh/id_rsa

# Скопировать ключ на оба хоста
ssh-copy-id root@10.1.100.21

# Проверить, что заходит без пароля
ssh root@10.1.100.21 'hostname'
```
##### 0.4. Инвентарь /etc/ansible/hosts
```
cat <<'EOF' >> /etc/ansible/hosts

[vpn]
10.1.100.21 ansible_user=root
EOF
```
#### 0.5. Проверка связности
```bash
ansible vpn -m ping
# Ожидаемо: оба хоста → SUCCESS
```
Если ping не проходит — дальше идти нельзя. Разбирайтесь с SSH/ключами/сетью.

<img width="1123" height="235" alt="image" src="https://github.com/user-attachments/assets/08d381ae-8326-499f-a684-a51fda33616b" />


#### Какие порты открыть в firewalld
Для работы WireGuard на сервере (10.1.100.21) нужно открыть один UDP-порт и включить маскарадинг (NAT), чтобы клиенты могли выходить в интернет через VPN.

Порт / Протокол  	|Назначение
----------------- | ---------------------------------------------
51820/udp	 |Основной порт WireGuard для подключения клиентов.
masquerade: yes	 |Включает NAT для трафика клиентов (без этого они не получат доступ в интернет).

Выполнить это можно так:

```bash
# Открыть порт WireGuard
ansible vpn -m shell -a "firewall-cmd --permanent --add-port=51820/udp --zone=public"
# Включить маскарадинг
ansible vpn -m shell -a "firewall-cmd --permanent --add-masquerade"
# Применить изменения
ansible vpn -m shell -a "firewall-cmd --reload"
# Проверить
ansible vpn -m shell -a "firewall-cmd --list-all"
```
> Примечание: Если вы используете WireGuard Easy (веб-интерфейс), также может понадобиться TCP-порт 51821 для доступа к панели управления, но для чистого WireGuard он не нужен.

<img width="1127" height="406" alt="image" src="https://github.com/user-attachments/assets/1d9e6c5a-d2be-4b00-b2a8-87bd99e95f1e" />


#### ЭТАП 1. Структура роли
```bash
mkdir -p /etc/ansible/roles/wireguard/{tasks,templates,handlers,defaults}
```
#### ЭТАП 2. defaults/main.yml — переменные роли
```bash
cat <<'EOF' > /etc/ansible/roles/wireguard/defaults/main.yml
---
# Параметры VPN-сервера
wg_interface: wg0
wg_port: 51820
wg_server_ip: "10.8.0.1"
wg_network: "10.8.0.0/24"
wg_dns: "8.8.8.8, 8.8.4.4"

# Список клиентов: имя и IP-адрес в VPN-сети
wg_clients:
  - name: client1
    ip: "10.8.0.2"
  - name: client2
    ip: "10.8.0.3"
...
EOF
```
#### ЭТАП 3. tasks/main.yml — задачи роли
Ключевые фиксы: wireguard-dkms из COPR (не kmod-wireguard), чтение ключей через slurp, никаких lookup('file').
```bash
cat <<'EOF' > /etc/ansible/roles/wireguard/tasks/main.yml
---
# 1. Установка репозиториев
- name: Установка EPEL
  yum:
    name: epel-release
    state: present

- name: Добавление репозитория COPR с пакетом wireguard-dkms
  get_url:
    url: https://copr.fedorainfracloud.org/coprs/jdoss/wireguard/repo/epel-7/jdoss-wireguard-epel-7.repo
    dest: /etc/yum.repos.d/jdoss-wireguard-epel-7.repo
    mode: '0644'

# 2. Установка WireGuard через DKMS (работает с любым ядром CentOS 7)
- name: Установка wireguard-dkms, wireguard-tools и qrencode
  yum:
    name:
      - wireguard-dkms
      - wireguard-tools
      - qrencode
    state: present

# 3. Загрузка модуля ядра
- name: Загрузка модуля wireguard
  modprobe:
    name: wireguard
    state: present

- name: Добавление wireguard в автозагрузку модулей
  copy:
    dest: /etc/modules-load.d/wireguard.conf
    content: "wireguard\n"
    mode: '0644'

# 4. Создание директории для конфигов
- name: Создание /etc/wireguard
  file:
    path: /etc/wireguard
    state: directory
    owner: root
    group: root
    mode: '0700'

# 5. Генерация ключей сервера (если ещё нет)
- name: Проверка наличия приватного ключа сервера
  stat:
    path: /etc/wireguard/server_private.key
  register: server_key

- name: Генерация приватного ключа сервера
  shell: wg genkey > /etc/wireguard/server_private.key
  when: not server_key.stat.exists

- name: Генерация публичного ключа сервера
  shell: wg pubkey < /etc/wireguard/server_private.key > /etc/wireguard/server_public.key
  when: not server_key.stat.exists

# 6. Чтение ключей сервера на целевой машине (slurp, а не lookup)
- name: Чтение приватного ключа сервера
  slurp:
    src: /etc/wireguard/server_private.key
  register: server_priv_key

- name: Чтение публичного ключа сервера
  slurp:
    src: /etc/wireguard/server_public.key
  register: server_pub_key

# 7. Генерация ключей клиентов
- name: Создание директории для клиентских ключей
  file:
    path: /etc/wireguard/clients
    state: directory
    mode: '0700'

- name: Генерация ключей для каждого клиента
  shell: |
    wg genkey > /etc/wireguard/clients/{{ item.name }}_private.key
    wg pubkey < /etc/wireguard/clients/{{ item.name }}_private.key > /etc/wireguard/clients/{{ item.name }}_public.key
  args:
    creates: /etc/wireguard/clients/{{ item.name }}_private.key
  loop: "{{ wg_clients }}"

# 8. Чтение ключей клиентов на целевой машине
- name: Чтение приватных ключей клиентов
  slurp:
    src: /etc/wireguard/clients/{{ item.name }}_private.key
  register: client_priv_keys
  loop: "{{ wg_clients }}"

- name: Чтение публичных ключей клиентов
  slurp:
    src: /etc/wireguard/clients/{{ item.name }}_public.key
  register: client_pub_keys
  loop: "{{ wg_clients }}"

# 9. Создание конфигурации сервера wg0.conf
- name: Создание /etc/wireguard/wg0.conf
  template:
    src: wg0.conf.j2
    dest: /etc/wireguard/wg0.conf
    owner: root
    group: root
    mode: '0600'
  notify: restart wireguard

# 10. Включение IP-форвардинга
- name: Включение ip_forward
  sysctl:
    name: net.ipv4.ip_forward
    value: '1'
    sysctl_set: yes
    state: present
    reload: yes

# 11. Настройка firewalld
- name: Открытие UDP-порта WireGuard
  firewalld:
    port: "{{ wg_port }}/udp"
    permanent: yes
    state: enabled
    immediate: yes

- name: Включение masquerade (для выхода клиентов в интернет)
  firewalld:
    masquerade: yes
    permanent: yes
    state: enabled
    immediate: yes

# 12. Запуск и включение сервиса WireGuard
- name: Запуск и включение wg-quick@wg0
  systemd:
    name: "wg-quick@{{ wg_interface }}"
    state: started
    enabled: yes

# 13. Генерация клиентских конфигураций
- name: Создание клиентских конфигов
  template:
    src: client.conf.j2
    dest: "/etc/wireguard/clients/{{ item.name }}.conf"
    mode: '0600'
  loop: "{{ wg_clients }}"

# 14. Вывод информации для подключения
- name: Показать публичный ключ сервера
  debug:
    msg: "Публичный ключ сервера: {{ server_pub_key.content | b64decode | trim }}"

- name: Показать IP-адрес сервера
  debug:
    msg: "IP сервера: {{ ansible_default_ipv4.address }}:{{ wg_port }}"
...
EOF
```
#### ЭТАП 4. templates/wg0.conf.j2 — конфиг сервера
Фикс: используем server_priv_key (из slurp) и вложенный {% for %} для сопоставления клиентов.
```bash
cat <<'EOF' > /etc/ansible/roles/wireguard/templates/wg0.conf.j2
[Interface]
Address = {{ wg_server_ip }}/24
ListenPort = {{ wg_port }}
PrivateKey = {{ server_priv_key.content | b64decode | trim }}
SaveConfig = false
PostUp = firewall-cmd --add-masquerade --permanent; firewall-cmd --reload
PostDown = firewall-cmd --remove-masquerade --permanent; firewall-cmd --reload

{% for client in wg_clients %}
{% for result in client_pub_keys.results %}
{% if result.item.name == client.name %}
[Peer]
# {{ client.name }}
PublicKey = {{ result.content | b64decode | trim }}
AllowedIPs = {{ client.ip }}/32
{% endif %}
{% endfor %}
{% endfor %}
EOF
```
#### ЭТАП 5. templates/client.conf.j2 — конфиг клиента
Фикс: используем client_priv_keys (из slurp) и вложенный {% for %}.
```bash
cat <<'EOF' > /etc/ansible/roles/wireguard/templates/client.conf.j2
[Interface]
{% for result in client_priv_keys.results %}
{% if result.item.name == item.name %}
PrivateKey = {{ result.content | b64decode | trim }}
{% endif %}
{% endfor %}
Address = {{ item.ip }}/24
DNS = {{ wg_dns }}

[Peer]
PublicKey = {{ server_pub_key.content | b64decode | trim }}
Endpoint = {{ ansible_default_ipv4.address }}:{{ wg_port }}
AllowedIPs = 0.0.0.0/0, ::/0
PersistentKeepalive = 25
EOF
```
#### ЭТАП 6. handlers/main.yml — обработчики
```bash
cat <<'EOF' > /etc/ansible/roles/wireguard/handlers/main.yml
---
- name: restart wireguard
  systemd:
    name: "wg-quick@{{ wg_interface }}"
    state: restarted
...
EOF
```
#### ЭТАП 7. wireguard.yml — главный плейбук
```bash
cat <<'EOF' > /etc/ansible/wireguard.yml
---
- name: Развертывание VPN-сервера WireGuard
  hosts: vpn
  become: yes
  roles:
    - wireguard
...
EOF
```
#### ЭТАП 8. Запуск
```bash
ansible-playbook /etc/ansible/wireguard.yml
```
Что произойдёт:
- Установится wireguard-dkms (компиляция модуля 2–5 минут) + wireguard-tools + qrencode.
- Загрузится модуль ядра wireguard.
- Сгенерируются ключи сервера и всех клиентов из wg_clients.
- Создадутся wg0.conf и клиентские конфиги.
- Включится IP-форвардинг и masquerade.
- Откроется UDP-порт 51820.
- Сервис wg-quick@wg0 запустится и добавится в автозагрузку.
- Ansible выведет публичный ключ сервера и его IP:порт.
- Ожидаемый результат: PLAY RECAP с failed=0.

<img width="1111" height="305" alt="image" src="https://github.com/user-attachments/assets/b4b4244e-876a-4d92-8440-0d1f5b748666" />

#### ЭТАП 9. Проверка на сервере
```bash
# 1. Интерфейс поднят
ansible vpn -m shell -a "wg show"

# 2. Сервис активен
ansible vpn -m shell -a "systemctl is-active wg-quick@wg0"

# 3. Порт слушается
ansible vpn -m shell -a "ss -ulpn | grep 51820"

# 4. Firewalld
ansible vpn -m shell -a "firewall-cmd --list-all | grep -E 'masquerade|51820'"

# 5. IP-форвардинг
ansible vpn -m shell -a "sysctl net.ipv4.ip_forward"
# Ожидаемо: net.ipv4.ip_forward = 1

# 6. Конфиг сервера
ansible vpn -m shell -a "cat /etc/wireguard/wg0.conf"

# 7. Клиентский конфиг
ansible vpn -m shell -a "cat /etc/wireguard/clients/client1.conf"
```

<img width="1165" height="1073" alt="image" src="https://github.com/user-attachments/assets/4765e9e1-0fcd-432a-80c9-7e6014accdc6" />

#### ТАП 10. Настройка клиента (centManager)
##### 10.1. Установка модуля ядра WireGuard на клиенте
Ключевой фикс: без модуля ядра wg-quick up выдаст Operation not supported.

```bash
# 1. EPEL
yum install -y epel-release

# 2. COPR-репозиторий
curl -o /etc/yum.repos.d/jdoss-wireguard-epel-7.repo \
  https://copr.fedorainfracloud.org/coprs/jdoss/wireguard/repo/epel-7/jdoss-wireguard-epel-7.repo

# 3. Установка модуля и утилит
yum install -y wireguard-dkms wireguard-tools

# 4. Загрузка модуля
modprobe wireguard
lsmod | grep wireguard
```

<img width="1138" height="376" alt="image" src="https://github.com/user-attachments/assets/d85d3d34-a335-421e-87cd-ac18a7716795" />

##### 10.2. Копирование клиентского конфига
```bash
mkdir -p /etc/wireguard
scp root@10.1.100.21:/etc/wireguard/clients/client1.conf /etc/wireguard/client1.conf
chmod 600 /etc/wireguard/client1.conf
```
##### 10.3. Подключение
```bash
wg-quick up client1
```
##### 10.4. Проверка
```bash
# 1. Handshake установлен
wg show
# Ожидаемо: "latest handshake: X seconds ago"

# 2. Пинг до сервера
ping -c 3 10.8.0.1
# Ожидаемо: 0% packet loss

# 3. Трафик идёт через VPN
ip route get 8.8.8.8
# Ожидаемо: "dev client1" в выводе

# 4. Счётчики transfer растут
wg show
```

<img width="1090" height="755" alt="image" src="https://github.com/user-attachments/assets/e6741bcd-f2e4-43dc-9004-d80c5b9bff42" />

##### 10.5. Автозагрузка клиента
```bash
systemctl enable wg-quick@client1
```
#### ЭТАП 11. Итоговая проверка
Проверка	|Команда	|Ожидаемо
---------- | ------- | -------------------------
Сервер: интерфейс		|ansible vpn -m shell -a "wg show"		|interface: wg0 + 2 пира
Сервер: сервис		|ansible vpn -m shell -a "systemctl is-active wg-quick@wg0"		|active
Сервер: порт		|ansible vpn -m shell -a "ss -ulpn l grep 51820"		|*:51820
Сервер: NAT		|ansible vpn -m shell -a "firewall-cmd --list-all grep masquerade"		|masquerade: yes
Сервер: ip_forward		|ansible vpn -m shell -a "sysctl net.ipv4.ip_forward"		|= 1
Клиент: handshake		|wg show		|latest handshake: X sec ago
Клиент: пинг		|ping 10.8.0.1		|0% packet loss
Клиент: маршрут		|ip route get 8.8.8.8		|dev client1

```txt

┌──────────────────────────────────────────────────────────────┐
│  Локальная сеть 10.1.100.0/24                                │
│                                                              │
│  ┌─────────────┐       ┌──────────────┐                      │
│  │ centManager │──────→│ 10.1.100.21  │                      │
│  │ (клиент)    │  VPN  │ (VPN-сервер) │                      │
│  └─────────────┘       └──────┬───────┘                      │
│                               │                              │
│                               │ masquerade                   │
│                               ▼                              │
│                    ┌───────────────────┐                     │
│                    │  Общий NAT-шлюз   │                     │
│                    │  (195.178.4.0/22) │                     │
│                    └─────────┬─────────┘                     │
└──────────────────────────────┼───────────────────────────────┘
                               │
                               ▼
                        ┌──────────────┐
                        │  Интернет    │
                        └──────────────┘
```
##### Добавление нового клиента
В defaults/main.yml добавьте запись:

```yaml
- name: client3
  ip: "10.8.0.4"
```

<img width="1128" height="245" alt="image" src="https://github.com/user-attachments/assets/16f50659-cfbf-43a6-aa1d-a2062f79e077" />

Запустите плейбук:
```bash
ansible-playbook /etc/ansible/wireguard.yml
Ansible сгенерирует ключи для client3, добавит его в wg0.conf и создаст /etc/wireguard/clients/client3.conf.
```
Сервис перезапустится автоматически (через handler).

##### Итоговая архитектура
```text
┌────────────────────────────────────────────────────────────┐
│  10.1.100.21 (VPN-сервер, WireGuard)                       │
│                                                            │
│  wg0: 10.8.0.1/24                                          │
│  UDP: 51820 (открыт в firewalld)                           │
│  ip_forward = 1, masquerade = yes                          │
│                                                            │
│  ┌──────────────┐  ┌──────────────┐                        │
│  │ client1      │  │ client2      │                        │
│  │ 10.8.0.2/24  │  │ 10.8.0.3/24  │                        │
│  └──────────────┘  └──────────────┘                        │
└────────────────────────────────────────────────────────────┘
              ▲                    ▲
              │ UDP 51820          │ UDP 51820
              │                    │
    ┌─────────┴─────────┐  ┌───────┴─────────┐
    │ centManager       │  │ другое устройство│
    │ 10.8.0.2          │  │ 10.8.0.3         │
    └───────────────────┘  └─────────────────┘
```





