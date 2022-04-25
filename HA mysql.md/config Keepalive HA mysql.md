## Xây dựng High Available cho MySQL Server với HAproxy và Keepalived trên Ubuntu

### Mô hình lab logic :

![image](https://user-images.githubusercontent.com/83824403/164588964-c4be66a6-78a8-4acd-971f-9de264c25da6.png)


- Trên thực tế, mình chỉ cần dựng 2 máy chủ, với mỗi máy được cài đặt cả 3 dịch vụ haproxy , keepalived và mysql-server. 
- vì haproxy được cài đặt cùng với mysql-server nên để không xảy ra conflict thì chúng ta có 2 cách xử lý như sau : **`1 - Giữ nguyên port mặc định của mysql và cho haproxy listen trên một port khác. 2 - Thay đổi port mặc định của mysql.`**


- Ở đây mình sẽ chọn cách thứ 2. Mình sẽ đổi port mặc định của MySQL.

- Mô hình thực tế sẽ như thế này :

**– MySQL 1**


```
Hostname: mysql1
OS: Ubuntu server 16.04
Service : Keepalived + HAproxy + mysql-server
Private IP: 172.17.3.111
```

**– MySQL 2**

```
Hostname: mysql2
OS: Ubuntu server 16.04
Service : Keepalived + HAproxy + mysql-server
Private IP: 172.17.3.112
```


### Chuẩn bị các server :


- Sau khi cài đặt OS thì chúng ta lần lượt cài đặt các dịch vụ cần thiết lên mỗi server.

```
sudo apt-get keepalived 
sudo apt-get install haproxy
sudo apt-get install mysql-server        #Đặt pass gì thì nhớ nhé, coi chừng quên !
...
sudo apt-get update
```

### Sau khi cài đặt và update xong xuôi, chúng ta bắt đầu config Replication cho MySQL trước và đồng thời tiến hành custom port listen của MySQL luôn thể. 

## Config Replication MySQL Master - Master


- Replication trong MySQL các bạn có thể hiểu nôm na là kỹ thuật tạo một bản sao từ một MySQL server được chỉ định. Trong đó đối tượng được chỉ định có thể là một table , một database hoặc cả một server MySQL.
-  Trong kỹ thuật này, ta sẽ có tối thiểu là 2 server MySQL. Với server gốc được gọi là Master , và bản clone của nó được gọi là Slave. Slave server sẽ luôn cập nhật những thay đổi từ Master. 
-  ( Đại khái thì thằng Slave sẽ là 1 tấm gương phản chiếu của thằng Master, Master làm gì thì Slave làm y chang thế. Để hiểu rõ hơn thì các bạn google với từ khóa replication mysql nhá)

## Và cụ thể trong bài viết này thì mình sẽ làm mô hình replication Master - Master.
![image](https://user-images.githubusercontent.com/83824403/164589023-a6644141-2539-40c6-92b3-785dd1dc07a1.png)



## Bước 1 : Config Master 1 - Slave 1
Trên cả 2 server ta tiến hành edit file **/etc/mysql/mysql.conf.d/mysqld.cnf** . Tìm và comment hoặc **xóa dòng bind-address = 127.0.0.1**

```
#bind-address = 127.0.0.1
```

- Lý do làm việc này là vì mặc định thằng mysql chỉ cho login local. Nên bỏ nó đi, hoặc muốn login từ đâu thì chỉ định.

- Cũng trên file **/etc/mysql/mysql.conf.d/mysqld.cnf** Ta cũng tìm dòng **port=3306** và thay đổi theo ý thích nhé. Cụ thể ở đây mình sẽ đổi port MySQL 1 thành 3307 và MySQL 2 thành 3308.

```
port = 3307      #đổi trên MySQL 1
port = 3308      #đổi trên MySQL 2
```


- Xong rồi save lại và thoát ra. Tiếp tục chỉnh sửa file **/etc/mysql/my.cnf** - đây là file config chính của mysql, tất nhiên bạn cũng có thể config bằng file trên, nhưng file kia nhìn có vẻ hơi rối, nhất là với newbie như mình nên thôi mình config file my.cnf cho chắc ăn 😃)

### Trên MySQL1 - Master 1 ( sẽ là Slave 2):
- Mở file /etc/mysql/my.cnf và điền vào như sau :

```
server-id = 1
log_bin = /var/log/mysql/mysql-bin.log
```

- **Save lại và service mysql restart.**
-  Ý nghĩa của khai báo trên chỉ đơn giản là đặt ID cho server mysql của bạn và khai báo lưu trữ binlog.
-   Cái này liên quan đến cách thức hoạt động của kỹ thuật Replication. Đơn giản có thể hiểu như thế này . 
-   Thằng Master sẽ lưu mọi thay đổi database của nó vào một file binlog và để đó (có thể cấu hình được các thông số như kích thước tối đa, thời gian lưu trên server). Thằng Slave sẽ "đi qua" và lấy binlog từ Master đem về, đọc và ghi vào replaylog. 
-   Và cuối cùng nó sẽ đọc replaylog và cập nhật các event trong đó => Replication xong.

**`Config như trên tuy trông hơi nhà quê nhưng vậy là đủ để hoàn thành Replicate. Ngoài ra còn có thể khai báo thêm các loại log hoặc custom hoạt động của mysql. Bạn nào hiểu biết chuyên sâu hơn thì bổ sung giúp mình cho hoàn thiện và tối ưu hơn 😃`**

## Tiếp theo là tạo một user trên Master1, Slave1 sẽ dùng user này để đi vào master1 và lụm binlog đem về.


```
mysql1@ubuntu:~$  mysql -u root -p
Password:
mysql> CREATE USER 'Username'@'IP_Slave_Server' IDENTIFIED BY 'password';
Query OK, 0 rows affected (0.00 sec)
```


- Tạo user cho phép truy cập từ Salve server. Username và password các bạn thích điền gì thì điền, còn IP_Slave_Server thì điền IP của thằng Slave vào. Sau khi tạo xong thì cấp quyền Replication cho nó.
```
mysql> GRANT REPLICATION SLAVE ON *.* TO 'Username'@'IP_Slave_Server';
```

- Nhớ điền username và IP_Slave_Server giống khởi tạo ở trên nhé. Không thì chả biết phân quyền cho thằng nào đâu 😃)

