
# Инструкция по настройке Master и Slave серверов mysql на локальной машине под windows для домашнего задания к уроку №6.
* Скачиваем Community Server в ZIP архиве (https://dev.mysql.com/downloads/mysql/)
* Распаковываем в какую-нибудь папку (с коротким путём), я распаковал в c:\temp\mysql. Соотвественно, делать все операции буду исходя из этой папки.
* Нам нужно создать два конфигурационных файла для серверов. Один для Master (слушает порт 3310, папка с базой data-master), другой для Slave (слушает порт 3320, папка с базой 3320). Идём в папку etc и создаём там конфиги my1.ini и my2.ini

Я решил использовать базу employees для экспериментов, поэтому её и указываю. Если у вас другая - пишите другую.

my1.ini
```ini
[mysqld]
basedir=C:\\Temp\\mysql
datadir=C:\\Temp\\mysql\\data-master
port=3310

server-id=1
binlog_do_db=employees
```

my2.ini
```ini
[mysqld]
basedir=C:\\Temp\\mysql
datadir=C:\\Temp\\mysql\\data-slave
port=3320

server-id=2
binlog_do_db=employees
```

# Инициализация
Инициализируем базы данных. Во время инициализации сервер создаёт системную БД и таблицы.
- Открываем cmd.exe, переходим к бинарникам: cd c:\temp\mysql\bin
- Запускаем инициализацию для первого (мастер) сервера с указанием его конфига: 
```
c:\Temp\mysql\bin>mysqld --defaults-file=C:\Temp\mysql\etc\my1.ini --initialize --console

2018-12-14T10:06:50.769491Z 0 [System] [MY-013169] [Server] c:\Temp\mysql\bin\mysqld.exe (mysqld 8.0.13) initializing of server in progress as process 6284
2018-12-14T10:06:53.885400Z 5 [Note] [MY-010454] [Server] A temporary password is generated for root@localhost: qCTaBeraH8=J
2018-12-14T10:06:54.896882Z 0 [System] [MY-013170] [Server] c:\Temp\mysql\bin\mysqld.exe (mysqld 8.0.13) initializing of server has completed
```
- в строчке вывода с отметкой [Note] выводится сгенерированный временный пароль сервера (в случае выше **qCTaBeraH8=J**). Запомните его.
- инициализируем слейв сервер, с указанием его конфига:
```
c:\Temp\mysql\bin>mysqld --defaults-file=C:\Temp\mysql\etc\my2.ini --initialize --console

2018-12-14T10:08:52.991767Z 0 [System] [MY-013169] [Server] c:\Temp\mysql\bin\mysqld.exe (mysqld 8.0.13) initializing of server in progress as process 6376
2018-12-14T10:08:56.050518Z 5 [Note] [MY-010454] [Server] A temporary password is generated for root@localhost: QpP5s+a(mwZf
2018-12-14T10:08:57.091685Z 0 [System] [MY-013170] [Server] c:\Temp\mysql\bin\mysqld.exe (mysqld 8.0.13) initializing of server has completed
```
- Запоминаем пароль слейва
- Запускаем два окна cmd, в каждом будет запущен свой сервер. В каждом делаем cd c:\temp\mysql\bin
- Запускаем первый сервер в одном окне

```
mysqld --defaults-file=C:\Temp\mysql\etc\my1.ini --standalone --console
```

- Запускаем второй сервер во втором окне
```
mysqld --defaults-file=C:\Temp\mysql\etc\my2.ini --standalone --console
```

Они должны работать, то есть после запуска не вываливаться в шелл.


# Проверяем слушают ли наши сервера порты, среди прочего 
netstat -nap tcp | findstr /R ":33.0"
```
  TCP    0.0.0.0:3310           0.0.0.0:0              LISTENING
  TCP    0.0.0.0:3320           0.0.0.0:0              LISTENING
```  
  
Наши оба сервера слушают сеть.  
  
  
# После первого подключения
- запускаем шелл mysql для мастера (предварительно запустив cmd и перейдя в c:\temp\mysql\bin): ```mysql --host=localhost --user=root --port=3310 -p```
- вводим пароль из консоли, который получили при иницициализации
- нам нужно его менять, иначе сервер нас дальше не пустит. Вводим: ```ALTER USER 'root'@'localhost' IDENTIFIED BY 'pass';```
- пароль поменян на pass
- тоже самое делаем на слейве, только порт при подключении задаём 3320

Всё, сервера настроены для домашки. Дальше надо синхронизировать.

На мастер заливаем базу employees (из второй ДЗ), делаем USE employees;
Лочим базу: ```FLUSH TABLES WITH READ LOCK;```
Делаем ```SHOW MASTER STATUS;```
Запоминаем что за лог он вывел и positon

Выходим (\q), делаем дамп базы: ```mysqldump.exe -u root -P 3310 -p employees > repl.sql```

На слейве создаем базу employees:
```SQL
create database employees
```

Закидываем дамп на слейв: 
```
mysql -u root -P 3320 -p employees < repl.sql```
Заходим вновь на мастер, разлочиваем базу: ```UNLOCK TABLES;```

Выходим. Заходим на слейв (порт 3320).
Создаём пользователя slave_user:
```SQL
create user 'slave_user';
alter user 'slave_user' identified by 'password';
grant replication SLAVE on *.* to 'slave_user'@'%' ;
```

Вводим шаманскую строку:
```SQL
CHANGE MASTER TO GET_MASTER_PUBLIC_KEY=1;
```

Проиписываем кто хозяин, так же вставляем имя лога и position из ```SHOW MASTER STATUS``` выше:
```SQL
change master to master_host='localhost', master_port=3310, master_user='slave_user', master_password='password', master_log_file='binlog.000003', master_log_pos=155;
```
- запускаем слейв: 
```SQL
start slave;
```


Всё.

Дальше смотрим статус слейва и пробуем на мастере менять строки.

На мастере:
```SQL
select * from employees limit 1;
update employees set first_name='Name1', last_name='Family1' where emp_no=10001;
select * from employees limit 1;
```

На слейве должны быть те же изменения:
```SQL
select * from employees limit 1;
```
