# Hướng Dẫn Thiết Lập Ubuntu & Chạy Thực Hành GitOps (Dành Cho Báo Cáo)

Tài liệu này hướng dẫn chi tiết các bước bạn cần làm sau khi boot sang Ubuntu: từ việc cài đặt môi trường, chạy từng bài tập, cho đến những hình ảnh/kết quả cần chụp lại để đưa vào báo cáo môn học hoặc đồ án.

## Phần 1: Thiết lập môi trường trên Ubuntu (Prerequisites)

Khi mới boot sang Ubuntu, bạn cần cài đặt các công cụ sau để giả lập một cụm Kubernetes và chạy GitOps. Mở Terminal lên và chạy lần lượt các bước sau:

### 1. Cài đặt Docker
Docker là nền tảng cốt lõi để chạy các container.
```bash
sudo apt-get update
sudo apt-get install -y docker.io
sudo usermod -aG docker $USER && newgrp docker
```

### 2. Cài đặt Minikube & kubectl
Minikube giúp bạn chạy một cụm Kubernetes nhỏ ngay trên máy tính cá nhân.
```bash
# Cài kubectl (công cụ giao tiếp với Kubernetes)
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Cài Minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Khởi động cụm Kubernetes (Tùy cấu hình máy mà chọn CPU/RAM phù hợp)
minikube start --driver=docker --memory=4096 --cpus=4
```

### 3. Cài đặt Kubeseal (CLI cho bài Sealed Secrets)
```bash
# Lấy phiên bản kubeseal mới nhất
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.26.2/kubeseal-0.26.2-linux-amd64.tar.gz
tar -xvzf kubeseal-0.26.2-linux-amd64.tar.gz kubeseal
sudo install -m 755 kubeseal /usr/local/bin/kubeseal
```

### 4. Cài đặt ArgoCD vào Cluster
Bạn cần cài đặt Core của ArgoCD lên Minikube để nó tiếp quản việc quản lý GitOps.
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Cài ArgoCD CLI (Tùy chọn, dùng để thao tác dòng lệnh cho tiện)
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
```

### 5. Cài đặt Sealed Secrets Controller vào Cluster
Phải có Controller này chạy ngầm trong Cluster thì nó mới giải mã được file cấu hình bạn mã hóa.
```bash
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.26.2/controller.yaml
```

---

## Phần 2: Hướng dẫn chạy từng bài tập và Chụp ảnh báo cáo

Trước khi bắt đầu, hãy **clone repo GitOps** của bạn về máy Ubuntu và `cd` vào đó:
```bash
git clone https://github.com/Tung-it2-k67-hust/gitops-training-tungbeo-2026-VDT.git
cd gitops-training-tungbeo-2026-VDT
```

### Bài 1: Thực hành Sealed Secrets
**Thao tác cần làm:**
1. Chạy file script để mã hóa (Kubeseal sẽ kết nối với cụm Minikube qua kubectl để lấy Public Key và thực hiện khóa):
   ```bash
   ./scripts/seal.sh mention-mate-dev mention-mate-app-dev-secret DB_PASSWORD=my_super_secret_password API_KEY=123456
   ```
2. Copy chuỗi `encryptedData` in ra màn hình và dán vào file `apps/mention-mate/app/overlays/dev/values.yaml`. (Sau đó dùng git add, commit, push lên Github).

**KẾT QUẢ CẦN CHỤP LẠI ĐỂ VIẾT BÁO CÁO:**
📸 **Ảnh 1:** Chụp màn hình Terminal lúc chạy lệnh `./scripts/seal.sh` in ra chuỗi mã hóa thành công (minh chứng bạn đã mã hóa được Secret).
📸 **Ảnh 2:** Sau khi chờ ArgoCD đồng bộ, chạy lệnh `kubectl get secret mention-mate-app-dev-secret -n mention-mate-dev -o yaml`. Chụp lại màn hình để chứng minh SealedSecret đã được Controller giải mã thành Secret thật sự (ở dạng base64) nằm an toàn trong Cluster.

---

### Bài 2 & 3: Bootstrap Cụm (Kích hoạt ArgoCD) & Tạo môi trường UAT
**Thao tác cần làm:**
Kích hoạt mô hình App-of-Apps bằng cách apply trực tiếp 2 file root duy nhất (điều này chỉ làm 1 lần duy nhất trong đời của cụm):
```bash
kubectl apply -f root-nonproduction.yaml
kubectl apply -f root-production.yaml
```
Tiếp theo, mở giao diện UI của ArgoCD để quan sát độ "ảo diệu":
```bash
# Lấy mật khẩu đăng nhập ArgoCD (Username mặc định là admin)
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo

# Mở port-forward để truy cập giao diện UI ở localhost:8080
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
Truy cập `https://localhost:8080` trên trình duyệt của Ubuntu, chấp nhận cảnh báo SSL và đăng nhập.

