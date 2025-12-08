# Monitoring, Security, and DevOps Scenarios

## Table of Contents

1. [Monitoring and Observability](#monitoring-and-observability)
2. [Security and DevSecOps](#security-and-devsecops)
3. [Troubleshooting Scenarios](#troubleshooting-scenarios)
4. [Architecture and Design](#architecture-and-design)
5. [Linux Essentials](#linux-essentials)

---

## Monitoring and Observability

### Q1: Explain the three pillars of observability.

**Answer:**

| Pillar | Description | Tools |
|--------|-------------|-------|
| Metrics | Numeric measurements over time | Prometheus, CloudWatch |
| Logs | Event records with context | ELK, Loki, CloudWatch |
| Traces | Request flow across services | Jaeger, Zipkin, X-Ray |

---

### Q2: Explain the Four Golden Signals.

**Answer:**

| Signal | Description | Example |
|--------|-------------|---------|
| Latency | Time to serve request | p50, p95, p99 response time |
| Traffic | Demand on system | Requests/second |
| Errors | Failed request rate | HTTP 5xx rate |
| Saturation | System fullness | CPU%, memory%, queue depth |

**Prometheus recording rules:**

```yaml
groups:
  - name: golden_signals
    rules:
      - record: http_request_duration_seconds:p99
        expr: histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))

      - record: http_requests:rate5m
        expr: rate(http_requests_total[5m])

      - record: http_error_rate:ratio
        expr: rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m])
```

---

### Q3: Explain RED and USE methods.

**Answer:**

**RED Method (for services):**

| Metric | Description |
|--------|-------------|
| Rate | Requests per second |
| Errors | Failed requests per second |
| Duration | Time per request |

**USE Method (for resources):**

| Metric | Description |
|--------|-------------|
| Utilization | % time resource busy |
| Saturation | Queue length |
| Errors | Error count |

---

### Q4: Explain Prometheus PromQL basics.

**Answer:**

```promql
# Instant vector
http_requests_total

# Label filtering
http_requests_total{method="GET", status="200"}
http_requests_total{status=~"5.."}

# Rate (per-second over time)
rate(http_requests_total[5m])

# Sum by label
sum(rate(http_requests_total[5m])) by (service)

# Histogram percentiles
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))

# Aggregations
avg(cpu_usage) by (instance)
max(memory_usage) by (pod)
```

---

### Q5: Explain ELK Stack.

**Answer:**

| Component | Role |
|-----------|------|
| Elasticsearch | Search engine, stores logs |
| Logstash | Log processing |
| Kibana | Visualization |
| Beats | Log shippers |

**Flow:** App -> Filebeat -> Logstash -> Elasticsearch -> Kibana

---

## Security and DevSecOps

### Q6: Explain OWASP Top 10.

**Answer:**

| # | Vulnerability | Prevention |
|---|---------------|------------|
| 1 | Broken Access Control | RBAC, deny by default |
| 2 | Cryptographic Failures | Strong encryption |
| 3 | Injection | Parameterized queries |
| 4 | Insecure Design | Threat modeling |
| 5 | Security Misconfiguration | Hardening |
| 6 | Vulnerable Components | Dependency scanning |
| 7 | Auth Failures | MFA, rate limiting |
| 8 | Data Integrity Failures | Code signing |
| 9 | Logging Failures | Comprehensive logging |
| 10 | SSRF | Input validation |

---

### Q7: Container security best practices.

**Answer:**

```dockerfile
# 1. Minimal base images
FROM alpine:3.18

# 2. Run as non-root
RUN adduser -D appuser
USER appuser

# 3. Specific tags
FROM python:3.11.4-slim

# 4. Multi-stage builds
FROM maven:3.8 AS builder
FROM openjdk:17-slim
COPY --from=builder /app/target/app.jar .
```

**Kubernetes security context:**

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - ALL
```

---

### Q8: SAST vs DAST.

**Answer:**

| SAST (Static) | DAST (Dynamic) |
|---------------|----------------|
| Analyzes source code | Tests running app |
| Early in pipeline | Later in pipeline |
| SonarQube, Semgrep | OWASP ZAP, Burp |

---

### Q9: Secrets management best practices.

**Answer:**

| Practice | Description |
|----------|-------------|
| Never hardcode | No secrets in code |
| Use secret managers | Vault, AWS Secrets Manager |
| Rotate regularly | Automated rotation |
| Encrypt at rest | Encryption |
| Audit access | Logging |
| Least privilege | Minimal access |

---

### Q10: Zero Trust security.

**Answer:**

"Never trust, always verify"

**Principles:**
1. Verify explicitly
2. Least privilege access
3. Assume breach

**Implementation:**
- Service mesh (mTLS)
- Network policies
- Identity-based access

---

## Troubleshooting Scenarios

### Q11: Pod stuck in Pending - troubleshooting.

**Answer:**

```bash
kubectl get pod <pod> -o wide
kubectl describe pod <pod>
kubectl get events --sort-by='.lastTimestamp'
kubectl describe nodes | grep -A 5 "Allocated resources"
```

**Common causes:**

| Cause | Solution |
|-------|----------|
| Insufficient resources | Scale cluster |
| No matching nodes | Check selectors |
| PVC pending | Check StorageClass |
| ImagePullBackOff | Check image/registry |

---

### Q12: CrashLoopBackOff debugging.

**Answer:**

```bash
kubectl logs <pod> --previous
kubectl describe pod <pod>
kubectl run debug --image=<image> --command -- sleep 3600
```

**Common causes:**

| Cause | Solution |
|-------|----------|
| Application error | Check logs |
| Missing config | Verify ConfigMaps |
| OOMKilled | Increase memory |

---

### Q13: 502 errors after deployment.

**Answer:**

```bash
# Check pods
kubectl get pods
kubectl logs <pod>

# Check readiness
kubectl get endpoints <service>

# Check service
kubectl get svc <service> -o yaml

# Test internally
kubectl exec -it <debug> -- curl <service>:<port>
```

**Common causes:**
- App crashed
- Readiness probe failing
- Wrong port configuration
- OOMKill

---

### Q14: Pipeline takes 3x longer.

**Answer:**

**Check:**
1. Build logs for slow stages
2. Resource utilization
3. Network latency
4. Cache hits
5. Infrastructure changes

---

### Q15: Terraform apply fails midway.

**Answer:**

```bash
# Check state
terraform state list
terraform state show <resource>

# Fix and retry
terraform apply

# If corrupted
terraform import <resource> <id>
terraform state rm <resource>

# Debug
TF_LOG=DEBUG terraform apply
```

---

## Architecture and Design

### Q16: CI/CD pipeline for microservices.

**Answer:**

```
Developer Push
     |
CI Stage (per service): Lint -> Build -> Test -> Scan
     |
CD Stage: Dev -> Staging -> Production
```

**Key considerations:**
- Service-specific pipelines
- Shared libraries
- Contract testing
- Canary deployments

---

### Q17: Disaster recovery for Kubernetes.

**Answer:**

| Tier | RTO | RPO | Strategy |
|------|-----|-----|----------|
| Critical | <15min | <1min | Active-active |
| Important | <1hr | <15min | Active-passive |
| Standard | <4hr | <1hr | Backup/restore |

**Velero backup:**

```yaml
apiVersion: velero.io/v1
kind: Schedule
spec:
  schedule: "0 2 * * *"
  template:
    includedNamespaces:
      - production
    ttl: 720h
```

---

### Q18: GitOps multi-environment.

**Answer:**

```
gitops-repo/
+-- apps/
|   +-- base/
|   +-- overlays/
|       +-- dev/
|       +-- staging/
|       +-- production/
```

**Argo CD ApplicationSet:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
spec:
  generators:
    - list:
        elements:
          - cluster: dev
          - cluster: staging
          - cluster: production
  template:
    spec:
      source:
        path: 'apps/overlays/{{cluster}}'
      syncPolicy:
        automated:
          selfHeal: true
```

---

### Q19: Zero-downtime deployment.

**Answer:**

```yaml
# Rolling update
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0

# Readiness probe
readinessProbe:
  httpGet:
    path: /ready
    port: 8080

# PodDisruptionBudget
apiVersion: policy/v1
kind: PodDisruptionBudget
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: api
```

---

### Q20: Configuration drift handling.

**Answer:**

**Detection:**

```bash
terraform plan -detailed-exitcode
```

**Prevention:**

```yaml
syncPolicy:
  automated:
    selfHeal: true
```

---

## Linux Essentials

### Q21: Essential commands.

**Answer:**

```bash
# System resources
top / htop
free -h
df -h
iostat -x 1

# Process
ps aux | grep <process>
kill <pid>

# Network
netstat -tuln
ss -tuln
curl -v <url>

# Logs
tail -f /var/log/syslog
journalctl -u nginx -f

# Service
systemctl status nginx
systemctl restart nginx
```

---

### Q22: Find process using port.

**Answer:**

```bash
lsof -i :8080
netstat -tlnp | grep 8080
fuser -k 8080/tcp
```

---

### Q23: File permissions.

**Answer:**

```
-rwxr-xr-- owner group
 |  |  |
 |  |  +-- Others: r--
 |  +----- Group: r-x
 +-------- Owner: rwx

chmod 755 = rwxr-xr-x
chmod 644 = rw-r--r--
```

---

## Summary

| Category | Key Topics |
|----------|------------|
| Monitoring | Golden signals, RED, USE, PromQL |
| Security | OWASP, containers, secrets |
| Troubleshooting | Kubernetes, pipelines, networking |
| Architecture | CI/CD, DR, GitOps |
| Linux | Commands, permissions |

**Security checklist:**
- Run as non-root
- Minimal base images
- Scan vulnerabilities
- Manage secrets properly
- Least privilege
- Audit logging

**Troubleshooting workflow:**
1. Gather information
2. Identify symptom
3. Form hypothesis
4. Test and validate
5. Implement fix
6. Document