- Cuối cùng là thông tin MASTER STATUS , bạn phải ghi nhớ để khai báo vào Slave server.

```
mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000005 |      154 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)

```



- Bạn cần chú ý 2 thông tin, File name là mysql-bin.000005 và Position 154 để khai báo trên Slave. 
- 2 thông tin còn lại là Binlog_Do_DB và Binlog_Ignore_DB để chỉ định DB nào được replicate và DB nào không được replicate.
-  Nếu có khai báo ở my.cnf thì sẽ hiển thị ra. Các bạn có thể thử. Cuối cùng là Executed_Gtid_Set , mình cũng chả biết nó để làm gì nên thôi cứ kệ nó đi đã. Hình như nó thuộc dạng thông tin khai thì có, không khai thì thôi á. 
- Vậy là đã hoàn tất các bước chuẩn bị trên Master. Bây giờ chỉ việc khai báo trên Slave là ta đã hoàn thành Master 1 - Slave 1

### Trên MySQL 2 -Slave 1 ( sẽ là Master 2) :

- Nhớ comment hoặc xóa dòng bind-address = 127.0.0.1 trong /etc/mysql/mysql.conf.d/mysqld.cnf nhá. Quên làm ở trên thì bây giờ làm.

- Mở file /etc/mysql/my.cnf và điền vào như sau :

```
server-id = 2
log_bin = /var/log/mysql/mysql-bin.log
```

-**Save lại và service mysql restart.**

## Chuẩn bị xong thì vào khai báo Slave 1 để nó có thể replicate data từ Master 1


