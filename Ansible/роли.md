### Роли (Ansible Roles)
В реальном продакшене никто не пишет один огромный файл плейбука. Вместо этого код разделяют на роли (например, отдельно роль nginx, отдельно роль common_env). Это позволяет повторно использовать код и не путаться в сотнях строк YAML.
Мы создадим структуру ролей и перенесем туда нашу конфигурацию.

##### Шаг 1. Создание структуры директорий для ролей

В Ansible существует стандарт де-факто для структуры папок. Создадим роль с именем webserver с помощью встроенной утилиты ansible-galaxy:
```bash
mkdir -p /etc/ansible/roles
cd /etc/ansible/roles
ansible-galaxy init webserver --offline
```
Эта команда автоматически создаст правильную структуру папок для роли webserver:
• tasks/main.yml — здесь будут храниться сами задачи (установка, запуск).
• templates/ — сюда мы перенесем наш шаблон index.html.j2.

##### Шаг 2. Заполнение роли webserver

Теперь перенесем логику настройки веб-сервера в соответствующие файлы созданной роли.
##### 1. Запишем задачи в tasks/main.yml:
```bash
cat << 'EOF' > /etc/ansible/roles/webserver/tasks/main.yml
---
# tasks file for webserver
- name: Установка веб-сервера nginx
  yum:
    name: nginx
    state: present

    # Мы используем созданный ранее шаблон, но теперь Ansible автоматически 
    # будет искать его в папке templates/ внутри самой роли.
- name: Генерация index.html из шаблона Jinja2
  template:
    src: index.html.html.j2
    dest: /usr/share/nginx/html/index.html

- name: Запуск и включение автозагрузки nginx
  service:
    name: nginx
    state: started
    enabled: yes
EOF
```
##### 2. Перенесем наш шаблон Jinja2 внутрь роли:
```bash
mkdir -p /etc/ansible/roles/webserver/templates
cat << 'EOF' > /etc/ansible/roles/webserver/templates/index.html.html.j2
<html>
<body style="font-family: Arial, sans-serif; background-color: #f4f6f9; text-align: center; padding: 50px;">
    <div style="background: white; max-width: 500px; margin: auto; padding: 20px; border-radius: 8px; box-shadow: 0 4px 10px rgba(0,0,0,0.1);">
        <h1 style="color: #4b6cb7;">🚀 Развернуто через Ansible Roles!</h1>
        <p>Имя хоста: <b>{{ ansible_hostname }}</b></p>
        <p>IP-адрес: <code>{{ inventory_hostname }}</code></p>
    </div>
</body>
</html>
EOF
```
##### Шаг 3. Создание главного плейбука (Main Playbook)
Теперь наш главный файл сценария станет удивительно коротким и понятным. Вместо описания десятков задач мы просто укажем, какую роль применить к серверам.

Создайте файл /etc/ansible/site.yml:
```bash
cat << 'EOF' > /etc/ansible/site.yml
---
- name: Применение конфигурации веб-серверов через роли
  hosts: servers
  become: yes
  roles:
    - webserver
...
EOF
```
##### Шаг 4. Запуск
Запустите ваш новый структурированный плейбук:
```bash
ansible-playbook /etc/ansible/site.yml
```

> В чем главный плюс такого подхода?
> - Если завтра вам понадобится добавить еще 10 серверов или развернуть аналогичный веб-сервер в другом проекте, вам не придется копировать куски YAML-кода. Вы просто подключите готовую роль webserver одной строкой в любом новом плейбуке.


##### Переменные по умолчанию (Defaults)

Раз уж мы перешли на роли, давайте освоим еще одну мощную фишку — переменные по умолчанию (defaults).

рямо сейчас заголовок нашей страницы жестко зашит в шаблоне. Но что, если мы хотим сделать роль универсальной, чтобы заголовок сайта можно было легко менять прямо из главного плейбука, не ковыряя код внутри роли?

Для этого в структуре роли есть папка defaults/main.yml.
##### 1. Добавим переменную по умолчанию в роль:
```bash
cat << 'EOF' > /etc/ansible/roles/webserver/defaults/main.yml
---
# Переменные по умолчанию для роли webserver
web_page_title: "Развернуто через Ansible Roles!"
EOF
```
##### 2. Отредактируем шаблон Jinja2, чтобы он использовал эту переменную:
```bash
cat << 'EOF' > /etc/ansible/roles/webserver/templates/index.html.html.j2
<html>
<body style="font-family: Arial, sans-serif; background-color: #f4f6f9; text-align: center; padding: 50px;">
    <div style="background: white; max-width: 500px; margin: auto; padding: 20px; border-radius: 8px; box-shadow: 0 4px 10px rgba(0,0,0,0.1);">
        <h1 style="color: #4b6cb7;">🚀 {{ web_page_title }}</h1>
        <p>Имя хоста: <b>{{ ansible_hostname }}</b></p>
        <p>IP-адрес: <code>{{ inventory_hostname }}</code></p>
    </div>
</body>
</html>
EOF
```
##### 3. Переопределим переменную в главном плейбуке:
Теперь в файле site.yml мы можем легко переписать этот заголовок для нашей текущей инфраструктуры. Изменим файл:
```bash
cat << 'EOF' > /etc/ansible/site.yml
---
- name: Применение конфигурации веб-серверов через роли
  hosts: servers
  become: yes
  vars:
    web_page_title: "Мой крутой кластер балансировки v2!"
  roles:
    - webserver
...
EOF
```
##### 4. Запустите плейбук повторно:
```bash
ansible-playbook /etc/ansible/site.yml
```













































