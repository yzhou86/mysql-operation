# mysql-operation

## Basic mysql operation

### User and privileges
```
mysql> CREATE USER 'finley'@'localhost' IDENTIFIED BY 'some_pass';
mysql> GRANT ALL PRIVILEGES ON *.* TO 'finley'@'localhost' WITH GRANT OPTION;
mysql> CREATE USER 'finley'@'%' IDENTIFIED BY 'some_pass';
mysql> GRANT ALL PRIVILEGES ON *.* TO 'finley'@'%' WITH GRANT OPTION;
mysql> CREATE USER 'admin'@'localhost' IDENTIFIED BY 'admin_pass';
mysql> GRANT RELOAD,PROCESS ON *.* TO 'admin'@'localhost';
mysql> CREATE USER 'dummy'@'localhost';
```

### User password
#### Method 1
```
mysql> use mysql
mysql> update user set password = password('new_password') where user = 'user_name';
mysql> flush privileges;
```
#### Method 2
```
mysql> set password for user_name = password('new_password');
mysql> flush privileges;
```
#### Method 3
```
grant all privileges on db.table to user_name@localhost identified by 'your_pwd';
```

## Export and Import, to decrease size of ibdata file
* stop service accessing mysql
* Dump out database.
```
> mysqldump  -uroot -p --routines --databases TESTDB| gzip > /var/fullTESTDB-dump1.sql.zip
> ctrl+z
> bg
```
* drop schema TESTDB;
```
> mysql -uroot -p
> drop schema TESTDB;
```
* stop mysql service.
```
> service mysql stop
```
* delete ibdata files, delete ib_log files.
```
> cd /var/lib/mysql
> rm ibdata*
> rm ib_logfile*
```
* start mysql service.
```
> service mysql start
```
* create database TESTDB;
```
> mysql -uroot -p
mysql> create database TESTDB;
```
* dump in backuped TESTDB database.
```
> zcat /var/fullTESTDB-dump1.sql.zip | mysql -uroot -p TESTDB
> ctrl+z
> bg
```
* start service.

* Config slave master replication;

   On master:
```
mysql> show master status;
```
   On slave:
```
mysql> stop slave;
mysql> reset slave;
mysql> restart mysql;
mysql> CHANGE MASTER TO MASTER_HOST='10.252.56.106',MASTER_USER='repluser',MASTER_PASSWORD='replpass',MASTER_PORT=3306,MASTER_LOG_FILE='mysql-bin.004601',MASTER_LOG_POS=41758892;
mysql> start slave;
mysql> show slave status\G;
```
* Update replication user pass
```
mysql> GRANT FILE, REPLICATION SLAVE ON *.* TO 'repluser'@'10.252.56.107' IDENTIFIED BY 'replpass';
mysql> GRANT FILE, REPLICATION SLAVE ON *.* TO 'repluser'@'10.252.56.107' IDENTIFIED BY 'replpass’;
```
* Skip replication error
```
mysql> SET GLOBAL SQL_SLAVE_SKIP_COUNTER=1; START SLAVE;
```

## Master Slave replication config
* This is for Mysql 5 replication

* A Server： 192.168.1.2 master
* B Server： 192.168.1.3 slave

* A Server config
```
　　> mysql –u root –p
　　mysql> GRANT FILE ON *.* TO backup@192.168.1.3 IDENTIFIED BY ‘1234';
　　mysql> exit 
　　> mysqladmin –u root –p shutdown
```
　　Export the database data from master mysql server, then import to slave mysql server

　　On master mysql server A, edit config file: /etc/my.cnf

　　Append lines to section [mysqld]
```
　　log-bin=mysql-bin
　　server-id=1
　　binlog_do_db = gbbbs
　　binlog_ignore_db = mysql,test,information_schema
```

　　Restart Server A mysql

* B Server config

　　Update file: /etc/my.cnf

　　Append lines to section [mysqld]
```
　　server-id = 2
　　master-host = 192.168.112.71
　　master-user = backup
　　master-password = 1234
　　replicate-do-db = gbbbs
　　#replicate-do-db = database2
　　master-port=3306
　　master-connect-retry = 60
 ```
　　Restart B server mysql service

* Query log position of master mysql server, server A:         
```
mysql> show master status;
```
* Operation on slave mysql server, server B
```
mysql> slave stop;
mysql> slave start;
mysql> show slave status;
```
If you see this, then it works.
```
Slave_IO_Running: Yes
Slave_SQL_Running: Yes
```
### Migrate mysql database data issue and fix

Am struggling with getting slaving working, and I'm wondering someone could help. Not necessarily solve it, but SHOW ME THE TROUBLESHOOTING method that is typical here.

What I've done:
* on master box (note I did not try --master-data)
```
mysql> flush tables with read lock;
mysql> show master status\G;
$ mysqldump -u root -A > all_database.mysql
mysql> unlock tables;
```
* copy to slave box
* on slave box
```
mysql> stop slave; reset slave;
$ mysql -u root < all_database.mysql
```
* change master to ... (details from show master status above)
* mysql> start slave

### More tips about replication issue
When found that the replication not work, check the slave status:
```
mysql> show slave status\G
*************************** 1. row ***************************
               Slave_IO_State:
                  Master_Host: xxxx

                  Master_User: xxx
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: xxx-bin.000001
          Read_Master_Log_Pos: 1440046
               Relay_Log_File: host-relay-bin.000001
                Relay_Log_Pos: 4
        Relay_Master_Log_File:xxx-bin.000001
             Slave_IO_Running: No
            Slave_SQL_Running: Yes
              Replicate_Do_DB: xxx
          Replicate_Ignore_DB: mysql
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 1440046
              Relay_Log_Space: 106
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File:
           Master_SSL_CA_Path:
              Master_SSL_Cert:
            Master_SSL_Cipher:
               Master_SSL_Key:
        Seconds_Behind_Master: NULL
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 1236
                Last_IO_Error: Got fatal error 1236 from master when reading data from binary log: 'Could not find first log file name in binary log index file'
               Last_SQL_Errno: 0
               Last_SQL_Error:
1 row in set (0.00 sec)
```
 

And read from slave mysql, the log file print error logs：
```
101027 16:50:58 [Note] Slave I/O thread: connected to master 'xxx@xxxxxx:3306',replication started in log 'xxx-bin.000001' at position 1440046
101027 16:50:58 [ERROR] Error reading packet from server: Could not find first log file name in binary log index file ( server_errno=1236)
101027 16:50:58 [ERROR] Slave I/O: Got fatal error 1236 from master when reading data from binary log: 'Could not find first log file name in binary log index file', Error_code: 1236
101027 16:50:58 [Note] Slave I/O thread exiting, read up to log 'xxx-bin.000001', position 1440046
```
And check the master status:
```
mysql> show master status\G
*************************** 1. row ***************************
            File: mysql-bin.000001
        Position: 1440046
    Binlog_Do_DB:
Binlog_Ignore_DB:
1 row in set (0.00 sec)
```
As a common fix steps, it would be:
```
mysql> slave stop;
mysql> CHANGE MASTER TO MASTER_HOST='xxx',MASTER_USER='xx',MASTER_PASSWORD='xx',MASTER_PORT=3306,MASTER_LOG_FILE='xxx-bin.000001',MASTER_LOG_POS=1440046;
mysql> slave start;
```
But if you found issue still there, you can do this to fix:

1. restart slave mysql service

2. regrant privileges on master to replication user on slave.

3. Run these commands:
```
slave stop;
reset slave;
slave start;
```