```
$ mysql -u root -p
Password:

mysql> STOP SLAVE;                                  #Tắt Slave trước khi khai báo
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> CHANGE MASTER TO
    -> MASTER_HOST='IP_Master_Server',                 #Khai ip master1 server
    -> MASTER_USER='Username',                               #Username dành cho Slave 1 tạo ở trên
    -> MASTER_PASSWORD='password',                     #Password của nó, tất nhiên rồi @@
    -> MASTER_PORT=3307,                                          #Port của Master_Server nhé, vì đã đổi rồi nên là 3307. Nếu chưa đổi thì ko cần khai chỗ này. 
    -> MASTER_LOG_FILE='mysql-bin.000005',           #Thông tin file binlog lụm được từ Master 1 status
    -> MASTER_LOG_POS=154;                                      # Thông tin position tương ứng
Query OK, 0 rows affected, 2 warnings (0.01 sec)

mysql> START SLAVE;                                                          #Sau khi khai báo xong thì khởi động lại Slave
Query OK, 0 rows affected (0.00 sec)
```



### Sau khi start slave lại trên Slave 1 thì chúng ta có thể check status của nó



```
mysql> show slave status \G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 172.17.3.111
                  Master_User: repliuser
                  Master_Port: 3307
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000005
          Read_Master_Log_Pos: 154
               Relay_Log_File: ubuntu-relay-bin.000005
                Relay_Log_Pos: 367
        Relay_Master_Log_File: mysql-bin.000005
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 0
                   Last_Error: 
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 154
              Relay_Log_Space: 788
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 0
               Last_SQL_Error: 
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 1
                  Master_UUID: b05dfbb5-1463-11e7-8601-000c29042d55
             Master_Info_File: /var/lib/mysql/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind: 
      Last_IO_Error_Timestamp: 
     Last_SQL_Error_Timestamp: 
               Master_SSL_Crl: 
           Master_SSL_Crlpath: 
           Retrieved_Gtid_Set: 
            Executed_Gtid_Set: 
                Auto_Position: 0
         Replicate_Rewrite_DB: 
                 Channel_Name: 
           Master_TLS_Version: 
1 row in set (0.00 sec)
```





#### Check xem các khai báo mình khai đúng chưa, nếu chưa thì stop slave; , reset slave; và khai báo lại nhá. 

#### Cuối cùng nhìn cái dòng Slave_SQL_Running_State : Slave has read all relay log; waiting for more updates là biết nó đang chạy. Nếu không chạy thì nó sẽ nói là ko chạy 😄

- Vậy là xong Master 1 - Slave 1 . Tiếp đến làm Slave 2 - Master 2

## Bước 2 : Config Master 2 - Slave 2



- Các khai báo trong các file **`/etc/mysql/mysql.conf.d/mysqld.cnf`** và **`/etc/mysql/my.cnf`** đã làm ở trên rồi thì không cần làm lại. 
- Bây giờ mình chỉ việc tạo User và cấp quyền Replication trên Slave 1 để Master 1 có thể truy cập vào. Vậy là Slave 1 cũng đồng thời sẽ là Master của thằng Master 1

### Trên máy MySQL2 - Slave 1 & Master 2:
- Nhìn có vẻ lằng nhằng khó hiểu, nhưng thực tế là làm y chang ở trên, chỉ đổi vai trò của 2 thằng MySQL1 và MySQL2 với nhau thôi. Tạo Username và gán quyền Replication


```
$ mysql -u root -p
Password:

mysql> CREATE USER 'Username'@'IP_Slave_Server' IDENTIFIED BY 'password';
Query OK, 0 rows affected (0.00 sec)

mysql> GRANT REPLICATION SLAVE ON *.* TO 'Username'@'IP_Slave_Server';
Query OK, 0 rows affected (0.00 sec)
```

### Lấy thông tin của Master



```
mysql> SHOW MASTER STATUS;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000001 |      792 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
```


