### Автоматическое резервное копирование (бэкап) критически важных логов и конфигураций с удаленных серверов.
Этот плейбук научит вас работать с архивацией, локальным хранилищем, переменными времени и скачиванием файлов с удаленных нод на управляющий сервер.

##### Идея сценария
Сценарий зайдет на удаленные хосты, соберет конфигурации Nginx и системные логи в один архив, добавит в имя архива текущую дату, а затем скачает этот бэкап к вам на управляющую машину centpca в папку /backups/.

##### Шаг 1. Создание сценария

Выполните команду на машине centpca, чтобы создать новый плейбук:
```bash
cat << 'EOF' > /etc/ansible/second_playbook.yml
---
- name: Автоматическое создание бэкапов конфигураций и логов
  hosts: servers
  become: yes
  tasks:

    - name: Создание папки для временных бэкапов на удаленных серверах
      file:
        path: /tmp/backup_zone
        state: directory
        mode: '0700'

    - name: Архивирование конфигураций Nginx и системных сообщений
      archive:
        path:
          - /etc/nginx/
          - /var/log/messages
        dest: "/tmp/backup_zone/backup-{{ ansible_hostname }}-{{ ansible_date_time.date }}.tar.gz"
        format: gz

    - name: Создание локальной папки для хранения бэкапов на управляющей машине (centpca)
      file:
        path: /backups
        state: directory
        mode: '0755'
      delegate_to: localhost
      become: yes

    - name: Скачивание созданного архива с удаленного хоста на centpca
      fetch:
        src: "/tmp/backup_zone/backup-{{ ansible_hostname }}-{{ ansible_date_time.date }}.tar.gz"
        dest: "/backups/"
        flat: yes

    - name: Удаление временного архива на удаленном сервере (очистка места)
      file:
        path: "/tmp/backup_zone/backup-{{ ansible_hostname }}-{{ ansible_date_time.date }}.tar.gz"
        state: absent
...
EOF
```
##### Шаг 2. Запуск сценария
Запустите ваш второй плейбук:
```bash
ansible-playbook /etc/ansible/second_playbook.yml
```
##### Шаг 3. Проверка результата
После успешного завершения проверьте папку /backups/ на вашей машине centpca:
```bash
ls -lh /backups/
```

<img width="1123" height="89" alt="image" src="https://github.com/user-attachments/assets/0068b3e0-414c-4540-9a55-ba23f5b319fa" />

Вы увидите два готовых архива, например:
• backup-centpc1-2026-10-19.tar.gz
• backup-localhost-2026-10-19.tar.gz
