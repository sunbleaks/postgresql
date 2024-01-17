Создать инстанс ВМ с 2 ядрами и 4 Гб ОЗУ и SSD 10GB  
``` text
user@postgres2:~$ cat /proc/cpuinfo | grep 'cpu cores'
cpu cores	: 2

user@postgres2:~$ cat /proc/meminfo | head -n 1
MemTotal:     4005976 kB

user@postgres2:~$ lsblk
NAME MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda    8:0    0   10G  0 disk
```
Установить PostgreSQL 15 с дефолтными настройками
``` text
user@postgres2:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```  
Создать БД для тестов: выполнить pgbench -i бд
``` text
user@postgres2:~$ sudo -u postgres pgbench -i benchmark
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.05 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.28 s (drop tables 0.00 s, create tables 0.03 s, client-side generate 0.09 s, vacuum 0.05 s, primary keys 0.11 s).
```  
Запустить pgbench -c8 -P 6 -T 60 -U postgres бд
``` text
user@postgres2:~$ sudo -u postgres pgbench -c8 -P 6 -T 60 benchmark
pgbench (15.5 (Ubuntu 15.5-1.pgdg22.04+1))
starting vacuum...end.
progress: 6.0 s, 129.2 tps, lat 61.096 ms stddev 36.098, 0 failed
progress: 12.0 s, 130.0 tps, lat 61.697 ms stddev 34.918, 0 failed
progress: 18.0 s, 136.0 tps, lat 58.792 ms stddev 32.902, 0 failed
progress: 24.0 s, 135.0 tps, lat 58.968 ms stddev 32.998, 0 failed
progress: 30.0 s, 138.0 tps, lat 58.049 ms stddev 32.361, 0 failed
progress: 36.0 s, 137.7 tps, lat 57.909 ms stddev 32.763, 0 failed
progress: 42.0 s, 138.8 tps, lat 57.779 ms stddev 30.474, 0 failed
progress: 48.0 s, 138.8 tps, lat 57.555 ms stddev 31.974, 0 failed
progress: 54.0 s, 135.5 tps, lat 58.904 ms stddev 36.417, 0 failed
progress: 60.0 s, 137.7 tps, lat 58.061 ms stddev 32.171, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 8148
number of failed transactions: 0 (0.000%)
latency average = 58.861 ms
latency stddev = 33.345 ms
initial connection time = 17.046 ms
tps = 135.676582 (without initial connection time)
```  

Применить параметры настройки PostgreSQL из прикрепленного к материалам занятия файла  
Протестировать заново  
Что изменилось и почему?  
``` text
user@postgres2:~$ sudo -u postgres pgbench -c8 -P 6 -T 60 benchmark
pgbench (15.5 (Ubuntu 15.5-1.pgdg22.04+1))
starting vacuum...end.
progress: 6.0 s, 135.5 tps, lat 58.350 ms stddev 31.442, 0 failed
progress: 12.0 s, 132.2 tps, lat 60.430 ms stddev 34.245, 0 failed
progress: 18.0 s, 132.3 tps, lat 60.560 ms stddev 30.938, 0 failed
progress: 24.0 s, 133.5 tps, lat 59.613 ms stddev 31.655, 0 failed
progress: 30.0 s, 139.2 tps, lat 57.586 ms stddev 33.486, 0 failed
progress: 36.0 s, 136.6 tps, lat 58.551 ms stddev 33.375, 0 failed
progress: 42.0 s, 135.7 tps, lat 58.826 ms stddev 30.956, 0 failed
progress: 48.0 s, 138.4 tps, lat 57.797 ms stddev 30.578, 0 failed
progress: 54.0 s, 138.2 tps, lat 57.697 ms stddev 33.510, 0 failed
progress: 60.0 s, 137.6 tps, lat 58.050 ms stddev 33.051, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 8162
number of failed transactions: 0 (0.000%)
latency average = 58.748 ms
latency stddev = 32.378 ms
initial connection time = 23.837 ms
tps = 135.956785 (without initial connection time)

не увидел разницы (
```  

Создать таблицу с текстовым полем и заполнить случайными или сгенерированными данным в размере 1млн строк
Посмотреть размер файла с таблицей
``` text
postgres=# \dt
          List of relations
 Schema |  Name   | Type  |  Owner   
--------+---------+-------+----------
 public | student | table | postgres
(1 row)

postgres=# select count(*) from student;
  count  
---------
 1000000
(1 row)

    postgres=# SELECT pg_size_pretty(pg_TABLE_size('student'));
 pg_size_pretty 
----------------
 135 MB
(1 row)
```  
5 раз обновить все строчки и добавить к каждой строчке любой символ
Посмотреть количество мертвых строчек в таблице и когда последний раз приходил автовакуум
Подождать некоторое время, проверяя, пришел ли автовакуум
``` text
 relname | n_live_tup | n_dead_tup | ratio% |       last_autovacuum        
---------+------------+------------+--------+------------------------------
 student |    1000000 |    1000000 |     99 | 2024-01-17 09:32:24.64025+00
(1 row)

 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum        
---------+------------+------------+--------+-------------------------------
 student |    1000000 |          0 |      0 | 2024-01-17 09:33:24.438468+00
(1 row)
```  

5 раз обновить все строчки и добавить к каждой строчке любой символ
Посмотреть размер файла с таблицей
``` text
postgres=# SELECT pg_size_pretty(pg_TABLE_size('student'));
 pg_size_pretty 
----------------
 943 MB
(1 row)
```  
10 раз обновить все строчки и добавить к каждой строчке любой символ
Посмотреть размер файла с таблицей
Объясните полученный результат
``` text
postgres=# CREATE TABLE student(
  id serial,
  fio char(100)
) WITH (autovacuum_enabled = off);
CREATE TABLE

postgres=# CREATE INDEX idx_fio ON student(fio);
CREATE INDEX

postgres=# INSERT INTO student(fio) SELECT 'noname' FROM generate_series(1,1000000);
INSERT 0 1000000

postgres=# SELECT pg_size_pretty(pg_TABLE_size('student'));
 pg_size_pretty 
----------------
 135 MB
(1 row)

Обновляем записи в таблице student
и проверяем размер файла с таблицей

postgres=# update student set fio = 'name';
UPDATE 1000000
postgres=# update student set fio = 'name';
UPDATE 1000000
postgres=# update student set fio = 'name';
UPDATE 1000000
postgres=# update student set fio = 'name';
UPDATE 1000000
postgres=# update student set fio = 'name';
UPDATE 1000000
postgres=# SELECT pg_size_pretty(pg_TABLE_size('student'));
 pg_size_pretty 
----------------
 808 MB
(1 row)

postgres=# update student set fio = 'name1';
UPDATE 1000000
postgres=# SELECT pg_size_pretty(pg_TABLE_size('student'));
 pg_size_pretty 
----------------
 943 MB
(1 row)


Опреция update состоит из delete и insert, при этом удаленные записи (касается и просто опреации delete) физически не удаляются, 
а помечаются как неактивные. Каждая строка имеет скрытые заголовки, 
и для update записи: 
xmax – идентификатор транзакции, удалившей запись;
ctid – ссылка на следующую, более новую, версию той же строки.

чтобы файл с таблицей не "разбухал", необходимо применять оперции vacuum/autovacuum.
если применять просто vacuum вручную - то остаются пропуски, бывает полезно для последующих вставок
НО autovacuum - вообще не рекомендуется отключать!!   
```  