### Trên máy MySQL1 - Master 1 & Slave 2:
- Khai báo cho Slave




```
$ mysql -u root -p
Password:

mysql> STOP SLAVE;                                                            #Tắt Slave trước khi khai báo
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> CHANGE MASTER TO
    -> MASTER_HOST='IP_Master_Server',                          #Khai ip master2 server
    -> MASTER_USER='Username',                                        #Username dành cho Slave 2 tạo ở trên
    -> MASTER_PASSWORD='password', 
    -> MASTER_PORT=3308,
    -> MASTER_LOG_FILE='mysql-bin.000005',                   #Thông tin file binlog lụm được từ Master 2 status
    -> MASTER_LOG_POS=154;                                              # Thông tin position tương ứng
Query OK, 0 rows affected, 2 warnings (0.01 sec)

mysql> START SLAVE;                                                           #Sau khi khai báo xong thì khởi động lại Slave
Query OK, 0 rows affected (0.00 sec)
```

### Check status của Slave 2.

```
mysql> show slave status \G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 172.17.3.112
                  Master_User: repliuser
                  Master_Port: 3308
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000001
          Read_Master_Log_Pos: 792
               Relay_Log_File: ubuntu-relay-bin.000005
                Relay_Log_Pos: 320
        Relay_Master_Log_File: mysql-bin.000001
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
......
Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
......
1 row in set (0.01 sec)
```


### Vậy là xong. Kế tiếp test thực tế xem như nào.

## Bước 3 : Test Replication Master - Master


- Theo như đúng config, thì cả 2 máy Mysql1 và Mysql2 sẽ có thể replicate dữ liệu của nhau.
-  Hay nói theo kiểu dân gian là thằng Mysql1 làm gì thì thằng Mysql2 làm y chang và ngược lại. Bất cứ thay đổi nào ( tạo , thêm , sửa , xóa ....) trên mysql1 đều sẽ được cập nhật lên mysql2 và ngược lại.

### Đầu tiên, cùng show databases; trên cả 2 máy. Mặc định mới cài mysql thì chưa có vẹo gì, cả 2 thằng đều sẽ như này :


```
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
4 rows in set (0.09 sec)
```

### Vào máy Mysql1 thử tạo 1 database

```
mysql> create database testrepli;
Query OK, 1 row affected (0.00 sec)
```
- và check Processlist trên máy Mysql1 xem nó đã làm gì nào




```
mysql> SHOW PROCESSLIST ;
SHOW PROCESSLIST;
+----+-------------+--------------------+------+-------------+------+---------------------------------------------------------------+------------------+
| Id | User        | Host               | db   | Command     | Time | State                                                         | Info             |
+----+-------------+--------------------+------+-------------+------+---------------------------------------------------------------+------------------+
|  1 | system user |                    | NULL | Connect     | 4928 | Slave has read all relay log; waiting for more updates        | NULL             |
|  2 | system user |                    | NULL | Connect     | 4928 | Waiting for master to send event                              | NULL             |
|  6 | repliuser   | 172.17.3.112:43906 | NULL | Binlog Dump | 4909 | Master has sent all binlog to slave; waiting for more updates | NULL             |
|  7 | root        | localhost          | NULL | Query       |    0 | starting                                                      | SHOW PROCESSLIST |
+----+-------------+--------------------+------+-------------+------+---------------------------------------------------------------+------------------+
4 rows in set (0.00 sec)

```

#### Để ý dòng có ID số 6 State nó ghi là Master has sent all binlog to slave; waiting for more updates . Nó nói đã sent binlog đến thằng slave. Giờ qua slave xem nó có nói xạo mình không.

### Vào máy Mysql2 show database thử xem sao



```
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| testrepli          |
+--------------------+
5 rows in set (0.00 sec)
```

### Ok, data testrepli đã xuất hiện ở Mysql2 , thằng Master kia nó đã không xạo mình =)). Nhưng ở chiều ngược lại thì sao ?? Test tiếp.

