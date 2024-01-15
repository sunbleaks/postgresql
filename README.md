<b>установил ВМ с Ubuntu 22.04</b>  
![1](https://github.com/sunbleaks/postgresql/assets/144436024/f71d6e7d-5b31-4975-863d-2efe53100255)  
  
<b>установил PostgreSQL 15 и создал таблицу с данными</b>  
![Снимок экрана от 2024-01-14 18-10-48](https://github.com/sunbleaks/postgresql/assets/144436024/5a8157b1-0f66-4ab3-97bf-4ba00cbe80bc)  
  
<b>создал vhd на 10 GB и подключил к VM</b>  
![Снимок экрана от 2024-01-14 18-59-25](https://github.com/sunbleaks/postgresql/assets/144436024/d84d00d9-0003-410c-9cc1-22d1126e53c2)  
  
<b>разметка, выбор файловой системы и монтирование по инструкции  
https://www.digitalocean.com/community/tutorials/how-to-partition-and-format-storage-devices-in-linux#step-5-mount-the-new-filesystem</b>
![Снимок экрана от 2024-01-14 19-03-37](https://github.com/sunbleaks/postgresql/assets/144436024/89f29879-4409-4611-af65-10f56897982e)  
  
<b>после перезагрузки системы, диск остается примонтированным</b>  
![Снимок экрана от 2024-01-14 19-24-52](https://github.com/sunbleaks/postgresql/assets/144436024/5f754111-0a8d-4dd8-a9de-3ad830a3961d)  
  
<b>выполнил команду mv /var/lib/postgresql/15 /mnt/data  
и попытался стартануть кластер, получил ошибку</b>    
``` text
user@postgres1:~$ sudo -u postgres pg_ctlcluster 15 main start  
Error: /var/lib/postgresql/15/main is not accessible or does not exist</b>    
```  
  
<b>для исправления ошибки, необходимо в файле /etc/postgresql/15/main/postgresql.conf  
указать новый каталог, где будут храниться данные</b>      
``` text
#data_directory = '/var/lib/postgresql/15/main'         # use data in another directory
data_directory = '/mnt/data/15/main'
```
![Снимок экрана от 2024-01-15 09-27-34](https://github.com/sunbleaks/postgresql/assets/144436024/984b3cd1-43f3-41b5-a107-6f0563ed7291)  
  
<b>в psql проверил наличие созднанной таблицы с данными вначале</b>      
