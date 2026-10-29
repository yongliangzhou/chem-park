# 化工园区信息化平台 · Kubernetes Helm 部署清单示例

> **版本**：V1.0 | **日期**：2026-08-22
> **配套文档**：《化工园区信息化平台-微服务架构设计方案.md》
> **目标**：私有化/园区本地 K8s 集群一键部署 14 个微服务 + 中间件，满足等保三级与高可用。

---

## 目录

1. [集群与命名空间规划](#一集群与命名空间规划)
2. [Chart 目录结构](#二chart-目录结构)
3. [全局 values.yaml](#三全局-valuesyaml)
4. [中间件部署（PostgreSQL/TimescaleDB/Kafka/Redis/Nacos/MinIO）](#四中间件部署)
5. [微服务 Deployment + Service 示例](#五微服务-deployment--service-示例)
6. [Ingress / Gateway 暴露](#六ingress--gateway-暴露)
7. [HPA / 资源配额 / 反亲和](#七hpa--资源配额--反亲和)
8. [安全上下文与网络策略](#八安全上下文与网络策略)
9. [部署命令](#九部署命令)

---

## 一、集群与命名空间规划

| 命名空间 | 用途 |
|----------|------|
| `chem-park-middleware` | Nacos / Kafka / PostgreSQL / TimescaleDB / Redis / MinIO / ELK |
| `chem-park-app` | 14 个微服务 + Gateway |
| `chem-park-monitor` | Prometheus / Grafana / Alertmanager |
| `ingress-nginx` | Ingress 控制器 |

> 关键服务（monitor / emergency / ioc）打 `priorityClassName: high-priority`，OOM/驱逐时优先保留。

---

## 二、Chart 目录结构

```
chem-park-platform/
├── Chart.yaml
├── values.yaml
├── templates/
│   ├── namespace.yaml
│   ├── middleware/
│   │   ├── postgres.yaml
│   │   ├── timescaledb.yaml
│   │   ├── kafka.yaml
│   │   ├── redis.yaml
│   │   ├── nacos.yaml
│   │   └── minio.yaml
│   ├── gateway/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   ├── services/
│   │   ├── _service.tpl          # 通用微服务模板
│   │   ├── iam.yaml
│   │   ├── mdm.yaml
│   │   ├── iot-ingest.yaml
│   │   ├── device.yaml
│   │   ├── safety.yaml
│   │   ├── monitor.yaml
│   │   ├── closedmgmt.yaml
│   │   ├── env.yaml
│   │   ├── energy.yaml
│   │   ├── corridor.yaml
│   │   ├── emergency.yaml
│   │   ├── operation.yaml
│   │   └── ioc.yaml
│   ├── ingress.yaml
│   ├── hpa.yaml
│   ├── networkpolicy.yaml
│   └── pdb.yaml
└── charts/                        # 中间件子 Chart（bitnami）
```

---

## 三、全局 values.yaml

```yaml
global:
  imageRegistry: "registry.chem-park.local"
  imagePullSecrets: ["chem-park-registry"]
  tenantId: "park_001"
  storageClass: "local-ssd"        # 高速存储(时序/PG)
  backupStorageClass: "nfs-backup"

domain: "ioc.chem-park.example"

gateway:
  replicas: 2
  resources:
    requests: { cpu: "500m", memory: "512Mi" }
    limits:   { cpu: "1",    memory: "1Gi" }

services:
  # 通用默认；各服务可覆盖
  resources:
    requests: { cpu: "250m", memory: "512Mi" }
    limits:   { cpu: "1",    memory: "1Gi" }
  javaOpts: "-Xms512m -Xmx1g -XX:+UseG1GC -Dfile.encoding=UTF-8"

  # 高频/关键服务单独配置
  monitor:
    replicas: 3
    priorityClass: high-priority
    resources:
      requests: { cpu: "500m", memory: "1Gi" }
      limits:   { cpu: "2",    memory: "2Gi" }
  emergency:
    replicas: 2
    priorityClass: high-priority
  ioc:
    replicas: 3
    priorityClass: high-priority

middleware:
  postgres:   { replicas: 2, storage: "100Gi" }
  timescaledb:{ replicas: 2, storage: "500Gi" }
  kafka:      { brokers: 3 }
  redis:      { replicas: 3, mode: sentinel }
  nacos:      { replicas: 3 }
  minio:      { replicas: 4, storage: "1Ti" }
```

---

## 四、中间件部署

### 4.1 Nacos（注册 + 配置中心，3 节点）

```yaml
# templates/middleware/nacos.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: nacos
  namespace: chem-park-middleware
spec:
  serviceName: nacos-headless
  replicas: 3
  selector:
    matchLabels: { app: nacos }
  template:
    metadata:
      labels: { app: nacos }
    spec:
      containers:
        - name: nacos
          image: "{{ .Values.global.imageRegistry }}/nacos-server:2.3.2"
          ports:
            - { containerPort: 8848, name: http }
            - { containerPort: 7848, name: raft }
          env:
            - { name: MODE, value: "cluster" }
            - { name: NACOS_REPLICAS, value: "3" }
            - { name: MYSQL_SERVICE_HOST, value: "postgres-middleware" }
          readinessProbe:
            httpGet: { path: /nacos/health/ready, port: 8848 }
            initialDelaySeconds: 30
          volumeMounts:
            - { name: data, mountPath: /home/nacos/data }
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: nacos-data
```

### 4.2 Kafka（3 broker + 副本）

```yaml
# templates/middleware/kafka.yaml (摘要)
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: kafka
  namespace: chem-park-middleware
spec:
  replicas: 3
  serviceName: kafka-headless
  template:
    spec:
      containers:
        - name: kafka
          image: "{{ .Values.global.imageRegistry }}/bitnami/kafka:3.7"
          env:
            - { name: KAFKA_CFG_NUM_PARTITIONS, value: "12" }
            - { name: KAFKA_CFG_DEFAULT_REPLICATION_FACTOR, value: "3" }
            - { name: KAFKA_CFG_MIN_INSYNC_REPLICAS, value: "2" }
            - { name: KAFKA_CFG_AUTO_CREATE_TOPICS_ENABLE, value: "false" }
          ports:
            - { containerPort: 9092, name: tcp }
```

> `min.insync.replicas=2` + `acks=all` 保障事件不丢（见事件设计文档）。

### 4.3 TimescaleDB / PostgreSQL

```yaml
# templates/middleware/timescaledb.yaml (摘要)
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: timescaledb
  namespace: chem-park-middleware
spec:
  replicas: 2
  template:
    spec:
      containers:
        - name: timescaledb
          image: "{{ .Values.global.imageRegistry }}/timescale/timescaledb:2.14-pg14"
          env:
            - { name: POSTGRES_PASSWORD, valueFrom:
                secretKeyRef: { name: db-secret, key: ts-pass } }
          volumeMounts:
            - { name: pgdata, mountPath: /var/lib/postgresql/data }
          readinessProbe:
            exec:
              command: ["pg_isready", "-U", "postgres"]
```

---

## 五、微服务 Deployment + Service 示例

### 5.1 通用模板 `_service.tpl`

```yaml
{{- define "chem.service" -}}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .name }}
  namespace: chem-park-app
  labels: { app: {{ .name }} }
spec:
  replicas: {{ .replicas | default 1 }}
  {{- if .priorityClass }}
  template:
    metadata:
      labels: { app: {{ .name }} }
    spec:
      priorityClassName: {{ .priorityClass }}
  {{- else }}
  template:
    metadata:
      labels: { app: {{ .name }} }
    spec:
  {{- end }}
      containers:
        - name: {{ .name }}
          image: "{{ $.Values.global.imageRegistry }}/{{ .name }}:{{ .tag | default "latest" }}"
          ports:
            - { containerPort: 8080 }
          env:
            - { name: NACOS_ADDR, value: "nacos.chem-park-middleware:8848" }
            - { name: KAFKA_ADDR, value: "kafka.chem-park-middleware:9092" }
            - { name: REDIS_ADDR, value: "redis.chem-park-middleware:6379" }
            - { name: TENANT_ID, value: "{{ $.Values.global.tenantId }}" }
            - { name: JAVA_OPTS, value: "{{ $.Values.services.javaOpts }}" }
          envFrom:
            - secretRef: { name: app-secret }
          readinessProbe:
            httpGet: { path: /actuator/health/readiness, port: 8080 }
            initialDelaySeconds: 20
          livenessProbe:
            httpGet: { path: /actuator/health/liveness, port: 8080 }
            initialDelaySeconds: 40
          resources:
            requests: {{ toYaml .resources.requests | nindent 14 }}
            limits:   {{ toYaml .resources.limits | nindent 14 }}
          securityContext:
            runAsNonRoot: true
            readOnlyRootFilesystem: true
            allowPrivilegeEscalation: false
---
apiVersion: v1
kind: Service
metadata:
  name: {{ .name }}
  namespace: chem-park-app
spec:
  selector: { app: {{ .name }} }
  ports:
    - { port: 8080, targetPort: 8080 }
{{- end -}}
```

### 5.2 monitor-svc 实例化（高频隔离）

```yaml
# templates/services/monitor.yaml
{{- include "chem.service" (dict
    "name" "monitor-svc"
    "replicas" .Values.services.monitor.replicas
    "priorityClass" .Values.services.monitor.priorityClass
    "resources" .Values.services.monitor.resources
    "tag" "v1.3"
    "Values" .Values) -}}
```

> `emergency-svc` / `ioc-svc` 同理，均带 `high-priority` 与多副本。

---

## 六、Ingress / Gateway 暴露

```yaml
# templates/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: chem-park-ingress
  namespace: chem-park-app
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "30"
    nginx.ingress.kubernetes.io/limit-rps: "50"
spec:
  ingressClassName: nginx
  tls:
    - hosts: ["{{ .Values.domain }}"]
      secretName: chem-park-tls
  rules:
    - host: "{{ .Values.domain }}"
      http:
        paths:
          - { path: /api/iam, pathType: Prefix, backend: { service: { name: gateway-svc, port: { number: 8080 } } } }
          - { path: /api/safety, pathType: Prefix, backend: { service: { name: gateway-svc, port: { number: 8080 } } } }
          - { path: /api/emergency, pathType: Prefix, backend: { service: { name: gateway-svc, port: { number: 8080 } } } }
          - { path: /api/ioc, pathType: Prefix, backend: { service: { name: gateway-svc, port: { number: 8080 } } } }
```

> 外部统一走 `gateway-svc`（Spring Cloud Gateway），由网关做路由/鉴权/限流；Ingress 仅做 TLS 终止与域名暴露。

---

## 七、HPA / 资源配额 / 反亲和

```yaml
# templates/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: monitor-hpa
  namespace: chem-park-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: monitor-svc
  minReplicas: 3
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target: { type: Utilization, averageUtilization: 70 }
---
# 反亲和：同服务 Pod 打散到不同节点
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels: { app: monitor-svc }
        topologyKey: kubernetes.io/hostname
```

---

## 八、安全上下文与网络策略

```yaml
# templates/networkpolicy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: app-isolate
  namespace: chem-park-app
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
  ingress:
    - from:
        - namespaceSelector: { matchLabels: { kubernetes.io/metadata.name: chem-park-app } }
        - podSelector: { matchLabels: { app: gateway-svc } }
      ports:
        - { protocol: TCP, port: 8080 }
  egress:
    - to:
        - namespaceSelector: { matchLabels: { kubernetes.io/metadata.name: chem-park-middleware } }
    - to:
        - podSelector: { matchLabels: { app: kafka } }
```

> 默认拒绝跨命名空间流量，仅放行 Gateway 入向与中间件出向，满足等保网络隔离。

---

## 九、部署命令

```bash
# 1. 添加中间件子 Chart 依赖
helm dependency build chem-park-platform

# 2. 渲染检查
helm template chem-park-platform -n chem-park-app --create-namespace > render.yaml

# 3. 部署（私有化）
helm upgrade --install chem-park chem-park-platform \
  -n chem-park-app --create-namespace \
  -f values.yaml -f values-prod.yaml

# 4. 验证
kubectl get pods -n chem-park-app
kubectl get pods -n chem-park-middleware

# 5. 滚动升级（灰度）
helm upgrade chem-park chem-park-platform --set services.monitor.tag=v1.4 -n chem-park-app

# 6. 备份（PG/TimescaleDB 定时）
kubectl exec -n chem-park-middleware timescaledb-0 -- \
  pg_dumpall | gzip > /backup/ts-$(date +%F).sql.gz
```

---

*文档结束 · 版本 V1.0*
