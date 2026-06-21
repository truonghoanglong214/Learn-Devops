# Phase 11 — Kubernetes

## Mục tiêu
- Chạy ShopLite trên cluster Kubernetes — kỹ năng được tuyển dụng săn đón nhất.
- Hiểu cách K8s tự phục hồi, scale và quản lý ứng dụng container ở quy mô lớn.

---

## Kiến thức sẽ học

- **Vì sao cần orchestration & kiến trúc K8s** (control plane, node, kubelet, etcd).
- **Đối tượng cốt lõi:** Pod, ReplicaSet, **Deployment**, Service, Namespace.
- **ConfigMap & Secret** — cấu hình và bí mật.
- **Ingress & Ingress Controller** — thay vai trò reverse proxy của Nginx ở quy mô cluster.
- **Volume & PersistentVolume/PVC** — lưu dữ liệu bền vững.
- **Health probe (liveness/readiness), resource limit** — vận hành ổn định.
- **Scaling (HPA) & self-healing** — tự động co giãn, tự phục hồi.
- **Helm** (chart, values, template) — đóng gói và triển khai ứng dụng K8s.
- **Managed K8s (EKS/GKE)** vs local (kind/minikube/k3s).

---

## Kiến thức chi tiết

### 1. Tại sao K8s sau Docker Compose

#### Giới hạn của Docker Compose

Docker Compose là công cụ tuyệt vời cho môi trường phát triển local, nhưng khi đưa lên production, nó bộc lộ nhiều giới hạn nghiêm trọng:

**Single host (chỉ chạy trên một máy):**
- Toàn bộ ứng dụng bị giam cầm trên một server duy nhất.
- Nếu server đó chết, toàn bộ hệ thống chết theo — không có failover tự động.
- Không thể phân tán workload ra nhiều máy để tận dụng tài nguyên hiệu quả.
- Giới hạn bởi RAM, CPU, disk của một máy vật lý.

**Manual scaling (scale thủ công):**
- Khi traffic tăng đột biến lúc 2 giờ sáng, bạn phải thức dậy chạy lệnh scale thủ công.
- Docker Compose không có cơ chế tự động theo dõi CPU/memory rồi scale lên hoặc down.
- Khi scale, bạn phải tự cân bằng tải (load balance) giữa các container — không có built-in support.

**No self-healing (không tự phục hồi):**
- Nếu một container crash, Docker Compose chỉ restart nếu bạn cấu hình `restart: always`.
- Không có health check thông minh để phân biệt container đang chạy nhưng không healthy (ví dụ: deadlock, OOM).
- Không tự phát hiện và replace container bị lỗi một cách có hệ thống.

**No rolling updates (không cập nhật cuốn chiếu):**
- Khi deploy phiên bản mới, Docker Compose dừng container cũ rồi khởi động container mới — downtime không tránh khỏi.
- Không có cơ chế rollback tự động nếu phiên bản mới lỗi.
- Không thể kiểm soát tốc độ deploy (bao nhiêu container được update cùng lúc).

#### Kubernetes giải quyết như thế nào

**Multi-host cluster:**
- K8s quản lý một pool các máy (nodes), phân phối workload tự động.
- Nếu một node chết, K8s tự di chuyển pods sang node khác.
- Có thể thêm node vào cluster mà không cần downtime (horizontal scaling của infrastructure).

**Auto-scaling:**
- HPA (Horizontal Pod Autoscaler) tự động tăng/giảm số lượng pods dựa trên CPU, memory hoặc custom metrics.
- Cluster Autoscaler tự động thêm/xóa nodes dựa trên nhu cầu của pods.
- VPA (Vertical Pod Autoscaler) tự điều chỉnh resource requests/limits.

**Self-healing:**
- Kubelet liên tục kiểm tra health của containers qua liveness/readiness probes.
- Nếu container không healthy, K8s tự restart hoặc thay thế.
- ReplicaSet đảm bảo luôn có đúng số lượng pods running. Xóa 1 pod → K8s tạo ngay pod mới.

**Rolling updates:**
- Deployment resource hỗ trợ rolling update: thay thế dần từng pod, đảm bảo ứng dụng luôn có pods available.
- `maxUnavailable: 0` đảm bảo zero downtime.
- Rollback về phiên bản cũ chỉ với một lệnh: `kubectl rollout undo`.

**Service discovery:**
- Mỗi Service có DNS name ổn định: `service-name.namespace.svc.cluster.local`.
- Không cần hard-code IP address. Pods có thể scale lên/down, IP thay đổi, nhưng DNS luôn ổn định.
- kube-proxy tự động cập nhật routing rules khi pods thay đổi.

#### Trade-off: Phức tạp hơn nhiều

Kubernetes không phải silver bullet. Đổi lại những lợi ích trên, bạn phải chấp nhận:

- **Độ phức tạp cao:** Hàng chục resource types, hàng trăm cấu hình options. Learning curve dốc.
- **Overhead hạ tầng:** Control plane cần ít nhất 3 nodes cho HA, mỗi node cần RAM để chạy kubelet, kube-proxy, v.v.
- **Debugging khó hơn:** Thay vì `docker logs`, bạn cần biết `kubectl logs`, `kubectl describe`, `kubectl exec`, đọc Events.
- **Networking phức tạp:** CNI plugins, Network Policies, Ingress Controllers — nhiều tầng abstraction.
- **Chi phí cao hơn:** EKS tốn tiền cho control plane ($0.10/hour), cộng thêm worker nodes.
- **Stateful workloads vẫn khó:** Databases trên K8s cần StatefulSets, PVCs, careful backup strategies.

#### Khi nào nên dùng K8s

**Nên dùng K8s khi:**
- Production workloads cần High Availability (HA) thực sự.
- Ứng dụng cần scale tự động theo traffic.
- Team DevOps đủ lớn (ít nhất 2-3 người hiểu K8s) để vận hành.
- Microservices architecture với nhiều services cần orchestration.
- Cần multi-region deployment, disaster recovery.
- Compliance yêu cầu isolation và security nghiêm ngặt.

**Không nên dùng K8s khi:**
- Startup nhỏ, ít traffic, team 1-2 người.
- Ứng dụng đơn giản, không cần scale.
- Budget hạn chế (Docker Compose + single server VPS rẻ hơn nhiều).
- Team chưa có kinh nghiệm K8s (overhead học tập quá cao).
- Prototype, MVP giai đoạn đầu.

---

### 2. K8s Architecture

#### Control Plane (Master Node)

Control plane là "bộ não" của cluster, chịu trách nhiệm quản lý toàn bộ trạng thái cluster.

**kube-apiserver:**
- REST API frontend của K8s. Mọi request (từ kubectl, từ các components khác, từ external clients) đều phải đi qua API server.
- Authenticate và authorize mọi request.
- Validate và persist resource objects vào etcd.
- Serve API tại `https://api-server:6443`.
- Stateless — có thể chạy nhiều replicas để HA.

```
kubectl apply -f deployment.yaml
  -> kubectl gửi POST request đến kube-apiserver
  -> apiserver validate manifest
  -> apiserver lưu vào etcd
  -> scheduler và controller-manager watch etcd changes
```

