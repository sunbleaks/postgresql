Настройте сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках,   
удерживаемых более 200 миллисекунд. Воспроизведите ситуацию, при которой в журнале   
появятся такие сообщения.  
``` text
Переводим log_lock_waits в on
Выставлем deadlock_timeout время ожидания блокировки пережд фиксацией в журнал  

locks=# show log_lock_waits;
 log_lock_waits 
----------------
 on

locks=# show deadlock_timeout;
 deadlock_timeout 
--------------
 200ms 
 
В первом и втором сеансах, открываем транзакцию и пытаемся проапдейтить 
одну и ту же запись

UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;

смотрим лог в /var/log/postgresql/postgresql-15-main.log 

2024-01-20 07:50:17.497 UTC [9394] postgres@locks LOG:  process 9394 still waiting for ShareLock on transaction 3776374 after 200.334 ms
2024-01-20 07:50:17.497 UTC [9394] postgres@locks DETAIL:  Process holding the lock: 9305. Wait queue: 9394.
2024-01-20 07:50:17.497 UTC [9394] postgres@locks CONTEXT:  while updating tuple (0,4) in relation "accounts"
2024-01-20 07:50:17.497 UTC [9394] postgres@locks STATEMENT:  UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
```

Смоделируйте ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах. 
Изучите возникшие блокировки в представлении pg_locks и убедитесь, что все они понятны. 
Пришлите список блокировок и объясните, что значит каждая.
``` text

1-й сеанс

locks=# BEGIN;
BEGIN
locks=*# SELECT txid_current(), pg_backend_pid();
 txid_current | pg_backend_pid 
--------------+----------------
      3776386 |           9305
(1 row)

locks=*# UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
UPDATE 1
locks=*# SELECT * FROM locks_v WHERE pid = 9305;
 pid  |   locktype    |  lockid  |       mode       | granted 
------+---------------+----------+------------------+---------
 9305 | relation      | accounts | RowExclusiveLock | t
 9305 | transactionid | 3776386  | ExclusiveLock    | t
(2 rows)

Транзакция всегда удерживает исключительную (ExclusiveLock) блокировку собственного номера,
а так же видим RowExclusiveLock и удерживается блокировка таблицы




2-й сеанс

locks=# BEGIN;
BEGIN
locks=*# SELECT txid_current(), pg_backend_pid();
 txid_current | pg_backend_pid 
--------------+----------------
      3776387 |           9394
(1 row)

locks=*# UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;

подвисли..

locks=# SELECT * FROM locks_v where pid = 9394; 
 pid  |   locktype    |   lockid    |       mode       | granted 
------+---------------+-------------+------------------+---------
 9394 | relation      | accounts    | RowExclusiveLock | t
 9394 | tuple         | accounts:11 | ExclusiveLock    | t
 9394 | transactionid | 3776387     | ExclusiveLock    | t
 9394 | transactionid | 3776386     | ShareLock        | f
(4 rows)

как и в первом сеансе: 
Транзакция всегда удерживает исключительную (ExclusiveLock) блокировку собственного номера,
а так же видим RowExclusiveLock и удерживается блокировка таблицы

+ 2 новые блокировки
9394 | tuple         | accounts:11 | ExclusiveLock    | t    - блокировка версии строки
9394 | transactionid | 3776386     | ShareLock        | f    - в ожидании заершения 3776386 




3-й сеанс

locks=# BEGIN;
BEGIN
locks=*# SELECT txid_current(), pg_backend_pid();
 txid_current | pg_backend_pid 
--------------+----------------
      3776388 |           9388
(1 row)

locks=*# UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;

подвисли..

locks=*# SELECT * FROM locks_v where pid = 9388;
 pid  |   locktype    |   lockid    |       mode       | granted 
------+---------------+-------------+------------------+---------
 9388 | relation      | accounts    | RowExclusiveLock | t
 9388 | transactionid | 3776388     | ExclusiveLock    | t
 9388 | tuple         | accounts:11 | ExclusiveLock    | f    - в ожидании завершения предыдущей блокировки строки
(3 rows)


Образовалась очередь блокировок.


2024-01-20 09:16:33.591 UTC [9394] postgres@locks LOG:  process 9394 still waiting for ShareLock on transaction 3776386 after 200.114 ms
2024-01-20 09:16:33.591 UTC [9394] postgres@locks DETAIL:  Process holding the lock: 9305. Wait queue: 9394.
2024-01-20 09:16:33.591 UTC [9394] postgres@locks CONTEXT:  while updating tuple (0,11) in relation "accounts"
2024-01-20 09:16:33.591 UTC [9394] postgres@locks STATEMENT:  UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
2024-01-20 09:28:06.225 UTC [9388] postgres@locks LOG:  process 9388 still waiting for ExclusiveLock on tuple (0,11) of relation 18553 of database 18552 after 200.284 ms
2024-01-20 09:28:06.225 UTC [9388] postgres@locks DETAIL:  Process holding the lock: 9394. Wait queue: 9388.
2024-01-20 09:28:06.225 UTC [9388] postgres@locks STATEMENT:  UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;


9394 ожидает завершения 9305
9388 ожидает завершения 9394

locks=*# SELECT locktype, relation::REGCLASS, mode, granted, pid, pg_blocking_pids(pid) AS wait_for
FROM pg_locks WHERE relation = 'accounts'::regclass order by pid;
 locktype | relation |       mode       | granted | pid  | wait_for 
----------+----------+------------------+---------+------+----------
 relation | accounts | RowExclusiveLock | t       | 9305 | {}
 relation | accounts | RowExclusiveLock | t       | 9388 | {9394}
 tuple    | accounts | ExclusiveLock    | f       | 9388 | {9394}
 relation | accounts | RowExclusiveLock | t       | 9394 | {9305}
 tuple    | accounts | ExclusiveLock    | t       | 9394 | {9305}
(5 rows)
```  

