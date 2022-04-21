## Cài thêm feature Switch tự động bằng corosync và pacemaker


![image](https://user-images.githubusercontent.com/83824403/164454081-f2b77665-77ad-4e3f-a336-931edbc85b84.png)


![image](https://user-images.githubusercontent.com/83824403/164454129-6e2ac754-c301-4b81-912c-ca8c912e0c6a.png)


## Các kiểu Cluster hỗ trợ

**Pacemaker hỗ trợ bất kể các thiết Cluster đáp ứng theo thiết kế đề ra, bao gồm:**

- Active / Active
- Active / Passive
- N + 1
- N + M
- N to 1
- N to M

## Cách thức hoạt động


- **Cân bằng tải của cụm (Load Balancing):** Các node bên trong cluster hoạt động song song, chia sẻ các tác vụ để năng cao hiệu năng.

- **Tính sẵn sàng cao (High availability):** Các tài nguyên bên trong cluster luôn sẵn sàng xử lý yêu cầu, ngay cả khi có vấn đề xảy ra với các thành phần bên trong (hardware, software).


- **Khả năng mở rộng (scalability):** Khi tài nguyên có thể sử dụng của hệ thống tới giới hạn, ta có thể dễ dàng bổ sung thêm tài nguyên vào cluster bằng các bổ sung thêm các node.

- **Độ tin cậy (reliability):** Hệ thống Cluster giảm thiểu tần số lỗi có thể xảy ra, giảm thiểu các vấn đề dẫn tới ngừng hoạt động của hệ thống.


### Các thuật ngữ quan trọng

- `Cluster`: Nhóm các server dành riêng để giải quyết 1 vấn đề, có khả năng kết nối, sản sẻ các tác vụ.
Node: Server thuộc Cluster
- `Failover`: Khi 1 hoặc nhiều node trong Cluster xảy ra vấn đề, các tài nguyên (resource) tự động được chuyển tới các node sẵn sàng phục vụ
- `Failback`: Khi node lỗi phục hồi, sẵn sàng phục vụ, Cluster tự động trả lại tài nguyên cho node
- `Fault-tolerant cluster`: Để cập đến khả năng chịu lỗi của hệ thống trên các thành phần, cho phép dịch vụ hoạt động ngay cả khi một vài thành phần gặp sự cố
- `Heartbeat`: Tín hiệu xuất phát từ các node trong cụm với mục đích xác minh chùng còn sống và đang hoạt động. Nếu heartbeat tại 1 node ngừng hoạt động, cluster sẽ đánh dấu thành phần đó gặp sự cố và chuyển tài nguyên tại node lỗi sang node đang sẵn sàng phục vụ
- `Interconnect`: Kết nối giữa các node. Thường là các thông tin về trạng thái, heartbeat, dữ liệu chia sẻ.
- `Primary server, secondary server`: Trong cluster dạng Active/Passise, node đang đáp ứng giải quyết yêu cầu gọi là Primary server. Node đang chờ hay dự phòng cho node Primary server được gọi là secondary server
- `Quorum`: Trong Cluster chứa nhiều tài nguyên, nên dễ xảy ra sự phân mảnh (split-brain - Tức cluster lớn bị tách ra thành nhiều cluster nhỏ). Điều này sẽ dẫn đến sự mất đồng bộ giữa các tài nguyên, ảnh hướng tới sự toàn vẹn của hệ thống. Quorum được thiết kế để ngăn chặn hiện tượng phân mảnh.
- `Resource:` Tài nguyên của cụm, cũng có thể hiểu là tài nguyên mà dịch vụ cung cấp
STONITH/ Fencing: STONITH là viết tắt của cụm từ Shoot Other Node In The Head đây là một kỹ thuật dành cho fencing. Fencing là kỹ thuật cô lập tài nguyên tại từng node trong Cluster. Mục tiêu STONITH là tắt hoặc khởi động lại node trong trường hợp Node trong trường hợp dịch vụ không thể khôi phục.



### Ví dụ

- Mô hình Active / Passive

Giống cấu hình Active - Active, Active Passive Cluster cần ít nhất 2 node, tuy nhiên không phải tất cả các node đều sẵn sàng xử lý yêu cầu. VD: Nếu có 2 node thì 1 node sẽ chạy ở chế độ Active, node còn lại sẽ chạy ở chế độ passive hoặc standby.


Passive Node sẽ hoạt động như 1 bản backup của Active Node. Trong trường hợp Active Node xảy ra vấn đề, Passive Node sẽ chuyển trạng thái thành active, tiếp quản xử lý các yêu cầu

![image](https://user-images.githubusercontent.com/83824403/164454445-6668c808-de25-48ed-a201-a82dcf7aaf16.png)


- Mô hình Shared Failover


![image](https://user-images.githubusercontent.com/83824403/164454384-1c343af2-a6df-40e4-b01c-cdd73929114e.png)


- Mô hình Active/ Active ( N to N)

Active Active cluster được tạo ra từ ít nhất 2 node, cả 2 node chạy đồng thời xử lý cùng 1 loại dịch vụ. Mục đích chính của Active Active Cluster là tối ưu hóa cho hoạt động cân bằng tải (Load balancing)


Khuyển cáo cho chế độ Active Active Cluster là các node trong cụm cần được cấu hình giống nhau tránh tình trạng phân mảnh cụm.

![image](https://user-images.githubusercontent.com/83824403/164454278-4e2a0f6d-4bcb-49a3-aa44-01436a32e2aa.png)
