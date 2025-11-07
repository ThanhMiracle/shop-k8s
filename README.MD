# I. ReplicationController (RC)
ReplicationController là controller đầu tiên trong Kubernetes dùng để đảm bảo số lượng Pod luôn duy trì cố định.

Nếu có 3 replica:

- Khi 1 Pod bị crash → RC tạo thêm 1 Pod mới.

- Nếu bạn xóa thủ công 1 Pod → RC sinh thêm 1 Pod mới.

# II. ReplicaSet (RS)
Khái niệm

ReplicaSet là phiên bản nâng cấp của ReplicationController.
Công dụng chính: Đảm bảo số lượng bản sao Pod (replica) luôn đúng với khai báo.

Nó cũng theo dõi các Pod qua selector, nếu thiếu thì tạo thêm, thừa thì xóa bớt.

# III. Deployment
Khái niệm

Deployment là lớp quản lý cao hơn, bao bọc ReplicaSet, giúp bạn:

- Tự động tạo ReplicaSet.

- Triển khai phiên bản mới (Rolling Update).

- Quay lại phiên bản cũ (Rollback).

- Tạm dừng / tiếp tục cập nhật.

- Dễ dàng scale Pod.

👉 Nói cách khác:
Deployment → quản lý ReplicaSet → quản lý Pod


| Lệnh                                             | Mô tả                               |
| ------------------------------------------------ | ----------------------------------- |
| `kubectl get deploy`                             | Xem danh sách Deployment            |
| `kubectl get rs`                                 | Xem ReplicaSet mà Deployment tạo ra |
| `kubectl get pods`                               | Xem các Pod được quản lý            |
| `kubectl scale deploy nginx-deploy --replicas=5` | Scale lên/xuống                     |
| `kubectl rollout status deploy nginx-deploy`     | Xem trạng thái cập nhật             |
| `kubectl rollout undo deploy nginx-deploy`       | Rollback về version trước           |





# IV. K8s Service 

🧩 Khái niệm:

Trong Kubernetes, Service là một đối tượng logic dùng để định danh và truy cập một nhóm Pod.
Vì Pod có thể thay đổi IP liên tục (khi restart, reschedule...), nên Service cung cấp một IP ổn định (ClusterIP) và DNS name cố định để các Pod khác hoặc người dùng truy cập.

👉 Nói đơn giản:

- Pod = tạm thời, IP thay đổi.

- Service = cố định, là “cửa” vào nhóm Pod.

Cách hoạt động

1. Mỗi Service có một label selector, chọn các Pod phù hợp.

2. Kubernetes tạo một endpoint list chứa IP của các Pod đó.

3. kube-proxy trên mỗi Node tạo iptables rules hoặc IPVS rules để chuyển tiếp request đến một trong các Pod (theo round-robin, random…).


| Loại Service     | Mục đích                                          | Truy cập từ đâu             | IP cấp phát                | Thường dùng khi                     |
| ---------------- | ------------------------------------------------- | --------------------------- | -------------------------- | ----------------------------------- |
| **ClusterIP**    | Truy cập **nội bộ cluster**                       | Pod → Pod                   | Internal IP                | Microservice nội bộ                 |
| **NodePort**     | Expose ra **bên ngoài cluster (qua Node IP)**     | Ngoài cluster → NodeIP:Port | Mỗi Node có 1 port cố định | Dev/Test hoặc cần truy cập thủ công |
| **LoadBalancer** | Dùng **Load Balancer bên ngoài** (Cloud Provider) | Internet hoặc mạng ngoài    | IP Public/External         | Production, public traffic          |
| **ExternalName** | Trỏ tới **service bên ngoài cluster** (DNS name)  | Nội bộ → External resource  | Không tạo IP               | Dùng khi kết nối external DB/API    |


# “Service có cơ chế Load Balancer không?” — Câu trả lời ngắn gọn

✅ Có!

Tất cả các Service (trừ ExternalName) trong Kubernetes đều thực hiện load balancing ở mức nội bộ — tức là khi bạn gửi request đến Service, nó sẽ phân phối traffic đều giữa các Pod backend (các Pod matching label selector).

