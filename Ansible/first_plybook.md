

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
