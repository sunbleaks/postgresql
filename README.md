выключаем auto commit:  
``` sql
postgres=# \set AUTOCOMMIT OFF
postgres=# \echo :AUTOCOMMIT
OFF
```
создаем и наполняем таблицу:  
``` sql
create table persons(id serial, first_name text, second_name text);   
insert into persons(first_name, second_name) values('ivan', 'ivanov');  
insert into persons(first_name, second_name) values('petr', 'petrov');  
commit;
```
``` sql
postgres=# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)
```

проверяем текущий уровень изоляции:
``` sql
postgres=# show transaction isolation level;
 transaction_isolation 
-----------------------
 read committed
(1 row)
```

начинаем новую транзакцию в обоих сессиях с дефолтным (не меняя) уровнем изоляции  
в первой сессии добавляем новую запись 
``` sql
insert into persons(first_name, second_name) values('sergey', 'sergeev');
```
во второй сессии
``` sql
select * from persons;
```

результат:
``` sql
postgres=# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)
```
<b>новая запись не отображается, т.к. установлен дефолтный уровень изоляции read committed и отключен автокомит  
(вставка первой транзакцией не зафиксирована)</b>    
фиксируем первую транзакцию 
``` sql
commit;
```
еще раз проверяем во второй транзакции
``` sql
select * from persons;
```

результат:
``` sql
postgres=*# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  4 | sergey     | sergeev
(3 rows)
```
<b>новая запись отображается по причине того, что в первой транзакции был commit - данные зафиксированы</b>    
фиксируем вторую транзакцию
``` sql
commit;
```
устанавливаем в сессиях уровень изоляции транзакций repeatable read 
``` sql
postgres=# set transaction isolation level repeatable read;
SET
postgres=*# show transaction isolation level;
 transaction_isolation 
-----------------------
 repeatable read
(1 row)
```
в первой сессии добавляем новую запись 
``` sql
postgres=*# insert into persons(first_name, second_name) values('sveta', 'svetova');
INSERT 0 1
```

во второй сессии выполняем
``` sql
postgres=*# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  4 | sergey     | sergeev
(3 rows)
```
<b>вставленные данные не отображаются, т.к. в первой сессии не было фиксации</b>  
фиксируем вставку в первой сессии
``` sql
commit;
```

во второй сессии повторно выполняем запрос
``` sql
postgres=*# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  4 | sergey     | sergeev
(3 rows)
```
<b>вставленные данные не отображаются, даже при фиксации транзакции в первой сессии.  
вторая транзакция видит данные (снимок данных) на момент своего начала  
и не ввидит остальные завершенные изменения, сделанные в других сессиях в этот момент (фантомное чтение).  
неповторяющее чтение так же не работает - сколько бы не читали данные - реезульт будет тот же.
</b>  

<b>фиксируем транзакцию во второй сессии и видим изменения.</b>
``` sql
postgres=*# commit;
COMMIT
postgres=# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  4 | sergey     | sergeev
  5 | sveta      | svetova
(4 rows)
```

