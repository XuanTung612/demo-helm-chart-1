# Demo helm chart

Project demo cách sử dụng helm chart

## Các cú pháp cơ bản:

Thay thế giá trị bằng `{{ }}`:

```yaml
# values.yaml
nameOverride: xxx

# templates/deployment.yaml
metadata:
  name: {{ .Values.nameOverride }}

# Result
metadata:
  name: xxx
```

Thay thế giá trị và loại bỏ khoảng trắng

```yaml
# values.yaml
nameOverride: xxx

# templates/deployment.yaml
# Lưu ý ở đây chỉ loại bỏ khoảng trắng và xuống dòng ở đầu
metadata:
  name: 
    {{- .Values.nameOverride }}
spec:

# Result
metadata:
  name: xxx
spec:
```

**With**: Tạo scope nhỏ

```yaml
# values.yaml
image: 
    repository: minhpq331/demo-deployment
    tag: v1.0
    pullPolicy: Always

# templates/deployment.yaml
containers:
    - name: web
    {{- with .Values.image  }}
      image: "{{ .repository }}:{{ .tag }}"
      imagePullPolicy: {{ .pullPolicy }}
    {{- end }}

# Result
containers:
    - name: web
      image: "minhpq331/demo-deployment:v1.0"
      imagePullPolicy: Always
```

**If else**: check điều kiện và rẽ nhánh

```yaml
# values.yaml
service:
    enableHttp: true
    enableHttps: false

# templates/deployment.yaml
ports:
{{- if .Values.service.enableHttp }}
- port: 80
  targetPort: http
  protocol: TCP
  name: http
{{- end }}
{{- if .Values.service.enableHttps }}
- port: 443
  targetPort: https
  protocol: TCP
  name: https
{{- end }}

# Result
ports:
- port: 80
  targetPort: http
  protocol: TCP
  name: http
```

**Range**: tạo vòng lặp

```yaml
# values.yaml
env: 
    PORT: "3000"
    MESSAGE: "Hello world"

# templates/deployment.yaml
containers:
    - name: web
      env:
      {{- range $key, $val := .Values.env }}
      - name: {{ $key | quote }}
        value: {{ $val | quote }}
      {{- end }}

# Result
containers:
    - name: web
      env:
      - name: "PORT"
        value: "3000"
      - name: "MESSAGE"
        value: "Hello world"
```

## 1 số hàm thường dùng:

**default**: lấy giá trị mặc định

```yaml
# values.yaml
image: 
    repository: minhpq331/demo-deployment
    tag: "v1.0"
    pullPolicy: ""

# templates/deployment.yaml
containers:
    - name: web
      image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
      imagePullPolicy: {{ .Values.image.pullPolicy | default "Always" }}

# Result
containers:
    - name: web
      image: "minhpq331/demo-deployment:v1.0"
      imagePullPolicy: Always
```

**quote**: Đóng nháy

```yaml
# values.yaml
env: 
    PORT: "3000"
    SOCKET_PORT: "3001"

# templates/deployment.yaml
containers:
    - name: web
      env:
      - name: PORT
        value: {{ .Values.env.PORT }}
      - name: SOCKET_PORT
        value: {{ .Values.env.SOCKET_PORT | quote }}

# Result
containers:
    - name: web
      env:
      - name: PORT
        value: 3000             # Cause error
      - name: SOCKET_PORT
        value: "3001"
```

**upper, lower**: biến đổi case

```yaml
# values.yaml
env: 
    HI: "Hi There"
    HELLO: "Hello world"

# templates/deployment.yaml
containers:
    - name: web
      env:
      - name: HI
        value: {{ .Values.env.HI | lower | quote }}
      - name: HELLO
        value: {{ .Values.env.HELLO | upper | quote }}

# Result
containers:
    - name: web
      env:
      - name: HI
        value: "hi there"
      - name: HELLO
        value: "HELLO WORLD"
```

**toYaml**: Include 1 đoạn yaml (lưu ý indent)

```yaml
# values.yaml
labels:
  abc: "123"
  xyz: "456"
ports:
  - 80
  - 443

# templates/deployment.yaml
metadata:
  labels:
{{ toYaml .Values.labels | indent 4 }}
spec:
  containers:
    - name: web
      ports: 
{{ toYaml .Values.ports | indent 8 }}

# Result
metadata:
  labels:
    abc: "123"
    xyz: "456"
spec:
  containers:
    - name: web
      ports:
        - 80
        - 443
```

**indent, nindent**: lùi bao nhiêu khoảng trắng

```yaml 
# values.yaml
labels:
  abc: "123"
  xyz: "456"
annotations:
  hello: "world"
  oh: "my god"

# templates/deployment.yaml
metadata:
  labels:
{{ toYaml .Values.labels | indent 4 }}
  annotations: {{ toYaml .Values.annotations | nindent 4 }}

# Result
metadata:
  labels:
    abc: "123"
    xyz: "456"
  annotations:
    hello: "world"
    oh: "my god"
```

**include**: Include 1 template trong `_helpers.tpl`

```yaml 
# values.yaml
nameOverride: ahihi

# templates/_helpers.tpl
{{- define "demo-chart.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" -}}
{{- end -}}

# templates/deployment.yaml
metadata:
  name: {{ include "demo-chart.name" . }}

# Result
metadata:
  name: ahihi
```

## Các lệnh tương tác với helm 3

```bash
# List all release trong cluster
helm ls

# Install 1 chart vào cluster (sẽ lỗi nếu release đã tồn tại)
helm install -n <ns> <release-name> <chart-name>

# Uninstall 1 chart khỏi cluster 
helm uninstall -n <ns> <release-name>

# Update hoặc install nếu chưa tồn tại 1 chart vào cluster
helm upgrade -n <ns> --install -f <custom-values.yaml> --set <key.prop>=<value> <release-name> <chart-name>

# Thêm helm repo 
helm repo add <name> <url>

# Debug template và check output yaml
helm upgrade ... --dry-run --debug
```

## Thực hành 1: Deploy nginx ingress controller

Nginx ingress controller là loại ingress controller thường được sử dụng nhất. Để cài đặt bằng helm, bạn hãy tìm hiểu file `values.yaml` đầy đủ tại: [https://github.com/kubernetes/ingress-nginx/blob/master/charts/ingress-nginx/values.yaml](https://github.com/kubernetes/ingress-nginx/blob/master/charts/ingress-nginx/values.yaml)

Các bước thực hiện:

- Trong file `nginx-values.yaml` chứa custom values để triển khai nginx-ingress cho 1 namespace
- Sửa giá trị `nodePort` để tránh trùng với người khác
- Chạy lệnh sau để cài đặt:
```bash
# Update repo
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Install nginx ingress
helm upgrade -n <ns> --install -f nginx-values.yaml nginx-ingress ingress-nginx/ingress-nginx
```

- Quay trở lại bài thực hành số 3 của demo-service và triển khai thành công ingress cho ứng dụng NodeJS

## Thực hành 2: Đóng gói ứng dụng vào helm chart

- Copy file `deployment.yaml`, `service.yaml`, `ingress.yaml` từ bài demo-service vào thư mục `demo-chart/templates`
- Thay các giá trị image, tên deployment, tên service, port, biến môi trường bằng template và sửa lại trong `values.yaml`
- Cài đặt app bằng lệnh 

```bash
helm upgrade -n <ns> --install node-app ./demo-chart
```

- Kiểm tra ứng dụng đã chạy thành công chưa bằng các phương pháp đã học từ bài trước.