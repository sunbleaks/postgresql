1. создайте новый кластер PostgresSQL 14  
``` text
user@postgres1:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory    Log file
15  main    5432 online postgres /mnt/data/15/main /var/log/postgresql/postgresql-15-main.log
```  
2. зайдите в созданный кластер под пользователем postgres
``` text
user@postgres1:~$ sudo -u postgres psql
[sudo] password for user: 
psql (15.5 (Ubuntu 15.5-1.pgdg22.04+1))
Type "help" for help.

postgres=# 
```  
3. создайте новую базу данных testdb
``` text
postgres=# create database testdb;
CREATE DATABASE
postgres=#
```  
4. зайдите в созданную базу данных под пользователем postgres
``` text
postgres=# \c testdb
You are now connected to database "testdb" as user "postgres".
testdb=# select current_user;
 current_user 
--------------
 postgres
```  
5. создайте новую схему testnm
``` text
testdb=# create schema testnm;
CREATE SCHEMA
testdb=#
```  
6. создайте новую таблицу t1 с одной колонкой c1 типа integer
``` text
testdb=# create table t1(c1 int);
CREATE TABLE
testdb=#
```  
7. вставьте строку со значением c1=1
``` text
testdb=# insert into t1 (c1) values (1);
INSERT 0 1
testdb=#
```  
8. создайте новую роль readonly
``` text
testdb=# create role readonly;
CREATE ROLE
```  
9. дайте новой роли право на подключение к базе данных testdb
``` text
testdb=# GRANT CONNECT ON DATABASE testdb TO readonly;
GRANT
```  
10. дайте новой роли право на использование схемы testnm
``` text
testdb=# GRANT USAGE ON SCHEMA testnm TO readonly;
GRANT
```  
11. дайте новой роли право на select для всех таблиц схемы testnm
``` text
testdb=# GRANT SELECT ON ALL TABLES in SCHEMA testnm TO readonly;
GRANT
```  
12. создайте пользователя testread с паролем test123
``` text
testdb=# CREATE USER testread with PASSWORD 'test123';
CREATE ROLE
``` 
13. дайте роль readonly пользователю testread
``` text
testdb=# GRANT readonly TO testread;
CREATE ROLE
``` 
14. зайдите под пользователем testread в базу данных testdb
``` text
user@postgres1:~$ psql -U testread -h localhost -d testdb
Password for user testread: 
psql (15.5 (Ubuntu 15.5-1.pgdg22.04+1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, compression: off)
Type "help" for help.

testdb=>
``` 
15. сделайте select * from t1;
``` text
testdb=> select * from t1;
``` 
16. получилось?
``` text
нет
``` 
17. напишите что именно произошло
``` text
ERROR:  permission denied for table t1
```
18. у вас есть идеи почему? ведь права то дали?
``` text
для роли readonly выданы гранты для работы с схемой testnm,
а t1 была создана под пользователем postgres в схеме public
``` 
19. посмотрите на список таблиц
``` text
testdb=> \dt
        List of relations
 Schema | Name | Type  |  Owner   
--------+------+-------+----------
 public | t1   | table | postgres
``` 
20. Schema public для t1  
    
21. а почему так получилось с таблицей
``` text
схема public создается по умолчанию при создании БД. 
если явно схему не указывать при создании таблицы, то таблица создается в public 
```
22. вернитесь в базу данных testdb под пользователем postgres
``` text
postgres=# \c testdb
You are now connected to database "testdb" as user "postgres".
testdb=# 
``` 
23.удалите таблицу t1
``` text
testdb=# drop table t1;
DROP TABLE
```
24.создайте ее заново но уже с явным указанием имени схемы testnm
``` text
testdb=# create table testnm.t1(c1 int);
CREATE TABLE
```
25. вставьте строку со значением c1=1
``` text
testdb=# insert into testnm.t1 (c1) values (1);
INSERT 0 1
```
26. зайдите под пользователем testread в базу данных testdb
``` text
user@postgres1:~$ psql -U testread -h localhost -d testdb
Password for user testread: 
psql (15.5 (Ubuntu 15.5-1.pgdg22.04+1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, compression: off)
Type "help" for help.

testdb=> 
```
27.сделайте select * from testnm.t1;
``` text
testdb=> select * from testnm.t1;
```
28.получилось?
``` text
нет
ERROR:  permission denied for table t1
```
29.причина
``` text
необходимо перевыдать гранты для новых таблиц
GRANT SELECT ON ALL TABLES in SCHEMA testnm TO readonly;
```
30. как сделать так чтобы такое больше не повторялось?
``` text
использовать 
ALTER default privileges in SCHEMA testnm grant SELECT on TABLES to readonly; 
```
37. теперь попробуйте выполнить команду create table t2(c1 integer); insert into t2 values (2);
``` text
не получилось создать таблицу, как указанно в инструкции. 
testdb=> create table t2(c1 integer);
ERROR:  permission denied for schema public

причина в обновленной 15-й версии https://www.postgresql.org/docs/release/15.0/ 
для обычных пользователей нет гранта на создание (CREATE) объектов в public схеме 

разрешаем создавать командой
testdb=# GRANT CREATE ON SCHEMA public TO readonly;
GRANT


запрещаем
testdb=# REVOKE CREATE on SCHEMA public FROM readonly;
REVOKE

```


