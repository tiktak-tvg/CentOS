### Сервер CI/CD (Jenkins): Автоматизация сборки и развертывания ваших проектов. 
Роль установит Jenkins, необходимые плагины и настроит его для работы с репозиториями кода.
#### ЭТАП 0. Инвентарь (/etc/ansible/hosts)
```bash
cat <<'EOF' >> /etc/ansible/hosts

[jenkins]
10.1.100.22 ansible_user=root
EOF
```
Проверка:

```bash
Если настраивали ключ, то выполняем это действие
ssh-copy-id root@10.1.100.22

ssh root@10.1.100.15 'hostname'

ansible jenkins -m ping
```
#### ЭТАП 1. Структура роли
```bash
mkdir -p /etc/ansible/roles/jenkins/{tasks,handlers,defaults}
```
#### ЭТАП 2. defaults/main.yml
```bash
cat <<'EOF' > /etc/ansible/roles/jenkins/defaults/main.yml
---
# Параметры Jenkins
jenkins_port: 8080
jenkins_java_home: "/usr/lib/jvm/temurin-21-jdk"

# Плагины для установки через CLI
# ID плагинов: https://plugins.jenkins.io/
jenkins_plugins:
  - git
  - workflow-aggregator
  - ansible
  - ssh-agent
  - credentials-binding
  - pipeline-stage-view
  - blueocean
...

EOF
```
#### ЭТАП 3. tasks/main.yml (полностью исправленный)
```bash
cat <<'EOF' > /etc/ansible/roles/jenkins/tasks/main.yml
---
# ============================================================
# 1. Установка Temurin JDK 21 (требуется для Jenkins 2.555+)
# ============================================================
- name: Добавление репозитория Adoptium (Temurin)
  copy:
    dest: /etc/yum.repos.d/adoptium.repo
    content: |
      [Adoptium]
      name=Adoptium
      baseurl=https://packages.adoptium.net/artifactory/rpm/centos/7/$basearch
      enabled=1
      gpgcheck=1
      gpgkey=https://packages.adoptium.net/artifactory/api/gpg/key/public
    mode: '0644'

- name: Импорт GPG-ключа Adoptium
  rpm_key:
    key: https://packages.adoptium.net/artifactory/api/gpg/key/public
    state: present

- name: Установка Temurin JDK 21 и fontconfig
  yum:
    name:
      - fontconfig
      - temurin-21-jdk
    state: present

# ============================================================
# 2. Репозиторий и установка Jenkins
# ============================================================
- name: Добавление Jenkins репозитория
  get_url:
    url: https://pkg.jenkins.io/redhat-stable/jenkins.repo
    dest: /etc/yum.repos.d/jenkins.repo
    mode: '0644'

- name: Импорт GPG-ключа Jenkins
  rpm_key:
    key: https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
    state: present

- name: Установка Jenkins
  yum:
    name: jenkins
    state: present

# ============================================================
# 3. systemd override: порт + JAVA_HOME
# ============================================================
- name: Создание директории для systemd override
  file:
    path: /etc/systemd/system/jenkins.service.d
    state: directory
    mode: '0755'

- name: Override systemd для настройки порта и JAVA_HOME
  copy:
    dest: /etc/systemd/system/jenkins.service.d/override.conf
    content: |
      [Service]
      Environment="JENKINS_PORT={{ jenkins_port }}"
      Environment="JAVA_HOME={{ jenkins_java_home }}"
    mode: '0644'
  notify: restart jenkins

- name: Перезагрузка systemd daemon
  systemd:
    daemon_reload: yes

# ============================================================
# 4. Первый запуск Jenkins
# ============================================================
- name: Сброс failed-статуса jenkins (если был)
  command: systemctl reset-failed jenkins
  changed_when: false
  failed_when: false

- name: Запуск и включение Jenkins
  systemd:
    name: jenkins
    state: started
    enabled: yes

# ============================================================
# 5. Настройка глобального URL Jenkins (критично для CLI)
# ============================================================
- name: Настройка глобального URL Jenkins
  copy:
    dest: /var/lib/jenkins/jenkins.model.JenkinsLocationConfiguration.xml
    content: |
      <?xml version='1.1' encoding='UTF-8'?>
      <jenkins.model.JenkinsLocationConfiguration>
        <jenkinsUrl>http://{{ ansible_default_ipv4.address }}:{{ jenkins_port }}/</jenkinsUrl>
      </jenkins.model.JenkinsLocationConfiguration>
    owner: jenkins
    group: jenkins
    mode: '0644'
  notify: restart jenkins

# Применяем перезапуск немедленно, чтобы Jenkins прочитал URL
- name: Применение настроек Jenkins (flush handlers)
  meta: flush_handlers

- name: Ожидание запуска Jenkins после настройки URL
  wait_for:
    port: "{{ jenkins_port }}"
    delay: 10
    timeout: 180

- name: Пауза для завершения инициализации
  pause:
    seconds: 20

# ============================================================
# 6. Firewalld
# ============================================================
- name: Открытие порта {{ jenkins_port }} в firewalld
  firewalld:
    port: "{{ jenkins_port }}/tcp"
    permanent: yes
    state: enabled
    immediate: yes

# ============================================================
# 7. Чтение initialAdminPassword
# ============================================================
- name: Чтение initialAdminPassword
  slurp:
    src: /var/lib/jenkins/secrets/initialAdminPassword
  register: jenkins_initial_password
  until: jenkins_initial_password is succeeded
  retries: 10
  delay: 10

# ============================================================
# 8. Скачивание Jenkins CLI
# ============================================================
- name: Скачивание Jenkins CLI
  get_url:
    url: "http://{{ ansible_default_ipv4.address }}:{{ jenkins_port }}/jnlpJars/jenkins-cli.jar"
    dest: /opt/jenkins-cli.jar
    mode: '0644'
    timeout: 120

# ============================================================
# 9. Установка плагинов через Jenkins CLI
#    ВАЖНО: используем ansible_default_ipv4.address (не localhost),
#    чтобы origin запроса совпал с jenkinsUrl
# ============================================================
- name: Установка плагинов через Jenkins CLI
  shell: |
    java -jar /opt/jenkins-cli.jar \
      -s http://{{ ansible_default_ipv4.address }}:{{ jenkins_port }}/ \
      -auth admin:{{ jenkins_initial_password.content | b64decode | trim }} \
      install-plugin {{ item }} -deploy
  loop: "{{ jenkins_plugins }}"
  register: plugin_install
  until: plugin_install.rc == 0
  retries: 3
  delay: 15
  changed_when: plugin_install.rc == 0

# ============================================================
# 10. Перезапуск после установки плагинов
# ============================================================
- name: Перезапуск Jenkins после установки плагинов
  systemd:
    name: jenkins
    state: restarted

- name: Ожидание Jenkins после перезапуска
  wait_for:
    port: "{{ jenkins_port }}"
    delay: 10
    timeout: 180
...
EOF
```

