развернуть виртуальную машину любым удобным способом  
поставить на неё PostgreSQL 15 любым способом 
``` text
user@postgres2:~$ sudo -u postgres pg_createcluster 15 main
Creating new PostgreSQL cluster 15/main ...
/usr/lib/postgresql/15/bin/initdb -D /var/lib/postgresql/15/main --auth-local peer --auth-host scram-sha-256 --no-instructions
The files belonging to this database system will be owned by user "postgres".
This user must also own the server process.

The database cluster will be initialized with locale "C.UTF-8".
The default database encoding has accordingly been set to "UTF8".
The default text search configuration will be set to "english".

Data page checksums are disabled.

fixing permissions on existing directory /var/lib/postgresql/15/main ... ok
creating subdirectories ... ok
selecting dynamic shared memory implementation ... posix
selecting default max_connections ... 100
selecting default shared_buffers ... 128MB
selecting default time zone ... Etc/UTC
creating configuration files ... ok
running bootstrap script ... ok
performing post-bootstrap initialization ... ok
syncing data to disk ... ok
Warning: systemd does not know about the new cluster yet. Operations like "service postgresql start" will not handle it. To fix, run:
  sudo systemctl daemon-reload
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 down   postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log


user@postgres2:~$ sudo pg_ctlcluster 15 main start
user@postgres2:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```
настроить кластер PostgreSQL 15 на максимальную производительность не обращая внимание на возможные   
проблемы с надежностью в случае аварийной перезагрузки виртуальной машины
``` text
Для начала запустим тест с дефолтными настройками

user@postgres2:~$ sudo -u postgres pgbench -i benchmark;
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.04 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.37 s (drop tables 0.00 s, create tables 0.04 s, client-side generate 0.17 s, vacuum 0.06 s, primary keys 0.09 s).



user@postgres2:~$ sudo -u postgres pgbench -c 50 -j 2 -P 10 -T 60 benchmark
pgbench (15.5 (Ubuntu 15.5-1.pgdg22.04+1))
starting vacuum...end.
progress: 10.0 s, 133.7 tps, lat 359.350 ms stddev 345.143, 0 failed
progress: 20.0 s, 137.4 tps, lat 362.557 ms stddev 428.098, 0 failed
progress: 30.0 s, 137.9 tps, lat 365.456 ms stddev 414.613, 0 failed
progress: 40.0 s, 135.4 tps, lat 364.453 ms stddev 429.457, 0 failed
progress: 50.0 s, 136.1 tps, lat 374.392 ms stddev 382.608, 0 failed
progress: 60.0 s, 135.0 tps, lat 367.478 ms stddev 325.918, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 8205
number of failed transactions: 0 (0.000%)
latency average = 366.803 ms
latency stddev = 389.977 ms
initial connection time = 38.332 ms
tps = 135.788205 (without initial connection time)
```

нагрузить кластер через утилиту через утилиту pgbench
``` text
вот что предложил PGTune
# DB Version: 15
# OS Type: linux
# DB Type: web
# Total Memory (RAM): 4 GB
# CPUs num: 2
# Connections num: 200
# Data Storage: ssd

max_connections = 200
shared_buffers = 1GB
effective_cache_size = 3GB
maintenance_work_mem = 256MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 100
random_page_cost = 1.1
effective_io_concurrency = 200
work_mem = 2621kB
huge_pages = off
min_wal_size = 1GB
max_wal_size = 4GB

Результат по tps оказался почти таким же

user@postgres2:~$ sudo -u postgres pgbench -c 50 -j 2 -P 10 -T 60 benchmark
pgbench (15.5 (Ubuntu 15.5-1.pgdg22.04+1))
starting vacuum...end.
progress: 10.0 s, 125.4 tps, lat 373.911 ms stddev 378.597, 0 failed
progress: 20.0 s, 134.1 tps, lat 382.873 ms stddev 416.664, 0 failed
progress: 30.0 s, 134.4 tps, lat 369.294 ms stddev 435.749, 0 failed
progress: 40.0 s, 139.4 tps, lat 356.195 ms stddev 333.847, 0 failed
progress: 50.0 s, 137.6 tps, lat 368.283 ms stddev 368.109, 0 failed
progress: 60.0 s, 138.2 tps, lat 355.055 ms stddev 335.846, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 8141
number of failed transactions: 0 (0.000%)
latency average = 369.561 ms
latency stddev = 384.416 ms
initial connection time = 42.090 ms
tps = 134.794235 (without initial connection time)


убираем предзапись в журнал (WAL)
alter system set synchronous_commit = 'off';
select pg_reload_conf();


user@postgres2:~$ sudo -u postgres pgbench -c 50 -j 2 -P 10 -T 60 benchmark
pgbench (15.5 (Ubuntu 15.5-1.pgdg22.04+1))
starting vacuum...end.
progress: 10.0 s, 3636.6 tps, lat 13.628 ms stddev 12.162, 0 failed
progress: 20.0 s, 3583.1 tps, lat 13.917 ms stddev 12.465, 0 failed
progress: 30.0 s, 3610.5 tps, lat 13.813 ms stddev 12.132, 0 failed
progress: 40.0 s, 3623.6 tps, lat 13.757 ms stddev 12.522, 0 failed
progress: 50.0 s, 3576.1 tps, lat 13.936 ms stddev 12.305, 0 failed
progress: 60.0 s, 3543.3 tps, lat 14.066 ms stddev 13.005, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 215789
number of failed transactions: 0 (0.000%)
latency average = 13.861 ms
latency stddev = 12.454 ms
initial connection time = 45.941 ms
tps = 3594.154321 (without initial connection time)
```

написать какого значения tps удалось достичь, показать какие параметры в какие значения устанавливали и почему
``` text
Отключение предзаписи в журнал (асинхронный режим)
помогло увеличить выполнение операций в ~ 26 раз. 
Все показатели намного лучше.
```