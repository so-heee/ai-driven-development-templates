# セキュリティパターンガイド

## コンテナセキュリティ

### Dockerセキュリティベストプラクティス
```dockerfile
# セキュアなベースイメージの使用
FROM node:18-alpine

# セキュリティアップデートの適用
RUN apk update && apk upgrade && \
    apk add --no-cache dumb-init && \
    rm -rf /var/cache/apk/*

# 非rootユーザーの作成
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nextjs -u 1001 -G nodejs

# 最小権限でのファイル配置
WORKDIR /app
COPY --chown=nextjs:nodejs package*.json ./
RUN npm ci --only=production && npm cache clean --force

COPY --chown=nextjs:nodejs . .

# 不要なパッケージの削除
RUN apk del .build-deps

# ユーザー切り替え
USER nextjs

# セキュリティオプションの設定
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:3000/health || exit 1

# 読み取り専用ルートファイルシステム対応
VOLUME ["/tmp", "/var/log"]

ENTRYPOINT ["dumb-init", "--"]
CMD ["node", "server.js"]
```

### コンテナランタイムセキュリティ
```yaml
# Pod Security Standards
apiVersion: v1
kind: Pod
metadata:
  name: secure-app
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1001
    runAsGroup: 1001
    fsGroup: 1001
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: myapp:latest
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
    resources:
      limits:
        memory: "256Mi"
        cpu: "250m"
      requests:
        memory: "128Mi"
        cpu: "100m"
    volumeMounts:
    - name: tmp-volume
      mountPath: /tmp
    - name: log-volume
      mountPath: /var/log
  volumes:
  - name: tmp-volume
    emptyDir: {}
  - name: log-volume
    emptyDir: {}
```

## シークレット管理

### Kubernetes Secrets
```yaml
# Secret作成
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
stringData:
  database-url: "postgresql://user:password@db:5432/myapp"
  api-key: "your-api-key-here"
  jwt-secret: "your-jwt-secret"

---
# ConfigMapとSecretの使い分け
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  NODE_ENV: "production"
  LOG_LEVEL: "info"
  CACHE_TTL: "3600"

---
# Deployment での使用
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:latest
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: database-url
        - name: NODE_ENV
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: NODE_ENV
        envFrom:
        - configMapRef:
            name: app-config
        - secretRef:
            name: app-secrets
```

### Sealed Secrets
```yaml
# SealedSecret の使用
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: app-secrets
  namespace: myapp-prod
spec:
  encryptedData:
    database-url: AgBy3i4OJSWK+PiTySYZZA9rO43cGDEQAx...
    api-key: AgAAowqRQ0xqTZdyUMU0NgCnvb4WT6y7pF...
  template:
    metadata:
      name: app-secrets
      namespace: myapp-prod
    type: Opaque
```

### External Secrets Operator
```yaml
# SecretStore の設定
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
spec:
  provider:
    vault:
      server: "https://vault.example.com"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "myapp"

---
# ExternalSecret の定義
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: app-secrets
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: app-secrets
    creationPolicy: Owner
  data:
  - secretKey: database-url
    remoteRef:
      key: myapp/database
      property: url
  - secretKey: api-key
    remoteRef:
      key: myapp/external
      property: api-key
```

## ネットワークセキュリティ

### Network Policies
```yaml
# デフォルト拒否ポリシー
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress

---
# アプリケーション固有のポリシー
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: myapp-network-policy
spec:
  podSelector:
    matchLabels:
      app: myapp
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    ports:
    - protocol: TCP
      port: 3000
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 5432
  - to: []
    ports:
    - protocol: TCP
      port: 443  # HTTPS
    - protocol: TCP
      port: 53   # DNS
    - protocol: UDP
      port: 53   # DNS
```

### Service Mesh セキュリティ
```yaml
# Istio AuthorizationPolicy
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: myapp-authz
spec:
  selector:
    matchLabels:
      app: myapp
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/default/sa/frontend"]
    to:
    - operation:
        methods: ["GET", "POST"]
        paths: ["/api/*"]
    when:
    - key: request.headers[authorization]
      values: ["Bearer *"]

---
# PeerAuthentication
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: myapp-peer-auth
spec:
  selector:
    matchLabels:
      app: myapp
  mtls:
    mode: STRICT
```

## 脆弱性スキャン

### Trivy を使った継続的スキャン
```yaml
# Trivy Operator での脆弱性スキャン
apiVersion: aquasecurity.github.io/v1alpha1
kind: VulnerabilityReport
metadata:
  name: pod-myapp-vulnerability-report
spec:
  scanner:
    name: Trivy
  registry:
    server: docker.io
  artifact:
    repository: library/myapp
    tag: latest
```