> [!WARNING]
> Если вы уже авторизовались и переустанавливаете, то конфиг будет такой (замените блок с 7 по 10 (после firewalld) на следующий)

```bash
cat <<'EOF' > /etc/ansible/roles/jenkins/tasks/main.yml
---
# ============================================================
# 1. Установка Temurin JDK 21
# ============================================================
- name: Добавление репозитория Adoptium (Temurin)
  copy:
    dest: /etc/yum.repos.d/adoptium.repo
    content: |
      [Adoptium]
      name=Adoptium
      baseurl=https://packages.adoptium.net/artifactory/rpm/centos/7/$basearch
      enabled=1
      gpgcheck=1
      gpgkey=https://packages.adoptium.net/artifactory/api/gpg/key/public
    mode: '0644'

- name: Импорт GPG-ключа Adoptium
  rpm_key:
    key: https://packages.adoptium.net/artifactory/api/gpg/key/public
    state: present

- name: Установка Temurin JDK 21 и fontconfig
  yum:
    name:
      - fontconfig
      - temurin-21-jdk
    state: present

# ============================================================
# 2. Репозиторий и установка Jenkins
# ============================================================
- name: Добавление Jenkins репозитория
  get_url:
    url: https://pkg.jenkins.io/redhat-stable/jenkins.repo
    dest: /etc/yum.repos.d/jenkins.repo
    mode: '0644'

- name: Импорт GPG-ключа Jenkins
  rpm_key:
    key: https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
    state: present

- name: Установка Jenkins
  yum:
    name: jenkins
    state: present

# ============================================================
# 3. systemd override: порт + JAVA_HOME
# ============================================================
- name: Создание директории для systemd override
  file:
    path: /etc/systemd/system/jenkins.service.d
    state: directory
    mode: '0755'

- name: Override systemd для настройки порта и JAVA_HOME
  copy:
    dest: /etc/systemd/system/jenkins.service.d/override.conf
    content: |
      [Service]
      Environment="JENKINS_PORT={{ jenkins_port }}"
      Environment="JAVA_HOME={{ jenkins_java_home }}"
    mode: '0644'
  notify: restart jenkins

- name: Перезагрузка systemd daemon
  systemd:
    daemon_reload: yes

# ============================================================
# 4. Первый запуск Jenkins
# ============================================================
- name: Сброс failed-статуса jenkins (если был)
  command: systemctl reset-failed jenkins
  changed_when: false
  failed_when: false

- name: Запуск и включение Jenkins
  systemd:
    name: jenkins
    state: started
    enabled: yes

# ============================================================
# 5. Настройка глобального URL Jenkins (критично для CLI)
# ============================================================
- name: Настройка глобального URL Jenkins
  copy:
    dest: /var/lib/jenkins/jenkins.model.JenkinsLocationConfiguration.xml
    content: |
      <?xml version='1.1' encoding='UTF-8'?>
      <jenkins.model.JenkinsLocationConfiguration>
        <jenkinsUrl>http://{{ ansible_default_ipv4.address }}:{{ jenkins_port }}/</jenkinsUrl>
      </jenkins.model.JenkinsLocationConfiguration>
    owner: jenkins
    group: jenkins
    mode: '0644'
  notify: restart jenkins

- name: Применение настроек Jenkins (flush handlers)
  meta: flush_handlers

- name: Ожидание запуска Jenkins после настройки URL
  wait_for:
    port: "{{ jenkins_port }}"
    delay: 10
    timeout: 180

- name: Пауза для завершения инициализации
  pause:
    seconds: 20

# ============================================================
# 6. Firewalld
# ============================================================
- name: Открытие порта {{ jenkins_port }} в firewalld
  firewalld:
    port: "{{ jenkins_port }}/tcp"
    permanent: yes
    state: enabled
    immediate: yes

# ============================================================
# 7. Проверка наличия initialAdminPassword (только после 1-й установки)
# ============================================================
- name: Проверка существования initialAdminPassword
  stat:
    path: /var/lib/jenkins/secrets/initialAdminPassword
  register: jenkins_initial_password_file

- name: Jenkins уже прошёл setup wizard (файл пароля отсутствует)
  debug:
    msg: "initialAdminPassword не найден — значит, setup wizard уже пройден. Установка плагинов пропущена."
  when: not jenkins_initial_password_file.stat.exists

# ============================================================
# 8. Установка плагинов (ТОЛЬКО при первом запуске)
# ============================================================
- name: Чтение initialAdminPassword
  slurp:
    src: /var/lib/jenkins/secrets/initialAdminPassword
  register: jenkins_initial_password
  when: jenkins_initial_password_file.stat.exists

- name: Скачивание Jenkins CLI
  get_url:
    url: "http://{{ ansible_default_ipv4.address }}:{{ jenkins_port }}/jnlpJars/jenkins-cli.jar"
    dest: /opt/jenkins-cli.jar
    mode: '0644'
    timeout: 120
  when: jenkins_initial_password_file.stat.exists

- name: Установка плагинов через Jenkins CLI
  shell: |
    java -jar /opt/jenkins-cli.jar \
      -s http://{{ ansible_default_ipv4.address }}:{{ jenkins_port }}/ \
      -auth admin:{{ jenkins_initial_password.content | b64decode | trim }} \
      install-plugin {{ item }} -deploy
  loop: "{{ jenkins_plugins }}"
  when: jenkins_initial_password_file.stat.exists
  register: plugin_install
  until: plugin_install.rc == 0
  retries: 3
  delay: 15
  changed_when: plugin_install.rc == 0

# ============================================================
# 9. Перезапуск после установки плагинов (только если ставили)
# ============================================================
- name: Перезапуск Jenkins после установки плагинов
  systemd:
    name: jenkins
    state: restarted
  when: jenkins_initial_password_file.stat.exists

- name: Ожидание Jenkins после перезапуска
  wait_for:
    port: "{{ jenkins_port }}"
    delay: 10
    timeout: 180
...
EOF
```
#### ЭТАП 4. handlers/main.yml
```bash
cat <<'EOF' > /etc/ansible/roles/jenkins/handlers/main.yml
---
- name: restart jenkins
  systemd:
    name: jenkins
    state: restarted
    daemon_reload: yes
...
EOF
```
#### ЭТАП 5. jenkins.yml (главный плейбук)
```bash
cat <<'EOF' > /etc/ansible/jenkins.yml
---
- name: Развертывание Jenkins CI (Java 21 + плагины)
  hosts: jenkins
  become: yes
  roles:
    - jenkins
...
EOF
```
#### ЭТАП 6. Запуск
```bash
ansible-playbook /etc/ansible/jenkins.yml
```
##### Что произойдёт (по шагам):
- Добавится репозиторий Adoptium, установится Temurin JDK 21.
- Добавится репозиторий Jenkins, установится jenkins.
- Создастся systemd override с портом 8080 и JAVA_HOME.
- Jenkins запустится на Java 21.
- Создастся JenkinsLocationConfiguration.xml с URL http://10.1.100.22:8080/.
- Jenkins перезапустится (через flush_handlers).
- Откроется порт 8080 в firewalld.
- Ansible прочитает initialAdminPassword через slurp.
- Скачает jenkins-cli.jar с того же URL.
- Установит плагины через CLI (git, ansible, blueocean и др.).
- Перезапустит Jenkins — плагины активируются.
- Ожидаемый PLAY RECAP: failed=0.

