# Infrastructure as Code パターンガイド

## Terraform パターン

### 基本的なプロジェクト構造

```
terraform/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── terraform.tfvars
│   ├── staging/
│   └── production/
├── modules/
│   ├── vpc/
│   ├── ec2/
│   ├── rds/
│   └── efs/
└── shared/
    ├── backend.tf
    └── providers.tf
```

### モジュール設計パターン

```hcl
# modules/vpc/main.tf
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = merge(var.common_tags, {
    Name = "${var.project_name}-vpc"
  })
}

resource "aws_subnet" "public" {
  count = length(var.public_subnet_cidrs)

  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.public_subnet_cidrs[count.index]
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true

  tags = merge(var.common_tags, {
    Name = "${var.project_name}-public-${count.index + 1}"
    Type = "Public"
  })
}

# modules/vpc/variables.tf
variable "project_name" {
  description = "Name of the project"
  type        = string
}

variable "vpc_cidr" {
  description = "CIDR block for VPC"
  type        = string
  default     = "10.0.0.0/16"
}

variable "public_subnet_cidrs" {
  description = "CIDR blocks for public subnets"
  type        = list(string)
  default     = ["10.0.1.0/24", "10.0.2.0/24"]
}

variable "common_tags" {
  description = "Common tags to be applied to all resources"
  type        = map(string)
  default     = {}
}
```

### 環境別設定

```hcl
# environments/production/main.tf
terraform {
  backend "s3" {
    bucket = "myapp-terraform-state"
    key    = "production/terraform.tfstate"
    region = "ap-northeast-1"
  }
}

module "vpc" {
  source = "../../modules/vpc"

  project_name         = var.project_name
  vpc_cidr            = "10.0.0.0/16"
  public_subnet_cidrs = ["10.0.1.0/24", "10.0.2.0/24"]

  common_tags = {
    Environment = "production"
    Project     = var.project_name
    ManagedBy   = "terraform"
  }
}

module "application" {
  source = "../../modules/ecs"

  vpc_id           = module.vpc.vpc_id
  subnet_ids       = module.vpc.public_subnet_ids
  desired_capacity = 3

  common_tags = local.common_tags
}
```

## AWS CDK パターン

### TypeScript CDK スタック

```typescript
import * as cdk from 'aws-cdk-lib'
import * as ec2 from 'aws-cdk-lib/aws-ec2'
import * as ecs from 'aws-cdk-lib/aws-ecs'
import * as elbv2 from 'aws-cdk-lib/aws-elasticloadbalancingv2'
import { Construct } from 'constructs'

export class ApplicationStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props)

    // VPC
    const vpc = new ec2.Vpc(this, 'ApplicationVpc', {
      maxAzs: 2,
      subnetConfiguration: [
        {
          name: 'Public',
          subnetType: ec2.SubnetType.PUBLIC,
          cidrMask: 24,
        },
        {
          name: 'Private',
          subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS,
          cidrMask: 24,
        },
      ],
    })

    // ECS Cluster
    const cluster = new ecs.Cluster(this, 'ApplicationCluster', {
      vpc,
      containerInsights: true,
    })

    // Application Load Balancer
    const alb = new elbv2.ApplicationLoadBalancer(this, 'ApplicationALB', {
      vpc,
      internetFacing: true,
      securityGroup: this.createALBSecurityGroup(vpc),
    })

    // ECS Service
    this.createECSService(cluster, alb)
  }

  private createALBSecurityGroup(vpc: ec2.Vpc): ec2.SecurityGroup {
    const sg = new ec2.SecurityGroup(this, 'ALBSecurityGroup', {
      vpc,
      description: 'Security group for Application Load Balancer',
      allowAllOutbound: true,
    })

    sg.addIngressRule(ec2.Peer.anyIpv4(), ec2.Port.tcp(80), 'Allow HTTP traffic')

    sg.addIngressRule(ec2.Peer.anyIpv4(), ec2.Port.tcp(443), 'Allow HTTPS traffic')

    return sg
  }

  private createECSService(cluster: ecs.Cluster, alb: elbv2.ApplicationLoadBalancer): void {
    const taskDefinition = new ecs.FargateTaskDefinition(this, 'AppTaskDef', {
      memoryLimitMiB: 512,
      cpu: 256,
    })

    const container = taskDefinition.addContainer('app', {
      image: ecs.ContainerImage.fromRegistry('myapp:latest'),
      memoryLimitMiB: 512,
      logging: ecs.LogDrivers.awsLogs({
        streamPrefix: 'ecs',
        logRetention: logs.RetentionDays.ONE_WEEK,
      }),
      environment: {
        NODE_ENV: 'production',
      },
    })

    container.addPortMappings({
      containerPort: 3000,
      protocol: ecs.Protocol.TCP,
    })

    const service = new ecs.FargateService(this, 'AppService', {
      cluster,
      taskDefinition,
      desiredCount: 2,
      assignPublicIp: false,
    })

    // Target Group
    const targetGroup = new elbv2.ApplicationTargetGroup(this, 'AppTargetGroup', {
      vpc: cluster.vpc,
      port: 80,
      protocol: elbv2.ApplicationProtocol.HTTP,
      targetType: elbv2.TargetType.IP,
      healthCheck: {
        path: '/health',
        healthyHttpCodes: '200',
      },
    })

    service.attachToApplicationTargetGroup(targetGroup)

    // Listener
    alb.addListener('AppListener', {
      port: 80,
      defaultTargetGroups: [targetGroup],
    })
  }
}
```