**etcd:**
- Distributed key-value store, nền tảng của toàn bộ cluster state.
- Lưu: pod definitions, service configs, secrets, configmaps, RBAC rules, v.v.
- Dùng Raft consensus algorithm để đảm bảo consistency.
- Backup etcd = backup toàn bộ cluster. Mất etcd = mất cluster.
- Production: chạy etcd cluster 3 hoặc 5 nodes (số lẻ, đảm bảo quorum).
- Port: 2379 (client), 2380 (peer).

**kube-scheduler:**
- Watch API server cho pods chưa được assign node (field `spec.nodeName` trống).
- Quyết định pod sẽ chạy trên node nào dựa trên:
  - **Resource requests:** Node có đủ CPU/memory không?
  - **Node affinity/anti-affinity:** Pod muốn/không muốn chạy trên node nào?
  - **Taints và Tolerations:** Node có taints không? Pod có toleration không?
  - **Pod affinity:** Pod muốn chạy gần hoặc xa pods khác không?
  - **Custom schedulers:** Bạn có thể viết scheduler riêng.
- Sau khi quyết định, scheduler update pod spec với `nodeName`.

**kube-controller-manager:**
- Chạy nhiều controllers trong một process. Mỗi controller là một control loop:
  - **ReplicaSet controller:** Đảm bảo số lượng pods đúng với `replicas` field.
  - **Deployment controller:** Quản lý rolling updates cho Deployments.
  - **Node controller:** Theo dõi node health, đánh dấu node không healthy.
  - **Endpoints controller:** Cập nhật Endpoints objects khi pods thay đổi.
  - **ServiceAccount controller:** Tạo default service accounts cho namespaces mới.
  - **Job controller:** Quản lý Jobs và CronJobs.

**cloud-controller-manager:**
- Tách biệt cloud-specific logic khỏi core K8s.
- Quản lý:
  - **Load balancers:** Khi tạo Service type LoadBalancer, CCM gọi AWS API tạo ELB.
  - **Volumes:** Dynamic provisioning PVs trên AWS EBS, GCP Persistent Disk.
  - **Node lifecycle:** Xóa nodes khỏi cluster khi EC2 instance bị terminate.
- Chạy riêng trên managed K8s (EKS, GKE), bạn không cần lo.

#### Worker Node

**kubelet:**
- Agent chạy trên mỗi worker node (và cả master node trong single-node setups).
- Nhận PodSpec từ API server (thông qua watch mechanism).
- Đảm bảo containers trong pod đang running và healthy.
- Thực thi liveness và readiness probes.
- Report node và pod status về API server.
- Giao tiếp với container runtime (containerd) qua CRI (Container Runtime Interface).
- Không quản lý containers không được tạo bởi K8s.

**kube-proxy:**
- Chạy trên mỗi node, implement K8s Service abstraction.
- Duy trì network rules (iptables hoặc IPVS) để forward traffic đến đúng pods.
- Khi bạn access `ClusterIP:Port`, kube-proxy rules chuyển traffic đến pod IP thực.
- Không phải actual proxy — nó chỉ setup kernel-level routing rules.
- IPVS mode nhanh hơn iptables mode cho clusters lớn (hàng nghìn services).

**Container runtime:**
- Thực sự chạy containers. K8s dùng CRI để giao tiếp với runtime.
- **containerd:** Runtime phổ biến nhất hiện nay (sau khi Docker Engine bị deprecated từ K8s 1.24).
- **CRI-O:** Runtime nhẹ hơn, được thiết kế cho K8s.
- **Docker Engine:** Không còn được hỗ trợ trực tiếp (nhưng images Docker vẫn dùng được vì cùng OCI format).

#### Luồng hoạt động khi deploy

```
1. kubectl apply -f deployment.yaml
   -> Gửi manifest đến kube-apiserver

2. kube-apiserver:
   -> Authenticate (kubeconfig credentials)
   -> Authorize (RBAC check)
   -> Validate manifest (schema check)
   -> Lưu Deployment object vào etcd

3. Deployment controller (trong kube-controller-manager):
   -> Watch etcd, thấy Deployment mới
   -> Tạo ReplicaSet object
   -> Lưu ReplicaSet vào etcd

4. ReplicaSet controller:
   -> Watch etcd, thấy ReplicaSet mới, current pods = 0, desired = 3
   -> Tạo 3 Pod objects với nodeName = "" (chưa được schedule)
   -> Lưu Pods vào etcd

5. kube-scheduler:
   -> Watch etcd, thấy 3 pods unscheduled
   -> Chạy scheduling algorithm cho từng pod
   -> Assign nodeName cho mỗi pod
   -> Update pod objects trong etcd

6. kubelet (trên từng node được assign):
   -> Watch API server, thấy pod mới được assign cho node mình
   -> Gọi containerd để pull image
   -> Start containers
   -> Execute probes
   -> Update pod status về API server

7. kube-proxy (trên mỗi node):
   -> Watch Service và Endpoints objects
   -> Update iptables/IPVS rules để route traffic đến pods mới
```

---

### 3. Pod

Pod là đơn vị nhỏ nhất trong K8s có thể được deploy và quản lý. Một Pod chứa một hoặc nhiều containers chia sẻ cùng network namespace và volumes.

#### Đặc điểm của Pod

**Shared network namespace:**
- Tất cả containers trong pod chia sẻ cùng network interface, IP address, và port space.
- Containers trong pod giao tiếp với nhau qua `localhost`.
- Từ bên ngoài pod, mỗi pod có một IP duy nhất trong cluster.

**Shared volumes:**
- Volumes được mount vào pod, tất cả containers trong pod có thể access.
- Dùng để chia sẻ dữ liệu giữa containers (ví dụ: sidecar pattern).

**Pod lifetime:**
- Pods là ephemeral (tạm thời). Khi pod chết, nó không tự restart — một pod mới được tạo với IP khác.
- Vì vậy, không bao giờ hard-code pod IP. Dùng Service DNS.

#### Sidecar Pattern

Một ứng dụng phổ biến của multi-container pods:

```
Pod: shoplite-backend
  Container 1: backend (Node.js app)
  Container 2: log-shipper (Fluentd/Filebeat)
    - Mount cùng volume chứa logs
    - Đọc logs từ volume, ship lên Elasticsearch/CloudWatch
```

Sidecar containers chạy cùng lifecycle với main container, tăng cường chức năng mà không sửa main app.

#### Pod Manifest hoàn chỉnh

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shoplite-backend
  namespace: shoplite
  labels:
    app: shoplite
    tier: backend
    version: v1.0.0
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "3000"
    prometheus.io/path: "/metrics"
