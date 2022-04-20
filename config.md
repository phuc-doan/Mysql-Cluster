## Bài Lab cài trên 2 server có IP lần lượt là 
- 192.168.252.151 (slave)
- 192.168.252.152 (master)

### let get started!


- Trên ALL node cài Mysql/Mariadb (Nếu lab thì nên dùng mariadb vì password ko require theo tiêu chuẩn, Nếu mô hình thực thì chọn mysql vì bảo mật hơn - *ý kiến cá nhân*)

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

## Step2: Chúng ta cần cấu hình điều chỉnh cơ sở dữ liệu /etc/my.cnf cho Node 1 và Node 2 để kích hoạt binary logging (binary logging có thể được coi là log của tất cả các câu lệnh SQL mà nó nhận được)


- Trên node master:
