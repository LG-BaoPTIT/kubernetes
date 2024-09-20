# Normal user

## Instructions

### *Method for submitting CSR to Kubernetes

#### 1. Create private key and certificate signing request

``` shell
openssl genrsa -out "private.key" 2048

```
```shell
openssl req -new -key private.key -out private.csr -subj "/CN=baolg/O=devops"
```
Lệnh trên sẽ tạo một yêu cầu chứng chỉ (CSR) với các thông tin được chỉ định.
Trong lệnh `openssl req -new -key private.key -out private.csr -subj "/CN=baolg/O=devops"`, có thể chỉ định nhiều trường khác nhau trong phần `-subj`. Dưới đây là các trường phổ biến có thể sử dụng:

- **CN (Common Name):** Tên thông thường, thường là tên miền hoặc tên người dùng.
- **C (Country):** Quốc gia, sử dụng mã quốc gia ISO 2 ký tự, ví dụ: `C=VN` cho Hoa Kỳ.
- **ST (State or Province Name):** Bang hoặc tỉnh, ví dụ: `ST=Thai Nguyen`.
- **L (Locality Name):** Thành phố hoặc địa phương, ví dụ: `L=Pho Yen city`.
- **O (Organization Name):** Tên tổ chức, ví dụ: `O=ITE`.
- **OU (Organizational Unit Name):** Đơn vị trực thuộc của tổ chức, ví dụ: `OU=IT Department`.
- **emailAddress:** Địa chỉ email, ví dụ: `emailAddress=baolg@itegroup.vn`.


#### 2. Create CSR resource in k8s
You need to encode your CSR in base64 so that it can be inserted into the YAML file.

``` shell
cat private.csr | base64 | tr -d '\n'
```

```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: baolg
spec:
  groups:
    - system:authenticated
  request: base64 of file .csr
  signerName: kubernetes.io/kube-apiserver-client
  usages:
    - Client auth
  expirationSeconds: 86400 #expire after 24 hours


```
```shell
cat << EOF > csr.yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: baolg
spec:
  groups:
    - system:authenticated
  request: $(cat private.csr | base64 | tr -d '\n')
  signerName: kubernetes.io/kube-apiserver-client
  usages:
    - client auth
  expirationSeconds: 86400 #expire after 24 hours
EOF
```

Trong cấu trúc `CertificateSigningRequest` (CSR) của Kubernetes, phần `spec` có các trường chính như `groups`, `request`, `signerName`, và `usages`. Mỗi trường có các giá trị tùy chọn (options) cụ thể. Dưới đây là các lựa chọn phổ biến cho từng trường trong `spec`:

### 1. **groups**
`groups` là danh sách các nhóm mà người dùng yêu cầu chứng chỉ thuộc về. Các nhóm này xác định quyền hạn và vai trò của người dùng trong Kubernetes. Một số nhóm phổ biến là:

- **`system:authenticated`**: Nhóm này bao gồm tất cả người dùng đã được xác thực.
- **`system:serviceaccounts`**: Nhóm này dành cho tất cả các tài khoản dịch vụ trong cluster.
- **`system:masters`**: Nhóm này thường được dành cho quản trị viên cluster, có quyền quản lý cao nhất.

### 2. **request**
`request` là chuỗi CSR được mã hóa base64. Để tạo chuỗi này, bạn cần tạo một CSR bằng cách sử dụng công cụ như `openssl`, sau đó mã hóa nội dung CSR đó thành base64. Đây là yêu cầu bắt buộc và không có nhiều tùy chọn khác.

### 3. **signerName**
`signerName` xác định ai là người ký (signer) chứng chỉ này. Một số tùy chọn phổ biến:

- **`kubernetes.io/kube-apiserver-client`**: Chứng chỉ dành cho client (người dùng hoặc ứng dụng) sử dụng API server của Kubernetes.
- **`kubernetes.io/kube-apiserver-client-kubelet`**: Chứng chỉ dành cho Kubelet để giao tiếp với API server.
- **`kubernetes.io/kubelet-serving`**: Chứng chỉ dành cho Kubelet phục vụ API (thường là HTTPS).
- **`kubernetes.io/legacy-unknown`**: Chứng chỉ được ký bởi API server nhưng không có kiểm tra sử dụng cụ thể. Loại này ít được khuyến khích sử dụng.

### 4. **usages**
`usages` xác định mục đích sử dụng của chứng chỉ. Một CSR có thể có nhiều `usages`. Các tùy chọn phổ biến bao gồm:

- **`client auth`**: Chứng chỉ này được sử dụng để xác thực client với server (thường là Kubernetes API server).
- **`server auth`**: Chứng chỉ này được sử dụng để xác thực server với client.
- **`digital signature`**: Chứng chỉ này có thể được sử dụng để ký số dữ liệu.
- **`key encipherment`**: Chứng chỉ này được sử dụng để mã hóa khóa phiên (session key).
- **`key agreement`**: Chứng chỉ này được sử dụng để đồng ý khóa giữa các bên.
- **`data encipherment`**: Chứng chỉ này được sử dụng để mã hóa dữ liệu.

### 5. **expirationSeconds** (tùy chọn)
`expirationSeconds` xác định thời gian hết hạn của chứng chỉ sau khi được cấp (tính bằng giây). Nếu không được đặt, Kubernetes sử dụng giá trị mặc định là 1 năm (31536000 giây).

