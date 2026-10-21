### Чтобы у вас появились красивые, сохраняемые графики, которые не пропадут, нужно поставить Grafana.
Она подключится к вашему Prometheus и покажет все эти же метрики в разы красивее и удобнее.

Вот команды для установки Grafana на вашу управляющую машину (centpca):
```bash
# 1. Скачать и установить Grafana
wget https://dl.grafana.com/oss/release/grafana-10.4.2-1.x86_64.rpm
yum localinstall grafana-10.4.2-1.x86_64.rpm -y

# 2. Запустить сервис
systemctl daemon-reload
systemctl enable --now grafana-server

# 3. Открыть порт в firewalld
firewall-cmd --permanent --add-port=3000/tcp
firewall-cmd --reload
```
После этого зайдите в браузере по адресу http://10.1.100.10:3000 (логин и пароль: admin, admin).
```txt
В меню слева выберите Connections -> Data sources -> Add data source -> Prometheus.

В поле Connection URL укажите: http://10.1.100.10:9091

Нажмите Save & test (должно появиться "Data source is working").

В меню слева выберите Dashboards -> Import.

В поле Import via grafana.com введите 1860 и нажмите Load.

Выберите ваш Prometheus и нажмите Import.
```
У вас появится огромный красивый дашборд с графиками CPU, памяти, дисков и сети для обеих нод. 

Он будет сохраняться в базе Grafana, и вы сможете открывать его с любого компьютера.

<img width="1369" height="900" alt="36" src="https://github.com/user-attachments/assets/506a9e07-2f94-40d7-b82f-744f6180422a" />

Дашборд Node Exporter Full импортировался, но он пуст («No data», «N/A»), потому что Grafana пока не знает, откуда брать данные.

Обратите внимание на выпадающий список Datasource в самом верху слева — там написано No data sources found (или он пустой). Grafana нужно вручную связать с вашим Prometheus.
