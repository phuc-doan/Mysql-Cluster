## Dựa trên bài config đã thực hành trước đó, Sau đây là bài add thêm feature switch tự động

*link bài trước: https://github.com/phuc-doan/Mysql-DB-Replication-Master--Slave/blob/main/config.md*

### Sau khi hoàn thành các bước trên mà không có lỗi lầm gì, ta add thêm tính năng cho cụm này


## Lets go........

## Step1: Cài pcsd trên cả 2 node master/slave

```
yum install pacemaker pcs -y
```

- Khởi động pcs deamon và cho chạy cùng hệ thống:

```
systemctl start pcsd.service
systemctl enable pcsd.service
```

## Step2: Sửa file `hosts` để 2 node có thể thấy nhau

- Thao tác này cũng thực hiện trên cả 2 node


```
vi /etc/hosts
```

- Với máy cá nhân của mình thì ip 2 node lần lượt là dưới đây, và thêm nó vào duói của file kia:

```
192.168.252.151 masterdb
192.168.252.152 slavedb
```

- Sửa hostname với hostnameclt trên node 1 và tương tự với node 2:

```
hostnamectl set-hostname masterdb
```

- Mở firewall `high-availability` và `http` 

![image](https://user-images.githubusercontent.com/83824403/164457250-965ff158-e9ce-4790-8b87-9b2b28ac2868.png)

## Step3: tạo password và user để pcsd authorize giữa 2 node

```
passwd hacluster
```
![image](https://user-images.githubusercontent.com/83824403/164457403-ed80de94-2649-417f-a537-6b06595ee842.png)

**Lưu ý**: user mình chọn là hacluster

## Step4: Xác thực cho nó

```
pcs cluster auth masterdb slavedb
```

- Nhập user và password vừa tạo

## Step5: Tạo tên cluster 


```
pcs cluster setup --name mysql_cluster masterdb slavedb
```
- Tên mình chọn là mysql_cluster



![image](https://user-images.githubusercontent.com/83824403/164457780-4ca39388-cd1b-4365-9b2c-dfa817442bca.png)

- sau đó **`start all node lên`**



```
pcs cluster start --all
```
## Step5: Enable corosyn + pacemaker

```
# systemctl start pcsd
# systemctl enable pcsd
```

![image](https://user-images.githubusercontent.com/83824403/164458284-3b7b28eb-c350-45ec-908e-9f094efc5cc2.png)



- Check lại cluster:



![image](https://user-images.githubusercontent.com/83824403/164458572-e83f880e-7c6f-47ba-adb4-334f42bfaa32.png)



## Step6: Tạo vip cho cluster

- ở đây mifnh để vip là `192.168.252.10`. Nói chung để ip nào cũng được miễn là ko trùng và vẫn thuộc dải 🩹



```
pcs resource create mysql_cluster ocf:heartbeat:IPaddr2 ip=192.168.252.10 cidr_netmask=24 op monitor interval=20s
```




![image](https://user-images.githubusercontent.com/83824403/164458682-fe5afafc-58dd-447a-b7dd-2b3023e9c9aa.png)


## Step 7: Thêm ít thông số cấu hình thời gian check interval


```
pcs resource create hacluster systemd:mysql op monitor interval=10s
```
![image](https://user-images.githubusercontent.com/83824403/164459013-a9b791b0-5663-4752-b973-baa3e17c9440.png)




## Step8: Start lên cho chắc :))

```
pcs cluster start
```

![image](https://user-images.githubusercontent.com/83824403/164459415-0647802d-5c53-4694-b33e-15967f060a58.png)



- Cấu hình bỏ qourum...

```
[root@masterdb ~]# pcs property set stonith-enabled=false
[root@masterdb ~]# pcs property set no-quorum-policy=ignore
[root@masterdb ~]# pcs resource create httpd systemd:httpd op monitor interval=10s
[root@masterdb ~]# pcs status
```

## Step9 : XOng rồi test thôi

- thoát ra và test ssh vào địa chỉ VIP `192.168.252.10`

![image](https://user-images.githubusercontent.com/83824403/164459816-ea5b56b5-77b0-4497-bf63-6c8168293ab8.png)

- nó ra đc node master thế này là thành công!


## Step 10: off thử node master xem pacemaker nó có auto nhảy sang slave không

![image](https://user-images.githubusercontent.com/83824403/164460031-4dbb4367-681b-4e26-81b5-74b21c8b17ba.png)

- Vào ssh lại, nếu nó auto nhảy sang node slave là thành công <3


`AUTHOR: ĐOÀN VĂN PHÚC`




