  Báo cáo chi tiết: Gỡ lỗi và Khắc phục sự cố ứng dụng tmcp-marketing

  1. Tình trạng ban đầu


  Ứng dụng tmcp-marketing trên Argo CD có trạng thái `Degraded`, cho thấy sự
  khác biệt giữa cấu hình mong muốn trong Git và trạng thái thực tế trên cluster
  Kubernetes.

  2. Nhật ký Điều tra và Phân tích (step-by-step)


  ##### Bước 1: Kiểm tra tổng quan ứng dụng trên Argo CD
  Để xác định thành phần nào đang báo lỗi, chúng ta kiểm tra trạng thái chi tiết
  của ứng dụng.
   - Lệnh: kubectl get application tmcp-marketing -n argocd -o yaml
   - Kết quả: Trạng thái health.status là Degraded. Mục
     status.syncResult.resources cho thấy nhiều Deployment (bao gồm
     marketing-agent, kibana, mcp-bridge) gặp lỗi exceeded its progress deadline
     (quá thời hạn triển khai).


  ##### Bước 2: Kiểm tra trạng thái các Pod
  Lỗi "quá thời hạn" thường do Pod không thể khởi động. Chúng ta kiểm tra tất cả
  các Pod trong namespace của ứng dụng.
   - Lệnh: kubectl get pods -n tmcp-marketing
   - Kết quả: Nhiều Pod (của kibana, marketing-agent, mcp-bridge) có trạng thái
     `CreateContainerConfigError`. Lỗi này có nghĩa là Kubernetes không thể tạo
     container cho Pod vì thiếu thông tin cấu hình.


  ##### Bước 3: Phân tích một Pod bị lỗi
  Chúng ta chọn một Pod (mcp-bridge) để "describe" và xem sự kiện chi tiết.
   - Lệnh: kubectl describe pod mcp-bridge-7ffb4b9965-pgwlb -n tmcp-marketing
   - Kết quả: Phần Environment Variables from cho thấy Pod này yêu cầu một
     Kubernetes Secret tên là bridge-secrets-k8s. Cờ Optional: false có nghĩa là
     Pod sẽ không khởi động nếu không tìm thấy Secret này.


  ##### Bước 4: Xác minh sự tồn tại của Kubernetes Secret
  Chúng ta kiểm tra xem Secret bridge-secrets-k8s có thực sự tồn tại trong
  cluster không.
   - Lệnh: kubectl get secret bridge-secrets-k8s -n tmcp-marketing
   - Kết quả: Error from server (NotFound): secrets "bridge-secrets-k8s" not
     found. Điều này xác nhận Pod lỗi vì thiếu Secret.


  ##### Bước 5: Kiểm tra tài nguyên ExternalSecret
  Kubernetes Secret này المفروض được tạo bởi một tài nguyên ExternalSecret.
  Chúng ta kiểm tra trạng thái của nó.
   - Lệnh: kubectl get externalsecret bridge-secrets -n tmcp-marketing -o yaml
   - Kết quả: Trạng thái báo lỗi SecretSyncedError với thông báo could not get
     secret data from provider. Điều này có nghĩa là External Secrets Operator
     (ESO) đã thất bại khi cố gắng lấy dữ liệu từ Vault.


  ##### Bước 6: Kiểm tra Log của External Secrets Operator
  Để tìm hiểu lý do ESO thất bại, chúng ta xem log của nó.
   - Lệnh: kubectl logs <external-secrets-pod-name> -n external-secrets
   - Kết quả: Log cho thấy một chuỗi các lỗi khác nhau theo thời gian:
       1. connect: connection refused: Lỗi kết nối mạng tới Vault.
       2. Code: 503. Errors: * Vault is sealed: Lỗi do Vault đang bị niêm phong
          (Sealed).
       3. Code: 403. Errors: * permission denied: Lỗi bị từ chối quyền khi đăng
          nhập.
       4. err: Secret does not exist: Lỗi báo bí mật không tồn tại trong Vault.


  ##### Bước 7: Phân tích các lỗi từ Log
   - Lỗi "Sealed" và "Connection Refused" có thể là tạm thời, nhưng lỗi `403
     Permission Denied` là manh mối quan trọng, cho thấy có vấn đề về cấu hình
     xác thực.
   - Chúng ta đào sâu vào lỗi này bằng cách kiểm tra cấu hình K8s auth của
     Vault.
   - Lệnh: kubectl exec -it -n vault vault-0 -- vault read
     auth/kubernetes/config
   - Phát hiện (Nguyên nhân gốc rễ #1): Kết quả cho thấy trường
     kubernetes_ca_cert có giá trị n/a. Đây là lỗi cốt lõi. Vault không có CA
     Certificate của K8s, nên nó không thể tin tưởng và xác thực token từ ESO,
     dẫn đến từ chối đăng nhập.


   - Tiếp theo, chúng ta xác minh lỗi "Secret does not exist".
   - Lệnh: kubectl exec -it -n vault vault-0 -- vault kv get secret/tmcp/bridge
   - Phát hiện (Nguyên nhân gốc rễ #2): Kết quả là No value found. Điều này xác
     nhận bí mật thực sự không tồn tại trong Vault.

  3. Giải pháp & Các câu lệnh khắc phục

  Dựa trên hai nguyên nhân gốc rễ đã tìm thấy, chúng ta đã thực hiện các bước
  sửa lỗi sau:


  ##### Fix 1: Sửa lỗi Cấu hình Xác thực K8s của Vault
  Để Vault có thể xác thực Service Account token, chúng ta cung cấp cho nó CA
  cert và token reviewer JWT của cluster.
   - Lệnh đã thực thi:


   1   kubectl exec -n vault vault-0 -- sh -c '\
   2     vault write auth/kubernetes/config \
   3       token_reviewer_jwt="$(cat
     /var/run/secrets/kubernetes.io/serviceaccount/token)" \
   4       kubernetes_host="https://kubernetes.default.svc" \
   5
     kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
   6   '
   - Kết quả: Success! Data written to: auth/kubernetes/config. Lỗi permission
     denied đã được giải quyết.


  ##### Fix 2: Tạo đầy đủ các Bí mật trong Vault
  Sau khi ESO có thể đăng nhập, nó vẫn cần dữ liệu để đọc. Chúng ta đã tạo tất
  cả các bí mật còn thiếu với đầy đủ các trường (key) cần thiết.
   - Lệnh đã thực thi (ví dụ cho Kibana):


   1   kubectl exec -it -n vault vault-0 -- \
   2     vault kv put secret/tmcp/kibana \
   3       ELASTICSEARCH_HOSTS='["http://elasticsearch-service:9200"]' \
   4       XPACK_ENCRYPTEDSAVEDOBJECTS_ENCRYPTIONKEY=$(openssl rand -hex 16) \
   5       XPACK_REPORTING_ENCRYPTIONKEY=$(openssl rand -hex 16) \
   6       XPACK_SECURITY_ENCRYPTIONKEY=$(openssl rand -hex 16)
   - Kết quả: Các lệnh vault kv get sau đó đã xác nhận dữ liệu tồn tại trong
     Vault.


  ##### Fix 3: Khởi động lại External Secrets Operator
  Để buộc ESO áp dụng ngay các thay đổi và xóa bỏ trạng thái lỗi cũ, chúng ta đã
  khởi động lại pod của nó.
   - Lệnh đã thực thi:


   1   kubectl delete pod <external-secrets-pod-name> -n external-secrets
   - Kết quả: Pod mới được tạo ra và bắt đầu quá trình đối chiếu lại từ đầu.

  4. Kết quả cuối cùng: Hệ thống ổn định


  Sau khi thực hiện 3 bước khắc phục trên:
   1. Log của ESO cho thấy nó đã kết nối và xác thực với Vault thành công (store
      validated).
   2. Tiếp theo, nó đọc các ExternalSecret và đồng bộ thành công (reconciled
      secret).
   3. Kiểm tra lại trạng thái Pod, tất cả đã chuyển sang `Running`.
       - Lệnh: kubectl get pods -n tmcp-marketing
       - Kết quả: Tất cả các Pod, bao gồm cả kibana, mcp-bridge, đều có trạng
         thái READY: 1/1 và STATUS: Running.
   4. Cuối cùng, trạng thái tổng của ứng dụng trong Argo CD đã được xác nhận là
      `Healthy`.
       - Lệnh: kubectl get application tmcp-marketing -n argocd -o
         jsonpath='{.status.health.status}'
       - Kết quả: Healthy


  Hệ thống đã hoạt động ổn định trở lại vì luồng đồng bộ bí mật đã được khơi
  thông hoàn toàn: Pod > K8s Secret > ExternalSecret > Vault.