spec:
  containers:
    - name: backend
      image: ghcr.io/user/shoplite-backend:v1.0.0
      imagePullPolicy: IfNotPresent
      ports:
        - containerPort: 3000
          name: http
          protocol: TCP
      env:
        - name: NODE_ENV
          value: production
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: shoplite-secrets
              key: database-url
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: shoplite-secrets
              key: jwt-secret
        - name: REDIS_URL
          valueFrom:
            configMapKeyRef:
              name: shoplite-config
              key: redis-url
      resources:
        requests:
          memory: "128Mi"
          cpu: "100m"
        limits:
          memory: "512Mi"
          cpu: "500m"
      livenessProbe:
        httpGet:
          path: /health
          port: 3000
        initialDelaySeconds: 30
        periodSeconds: 10
        timeoutSeconds: 5
        failureThreshold: 3
        successThreshold: 1
      readinessProbe:
        httpGet:
          path: /ready
          port: 3000
        initialDelaySeconds: 5
        periodSeconds: 5
        timeoutSeconds: 3
        failureThreshold: 3
        successThreshold: 1
      startupProbe:
        httpGet:
          path: /health
          port: 3000
        initialDelaySeconds: 10
        periodSeconds: 10
        failureThreshold: 30
      volumeMounts:
        - name: app-logs
          mountPath: /app/logs
        - name: tmp
          mountPath: /tmp
  volumes:
    - name: app-logs
      emptyDir: {}
    - name: tmp
      emptyDir: {}
  restartPolicy: Always
  terminationGracePeriodSeconds: 30
  imagePullSecrets:
    - name: ghcr-secret
```

#### Giải thích Probes

**livenessProbe:**
- Kiểm tra container có còn alive không.
- Nếu fail `failureThreshold` lần liên tiếp, K8s restart container.
- Dùng cho: detect deadlocks, OOM không recover được, bất kỳ trạng thái unrecoverable nào.
- `initialDelaySeconds`: chờ bao lâu trước khi bắt đầu probe (cho app khởi động).

**readinessProbe:**
- Kiểm tra container có sẵn sàng nhận traffic không.
- Nếu fail, pod bị remove khỏi Service Endpoints (traffic không được route đến).
- Container không bị restart — chỉ bị tạm thời excluded khỏi load balancing.
- Dùng cho: warm-up time, dependency checks (database connection ready chưa?).

**startupProbe:**
- Dành cho apps khởi động chậm.
- Trong khi startupProbe chưa success, liveness và readiness probes bị disable.
- Tránh việc K8s kill app đang khởi động vì liveness probe fail.

#### Resource Requests và Limits

```yaml
resources:
  requests:
    memory: "128Mi"   # Guaranteed memory cho scheduler
    cpu: "100m"       # 100 millicores = 0.1 CPU core
  limits:
    memory: "512Mi"   # Max memory, vượt quá -> OOMKilled
    cpu: "500m"       # Max CPU, vượt quá -> throttled (không kill)
```

- **Requests:** Lượng tài nguyên được đảm bảo. Scheduler dùng requests để quyết định đặt pod ở node nào.
- **Limits:** Lượng tài nguyên tối đa. Vượt memory limit -> OOMKilled. Vượt CPU limit -> throttled.
- **Không set limits** -> pod có thể chiếm toàn bộ tài nguyên node, ảnh hưởng pods khác.
- **Set limits quá thấp** -> app bị throttle/kill liên tục. Phải monitor và điều chỉnh.

#### Tại sao không deploy Pod trực tiếp

Pod trực tiếp (bare pod) sẽ không được recreate nếu node fail hoặc pod crash (không tính restartPolicy). Luôn dùng:
- **Deployment** cho stateless apps (web servers, APIs).
- **StatefulSet** cho stateful apps (databases, message queues).
- **DaemonSet** cho agents cần chạy trên mỗi node (log collectors, monitoring agents).
- **Job/CronJob** cho batch workloads.

---

### 4. Deployment — Resource chính cho Stateless Apps

Deployment quản lý ReplicaSets, cung cấp declarative updates, rolling updates, và rollbacks.

#### Deployment Manifest hoàn chỉnh

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shoplite-backend
  namespace: shoplite
  labels:
    app: shoplite
    tier: backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: shoplite
      tier: backend
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1         # Tối đa bao nhiêu pods THÊM vào trong lúc update
      maxUnavailable: 0   # Tối đa bao nhiêu pods có thể unavailable -> zero downtime
  minReadySeconds: 10     # Pod phải healthy ít nhất 10s trước khi tiếp tục update
  revisionHistoryLimit: 10  # Giữ 10 ReplicaSet cũ để rollback
  progressDeadlineSeconds: 600  # Timeout nếu update không hoàn thành trong 10 phút
  template:
    metadata:
      labels:
        app: shoplite
        tier: backend
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "3000"
    spec:
      containers:
        - name: backend
          image: ghcr.io/user/shoplite-backend:v1.0.0
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 3000
              name: http
          env:
            - name: NODE_ENV
              value: production
            - name: PORT
              value: "3000"
          envFrom:
            - configMapRef:
                name: shoplite-config
            - secretRef:
                name: shoplite-secrets
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /ready
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 3
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values: [shoplite]
                topologyKey: kubernetes.io/hostname
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: shoplite
              tier: backend
      terminationGracePeriodSeconds: 30
      imagePullSecrets:
        - name: ghcr-secret
```

#### Kubectl commands cho Deployment

```bash
# Xem trạng thái deployment
kubectl get deployment shoplite-backend -n shoplite
kubectl describe deployment shoplite-backend -n shoplite

# Xem rollout status (theo dõi real-time)
kubectl rollout status deployment/shoplite-backend -n shoplite

# Xem lịch sử rollout
kubectl rollout history deployment/shoplite-backend -n shoplite

# Xem chi tiết một revision
kubectl rollout history deployment/shoplite-backend -n shoplite --revision=2

# Update image (trigger rolling update)
kubectl set image deployment/shoplite-backend \
  backend=ghcr.io/user/shoplite-backend:v1.1.0 -n shoplite

# Rollback về revision trước
kubectl rollout undo deployment/shoplite-backend -n shoplite

# Rollback về revision cụ thể
kubectl rollout undo deployment/shoplite-backend --to-revision=2 -n shoplite

# Scale thủ công
kubectl scale deployment/shoplite-backend --replicas=5 -n shoplite

# Pause rolling update (để check)
kubectl rollout pause deployment/shoplite-backend -n shoplite

# Resume rolling update
kubectl rollout resume deployment/shoplite-backend -n shoplite

# Restart tất cả pods (rolling restart)
kubectl rollout restart deployment/shoplite-backend -n shoplite
```

#### Rolling Update hoạt động như thế nào

Giả sử có 3 pods v1.0.0, deploy v1.1.0 với `maxSurge: 1, maxUnavailable: 0`:

```
Initial state:  [v1.0.0] [v1.0.0] [v1.0.0]  (3 pods running)

Step 1: Create 1 new pod (maxSurge=1 allows 4 total)
        [v1.0.0] [v1.0.0] [v1.0.0] [v1.1.0-starting]

Step 2: v1.1.0 pod passes readiness probe
        [v1.0.0] [v1.0.0] [v1.0.0] [v1.1.0]

Step 3: Terminate 1 old pod (4 -> 3, maxUnavailable=0 OK because v1.1.0 is ready)
        [v1.0.0] [v1.0.0] [v1.1.0]

Step 4: Create 1 new pod
        [v1.0.0] [v1.0.0] [v1.1.0] [v1.1.0-starting]

... tiếp tục cho đến khi tất cả pods là v1.1.0

Final:  [v1.1.0] [v1.1.0] [v1.1.0]
```

