#### Реализацию NFS в CentOS 7.

Для начала установим пакет ``nfs`` на две машины. У меня одна виртуалка называется ``cent1``, другая ``cent2``.<br>
Сервер ``nfs`` будем ставить на ``cent1``, подключаться с ``cent2``.<br>
Так как у меня DNS не настроет, подключение будет по IP адресу.<br>

Устанавливаем пакет на две виртуальные машины.

``yum install nfs-utils nfs4-acl-tools [On CentOS/RHEL]``

``apt install nfs-common nfs4-acl-tools [On Debian/Ubuntu]``

```
yum install nfs-utils
```

![image](https://github.com/tvgVita69/Linux_begin/assets/98489171/f7db2615-6fa6-4397-8883-d4e702e46c3d)

Проверяем, что всё установилось, только не запущено.

```
systemctl status {nfs,rpcbind,nfslock}
systemctl status nfs-server
```

![image](https://github.com/tvgVita69/Linux_begin/assets/98489171/a94d051b-ba75-4418-af81-b51f3eb69094)

![image](https://github.com/tvgVita69/Linux_begin/assets/98489171/9335e9f1-4a11-48d6-a185-3922660c9a82)

Далее, запускаем сервер на ``cent1``.

```
systemctl start nfs-server
```

![image](https://github.com/tvgVita69/Linux_begin/assets/98489171/7ccc11fb-9734-4ac7-abb2-ce21f8e19e65)

![image](https://github.com/tvgVita69/Linux_begin/assets/98489171/484f6dfc-2469-47ef-8bf3-1a810f924b90)

Теперь, создаём каталог, сделаем файл с надписью ``Hello, NFS!`` на ``cent1``.

```
mkdir /var/nfs/export01 -p
echo 'Hello, NFS!' > /var/nfs/export01/file.txt
cat /var/nfs/export01/file.txt
```

![image](https://github.com/tvgVita69/Linux_begin/assets/98489171/4e1ed825-4f59-42f7-8ed7-a7e7ccd21fb0)

Редактируем файл подключения.<br>
Eсли настроен dns можно прописать имя компьютера вместо IP адреса с которого будем подключаться ``cent2.local``. <br>
```
nano /etc/exports
/var/nfs/export01   192.168.85.138(sync) 
```

Осталось дело за малым, включить службу в файерволе.

```
firewall-cmd --add-service=nfs --permanent
firewall-cmd --add-service=rpc-bind --permanent
firewall-cmd --add-service=mountd --permanent
firewall-cmd --permanent --add-port=111/tcp        можно открыть порт для RPC
firewall-cmd --permanent --add-port=20048/tcp      можно открыть порт для NFS
firewall-cmd --reload
если хотите использовать iptable
iptables -t filter -A INPUT -p tcp --dport 111 -j ACCEPT
iptables -t filter -A INPUT -p tcp --dport 2049 -j ACCEPT
iptables -t filter -A INPUT -p tcp --dport 20048 -j ACCEPT
service iptables save
iptables-save
systemctl restart iptables
```

Применяем ``exportfs -a`` и проверяем.

```
exportfs
exportfs -v
exportfs -va
```

![image](https://github.com/tvgVita69/Linux_begin/assets/98489171/ea5cf174-a55a-420d-98dd-bbd69c2b496a)

Если нет ошибок переходим на вторую виртуальную машину ``cent2``. <br>
И пытаемся примонтировать диск на ``/mnt`` с удаленной машины. 

```
mount 192.168.85.139:/var/nfs/export01 /mnt
df -h   смотрим внизу должен появиться смонтированный диск.
```

![image](https://github.com/tvgVita69/Linux_begin/assets/98489171/6dbd3a42-c92e-4270-9ece-40c982062e3c)

Проверяем, что у нас есть доступ к созданному файле на виртуальной машине ``cent1`` и смотрим поледнее, что было примонтированно.

```
cat /mnt/file.txt
[root@cent2 sol]# cat /mnt/file.txt
Hello, NFS!
[root@cent2 sol]# mount | tail -n1
192.168.85.139:/var/nfs/export01 on /mnt type nfs4 (rw,relatime,vers=4.1,rsize=524288,wsize=524288,namlen=255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,clientaddr=192.168.85.138,local_lock=none,addr=192.168.85.139)
```

- *rw* – права на запись в каталоге, ro – доступ только на чтение;
- *sync* – синхронный режим доступа. async – подразумевает, что не нужно ожидать подтверждения записи на диск (повышает производительность NFS, но уменьшает надежность);
- *no_root_squash* – разрешает root пользователю с клиента получать доступ к NFS каталогу (обычно не рекомендуется);
- *no_all_squash* — включение авторизации пользователя. all_squash – доступ к ресурсам от анонимного пользователя;
- *no_subtree_check* – отключить проверку, что пользователь обращается в файлу в определенном каталоге (subtree_check – используется по умолчанию);
- *anonuid, anongid* – сопоставить NFS пользователю/группу указанному локальному пользователю/группе(UID и GID).

![image](https://github.com/tvgVita69/Linux_begin/assets/98489171/c52a5be2-9d26-4555-8d8c-7d3c53dc5ce5)

По поиску ищем список модуля ядра и какие процессы на данный момент запущены по ``nfs``.

```
lsmod | grep nfs
ps -A | grep nfs
```

![image](https://github.com/tvgVita69/Linux_begin/assets/98489171/3db34e57-2750-432c-a055-6732e4d9c16a)

Подключение настроено.

Чтобы NFS каталог автоматически монтировался при перезагрузке, нужно открыть файл ``fstab``.

```
nano /etc/fstab
добавить строку
192.168.85.139:/var/nfs/export01 /mnt/ nfs rw,sync,hard,intr 0 0
Сохранить и выйти. Далее монтируем
mount a
```

Полезные команды.

Для перезапуска сервера ``systemctl restart nfs``

Для перезапуска только службы сервера ``systemctl try-restart nfs``

Для перезагрузки конфигурационных файлов ``systemctl reload nfs``

Если редактируете файл ``nano /etc/sysconfig/nfs`` перезапуск ``systemctl restart nfs-config``

Как подключатся из Windows было ранее написано. Только дайте права на файл ``chmod -R 777 /var/nfs/exp или chmod 0777 /var/nfs/exp/*.*`` на ``cent1``.

Открываем консоль Windows (cmd или powershell) и пишем команды.

```
Для cmd - showmount -e 192.168.85.139                      
Mount -o anon \\192.168.85.139\var\nfs\exp R:

Для powershell - showmount -e 192.168.85.139
монтируем
New-PSdrive -PSProvider FileSystem -Name G -Root \\192.168.85.139\var\nfs\export01 -Persist
диск доступен с командной строке
```

![image](https://github.com/tvgVita69/Linux_begin/assets/98489171/76f314fb-25d1-4a55-8053-d588b0eff2e4)

![image](https://github.com/tvgVita69/Linux_begin/assets/98489171/29f4017e-84da-4d7e-b9e5-8d8c85b060dc)

- *-o anon* - подключаться с правами анонимного пользователя;
- *\\192.168.85.139" - имя или IP адрес NFS-сервера;
- *\var\nfs\exp" - локальный путь к каталогу на NFS-сервере;
- *G* - буква диска Windows

По UID и GUI. По желанию. Работает и без правки.

![image](https://github.com/tvgVita69/Linux_begin/assets/98489171/0938befb-a1b9-4ec1-bfc5-7e95abe82cb9)

Редактируем реестр.

![image](https://github.com/tvgVita69/Linux_begin/assets/98489171/3310c7c4-1094-4c15-9fcd-3cf0479e8fd5)

Добавляем два параметра в реестр. Значение десятичное с полными правами ``root`` 0.

![image](https://github.com/tvgVita69/Linux_begin/assets/98489171/8d3a5962-c322-4b92-a4ea-98a4c9bfbd6e)

```
AnonymousUid
AnonymousGid
```
Далее перезагружаем клиентский компьютер, тот где добавляли данные в реестр.

Вот это рестартить бесполезно. ``gpupdate /force`` и прочие манипуляции бесполезны.<br> Будет всё равно ошибка подключения Ошибка сети - 85.

![image](https://github.com/tvgVita69/Linux_begin/assets/98489171/8560e868-cc9c-4a7e-b57b-f859e01d7bb0)

![image](https://github.com/tvgVita69/Linux_begin/assets/98489171/235209d8-37ff-406e-b4ef-731843833456)


Результат, после перезагрузки.

![image](https://github.com/tvgVita69/Linux_begin/assets/98489171/e7e1e3fb-6605-489b-96b9-1c95c981560c)

![image](https://github.com/tvgVita69/Linux_begin/assets/98489171/2fcaa135-5afd-46fa-b971-cfadc549c1a2)

Можно читать и записывать, полные права.

![image](https://github.com/tvgVita69/Linux_begin/assets/98489171/e5b24e33-82a2-4232-a134-4fbd3c7e71a6)

![image](https://github.com/tvgVita69/Linux_begin/assets/98489171/95208f46-638d-4f9f-97c1-e2b07ab1bb85)

Современные Linux системы давно работают с UTF-8, в то время как Windows продолжает использовать региональные кодовые страницы, например, CP-1251 для русского языка. 

Это приводит к тому, что имена файлов, набранные кириллицей (или любыми иными национальными символами) отображаются "крякозябликами". 

Сами файлы при этом доступны и могут быть отредактированы.

![image](https://github.com/user-attachments/assets/70b12b44-4071-47fa-8a0b-3787257b1f6e)

Если же мы со стороны Windows поместим на NFS-ресурс файл с кириллицей в имени, то со стороны Linux мы увидим веселые ромбики.

В качестве решения можно найти совет включить поддержку UTF-8 в Windows, которая пока находится в состоянии бета. 

Эта возможность доступа в языковых настройках панели управления.

![image](https://github.com/user-attachments/assets/eeec2421-833f-4ffc-98b8-86a107bd2d3e)

Но это решение из разряда "одно лечим - другое калечим" и покалечено будет гораздо больше, чем вылечено. 

Дело в том, что достаточно большое количество вполне современных программ ничего не знают об UTF-8 и не умеют с ним работать, в итоге веселые ромбики начнут попадаться вам в самых неожиданных местах вашей системы.

Поэтому, используя NFS-клиент для Windows следует четко понимать все плюсы, минусы и имеющиеся недостатки. 

Но в целом появление поддержки NFS в Windows - хорошо, так как делает поддержку гетерогенных сред проще.
