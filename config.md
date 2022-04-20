## Bài Lab cài trên 2 server có IP lần lượt là 
- 192.168.252.151 (slave)
- 192.168.252.152 (master)

### Let get started!


- Trên ALL node cài Mysql/Mariadb (Nếu lab thì nên dùng mariadb vì password ko require theo tiêu chuẩn, Nếu mô hình thực thì chọn mysql vì bảo mật hơn - **`ý kiến cá nhân`**)

 ## Step1: Cấu hình replication MariaDB Master/Master

- Cài đặt MariaDB

```
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
```
-   mysql> show master status;

```





















