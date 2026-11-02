### Пошаговое руководство по развёртыванию лёгкого Kubernetes-кластера (K3s) на трёх и более машинах с помощью официальной Ansible-коллекции k3s-io/k3s-ansible. Это production-подход: control-plane в HA-режиме (3 сервера с встроенным etcd), воркеры-агенты, автоматическая подготовка узлов и проверка отказоустойчивости.
На всех хостах: 
cat /etc/os-release
NAME="CentOS Linux"
VERSION="7 (Core)"
ID="centos"
ID_LIKE="rhel fedora"
VERSION_ID="7"
PRETTY_NAME="CentOS Linux 7 (Core)"
ANSI_COLOR="0;31"
CPE_NAME="cpe:/o:centos:centos:7"
HOME_URL="https://www.centos.org/"
BUG_REPORT_URL="https://bugs.centos.org/"

CENTOS_MANTISBT_PROJECT="CentOS-7"
CENTOS_MANTISBT_PROJECT_VERSION="7"
REDHAT_SUPPORT_PRODUCT="centos"
REDHAT_SUPPORT_PRODUCT_VERSION="7"

uname -r
3.10.0-1160.119.1.el7.x86_64
cat /etc/centos-release
CentOS Linux release 7.9.2009 (Core)


Все команды выполняются на управляющем хосте hostname:managerK3s (ip:10.1.100.10) под root.
На нём установлены Ansible, nginx, firewalld

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

На них установлен firewalld

[root@managerk3s ~]# ssh root@10.1.100.28 'uname -r'
3.10.0-1160.119.1.el7.x86_64
[root@managerk3s ~]# ssh root@10.1.100.24 'uname -r'
3.10.0-1160.119.1.el7.x86_64
[root@managerk3s ~]# ssh root@10.1.100.25 'uname -r'
3.10.0-1160.119.1.el7.x86_64
[root@managerk3s ~]# ssh root@10.1.100.26 'uname -r'
3.10.0-1160.119.1.el7.x86_64
[root@managerk3s ~]# ssh root@10.1.100.27 'uname -r'
3.10.0-1160.119.1.el7.x86_64

Схема кластера (пример)
```text
        ┌─────────────────────────────────────┐
        │  managerK3s  (10.1.100.10)          │
        │  Ansible control node / kubectl     │
        └─────────────────┬───────────────────┘
                          │ SSH
        ┌─────────────────┼───────────────────┐
        │                 │                   │
   ┌────┴────┐       ┌────┴────┐         ┌────┴────┐
   │ .24     │       │ .25     │         │ .26     │
   │ server  │       │ server  │         │ server  │
   │ (etcd)  │       │ (etcd)  │         │ (etcd)  │
   └────┬────┘       └────┬────┘         └────┬────┘
        │                 │                   │
   ┌────┴────┐       ┌────┴────┐         ┌────┴────┐
   │ .27     │       │ .28     │         │ ...     │
   │ agent   │       │ agent   │         │ agent   │
   └─────────┘       └─────────┘         └─────────┘
```
> [!Warning]
> для HA-режима с встроенным etcd количество серверов должно быть нечётным (3, 5, 7).

Если третьего сервера нет, можно развернуть минимальный кластер: 1 server + 2 agent (без HA control-plane). В примере ниже показан HA-вариант (3 сервера).

### Предустановка
Установка Ansible 2.15.2 и netaddr.