**KẾT QUẢ CẦN CHỤP LẠI ĐỂ VIẾT BÁO CÁO:**
📸 **Ảnh 3:** Giao diện ArgoCD UI hiện ra một mạng lưới các Application đang Syncing rồi chuyển xanh (Healthy). Đây là minh chứng cực mạnh cho mô hình **App-of-Apps** đã tự động hóa mọi thứ.
📸 **Ảnh 4:** Tìm ô Application có tên `birdnet-market-frontend-uat` trên giao diện ArgoCD và click vào nó. Chụp lại cây mạng lưới tài nguyên (Deployment, Service, Pods) của app này. Minh chứng bạn đã cấu hình ApplicationSet để nó tự sinh thêm môi trường UAT thành công.

---

### Bài 4: Thêm Deployment Scheduler vào mention-mate
**Thao tác cần làm:**
Vì phần code này đã nằm sẵn trên GitHub (ta đã làm trên Windows), ArgoCD sẽ tự động kéo về và tạo thêm một component tên là scheduler.

**KẾT QUẢ CẦN CHỤP LẠI ĐỂ VIẾT BÁO CÁO:**
📸 **Ảnh 5:** Trên giao diện ArgoCD của app `mention-mate-app-dev` (hoặc staging/prod), chụp lại hình ảnh có xuất hiện ô (node) tên là `scheduler` bên cạnh các ô `backend` và `worker`.
📸 **Ảnh 6 (Tùy chọn):** Gõ lệnh trên Terminal `kubectl get pods -n mention-mate-dev`. Chụp lại danh sách các Pod đang chạy (`Running`), trong đó sẽ thấy pod của `scheduler` nằm chung với `backend`. Minh chứng cơ chế Overlay ghi đè thành công mà không chạm vào Base Helm Chart.

---

### Bài 5: Thêm app mới `pipeline` cho `birdnet-market`
**Thao tác cần làm:**
Không cần làm gì thêm trên Ubuntu, ArgoCD AppSet Generator đã tự động quét thấy thư mục `pipeline` ta tạo và tự sinh ra App.

**KẾT QUẢ CẦN CHỤP LẠI ĐỂ VIẾT BÁO CÁO:**
📸 **Ảnh 7:** Chụp màn hình trang chủ ArgoCD, dùng thanh search gõ từ khóa `pipeline`, bạn sẽ thấy các app `birdnet-market-pipeline-dev`, `birdnet-market-pipeline-staging`, v.v. xuất hiện và có màu xanh. Nhấn mạnh vào việc **ApplicationSet tự quét (Generator)** và sinh app không cần tạo file Application thủ công.

---

### Bài 6: (Nâng cao) Thêm Project Notification mới
**Thao tác cần làm:**
Không cần thao tác gì thêm, ArgoCD đã tự động phát hiện thư mục `bootstrap/.../appprojects/notification/` và thiết lập ApplicationSet riêng cho Project này.

**KẾT QUẢ CẦN CHỤP LẠI ĐỂ VIẾT BÁO CÁO:**
📸 **Ảnh 8:** Vào mục **Settings -> Projects** trên thanh menu bên trái của ArgoCD UI. Chụp lại danh sách có Project mới tên là `notification` (minh chứng ta không tạo qua UI mà tạo qua code GitOps).
📸 **Ảnh 9:** Ra ngoài trang chủ Applications của ArgoCD, ở bộ lọc (Filter) bên trái, chọn Project `notification`. Chụp lại các Application của notification (dev, staging, prod) đang được deploy thành công.

---

## 📝 Tóm tắt cấu trúc khuyên dùng để viết Báo Cáo / Document:
Trong tài liệu báo cáo của bạn, để giảng viên thấy rõ sự hiểu biết, hãy trình bày theo dàn ý sau:
1. **Mục tiêu thực hành:** Tự động hóa quá trình deploy (GitOps) với ArgoCD và quản lý Secret bảo mật bằng luồng Public/Private Key của Sealed Secrets.
2. **Khái niệm & Cơ chế (có thể lấy từ file EXPLANATION.md):** 
   - Giải thích ngắn gọn cơ chế Base và Overlay trong Kustomize/Helm.
   - Trình bày sức mạnh của ApplicationSet (Generator + Template) so với việc tạo App thủ công.
3. **Thực nghiệm và Kết quả:** Dán lần lượt 9 bức ảnh chụp màn hình ở trên theo từng mục, bên dưới mỗi ảnh ghi dòng chú thích giải thích cụ thể bức ảnh đó chứng minh điều gì.
4. **Kết luận:** Khẳng định hệ thống đã hoạt động đúng theo triết lý No-Ops: "Chỉ cần push code lên Github, ArgoCD tự lo phần còn lại". Thảm họa rủi ro bảo mật Secret cũng được giải quyết.