Zero downtime vì luôn có ít nhất 3 pods running và healthy.

---

### 5. Service — Stable Network Endpoint

Pods có IP thay đổi liên tục. Service cung cấp IP và DNS name ổn định, load balance traffic đến các pods phù hợp.

#### Service Types

**ClusterIP (default):**
- Chỉ accessible từ trong cluster.
- Stable virtual IP được assign tự động.
- Dùng cho: internal communication giữa services.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: shoplite-backend-svc
  namespace: shoplite
  labels:
    app: shoplite
    tier: backend
spec:
  type: ClusterIP
  selector:
    app: shoplite
    tier: backend
  ports:
    - name: http
      port: 80          # Port của Service
      targetPort: 3000  # Port của container
      protocol: TCP
```

DNS: `shoplite-backend-svc.shoplite.svc.cluster.local`
Short form (cùng namespace): `shoplite-backend-svc`

**NodePort:**
- Expose service trên port 30000-32767 của tất cả nodes.
- Accessible từ ngoài cluster: `<NodeIP>:<NodePort>`.
- Dùng cho: development, testing khi không có LoadBalancer.

```yaml
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 3000
      nodePort: 30080   # Optional: nếu không set, K8s tự assign
```

**LoadBalancer:**
- Tạo external load balancer (AWS ELB, GCP Load Balancer) tự động.
- Accessible từ internet qua external IP.
- Dùng cho: production external access (thường kết hợp với Ingress thay vì dùng trực tiếp).

```yaml
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 3000
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-internal: "false"
```

**ExternalName:**
- Alias cho external DNS name.
- Dùng cho: trỏ đến external services (RDS database, external APIs).

```yaml
spec:
  type: ExternalName
  externalName: mydb.abc123.us-east-1.rds.amazonaws.com
```

#### Headless Service (StatefulSet)

```yaml
spec:
  clusterIP: None   # Headless: không có ClusterIP
  selector:
    app: postgres
```

DNS trả về pod IPs trực tiếp thay vì ClusterIP. Dùng với StatefulSets để address từng pod cụ thể:
- `postgres-0.postgres-svc.shoplite.svc.cluster.local`
- `postgres-1.postgres-svc.shoplite.svc.cluster.local`

#### Endpoints và EndpointSlices

Khi bạn tạo Service với selector, K8s tự tạo Endpoints object chứa danh sách pod IPs:

```bash
kubectl get endpoints shoplite-backend-svc -n shoplite
# NAME                   ENDPOINTS                                         AGE
# shoplite-backend-svc   10.0.1.5:3000,10.0.1.6:3000,10.0.1.7:3000        5m
```

Nếu service không route đến pods, check endpoints: nếu trống, labels selector không match.

---

### 6. Namespace

Namespace cung cấp virtual cluster isolation trong cùng physical cluster.

#### Default Namespaces

- **default:** Namespace mặc định nếu không specify.
- **kube-system:** System components (kube-dns, kube-proxy, metrics-server).
- **kube-public:** Resources có thể đọc bởi tất cả users (kể cả unauthenticated).
- **kube-node-lease:** Node heartbeat objects.

#### ShopLite Namespace Strategy

```yaml
# namespace-shoplite.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: shoplite
  labels:
    env: production
    project: shoplite

---
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
  labels:
    purpose: observability
```

#### ResourceQuota — Giới hạn tài nguyên cho Namespace

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: shoplite-quota
  namespace: shoplite
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "50"
    services: "20"
    persistentvolumeclaims: "10"
```

#### LimitRange — Default resource limits

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: shoplite-limits
  namespace: shoplite
spec:
  limits:
    - type: Container
      default:
        cpu: "200m"
        memory: "256Mi"
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
      max:
        cpu: "2"
        memory: "2Gi"
```

#### Kubectl commands cho Namespace

```bash
# Tạo namespace
kubectl create namespace shoplite

# Xem tất cả namespaces
kubectl get namespaces

# Làm việc với namespace cụ thể
kubectl get pods -n shoplite
kubectl get all -n shoplite

# Xem tất cả namespaces cùng lúc
kubectl get pods -A
kubectl get pods --all-namespaces

# Set default namespace cho context (tránh phải gõ -n mỗi lần)
kubectl config set-context --current --namespace=shoplite

# Xem namespace hiện tại
kubectl config view --minify | grep namespace
```

---

### 7. ConfigMap & Secret

#### ConfigMap — Cấu hình non-sensitive

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: shoplite-config
  namespace: shoplite
data:
  NODE_ENV: production
  PORT: "3000"
  LOG_LEVEL: info
  REDIS_URL: redis://redis-svc:6379
  API_RATE_LIMIT: "100"
  CORS_ORIGIN: "https://shoplite.com"
  UPLOAD_MAX_SIZE: "10mb"
```

**Dùng ConfigMap trong Pod:**

```yaml
spec:
  containers:
    - name: backend
      # Option 1: Load tất cả keys thành env vars
      envFrom:
        - configMapRef:
            name: shoplite-config

      # Option 2: Load từng key cụ thể
      env:
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: shoplite-config
              key: LOG_LEVEL

      # Option 3: Mount ConfigMap như file
      volumeMounts:
        - name: config-volume
          mountPath: /app/config

  volumes:
    - name: config-volume
      configMap:
        name: shoplite-config
```

#### Secret — Dữ liệu nhạy cảm

**Quan trọng:** Secret trong K8s chỉ được base64 encoded, KHÔNG phải encrypted by default. Bất kỳ ai có quyền `kubectl get secret` đều có thể decode. Để thực sự bảo mật:
- Bật encryption at rest trong etcd.
- Dùng External Secrets Operator + AWS Secrets Manager / HashiCorp Vault.
- Dùng Sealed Secrets (bitnami).

**Tạo Secret từ command line:**

```bash
# Generic secret từ literals
kubectl create secret generic shoplite-secrets \
  --from-literal=database-url="postgresql://shoplite:StrongPass123@postgres-svc:5432/shoplite" \
  --from-literal=jwt-secret="your-super-secret-jwt-key-32chars-minimum" \
  --from-literal=redis-password="redis-password-here" \
  -n shoplite

# TLS secret
kubectl create secret tls shoplite-tls \
  --cert=path/to/tls.crt \
  --key=path/to/tls.key \
  -n shoplite

# Docker registry credentials
kubectl create secret docker-registry ghcr-secret \
  --docker-server=ghcr.io \
  --docker-username=your-github-username \
  --docker-password=your-github-token \
  -n shoplite
```

**Secret manifest (dùng stringData cho dễ đọc):**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: shoplite-secrets
  namespace: shoplite