Воспроизведите взаимоблокировку трех транзакций. Можно ли разобраться в ситуации постфактум, изучая журнал сообщений?
``` text

locks=# BEGIN;
BEGIN
locks=*# SELECT txid_current(), pg_backend_pid();
 txid_current | pg_backend_pid 
--------------+----------------
      3776392 |           9305
(1 row)

locks=# BEGIN;                                                       
UPDATE accounts SET amount = amount - 200.00 WHERE acc_no = 1;
BEGIN
UPDATE 1
locks=*# BEGIN;
UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 2;
WARNING:  there is already a transaction in progress
BEGIN




locks=# BEGIN;
BEGIN
locks=*# SELECT txid_current(), pg_backend_pid();
 txid_current | pg_backend_pid 
--------------+----------------
      3776393 |           9394
(1 row)
  
locks=# BEGIN;                                                       
UPDATE accounts SET amount = amount - 20.00 WHERE acc_no = 2;
BEGIN
UPDATE 1
locks=*# BEGIN;
UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 3;
WARNING:  there is already a transaction in progress
BEGIN
UPDATE 1
locks=*# 




locks=# BEGIN;
BEGIN
locks=*# SELECT txid_current(), pg_backend_pid();
 txid_current | pg_backend_pid 
--------------+----------------
      3776394 |           9388
(1 row)
locks=*# UPDATE accounts SET amount = amount - 20.00 WHERE acc_no = 3;
UPDATE 1
locks=*# UPDATE accounts SET amount = amount + 10.00 WHERE acc_no = 2;
ERROR:  deadlock detected
DETAIL:  Process 9388 waits for ExclusiveLock on tuple (0,39) of relation 18553 of database 18552; blocked by process 9305.
Process 9305 waits for ShareLock on transaction 3776398; blocked by process 9394.
Process 9394 waits for ShareLock on transaction 3776399; blocked by process 9388.
HINT:  See server log for query details.
locks=!# 


в журнале можно понять, кто кого заблокировал
так же фиксируется фраза deadlock detected

2024-01-20 10:49:12.612 UTC [9388] postgres@locks DETAIL:  Process holding the lock: 9305. Wait queue: .
2024-01-20 10:49:12.612 UTC [9388] postgres@locks STATEMENT:  UPDATE accounts SET amount = amount + 10.00 WHERE acc_no = 2;
2024-01-20 10:49:12.612 UTC [9388] postgres@locks ERROR:  deadlock detected
2024-01-20 10:49:12.612 UTC [9388] postgres@locks DETAIL:  Process 9388 waits for ExclusiveLock on tuple (0,39) of relation 18553 of database 18552; blocked by process 9305.
        Process 9305 waits for ShareLock on transaction 3776398; blocked by process 9394.
        Process 9394 waits for ShareLock on transaction 3776399; blocked by process 9388.
        Process 9388: UPDATE accounts SET amount = amount + 10.00 WHERE acc_no = 2;
        Process 9305: UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 2;
        Process 9394: UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 3;

```  

Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга?
``` text
locks=# BEGIN;
BEGIN
locks=*# SELECT txid_current(), pg_backend_pid();
 txid_current | pg_backend_pid 
--------------+----------------
      3776400 |           9388
(1 row)

locks=*# UPDATE accounts SET amount = amount + 10.00;
UPDATE 3
locks=*# 



locks=# BEGIN;
BEGIN
locks=*# SELECT txid_current(), pg_backend_pid();
 txid_current | pg_backend_pid 
--------------+----------------
      3776401 |           9394
(1 row)

locks=*# UPDATE accounts SET amount = amount + 10.00;

Подвисло..



2024-01-20 11:03:13.604 UTC [9388] postgres@locks LOG:  process 9388 still waiting for ShareLock on transaction 3776401 after 200.246 ms
2024-01-20 11:03:13.604 UTC [9388] postgres@locks DETAIL:  Process holding the lock: 9394. Wait queue: 9388.
2024-01-20 11:03:13.604 UTC [9388] postgres@locks CONTEXT:  while updating tuple (0,37) in relation "accounts"
2024-01-20 11:03:13.604 UTC [9388] postgres@locks STATEMENT:  UPDATE accounts SET amount = amount + 10.00;

locks=# SELECT locktype, relation::REGCLASS, mode, granted, pid, pg_blocking_pids(pid) AS wait_for
FROM pg_locks WHERE relation = 'accounts'::regclass order by pid;
 locktype | relation |       mode       | granted | pid  | wait_for 
----------+----------+------------------+---------+------+----------
 relation | accounts | RowExclusiveLock | t       | 9388 | {9394}
 tuple    | accounts | ExclusiveLock    | t       | 9388 | {9394}
 relation | accounts | RowExclusiveLock | t       | 9394 | {}
(3 rows)



Транзакции становятся в очередь.  
Друг друга не блокируют, транзакция 9394 блокирует 9388

``` 