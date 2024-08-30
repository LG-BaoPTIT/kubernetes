
### Tổng quan về xác thực (Authenticating)

Trong Kubernetes, việc xác thực (authentication) là quá trình kiểm tra danh tính của người dùng hoặc dịch vụ khi họ cố gắng truy cập vào API server. API server là thành phần trung tâm trong Kubernetes, nơi tiếp nhận và xử lý các yêu cầu từ các người dùng và dịch vụ khác nhau.

### Người dùng trong Kubernetes

Kubernetes có hai loại người dùng chính:
1. **Service Accounts**: Là những tài khoản dịch vụ được Kubernetes quản lý. Chúng được sử dụng chủ yếu bởi các ứng dụng hoặc dịch vụ chạy trong cluster.
2. **Normal Users**: Là những người dùng bình thường, chẳng hạn như quản trị viên hoặc nhà phát triển sử dụng `kubectl` từ bên ngoài cluster.

#### 1. **Normal Users (Người dùng bình thường)**
- Kubernetes không trực tiếp quản lý các tài khoản người dùng bình thường. Thay vào đó, việc quản lý người dùng này thường được thực hiện bởi các dịch vụ bên ngoài, chẳng hạn như quản lý khóa bảo mật (private keys), dịch vụ xác thực như Keystone hoặc Google Accounts, hoặc thông qua tệp tin chứa danh sách tên người dùng và mật khẩu.
- Kubernetes không có các đối tượng đại diện trực tiếp cho tài khoản người dùng bình thường. Điều này có nghĩa là bạn không thể thêm người dùng bình thường vào cluster thông qua một lệnh API của Kubernetes.

#### 2. **Xác thực qua chứng chỉ**
- Mặc dù không thể thêm người dùng bình thường thông qua lệnh API, nhưng bất kỳ người dùng nào cung cấp một chứng chỉ hợp lệ được ký bởi CA (Certificate Authority) của cluster đều được coi là đã xác thực.
- Kubernetes sẽ xác định tên người dùng từ trường "Common Name" (CN) trong chứng chỉ. Ví dụ, nếu CN trong chứng chỉ là "bob", thì Kubernetes sẽ coi người dùng là "bob".
- Sau khi xác thực, hệ thống kiểm soát truy cập dựa trên vai trò (RBAC - Role Based Access Control) sẽ xác định xem người dùng này có được phép thực hiện các thao tác cụ thể nào trên tài nguyên hay không.

#### 3. **Service Accounts**
- **Service Accounts** được quản lý bởi API server của Kubernetes và có thể được tạo tự động hoặc thủ công thông qua các lệnh API.
- Chúng gắn liền với một không gian tên (namespace) cụ thể và đi kèm với một bộ thông tin đăng nhập được lưu trữ dưới dạng Secrets.
- Những thông tin đăng nhập này thường được gắn vào các pods để các quy trình trong cluster có thể giao tiếp với API server của Kubernetes.

### Yêu cầu API và Xác thực
- Mọi yêu cầu API đến từ người dùng bình thường hoặc Service Accounts đều phải được xác thực.
- Nếu một yêu cầu không được xác thực, nó sẽ được coi là một yêu cầu ẩn danh (anonymous).


### Các chiến lược xác thực trong Kubernetes

Kubernetes sử dụng các chứng chỉ client (client certificates), token, hoặc proxy xác thực (authenticating proxy) để xác thực các yêu cầu API thông qua các plugin xác thực. Khi một yêu cầu HTTP được gửi đến API server, các plugin này sẽ cố gắng liên kết các thuộc tính sau với yêu cầu:

- **Username**: Một chuỗi xác định người dùng cuối (end user). Ví dụ như `kube-admin` hoặc `jane@example.com`.
- **UID**: Một chuỗi khác xác định người dùng cuối, cố gắng duy trì tính nhất quán và duy nhất hơn so với `Username`.
- **Groups**: Một tập hợp các chuỗi, mỗi chuỗi biểu thị một nhóm mà người dùng thuộc về. Ví dụ như `system:masters` hoặc `devops-team`.
- **Extra fields**: Một bản đồ chứa thông tin bổ sung mà các thành phần kiểm tra quyền truy cập (authorizers) có thể thấy hữu ích.

Các giá trị này chỉ có ý nghĩa khi được giải thích bởi hệ thống phân quyền.

### Kích hoạt nhiều phương thức xác thực

Bạn có thể kích hoạt nhiều phương thức xác thực cùng lúc. Thường thì bạn nên sử dụng ít nhất hai phương thức:

- **Service account tokens**: Dùng để xác thực các service accounts.
- **Một phương thức khác**: Dành cho xác thực người dùng (user authentication).

Khi nhiều module xác thực được kích hoạt, module đầu tiên xác thực thành công yêu cầu sẽ kết thúc quá trình kiểm tra. API server không đảm bảo thứ tự chạy của các module xác thực.

Tất cả các người dùng được xác thực sẽ tự động được thêm vào nhóm `system:authenticated`.

### Tích hợp với các giao thức xác thực khác

Kubernetes có thể tích hợp với các giao thức xác thực khác như LDAP, SAML, Kerberos, hoặc các x509 schemes khác thông qua một proxy xác thực hoặc webhook xác thực.

### Chứng chỉ client X509

Kubernetes hỗ trợ xác thực thông qua chứng chỉ client (client certificate) bằng cách cung cấp tùy chọn `--client-ca-file=SOMEFILE` cho API server. Tệp này phải chứa một hoặc nhiều chứng chỉ CA để xác thực các chứng chỉ client được trình bày với API server.

- Nếu một chứng chỉ client được trình bày và xác minh, tên thông thường (Common Name) của chủ thể sẽ được sử dụng làm tên người dùng cho yêu cầu.
- Bắt đầu từ Kubernetes 1.4, chứng chỉ client cũng có thể chỉ ra các nhóm (group) mà người dùng thuộc về thông qua các trường tổ chức (organization fields) trong chứng chỉ.

Ví dụ, để tạo một yêu cầu ký chứng chỉ (CSR) với OpenSSL:

```bash
openssl req -new -key jbeda.pem -out jbeda-csr.pem -subj "/CN=jbeda/O=app1/O=app2"
```

Câu lệnh này sẽ tạo một CSR cho tên người dùng `jbeda`, thuộc về hai nhóm `app1` và `app2`.

### Kết luận

Các phương thức xác thực trong Kubernetes rất linh hoạt và có thể được tùy chỉnh để phù hợp với nhiều mô hình bảo mật khác nhau, tùy thuộc vào yêu cầu của hệ thống. Bạn có thể kết hợp nhiều phương thức để đảm bảo tính bảo mật cao nhất cho cụm Kubernetes của mình.