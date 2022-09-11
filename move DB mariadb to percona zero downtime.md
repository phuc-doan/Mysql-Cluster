# Move DB mariadb to percona zero downtime

## Prepare 
- Mô hình đang có sẵn 


![image](https://user-images.githubusercontent.com/83824403/186573923-7db11cd3-c618-4067-8239-a9c809e423bc.png)

## Thực hiện

![image](https://user-images.githubusercontent.com/83824403/186574476-47f3db83-07b7-4dbc-b9f7-04b1f73a6def.png)
 
 
 - Targert là join node Percona.1 vào cluster A 


### Bước 1: 
- Cài cắm các dịch vụ, service bắt buộc

### Cluster A:
```
## install mariadb

```

### Cluster B:
```
sudo apt update
sudo apt install -y wget gnupg2 lsb-release curl
wget https://repo.percona.com/apt/percona-release_latest.generic_all.deb
sudo dpkg -i percona-release_latest.generic_all.deb
sudo apt update

sudo apt install percona-xtradb-cluster-57
### Sau đó cài mô hình Percona Master Master để giống được như mô hìnhh
```
#### node Mariadb.IP1
```

[mysqld]
bind-address = 10.5.69.173
server_id=1
report_host = master
log_bin = /var/lib/mysql/mariadb-bin
log_bin_index = /var/lib/mysql/mariadb-bin.index
relay_log = /var/lib/mysql/relay-bin
relay_log_index = /var/lib/mysql/relay-bin.index
[client-server]
```

#### node Percona.1 

- sửa file /etc/mysql/my.cnf, thêm những dòng sau

```
[mysqld]
server_id = 2
report_host = master2
log_bin = /var/lib/mysql/mariadb-bin
log_bin_index = /var/lib/mysql/mariadb-bin.index
relay_log = /var/lib/mysql/relay-bin

relay_log_index = /var/lib/mysql/relay-bin.index
```
- Đảm bảo server_id trên **Percona.1 là 2**. Con số này phải được thống nhất



- Chúng ta cần vào file config **wsrep.conf** để sửa mode ENFORCING thành PERMISSION 


![image](https://user-images.githubusercontent.com/83824403/189520607-d23e41a0-77bc-45c8-a305-c05914275f70.png)

*Tham khảo https://docs.percona.com/percona-xtradb-cluster/5.7/features/pxc-strict-mode.html


```
 Booostrap lại service
/etc/init.d/mysql bootstrap-pxc
```



### Bước 2: Dump DB từ node mariadb sang mysql

- Lưu ý trước khi dump show master status trên mariadb và lưu ý OUTPUT

![image](https://user-images.githubusercontent.com/83824403/189519681-80410915-312f-4592-a5c6-649769d78b0e.png)

*NOTE: Lưu ý OUTPUT lúc này




### Bước 3: TEST

- Trên node maridb lúc này ta dump và restore DB mới. Lúc này hoàn toàn vẫn chưa join node mysql vào cụm ( Mục đích là làm thay đổi DB để OUTPUT ở trên cũng thay đổi)


```
root@mariadb:/mnt# time mysqldump -h 10.5.69.173 -u backup -p  etherpad > /mnt/db.sql
MariaDB [(none)]> create database etherpad; 
root@mariadb:/mnt# mysql etherpad < db.sql
```


**Lúc này show master status trên maridb, ta thấy OUTPUT không còn như trước



![image](https://user-images.githubusercontent.com/83824403/189519977-675b91de-1c1b-40a8-9820-b30f93551688.png)


### Bước 4, Join node mysql vào cụm. Tạo replication


- Cho phep user để replicate trên node maridb
```
STOP SLAVE;

GRANT REPLICATION SLAVE ON *.* TO 'slave_user'@'%' IDENTIFIED BY 'password';
FLUSH PRIVILEGES;

```


- Tạo replication, Pos và master log file ta lấy từ thời điểm trước khi tạo thêm DB


```
Stop slave
CHANGE MASTER TO MASTER_HOST='10.5.9.182', MASTER_USER='slave_user', MASTER_PASSWORD='password', MASTER_LOG_FILE='mariadb-bin.000005', MASTER_LOG_POS=26445659
;
start slave;
Show slave status\G;

```

- OUTPUT như sau 

![image](https://user-images.githubusercontent.com/83824403/189520221-5572ceb1-6058-44de-b48e-7614c0911915.png)





*- Note: Phần *master_log_file* và *Master_log_pos*chúng ta lấy ở Mariadb.IP1 bằng lệnh *show master status*;*



### Test:

- Đứng ở node mysql vừa thêm vào cụm, ta show database, lúc này đã có DB **`etherpad`** trước đõ đã ghi mới ở cụm Maridb. Describe Table trong DB này, OUTPUT như sau:

![image](https://user-images.githubusercontent.com/83824403/189520351-745b1827-6ac2-4a1e-89dd-d0d93ff2c6d9.png)

- Select DB đã có giống node MariaDB cũ


- PP này giống việc Restore Point in Time của Mysql. Sau khi đã Dump ở 1 thời điểm, ta lợi dụng binlog để restore về thời điểm mới gần nhất

**`Như vậy việc replicate giữa Mariadb và Mysql đã thành công`**
#### Lúc này giả sử môi trường Production có Request write và CLUSTER A thì bên CLUSTER B vẫn synced bình thường




- Sau khi Dump và Restore DB cần thiết, chúng ta sao lưu tiếp đến User và quyền User 



```
# mysql -B -N -h 10.5.69.173 -u backup -p -e "SELECT CONCAT('\'', user,'\'@\'', host, '\'') FROM user WHERE user != 'debian-sys-maint' AND user != 'root' AND user != ''" mysql > /mnt/mysql_all_users.txt - Dump tất cả user của cụm cũ
# while read line; do mysql -B -N -h 10.5.69.173 -u backup -p -e "SHOW GRANTS FOR $line"; done < mysql_all_users.txt > mysql_all_users_sql.sql - Dump tất cả các quyền cho user ở cụm cũ
# sed -i 's/$/;/' /mnt/mysql_all_users_sql.sql - Chèn dấu ; vào cuối mỗi dòng trong file User-priviledges.sql

Restore User, User Priviledges cho cụm mới
# mysql core_dashboard < /mnt/mysql_all_users_sql.sql - Restore lại toàn bộ quyền và user cho DB mới


````




