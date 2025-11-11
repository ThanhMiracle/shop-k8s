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
### Khái niệm là một controller trong Kubernetes, dùng để quản lý vòng đời của Pod.

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

Service trong Kubernetes chính là cách “expose” (mở ra) một địa chỉ IP cố định để người khác (hoặc Pod khác) có thể truy cập vào Deployment của bạn.

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


## “Service có cơ chế Load Balancer không?” — Câu trả lời ngắn gọn

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


**Note: Nếu như các pod ko thể curl/ping đến DNS của service thì hãy nhớ kiểm tra vxLanMode trong file deployment của Calico CNI.**


Trong Kubernetes, mỗi Service là một lớp “proxy” để truy cập các Pod.
Để kết nối chính xác, Kubernetes phải biết Service lắng nghe ở port nào, và Container thật bên trong Pod đang chạy ở port nào.

Do đó, có 3 mức:

- containerPort → cổng mà ứng dụng bên trong container đang chạy.

- targetPort → cổng trên Pod mà Service sẽ forward request tới.

- port → cổng mà Service expose cho các client trong cluster.



# V. Kube-proxy là gì?

kube-proxy là một thành phần mạng (network component) chạy trên mỗi node trong cluster Kubernetes.
Nó có nhiệm vụ:

🧭 Duy trì và áp dụng các quy tắc định tuyến mạng (routing rules) để đảm bảo traffic từ Service được chuyển đúng đến Pod.

Khi bạn gọi Service, kube-proxy sẽ biết phải gửi request đó đến Pod nào. 
``` bash

+---------------------------------------------------+
|                 Kubernetes Cluster                |
|---------------------------------------------------|
|   API Server   |   Scheduler   |   Controller     |
|---------------------------------------------------|
|   Node 1              Node 2               Node 3 |
|  +---------+          +---------+           +---------+
|  | kubelet |          | kubelet |           | kubelet |
|  | kube-proxy|        | kube-proxy|         | kube-proxy|
|  +---------+          +---------+           +---------+
|     |                     |                    |
|     v                     v                    v
|   Pod A                 Pod B                Pod C
|  (app)                 (app)                (app)
+---------------------------------------------------+
```


→ Mỗi node có một kube-proxy chạy như một daemon (thường là DaemonSet).
Nó theo dõi Service và Endpoint từ API Server, sau đó thiết lập iptables / IPVS rules tương ứng để chuyển hướng traffic.

# VI. Kubelet
## Kubelet là thành phần “agent” chạy trên mỗi Node trong cluster Kubernetes.
Nó chịu trách nhiệm đảm bảo tất cả các Pod được mô tả bởi Kubernetes API (hoặc file manifest cục bộ) đang chạy đúng cách trên Node đó.

💡 Nói ngắn gọn: kubelet là “người quản gia” của Node — nó nhận chỉ thị từ Control Plane, rồi đảm bảo container trong Node chạy đúng yêu cầu.

# VII. Endpoint của Service 

Trong Kubernetes, Endpoint là một đối tượng (object) đại diện cho danh sách các IP:Port của Pod mà một Service trỏ đến.

👉 Nói ngắn gọn:
Service = “địa chỉ ảo”,
Endpoint = “địa chỉ thật” (PodIP + Port).

Khi bạn tạo một Service với selector, Kubernetes sẽ:

- Tìm tất cả các Pod có nhãn (label) khớp với selector đó.

- Lấy Pod IP và cổng container của chúng.

- Tạo ra một Endpoint object chứa thông tin đó.


1. Service
``` bash
kubectl get svc web-svc

NAME       TYPE        CLUSTER-IP     PORT(S)
web-svc    ClusterIP   10.0.0.50      80/TCP
```

2. Endpoint
``` bash 
kubectl get endpoints web-svc

NAME       ENDPOINTS
web-svc    10.244.1.10:80,10.244.2.15:80,10.244.3.20:80
```

💡 Giải thích:

10.244.x.x là IP của từng Pod thực tế.

Endpoint giữ danh sách đó để kube-proxy hoặc IPVS dùng khi định tuyến request.