### CI/CD での脆弱性チェック
```yaml
# GitHub Actions での脆弱性スキャン
name: Security Scan

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  security-scan:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Build Docker image
      run: docker build -t myapp:${{ github.sha }} .
    
    - name: Run Trivy vulnerability scanner
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: 'myapp:${{ github.sha }}'
        format: 'sarif'
        output: 'trivy-results.sarif'
    
    - name: Upload Trivy scan results to GitHub Security tab
      uses: github/codeql-action/upload-sarif@v2
      if: always()
      with:
        sarif_file: 'trivy-results.sarif'
    
    - name: Fail on HIGH or CRITICAL vulnerabilities
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: 'myapp:${{ github.sha }}'
        format: 'table'
        exit-code: '1'
        ignore-unfixed: true
        vuln-type: 'os,library'
        severity: 'CRITICAL,HIGH'
```

## RBAC とアクセス制御

### Kubernetes RBAC
```yaml
# ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-service-account
  namespace: myapp-prod

---
# Role
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: myapp-prod
  name: myapp-role
rules:
- apiGroups: [""]
  resources: ["configmaps", "secrets"]
  verbs: ["get", "list"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]

---
# RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: myapp-rolebinding
  namespace: myapp-prod
subjects:
- kind: ServiceAccount
  name: myapp-service-account
  namespace: myapp-prod
roleRef:
  kind: Role
  name: myapp-role
  apiGroup: rbac.authorization.k8s.io

---
# Deployment での ServiceAccount 使用
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      serviceAccountName: myapp-service-account
      automountServiceAccountToken: false  # 必要でない場合は無効化
```

### OPA Gatekeeper ポリシー
```yaml
# ConstraintTemplate
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
      validation:
        properties:
          labels:
            type: array
            items:
              type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredlabels

        violation[{"msg": msg}] {
          required := input.parameters.labels
          provided := input.review.object.metadata.labels
          missing := required[_]
          not provided[missing]
          msg := sprintf("Missing required label: %v", [missing])
        }

---
# Constraint
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: must-have-security-labels
spec:
  match:
    kinds:
      - apiGroups: ["apps"]
        kinds: ["Deployment"]
  parameters:
    labels: ["security-scanned", "version"]
```

## セキュリティ監視とログ

### Falco ルール
```yaml
# カスタム Falco ルール
- rule: Suspicious network activity
  desc: Detect suspicious network connections
  condition: >
    spawned_process and
    (proc.name in (nc, ncat, netcat) or
     proc.cmdline contains "wget" or
     proc.cmdline contains "curl")
  output: >
    Suspicious network activity detected
    (user=%user.name command=%proc.cmdline
     container=%container.name)
  priority: WARNING

- rule: Container with privileged capabilities
  desc: Detect containers running with privileged capabilities
  condition: >
    container and
    (proc.name in (docker, runc, containerd) and
     proc.cmdline contains "--privileged")
  output: >
    Container running with privileged capabilities
    (user=%user.name command=%proc.cmdline
     container=%container.name)
  priority: CRITICAL
```

### セキュリティメトリクス
```typescript
import { Counter, Histogram } from 'prom-client';

// セキュリティ関連メトリクス
export const securityMetrics = {
  authAttempts: new Counter({
    name: 'auth_attempts_total',
    help: 'Total authentication attempts',
    labelNames: ['result', 'method'],
  }),
  
  authFailures: new Counter({
    name: 'auth_failures_total',
    help: 'Total authentication failures',
    labelNames: ['reason', 'ip'],
  }),
  
  rateLimitHits: new Counter({
    name: 'rate_limit_hits_total',
    help: 'Total rate limit hits',
    labelNames: ['endpoint', 'ip'],
  }),
  
  suspiciousActivity: new Counter({
    name: 'suspicious_activity_total',
    help: 'Suspicious activity detected',
    labelNames: ['type', 'severity'],
  }),
  
  tlsHandshakeTime: new Histogram({
    name: 'tls_handshake_duration_seconds',
    help: 'TLS handshake duration',
    buckets: [0.01, 0.05, 0.1, 0.5, 1.0],
  }),
};

// セキュリティミドルウェア
export const securityMiddleware = (req: Request, res: Response, next: NextFunction) => {
  const startTime = Date.now();
  
  // IP-based rate limiting check
  const clientIP = req.ip || req.connection.remoteAddress;
  
  // Suspicious pattern detection
  const suspiciousPatterns = [
    /(\.|%2e){2,}/, // Path traversal
    /(union|select|insert|update|delete|drop)/i, // SQL injection
    /<script|javascript:/i, // XSS attempts
    /\.(exe|bat|cmd|sh)$/i, // Executable files
  ];
  
  const requestContent = JSON.stringify(req.query) + JSON.stringify(req.body) + req.path;
  
  suspiciousPatterns.forEach((pattern, index) => {
    if (pattern.test(requestContent)) {
      securityMetrics.suspiciousActivity.inc({
        type: ['path_traversal', 'sql_injection', 'xss_attempt', 'executable_upload'][index],
        severity: 'high',
      });
    }
  });
  
  res.on('finish', () => {
    const duration = (Date.now() - startTime) / 1000;
    
    if (req.secure) {
      securityMetrics.tlsHandshakeTime.observe(duration);
    }
  });
  
  next();
};
```

