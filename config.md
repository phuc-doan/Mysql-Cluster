## Bài Lab cài trên 2 server có IP lần lượt là 
- 192.168.252.151 (slave)
- 192.168.252.152 (master)

### Let get started!


- Trên ALL node cài Mysql/Mariadb (Nếu lab thì nên dùng mariadb vì password ko require theo tiêu chuẩn, Nếu mô hình thực thì chọn mysql vì bảo mật hơn - **`ý kiến cá nhân`**)

 ## Step1: Cấu hình replication MariaDB Master/slave

- Cài đặt MariaDB

```
rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2022
rpm -Uvh https://repo.mysql.com/mysql57-community-release-el7-11.noarch.rpm
yum install -y mariadb-server mariadb
```

- Khởi động dịch vụ MariaDB trên mỗi node

```
systemctl start mariadb
```

- Chạy cài đặt an toàn cho mariadb

```
mysql_secure_installation
```
- Cấu hình tường lửa mở port 3306

```
firewall-cmd --add-port=3306/tcp --zone=public --permanent
firewall-cmd --reload
```
- Bật Binary Logging

## Step2: Chỉnh cấu hình mariadb trong /etc/my.cnf trong 2 node
## Với node Master:


- Chỉnh conf của mariadb trên Node 1 và Node 2 để kích hoạt binary logging (binary logging có thể được coi là log của tất cả các câu lệnh SQL mà nó nhận được)


- Trên node master:
 
![image](https://user-images.githubusercontent.com/83824403/164171880-22fa44a0-7c96-4593-b527-92209896cf36.png)


#### Trong đó:

- **server-id** để xác định các mysql server, nếu không khai báo thông số này mysql server sẽ mặc định server-id là 0 hay nếu đặt server-id trên các mysql server giống nhau thì sẽ không nhận các kết nối từ các server mysql khác.
- **log_bin** để khai báo file log ghi lại các thay đổi dữ liệu
- **binlog_do_db** để xác định database sẽ được replication



#### *Bạn có thể chỉ định thay vì một mà là nhiều các slave bằng cách lặp lại dòng này cho tất cả các cơ sở dữ liệu mà bạn cần:*

```
binlog_do_db            = PHUCDOAN
```

## Step3: Khởi động lại để mariadb ăn cấu hình

```
systemctl restart mariadb
```

## Step4: Giờ ta cần tạo user replication

```
mysql> grant replication slave on *.* to mysql@'%' identified by 'mysql';

mysql> flush privileges;

```
## Step5: Tao DB va phải đồng bộ database giữa Master và Slave 

- Tạo mới 1 DB


```
mysql -u root -p
```
```
mysql> create database PHUCDOAN;
```


- Đồng bộ:


```
mysqldump -u root -p PHUCDOAN > PHUCDOAN.sql
```

## Step6: Copy database sang server Slave 

```
scp replication_database.sql 192.168.252.151:/opt
```

- Lấy giá trị Position - giá trị này sẽ được Slave sử dụng để xác định thời điểm đồng bộ dữ liệu với Master.




![image](https://user-images.githubusercontent.com/83824403/164179177-cdf83c69-aaf9-4f66-bbe4-9a5833da01ca.png)



```
-   mysql> show master status;

```


## Trên Slave server:

### Step1: Cấu hình file my.cnf:

![image](https://user-images.githubusercontent.com/83824403/164178817-3b8404bd-708d-435c-b690-597fefe51ec5.png)




```
vim /etc/my.cnf
```

- Thêm 1 vài dòng sau:
```

server-id=2 
relay-log=/var/log/mysql/mysql-relay-bin.log 
binlog_do_db=PHUCDOAN
```

- Trong đó:

- **relay-log** là nơi ghi lại thông tin dữ liệu bị thay đổi được lấy từ Master server.

### Step2: Khởi động lại mysql để nhận cấu hình

```
systemctl restart mariadb
```


- Tạo database và đồng bộ dữ liệu với database của Master server
**Tên DB phải giống với DB trên Master**

```
mysql> CREATE DATABASE PHUCDOAN;

mysql -u root -p PHUCDOAN < /opt/PHUCDOAN.sql
```

- cấu hình replication trên slave bằng câu lệnh trong mysql như sau:

```
mysql> CHANGE MASTER TO MASTER_HOST='192.168.252.152',MASTER_USER='mysql', MASTER_PASSWORD='mysql', MASTER_LOG_FILE='mysql-bin.000005', MASTER_LOG_POS= 245;
```

### Step3: Start server Slave:
```
mysql> start slave;
```

### Step4: Test thôi :v






- Để kiểm tra, ta thử thay đổi dữ liệu trên **Master** bằng cách tạo 1 table mới
- Vào server Master:


```
mysql> use PHUCDOAN;

mysql> create table test(job varchar(20), salary int(10));

mysql> show tables;
```

![image](https://user-images.githubusercontent.com/83824403/164179820-dd0f0353-a6ef-4db6-8213-e0772fa4bca7.png)






- Trên Slave server ta chạy lệnh:

```
mysql> use PHUCDOAN;

mysql> show tables;

```

![image](https://user-images.githubusercontent.com/83824403/164179657-67ef8292-2233-4c6a-9b7e-2ad6f37c2d8b.png)












- Day la cau hinh `Master-Slave`. 


- Ở bài bên chúng ta sẽ config `Master-Master` và thêm `Pacemaker/Corosync` để nó switch tự đông khi 1 DB down


`AUTHOR: DOAN VAN PHUC`

### REFERENCE:
https://www.tecmint.com/mariadb-master-slave-replication-on-centos-rhel-debian/