## Bây giờ trên máy Mysql2 mình sẽ tạo 1 bảng trong database vừa tạo.

```
mysql> use testrepli
Database changed
mysql> create table TenTuoi(ten varchar(255),tuoi int(10));
Query OK, 0 rows affected (0.04 sec)
```




- Check processlist




```
mysql> show processlist;
+----+-------------+--------------------+-----------+-------------+-------+---------------------------------------------------------------+------------------+
| Id | User        | Host               | db        | Command     | Time  | State                                                         | Info             |
+----+-------------+--------------------+-----------+-------------+-------+---------------------------------------------------------------+------------------+
|  5 | system user |                    | NULL      | Connect     | 62749 | Waiting for master to send event                              | NULL             |
|  6 | system user |                    | NULL      | Connect     |   161 | Slave has read all relay log; waiting for more updates        | NULL             |
| 10 | repliuser   | 172.17.3.111:59418 | NULL      | Binlog Dump | 49052 | Master has sent all binlog to slave; waiting for more updates | NULL             |
| 11 | root        | localhost          | testrepli | Query       |     0 | starting                                                      | show processlist |
+----+-------------+--------------------+-----------+-------------+-------+---------------------------------------------------------------+------------------+
4 rows in set (0.00 sec)
```


- Nó cũng kêu đã sent cho thằng slave ở process ID=10. Qua bên kia check coi sao.

#### Vào máy Mysql1



```
mysql> show tables in testrepli;
+---------------------+
| Tables_in_testrepli |
+---------------------+
| TenTuoi             |
+---------------------+
1 row in set (0.00 sec)

```


- Đã xuất hiện bảng TenTuoi ở trong database testrepli . 
- Mình tạo trên mysql2 mà nó update luôn vào mysql1 vậy là đã thành công config Replication Master - Master. 
- Nhưng mới chỉ xong quá trình đồng bộ data. Giờ ta tạo thêm 1 user cho phép login từ ngoài vào để tý test loadbalancer luôn nhá.



```
GRANT ALL PRIVILEGES ON *.* TO 'super'@'%' IDENTIFIED BY '12345' WITH GRANT OPTION;
```


- Chỉ cần tạo trên 1 thằng thôi nhá, thằng kia tự đồng bộ về ^^. **Kế đến sẽ là xây dựng loadbalancer.**

## Xây dựng Loadbalancer cho các server MySQL bằng HAproxy:


-  tham khảo https://viblo.asia/hoangminhung/posts/Az45bpeOZxY

#### Lần lượt trên cả 2 server , tiến hành chỉnh sửa file config của HAproxy

**sudo nano /etc/haproxy/haproxy.cfg** Mở file lên và điền nội dung vào như sau :


```
global
        maxconn 256
        daemon
    defaults
        log global
        retries 2
        timeout connect 5000ms
        timeout client  50000ms
        timeout server  50000ms

    listen mysql-cluster
        bind 172.17.3.200:3306    #Cái này là Virtual IP , lát sẽ khai báo cùng Keepalived nhé.
        mode tcp
        balance roundrobin
            server mysql1 172.17.3.111:3307 check
            server mysql2 172.17.3.112:3308 check

```


- Sau khi xong thì save lại và sudo service haproxy restart nhé. Tiếp đến sẽ là Keepalived.

### Config Keepalived:



- Keepalived sẽ được dùng để tạo một Virtual IP để cho user truy cập vào. 
- Qua đó cung cấp tính năng fail-over cho HAproxy. 1 trong 2 thằng có tử ẹo cũng chả sao (lol). có 1 bài viết về Keepalived ở link này https://viblo.asia/hoangminhung/posts/jOxKdqWlzdm **Các bạn vào xem để hiểu rõ hơn về cách nó hoạt động**.

- Trước khi định nghĩa Vir_IP, chúng ta phải thực hiện dòng lệnh sau trong /etc/sysctl.conf. Mở nó lên và thêm vào dòng sau