type: Opaque
stringData:
  database-url: "postgresql://shoplite:StrongPass123@postgres-svc:5432/shoplite"
  jwt-secret: "your-super-secret-jwt-key-minimum-32-chars"
  redis-password: "redis-strong-password"
  smtp-password: "smtp-password-here"
```

**Dùng Secret trong Pod:**

```yaml
spec:
  containers:
    - name: backend
      env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: shoplite-secrets
              key: database-url
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: shoplite-secrets
              key: jwt-secret
      # Hoặc load tất cả (cẩn thận — expose tất cả keys)
      envFrom:
        - secretRef:
            name: shoplite-secrets
  imagePullSecrets:
    - name: ghcr-secret
```

**Decode Secret:**

```bash
kubectl get secret shoplite-secrets -n shoplite -o jsonpath='{.data.database-url}' | base64 -d
```

---

### 8. Storage — Volumes, PV, PVC, StorageClass

#### Vấn đề với container storage

Container filesystem là ephemeral — khi container restart, data mất. Pod storage cần:
- **Persist across container restarts** (volumes giải quyết).
- **Persist across pod rescheduling** (PersistentVolumes giải quyết).
- **Share data between containers** (emptyDir volumes giải quyết).

#### Volume Types

**emptyDir:**
- Tạo khi pod được tạo, xóa khi pod bị xóa.
- Dùng cho: shared scratch space giữa containers, temp files.

```yaml
volumes:
  - name: shared-data
    emptyDir: {}
```

**hostPath:**
- Mount file/directory từ host node.
- Dùng cho: development, DaemonSets cần access host filesystem.
- Nguy hiểm trong production — không portable.

```yaml
volumes:
  - name: host-logs
    hostPath:
      path: /var/log/pods
      type: Directory
```

**configMap / secret:**
- Mount ConfigMap hoặc Secret như files trong container.
- Tự động update khi ConfigMap/Secret thay đổi (sau ~1 phút).

#### PersistentVolume (PV)

PV là piece of storage được provision bởi admin hoặc dynamically. Có lifecycle độc lập với pods.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: postgres-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce       # Chỉ 1 node có thể mount read-write
  persistentVolumeReclaimPolicy: Retain   # Không xóa data khi PVC bị xóa
  storageClassName: gp2
  awsElasticBlockStore:
    volumeID: vol-0123456789abcdef0
    fsType: ext4
```

**Access Modes:**
- `ReadWriteOnce (RWO)`: Mount read-write bởi 1 node (AWS EBS, GCP PD).
- `ReadOnlyMany (ROX)`: Mount read-only bởi nhiều nodes.
- `ReadWriteMany (RWX)`: Mount read-write bởi nhiều nodes (NFS, EFS).
- `ReadWriteOncePod (RWOP)`: Mount read-write bởi 1 pod duy nhất (K8s 1.22+).

#### PersistentVolumeClaim (PVC)

PVC là yêu cầu storage của user. K8s tìm PV phù hợp và bind với PVC.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: shoplite
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: gp2
  resources:
    requests:
      storage: 10Gi
```

**Dùng PVC trong Pod:**

```yaml
spec:
  volumes:
    - name: postgres-data
      persistentVolumeClaim:
        claimName: postgres-pvc
  containers:
    - name: postgres
      image: postgres:15
      volumeMounts:
        - name: postgres-data
          mountPath: /var/lib/postgresql/data
```

#### StorageClass — Dynamic Provisioning

Thay vì tạo PV thủ công, StorageClass tự động provision PV khi PVC được tạo.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp2
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
  fsType: ext4
  encrypted: "true"
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

#### StatefulSet cho PostgreSQL

StatefulSet đảm bảo:
- Pods có stable identity: `postgres-0`, `postgres-1`, `postgres-2`.
- Stable DNS: `postgres-0.postgres-svc.shoplite.svc.cluster.local`.
- Pods được deploy/scale/delete theo thứ tự.
- Mỗi pod có PVC riêng.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: shoplite
spec:
  serviceName: postgres-svc   # Headless service
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:15-alpine
          ports:
            - containerPort: 5432
              name: postgres
          env:
            - name: POSTGRES_DB
              value: shoplite
            - name: POSTGRES_USER
              value: shoplite
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: shoplite-secrets
                  key: postgres-password
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "1Gi"
              cpu: "1"
          volumeMounts:
            - name: postgres-data
              mountPath: /var/lib/postgresql/data
          livenessProbe:
            exec:
              command: [pg_isready, -U, shoplite]
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            exec:
              command: [pg_isready, -U, shoplite]
            initialDelaySeconds: 5
            periodSeconds: 5
  volumeClaimTemplates:
    - metadata:
        name: postgres-data
      spec:
        accessModes: [ReadWriteOnce]
        storageClassName: gp2
        resources:
          requests:
            storage: 10Gi
```

---

### 9. Ingress

Ingress là L7 (HTTP/HTTPS) routing layer, thay thế cho việc tạo nhiều LoadBalancer services.

#### Tại sao dùng Ingress

- **Không có Ingress:** Mỗi service cần 1 LoadBalancer = 1 IP = tiền.
- **Với Ingress:** 1 LoadBalancer cho tất cả services, routing dựa trên host/path.
- Handle SSL/TLS termination tại một chỗ.
- Path-based routing: `/api` -> backend, `/` -> frontend.
- Host-based routing: `api.shoplite.com` -> backend, `app.shoplite.com` -> frontend.

#### Ingress Controller

Ingress resource chỉ là config. Cần Ingress Controller để thực sự implement:

**Install NGINX Ingress Controller với Helm:**

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  -n ingress-nginx \
  --create-namespace \
  --set controller.replicaCount=2 \
  --set controller.service.type=LoadBalancer \
  --set controller.metrics.enabled=true

# Verify
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
# ingress-nginx-controller   LoadBalancer   ...   EXTERNAL-IP   80:30080/TCP,443:30443/TCP
```

**Install cert-manager cho tự động SSL:**

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml

# Tạo ClusterIssuer cho Let's Encrypt
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@shoplite.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01:
          ingress:
            class: nginx
EOF
```

#### Ingress Resource hoàn chỉnh

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: shoplite-ingress
  namespace: shoplite
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    nginx.ingress.kubernetes.io/rate-limit: "100"
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - shoplite.com
        - www.shoplite.com
      secretName: shoplite-tls
  rules:
    - host: shoplite.com
      http:
        paths:
          - path: /api(/|$)(.*)
            pathType: Prefix
            backend:
              service:
                name: shoplite-backend-svc
                port:
                  number: 80
          - path: /()(.*)
            pathType: Prefix
            backend:
              service:
                name: shoplite-frontend-svc
                port:
                  number: 80
    - host: www.shoplite.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: shoplite-frontend-svc
                port:
                  number: 80
```

---

### 10. HPA — Horizontal Pod Autoscaler

HPA tự động scale số lượng pods dựa trên metrics.

#### Yêu cầu: metrics-server

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Verify
kubectl top nodes
kubectl top pods -n shoplite
```