## Kubernetes パターン

### Helm Chart 構造

```
charts/myapp/
├── Chart.yaml
├── values.yaml
├── values-dev.yaml
├── values-prod.yaml
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml
    ├── configmap.yaml
    └── secret.yaml
```

### Deployment テンプレート

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "myapp.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
      labels:
        {{- include "myapp.selectorLabels" . | nindent 8 }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      securityContext:
        {{- toYaml .Values.podSecurityContext | nindent 8 }}
      containers:
        - name: {{ .Chart.Name }}
          securityContext:
            {{- toYaml .Values.securityContext | nindent 12 }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: 3000
              protocol: TCP
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
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          env:
            - name: NODE_ENV
              value: {{ .Values.env.nodeEnv }}
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: {{ include "myapp.fullname" . }}-secret
                  key: database-url
```

### Kustomize パターン

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
  - configmap.yaml

commonLabels:
  app: myapp

# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: myapp-prod

resources:
  - ../../base

patchesStrategicMerge:
  - deployment-patch.yaml

replicas:
  - name: myapp
    count: 3

images:
  - name: myapp
    newTag: v1.2.3
```

## GitOps パターン

### ArgoCD Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-production
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/myapp-k8s
    targetRevision: HEAD
    path: overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: myapp-prod
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### Flux2 設定

```yaml
# clusters/production/myapp-source.yaml
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: GitRepository
metadata:
  name: myapp
  namespace: flux-system
spec:
  interval: 1m
  ref:
    branch: main
  url: https://github.com/myorg/myapp-k8s

---
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: myapp
  namespace: flux-system
spec:
  interval: 10m
  path: './overlays/production'
  prune: true
  sourceRef:
    kind: GitRepository
    name: myapp
```

## 監視・観測可能性

### Prometheus 設定

```yaml
# prometheus-config.yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - 'alert-rules.yml'

scrape_configs:
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)

alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093
```

### Grafana ダッシュボード

```json
{
  "dashboard": {
    "title": "Application Metrics",
    "panels": [
      {
        "title": "Request Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(http_requests_total[5m])",
            "legendFormat": "{{method}} {{status}}"
          }
        ]
      },
      {
        "title": "Response Time",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))",
            "legendFormat": "95th percentile"
          }
        ]
      }
    ]
  }
}
```

## セキュリティパターン

### Network Policies

```yaml
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
```

### Pod Security Standards

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: myapp-prod
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

## バックアップ・災害復旧

### Velero バックアップ

```yaml
apiVersion: velero.io/v1
kind: Backup
metadata:
  name: myapp-backup
spec:
  includedNamespaces:
    - myapp-prod
  storageLocation: default
  ttl: 720h
  snapshotVolumes: true
```

### データベースバックアップ

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: db-backup
spec:
  schedule: '0 2 * * *'
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: db-backup
              image: postgres:15
              command:
                - /bin/bash
                - -c
                - |
                  pg_dump $DATABASE_URL | gzip > /backup/backup-$(date +%Y%m%d).sql.gz
                  aws s3 cp /backup/backup-$(date +%Y%m%d).sql.gz s3://myapp-backups/
              env:
                - name: DATABASE_URL
                  valueFrom:
                    secretKeyRef:
                      name: db-secret
                      key: url
              volumeMounts:
                - name: backup-storage
                  mountPath: /backup
          restartPolicy: OnFailure
          volumes:
            - name: backup-storage
              emptyDir: {}
```

## トラブルシューティング

### リソース監視

```bash
# ノードリソース確認
kubectl top nodes

# Podリソース確認
kubectl top pods -n myapp-prod

# イベント確認
kubectl get events -n myapp-prod --sort-by='.lastTimestamp'

# ログ確認
kubectl logs -f deployment/myapp -n myapp-prod
```

### デバッグ用Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: debug-pod
spec:
  containers:
    - name: debug
      image: nicolaka/netshoot
      command: ['/bin/bash']
      args: ['-c', 'while true; do sleep 30; done;']
      securityContext:
        capabilities:
          add: ['NET_ADMIN']
```