| Thành phần     | Vai trò                                     | Được tạo khi                               |
| -------------- | ------------------------------------------- | ------------------------------------------ |
| **Service**    | Địa chỉ ảo để truy cập Pod                  | Khi bạn định nghĩa `Service`               |
| **Endpoint**   | Danh sách IP:Port thật của Pod       | Khi Pod có label khớp với Service selector |
| **kube-proxy** | Dùng Endpoint để định tuyến request đến Pod | Khi Service có traffic                     |

# VIII. CNI là gì 
CNI là một bộ giao thức và thư viện cho phép các hệ thống như Kubernetes, Mesos hay Cloud Foundry kết nối mạng cho container.
CNI định nghĩa cách một plugin mạng nên tạo, cấu hình và xóa interface mạng trong container.

Nói đơn giản: Kubernetes không tự quản lý mạng — nó dựa vào CNI plugin để làm việc này.

### Vai trò của CNI trong Kubernetes

Trong Kubernetes, khi một Pod được tạo, kubelet sẽ:

1. Gọi CNI plugin để cấp IP cho Pod.

2. Cấu hình interface mạng (eth0) trong container.

3. Đảm bảo Pod có thể liên lạc với các Pod khác trong cluster.

Khi Pod bị xóa, kubelet lại gọi CNI để thu hồi IP và xóa cấu hình mạng tương ứng.

| Tên Plugin    | Đặc điểm chính                                                         |
| ------------- | ---------------------------------------------------------------------- |
| **Flannel**   | Dễ cài đặt, phù hợp cho môi trường nhỏ; dùng VXLAN để overlay network. |
| **Calico**    | Hiệu năng cao, hỗ trợ network policy và routing ở layer 3.             |
| **Weave Net** | Dễ thiết lập, tự động mesh network giữa các node.                      |
| **Cilium**    | Sử dụng eBPF, mạnh về bảo mật và quan sát lưu lượng.                   |
| **Canal**     | Kết hợp Calico (network policy) + Flannel (overlay).                   |


Kubelet<br>
  └── gọi CNI plugin (ví dụ: Calico)<br>
        ├── Tạo interface eth0 trong container<br>
        ├── Gán địa chỉ IP<br>
        ├── Thiết lập routing table<br>
        └── Cập nhật cấu hình mạng cho Pod
 

# IX. PVC và PV trong K8S
## 💾 1. PV (Persistent Volume) — “Ổ cứng thật”

👉 Hãy tưởng tượng PV giống như một ổ cứng hoặc USB thật mà quản trị viên (admin) chuẩn bị sẵn cho bạn.
Nó là nơi để lưu dữ liệu lâu dài, không mất đi khi Pod bị xóa hoặc khởi động lại.

Ví dụ:

PV có thể là ổ đĩa trên máy chủ, ổ EBS của AWS, hoặc thư mục /mnt/data trên Node.

## 📜 2. PVC (Persistent Volume Claim) — “Yêu cầu mượn ổ cứng”

PVC giống như phiếu mượn ổ cứng 🧾.
Khi bạn (developer) muốn lưu dữ liệu, bạn không chạm vào ổ cứng trực tiếp — bạn chỉ gửi yêu cầu (PVC) cho Kubernetes:

“Tôi cần 2GB để lưu dữ liệu.”

Kubernetes sẽ tìm một PV phù hợp (có đủ dung lượng và còn trống) để “cho bạn mượn”.


## 🔄 3. Cách hoạt động dễ hiểu

          Người dùng (Pod)
                    ↓
          Tạo PVC (yêu cầu 2GB)
                    ↓
    Kubernetes tìm PV (ổ 5GB có sẵn)
                    ↓
            PVC gắn với PV
                    ↓
    Pod lưu dữ liệu lên PV qua PVC

# X. K8S Network
## 🌐 1. Mục tiêu của mạng trong Kubernetes

Kubernetes có một nguyên tắc cơ bản:

"Mỗi Pod có địa chỉ IP riêng, và tất cả Pod có thể giao tiếp trực tiếp với nhau mà không cần NAT."

Điều này nghĩa là:

- Mỗi Pod là một “máy ảo nhỏ” có IP riêng.

- Các Pod trên cùng hoặc khác Node đều có thể nói chuyện trực tiếp.

- Người dùng hoặc các Pod khác có thể truy cập Pod thông qua Service.

