## HA proxy là gì

HAProxy (High Availability Proxy) là một công cụ mã nguồn mở ứng dụng cho giải pháp cần bằng tải TCP và HTTP. 

Người dùng có thể sử dụng HAProxy để cải thiện suất hoàn thiện của các trang web và ứng dụng bằng cách phân tán khối lượng công việc của chúng trên nhiều máy chủ. 

Cải thiện hiệu suất bao gồm giảm phản hồi thời gian và tăng thông lượng. 

HAProxy cũng được sử dụng trong các hệ thống lớn có lưu lượng truy cập cao như GitHub, Twitter, Reddit, Bitbucket, Stack Overflow,…


## Feature:

- HAProxy bao gồm các tính năng như:

Hỗ trợ cân bằng tải ở lớp 4 và lớp 7 (tương ứng với TCP và HTTP).

Support HTTP protocol, HTTP / 2, gRPC, FastCGI.

Chấm dứt SSL / TLS.

Lưu trữ chứng chỉ SSL động.

Chuyển đổi nội dung và kiểm tra.

Ủy quyền minh bạch.

Ghi nhật ký chi tiết.

CLI.

Xác thực HTTP.

Đa luồng.

Rewrite URL.

Kiểm tra sức khỏe nâng cao.

Giới hạn tần số kết nối.


## Các thuật ngữ trong HAProxy

- Access Control List (ACL)


Trong cân bằng tải, Acess Control List (ACL) được sử dụng để kiểm tra điều kiện và thực hiện một hành động (VD: chọn một server hay chặn một request) dựa trên kết quả của việc kiểm tra đó. Khi sử dụng ACL cho phép bạn tạo một môi trường có khả năng chuyển tiếp các request linh hoạt dựa trên các yếu tố khác nhau.

- Ví dụ một ACL:
- 
```
acl url_blog        src         /something
Bash
```

Trong đó: ACL này sử dụng cho các request có chứa /something.

- Backend


Backend là một tập các server mà HAProxy có thể chuyển tiếp các request tới. Backend được cấu hình trong mục backend trong file configuration của HAProxy. Backend có thể cài đặt bằng cách:

Đặt thuật toán cân bằng tải (round-robin, least-connection,…)

Danh sách các máy chủ và port có thể nhận request từ HAProxy.

Một backend có thể chứa một hoặc nhiều server, về cơ bản thì càng nhiều server thì khả năng chịu tải và performace của hệ thống càng tăng. 

HAProxy cho phép một server backup chuyên dụng, được sử dụng khi các server offline.


- Ví dụ về cấu hình backend:

```
backend web-backend
    balance leastconn
    mode http
    server backend-1 web-backend-1.example.com check
    server backend-2 web-backend-2.example.com check
    server backend-3 backup-backend.example.com check backup
    
backend forum
    balance leastconn
    server forum-1 forum-1.example.com check
    server forum-2 forum-2.example.com check
    server forum-3 backup-forum.example.com check backup
```

- Trong đó:


Dòng blance leaseconn chỉ ra thuật toán cân bằng tải là chọn các server có ít kết nối đến nó nhất.

Dòng mode http chỉ ra rằng các proxy sẽ chỉ cân bằng cho các kết nối tại tầng 7 của Internet Layer.


- Frontend


Fontend được sử dụng để định nghĩa cách mà các request điều hướng cho backend. Và được định nghĩa trong mục fontend của HAProxy configuration. Các cấu hình cho fontend gồm:


Một địa chỉ IP và port.

Các ACL do người dùng định nghĩa.

Backend được sử dụng để nhận các request.

Ví dụ về cấu hình fontend:


```
frontend web
  bind 0.0.0.0
  default_backend web-backend

frontend forum
  bind 0.0.0.0:8080
  default_backend forum
  
```


## Mô tả hoạt động HAproxy


- Bước 1: Request từ phía USER đến VIP của HAProxy

- Bước 2: Request từ phía USER được HAProxy tiếp nhận và chuyển tới các Webserver

- Bước 3: Các Webserver xử lý và response lại HAProxy

-Bước 4: HAProxy tiếp nhận các response và gửi lại cho USER bằng VIP

![image](https://user-images.githubusercontent.com/83824403/164873419-5b56ea38-119d-4ddc-940b-d1090dd1d613.png)


## Mô hình

![image](https://user-images.githubusercontent.com/83824403/164873424-3ed2861b-c1f3-4957-91b5-cb37c06ed8c9.png)

