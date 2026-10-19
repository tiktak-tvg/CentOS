#### В реальном продакшене рассинхронизация времени на серверах критична (могут ломаться логи, сертификаты HTTPS и базы данных). Давайте за пару минут напишем third_playbook.yml, который автоматически синхронизирует время на всех серверах с помощью утилиты chrony.

##### Принудительная установка даты в обход демонов

Мы напишем скрипт, который возьмет текущее точное время с вашей мастер-машины и жестко запишет его в систему хоста 10.1.100.13 через команду date -s.
##### 1. Создайте плейбук принудительной установки времени:
```bash
cat << 'EOF' > /etc/ansible/force_time.yml
---
- name: Принудительная настройка времени и синхронизация
  hosts: servers
  become: yes
  gather_facts: no

  tasks:
    - name: Установка часового пояса Europe/Moscow
      timezone:
        name: Europe/Moscow

    - name: Установка пакета chrony
      yum:
        name: chrony
        state: present

    - name: Запуск и включение службы chronyd
      service:
        name: chronyd
        state: started
        enabled: yes

    - name: Принудительная синхронизация времени (сброс смещения)
      command: chronyc makestep
      changed_when: false

    - name: Ожидание синхронизации (3 секунды)
      pause:
        seconds: 3

    - name: Запись текущего времени в аппаратные часы (RTC)
      command: hwclock --systohc
      changed_when: false

    - name: Проверка статуса синхронизации
      command: chronyc tracking
      register: chrony_status
      changed_when: false

    - name: Вывод статуса синхронизации
      debug:
        msg: "{{ chrony_status.stdout_lines }}"
```
##### 2. Запустите сценарий:
```bash
ansible-playbook /etc/ansible/force_time.yml
```
##### 3. Проверка
После выполнения проверьте, выровнялись ли часы на серверах:
```bash
ansible servers -m command -a "date"
```
<img width="1059" height="704" alt="image" src="https://github.com/user-attachments/assets/ebe131a2-a355-4ead-9cbb-74d2027ba7ea" />