## 🧩 2. Các loại mạng trong Kubernetes

K8s có 4 tầng mạng chính, mỗi tầng có vai trò riêng:
| Tầng mạng                     | Mô tả ngắn                                                    |
| ----------------------------- | ------------------------------------------------------------- |
| **1. Container-to-Container** | Các container trong cùng một Pod nói chuyện qua `localhost`.  |
| **2. Pod-to-Pod**             | Các Pod (cùng hoặc khác Node) nói chuyện với nhau qua IP.     |
| **3. Pod-to-Service**         | Pod truy cập Service (qua DNS, ClusterIP, hoặc LoadBalancer). |
| **4. External-to-Service**    | Người dùng bên ngoài cluster truy cập ứng dụng trong cluster. |

## 🧱 3. Các thành phần chính của mạng K8s
### 🔹CNI Plugin (Container Network Interface)

CNI (như Calico, Flannel, Cilium) chịu trách nhiệm cấp IP cho Pod, thiết lập route, và đảm bảo Pod có thể liên lạc với nhau.

👉 Đây là tầng "cơ sở hạ tầng" của mạng trong Kubernetes.

### 🔹Kube-proxy

- Kube-proxy chạy trên mỗi Node và xử lý traffic của Service.

- Duy trì iptables hoặc IPVS rules để định tuyến traffic đến Pod phù hợp.

- Giúp Service có thể phân tải (load balancing) giữa nhiều Pod.

### 🔹CoreDNS

- CoreDNS là DNS server nội bộ trong Kubernetes.

Khi bạn gọi http://nginx.default.svc.cluster.local, DNS sẽ phân giải tên thành ClusterIP của Service.

## 🔄 4. Cách Pod giao tiếp trong mạng
### 🔸 Pod cùng Node

Các Pod trên cùng Node nói chuyện qua bridge network (cni0 hoặc flannel.1).

Không cần NAT, chỉ route nội bộ.

### 🔸 Pod khác Node

CNI plugin thiết lập overlay network (VXLAN, BGP, eBPF…).

Traffic được đóng gói và gửi qua Node khác.

### 🔸 Pod truy cập Service

Pod gửi request đến ClusterIP (ảo).

kube-proxy điều hướng traffic đến một trong các Pod backend.

### 🔸 Bên ngoài truy cập cluster

Qua NodePort, LoadBalancer, hoặc Ingress Controller (ví dụ: Nginx Ingress, Traefik).

## 🌉 5. Các loại Service trong Kubernetes
| Loại             | Mục đích                                      | Cách truy cập     |
| ---------------- | --------------------------------------------- | ----------------- |
| **ClusterIP**    | Mặc định, chỉ truy cập trong cluster          | IP nội bộ (ảo)    |
| **NodePort**     | Cho phép truy cập từ bên ngoài qua port Node  | `NodeIP:NodePort` |
| **LoadBalancer** | Dùng cho cloud (AWS, GCP...) để cấp IP public | IP công khai      |
| **ExternalName** | Chuyển hướng DNS đến service bên ngoài        | DNS record        |

## 📊 6. Tóm tắt
| Thành phần     | Vai trò                                  |
| -------------- | ---------------------------------------- |
| **CNI Plugin** | Cấp IP, thiết lập mạng giữa Pod          |
| **kube-proxy** | Định tuyến và cân bằng tải Service       |
| **CoreDNS**    | Phân giải tên miền nội bộ                |
| **Service**    | Tạo điểm truy cập ổn định cho Pod        |
| **Ingress**    | Quản lý truy cập HTTP/HTTPS từ bên ngoài |

# XI. CoreDNS
## 🧭 1. CoreDNS 
- Là DNS Server (máy chủ phân giải tên miền) mặc định trong Kubernetes.
- Nó giúp các Pod và Service trong cluster tìm thấy nhau bằng tên, thay vì phải nhớ địa chỉ IP.

## 🌐 2. Ví dụ dễ hiểu
Giả sử bạn có 2 Service:

- frontend trong namespace default

- backend trong namespace default

Pod trong frontend muốn gọi backend, chỉ cần gọi:

```bash
curl http://backend.default.svc.cluster.local
```

