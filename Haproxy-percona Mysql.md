## Xây dựng Mysql Percona và haproxy cho các server MySQL với HAproxy trên Ubuntu


## I.Xây dựng mysql Percona Mô hình Master-Master-Master (M-M-M)



### Bước 1: Thực hiện trên 3 node lần lượt như sau:

```

sudo apt update
sudo apt install -y wget gnupg2 lsb-release curl
wget https://repo.percona.com/apt/percona-release_latest.generic_all.deb
sudo dpkg -i percona-release_latest.generic_all.deb
sudo apt update
#sudo percona-release setup pxc80
sudo apt install percona-xtradb-cluster-57
```
### Bước 2: Thực hiện việc cấu hình replication

- Thực hiện trên node 1:


```
$ sudo service mysql stop
```
Sửa file **`/etc/mysql/percona-xtradb-cluster.conf.d/wsrep.conf:`**

```
[mysqld]

wsrep_provider=/usr/lib/galera3/libgalera_smm.so
wsrep_cluster_address=gcomm://10.5.10.38,10.5.9.28,10.5.9.36
binlog_format=ROW
default_storage_engine=InnoDB
wsrep_slave_threads= 8
wsrep_log_conflicts
innodb_autoinc_lock_mode=2
wsrep_node_address=10.5.10.38
wsrep_cluster_name=pxc-cluster
wsrep_node_name=pxc1
#pxc_strict_mode allowed values: DISABLED,PERMISSIVE,ENFORCING,MASTER
pxc_strict_mode=ENFORCING
wsrep_sst_method=xtrabackup-v2
#wsrep_sst_method=rsync
#Authentication for SST method
wsrep_sst_auth="phuc:1"

```

- Nếu chúng ta setup **`wsrep_sst_auth`** thì cần tạo DB với password đúng như vậy để phục vụ cho việc xác thực (tạo ở 1 node duy nhất):

```
mysql@node1> CREATE USER 'phuc'@'1' IDENTIFIED BY 'passw0rd';
mysql@node1> GRANT RELOAD, LOCK TABLES, PROCESS, REPLICATION CLIENT ON *.* TO 'phuc'@'localhost';
mysql@node1> FLUSH PRIVILEGES;
```



- Trông nó như sau:


![image](https://user-images.githubusercontent.com/83824403/181480054-cf5a79af-92bc-49b3-bd0a-12735e6315e8.png)


- Thực hiện trên node 2,3 tương tự nhau :

- Thiết lập **`nút 2 và nút 3`** giống nhau: Vẫn stop servicev và Tất cả các cấu hình giống node 1 ngoại trừ **`wsrep_node_name và wsrep_node_address.`**

```
For node 2
wsrep_node_name=pxc2
wsrep_node_address=10.5.9.28

For node 3
wsrep_node_address=10.5.9.36
wsrep_node_name=pxc3


```

### Bước 3: Boootstrap node 1

- chạy lệnh sau trên **`node 1`**

```
[root@m1 ~]# systemctl start mysql@bootstrap.service
mysql@m1> show status like 'wsrep%';
```
- Về cơ bản nó sẽ như sau:

```
+----------------------------+--------------------------------------+
| Variable_name              | Value                                |
+----------------------------+--------------------------------------+
| wsrep_local_state_uuid     | c2883338-834d-11e2-0800-03c9c68e41ec |
| ...                        | ...                                  |
| wsrep_local_state          | 4                                    |
| wsrep_local_state_comment  | Synced                               |
| ...                        | ...                                  |
| wsrep_cluster_size         | 1                                    |
| wsrep_cluster_status       | Primary                              |
| wsrep_connected            | ON                                   |
| ...                        | ...                                  |
| wsrep_ready                | ON                                   |
+----------------------------+--------------------------------------+
40 rows in set (0.01 sec)
```

### Bước 4: join 2 node còn lại (tương tự nhau)

- Thực hiện trên **`node 2`**

```
[root@m2 ~]# systemctl start mysql
mysql@m2> show status like 'wsrep%';

```

```
+----------------------------------+--------------------------------------------------+
| Variable_name                    | Value                                            |
+----------------------------------+--------------------------------------------------+
| wsrep_local_state_uuid           | a08247c1-5807-11ea-b285-e3a50c8efb41             |
| ...                              | ...                                              |
| wsrep_local_state                | 4                                                |
| wsrep_local_state_comment        | Synced                                           |
| ...                              |                                                  |
| wsrep_cluster_size               | 2                                                |
| wsrep_cluster_status             | Primary                                          |
| wsrep_connected                  | ON                                               |
| ...                              | ...                                              |
| wsrep_provider_capabilities      | :MULTI_MASTER:CERTIFICATION: ...                 |
| wsrep_provider_name              | Galera                                           |
| wsrep_provider_vendor            | Codership Oy <info@codership.com>                |
| wsrep_provider_version           | 4.3(r752664d)                                    |
| wsrep_ready                      | ON                                               |
| ...                              | ...                                              |
+----------------------------------+--------------------------------------------------+
75 rows in set (0.00 sec)
```
- Thực hiện trên **`node 3:`**

```
[root@m3 ~]# systemctl start mysql
mysql@m3 show status like 'wsrep%';
```

```
+----------------------------+--------------------------------------+
| Variable_name              | Value                                |
+----------------------------+--------------------------------------+
| wsrep_local_state_uuid     | c2883338-834d-11e2-0800-03c9c68e41ec |
| ...                        | ...                                  |
| wsrep_local_state          | 4                                    |
| wsrep_local_state_comment  | Synced                               |
| ...                        | ...                                  |
| wsrep_cluster_size         | 3                                    |
| wsrep_cluster_status       | Primary                              |
| wsrep_connected            | ON                                   |
| ...                        | ...                                  |
| wsrep_ready                | ON                                   |
+----------------------------+--------------------------------------+
40 rows in set (0.01 sec)
```

![image](https://user-images.githubusercontent.com/83824403/181483027-02a4a71a-8ee0-4558-a336-0ce81caeda33.png)

## Bước 5: Thực hiện việc test DB với các node

- Thực hiện ghi DB vào bất kì node nào và check sync trên các node còn lại
- Stop 1 node bất kì, vẫn tiếp tục ghi DB vào trên các node khác và bật lại => kiểm tra
- Stop 2 node và ghi liên tục vào node còn lại sau đó bật 2 node đã stop lên => kiểm tra

=>> Kết quả của 3 PA trên là DB đều được sync đầy đủ ko thiếu 1 trường nào

![image](https://user-images.githubusercontent.com/83824403/181483963-33bbc50d-e28b-4ea7-8927-64d419ad661a.png)
 
- DB trên 3 node đều như này



## Bench march cho cụm M_M percona này

**- Step 1:** Cài sysbench

```
apt install sysbench
```
**- Step 2:** Phân quyền cho DB và host để phục vụ mục đích bench march

```
mysql> create database sysbench
mysql> GRANT ALL ON sysbench* to 'phuc'@'10.5.9.170' IDENTIFIED BY '1';
mysql> FLUSH PRIVILEGES;
mysql> SELECT host FROM mysql.user WHERE user = "phuc";
```

![image](https://user-images.githubusercontent.com/83824403/181728688-2e530aec-adb5-4c82-b9bb-e0b7b8d51158.png)


**- Step 3:** Tiến hành bench march

```
sysbench /usr/share/sysbench/oltp_read_write.lua --mysql-host=10.5.9.170 --mysql-port=3306 --mysql-user=phuc --mysql-password='1' --mysql-db=sysbench --db-driver=mysql --tables=2 --table-size=200000  prepare
```
**=> Việc trên là chúng ta đã insest liên tục vào db sysbench 2 table và mỗi table 2 triệu row**

**=> Quá trình trên thực hiện trên node 2 của cụm và trong lúc ghi chúng ta sẽ stop node 1 và node 3**

**=> Khi quá trình success thì bật lại 2 node để check**

![image](https://user-images.githubusercontent.com/83824403/181729540-df483ea6-b08b-41e9-9bff-ab4386d48004.png)


- **Step 4:** Tiến hành bật node 1 và node 3. Show xem quá trình replication sang 2 node xem như nào

![image](https://user-images.githubusercontent.com/83824403/181730808-9fef3efa-23cd-4dd5-be13-ae016309fc57.png)

- Lệnh cuối cùng sẽ show ra rất nhiều row. DEMO:

![image](https://user-images.githubusercontent.com/83824403/181731030-8aa047e6-ccc1-4d31-9945-0dcaaf59c98f.png)

- **`Và cuối cùng là Node 3 cũng tương tự như vậy`**

- Việc benchmark thử thực hiện 1 triệu phép tính 1+1 một lúc, kết quả mysql service cần 17.8s để thực hiện xong

![image](https://user-images.githubusercontent.com/83824403/181866913-d484554f-35c3-4aae-874f-03eb987f340f.png)


- Chạy xem performance đọc ghi của DB

```
sysbench /usr/share/sysbench/oltp_read_write.lua --db-driver=mysql --mysql-host=10.5.9.170,10.5.8.249,10.5.10.214 --mysql-user=phuc --mysql-password=1 --mysql-port=3306 --max-requests=1000000  --mysql-db=sysbench --time=120 run
```

![image](https://user-images.githubusercontent.com/83824403/181867844-41ccdc4c-de65-4fa5-8f1f-34433b7c729c.png)




### Bước 6: Cài Haproxy phục vụ việc failover service mysql M-M percona

https://www.percona.com/doc/percona-xtradb-cluster/5.7/howtos/haproxy.html


## II.Xây dựng mysql Percona Mô hình Master-Slave (M-S)
### Mô hình
### Bước 1: Thực hiện trên 2 node lần lượt như sau:

```

sudo apt update
sudo apt install -y wget gnupg2 lsb-release curl
wget https://repo.percona.com/apt/percona-release_latest.generic_all.deb
sudo dpkg -i percona-release_latest.generic_all.deb
sudo apt update
#sudo percona-release setup pxc80
sudo apt install percona-xtradb-cluster-57
```


### Bước 2: Tạo DB, thực hiện việc connect replication

#### Thực hiện trên node master

- Sửa thông tin của file **` /etc/mysql/percona-xtradb-cluster.conf.d/mysqld.cnf `** như sau
```
[mysqld]
server-id=1
```

- Tạo user phục vụ việc replication

```
CREATE USER 'slave'@'10.5.9.106' IDENTIFIED BY '1';

GRANT REPLICATION SLAVE ON *.* TO slave IDENTIFIED BY '1' WITH GRANT OPTION;

flush privileges;
```
- show master status

![image](https://user-images.githubusercontent.com/83824403/182075211-6917ecef-330e-4277-b3eb-2390101f95ca.png)



#### Thực hiện trên node slave

- Sửa thông tin của file **` /etc/mysql/percona-xtradb-cluster.conf.d/mysqld.cnf `** như sau
```
[mysqld]
server-id=2
```

- Thực hiện việc kết nối với master

```
mysql> stop slave;
mysql> CHANGE MASTER TO MASTER_HOST='10.5.9.106',MASTER_USER='slave', MASTER_PASSWORD='1', MASTER_LOG_FILE='m-bin.000003', MASTER_LOG_POS= 360;
mysql> start slave;
mysql> show slave status\G;
```

![image](https://user-images.githubusercontent.com/83824403/182093019-527517c1-1392-4ea6-8dce-1ccd4c843998.png)

- Khéo xuống dưới sẽ có state như sau
```
Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
```


### Buớc 3: Tạo DB, test việc replication của DB

- Trên master

```
mysql> create database PHUCDV;
mysql> use PHUCDV;
mysql> create table thong_tin;
mysql> create table thong_tin( ten varchar (10), quequan varchar(20), tuoi int(3), hocvan varchar(30);
mysql> insert into thong_tin values("phuc","hy","22","dh");

```
- Sang node slave xem thử

```
mysql> describe thong_tin;
mysql> select * from thong_tin;
```

![image](https://user-images.githubusercontent.com/83824403/182094244-73bdd4c9-c459-41e5-8173-32d418d31de2.png)

- Như vậy là quá trình replication hoàn thành

### Bước 4: Tiến hành Benchmark

- Node master

#### Step 1: Cài sysbench
```
apt install sysbench
```
#### Step 2: Phân quyền cho DB và host để phục vụ mục đích bench march

```
mysql> create database sysbench
mysql> GRANT ALL ON sysbench* to 'phuc'@'10.5.9.106' IDENTIFIED BY '1';
mysql> FLUSH PRIVILEGES;
mysql> SELECT host FROM mysql.user WHERE user = "phuc";
```
#### Step 3: Tiến hành bench march
```
sysbench /usr/share/sysbench/oltp_read_write.lua --mysql-host=10.5.9.106 --mysql-port=3306 --mysql-user=phuc --mysql-password='1' --mysql-db=sysbench --db-driver=mysql --tables=2 --table-size=200000  prepare
```
- Câu lệnh trên chúng ta sẽ thêm 2 table, mỗi table 2 triệu row vào db sysbench 

![image](https://user-images.githubusercontent.com/83824403/182094920-8a4bcd3f-8595-4fe3-9c29-862bc5c1928a.png)

- Tiến hành benchmark

```
sysbench /usr/share/sysbench/oltp_read_write.lua --db-driver=mysql --mysql-host=10.5.9.106,10.5.8.159 --mysql-user=phuc --mysql-password=1 --mysql-port=3306 --max-requests=1000000  --mysql-db=sysbench --time=120 run
```
- Kết quả của việc benmarch 2 cụm server (Master-Slave) như sau:
- So sánh performance giữa mô hình M-M-M ở trên 


![image](https://user-images.githubusercontent.com/83824403/182094492-ddbbed10-f5ff-4d95-af59-c49268d629c5.png)