## インシデント対応

### セキュリティインシデント検知
```yaml
# Prometheus アラートルール
groups:
  - name: security.rules
    rules:
      - alert: HighAuthFailureRate
        expr: |
          rate(auth_failures_total[5m]) > 10
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High authentication failure rate"
          description: "Authentication failure rate is {{ $value }} per second"

      - alert: SuspiciousActivityDetected
        expr: |
          increase(suspicious_activity_total[5m]) > 5
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Suspicious activity detected"
          description: "{{ $value }} suspicious activities in the last 5 minutes"

      - alert: PodWithPrivilegedAccess
        expr: |
          kube_pod_security_policy_privileged == 1
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: "Pod running with privileged access"
          description: "Pod {{ $labels.pod }} is running with privileged access"
```

### 自動対応アクション
```yaml
# Kubernetes Job for automated response
apiVersion: batch/v1
kind: Job
metadata:
  name: security-response
spec:
  template:
    spec:
      containers:
      - name: response
        image: security-tools:latest
        command:
        - /bin/bash
        - -c
        - |
          # Isolate suspicious pod
          kubectl patch pod $SUSPICIOUS_POD -p '{"spec":{"containers":[{"name":"'$CONTAINER_NAME'","securityContext":{"readOnlyRootFilesystem":true}}]}}'
          
          # Scale down deployment if necessary
          kubectl scale deployment $DEPLOYMENT_NAME --replicas=0
          
          # Create forensic snapshot
          kubectl create job forensic-capture --from=cronjob/forensic-job
          
          # Notify security team
          curl -X POST $SLACK_WEBHOOK_URL -d '{"text":"Security incident detected and contained"}'
      restartPolicy: Never
```

## コンプライアンス

### CIS Benchmarks 準拠
```yaml
# Pod Security Standards - Restricted
apiVersion: v1
kind: Namespace
metadata:
  name: secure-namespace
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted

---
# CIS Kubernetes Benchmark 準拠設定
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
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
    image: myapp:latest
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      runAsNonRoot: true
      runAsUser: 1000
      capabilities:
        drop:
        - ALL
        add:
        - NET_BIND_SERVICE
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```

### セキュリティポリシーの自動化
```yaml
# Kustomize を使ったセキュリティポリシーの適用
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

patchesStrategicMerge:
  - security-patches.yaml

patches:
  - target:
      kind: Deployment
    patch: |-
      - op: add
        path: /spec/template/spec/securityContext
        value:
          runAsNonRoot: true
          runAsUser: 1001
      - op: add
        path: /spec/template/spec/containers/0/securityContext
        value:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
```

## トラブルシューティング

### セキュリティ診断コマンド
```bash
# Pod のセキュリティ設定確認
kubectl get pod myapp -o jsonpath='{.spec.securityContext}'

# RBAC 権限確認
kubectl auth can-i --list --as=system:serviceaccount:myapp:myapp-sa

# Network Policy の確認
kubectl describe networkpolicy myapp-network-policy

# Secret の暗号化確認
kubectl get secrets -o json | jq '.items[].data | keys'

# 脆弱性スキャン結果確認
kubectl get vulnerabilityreports

# Falco アラート確認
kubectl logs -n falco daemonset/falco
```

### セキュリティ設定の検証
```bash
# CIS Benchmark スキャン
kube-bench run --targets node,policies,managedservices

# Pod Security Standards 確認
kubectl --dry-run=server create -f pod.yaml

# OPA Gatekeeper 制約確認
kubectl get constraints

# Istio セキュリティポリシー確認
istioctl authz check pod/myapp-pod
```