<img width="1115" height="1124" alt="image" src="https://github.com/user-attachments/assets/d1030933-c362-4cde-8e5f-867f30b0b368" />

#### ЭТАП 7. Первый вход в браузере
Получите пароль:

```bash
ansible jenkins -m shell -a "cat /var/lib/jenkins/secrets/initialAdminPassword"
```
- Откройте http://10.1.100.22:8080
- Вставьте пароль.
- Выберите «Install suggested plugins» (Jenkins пропустит уже установленные).
- Создайте свою учётную запись администратора (логин/пароль запомните).

<img width="1341" height="573" alt="image" src="https://github.com/user-attachments/assets/39e82424-a580-4862-8518-04df9321bcfc" />

#### ЭТАП 8. Проверка после развёртывания
```bash
# 1. Java 21 установлена
ansible jenkins -m shell -a "java -version"
# Ожидаемо: openjdk version "21.0.x"

# 2. Jenkins активен
ansible jenkins -m shell -a "systemctl is-active jenkins"
# Ожидаемо: active

# 3. Порт слушается
ansible jenkins -m shell -a "ss -tlnp | grep :8080"
# Ожидаемо: *:8080 (java)

# 4. URL в конфиге
ansible jenkins -m shell -a "cat /var/lib/jenkins/jenkins.model.JenkinsLocationConfiguration.xml | grep jenkinsUrl"
# Ожидаемо: <jenkinsUrl>http://10.1.100.22:8080/</jenkinsUrl>

# 5. Плагины установлены
ansible jenkins -m shell -a "ls /var/lib/jenkins/plugins/ | wc -l"
# Ожидаемо: > 7 (по числу из jenkins_plugins + зависимости)

# 6. Jenkins отвечает 200
ansible jenkins -m shell -a "curl -s -o /dev/null -w '%{http_code}' http://10.1.100.22:8080/login"
# Ожидаемо: 200
```