#### HPA Manifest

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: shoplite-backend-hpa
  namespace: shoplite
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: shoplite-backend
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70    # Scale up khi CPU > 70%
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80    # Scale up khi memory > 80%
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60    # Chờ 60s trước khi scale up tiếp
      policies:
        - type: Pods
          value: 2
          periodSeconds: 60             # Thêm tối đa 2 pods mỗi 60 giây
    scaleDown:
      stabilizationWindowSeconds: 300   # Chờ 5 phút trước khi scale down
      policies:
        - type: Pods
          value: 1
          periodSeconds: 60             # Xóa tối đa 1 pod mỗi 60 giây
```

#### Test HPA với load generator

```bash
# Mở terminal 1: watch HPA
kubectl get hpa shoplite-backend-hpa -n shoplite -w

# Mở terminal 2: watch pods
kubectl get pods -n shoplite -w

# Mở terminal 3: generate load
kubectl run load-generator \
  --image=busybox \
  --restart=Never \
  -n shoplite \
  -- /bin/sh -c "while true; do wget -q -O- http://shoplite-backend-svc/api/products; done"

# Sau khi test, xóa load generator
kubectl delete pod load-generator -n shoplite
```

```bash
# Xem HPA status
kubectl get hpa -n shoplite
# NAME                    REFERENCE                       TARGETS         MINPODS   MAXPODS   REPLICAS
# shoplite-backend-hpa    Deployment/shoplite-backend     45%/70%, ...    2         10        3

kubectl describe hpa shoplite-backend-hpa -n shoplite
```

---

### 11. Helm — Package Manager cho Kubernetes

#### Tại sao cần Helm

Không có Helm, để deploy ShopLite lên K8s bạn cần apply hàng chục YAML files:

```bash
kubectl apply -f namespace.yaml
kubectl apply -f secrets.yaml
kubectl apply -f configmap.yaml
kubectl apply -f backend-deployment.yaml
kubectl apply -f backend-service.yaml
kubectl apply -f frontend-deployment.yaml
kubectl apply -f frontend-service.yaml
kubectl apply -f redis-deployment.yaml
kubectl apply -f redis-service.yaml
kubectl apply -f postgres-statefulset.yaml
kubectl apply -f postgres-service.yaml
kubectl apply -f ingress.yaml
kubectl apply -f hpa.yaml
# ... và mỗi lần deploy prod vs dev thì values khác nhau -> copy-paste, error-prone
```

Với Helm:

```bash
helm install shoplite ./shoplite-chart -n shoplite --create-namespace -f values.prod.yaml
```

#### Helm Chart Structure

```
shoplite/
  Chart.yaml              # Metadata của chart
  values.yaml             # Default values
  values.dev.yaml         # Override cho dev
  values.prod.yaml        # Override cho prod
  charts/                 # Dependencies (sub-charts)
  templates/
    _helpers.tpl          # Template helpers (reusable snippets)
    namespace.yaml
    configmap.yaml
    secret.yaml
    backend/
      deployment.yaml
      service.yaml
      hpa.yaml
    frontend/
      deployment.yaml
      service.yaml
    redis/
      deployment.yaml
      service.yaml
    postgres/
      statefulset.yaml
      service.yaml
      pvc.yaml
    ingress.yaml
    NOTES.txt             # Hiển thị sau khi install
```

#### Chart.yaml

```yaml
apiVersion: v2
name: shoplite
description: ShopLite - Full-stack e-commerce application
type: application
version: 1.0.0          # Chart version
appVersion: "1.0.0"     # App version
keywords:
  - ecommerce
  - nodejs
  - postgresql
maintainers:
  - name: DevOps Team
    email: devops@shoplite.com
dependencies:
  - name: redis
    version: "17.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: redis.enabled
```

#### values.yaml hoàn chỉnh

```yaml
# Global settings
global:
  imageRegistry: ghcr.io
  imagePullSecrets:
    - name: ghcr-secret
  storageClass: gp2

# Backend
backend:
  enabled: true
  replicaCount: 3
  image:
    repository: ghcr.io/user/shoplite-backend
    tag: latest
    pullPolicy: IfNotPresent
  service:
    type: ClusterIP
    port: 80
    targetPort: 3000
  resources:
    limits:
      cpu: 500m
      memory: 512Mi
    requests:
      cpu: 100m
      memory: 128Mi
  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 10
    targetCPUUtilizationPercentage: 70
  env:
    NODE_ENV: production
    PORT: "3000"
    LOG_LEVEL: info

# Frontend
frontend:
  enabled: true
  replicaCount: 2
  image:
    repository: ghcr.io/user/shoplite-frontend
    tag: latest
    pullPolicy: IfNotPresent
  service:
    type: ClusterIP
    port: 80
  resources:
    limits:
      cpu: 200m
      memory: 256Mi
    requests:
      cpu: 50m
      memory: 64Mi

# PostgreSQL
postgres:
  enabled: true
  image:
    repository: postgres
    tag: "15-alpine"
  persistence:
    enabled: true
    size: 10Gi
    storageClass: gp2
  resources:
    limits:
      cpu: "1"
      memory: 1Gi
    requests:
      cpu: 250m
      memory: 256Mi

# Redis
redis:
  enabled: true
  architecture: standalone
  auth:
    enabled: true

# Ingress
ingress:
  enabled: true
  className: nginx
  host: shoplite.com
  tls:
    enabled: true
    secretName: shoplite-tls
    clusterIssuer: letsencrypt-prod

# Secrets (in production, use External Secrets Operator)
secrets:
  databaseUrl: ""         # Override với --set secrets.databaseUrl=...
  jwtSecret: ""
  postgresPassword: ""
