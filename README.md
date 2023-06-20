# 31-PostgreSQL
Цель: научиться настраивать репликацию и создавать резервные копии в СУБД PostgreSQL;
1.    настроить hot_standby репликацию с использованием слотов
2.    настроить правильное резервное копирование
  Для сдачи работы присылаем ссылку на репозиторий, в котором должны обязательно быть
  -  Vagranfile (2 машины)
  -  плейбук Ansible
  -  конфигурационные файлы postgresql.conf, pg_hba.conf и recovery.conf,
  -  конфиг barman, либо скрипт резервного копирования.

## Выполнение: 
Настраиваем сервера с помощью `ansible` и проверяем соотвествие условиям задания.
Проверяем наличие созданной на `node1` базы данных на `node2`:
```bash
[vagrant@node1 ~]$ sudo -u postgres psql
could not change directory to "/home/vagrant": Permission denied
psql (14.8)
Type "help" for help.

postgres=# \|
invalid command \|
Try \? for help.
postgres=# \l
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges   
-----------+----------+----------+-------------+-------------+-----------------------
 otus      | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
(4 rows)

postgres=# ^C
postgres=# select * from pg_stat_replication;
  pid  | usesysid |   usename   |  application_name  |  client_addr  | client_hostname | 
client_port |         backend_start         | backend_xmin |   state   | sent_lsn  | writ
e_lsn | flush_lsn | replay_lsn |    write_lag    |   flush_lag    |   replay_lag    | syn
c_priority | sync_state |          reply_time           
-------+----------+-------------+--------------------+---------------+-----------------+-
------------+-------------------------------+--------------+-----------+-----------+-----
------+-----------+------------+-----------------+----------------+-----------------+----
-----------+------------+-------------------------------
 28824 |    16384 | replication | walreceiver        | 192.168.57.12 |                 | 
      56966 | 2023-06-20 10:59:31.635641-03 |          738 | streaming | 0/4000148 | 0/40
00148 | 0/4000148 | 0/4000148  |                 |                |                 |    
         0 | async      | 2023-06-20 11:14:47.556428-03
 28829 |    16385 | barman      | barman_receive_wal | 192.168.57.13 |                 | 
      39988 | 2023-06-20 10:59:34.501874-03 |              | streaming | 0/4000148 | 0/40
00148 | 0/4000000 |            | 00:00:03.723129 | 00:15:03.14564 | 00:15:13.805409 |    
         0 | async      | 2023-06-20 11:14:48.335562-03
(2 rows)
```

```bash
[vagrant@node2 ~]$ sudo -u postgres psql
could not change directory to "/home/vagrant": Permission denied
psql (14.8)
Type "help" for help.

postgres=# \l
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges   
-----------+----------+----------+-------------+-------------+-----------------------
 otus      | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
(4 rows)

postgres=# select * from pg_stat_wal_receiver;
  pid  |  status   | receive_start_lsn | receive_start_tli | written_lsn | flushed_lsn |
 received_tli |      last_msg_send_time       |     last_msg_receipt_time     | latest_e
nd_lsn |        latest_end_time        | slot_name |  sender_host  | sender_port |      
                                                                                        
                                           conninfo                                     
                                                                                        
            
-------+-----------+-------------------+-------------------+-------------+-------------+
--------------+-------------------------------+-------------------------------+---------
-------+-------------------------------+-----------+---------------+-------------+------
----------------------------------------------------------------------------------------
----------------------------------------------------------------------------------------
----------------------------------------------------------------------------------------
------------
 28217 | streaming | 0/3000000         |                 1 | 0/4000148   | 0/4000148   |
            1 | 2023-06-20 11:15:02.941212-03 | 2023-06-20 11:15:02.941356-03 | 0/400014
8      | 2023-06-20 11:04:32.088681-03 |           | 192.168.57.11 |        5432 | user=
replication password=******** channel_binding=prefer dbname=replication host=192.168.57.
11 port=5432 fallback_application_name=walreceiver sslmode=prefer sslcompression=0 sslsn
i=1 ssl_min_protocol_version=TLSv1.2 gssencmode=prefer krbsrvname=postgres target_sessio
n_attrs=any
```

