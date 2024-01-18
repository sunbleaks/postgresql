Настройте выполнение контрольной точки раз в 30 секунд.
``` text
alter system set checkpoint_timeout = '30s';

user@postgres1:~$ sudo pg_ctlcluster 15 main restart
[sudo] password for user: 
user@postgres1:~$
```
10 минут c помощью утилиты pgbench подавайте нагрузку.
``` text
Создаем БД для тестирования и инициализируем таблицами
create database benchmark;
user@postgres1:~$ sudo -u postgres pgbench -i benchmark

Запоминаем текущую позицию lsn
select pg_current_wal_lsn();     -- 0/21718A20

Запускаем тест
user@postgres1:~$ sudo -u postgres pgbench -c8 -P 6 -T 600 -U postgres benchmark
pgbench (15.5 (Ubuntu 15.5-1.pgdg22.04+1))
starting vacuum...end.
progress: 6.0 s, 136.7 tps, lat 57.983 ms stddev 15.643, 0 failed
progress: 12.0 s, 135.3 tps, lat 59.214 ms stddev 16.513, 0 failed
........
........
progress: 594.0 s, 151.2 tps, lat 52.943 ms stddev 11.395, 0 failed
progress: 600.0 s, 154.2 tps, lat 51.847 ms stddev 12.693, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 89200
number of failed transactions: 0 (0.000%)
latency average = 53.812 ms
latency stddev = 13.163 ms
initial connection time = 13.959 ms
tps = 148.655089 (without initial connection time)
```  
Измерьте, какой объем журнальных файлов был сгенерирован за это время. Оцените, какой объем приходится в среднем на одну контрольную точку.
``` text
select pg_current_wal_lsn();    -- 0/345A81C8
select '0/345A81C8'::pg_lsn - '0/21718A20'::pg_lsn as bytes;

Объем журнальных файлов:     
317 257 640 bytes (302 MB)

Примерный объем на одну контрольную точку
317 257 640 / (600/30) = 15 862 882 bytes (15,13 MB)   
```  
Проверьте данные статистики: все ли контрольные точки выполнялись точно по расписанию. Почему так произошло?
``` text
user@postgres1:~$ cat /var/log/postgresql/postgresql-15-main.log | grep 'checkpoint starting'

2024-01-17 17:28:20.914 UTC [3898] LOG:  checkpoint starting: time
2024-01-17 17:28:50.450 UTC [3898] LOG:  checkpoint starting: time
2024-01-17 17:29:20.936 UTC [3898] LOG:  checkpoint starting: time
2024-01-17 17:29:50.605 UTC [3898] LOG:  checkpoint starting: time
2024-01-17 17:30:20.767 UTC [3898] LOG:  checkpoint starting: time
2024-01-17 17:30:50.874 UTC [3898] LOG:  checkpoint starting: time
2024-01-17 17:31:20.572 UTC [3898] LOG:  checkpoint starting: time
2024-01-17 17:31:50.268 UTC [3898] LOG:  checkpoint starting: time
2024-01-17 17:32:20.760 UTC [3898] LOG:  checkpoint starting: time
2024-01-17 17:32:50.864 UTC [3898] LOG:  checkpoint starting: time
2024-01-17 17:33:20.972 UTC [3898] LOG:  checkpoint starting: time
2024-01-17 17:33:50.469 UTC [3898] LOG:  checkpoint starting: time
2024-01-17 17:34:20.368 UTC [3898] LOG:  checkpoint starting: time
2024-01-17 17:34:50.106 UTC [3898] LOG:  checkpoint starting: time
2024-01-17 17:35:20.679 UTC [3898] LOG:  checkpoint starting: time
2024-01-17 17:35:50.011 UTC [3898] LOG:  checkpoint starting: time
2024-01-17 17:36:20.006 UTC [3898] LOG:  checkpoint starting: time
2024-01-17 17:36:50.099 UTC [3898] LOG:  checkpoint starting: time
2024-01-17 17:37:20.866 UTC [3898] LOG:  checkpoint starting: time
2024-01-17 17:37:50.678 UTC [3898] LOG:  checkpoint starting: time

контрольная точка срабатывала, с интервалом чуть больше чем 30 секунд, 
чаще срабатывание происходит, если достигается обьем max_wal_size

select * from pg_stat_bgwriter;

Name                 |Value                        |
---------------------+-----------------------------+
checkpoints_timed    |960                          |
checkpoints_req      |19                           |
checkpoint_write_time|1514859.0                    |
checkpoint_sync_time |8945.0                       |
buffers_checkpoint   |25572                        |
buffers_clean        |113238                       |
maxwritten_clean     |71                           |
buffers_backend      |93049                        |
buffers_backend_fsync|0                            |
buffers_alloc        |211821                       |
stats_reset          |2024-01-14 22:33:52.707 +0200|

статистика не сбрасывалась, но видно общее отношение в 2%

checkpoints_timed — по расписанию (по достижению checkpoint_timeout)
checkpoints_req   — по требованию (в том числе по достижению max_wal_size)
```  

