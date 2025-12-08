# Kubernetes Interview Questions and Answers

## Table of Contents

1. [Kubernetes Architecture](#kubernetes-architecture)
2. [Pods and Workloads](#pods-and-workloads)
3. [Services and Networking](#services-and-networking)
4. [Storage](#storage)
5. [Configuration Management](#configuration-management)
6. [Security and RBAC](#security-and-rbac)
7. [Scheduling and Resource Management](#scheduling-and-resource-management)
8. [Monitoring and Troubleshooting](#monitoring-and-troubleshooting)

---

## Kubernetes Architecture

### Q1: Explain Kubernetes architecture and components.

**Answer:**

```
+------------------------------------------------------------------+
|                     Kubernetes Cluster                            |
+------------------------------------------------------------------+
|                                                                    |
|  +------------------------+    +-------------------------------+  |
|  |     Control Plane      |    |         Worker Nodes          |  |
|  |                        |    |                               |  |
|  |  +------------------+  |    |  +-------------------------+  |  |
|  |  |   API Server     |  |    |  |       kubelet           |  |  |
|  |  +------------------+  |    |  +-------------------------+  |  |
|  |  +------------------+  |    |  +-------------------------+  |  |
|  |  |      etcd        |  |    |  |      kube-proxy         |  |  |
|  |  +------------------+  |    |  +-------------------------+  |  |
|  |  +------------------+  |    |  +-------------------------+  |  |
|  |  | Controller Mgr   |  |    |  |  Container Runtime      |  |  |
|  |  +------------------+  |    |  +-------------------------+  |  |
|  |  +------------------+  |    |  +-------------------------+  |  |
|  |  |    Scheduler     |  |    |  |        Pods             |  |  |
|  |  +------------------+  |    |  +-------------------------+  |  |
|  +------------------------+    +-------------------------------+  |
+------------------------------------------------------------------+
```

**Control Plane Components:**

| Component | Description |
|-----------|-------------|
| API Server | Frontend for Kubernetes, handles REST requests |
| etcd | Distributed key-value store for cluster data |
| Controller Manager | Runs controllers (node, replication, endpoints) |
| Scheduler | Assigns pods to nodes based on resources |
| Cloud Controller | Interfaces with cloud provider APIs |

**Worker Node Components:**

| Component | Description |
|-----------|-------------|
| kubelet | Agent ensuring containers run in pods |
| kube-proxy | Network proxy implementing Services |
| Container Runtime | Runs containers (containerd, CRI-O) |

---

### Q2: What is the difference between a Deployment, StatefulSet, and DaemonSet?

**Answer:**

| Workload | Use Case | Characteristics |
|----------|----------|-----------------|
| Deployment | Stateless applications | Rolling updates, scaling, replicas |
| StatefulSet | Stateful applications | Stable network IDs, ordered deployment |
| DaemonSet | Per-node workloads | One pod per node, system services |

**Deployment (Stateless):**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: web
          image: nginx:1.24
          ports:
            - containerPort: 80
```

**StatefulSet (Stateful):**

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 3
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
          image: postgres:15
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 10Gi
```

**DaemonSet (Per-Node):**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
spec:
  selector:
    matchLabels:
      app: fluentd
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      containers:
        - name: fluentd
          image: fluentd:latest
          volumeMounts:
            - name: varlog
              mountPath: /var/log
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
```

---

### Q3: Explain the Pod lifecycle.

**Answer:**

**Pod Phases:**

| Phase | Description |
|-------|-------------|
| Pending | Pod accepted but containers not created |
| Running | At least one container running |
| Succeeded | All containers terminated successfully |
| Failed | All containers terminated, at least one failed |
| Unknown | Pod state cannot be determined |

**Container States:**

| State | Description |
|-------|-------------|
| Waiting | Container not running (pulling image, etc.) |
| Running | Container executing |
| Terminated | Container finished execution |

**Pod Conditions:**

```bash
kubectl get pod mypod -o jsonpath='{.status.conditions}'
```

| Condition | Description |
|-----------|-------------|
| PodScheduled | Pod scheduled to node |
| Initialized | Init containers completed |
| ContainersReady | All containers ready |
| Ready | Pod ready to serve requests |

---

## Pods and Workloads

### Q4: What are Init Containers?

**Answer:**

Init containers run before app containers start, used for setup tasks.

**Characteristics:**
- Run sequentially (one after another)
- Must complete successfully before app containers start
- Can have different images than app containers
- Share volumes with app containers

**Use cases:**
- Wait for dependencies
- Clone Git repositories
- Generate configuration files
- Run database migrations

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  initContainers:
    - name: wait-for-db
      image: busybox
      command: ['sh', '-c', 'until nc -z postgres 5432; do echo waiting; sleep 2; done']
    
    - name: init-config
      image: busybox
      command: ['sh', '-c', 'echo "initializing..." > /config/init.txt']
      volumeMounts:
        - name: config
          mountPath: /config
  
  containers:
    - name: app
      image: myapp:latest
      volumeMounts:
        - name: config
          mountPath: /app/config
  
  volumes:
    - name: config
      emptyDir: {}
```

---

### Q5: Explain Kubernetes probes (liveness, readiness, startup).

**Answer:**

| Probe | Purpose | Failure Action |
|-------|---------|----------------|
| Liveness | Is container alive? | Restart container |
| Readiness | Is container ready for traffic? | Remove from Service endpoints |
| Startup | Has container started? | Kill container |

**Probe configuration:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
    - name: app
      image: myapp:latest
      
      # Liveness probe
      livenessProbe:
        httpGet:
          path: /healthz
          port: 8080
        initialDelaySeconds: 15
        periodSeconds: 10
        timeoutSeconds: 5
        failureThreshold: 3
        successThreshold: 1
      
      # Readiness probe
      readinessProbe:
        httpGet:
          path: /ready
          port: 8080
        initialDelaySeconds: 5
        periodSeconds: 5
        failureThreshold: 3
      
      # Startup probe (for slow-starting containers)
      startupProbe:
        httpGet:
          path: /healthz
          port: 8080
        failureThreshold: 30
        periodSeconds: 10
        # App has 30 * 10 = 300 seconds to start
```

**Probe types:**

```yaml
# HTTP GET
httpGet:
  path: /health
  port: 8080
  httpHeaders:
    - name: Custom-Header
      value: value

# TCP Socket
tcpSocket:
  port: 3306

# Exec command
exec:
  command:
    - cat
    - /tmp/healthy

# gRPC (Kubernetes 1.24+)
grpc:
  port: 50051
  service: health
```

---

### Q6: How do you perform rolling updates and rollbacks?

**Answer:**

**Rolling Update Strategy:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Max pods above desired during update
      maxUnavailable: 1  # Max pods unavailable during update
  template:
    spec:
      containers:
        - name: app
          image: myapp:v2
```

**Update commands:**

```bash
# Update image
kubectl set image deployment/myapp app=myapp:v2

# Apply updated manifest
kubectl apply -f deployment.yaml

# Check rollout status
kubectl rollout status deployment/myapp

# Pause rollout
kubectl rollout pause deployment/myapp

# Resume rollout
kubectl rollout resume deployment/myapp
```

**Rollback commands:**

```bash
# View rollout history
kubectl rollout history deployment/myapp

# View specific revision
kubectl rollout history deployment/myapp --revision=2

# Rollback to previous version
kubectl rollout undo deployment/myapp

# Rollback to specific revision
kubectl rollout undo deployment/myapp --to-revision=2
```

---

## Services and Networking

### Q7: Explain Kubernetes Service types.

**Answer:**

| Type | Access | Use Case |
|------|--------|----------|
| ClusterIP | Internal only | Backend services |
| NodePort | Node IP:Port | Development, on-prem |
| LoadBalancer | External LB | Cloud production |
| ExternalName | DNS CNAME | External services |

**ClusterIP (default):**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: 8080
```

**NodePort:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  type: NodePort
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080  # Range: 30000-32767
```

**LoadBalancer:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: nlb
spec:
  type: LoadBalancer
  selector:
    app: frontend
  ports:
    - port: 80
      targetPort: 8080
```

**ExternalName:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: db.external-service.com
```

---

### Q8: Explain Kubernetes Ingress.

**Answer:**

Ingress manages external HTTP/HTTPS access to services.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - app.example.com
        - api.example.com
      secretName: app-tls
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
    - host: api.example.com
      http:
        paths:
          - path: /v1
            pathType: Prefix
            backend:
              service:
                name: api-v1
                port:
                  number: 80
          - path: /v2
            pathType: Prefix
            backend:
              service:
                name: api-v2
                port:
                  number: 80
```

**Path types:**

| PathType | Behavior |
|----------|----------|
| Prefix | Matches URL path prefix |
| Exact | Matches exact URL path |
| ImplementationSpecific | Depends on IngressClass |

---

### Q9: What are Network Policies?

**Answer:**

Network Policies control pod-to-pod traffic (firewall rules).

**Default deny all:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}  # Apply to all pods
  policyTypes:
    - Ingress
    - Egress
```

**Allow specific traffic:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
    - Ingress
    - Egress
  
  ingress:
    # Allow from frontend pods
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080
    
    # Allow from monitoring namespace
    - from:
        - namespaceSelector:
            matchLabels:
              name: monitoring
  
  egress:
    # Allow to database
    - to:
        - podSelector:
            matchLabels:
              app: postgres
      ports:
        - protocol: TCP
          port: 5432
    
    # Allow DNS
    - to:
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
```

---

## Storage

### Q10: Explain Kubernetes storage concepts.

**Answer:**

**Storage hierarchy:**

```
StorageClass (defines provisioner and parameters)
     |
     v
PersistentVolume (actual storage resource)
     |
     v
PersistentVolumeClaim (request for storage)
     |
     v
Pod (mounts the PVC)
```

**StorageClass:**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-storage
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
  iopsPerGB: "10"
  fsType: ext4
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

**PersistentVolume:**

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-data
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: fast-storage
  awsElasticBlockStore:
    volumeID: vol-xxx
    fsType: ext4
```

**PersistentVolumeClaim:**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
  storageClassName: fast-storage
```

**Pod using PVC:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
    - name: app
      image: myapp
      volumeMounts:
        - name: data
          mountPath: /app/data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: data-claim
```

**Access modes:**

| Mode | Description |
|------|-------------|
| ReadWriteOnce (RWO) | Single node read-write |
| ReadOnlyMany (ROX) | Multiple nodes read-only |
| ReadWriteMany (RWX) | Multiple nodes read-write |
| ReadWriteOncePod (RWOP) | Single pod read-write |

---

## Configuration Management

### Q11: Explain ConfigMaps and Secrets.

**Answer:**

| ConfigMap | Secret |
|-----------|--------|
| Non-sensitive configuration | Sensitive data |
| Stored as plain text | Base64 encoded |
| Environment vars, config files | Passwords, tokens, keys |

**ConfigMap:**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  # Simple key-value
  DATABASE_HOST: "postgres"
  LOG_LEVEL: "info"
  
  # File content
  config.yaml: |
    server:
      port: 8080
      timeout: 30s
    database:
      pool_size: 10
```

**Secret:**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
stringData:  # Plain text (auto-encoded)
  DATABASE_PASSWORD: "mysecretpassword"
data:  # Already base64 encoded
  API_KEY: bXlhcGlrZXk=
```

**Using in Pod:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
    - name: app
      image: myapp
      
      # Environment from ConfigMap
      env:
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: LOG_LEVEL
        
        # Environment from Secret
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: DATABASE_PASSWORD
      
      # All values from ConfigMap
      envFrom:
        - configMapRef:
            name: app-config
      
      # Mount as files
      volumeMounts:
        - name: config-volume
          mountPath: /etc/config
          readOnly: true
        - name: secret-volume
          mountPath: /etc/secrets
          readOnly: true
  
  volumes:
    - name: config-volume
      configMap:
        name: app-config
    - name: secret-volume
      secret:
        secretName: app-secrets
        defaultMode: 0400
```

---

## Security and RBAC

### Q12: Explain Kubernetes RBAC.

**Answer:**

**RBAC Components:**

| Resource | Scope | Purpose |
|----------|-------|---------|
| Role | Namespace | Define permissions |
| ClusterRole | Cluster-wide | Define permissions |
| RoleBinding | Namespace | Assign Role to subjects |
| ClusterRoleBinding | Cluster-wide | Assign ClusterRole |

**Role:**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: development
  name: pod-manager
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log", "pods/exec"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch"]
```

**ClusterRole:**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: secret-reader
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list"]
    resourceNames: ["specific-secret"]  # Optional: limit to specific resources
```

**RoleBinding:**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-manager-binding
  namespace: development
subjects:
  - kind: User
    name: developer@example.com
    apiGroup: rbac.authorization.k8s.io
  - kind: ServiceAccount
    name: ci-bot
    namespace: development
  - kind: Group
    name: developers
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-manager
  apiGroup: rbac.authorization.k8s.io
```

**Verbs:**

| Verb | HTTP Method | Description |
|------|-------------|-------------|
| get | GET | Read single resource |
| list | GET | Read collection |
| watch | GET (streaming) | Watch for changes |
| create | POST | Create resource |
| update | PUT | Replace resource |
| patch | PATCH | Partial update |
| delete | DELETE | Delete resource |

---

### Q13: Explain Pod Security Standards.

**Answer:**

| Level | Description | Use Case |
|-------|-------------|----------|
| Privileged | No restrictions | System workloads |
| Baseline | Minimal restrictions | Standard workloads |
| Restricted | Hardened | Security-sensitive |

**Enforce via namespace labels:**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/audit: restricted
```

**Restricted requirements:**
- Must run as non-root
- Must drop all capabilities
- No privilege escalation
- No hostPath volumes
- No hostNetwork/hostPID/hostIPC
- Seccomp profile must be set

**Compliant Pod:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-app
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      image: myapp
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop:
            - ALL
      resources:
        limits:
          memory: "256Mi"
          cpu: "500m"
        requests:
          memory: "128Mi"
          cpu: "250m"
```

---

## Scheduling and Resource Management

### Q14: Explain resource requests and limits.

**Answer:**

| Requests | Limits |
|----------|--------|
| Guaranteed minimum | Maximum allowed |
| Used for scheduling | Enforced at runtime |
| Pod scheduled if node has capacity | Container killed/throttled if exceeded |

```yaml
spec:
  containers:
    - name: app
      resources:
        requests:
          memory: "256Mi"
          cpu: "250m"     # 0.25 CPU cores
        limits:
          memory: "512Mi"
          cpu: "500m"     # 0.5 CPU cores
```

**QoS Classes:**

| Class | Condition | Behavior |
|-------|-----------|----------|
| Guaranteed | requests = limits for all containers | First priority, last to evict |
| Burstable | requests < limits | Medium priority |
| BestEffort | No requests or limits | First to evict |

**LimitRange (namespace defaults):**

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: development
spec:
  limits:
    - type: Container
      default:
        cpu: "500m"
        memory: "256Mi"
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
      max:
        cpu: "2"
        memory: "2Gi"
      min:
        cpu: "50m"
        memory: "64Mi"
```

**ResourceQuota (namespace limits):**

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: development
spec:
  hard:
    requests.cpu: "10"
    requests.memory: "20Gi"
    limits.cpu: "20"
    limits.memory: "40Gi"
    pods: "50"
    persistentvolumeclaims: "10"
```

---

### Q15: Explain node affinity and pod affinity.

**Answer:**

**Node Selector (simple):**

```yaml
spec:
  nodeSelector:
    disktype: ssd
    zone: eu-west-1a
```

**Node Affinity (advanced):**

```yaml
spec:
  affinity:
    nodeAffinity:
      # Required (must match)
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: In
                values:
                  - eu-west-1a
                  - eu-west-1b
      
      # Preferred (try to match)
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          preference:
            matchExpressions:
              - key: node-type
                operator: In
                values:
                  - high-memory
```

**Pod Affinity (schedule near other pods):**

```yaml
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app: cache
          topologyKey: kubernetes.io/hostname
```

**Pod Anti-Affinity (spread pods apart):**

```yaml
spec:
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchLabels:
                app: web
            topologyKey: topology.kubernetes.io/zone
```

**Operators:**

| Operator | Description |
|----------|-------------|
| In | Label value in set |
| NotIn | Label value not in set |
| Exists | Label key exists |
| DoesNotExist | Label key does not exist |
| Gt | Label value greater than |
| Lt | Label value less than |

---

### Q16: What is Horizontal Pod Autoscaler?

**Answer:**

HPA automatically scales pods based on metrics.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 10
  
  metrics:
    # CPU utilization
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    
    # Memory utilization
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
    
    # Custom metric (from Prometheus)
    - type: Pods
      pods:
        metric:
          name: requests_per_second
        target:
          type: AverageValue
          averageValue: "1000"
  
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 10
          periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - type: Percent
          value: 100
          periodSeconds: 15
        - type: Pods
          value: 4
          periodSeconds: 15
      selectPolicy: Max
```

**Commands:**

```bash
# Create HPA imperatively
kubectl autoscale deployment myapp --cpu-percent=70 --min=2 --max=10

# View HPA status
kubectl get hpa
kubectl describe hpa app-hpa

# View HPA metrics
kubectl get hpa -o yaml
```

---

## Monitoring and Troubleshooting

### Q17: How do you troubleshoot a Pod stuck in Pending state?

**Answer:**

```bash
# 1. Check pod status
kubectl get pod <pod-name> -o wide

# 2. Describe pod for events
kubectl describe pod <pod-name>
# Look at Events section

# 3. Check cluster events
kubectl get events --sort-by='.lastTimestamp'

# 4. Check node resources
kubectl describe nodes | grep -A 5 "Allocated resources"
kubectl top nodes
```

**Common causes and solutions:**

| Cause | Diagnosis | Solution |
|-------|-----------|----------|
| Insufficient resources | Events show "Insufficient cpu/memory" | Scale cluster or reduce requests |
| No matching nodes | Node selector/affinity not matched | Check labels, adjust selectors |
| PVC pending | PVC not bound | Check StorageClass, provisioner |
| Image pull error | ImagePullBackOff | Check image name, registry access |
| Taints/tolerations | Node tainted | Add tolerations to pod |

---

### Q18: How do you debug a CrashLoopBackOff?

**Answer:**

```bash
# 1. Check pod status and restart count
kubectl get pod <pod-name>

# 2. View logs from crashed container
kubectl logs <pod-name> --previous

# 3. Describe pod for events
kubectl describe pod <pod-name>

# 4. If container starts briefly, exec into it
kubectl exec -it <pod-name> -- /bin/sh

# 5. Override entrypoint for debugging
kubectl run debug --image=<image> --command -- sleep 3600
kubectl exec -it debug -- /bin/sh

# 6. Check resource limits
kubectl get pod <pod-name> -o yaml | grep -A 10 resources
```

**Common causes:**

| Cause | Solution |
|-------|----------|
| Application error | Check logs, fix code |
| Missing config/secrets | Verify ConfigMaps/Secrets exist |
| Database connection failed | Check network, credentials |
| OOMKilled | Increase memory limit |
| Permission denied | Check file permissions, security context |
| Missing dependencies | Verify container image |

---

### Q19: How do you troubleshoot network connectivity issues?

**Answer:**

```bash
# 1. Verify pods are running
kubectl get pods -o wide

# 2. Test DNS resolution
kubectl exec -it <pod> -- nslookup <service-name>
kubectl exec -it <pod> -- nslookup kubernetes.default

# 3. Test connectivity to service
kubectl exec -it <pod> -- curl <service>:<port>
kubectl exec -it <pod> -- wget -qO- <service>:<port>

# 4. Check service endpoints
kubectl get endpoints <service>
kubectl describe service <service>

# 5. Check Network Policies
kubectl get networkpolicies -A
kubectl describe networkpolicy <name>

# 6. Verify CNI plugin is healthy
kubectl get pods -n kube-system | grep -E 'calico|flannel|weave|cilium'

# 7. Check kube-proxy
kubectl get pods -n kube-system -l k8s-app=kube-proxy
kubectl logs -n kube-system -l k8s-app=kube-proxy

# 8. Use debug container
kubectl debug -it <pod> --image=nicolaka/netshoot -- /bin/bash
```

---

### Q20: Essential kubectl commands for troubleshooting.

**Answer:**

```bash
# Cluster info
kubectl cluster-info
kubectl get nodes -o wide
kubectl top nodes
kubectl top pods

# Pod operations
kubectl get pods -A                    # All namespaces
kubectl get pods -o wide               # More details
kubectl get pods -w                    # Watch mode
kubectl get pods --show-labels
kubectl get pods -l app=myapp          # Filter by label

# Logs
kubectl logs <pod>
kubectl logs <pod> -c <container>      # Specific container
kubectl logs <pod> --previous          # Previous instance
kubectl logs <pod> -f                  # Follow
kubectl logs <pod> --tail=100
kubectl logs <pod> --since=1h
kubectl logs -l app=myapp              # By label

# Describe resources
kubectl describe pod <pod>
kubectl describe node <node>
kubectl describe service <service>

# Execute commands
kubectl exec -it <pod> -- /bin/sh
kubectl exec <pod> -- env
kubectl exec <pod> -- cat /etc/hosts

# Port forwarding
kubectl port-forward pod/<pod> 8080:80
kubectl port-forward svc/<service> 8080:80

# Copy files
kubectl cp <pod>:/path/file ./local-file
kubectl cp ./local-file <pod>:/path/file

# Events
kubectl get events --sort-by='.lastTimestamp'
kubectl get events --field-selector type=Warning

# Resource usage
kubectl top pods --containers
kubectl top pods --sort-by=memory

# Debug
kubectl debug -it <pod> --image=busybox
kubectl run debug --rm -it --image=busybox -- /bin/sh
```

---

## Summary

| Topic | Key Resources |
|-------|---------------|
| Workloads | Deployment, StatefulSet, DaemonSet, Job, CronJob |
| Networking | Service, Ingress, NetworkPolicy |
| Storage | PV, PVC, StorageClass |
| Config | ConfigMap, Secret |
| Security | RBAC, ServiceAccount, PodSecurityStandard |
| Scaling | HPA, VPA, Cluster Autoscaler |
| Scheduling | NodeSelector, Affinity, Taints/Tolerations |

**Quick reference:**

```bash
# Create resources
kubectl apply -f manifest.yaml
kubectl create deployment nginx --image=nginx

# Scale
kubectl scale deployment/myapp --replicas=5

# Update
kubectl set image deployment/myapp app=myapp:v2
kubectl rollout undo deployment/myapp

# Delete
kubectl delete -f manifest.yaml
kubectl delete pod <pod> --grace-period=0 --force
```