```

#### _helpers.tpl

```
{{/*
Expand the name of the chart.
*/}}
{{- define "shoplite.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a default fully qualified app name.
*/}}
{{- define "shoplite.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "shoplite.labels" -}}
helm.sh/chart: {{ include "shoplite.chart" . }}
{{ include "shoplite.selectorLabels" . }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels
*/}}
{{- define "shoplite.selectorLabels" -}}
app.kubernetes.io/name: {{ include "shoplite.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

#### Backend Deployment Template

```yaml
{{- if .Values.backend.enabled }}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "shoplite.fullname" . }}-backend
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "shoplite.labels" . | nindent 4 }}
    component: backend
spec:
  {{- if not .Values.backend.autoscaling.enabled }}
  replicas: {{ .Values.backend.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "shoplite.selectorLabels" . | nindent 6 }}
      component: backend
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        {{- include "shoplite.selectorLabels" . | nindent 8 }}
        component: backend
    spec:
      {{- with .Values.global.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      containers:
        - name: backend
          image: "{{ .Values.backend.image.repository }}:{{ .Values.backend.image.tag }}"
          imagePullPolicy: {{ .Values.backend.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.backend.service.targetPort }}
              name: http
          env:
            {{- range $key, $value := .Values.backend.env }}
            - name: {{ $key }}
              value: {{ $value | quote }}
            {{- end }}
          envFrom:
            - secretRef:
                name: {{ include "shoplite.fullname" . }}-secrets
          resources:
            {{- toYaml .Values.backend.resources | nindent 12 }}
          livenessProbe:
            httpGet:
              path: /health
              port: http
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /ready
              port: http
            initialDelaySeconds: 5
            periodSeconds: 5
{{- end }}
```

#### Helm Commands

```bash
# Install chart lần đầu
helm install shoplite ./shoplite -n shoplite --create-namespace

# Install với file values tùy chỉnh
helm install shoplite ./shoplite \
  -n shoplite \
  --create-namespace \
  -f values.prod.yaml \
  --set backend.image.tag=v1.0.0 \
  --set secrets.databaseUrl="postgresql://..."

# Upgrade (deploy phiên bản mới)
helm upgrade shoplite ./shoplite \
  -n shoplite \
  --set backend.image.tag=v1.1.0

# Upgrade với --install (install nếu chưa có, upgrade nếu đã có)
helm upgrade --install shoplite ./shoplite \
  -n shoplite \
  --create-namespace \
  -f values.prod.yaml

# Rollback về revision trước
helm rollback shoplite -n shoplite

# Rollback về revision cụ thể
helm rollback shoplite 2 -n shoplite

# Xem history
helm history shoplite -n shoplite

# Uninstall
helm uninstall shoplite -n shoplite

# List releases
helm list -A
helm list -n shoplite

# Debug: xem rendered templates (không apply)
helm template shoplite ./shoplite -f values.prod.yaml

# Validate chart
helm lint ./shoplite

# Dry run
helm install shoplite ./shoplite --dry-run --debug

# Get values của release đang chạy
helm get values shoplite -n shoplite
```

---

### 12. kubectl Cheat Sheet

#### Xem resources

```bash
# Nodes
kubectl get nodes
kubectl get nodes -o wide      # Thêm IP, OS, container runtime
kubectl describe node node-name

# Pods
kubectl get pods -n shoplite
kubectl get pods -n shoplite -o wide     # Thêm node name, IP
kubectl get pods -n shoplite -w          # Watch real-time changes
kubectl get pods -n shoplite -l app=shoplite  # Filter by label
kubectl get pods -A                      # Tất cả namespaces

# Deployments
kubectl get deployments -n shoplite
kubectl describe deployment shoplite-backend -n shoplite

# Services
kubectl get services -n shoplite
kubectl get svc -n shoplite   # Short form

# Ingress
kubectl get ingress -n shoplite

# ConfigMaps & Secrets
kubectl get configmaps -n shoplite
kubectl get secrets -n shoplite

# PVCs
kubectl get pvc -n shoplite

# Tất cả resources trong namespace
kubectl get all -n shoplite
```

#### Logs

```bash
# Logs của pod (container đầu tiên)
kubectl logs pod-name -n shoplite

# Follow logs real-time
kubectl logs -f pod-name -n shoplite

# Logs của container cụ thể trong pod
kubectl logs pod-name -c container-name -n shoplite

# Logs của pod đã restart (container trước)
kubectl logs pod-name --previous -n shoplite

# Logs của tất cả pods với label
kubectl logs -l app=shoplite -n shoplite

# Logs với timestamp
kubectl logs pod-name --timestamps -n shoplite

# Logs 100 dòng cuối
kubectl logs pod-name --tail=100 -n shoplite
```

#### Execute commands

```bash
# Exec vào container
kubectl exec -it pod-name -n shoplite -- /bin/sh

# Exec vào container cụ thể
kubectl exec -it pod-name -c container-name -n shoplite -- /bin/bash

# Chạy lệnh không cần interactive
kubectl exec pod-name -n shoplite -- env

# Check connectivity từ trong pod
kubectl exec -it pod-name -n shoplite -- curl http://shoplite-backend-svc/health
```

#### Apply và Delete

```bash
# Apply manifest
kubectl apply -f manifest.yaml
kubectl apply -f ./k8s/           # Apply tất cả files trong folder
kubectl apply -f ./k8s/ -R        # Recursive

# Delete resource
kubectl delete -f manifest.yaml
kubectl delete pod pod-name -n shoplite
kubectl delete pod pod-name -n shoplite --grace-period=0  # Force delete

# Delete tất cả pods trong namespace (sẽ được recreate bởi deployment)
kubectl delete pods -l app=shoplite -n shoplite
```

#### Port forwarding

```bash
# Forward port để test service locally
kubectl port-forward svc/shoplite-backend-svc 8080:80 -n shoplite
# Sau đó: curl http://localhost:8080/api/products

# Forward trực tiếp đến pod
kubectl port-forward pod/shoplite-backend-xxx 3000:3000 -n shoplite
```

#### Resource usage

```bash
kubectl top nodes
kubectl top pods -n shoplite
kubectl top pods -n shoplite --sort-by=cpu
kubectl top pods -n shoplite --sort-by=memory
```

#### Events

```bash
# Xem events trong namespace (debug khi pods không start)
kubectl get events -n shoplite
kubectl get events -n shoplite --sort-by=.lastTimestamp

# Events của resource cụ thể
kubectl describe pod pod-name -n shoplite  # Events ở cuối output
```

#### Labels và Annotations

```bash
# Add label cho node
kubectl label node node-name role=worker

# Add label cho pod
kubectl label pod pod-name env=production -n shoplite

# Remove label
kubectl label pod pod-name env- -n shoplite

# Filter bằng label selector
kubectl get pods -l "app=shoplite,tier=backend" -n shoplite
kubectl get pods -l "tier in (backend,frontend)" -n shoplite
```

---

### 13. Debugging Kubernetes

#### Pod không start — Debugging Flow

```bash
# Bước 1: Xem trạng thái pod
kubectl get pods -n shoplite

# Bước 2: Describe pod để xem Events
kubectl describe pod pod-name -n shoplite
# Đọc kỹ section "Events:" ở cuối output

# Bước 3: Xem logs
kubectl logs pod-name -n shoplite
kubectl logs pod-name --previous -n shoplite  # Nếu container đã restart
```

#### Trạng thái và nguyên nhân thường gặp

**Pending:**
- Pod được tạo nhưng chưa được schedule lên node nào.
- Nguyên nhân thường gặp:
  - Không đủ resources (CPU/memory) trên bất kỳ node nào.
  - Node selector hoặc affinity không match với bất kỳ node nào.
  - Taint trên nodes mà pod không có toleration.
  - PVC không được bind (không có PV phù hợp).

```bash
# Debug Pending pod
kubectl describe pod pod-name -n shoplite
# Tìm: "0/3 nodes are available: 3 Insufficient memory"
# Tìm: "didn't match Pod's node affinity"

# Xem resources còn lại của nodes
kubectl describe nodes | grep -A 5 "Allocated resources"
```

**CrashLoopBackOff:**
- Container khởi động, crash, K8s restart, crash lại liên tục.
- Nguyên nhân:
  - Application code có bug, crash ngay khi start.
  - Thiếu environment variables bắt buộc.
  - Config sai (database URL sai, port conflict).
  - Liveness probe fail (probe quá aggressive).
  - OOMKilled (thiếu memory).

```bash
# Xem logs trước khi crash
kubectl logs pod-name --previous -n shoplite

# Xem exit code
kubectl describe pod pod-name -n shoplite
# Tìm: "Last State: Terminated, Exit Code: 1"
# Exit Code 137 = OOMKilled
# Exit Code 1 = Application error
```

**ImagePullBackOff / ErrImagePull:**
- K8s không pull được image.
- Nguyên nhân:
  - Tên image sai (typo, tag không tồn tại).
  - Private registry nhưng không có `imagePullSecrets`.
  - Registry credentials expired.
  - Network issue.

```bash
# Verify image name
kubectl describe pod pod-name -n shoplite
# Tìm: "Failed to pull image: rpc error..."

# Check imagePullSecrets
kubectl get pod pod-name -n shoplite -o jsonpath='{.spec.imagePullSecrets}'
```

**OOMKilled:**
- Container dùng quá memory limit, bị kernel kill.
- Solution: tăng memory limit, tìm memory leak trong app.

```bash
kubectl describe pod pod-name -n shoplite
# Tìm: "OOMKilled"
# "Last State: Terminated, Exit Code: 137, Reason: OOMKilled"
```

#### Service không reachable

```bash
# Bước 1: Check service tồn tại
kubectl get svc shoplite-backend-svc -n shoplite

# Bước 2: Check endpoints (pods có match selector không?)
kubectl get endpoints shoplite-backend-svc -n shoplite
# Nếu Endpoints rỗng hoặc <none> -> labels không match

# Bước 3: Check labels của pods
kubectl get pods -n shoplite -l app=shoplite,tier=backend

# Bước 4: Test connectivity từ trong cluster
kubectl run debug --image=curlimages/curl -it --rm -n shoplite -- \
  curl http://shoplite-backend-svc.shoplite.svc.cluster.local/health

# Bước 5: Check kube-proxy logs nếu vẫn không được
kubectl logs -n kube-system -l k8s-app=kube-proxy
```

#### Node không healthy

```bash
# Xem trạng thái nodes
kubectl get nodes
# Status: NotReady -> node có vấn đề

# Describe node để xem conditions
kubectl describe node node-name
# Conditions: MemoryPressure, DiskPressure, PIDPressure, Ready

# Xem pods trên node
kubectl get pods -A --field-selector spec.nodeName=node-name

# Xem kubelet logs (trên node đó)
# journalctl -u kubelet -f
```

---

### 14. Managed K8s — EKS trên AWS

#### Tại sao dùng EKS thay vì tự setup

- AWS quản lý control plane (API server, etcd, scheduler, controller-manager).
- Automatic upgrades cho control plane.
- Integrated với AWS services (IAM, ALB, EBS, EFS, CloudWatch).
- SLA 99.95% cho control plane.
- Bạn chỉ quản lý worker nodes.

#### Tạo EKS cluster với eksctl

```bash
# Cài eksctl
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

# Tạo cluster cơ bản
eksctl create cluster \
  --name shoplite-prod \
  --region ap-southeast-1 \
  --nodegroup-name workers \
  --node-type t3.medium \
  --nodes 3 \
  --nodes-min 2 \
  --nodes-max 5 \
  --managed

# Hoặc dùng cluster config file
eksctl create cluster -f cluster.yaml
```

**cluster.yaml:**

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: shoplite-prod
  region: ap-southeast-1
  version: "1.28"

iam:
  withOIDC: true

managedNodeGroups:
  - name: workers
    instanceType: t3.medium
    minSize: 2
    maxSize: 5
    desiredCapacity: 3
    volumeSize: 30
    labels: {role: worker}
    iam:
      attachPolicyARNs:
        - arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
        - arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
        - arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly

addons:
  - name: vpc-cni
  - name: coredns
  - name: kube-proxy
  - name: aws-ebs-csi-driver

cloudWatch:
  clusterLogging:
    enable: true
    logRetentionInDays: 30
```

#### Update kubeconfig sau khi tạo cluster

```bash
aws eks update-kubeconfig \
  --region ap-southeast-1 \
  --name shoplite-prod

# Verify
kubectl get nodes
kubectl get pods -n kube-system
```

---

### 15. Local Development với kind hoặc minikube

#### kind (Kubernetes in Docker)

```bash
# Cài kind
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
chmod +x ./kind && sudo mv ./kind /usr/local/bin/kind

# Tạo cluster đơn giản
kind create cluster --name shoplite-dev

# Tạo cluster multi-node
cat > kind-config.yaml << EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
EOF
kind create cluster --config kind-config.yaml --name shoplite-dev

# Load local image vào kind cluster (không cần push lên registry)
kind load docker-image shoplite-backend:local --name shoplite-dev

# Delete cluster
kind delete cluster --name shoplite-dev
```

#### minikube

```bash
# Start cluster
minikube start --cpus=4 --memory=8g --driver=docker

# Enable addons
minikube addons enable ingress
minikube addons enable metrics-server
minikube addons enable dashboard

# Truy cập service
minikube service shoplite-backend-svc -n shoplite

# Xem IP của minikube
minikube ip

# Load local image
minikube image load shoplite-backend:local

# Stop/delete
minikube stop
minikube delete
```

---

## Kỹ năng đạt được
- Triển khai ứng dụng đa service lên Kubernetes.
- Cấu hình networking, config, secret, storage trong K8s.
- Đóng gói ứng dụng bằng Helm chart.

---

## Thực hành

**Môi trường:** Local (kind/minikube) trước, sau đó EKS trên AWS.

**Lab:**
- Viết manifest cho frontend, backend, Redis (Deployment + Service).
- Dùng ConfigMap/Secret cho cấu hình & mật khẩu.
- Cài Ingress Controller, định tuyến traffic vào cluster.
- Cấu hình HPA, test tự scale khi tải cao.
- Đóng gói toàn bộ thành một Helm chart.
- Triển khai lên EKS.

**Công cụ:** kubectl, kind/minikube, EKS, Helm, Ingress-NGINX.

---

## Bài tập

- **Bắt buộc:** ShopLite chạy được trên cluster K8s, truy cập qua Ingress.
- **Nâng cao:** Cấu hình HPA + minh họa self-healing (xóa pod, K8s tự tạo lại).
- **Mô phỏng doanh nghiệp:** Đóng gói ShopLite thành Helm chart có `values.yaml` cho dev/prod, deploy bằng một lệnh.

---

## Deliverable
- [ ] Thư mục `k8s/` chứa manifest hoặc Helm chart.
- [ ] ShopLite chạy trên EKS, truy cập qua Ingress + domain.
- [ ] Tài liệu kiến trúc K8s.

---

## Tiêu chí hoàn thành
- [ ] Toàn bộ hệ thống chạy trên K8s, truy cập công khai.
- [ ] Xóa một pod -> hệ thống tự phục hồi, không downtime.
- [ ] Triển khai lại bằng Helm chỉ với một lệnh.

---

## Ghi chú học tập

> _Ghi lại những điều học được, vướng mắc và insight trong quá trình học phase này._
