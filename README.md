1. <b>Создаем ВМ/докер c ПГ.</b>

    В VirtualBox установлены Ubuntu 22.04.3 LTS и кластер postgresql 15  
``` text
user@postgres2:~$ cat /etc/os-release
PRETTY_NAME="Ubuntu 22.04.3 LTS"
NAME="Ubuntu"
VERSION_ID="22.04"
VERSION="22.04.3 LTS (Jammy Jellyfish)"
VERSION_CODENAME=jammy
ID=ubuntu
ID_LIKE=debian
HOME_URL="https://www.ubuntu.com/"
SUPPORT_URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
UBUNTU_CODENAME=jammy


user@postgres2:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
user@postgres2:~$ 
```  

2. <b>Создаем БД, схему и в ней таблицу.</b> (БД - otus, схема - scm)  
![Снимок экрана от 2024-02-06 13-20-05](https://github.com/sunbleaks/postgresql/assets/144436024/1c67ae13-5292-4a4a-a283-f688fe5615fe)

3. <b>Заполняем таблицы автосгенерированными 100 записями.</b> (таблицы - students и films)  
![Снимок экрана от 2024-02-06 13-20-38](https://github.com/sunbleaks/postgresql/assets/144436024/32cefc91-9a72-4651-a4f9-30850c5ec29c)

4. <b>Под линукс пользователем Postgres создадим каталог для бэкапов</b> (/tmp/backup)  
![Снимок экрана от 2024-02-06 14-16-00](https://github.com/sunbleaks/postgresql/assets/144436024/4d930bcd-1c3f-4c2d-a359-2ad0da9e251b)

5. <b>Сделаем логический бэкап используя утилиту COPY</b>
``` text
user@postgres2:~$ sudo -u postgres psql
psql (15.5 (Ubuntu 15.5-1.pgdg22.04+1))
Type "help" for help.

postgres=# \c otus
You are now connected to database "otus" as user "postgres".
otus=# \copy scm.students to '/tmp/backup/backup_scm.students.sql';
COPY 100
otus=# \copy scm.films to '/tmp/backup/backup_scm.films.sql';
COPY 100
otus=# 
```  
Проверяем содержимое каталога:   
![Снимок экрана от 2024-02-06 14-32-48](https://github.com/sunbleaks/postgresql/assets/144436024/43555038-58f4-4af2-aebd-5039482469d9)

6. <b>Восстановим в 2 таблицы данные из бэкапа.</b>
``` text
Удалим записи
postgres=# \c otus;
You are now connected to database "otus" as user "postgres".
otus=# delete from scm.students;
DELETE 100
otus=# delete from scm.films;
DELETE 100
```  
Восстанавливаем:  
![Снимок экрана от 2024-02-06 14-40-49](https://github.com/sunbleaks/postgresql/assets/144436024/2a3b9204-0cb3-4270-a4ae-15282d49377b)


7. <b>Используя утилиту pg_dump создаё бэкап в кастомном сжатом формате двух таблиц</b>
``` text
Попытка выполнить pg_dump и ошибка доступа.
Необходимо выполнять операцию по юзером, который имеет права для работы с директорие /tmp/backup
  
user@postgres2:~$ sudo -u postgres pg_dump -d otus --create -Fc | gzip > /tmp/backup/backup_otus.gz
-bash: /tmp/backup/backup_otus.gz: Permission denied
user@postgres2:~$ 

Логинемся: 
user@postgres2:~$ sudo su postgres
postgres@postgres2:/home/user$ 

Нет необходимости выполнять команду с sudo
postgres@postgres2:/home/user$ pg_dump -d otus --create -Fc | gzip > /tmp/backup/backup_otus.gz
postgres@postgres2:/home/user$ 
``` 
Проверяем:  
![Снимок экрана от 2024-02-06 14-55-42](https://github.com/sunbleaks/postgresql/assets/144436024/259ab5f4-4575-46bb-922a-20a33abde491)

8. <b>Используя утилиту pg_restore восстанавливаем в новую БД только вторую таблицу!</b>  
Восстанавливаю только таблицу scm.films    
Для начала распакуем архив в файл 
![Снимок экрана от 2024-02-06 16-00-33](https://github.com/sunbleaks/postgresql/assets/144436024/fd78eee7-80ee-487d-a7e8-adfc8f9c71d5)  

Без создания БД и схемы не получилось достать из бэкаба конкретную таблицу: 
``` text
Флаг -С (--reate) не помог

user@postgres2:/tmp/backup$ sudo -u postgres pg_restore -v -C -d otus -n scm -t films /tmp/backup/backup_otus
pg_restore: connecting to database for restore
pg_restore: error: connection to server on socket "/var/run/postgresql/.s.PGSQL.5432" failed: FATAL:  database "otus" does not exist
``` 
![Снимок экрана от 2024-02-06 16-21-09](https://github.com/sunbleaks/postgresql/assets/144436024/a2316fc0-487a-43e3-9188-e3b25f727cac)  

``` text
Когда созданны БД и схема, тогда успешно удается восстановить таблицу из бэкапа  
``` 
![Снимок экрана от 2024-02-06 16-24-28](https://github.com/sunbleaks/postgresql/assets/144436024/bb3f8150-9e43-4f86-aa46-2f23fa8dd953)





