Делаем систему по схеме

<img width="737" height="328" alt="image" src="https://github.com/user-attachments/assets/7d4c44b8-6e49-4c7b-b11d-7efa05d25742" />

Слой   	| Компонент	                 |Порт
:--|--:     :--|--:
Отказоустойчивость |	Keepalived (VIP 10.1.100.100)	VRRP  |	(112)
Балансировка	 |	HAProxy (roundrobin)	 |	80
Мониторинг	 |	HAProxy stats	 |	8404
Веб-сервер (системный)	 |	nginx	 |	81
Веб-сервер (контейнер)	 |	nginx:1.21 в Docker	 |	8080
Контент	 |	Шаблон Jinja2 (/usr/share/nginx/html) |		—
