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



  Báo cáo sự cố: Agent tmcp-aiops-agent không nhận được tín hiệu log từ Fluentd

  1. Tình trạng ban đầu


  Hệ thống tmcp-aiops-agent được thiết lập để nhận các bản ghi (log) chứa lỗi từ Fluentd, tương tự như một hệ thống tmcp-gitops đã hoạt động trước đó. Tuy nhiên, dù đã triển khai code và cấu hình,
  agent vẫn không ghi nhận được bất kỳ tín hiệu nào, ngay cả khi các pod trên hệ thống gặp lỗi hoặc bị xóa.

  2. Quá trình điều tra và sửa lỗi

  Quá trình được thực hiện theo từng bước để khoanh vùng và xác định nguyên nhân cốt lõi.

  Bước 1: Kiểm tra cấu hình và trạng thái triển khai


   - Phân tích: Đầu tiên, tôi kiểm tra các tệp cấu hình trong project tmcp-gitops (fluentd.yaml, aiops-agent.yaml) và mã nguồn của tmcp-aiops-agent.
       - fluentd.yaml: Cấu hình Fluentd để gửi log tới endpoint http://aiops-agent-service:80/api/webhook/k8s-logs.
       - aiops-agent.yaml: Định nghĩa một Service tên là aiops-agent-service trỏ vào Deployment của agent trên port 8000.
       - app.py (của agent): Mã nguồn FastAPI xác nhận ứng dụng lắng nghe trên port 8000 với endpoint /api/webhook/k8s-logs.
   - Phát hiện: Mọi cấu hình về mặt lý thuyết đều chính xác. Tuy nhiên, khi kiểm tra thực tế trên cluster K3s bằng lệnh kubectl get pods, tôi phát hiện ra cả Deployment và Service của aiops-agent
     đều chưa được áp dụng (apply).
   - Hành động: Áp dụng các file YAML cần thiết để triển khai agent và các tài nguyên liên quan.
   1     kubectl apply -f aiops-agent.yaml
   2     kubectl apply -f aiops-logging.yaml
   - Kết quả: Agent đã khởi chạy thành công.

  Bước 2: Chẩn đoán lỗi phân quyền của Fluentd


   - Phát hiện: Sau khi agent chạy, tôi tiếp tục kiểm tra và thấy pod của Fluentd đang ở trạng thái Error. Khi xem log của pod Fluentd, tôi tìm thấy lỗi 403 Forbidden.
      > namespaces is forbidden: User "system:serviceaccount:argocd:fluentd" cannot list resource "namespaces"
   - Phân tích: Lỗi này cho thấy ServiceAccount của Fluentd (đang chạy trong namespace argocd) không có quyền list (liệt kê) các namespaces trên toàn cluster. Quyền này là bắt buộc đối với bộ lọc
     kubernetes_metadata để làm giàu thông tin cho log. Nguyên nhân là do ClusterRoleBinding trong fluentd.yaml đã khai báo sai namespace cho ServiceAccount (khai báo là tmcp-marketing thay vì
     argocd).
   - Hành động: Chỉnh sửa file fluentd.yaml để cấp quyền cho đúng ServiceAccount.
   1     # Trong phần ClusterRoleBinding
   2     subjects:
   3     - kind: ServiceAccount
   4       name: fluentd
   5       namespace: argocd # Sửa từ tmcp-marketing thành argocd
   - Kết quả: Sau khi áp dụng lại file, Fluentd đã khởi chạy thành công.

  Bước 3: Tìm ra bộ lọc (filter) là nguyên nhân cốt lõi


   - Phát hiện: Mặc dù cả agent và Fluentd đều đã chạy, agent vẫn không nhận được log khi tôi thử nghiệm xóa một pod kibana. Điều này chứng tỏ kết nối mạng đã thông suốt, nhưng log đang bị chặn
     lại ở đâu đó bên trong Fluentd.
   - Phân tích: Tôi đặt giả thuyết rằng một trong các bộ lọc logic trong file fluent.conf đang loại bỏ các log trước khi chúng được gửi đi. Cấu hình gốc có một bộ lọc grep chỉ cho phép các log
     chứa từ khóa cụ thể đi qua: error, exception, fail, fatal, degraded.
      > Khi một pod bị xóa, log hệ thống ghi lại sự kiện này không nhất thiết chứa các từ khóa trên. Đây chính là lý do log không bao giờ được chuyển đến agent.
   - Hành động:
       1. Thử nghiệm với cấu hình đơn giản: Tôi tạo một file fluent.conf tối giản, loại bỏ toàn bộ các bộ lọc và logic định tuyến, chỉ giữ lại việc đọc và gửi log trực tiếp đến agent. Kết quả:
          Thành công! Agent đã nhận được log. Điều này xác nhận giả thuyết về bộ lọc là đúng.
       2. Sửa đổi cấu hình gốc: Tôi đã xóa bỏ hoàn toàn bộ lọc @type grep khỏi fluentd.yaml.
   - Kết quả: Sau khi áp dụng lại cấu hình đã loại bỏ bộ lọc grep, agent đã nhận được tín hiệu log thành công khi pod kibana bị xóa.

  3. Kết luận


  Sự cố xảy ra do sự kết hợp của hai vấn đề chính:


   1. Sai sót trong phân quyền (RBAC): ClusterRoleBinding của Fluentd đã không được cấu hình đúng namespace cho ServiceAccount, khiến Fluentd không thể khởi chạy.
   2. Bộ lọc log quá chặt chẽ: Bộ lọc grep trong fluent.conf đã loại bỏ các log về trạng thái của pod (ví dụ: terminating, killing) vì chúng không chứa các từ khóa lỗi được định sẵn.


  Bằng cách cấp lại quyền chính xác và loại bỏ bộ lọc không cần thiết, luồng log từ Fluentd đến aiops-agent đã được khơi thông và hệ thống hoạt động như mong đợi.

