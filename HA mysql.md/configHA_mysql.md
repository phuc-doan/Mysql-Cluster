## Config HA for mysql

**`Bug and resolve by: https://stackoverflow.com/questions/62808822/2013-lost-connection-to-mysql-server-at-reading-initial-communication-packet`**

## Now, Let get started!............




# Cấu hình :



## - Chuẩn bị các node database server :


- Tiến hành cài đặt MySQL lần lượt trên các node :

```
rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2022
rpm -Uvh https://repo.mysql.com/mysql57-community-release-el7-11.noarch.rpm
yum install -y mariadb-server mariadb
```

## Chỉnh sửa file mysql.cnf 

- tiến hành edit 2 dòng như sau :





![image](https://user-images.githubusercontent.com/83824403/164868465-9c359cfb-bfa1-4760-89ee-92eddffd7b2a.png)




*` Thực hiện trên cả 2 máy MySQL server.`* 
 
 
 
 
- Bỏ Comment dòng server-id = 1 và lần lượt cho cái giá trị ID mà mình mong muốn trên mỗi server MySQL. Ở đây mình cho ID lần lượt là 1 và 2 tương ứng với 2 máy M1 và M2 . 




```
sudo service mysql restart
```

### Tiếp theo là tiến hành tạo User để connect với HAproxy.


- Trên mỗi node database cần tạo 2 user. 1 dùng cho HAproxy check status và 1 dùng connect sử dụng. Username và password nên tạo như nhau ở cả 2 server vì trên thực tế sử dụng thì 2 node database này sẽ đồng bộ với nhau.

- User thứ nhất



```
mysql > Inser into mysql.user (Host,User,ssl_cipher,x509_issuer,x509_subject) values ('IP_HAproxy','USER','abc','abc','abc');


```

##### *Lưu ý : trường Host thì các bạn điền IP của HAproxy vào nhé,còn user thì thích điền gì thì điền. Ví dụ : mysql > Inser into mysql.user (Host,User,ssl_cipher,x509_issuer,x509_subject) values ('192.168.187.133','haproxy_checkstatus','abc','abc','abc');*


- User thứ hai Tạo và gắn full quyền luôn.

```
mysql > GRANT ALL PRIVILEGES ON *.* TO 'USER'@'IP_HAproxy' IDENTIFIED BY 'PASSWORD' WITH GRANT OPTION; 
```


Lưu ý Các trường USER ,IP_HAproxy và PASSWORD  Ví dụ: `mysql > GRANT ALL PRIVILEGES ON *.* TO 'super'@'172.17.3.102' IDENTIFIED BY '12345' WITH GRANT OPTION;` 


- Sau khi tạo xong thì chạy `mysql > FLUSH PRIVILEGES;` 

### Lần lượt làm tương tự trên cả 2 máy MySQL Sau khi chuẩn bị xong, các bạn có thể check kết quả các bước trên như sau :

- Xem các user đã tạo :



```

mysql> select user from mysql.user;
+---------------------+
| user                |
+---------------------+
| haproxy_checkstatus |
| USER                |
| haproxy             |
| haproxy_check       |
| haproxy_root        |
| super               |
| mysql.session       |
| mysql.sys           |
| root                |
+---------------------+
9 rows in set (0.00 sec)


```
![image](https://user-images.githubusercontent.com/83824403/164869011-4dc3dc1f-6428-471b-92a7-5db78d522c7b.png)




Xem server_ID đã gán

mysql> show variables like 'server_id';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 1     |
+---------------+-------+


![image](https://user-images.githubusercontent.com/83824403/164869052-1b091e9c-b70d-4bda-b763-f2c776edfd4b.png)


### Config HAproxy :

- Install HAproxy : 


```
yum -y install haproxy Khởi động HAproxy
```


Sua file config haproxy

```
sudo nano /etc/haproxy/haproxy.cfg Nội dung file config như sau :
```

- dai khai no nhu sau:



```
global
        log 127.0.0.1 local0 notice
        user haproxy
        group haproxy
        maxconn 256
        daemon
    defaults
        log global
        retries 2
        timeout connect 5000ms
        timeout client  50000ms
        timeout server  50000ms

    listen mysql-cluster
        bind 192.168.187.133:3306
        mode tcp
#        option mysql-check user haproxy_checkstatus

        balance roundrobin
            server M1 192.168.187.134:3306 check
            server M2 192.168.187.132:3306 check

```

![image](https://user-images.githubusercontent.com/83824403/164869545-1a321766-4f38-4ee1-a395-e2b8826883f1.png)


#### đoạn trên mình cũng viết là có bug khi đặt config của haproxy theo như default `ERROR 2013 (HY000): Lost connection to MySQL server at 'reading initial communication packet', system error: 0`

- Giải pháp đoạn này check trên stackoverflow thì là comment dòng `# option mysql-check user haproxy_checkstatus`. Đại loại như này:

![image](https://user-images.githubusercontent.com/83824403/164871387-84dbd080-2971-48df-ad8d-27d3945d8438.png)

=> **CUối cùng nó resolve được, còn tại sao thì vẫn đang tìm hiểu ^^**


- Các cần lưu ý phần khai báo này nhé :


```
listen mysql-cluster
        bind 192.168.187.133:3306
        mode tcp
        #option mysql-check user haproxy_checkstatus
```


- Khai báo cho HAproxy listen mysql-cluster ( cụm database server) trên ip 192.168.187.133:3306 của HAproxy port mặc định của MySQL là 3306 

- Khai báo này giúp cho chúng ta có thể truy cập đến các server MySQL thông qua HAproxy 

- Tùy chọn option hỗ trợ chúng ta theo dõi tình trạng của các node tham gia vào cùng một Loadbalancing của HAproxy 
 
- Với MySQL, HAproxy có hỗ trợ tùy chọn mysql-check với khai báo user tương ứng đã tạo ở trên



### Tiếp theo là khai báo cân bằng tải cho các server MySQL.

```
        balance roundrobin
            server M1 192.168.187.134:3306 check
            server M2 192.168.187.132:3306 check
```

##### *Thuật toán HA đc chọn là roundrobin, ngoài ra còn khá nhiều option:*





- Round Robin
- Weighted Round Robin
- Dynamic Round Robin
- Fastest
- Least Connections 



## Test :

- Sau khi đã hoàn thành các bước trên. Các bạn tiến hành test như sau : dùng một máy khác cài đặt mysql-client và connect vào cụm database thông qua HAproxy.

```
mysql -h 192.168.187.133-u super -p
```

![image](https://user-images.githubusercontent.com/83824403/164872685-432b5f4e-6fd8-4842-a129-11feec9da2c4.png)

=>>> **Connect thành công, tiến hành kiểm tra xem đang connect vào Node data nào:**



```
mysql> show variables like 'server_id';
ERROR 2006 (HY000): MySQL server has gone away
No connection. Trying to reconnect...
Connection id:    3348
Current database: *** NONE ***

+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 1     |
+---------------+-------+
1 row in set (0.01 sec)

```


- Đang connect vào Node mysql-1 với ID khi nãy gán là 1. Giờ thoát ra và connect lại và kiểm tra để thấy sự khác biệt .



```
mysql> show variables like 'server_id';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 2     |
+---------------+-------+
1 row in set (0.01 sec)
```



#### - Đại khái nó như vậy:


![image](https://user-images.githubusercontent.com/83824403/164872711-8e320c20-f555-4125-8da5-d29b0854a195.png)



- Sau khi thoát ra và connect lại thì chúng ta đã connect được vào Node M2 với ID là 2.
- Đây là roundrobin nó sẽ call tuần tự sang các node db setup



#### Kết luận:

**Trong phần này chúng ta đã xây dựng được một loadbalancer có nhiệm vụ cân bằng tải cho các Node database. Ở phần tiếp theo của bài viết, mình sẽ tiếp tục thực hiện việc đồng bộ các node data với nhau và tích hợp luôn cả Keepalived để tăng tính chịu lỗi cho cả mô hình 😃**