```yaml
spec:
  expirationSeconds: 86400  # Chứng chỉ hết hạn sau 24 giờ
```

#### 3. Send CSR to k8s
```shell
kubectl apply -f csr.yaml
```
OR
```shell
kubectl create -f csr.yaml
```
Difference between 2 commands: `search google ok` 

Command get csr: `kubectl get csr`
#### 4. K8s approve CSR
```shell
kubectl certificate approve baolg
```

#### 5. Create file config
```shell
kubectl get csr baolg -o jsonpath='{.status.certificate}' | base64 --decode 

```

```shell
kubectl config view --raw -o yaml | yq eval '.users[].user."client-certificate-data" = "" | .users[].user."client-key-data" = ""' - > config.sh

kubectl config view --raw -o yaml > config.sh

kubectl config view --raw -o yaml | yq eval '
   .users[] |= ( .user.client-certificate-data = null |
                 .user.client-key-data = null )' - > config.sh

kubectl config view --raw -o yaml | yq eval '
  .users[] |= (
    select(.name == "baolg") |=
    .user.client-certificate-data = null |
    .user.client-key-data = null
  )' - > config.sh


```
Let's edit file config.sh

```bash
#!/bin/bash
cat << EOF > config
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURCVENDQWUyZ0F3SUJBZ0lJSWxONGJGTm4yVVl3RFFZSktvWklodmNOQVFFTEJRQXdGVEVUTUJFR0ExVUUKQXhNS2EzVmlaWEp1WlhSbGN6QWVGdzB5TkRBNE1qY3dPREkyTWpsYUZ3MHpOREE0TWpVd09ETXhNamxhTUJVeApFekFSQmdOVkJBTVRDbXQxWW1WeWJtVjBaWE13Z2dFaU1BMEdDU3FHU0liM0RRRUJBUVVBQTRJQkR3QXdnZ0VLCkFvSUJBUUNsVjhtZ04ySlk0M1dKc2ozY3ZjTEd0QUlsVTZoTkNON0w4N0ZscGVHUkV5QjA0alVEV2pjWXo3bGoKSlFYNytUd2kxcWhpVEtsQkRFMHI5VEh1blZ1b1BOeEgzamdyOW5HbldHbDdZb0hnUmZDelJHb1ZQd2k3K3ZGbApHU1QvaWVLa01hdU90czZTSE9kVzBrakZMSmNKeC9Fa05uaGVWWlA0TkRaamNZM1hXTWVlWEJ6SjZRaHFUZnozCm9qY0hHY0xUcXRZSkxWTFBTZnRjMXFQMjA3S01kNVVRS0ZObE9rSlJQWFpLWWRMMEVUVG1YaFJhbGRQQ05zK3EKc3dOeVYvN2pkUTBGVFFJRGRtZUprYUdiK000d3RvVitJMWJKTzZIQm9xZEY1K1VlYi9CN0c4VEVCaGc0Y3oreApjOENWVndOTmcxNmNZaHR3R0t1M3AxYkhGR0piQWdNQkFBR2pXVEJYTUE0R0ExVWREd0VCL3dRRUF3SUNwREFQCkJnTlZIUk1CQWY4RUJUQURBUUgvTUIwR0ExVWREZ1FXQkJTMEJGSzZINHJDT3VJVWdZUnZmQmZFQ0UyT2V6QVYKQmdOVkhSRUVEakFNZ2dwcmRXSmxjbTVsZEdWek1BMEdDU3FHU0liM0RRRUJDd1VBQTRJQkFRQXh4dnBGYnFOMgp6cFhobnVabHRCMnhOSFhxUXRWbDdGTzdGRUY3cTdiUUE3WVJwZVBJUGF5UlBVZlZRbkxadldVa2dBUHpxc3IwClI2Q0ZXV0QxWFY5cmdXaENYVWo4QTZjRU9xTm5wMEgrUTl2TUZKcE45cjRRT2dPZFN0OXZXSE1CeGVkUHJtZjMKOEFQODFHUFl5YXRHTSsyaTFDVUdkMFJHbUcyd3FQVmNhakRYWXJGV2RoTmhYajRKZUlkbmJCaUhlbDV4WHpMMgpUTWYyZmVrSHFuTVVFaVpRTitKOEpPcUtwWmh3aW9UVDlUWk5jWlJnVmdIOExYWFcwenhwd28yQzlpVG1vV0pHCkVsaWN5Wkd6dDZxUWNSM2xNRUN4aGY2THJhYXVQNDNxN2tuQTZLekEzbnZ2eGg4RDhkaDF0OXR3MXphMUlVeVgKQ0RIY3RZcmZPSzZXCi0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K
    server: https://192.168.56.11:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: baolg
  name: baolg-context
current-context: baolg-context
kind: Config
preferences: {}
users:
- name: baolg
  user:
    client-certificate-data: $(kubectl get csr baolg -o jsonpath='{.status.certificate}')
    client-key-data: $(cat private.key | base64 | tr -d '\n' )
EOF
```
#### 6.Config role RBAC

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: readOnly
rules:
  - apiGroups: [""]
    resources: ["*"]
    verbs:
      - get
      - watch
      - list

---

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: readOnly-binding
subjects:
  - kind: User
    name: baolg
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: readOnly
  apiGroup: rbac.authorization.k8s.io


```
```shell
export KUBECONFIG=config
```
