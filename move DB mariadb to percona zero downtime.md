# Move DB mariadb to percona zero downtime

## Prepare 
- Mô hình đang có sẵn 


![image](https://user-images.githubusercontent.com/83824403/186573923-7db11cd3-c618-4067-8239-a9c809e423bc.png)

## Thực hiện

![image](https://user-images.githubusercontent.com/83824403/186574476-47f3db83-07b7-4dbc-b9f7-04b1f73a6def.png)
 
 
 - Targert là join node Percona.1 vào cluster A phục vụ mục đích Dual Write, vì bản chất replication mariadb không chỉ replicate DB từ lúc Register giữa master và Slave, còn những DB trước đó thì không
 - Vậy nên ở trong bài trước, ta luôn cần Backup và import to new DB 

### Bước 1: 
- Cài cắm các dịch vụ, service bắt buộc

### Cluster A:
```
## install mariadb
## import All DB test to Mariadb
```

### Cluster B:
```
sudo apt update
sudo apt install -y wget gnupg2 lsb-release curl
wget https://repo.percona.com/apt/percona-release_latest.generic_all.deb
sudo dpkg -i percona-release_latest.generic_all.deb
sudo apt update

sudo apt install percona-xtradb-cluster-57
### Sau đó cài mô hình Percona Master Master để giống được như mô hìnhhình]
```

### Bước 2: Join Percona.1 to Mariadb.IP1



### node Mariadb.IP1
```

[mysqld]
bind-address = 10.5.9.182
server_id=1
report_host = master
log_bin = /var/lib/mysql/mariadb-bin
log_bin_index = /var/lib/mysql/mariadb-bin.index
relay_log = /var/lib/mysql/relay-bin
relay_log_index = /var/lib/mysql/relay-bin.index
[client-server]
```
- Sau đó cho phep user để replicate
```
STOP SLAVE;

GRANT REPLICATION SLAVE ON *.* TO 'slave_user'@'%' IDENTIFIED BY 'password';
FLUSH PRIVILEGES;

```




### node Percona.1 

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
- Đảm bảo server_id trên **Percona.1** là 2. Con số này phải được thống nhất


- Join vào cluster M-S của mariadb
- Chúng ta cần vào file config wsrep.conf để sửa mode ENFORCING thành PERMISSION để tránh lỗi như này

![image](https://user-images.githubusercontent.com/83824403/186576662-b55d709b-50bc-41fe-a06b-8d8c5f8ec176.png)

```
 Booostrap lại service
/etc/init.d/mysql bootstrap-pxc
```

#### Restart lại service


```
Stop slave
CHANGE MASTER TO MASTER_HOST='10.5.9.182', MASTER_USER='slave_user', MASTER_PASSWORD='password', MASTER_LOG_FILE='mariadb-bin.000002', MASTER_LOG_POS=2018;
Show slave status\G;
```

*- Note: Phần *master_log_file* và *Master_log_pos*chúng ta lấy ở Mariadb.IP1 bằng lệnh *show master status*;*


- Cuối cùng như sau
![image](https://user-images.githubusercontent.com/83824403/186577379-7c419176-2d29-4a6a-9563-f4dc62d83a5c.png)

### Test:

- Đứng ở Mariadb.IP1 tạo DB, table, Import vào Table đó

```
MariaDB [(none)]> create database PHUCDV;
MariaDB [(none)]> use PHUCDV;
MariaDB [PHUCDV]> create table thong_tin ( ten varchar(5), quequan varchar(20), ABC_test varchar(20));
MariaDB [PHUCDV]> insert into thong_tin values ( 'phuc','hy','test replication mariadb to percona');
```
![image](https://user-images.githubusercontent.com/83824403/186578059-005ef69b-d2cb-4a25-8067-3475e6ba5858.png)


 
- Sang Percona.1, show DB

![image](https://user-images.githubusercontent.com/83824403/186578196-3e8d04ae-947d-429b-bc02-f3e08872b7e1.png)


#### Như vậy việc replicate giữa Mariadb và Mysql đã thành công
#### Lúc này giả sử môi trường Production có Request write và CLUSTER A thì bên CLUSTER B vẫn synced bình thường


### Bước 2: Import DB

- Đứng từ Percona.1 chúng ta chạy mysqldump và trỏ đến host là Mariadb.IP1

```
  time mysqldump -h 10.5.69.173 -u backup -p  core_dashboard  --ignore-table=core_dashboard.msgrcpt --ignore-table=core_dashboard.msgs > /mnt/db-data.sql 

```

- Đứng trên Percona.1 chúng ta Restore chính DB cần thiết

```
mysql core_dashboard < /mnt/db-data.sql
```

- Sau khi Dump và Restore DB cần thiết, chúng ta sao lưu tiếp đến User và quyền User



```
# mysql -B -N -h 10.5.69.173 -u backup -p -e "SELECT CONCAT('\'', user,'\'@\'', host, '\'') FROM user WHERE user != 'debian-sys-maint' AND user != 'root' AND user != ''" mysql > /mnt/mysql_all_users.txt - Dump tất cả user của cụm cũ
# while read line; do mysql -B -N -h 10.5.69.173 -u backup -p -e "SHOW GRANTS FOR $line"; done < mysql_all_users.txt > mysql_all_users_sql.sql - Dump tất cả các quyền cho user ở cụm cũ
# sed -i 's/$/;/' /mnt/mysql_all_users_sql.sql - Chèn dấu ; vào cuối mỗi dòng trong file User-priviledges.sql

Restore User, User Priviledges cho cụm mới
# mysql core_dashboard < /mnt/mysql_all_users_sql.sql - Restore lại toàn bộ quyền và user cho DB mới


````