Tuy nhiên:

Cơ chế “load balancing” này do kube-proxy đảm nhiệm, không phải là Load Balancer phần cứng hoặc cloud LB.


## Cách hoạt động chi tiết
### 1️⃣ Khi bạn tạo Service (ClusterIP, NodePort, hoặc LoadBalancer)

Kubernetes tạo một đối tượng Service có IP riêng (ClusterIP).

kube-proxy trên mỗi Node:

Theo dõi các endpoint (IP:Port của Pod).

Tạo iptables hoặc IPVS rules để phân phối request đến các Pod.

### 2️⃣ Cơ chế cân bằng tải nội bộ

Có 2 chế độ mà kube-proxy có thể dùng:

1. iptables mode (mặc định)

- kube-proxy cài đặt nhiều rule iptables NAT để chọn Pod ngẫu nhiên.

- Mỗi request đến Service IP → bị NAT sang một Pod IP.

- Cân bằng theo kiểu random hoặc round-robin.

- Ưu điểm: nhanh, ít tài nguyên.

2) IPVS mode

- Dùng Linux IP Virtual Server (IPVS) để làm load balancing kernel-level.

- Hiệu năng cao hơn, có thể chọn thuật toán:

- rr (round robin)

- lc (least connection)

- dh (destination hashing)

- sh (source hashing)

- Dùng trong cluster lớn, nhiều Service.


| Service Type     | Có cân bằng tải không? | Cơ chế                     | Mô tả                                                              |
| ---------------- | ---------------------- | -------------------------- | ------------------------------------------------------------------ |
| **ClusterIP**    | ✅ Có                   | kube-proxy (iptables/IPVS) | Cân bằng request nội bộ giữa các Pod backend                       |
| **NodePort**     | ✅ Có                   | kube-proxy (iptables/IPVS) | Khi truy cập qua NodeIP:NodePort, request được chuyển đều giữa Pod |
| **LoadBalancer** | ✅ Có (2 tầng)          | kube-proxy + external LB   | Cloud LB → NodePort → kube-proxy → Pod                             |
| **ExternalName** | ❌ Không                | DNS alias                  | Chỉ là ánh xạ DNS, không có balancing trong cluster                |





| Tính năng            | ClusterIP            | NodePort             | LoadBalancer        | ExternalName          |
| -------------------- | -------------------- | -------------------- | ------------------- | --------------------- |
| **Phạm vi truy cập** | Nội bộ cluster       | Bên ngoài qua NodeIP | Bên ngoài (qua LB)  | External service      |
| **Tự động scale**    | ✅                    | ✅                    | ✅                   | ❌                     |
| **IP công khai**     | ❌                    | ✅ (Node IP)          | ✅ (External IP)     | ✅ (DNS)               |
| **Dùng trong cloud** | ✅                    | ✅                    | ✅                   | ✅                     |
| **Phù hợp cho**      | Microservices nội bộ | Dev/test             | Production (public) | Kết nối dịch vụ ngoài |


| Thành phần       | Vai trò chính                                    |
| ---------------- | ------------------------------------------------ |
| **ClusterIP**    | Giao tiếp nội bộ trong cluster                   |
| **NodePort**     | Expose port ra ngoài qua Node IP                 |
| **LoadBalancer** | Tự động tạo LB (Cloud Provider) để expose public |
| **ExternalName** | Alias nội bộ cho dịch vụ ngoài cluster           |
| **Ingress**      | Quản lý routing HTTP(S) đến nhiều Service nội bộ |


### Note: Nếu như các pod ko thể curl/ping đến DNS của service thì hãy nhớ kiểm tra vxLanMode của Calico DNS.


Trong Kubernetes, mỗi Service là một lớp “proxy” để truy cập các Pod.
Để kết nối chính xác, Kubernetes phải biết Service lắng nghe ở port nào, và Container thật bên trong Pod đang chạy ở port nào.

Do đó, có 3 mức:

- containerPort → cổng mà ứng dụng bên trong container đang chạy.

- targetPort → cổng trên Pod mà Service sẽ forward request tới.

- port → cổng mà Service expose cho các client trong cluster.


