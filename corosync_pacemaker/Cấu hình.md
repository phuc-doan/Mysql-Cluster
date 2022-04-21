## Dựa trên bài config master/slave đã thực hành trước đó, Sau đây là bài add thêm feature switch tự động

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