Проверим работу `barman`:
```bash
[barman@barman ~]$ barman switch-wal node1
The WAL file 000000010000000000000004 has been closed on server 'node1'
[barman@barman ~]$ barman cron
Starting WAL archiving for server node1
[barman@barman ~]$ barman check node1
Server node1:
        PostgreSQL: OK
        superuser or standard user with backup privileges: OK
        PostgreSQL streaming: OK
        wal_level: OK
        replication slot: OK
        directories: OK
        retention policy settings: OK
        backup maximum age: FAILED (interval provided: 4 days, latest backup age: No available backups)
        backup minimum size: OK (0 B)
        wal maximum age: OK (no last_wal_maximum_age provided)
        wal size: OK (0 B)
        compression settings: OK
        failed backups: OK (there are 0 failed backups)
        minimum redundancy requirements: FAILED (have 0 backups, expected at least 1)
        pg_basebackup: OK
        pg_basebackup compatible: OK
        pg_basebackup supports tablespaces mapping: OK
        systemid coherence: OK (no system Id stored on disk)
        pg_receivexlog: OK
        pg_receivexlog compatible: OK
        receive-wal running: OK
        archiver errors: OK
```

Запускаем бэкап:
```bash
[barman@barman ~]$ barman backup node1
Starting backup using postgres method for server node1 in /var/lib/barman/node1/base/20230620T143519
Backup start at LSN: 0/5000148 (000000010000000000000005, 00000148)
Starting backup copy via pg_basebackup for 20230620T143519
Copy done (time: less than one second)
Finalising the backup.
This is the first backup for server node1
WAL segments preceding the current backup have been found:
        000000010000000000000004 from server node1 has been removed
Backup size: 33.5 MiB
Backup end at LSN: 0/7000000 (000000010000000000000006, 00000000)
Backup completed (start time: 2023-06-20 14:35:19.859121, elapsed time: 1 second)
Processing xlog segments from streaming for node1
        000000010000000000000005
```

Проверим восстановление из бэкапа:
```bash
[root@node1 ~]# sudo -u postgres psql
could not change directory to "/root": Permission denied
psql (14.8)
Type "help" for help.

postgres=# \l
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges   
-----------+----------+----------+-------------+-------------+-----------------------
 otus      | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
(4 rows)

postgres=# DROP DATABASE otus;
DROP DATABASE
postgres=# \l
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges   
-----------+----------+----------+-------------+-------------+-----------------------
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
(3 rows)
```

```bash
[barman@barman ~]$ barman list-backup node1
node1 20230620T143519 - Tue Jun 20 11:35:20 2023 - Size: 33.6 MiB - WAL Size: 0 B
[barman@barman ~]$ barman recover node1 20230620T143519 /var/lib/pgsql/14/data/ --remote
-ssh-command "ssh postgres@192.168.57.11"
The authenticity of host '192.168.57.11 (192.168.57.11)' can't be established.
ECDSA key fingerprint is SHA256:QX9yQL/OTh/80Fd0yTjyAgsmZ0VzZskzyeOpYkkX9i0.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Starting remote restore for server node1 using backup 20230620T143519
Destination directory: /var/lib/pgsql/14/data/
Remote command: ssh postgres@192.168.57.11
Copying the base backup.
Copying required WAL segments.
Generating archive status files
Identify dangerous settings in destination directory.

Recovery completed (start time: 2023-06-20 14:40:25.696859+00:00, elapsed time: 4 seconds)
Your PostgreSQL server has been successfully prepared for recovery!
```

```bash
[root@node1 ~]# systemctl restart postgresql-14.service 
[root@node1 ~]# sudo -u postgres psql
could not change directory to "/root": Permission denied
psql (14.8)
Type "help" for help.

postgres=# \l
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges   
-----------+----------+----------+-------------+-------------+-----------------------
 otus      | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
(4 rows)
```
Видно, что база данныч `otus` восстановилась.