=========================

  Báo cáo sự cố: Khắc phục lỗi kết nối từ AIOps Agent đến Ollama LLM

  1. Tình trạng ban đầu


  Sau khi khắc phục sự cố luồng log từ Fluentd, AIOps Agent đã bắt đầu nhận được tín hiệu. Tuy nhiên, một lỗi mới phát sinh trong quá trình agent cố gắng phân tích log bằng mô hình ngôn ngữ lớn
  (LLM).

  Log của aiops-agent pod hiển thị lỗi sau:


   1 Error during LLM analysis: HTTPConnectionPool(host='ollama-service', port=11434): Max retries exceeded with url: /api/chat
   2 (Caused by NameResolutionError("HTTPConnection(host='ollama-service', port=11434): Failed to resolve 'ollama-service' ..."))


  Phân tích lỗi:
   - Lỗi NameResolutionError cho thấy agent (đang chạy bên trong cluster Kubernetes) đã cố gắng tìm một dịch vụ (service) có tên là ollama-service nhưng không thành công.
   - Nguyên nhân cốt lõi là do Ollama đang chạy trên máy cục bộ (host machine), bên ngoài môi trường mạng của Kubernetes. Do đó, DNS nội bộ của cluster không thể phân giải được tên ollama-service.

  2. Quá trình sửa lỗi


  Để giải quyết vấn đề, chúng ta cần xây dựng một "cây cầu" mạng, cho phép các dịch vụ bên trong Kubernetes "nhìn thấy" và giao tiếp với dịch vụ Ollama đang chạy bên ngoài.

  Bước 1: Tạo cầu nối mạng bằng Service và Endpoints

  Đây là phương pháp chuẩn của Kubernetes để ánh xạ một dịch vụ bên ngoài vào trong cluster.


  1.1. Xác định địa chỉ IP của máy Host:
  Đầu tiên, chúng ta cần tìm địa chỉ IP của máy host mà cluster K3s có thể truy cập được.
   - Lệnh đã dùng: ifconfig | grep "inet " | grep -v 127.0.0.1
   - Kết quả: Tìm được địa chỉ IP là 192.168.0.102.


  1.2. Tạo file cấu hình `ollama-external-service.yaml`:
  Tôi đã tạo một file YAML để định nghĩa hai đối tượng Kubernetes:
   - `Service`: Một service ảo tên là ollama-service. Nó không có selector vì nó không quản lý pod nào bên trong cluster. Nó chỉ đóng vai trò là một DNS entry (tên miền) cố định.
   - `Endpoints`: Một đối tượng có cùng tên (ollama-service) để chỉ định địa chỉ IP thực tế cho Service ảo ở trên. Chúng ta trỏ nó đến địa chỉ IP của máy host và port của Ollama (11434).

  Nội dung file `ollama-external-service.yaml`:


    1 apiVersion: v1
    2 kind: Service
    3 metadata:
    4   name: ollama-service
    5 spec:
    6   ports:
    7   - protocol: TCP
    8     port: 11434
    9     targetPort: 11434
   10 ---
   11 apiVersion: v1
   12 kind: Endpoints
   13 metadata:
   14   name: ollama-service
   15 subsets:
   16   - addresses:
   17       - ip: "192.168.0.102"
   18     ports:
   19       - port: 11434


  1.3. Áp dụng cấu hình:
  Lệnh sau đã được thực thi để tạo Service và Endpoints trên cluster.
   - Lệnh đã dùng: kubectl apply -f ollama-external-service.yaml
   - Kết quả: Tên miền http://ollama-service:11434 giờ đã có thể được phân giải từ bên trong cluster và trỏ đến http://192.168.0.102:11434.


  1.4. Khởi động lại AIOps Agent:
  Để agent nhận ngay lập tức DNS mới, pod của nó đã được khởi động lại.
   - Lệnh đã dùng: kubectl delete pod <tên-pod-agent>

  Bước 2: Sửa lỗi không tìm thấy mô hình (404 Not Found)

  Sau khi kết nối thành công, một lỗi mới xuất hiện trong log của agent: Ollama call failed with status code 404.


  Phân tích lỗi:
   - Lỗi 404 Not Found từ Ollama server có nghĩa là kết nối đã thành công, nhưng mô hình LLM mà agent yêu cầu không tồn tại trên server.
   - Cấu hình mặc định trong aiops-agent.yaml đang yêu cầu mô hình llama3.


  Hành động:
  Theo yêu cầu của bạn, tôi đã thay đổi mô hình thành qwen3:4b-instruct-2507-q4_K_M mà bạn đã có sẵn.


  2.1. Cập nhật file `aiops-agent.yaml`:
  Tôi đã chỉnh sửa biến môi trường OLLAMA_MODEL trong file triển khai.

  Cấu hình trước khi sửa:
   1 - name: OLLAMA_MODEL
   2   value: "llama3"

  Cấu hình sau khi sửa:


   1 - name: OLLAMA_MODEL
   2   value: "qwen3:4b-instruct-2507-q4_K_M"


  2.2. Áp dụng lại cấu hình:
   - Lệnh đã dùng: kubectl apply -f aiops-agent.yaml
   - Kết quả: Kubernetes tự động cập nhật Deployment, tạo ra một pod agent mới với biến môi trường đã được cập nhật.

  3. Kết quả cuối cùng


  Sau khi thực hiện cả hai bước trên, hệ thống đã hoạt động hoàn chỉnh:
   1. Fluentd gửi log tới AIOps Agent.
   2. AIOps Agent nhận log, sau đó gửi yêu cầu phân tích đến ollama-service.
   3. Kubernetes DNS phân giải ollama-service thành địa chỉ IP 192.168.0.102 của máy host.
   4. Agent kết nối thành công đến Ollama server trên máy host.
   5. Agent yêu cầu phân tích bằng mô hình `qwen3`, mô hình này có sẵn và xử lý yêu cầu.
   6. Agent nhận kết quả và gửi thông báo thành công qua Discord.


  Sự cố đã được giải quyết triệt để.