<img width="1112" height="358" alt="image" src="https://github.com/user-attachments/assets/d3ecca91-8b49-4378-b8a1-c773607cd42c" />

#### ЭТАП 9. Первая задача: сборка проекта
##### 9.1. Через веб-интерфейс
- На главной странице Jenkins → «Создать Item» (или New Item).
- Имя: my-project-build, тип: Freestyle project.

<img width="1135" height="974" alt="image" src="https://github.com/user-attachments/assets/0078270d-590c-4d77-a0a6-1a09c736f4d5" />

<img width="1278" height="1113" alt="image" src="https://github.com/user-attachments/assets/b34f5380-4049-46a9-b575-6b0c0f13e69b" />

<img width="1294" height="951" alt="image" src="https://github.com/user-attachments/assets/e178015a-4b03-48b4-908e-d8fde37e5f1c" />

- Раздел Source Code Management → Git:
- Repository URL: https://10.1.100.15/root/test-project.git (ваш GitLab).
- Credentials: добавьте логин/пароль/ от GitLab через Add → Jenkins.

  <img width="558" height="626" alt="image" src="https://github.com/user-attachments/assets/1a32d6d7-682b-481f-ad6f-8135963df710" />

    - Нажмите на Username with password (он уже выделен на скриншоте).
    - В открывшемся окне заполните:
    - Scope: Global (Jenkins, nodes, items, all child items, etc) — как мы обсуждали, чтобы креды были видны везде.
    - Username: root (ваш логин в GitLab).
    - Password: здесь обязательно используйте Personal Access Token (PAT), а не обычный пароль от GitLab.
  
  <img width="556" height="663" alt="image" src="https://github.com/user-attachments/assets/6a7901b8-0259-4a08-8b24-566b24383310" />

    - Как его получить: зайдите в GitLab (под root) → Preferences → Access Tokens → создайте токен с правами read_repository (и api, если планируете больше интеграций) → скопируйте его.
 
  <img width="1254" height="1192" alt="image" src="https://github.com/user-attachments/assets/2359c6dc-8647-48b3-80c6-ce0054f2e737" />
    
    - ID: можно оставить пустым, Jenkins сгенерирует сам, или задать понятное имя, например gitlab-root-token.
    - Description: например, Доступ к GitLab для сборки.
    - Нажмите Create.

