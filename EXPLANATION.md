# Giải thích chi tiết Cơ chế GitOps & ArgoCD

Chào bạn! Dưới đây là phần giải thích chi tiết về các khái niệm cốt lõi đã được áp dụng trong bài thực hành này. Bạn có thể dùng tài liệu này để tự ôn tập hoặc thuyết trình lại cho team.

## 1. Cơ chế hoạt động của Sealed Secrets (Luồng Public/Private Key)

**Tại sao phải dùng Sealed Secrets?**
Trong GitOps, nguyên tắc số 1 là: **Mọi thứ phải nằm trên Git** (Single Source of Truth). Tuy nhiên, các Secret (mật khẩu DB, API Key) nếu đẩy lên Git dưới dạng plain text (chữ thường) hoặc Base64 thì cực kỳ nguy hiểm vì ai có quyền xem repo cũng có thể đọc được mật khẩu.

**Sealed Secrets giải quyết bài toán này như thế nào?**
Sealed Secrets hoạt động dựa trên cơ chế **Mã hóa bất đối xứng (Asymmetric Encryption)** giống như ổ khóa và chìa khóa:

- **Public Key (Ổ khóa):** Được công khai, ai cũng có thể lấy. Bạn dùng Public Key (thông qua CLI `kubeseal`) để khóa (mã hóa) Secret thô thành một chuỗi mã hóa gọi là `SealedSecret`. Chuỗi này rất an toàn, có thể thoải mái push lên Github.
- **Private Key (Chìa khóa):** Chỉ duy nhất `sealed-secrets-controller` nằm trong Kubernetes cluster mới giữ chìa khóa này. 

**Luồng hoạt động:**
1. Trưởng nhóm/Dev tạo secret trên máy cá nhân và dùng `kubeseal` (cùng public key lấy từ cluster) mã hóa nó.
2. File chứa `SealedSecret` được commit lên Git.
3. ArgoCD đồng bộ file này vào Kubernetes cluster.
4. `sealed-secrets-controller` trong cluster nhận diện được `SealedSecret`, nó dùng Private Key để giải mã và tự động tạo ra một `Secret` thật sự trong Kubernetes.
5. Pod/App của bạn mount cái `Secret` thật này để dùng như bình thường.

*(Lưu ý Scope Strict: Nghĩa là chuỗi mã hóa bị gắn chặt với `name` và `namespace` của Secret. Nếu bạn copy chuỗi đó sang namespace khác, controller sẽ từ chối giải mã để tránh việc người khác đánh cắp chuỗi mã hóa rồi tự đem về namespace của họ để mở!)*

---

## 2. ArgoCD ApplicationSet quét thư mục và tự sinh App như thế nào?

**Tại sao cần ApplicationSet?**
Nếu bạn có 10 Microservices và 3 môi trường (Dev, Staging, Prod), bạn sẽ phải tạo tay 30 cái Application trên ArgoCD. Quá mệt mỏi! ApplicationSet sinh ra để tự động hóa việc này.

**Cơ chế hoạt động:**
ApplicationSet hoạt động theo mô hình **Generator + Template**.
- **Generator (Máy quét):** Trong bài thực hành, chúng ta dùng Git Generator. Nó sẽ quét repo Git theo một đường dẫn (ví dụ: `apps/birdnet-market/*/overlays/dev`). 
- **Template (Khuôn đúc):** Mỗi khi Generator tìm thấy một thư mục khớp với đường dẫn quét, nó sẽ lấy các thông tin của thư mục đó (tên thư mục, đường dẫn) nhét vào Template để tự động đúc ra một file `Application` mới.

**Ví dụ thực tế trong bài:**
Khi bạn thêm `- path: "apps/birdnet-market/*/overlays/uat"` vào ApplicationSet:
1. Máy quét chạy vào repo, thấy thư mục `apps/birdnet-market/frontend/overlays/uat`.
2. Nó trích xuất ra: `project = birdnet-market`, `app = frontend`, `env = uat`.
3. Nó ném các biến này vào Template và "BÙM" 💥, một ArgoCD Application tên là `birdnet-market-frontend-uat` tự động xuất hiện trên giao diện ArgoCD và bắt đầu deploy ứng dụng!

---

## 3. Overlay trong Kustomize (hoặc Helm + Values) ghi đè Base như thế nào?

Trong bài thực hành, thay vì dùng Kustomize thuần, chúng ta đang dùng **Helm chart (Base) kết hợp với các Values files (Overlay/Môi trường)** qua cơ chế Multi-source của ArgoCD. Tuy nhiên, nguyên lý "Base/Overlay" vẫn hoàn toàn tương đồng:

**Tại sao cần Base/Overlay?**
Các môi trường (Dev, Staging, Prod) thường có cấu hình giống nhau đến 90%. Nếu copy-paste toàn bộ file YAML ra 3 bản thì rất dễ sinh rác và khó bảo trì.

**Cơ chế ghi đè:**
- **Base (Cốt lõi - Helm Chart):** Chứa các cấu hình cơ bản, mặc định và chung nhất cho mọi môi trường (Deployment, Service có gì, chạy port mấy...). Base tuyệt đối không được sửa đổi cho từng môi trường!
- **Overlay (Lớp phủ - `values.yaml` trong thư mục dev/prod):** Chỉ chứa những sự KHÁC BIỆT. Ví dụ: Dev cần 1 Replica, Prod cần 3 Replicas.

Khi ArgoCD deploy, nó sẽ lấy cái Base, sau đó đặt cái Overlay đè lên trên. Chỗ nào Overlay có khai báo thì lấy của Overlay, chỗ nào Overlay không nói gì thì lấy mặc định của Base.
Nhờ vậy, khi có thêm app `pipeline` hoặc component `scheduler`, bạn chỉ cần khai báo một vài dòng trong file `values.yaml` của Overlay thay vì phải viết lại hàng trăm dòng code YAML phức tạp!