Сравните tps в синхронном/асинхронном режиме утилитой pgbench. Объясните полученный результат.
``` text
wal_writer_delay|
----------------+
200ms           |

synchronous_commit|
------------------+
off               |

user@postgres1:~$ sudo -u postgres pgbench -c8 -P 6 -T 600 -U postgres benchmark
pgbench (15.5 (Ubuntu 15.5-1.pgdg22.04+1))
starting vacuum...end.
progress: 6.0 s, 4070.3 tps, lat 1.961 ms stddev 1.130, 0 failed
progress: 12.0 s, 4026.8 tps, lat 1.986 ms stddev 1.159, 0 failed
progress: 18.0 s, 3853.2 tps, lat 2.076 ms stddev 1.096, 0 failed
progress: 24.0 s, 3718.5 tps, lat 2.151 ms stddev 1.685, 0 failed
progress: 30.0 s, 3628.7 tps, lat 2.204 ms stddev 2.896, 0 failed
progress: 36.0 s, 3849.0 tps, lat 2.078 ms stddev 1.157, 0 failed
progress: 42.0 s, 4002.1 tps, lat 1.998 ms stddev 1.392, 0 failed
.........
.........
progress: 582.0 s, 3476.7 tps, lat 2.301 ms stddev 1.268, 0 failed
progress: 588.0 s, 3495.0 tps, lat 2.289 ms stddev 1.112, 0 failed
progress: 594.0 s, 3692.1 tps, lat 2.166 ms stddev 1.313, 0 failed
progress: 600.0 s, 3669.7 tps, lat 2.180 ms stddev 2.120, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 2212476
number of failed transactions: 0 (0.000%)
latency average = 2.169 ms
latency stddev = 1.571 ms
initial connection time = 11.572 ms
tps = 3687.490391 (without initial connection time)

Tранзакций в секунду (tps) на порядок выше. 
Связанно с отсутствием ожидания ответа записи в журнал для востановления (WAL предзапись)  
```  

Создайте новый кластер с включенной контрольной суммой страниц. Создайте таблицу. Вставьте несколько значений. 
Выключите кластер. Измените пару байт в таблице. Включите кластер и сделайте выборку из таблицы. 
Что и почему произошло? как проигнорировать ошибку и продолжить работу?
``` text
postgres=# show data_checksums;
 data_checksums 
----------------
 on

postgres=# create table if not exists test(i int); 
CREATE TABLE
postgres=# insert into test(i) values (1); 
INSERT 0 1
postgres=# insert into test(i) values (2); 
INSERT 0 1

postgres=# SELECT 'test'::regclass::oid;
  oid  
-------
 16394
(1 row)

user@postgres2:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 down   postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log


postgres=# select * from test;
WARNING:  page verification failed, calculated checksum 44320 but expected 54492
ERROR:  invalid page in block 0 of relation base/5/16394


Произошла ошибка верификации страницы, т.е данные были изменены после сброса на диск. 
Контрольная сумма вычисляется и записывается на страницу, 
в моменте когда страница записывается из буферного кеша в страничный кеш операционной системы.


Поиск поврежденого объекта:
SELECT pg_database.oid, pg_class.relfilenode, pg_namespace.nspname as schema_name, pg_class.relname, pg_class.relkind 
  from pg_class
  JOIN pg_namespace on pg_class.relnamespace = pg_namespace.oid
  WHERE pg_class.relfilenode = 16394
 ORDER BY pg_class.relfilenode ASC;

 relfilenode | schema_name | relname | relkind 
-------------+-------------+---------+---------
       16394 | public      | test    | r
(1 row)


чтобы проигнорировать ошибку, можно включить:

1) ignore_checksum_failure = on

после этого выдается результат с предупреждением (результат запроса может быть неправильным)
postgres=# select * from test;
WARNING:  page verification failed, calculated checksum 44320 but expected 54492
 i 
---
 1
 2
(2 rows)

2) zero_damaged_pages = on
запрос ничего не выдал, так как поврежденная страница была очищена

```  