CoreDNS sẽ phân giải tên này thành IP của Service backend, ví dụ:
```bash
10.96.0.15
```

## ⚙️ 3. Cấu trúc phân giải DNS trong Kubernetes
Tên đầy đủ của một Service có dạng:
```bash
<ServiceName>.<Namespace>.svc.cluster.local
```

Ví dụ:

```bash
nginx.default.svc.cluster.local
```

- nginx: Tên Service

- default: Namespace

- svc: Loại đối tượng (Service)

- cluster.local: Domain mặc định của cluster

## 🧩 4. Cách CoreDNS hoạt động

- Mỗi Pod trong cluster sẽ tự động cấu hình DNS để trỏ đến CoreDNS (qua file /etc/resolv.conf).

- Khi Pod gửi yêu cầu DNS, nó đi đến Pod CoreDNS (thường nằm trong namespace kube-system).

- CoreDNS đọc thông tin Service/Pod từ Kubernetes API Server.

- CoreDNS trả lại địa chỉ IP tương ứng cho Pod hoặc Service.

## 5. Tóm tắt
| Thành phần          | Vai trò                            |
| ------------------- | ---------------------------------- |
| **CoreDNS**         | DNS server nội bộ trong Kubernetes |
| **Nhiệm vụ**        | Phân giải tên Service/Pod thành IP |
| **Vị trí**          | Namespace `kube-system`            |
| **Thay thế**        | Trước đây là kube-dns              |
| **Domain mặc định** | `.cluster.local`                   |


<div align="center">

[Pod A] → "backend.default.svc.cluster.local" <br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;↓<br>
[CoreDNS]<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;↓<br>
Kubernetes API → tìm Service "backend"<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;↓<br>
Trả về IP: 10.96.0.15

</div>


👉 Tóm gọn 1 câu:

CoreDNS trong Kubernetes là bộ não DNS giúp các Pod và Service tìm thấy nhau qua tên, thay vì IP.


# XII. ConfigMap

## 🧠 1. ConfigMap là gì?

ConfigMap là một đối tượng (object) trong Kubernetes dùng để lưu trữ dữ liệu cấu hình (configuration data) cho ứng dụng.
Nó giúp bạn tách riêng phần cấu hình ra khỏi code của container.

💡 Nói ngắn gọn:
ConfigMap = file cấu hình của ứng dụng trong Kubernetes.

## 🔍 2. Tại sao cần ConfigMap?

Trong thực tế, ứng dụng thường có các thông số như:

- Địa chỉ cơ sở dữ liệu (DB_HOST)

- Cổng (PORT)

- Chế độ chạy (ENV=production)

Nếu bạn viết các giá trị này cứng trong code hoặc image, khi muốn thay đổi, bạn phải build lại image — rất tốn công.

👉 ConfigMap giúp giải quyết vấn đề đó:
Bạn chỉ cần chỉnh ConfigMap, Pod tự động đọc cấu hình mới mà không cần rebuild container.

## 🔄 3. Khi nào dùng ConfigMap

| Tình huống                                   | Giải pháp     |
| -------------------------------------------- | ------------- |
| Cần cấu hình không bảo mật (public)          | **ConfigMap** |
| Cần lưu thông tin nhạy cảm (password, token) | **Secret**    |
| Cần mount file cấu hình (JSON, YAML)         | **ConfigMap** |
| Cần truyền nhiều biến môi trường vào Pod     | **ConfigMap** |

## 🚀 4. Tóm tắt

| Thành phần          | Vai trò                                               |
| ------------------- | ----------------------------------------------------- |
| **ConfigMap**       | Lưu dữ liệu cấu hình dạng key–value                   |
| **Tạo bởi**         | YAML file hoặc lệnh `kubectl`                         |
| **Dùng trong**      | Pod, Deployment, hoặc Job                             |
| **Khác với Secret** | Secret dùng cho dữ liệu nhạy cảm (được mã hóa Base64) |


<div align="left">

ConfigMap<br>
&nbsp;&nbsp;├── APP_ENV: production<br>
&nbsp;&nbsp;└── DB_HOST: postgres-service<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;│<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;▼<br>
Pod<br>
&nbsp;&nbsp;└── container<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;└── image

</div>


