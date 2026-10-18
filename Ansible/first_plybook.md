#### Для базовой проверки создайте простейший файл инвентаря (список хостов), чтобы убедиться, что Ansible может управлять локальной или удаленными машинами.
##### 1. Откройте или отредактируйте стандартный файл хостов:
```bash
vi /etc/ansible/hosts
```
##### 2. Добавьте туда ваш текущий сервер (или удаленный IP), например:
```ini
[local]
localhost ansible_connection=local
```
##### 3. Выполните тестовую команду ping через Ansible:
```bash
ansible local -m ping
```

<img width="1121" height="138" alt="image" src="https://github.com/user-attachments/assets/f44516e7-2391-4a68-9a30-dba1db50c86f" />

Если всё настроено верно, Ansible вернет зеленый ответ **"ping": "pong"**

#### Давайте создадим простой сценарий автоматизации (Playbook). Он проверит, установлен ли в системе текстовый редактор vim и запущен ли веб-сервер, который я настраивал ранее (nginx), а при необходимости установит и включит их.
##### 1. Создайте файл сценария:
```bash
nano /etc/ansible/first_playbook.yml
```
##### 2. Вставьте в него следующий YAML-код:
```yaml
---
- name: Проверка и настройка базового ПО
  hosts: local
  become: yes
  tasks:

    - name: Установка последней версии vim
      yum:
        name: vim
        state: latest

    - name: Установка веб-сервера nginx
      yum:
        name: nginx
        state: present

    - name: Запуск и включение автозагрузки nginx
      service:
        name: nginx
        state: started
        enabled: yes
```
##### 3. Запустите сценарий на выполнение:
```bash
ansible-playbook /etc/ansible/first_playbook.yml
```
После выполнения вы увидите подробный отчет о проделанной работе.

<img width="1128" height="611" alt="image" src="https://github.com/user-attachments/assets/70205d14-60c5-4db6-9bb6-1ba808eefdfb" />

#### Чтобы запустить сценарий на удалённых хостах, например 10.1.100.12 и 10.1.100.13, необходимо сделать две вещи: добавить их в файл инвентаря (hosts) и настроить доступ к ним по SSH.

##### Шаг 1. Обновите файл инвентаря hosts
Вместо локального подключения создайте новую группу хостов (например, [servers]) и пропишите туда ваши IP-адреса. Выполните команду, чтобы перезаписать файл:
```bash
cat << 'EOF' > /etc/ansible/hosts
[servers]
10.1.100.12
10.1.100.13
EOF
```
##### Шаг 2. Настройте беспарольный доступ по SSH (Обязательно)
Ansible подключается к удалённым серверам по протоколу SSH. Чтобы он не запрашивал пароль при каждом запуске, настройте авторизацию по ключам:

##### 1. Сгенерируйте SSH-ключ на вашей текущей машине (если не делали этого ранее, на все вопросы нажимайте Enter):
```bash
ssh-keygen -t rsa -b 4048
```

<img width="1138" height="317" alt="image" src="https://github.com/user-attachments/assets/0d54719e-16c1-4e86-827c-d9359fde02fc" />

##### 2. Скопируйте ключ на оба удалённых хоста (система один раз попросит ввести пароль пользователя root от удалённых машин):
```bash
ssh-copy-id root@10.1.100.12
ssh-copy-id root@10.1.100.13
```

<img width="1125" height="448" alt="image" src="https://github.com/user-attachments/assets/574e2b28-0ef9-4661-b99f-78cc326f0b81" />

##### 3. Проверьте доступ с помощью модуля ping. Команда должна вернуть зеленый статус SUCCESS для обоих IP:
```bash
ansible servers -m ping
```

<img width="1021" height="255" alt="image" src="https://github.com/user-attachments/assets/9bc40c47-2784-486b-bfdc-6d9493fa50b9" />

Если всё настроено верно, Ansible вернет зеленый ответ **"ping": "pong"**.

```bash
[root@centpca ~]# cat << 'EOF' > /etc/ansible/first_playbook.yml
> ---
> - name: Подготовка репозиториев и настройка базового ПО
>   hosts: servers
>   become: yes
>   tasks:
>
>     - name: Удаление всех старых файлов репозиториев
>       file:
>         path: "{{ item }}"
>         state: absent
>       loop:
>         - /etc/yum.repos.d/CentOS-Base.repo
>         - /etc/yum.repos.d/CentOS-Vault.repo
>         - /etc/yum.repos.d/CentOS-CR.repo
>         - /etc/yum.repos.d/CentOS-Debuginfo.repo
>         - /etc/yum.repos.d/CentOS-Media.repo
>         - /etc/yum.repos.d/CentOS-Sources.repo
>         - /etc/yum.repos.d/CentOS-fasttrack.repo
>         - /etc/yum.repos.d/epel.repo
>         - /etc/yum.repos.d/epel-testing.repo
>         - /etc/yum.repos.d/myrepo.repo
>
>     - name: Создание рабочего CentOS-Base.repo (HTTP-архив Alibaba Cloud)
>       copy:
>         dest: /etc/yum.repos.d/CentOS-Base.repo
>         content: |
>           [base]
>           name=CentOS-7.9.2009 - Base
>           baseurl=http://mirrors.aliyun.com/centos-vault/7.9.2009/os/$basearch/
>           gpgcheck=0
>           enabled=1
>
>           [updates]
>           name=CentOS-7.9.2009 - Updates
>           baseurl=http://mirrors.aliyun.com/centos-vault/7.9.2009/updates/$basearch/
>           gpgcheck=0
>           enabled=1
>
>           [extras]
>           name=CentOS-7.9.2009 - Extras
>           baseurl=http://mirrors.aliyun.com/centos-vault/7.9.2009/extras/$basearch/
>           gpgcheck=0
>           enabled=1
>
>     - name: Создание репозитория EPEL (HTTP-архив Alibaba Cloud)
>       copy:
>         dest: /etc/yum.repos.d/epel.repo
>         content: |
>           [epel]
>           name=Extra Packages for Enterprise Linux 7 - $basearch
>           baseurl=http://mirrors.aliyun.com/epel-archive/7/$basearch/
>           enabled=1
>           gpgcheck=0
>
>     - name: Очистка кэша yum
>       command: yum clean all
>
>     - name: Обновление кэша yum
>       command: yum makecache
>
>     - name: Обновление криптографических библиотек для поддержки HTTPS
>       yum:
>         name:
>           - nss
>           - nss-util
>           - nss-sysinit
>           - nss-tools
>           - curl
>           - ca-certificates
>         state: latest
>
>     - name: Установка последней версии vim
>       yum:
>         name: vim
>         state: latest
>
>     - name: Установка веб-сервера nginx
>       yum:
>         name: nginx
>         state: present
>
>     - name: Запуск и включение автозагрузки nginx
>       service:
>         name: nginx
>         state: started
>         enabled: yes
> ...
> EOF
[root@centpca ~]# ansible-playbook /etc/ansible/first_playbook.yml
```

<img width="1068" height="1051" alt="image" src="https://github.com/user-attachments/assets/d21bb88d-7ec0-48b8-b206-2c85b8a8c32d" />

