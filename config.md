## Bài Lab cài trên 2 server có IP lần lượt là 
- 192.168.187.130 (slave)
- 192.168.187.128 (master)
**`- Tí sau mình sẽ add thêm 1 node slave nữa là 192.168.187.131 (slave2)`**


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


- Trên node master them 1 so truong sau vao file `etc/my.conf`:

```
log-bin=mysql-bin
server-id=1
innodb_flush_log_at_trx_commit=1
sync_binlog=1
binlog_do_db=DB2204
binlog_do_db=DB2021
binlog_ignore_db=nosync
```

![image](https://user-images.githubusercontent.com/83824403/164596615-5218ac58-edc7-48ec-b223-408e25c0c214.png)



#### Trong đó:

- **server-id** để xác định các mysql server, nếu không khai báo thông số này mysql server sẽ mặc định server-id là 0 hay nếu đặt server-id trên các mysql server giống nhau thì sẽ không nhận các kết nối từ các server mysql khác.
- **log_bin** để khai báo file log ghi lại các thay đổi dữ liệu
- **binlog_do_db** để xác định database sẽ được replication
- **binlog_ignore_db** la db ko muon sync


#### *Bạn có thể chỉ định thay vì một mà là nhiều các slave bằng cách lặp lại dòng này cho tất cả các cơ sở dữ liệu mà bạn cần:*

```
binlog_do_db            = DB2204
```

## Step3: Khởi động lại để mariadb ăn cấu hình

```
systemctl restart mariadb
```

## Step4: Giờ ta cần tạo user replication

```
CREATE USER 'slave'@'192.168.187.128' IDENTIFIED BY 'Phuc123@';

GRANT REPLICATION SLAVE ON *.* TO slave IDENTIFIED BY 'Phuc123@' WITH GRANT OPTION;

flush privileges;

```


![image](https://user-images.githubusercontent.com/83824403/164597732-d1d66e5a-a9f0-471b-a437-3442c40b1ef3.png)


## Step5: Tao DB va phải đồng bộ database giữa Master và Slave 

- Tạo mới 1 DB


```
mysql -u root -p
```
```
 create database DB2204;
 create database DB2021;
 create database nosync;
```


- Đồng bộ:

- Neu ca 3 node deu moi ( chua co db thi co the bo qua buoc nay), con neu master da co db thi can thao tac de slave dong bo db master truoc
- Gia su o day minh lam la `DB2021 chua co tren slave`


```
mysqldump -u root -p DB2021 > DB2021.sql
```

## Step6: Copy database sang server Slave 

```
scp DB2021.sql 192.168.187.130:/tmp
scp DB2021.sql 192.168.187.131:/tmp
```



- Lấy giá trị Position - giá trị này sẽ được Slave sử dụng để xác định thời điểm đồng bộ dữ liệu với Master.



![image](https://user-images.githubusercontent.com/83824403/164599549-1e831f43-0a14-44c8-8e91-e4f07b629212.png)




```
-   mysql> show master status;

```


## Trên Slave server:

### Step1: Cấu hình file my.cnf:






```
vim /etc/my.cnf
```

- Thêm 1 vài dòng sau:
```

server-id=2
binlog_do_db=DB2204
binlog_do_db=DB2021

```



![image](https://user-images.githubusercontent.com/83824403/164599657-1dbcb164-18fe-45b6-a77b-55ee40643ac5.png)




- Trong đó:

- **relay-log** là nơi ghi lại thông tin dữ liệu bị thay đổi được lấy từ Master server.




### Step2: Khởi động lại mysql để nhận cấu hình

```
systemctl restart mariadb
```


- Tạo database và đồng bộ dữ liệu với database của Master server
**Tên DB phải giống với DB trên Master**

```
 CREATE DATABASE DB2021;
create database 2204;
```


#### Get data tu con master vua scp sang

```
mysql -u root -p DB2021 < /opt/DB2021.sql
```

- cấu hình replication trên slave bằng câu lệnh trong mysql như sau:

```
mysql> CHANGE MASTER TO MASTER_HOST='192.168.187.128',MASTER_USER='slave', MASTER_PASSWORD='Phuc123@', MASTER_LOG_FILE='mysql-bin.000005', MASTER_LOG_POS= 1797;
```

*nhung thong so tren la nhung thong so rat quan trong*


![image](https://user-images.githubusercontent.com/83824403/164600303-a573a15b-5934-42d9-8181-0dd2c97924a2.png)


### Step3: Start server Slave:
```
start slave;
```
- show slave status;



### Step4: Test thôi :v






- Để kiểm tra, ta thử thay đổi dữ liệu trên **Master** bằng cách tạo 1 table mới
- Vào server Master:


```
mysql> use DB2204;

mysql> create test (job varchar(20), salary int(10));

mysql> show tables;
```

![image](https://user-images.githubusercontent.com/83824403/164600604-40373bda-35bf-4e1c-9497-667da1bf1ab2.png)







- Trên Slave server ta chạy lệnh:

```
mysql> use DB2204;

mysql> show tables;

```

![image](https://user-images.githubusercontent.com/83824403/164600706-9c87324c-62c9-4e56-a17b-b4e6dcf8a8d1.png)





**chac chan 1 dieu la DB `nosync` khi chung ta write thi slave se ko nhan dc vi trong phan config o tren thong so dat truoc no da la `binlog_ignore_db=nosync`**







- Day la cau hinh `Master-Slave`. 




## Phụ: Cấu hình add thêm 1 hoặc nhiều slave vào diagram

### Step1: Trên node slave2 cũng tải và cài mysql/mariadb

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


### CHỉnh file *`etc/hosts/`* 

- Ta sẽ chỉnh lại cả 3 nội dung của file hosts, nói chung nó như sau:

```
192.168.187.128 master
192.168.187.130 slave
192.168.187.131 slave2
```



### Chỉnh file conf mysql *`etc/my.conf`*


```
server-id=3
binlog_do_db=DB2204
binlog_do_db=DB2021
```



- Sau đó restart lại service:

```
systemctl restart mysqld
```
### Tạo thêm database giống trên master
```
create database DB2204;
create database DB2021;
```


![image](https://user-images.githubusercontent.com/83824403/164593092-2e22ba15-8c98-40e0-87b3-431ed5bbb36c.png)

### Change master cho slave2

- ở đây theo những bước cấu hình trước thì lệnh của mình là

```
 CHANGE MASTER TO MASTER_HOST='192.168.187.128',MASTER_USER='slave', MASTER_PASSWORD='Phuc123@', MASTER_LOG_FILE='mysql-bin.000005', MASTER_LOG_POS= 2385;
```
![image](https://user-images.githubusercontent.com/83824403/164593245-33d628d0-3394-454c-9be1-04db511c5c34.png)




- Post và log_file lấy ở đâu, thì là nó ở đây: "Sang master show `show master status\G;`" nó như này:

![image](https://user-images.githubusercontent.com/83824403/164593514-c60c18af-1c59-498f-9ed8-386c98f3829e.png)

- Cuối cùng là start kiểm tra và check status slave

![image](https://user-images.githubusercontent.com/83824403/164594987-54b5a289-e3ea-49f6-a3e8-8e0cd93f496a.png)

*`Xuất hiện 2 trường`*

```
Slave_IO_Running: Yes
Slave_SQL_Running: Yes
``` 
*`là có thể tạm thời yên tâm đi test, nếu lỗi check log và kiểm tra lại`*




### sang master vào thử 1 DB là `DB2204` và tạo 1 table mới để test

![image](https://user-images.githubusercontent.com/83824403/164593626-85dde108-6058-4604-91c2-56e009483e78.png)

-  và cuối cùng sang node slave và slave2 kiểm tra, nếu như 2 ảnh này là thành công:

![image](https://user-images.githubusercontent.com/83824403/164594240-e38381ea-8181-4ae4-a6a7-0a81130f453d.png)


**Slave2**

![image](https://user-images.githubusercontent.com/83824403/164594524-0893f661-e915-408c-9596-174f518e354d.png)



- Ở bài bên chúng ta sẽ config `Master-Master` và thêm `Pacemaker/Corosync` để nó switch tự đông khi 1 DB down


`AUTHOR: DOAN VAN PHUC`

### REFERENCE:
https://www.tecmint.com/mariadb-master-slave-replication-on-centos-rhel-debian/