<img width="1126" height="480" alt="image" src="https://github.com/user-attachments/assets/238415b8-f973-4caf-aecd-86bfd92f2c03" />


Раздел Build Triggers → отметьте GitHub hook trigger for GITScm polling (если используете вебхуки).
```bash
[root@centpc09 jenkins]# sudo -H -u jenkins git config --global http.sslVerify false
[root@centpc09 jenkins]# sudo -H -u jenkins git config --global --list | grep -i sslverify
http.sslverify=false
[root@centpc09 jenkins]# getent passwd jenkins
jenkins:x:993:991:Jenkins Automation Server:/var/lib/jenkins:/bin/false
[root@centpc09 jenkins]#
```
Раздел Build Steps → Add build step → Execute shell:

```bash
echo "Building project..."
ls -la
cat README.md || echo "No README"
Save → Build Now.
```
##### 9.2. Через Jenkins Pipeline (Jenkinsfile в GitLab)
В вашем GitLab-репозитории test-project создайте файл Jenkinsfile:

```groovy
pipeline {
    agent any
    stages {
        stage('Checkout') {
            steps {
                echo 'Клонирование репозитория...'
                checkout scm
            }
        }
        stage('Build') {
            steps {
                echo 'Сборка проекта...'
                sh 'ls -la'
            }
        }
        stage('Test') {
            steps {
                echo 'Запуск тестов...'
                sh 'echo "Tests passed"'
            }
        }
    }
    post {
        success {
            echo 'Сборка успешна!'
        }
        failure {
            echo 'Сборка упала!'
        }
    }
}
```
В Jenkins создайте новый Pipeline job, укажите репозиторий Git и путь к Jenkinsfile. При каждом коммите (если настроен вебхук) или по кнопке Build Now Jenkins выполнит пайплайн.

#### Итоговая архитектура
```text
┌────────────────────────────────────────────────────────────────┐
│  10.1.100.22 (Jenkins CI)                                      │
│                                                                │
│  Java 21 (Temurin)                                             │
│  Jenkins 2.555+ LTS                                            │
│  Порт: 8080                                                    │
│  Плагины: git, ansible, blueocean, ssh-agent, ...              │
│                                                                │
│  Источники кода:                                               │
│    ├─→ GitLab (10.1.100.15:443)                                │
│    └─→ GitHub (внешний, если нужно)                            │
│                                                                │
│  Запускает:                                                    │
│    ├─→ Ansible playbooks (деплой на ноды)                      │
│    └─→ Shell-скрипты (сборка, тесты)                           │
└────────────────────────────────────────────────────────────────┘
```
#### Итог
Все конфиги для Jenkins теперь полностью рабочие с нуля. Учтены:
- Java 21 (Temurin из Adoptium) — требование Jenkins 2.555+
- systemd override — правильный способ настройки порта
- JAVA_HOME явно прописан
- JenkinsLocationConfiguration.xml — CLI работает без 403
- CLI подключается по IP, а не localhost — origin совпадает
- flush_handlers — URL применяется до CLI-команд
- systemctl reset-failed — сбрасывает прошлые ошибки

> [!TOP]
> Jenkins разворачивается одной командой ansible-playbook /etc/ansible/jenkins.yml и сразу готов к работе.

```text
┌─────────────────┐      git push       ┌─────────────────┐
│   Разработчик   │────────────────────→│     GitLab      │
│  (centManager)  │                     │  10.1.100.15    │
└─────────────────┘                     └────────┬────────┘
                                                 │ webhook
                                                 │ (push events)
                                                 ▼
                                        ┌─────────────────┐
                                        │     Jenkins     │
                                        │  10.1.100.22    │
                                        │                 │
                                        │  Java 21        │
                                        │  GitLab Plugin  │
                                        └────────┬────────┘
                                                 │
                                                 ▼
                                        ┌─────────────────┐
                                        │  Build #N       │
                                        │  ✅ SUCCESS     │
                                        └─────────────────┘
```
Время от git push до завершения сборки: ~12 секунд. Это уровень production CI/CD.
