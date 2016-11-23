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