```
net.ipv4.ip_nonlocal_bind=1
```

### Save lại và thực thi dòng trên bằng lệnh sau :


```
sudo sysctl -p
```


- Lần lượt thực hiện việc này trên cả 2 máy chủ nhé. Sau khi xong xuôi, ta tiến hành config keepalived

### Trên MySQL1
- Mở file config keepalived (chưa có thì tạo mới nhé) sudo nano /etc/keepalived/keepalived.conf Điền vào nội dung sau :




```


global_defs {
  router_id test1                       #khai báo route_id của keepalived
}
vrrp_script chk_haproxy {
  script "killall -0 haproxy"
  interval 2
  weight 2
}
vrrp_instance VI_1 {
  virtual_router_id 51
  advert_int 1
  priority 100
  state MASTER
  interface ens33                     #thông tin tên interface của server, bạn dùng lệnh `ifconfig` để xem và điền cho đúng
  virtual_ipaddress {
    172.17.3.200 dev ens33     #Khai báo Virtual IP cho interface tương ứng và dùng IP này listen trên HAproxy
  }
 authentication {
     auth_type PASS
     auth_pass 123456              #Password này phải khai báo giống nhau giữa các server keepalived
     }
  track_script {
    chk_haproxy
  }
}
```

### Save lại và sudo service keepalived start

### Trên máy MySQL2
- Tương tự như trên nhưng phần khai báo khác 1 tý.




```
global_defs {
  router_id test2
}
vrrp_script chk_haproxy {
  script "killall -0 haproxy"
  interval 2
  weight 2
}
vrrp_instance VI_1 {
  virtual_router_id 51
  advert_int 1
  priority 99                   #priority khởi tạo sẽ thấp hơn máy 1
  state BACKUP           #Trạng thái khởi tạo sẽ là BACKUP
  interface ens33
  virtual_ipaddress {
    172.17.3.200 dev ens33
  }
authentication {
        auth_type PASS
        auth_pass 123456
        }
track_script {
    chk_haproxy
    }
  }
```

- Save lại và sudo service keepalived start

## Test



- Sau khi hoàn thành các bước config, chúng ta sẽ tiến hành test thử xem nó hoạt động thế nào.
-  Theo như mô hình mình đưa ra thì nó sẽ hoạt động như sau : 
-  1 - User sẽ không truy cập trực tiếp vào MySQL server mà sẽ thông qua một Vir_IP. 
-  2 - Vir_IP sẽ đưa User đi đến Loadbalancer nào đang giữ Vir_IP . Trường hợp Loadbalancer này chết, thì Vir_IP sẽ tự động được chuyển cho Loadbalancer còn lại. 
-  3 - Các loadbalancer sẽ chuyển các request đến MySQL theo thuật toán Roundrobin.

### Dùng một máy khác cài đặt mysql-client và tiến hành connect vào. Sử dụng user super:12345 mà khi nãy chúng ta đã tạo nhé.


```
mysql -h 172.17.3.200 -u super -p
Password:
mysql>
```


=> - Đã connect được. Chúng ta thử xem đang connect vào MySQL 1 hay 2 nhé. Chú ý thông số server_ID đã khai báo ở trên. ID=1 tương ứng với MySQL1 và ID=2 tương ứng MySQL2.



```
mysql> show variables like 'server_id';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 1     |
+---------------+-------+
1 row in set (0.01 sec)
```


- Đang connect vào MySQL 1. Theo như thuật toán Roundrobin thì request tiếp theo sẽ connect vào MySQL2. Thoát ra và connect lại xem nào.


```
minhhung@ubuntu:~$ mysql -h 172.17.3.200 -u super -p
Password:
mysql>
mysql> show variables like 'server_id';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 2     |
+---------------+-------+
1 row in set (0.01 sec)
```


## Ok, đúng là đang connect vào MySQL 2. Vậy là mô hình đã hoạt động .
