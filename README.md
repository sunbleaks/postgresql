<b>создать ВМ с Ubuntu 20.04/22.04 или развернуть докер любым удобным способом</b>  
![1](https://github.com/sunbleaks/postgresql/assets/144436024/f71d6e7d-5b31-4975-863d-2efe53100255)


<b>поставить на нем Docker Engine</b>  
![2](https://github.com/sunbleaks/postgresql/assets/144436024/22d45bf3-a33f-4d7b-a38f-528e510dadcd)


<b>сделать каталог /var/lib/postgres</b>  
![3](https://github.com/sunbleaks/postgresql/assets/144436024/fd14d611-eebf-47ac-91eb-c69a1b32cca9)


<b>развернуть контейнер с PostgreSQL 15 смонтировав в него /var/lib/postgresql</b>  
``` text
sudo docker network create pg-net  
sudo docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:15
```


<b>развернуть контейнер с клиентом postgres  
подключится из контейнера с клиентом к контейнеру с сервером и сделать таблицу с парой строк</b>  
``` text
sudo docker network create pg-net  
sudo docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:15  
sudo docker run -it --rm --network pg-net --name pg-client postgres:15 psql -h pg-server -U postgres  
create database otus11111;  
\c otus11111  
CREATE TABLE test (i serial, amount int);  
INSERT INTO test(amount) VALUES (100);  
INSERT INTO test(amount) VALUES (500);  
```
![4](https://github.com/sunbleaks/postgresql/assets/144436024/3fcf6531-1e1f-4a3d-a6bf-03566c518869)


<b>подключится к контейнеру с сервером с ноутбука/компьютера извне инстансов GCP/ЯО/места установки докера
(пробовал подключиться из другого инстанса ВМ)
</b>
![5](https://github.com/sunbleaks/postgresql/assets/144436024/30b03613-a259-48ec-a2da-2345d3c9345c)


<b>удалить контейнер с сервером  
создать его заново  
подключится снова из контейнера с клиентом к контейнеру с сервером  
проверить, что данные остались на месте</b>  

проверил, данные после удаления остались на хост машине, и после запуска контейнера с опцией -v стали доступными из контейнера  
