# Fiveline Kubernetes 매니페스트 가이드

> 이 문서는 `fiveline_manifest` 레포지토리와 `fiveline_terraform/k8s/` 디렉토리에 들어갈  
> 모든 Kubernetes 매니페스트를 정의하는 기준 문서입니다.  
> 다른 레포지토리에서도 이 문서를 프롬프트로 사용할 수 있습니다.

---

## 레포지토리별 역할 분리

| 위치                      | 역할                               | 담당         |
| ------------------------- | ---------------------------------- | ------------ |
| `fiveline_manifest/`      | 앱 배포 매니페스트 (ArgoCD가 감시) | CI/CD 담당자 |
| `fiveline_terraform/k8s/` | 인프라 수준 매니페스트 (1회 적용)  | 각 담당자    |

**분리 이유**: `fiveline_backend` GitHub Actions가 이미지 빌드 후 `fiveline_manifest` 레포에만 커밋하면, ArgoCD가 감지하여 자동 배포합니다. 앱 코드와 매니페스트를 같은 레포에 두면 Actions가 같은 레포에 커밋 → 다시 Actions 트리거 → 무한 루프가 발생합니다.

---

## 공통 정보

| 항목                  | 값                                                      |
| --------------------- | ------------------------------------------------------- |
| EKS 클러스터명        | `fiveline-eks`                                          |
| Kubernetes 버전       | `1.35`                                                  |
| 리전                  | `ap-northeast-2`                                        |
| 앱 네임스페이스       | `fiveline`                                              |
| On-Demand 노드 레이블 | `workload=stable`                                       |
| Spot 노드 taint       | `spot=true:NoSchedule`                                  |
| 헬스체크 경로         | `/api/health`                                           |
| ECR 레지스트리        | `<AWS_ACCOUNT_ID>.dkr.ecr.ap-northeast-2.amazonaws.com` |

---

---

# Part 1: fiveline_manifest 레포지토리

ArgoCD가 감시하는 GitOps 전용 레포지토리. GitHub Actions(fiveline_backend)가 이미지 빌드 후 이 레포의 이미지 태그만 업데이트합니다.

## 디렉토리 구조

```
fiveline_manifest/
├── README.md
├── base/
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   ├── user-service/
│   │   ├── kustomization.yaml
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── serviceaccount.yaml
│   │   ├── hpa.yaml
│   │   └── pdb.yaml
│   ├── product-service/
│   │   └── (user-service와 동일 구조, 포트 8002)
│   ├── order-service/
│   │   └── (user-service와 동일 구조, 포트 8003, HPA 60%)
│   ├── ingress.yaml
│   └── external-secrets/
│       ├── kustomization.yaml
│       ├── secretstore.yaml
│       └── externalsecret.yaml
└── overlays/
    └── deploy/
        ├── kustomization.yaml
        └── argocd-application.yaml
```

---

## base/namespace.yaml

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: fiveline
  labels:
    project: fiveline
    managed-by: argocd
```

---

## base/kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: fiveline

resources:
  - namespace.yaml
  - user-service/
  - product-service/
  - order-service/
  - ingress.yaml
  - external-secrets/
```

---

## base/user-service/kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - serviceaccount.yaml
  - deployment.yaml
  - service.yaml
  - hpa.yaml
  - pdb.yaml
```

---

## base/user-service/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
  namespace: fiveline
  labels:
    app: user-service
    project: fiveline
spec:
  replicas: 2
  selector:
    matchLabels:
      app: user-service
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0 # 배포 중 서비스 중단 없음
      maxSurge: 1 # 1개 추가 Pod 생성 후 교체
  template:
    metadata:
      labels:
        app: user-service
        project: fiveline
    spec:
      serviceAccountName: user-service-sa # IRSA 바인딩
      terminationGracePeriodSeconds: 30

      # Spot 노드 오버플로 허용 (On-Demand 포화 시 Spot 노드에도 스케줄링)
      tolerations:
        - key: "spot"
          operator: "Equal"
          value: "true"
          effect: "NoSchedule"

      # AZ 간 Pod 분산 (preferred: 불가능해도 스케줄링은 허용)
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values:
                        - user-service
                topologyKey: topology.kubernetes.io/zone

      containers:
        - name: user-service
          # GitHub Actions가 kustomize edit set image로 자동 업데이트
          image: <AWS_ACCOUNT_ID>.dkr.ecr.ap-northeast-2.amazonaws.com/fiveline/user-service:<IMAGE_TAG>
          imagePullPolicy: Always
          ports:
            - name: http
              containerPort: 8001
              protocol: TCP

          # ALB 연결 해제 전 5초 대기 (graceful shutdown)
          lifecycle:
            preStop:
              exec:
                command: ["sh", "-c", "sleep 5"]

          # ESO가 Secrets Manager → K8s Secret(fiveline-secret)으로 동기화한 값 전체 주입
          envFrom:
            - secretRef:
                name: fiveline-secret

          env:
            - name: SERVICE_NAME
              value: "user-service"
            - name: PORT
              value: "8001"

          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"

          readinessProbe:
            httpGet:
              path: /api/health
              port: 8001
            initialDelaySeconds: 10
            periodSeconds: 10
            failureThreshold: 3
            successThreshold: 1
            timeoutSeconds: 3

          livenessProbe:
            httpGet:
              path: /api/health
              port: 8001
            initialDelaySeconds: 30
            periodSeconds: 15
            failureThreshold: 3
            timeoutSeconds: 5

          securityContext:
            runAsNonRoot: true
            runAsUser: 1000
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop:
                - ALL

      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
```

---

## base/user-service/service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: user-service
  namespace: fiveline
  labels:
    app: user-service
spec:
  type: ClusterIP
  selector:
    app: user-service
  ports:
    - name: http
      port: 8001
      targetPort: 8001
      protocol: TCP
```

---

## base/user-service/serviceaccount.yaml

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: user-service-sa
  namespace: fiveline
  labels:
    app: user-service
  annotations:
    # IRSA: 이 SA를 사용하는 Pod는 fiveline-user-service-sa-role로 AWS API 호출
    # 보안 담당자가 iam.tf에서 생성한 IAM Role ARN
    eks.amazonaws.com/role-arn: arn:aws:iam::<AWS_ACCOUNT_ID>:role/fiveline-user-service-sa-role
    eks.amazonaws.com/token-expiration: "86400"
```

---

## base/user-service/hpa.yaml

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: user-service-hpa
  namespace: fiveline
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: user-service
  minReplicas: 2
  maxReplicas: 6
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Pods
          value: 2
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Pods
          value: 1
          periodSeconds: 120
```

---

## base/user-service/pdb.yaml

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: user-service-pdb
  namespace: fiveline
spec:
  selector:
    matchLabels:
      app: user-service
  # Spot 회수/노드 드레인 시 최소 1개 Pod 보장
  minAvailable: 1
```

---

## base/product-service/ (포트 8002, CPU 70%)

`user-service`와 동일 구조. 아래만 변경:

| 항목                  | 변경값                             |
| --------------------- | ---------------------------------- |
| 모든 `user-service` → | `product-service`                  |
| `containerPort`       | `8002`                             |
| `port` (Service)      | `8002`                             |
| `targetPort`          | `8002`                             |
| IRSA role-arn         | `fiveline-product-service-sa-role` |
| HPA CPU               | `70` (동일)                        |

---

## base/order-service/ (포트 8003, CPU 60% — 선제 발동)

`user-service`와 동일 구조. 아래만 변경:

| 항목                      | 변경값                                      |
| ------------------------- | ------------------------------------------- |
| 모든 `user-service` →     | `order-service`                             |
| `containerPort`           | `8003`                                      |
| `port` (Service)          | `8003`                                      |
| `targetPort`              | `8003`                                      |
| IRSA role-arn             | `fiveline-order-service-sa-role`            |
| HPA CPU                   | `60` (DB 트랜잭션 집중으로 선제 스케일아웃) |
| resources.requests.cpu    | `150m` (주문 처리 부하 반영)                |
| resources.requests.memory | `256Mi`                                     |

---

## base/ingress.yaml

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: fiveline-ingress
  namespace: fiveline
  labels:
    project: fiveline
  annotations:
    # ALB Controller가 이 Ingress를 감지하여 ALB를 자동 생성
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip

    # 보안 담당자가 acm.tf에서 생성한 ap-northeast-2 인증서 ARN
    alb.ingress.kubernetes.io/certificate-arn: <ACM_CERT_ARN>
    alb.ingress.kubernetes.io/ssl-redirect: "443"

    # 보안 담당자가 waf.tf에서 생성한 REGIONAL WAF WebACL ARN
    alb.ingress.kubernetes.io/wafv2-web-acl-arn: <WAF_REGIONAL_ARN>

    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
    alb.ingress.kubernetes.io/load-balancer-name: fiveline-alb

    alb.ingress.kubernetes.io/healthcheck-path: /api/health
    alb.ingress.kubernetes.io/healthy-threshold-count: "2"
    alb.ingress.kubernetes.io/unhealthy-threshold-count: "3"
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: "15"
    alb.ingress.kubernetes.io/healthcheck-timeout-seconds: "5"

    # ALB를 배치할 퍼블릭 서브넷 ID (network.tf의 public_2a, public_2c)
    alb.ingress.kubernetes.io/subnets: <PUBLIC_SUBNET_2A_ID>,<PUBLIC_SUBNET_2C_ID>

spec:
  ingressClassName: alb
  rules:
    - http:
        paths:
          - path: /api/auth
            pathType: Prefix
            backend:
              service:
                name: user-service
                port:
                  number: 8001
          - path: /api/users
            pathType: Prefix
            backend:
              service:
                name: user-service
                port:
                  number: 8001
          - path: /api/products
            pathType: Prefix
            backend:
              service:
                name: product-service
                port:
                  number: 8002
          - path: /api/cart
            pathType: Prefix
            backend:
              service:
                name: product-service
                port:
                  number: 8002
          - path: /api/orders
            pathType: Prefix
            backend:
              service:
                name: order-service
                port:
                  number: 8003
          - path: /api/health
            pathType: Prefix
            backend:
              service:
                name: user-service
                port:
                  number: 8001
```

---

## base/external-secrets/kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - secretstore.yaml
  - externalsecret.yaml
```

---

## base/external-secrets/secretstore.yaml

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: fiveline-aws-sm
  namespace: fiveline
spec:
  provider:
    aws:
      service: SecretsManager
      region: ap-northeast-2
      auth:
        # IRSA: ESO ServiceAccount(external-secrets 네임스페이스)가 SM에 접근
        jwt:
          serviceAccountRef:
            name: external-secrets-sa
            namespace: external-secrets # ESO 설치 네임스페이스 명시 필수
```

---

## base/external-secrets/externalsecret.yaml

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: fiveline-external-secret
  namespace: fiveline
spec:
  refreshInterval: 60m # 60분마다 Secrets Manager 값 재동기화

  secretStoreRef:
    name: fiveline-aws-sm
    kind: SecretStore

  target:
    name: fiveline-secret # 생성될 K8s Secret 이름 (Deployment의 secretRef와 일치)
    creationPolicy: Owner
    deletionPolicy: Retain

  data:
    # Secrets Manager 경로: fiveline/app/db-credential
    # JSON 형태로 저장: { "DB_HOST": "...", "DB_PORT": "5432", "DB_NAME": "fiveline",
    #                     "DB_USER": "fiveline_app", "DB_PASSWORD": "..." }
    - secretKey: DB_HOST
      remoteRef:
        key: fiveline/app/db-credential
        property: DB_HOST
    - secretKey: DB_PORT
      remoteRef:
        key: fiveline/app/db-credential
        property: DB_PORT
    - secretKey: DB_NAME
      remoteRef:
        key: fiveline/app/db-credential
        property: DB_NAME
    - secretKey: DB_USER
      remoteRef:
        key: fiveline/app/db-credential
        property: DB_USER
    - secretKey: DB_PASSWORD
      remoteRef:
        key: fiveline/app/db-credential
        property: DB_PASSWORD

    # Secrets Manager 경로: fiveline/app/jwt-signing-key
    # JSON 형태로 저장: { "JWT_SECRET_KEY": "..." }
    - secretKey: JWT_SECRET_KEY
      remoteRef:
        key: fiveline/app/jwt-signing-key
        property: JWT_SECRET_KEY
```

---

## overlays/deploy/kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base
  - argocd-application.yaml

namespace: fiveline

commonLabels:
  environment: prod

# GitHub Actions가 kustomize edit set image 명령으로 자동 업데이트
# 최초 배포 시 아래 images 블록은 비어있으며, Actions 첫 실행 시 자동 삽입됨
images: []
```

---

## overlays/deploy/argocd-application.yaml

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: fiveline
  namespace: argocd
  labels:
    project: fiveline
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default

  source:
    # fiveline_manifest 레포지토리 URL
    repoURL: https://github.com/<GITHUB_ORG>/fiveline_manifest
    targetRevision: main
    path: overlays/deploy

  destination:
    server: https://kubernetes.default.svc
    namespace: fiveline

  syncPolicy:
    # manual-sync: ArgoCD UI 또는 CLI로 명시적 동기화 (argocd app sync fiveline)
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

---

## GitHub Actions 이미지 태그 업데이트 방법

`fiveline_backend/.github/workflows/cicd.yml` 핵심 단계:

```yaml
- name: Set image tag
  id: meta
  run: echo "IMAGE_TAG=sha-$(git rev-parse --short HEAD)" >> $GITHUB_OUTPUT

- name: Checkout fiveline_manifest
  uses: actions/checkout@v4
  with:
    repository: <GITHUB_ORG>/fiveline_manifest
    token: ${{ secrets.MANIFEST_REPO_TOKEN }}
    path: fiveline_manifest

- name: Update image tags
  env:
    ECR: <AWS_ACCOUNT_ID>.dkr.ecr.ap-northeast-2.amazonaws.com
    TAG: ${{ steps.meta.outputs.IMAGE_TAG }}
  run: |
    cd fiveline_manifest/overlays/deploy

    kustomize edit set image \
      $ECR/fiveline/user-service=$ECR/fiveline/user-service:$TAG
    kustomize edit set image \
      $ECR/fiveline/product-service=$ECR/fiveline/product-service:$TAG
    kustomize edit set image \
      $ECR/fiveline/order-service=$ECR/fiveline/order-service:$TAG

- name: Commit and push
  run: |
    cd fiveline_manifest
    git config user.name "github-actions[bot]"
    git config user.email "github-actions[bot]@users.noreply.github.com"
    git add overlays/deploy/kustomization.yaml
    git diff --staged --quiet || \
      git commit -m "ci: update image tags to ${{ steps.meta.outputs.IMAGE_TAG }}"
    git push origin main
```

---

---

# Part 2: fiveline_terraform/k8s/ 인프라 매니페스트

**인프라 수준 매니페스트**: 각 담당자가 Helm 또는 kubectl로 1회 적용하는 컴포넌트.  
앱 배포 매니페스트(Deployment, Service 등)는 `fiveline_manifest`에 위치한다.

## 디렉토리 구조

```
fiveline_terraform/
└── k8s/
    ├── fluent-bit/
    │   ├── configmap.yaml    # 데이터 파이프라인 담당자
    │   └── daemonset.yaml    # 데이터 파이프라인 담당자
    └── argocd/
        └── fiveline-application.yaml   # CI/CD 담당자 (ArgoCD 설치 후 적용)
```

---

## k8s/fluent-bit/configmap.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: kube-system
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush             5
        Daemon            Off
        Log_Level         info
        Parsers_File      parsers.conf
        HTTP_Server       On
        HTTP_Listen       0.0.0.0
        HTTP_Port         2020

    [INPUT]
        Name              tail
        Tag               kube.*
        Path              /var/log/containers/*.log
        # kube-system, monitoring 네임스페이스 제외 (Fluent Bit 자신의 로그 무한 루프 방지)
        Exclude_Path      /var/log/containers/*_kube-system_*.log,/var/log/containers/*_monitoring_*.log,/var/log/containers/*_external-secrets_*.log,/var/log/containers/*_argocd_*.log
        multiline.parser  cri
        DB                /var/log/flb_kube.db
        DB.Sync           Normal
        Mem_Buf_Limit     50MB
        Skip_Long_Lines   On
        Refresh_Interval  10

    [FILTER]
        Name                kubernetes
        Match               kube.*
        Kube_URL            https://kubernetes.default.svc:443
        Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token
        Kube_Tag_Prefix     kube.var.log.containers.
        Merge_Log           On
        Merge_Log_Key       log_processed
        K8S-Logging.Parser  On
        K8S-Logging.Exclude Off

    # log_type 필드 기준으로 태그 분기
    # SERVICE_EVENT → service_event.* (Firehose로 S3 적재)
    # 그 외 → app_log.* (일반 애플리케이션 로그)
    [FILTER]
        Name          rewrite_tag
        Match         kube.*
        Rule          $log_processed['log_type'] SERVICE_EVENT service_event.$TAG false
        Rule          $log_processed['log_type'] .* app_log.$TAG false
        Emitter_Name  re_emitted

    # 서비스 이벤트 → CloudWatch (30일 보존, Subscription Filter로 Firehose 연결)
    [OUTPUT]
        Name                  cloudwatch_logs
        Match                 service_event.*
        region                ap-northeast-2
        log_group_name        /fiveline/service-events/$(kubernetes['labels']['app'])
        log_stream_prefix     pod/
        auto_create_group     true
        log_retention_days    30
        net.connect_timeout   10
        Retry_Limit           5

    # 애플리케이션 로그 → CloudWatch (14일 보존)
    [OUTPUT]
        Name                  cloudwatch_logs
        Match                 app_log.*
        region                ap-northeast-2
        log_group_name        /fiveline/app-logs/$(kubernetes['labels']['app'])
        log_stream_prefix     pod/
        auto_create_group     true
        log_retention_days    14
        Retry_Limit           5

  parsers.conf: |
    [PARSER]
        Name        cri
        Format      regex
        Regex       ^(?<time>[^ ]+) (?<stream>stdout|stderr) (?<logtag>[^ ]*) (?<log>.*)$
        Time_Key    time
        Time_Format %Y-%m-%dT%H:%M:%S.%L%z
        Time_Keep   On
```

---

## k8s/fluent-bit/daemonset.yaml

```yaml
# Fluent Bit ServiceAccount (IRSA 바인딩)
apiVersion: v1
kind: ServiceAccount
metadata:
  name: fluent-bit
  namespace: kube-system
  labels:
    app: fluent-bit
  annotations:
    # 데이터 파이프라인 담당자가 iam.tf에 추가한 Fluent Bit IRSA Role ARN
    eks.amazonaws.com/role-arn: arn:aws:iam::<AWS_ACCOUNT_ID>:role/fiveline-iam-fluent-bit
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  namespace: kube-system
  labels:
    app: fluent-bit
    project: fiveline
spec:
  selector:
    matchLabels:
      app: fluent-bit
  template:
    metadata:
      labels:
        app: fluent-bit
      annotations:
        # Prometheus가 Fluent Bit 메트릭 수집
        prometheus.io/scrape: "true"
        prometheus.io/port: "2020"
        prometheus.io/path: "/api/v1/metrics/prometheus"
    spec:
      serviceAccountName: fluent-bit
      terminationGracePeriodSeconds: 30

      # On-Demand + Spot 전 노드에 배포 (DaemonSet은 모든 노드에서 로그 수집 필수)
      tolerations:
        - key: "spot"
          operator: "Equal"
          value: "true"
          effect: "NoSchedule"
        - key: node-role.kubernetes.io/master
          effect: NoSchedule
        - operator: Exists # 기타 모든 taint 허용

      containers:
        - name: fluent-bit
          # AWS 공식 이미지: CloudWatch 플러그인 내장
          image: public.ecr.aws/aws-observability/aws-for-fluent-bit:stable
          imagePullPolicy: Always

          ports:
            - containerPort: 2020
              name: metrics
              protocol: TCP

          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"

          # /var/log 접근을 위해 root 실행 필요 (로그 수집 특성상 불가피)
          securityContext:
            runAsNonRoot: false
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: false

          livenessProbe:
            httpGet:
              path: /
              port: 2020
            initialDelaySeconds: 30
            periodSeconds: 30

          volumeMounts:
            - name: config
              mountPath: /fluent-bit/etc/
            - name: varlog
              mountPath: /var/log
              readOnly: true
            - name: varlibdockercontainers
              mountPath: /var/lib/docker/containers
              readOnly: true
            - name: flbdb
              mountPath: /var/log/flb_kube.db

      volumes:
        - name: config
          configMap:
            name: fluent-bit-config
        - name: varlog
          hostPath:
            path: /var/log
        - name: varlibdockercontainers
          hostPath:
            path: /var/lib/docker/containers
        - name: flbdb
          hostPath:
            path: /var/log/flb_kube.db
            type: FileOrCreate
```

---

## k8s/argocd/fiveline-application.yaml

```yaml
# ArgoCD Application 리소스
# ArgoCD 설치 후 kubectl apply로 1회 적용
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: fiveline
  namespace: argocd
  labels:
    project: fiveline
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/<GITHUB_ORG>/fiveline_manifest
    targetRevision: main
    path: overlays/deploy
  destination:
    server: https://kubernetes.default.svc
    namespace: fiveline
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

---

---

# Part 3: Helm 설치 명령어 모음

EKS 클러스터 생성 후 순서대로 실행합니다.

## 1. ArgoCD (CI/CD 담당자)

```bash
kubectl create namespace argocd

kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# ArgoCD Pod 기동 대기
kubectl wait --for=condition=available deployment/argocd-server \
  -n argocd --timeout=300s

# 초기 admin 비밀번호 확인
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

---

## 2. External Secrets Operator (보안 담당자)

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm repo update

helm install external-secrets external-secrets/external-secrets \
  --namespace external-secrets \
  --create-namespace \
  --set serviceAccount.name=external-secrets-sa \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=arn:aws:iam::<AWS_ACCOUNT_ID>:role/fiveline-eso-sa-role \
  --wait

# CRD 등록 확인
kubectl get crd | grep external-secrets
```

**노드 배치**: On-Demand 노드 (`workload=stable`) 권장 — `--set nodeSelector."workload"=stable`

---

## 3. AWS Load Balancer Controller (CI/CD 담당자)

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update

# VPC ID 확인 (terraform output 또는 AWS CLI)
VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=tag:Name,Values=fiveline-vpc" \
  --query "Vpcs[0].VpcId" --output text)

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  --namespace kube-system \
  --set clusterName=fiveline-eks \
  --set serviceAccount.create=true \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set "serviceAccount.annotations.eks\.amazonaws\.com/role-arn"=arn:aws:iam::<AWS_ACCOUNT_ID>:role/fiveline-iam-alb-controller \
  --set region=ap-northeast-2 \
  --set vpcId=$VPC_ID \
  --set nodeSelector."workload"=stable \
  --wait

# 정상 기동 확인
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller
```

---

## 4. Cluster Autoscaler (모니터링/알람 담당자)

> **선행 조건**: `eks.tf` 노드 그룹 tags에 CA 자동 발견 태그 추가 후 `terraform apply` 완료 필요
>
> ```hcl
> "k8s.io/cluster-autoscaler/enabled"       = "true"
> "k8s.io/cluster-autoscaler/fiveline-eks" = "owned"
> ```

```bash
helm repo add autoscaler https://kubernetes.github.io/autoscaler
helm repo update

helm install cluster-autoscaler autoscaler/cluster-autoscaler \
  --namespace kube-system \
  --set autoDiscovery.clusterName=fiveline-eks \
  --set awsRegion=ap-northeast-2 \
  --set "rbac.serviceAccount.annotations.eks\.amazonaws\.com/role-arn"=arn:aws:iam::<AWS_ACCOUNT_ID>:role/fiveline-iam-cluster-autoscaler \
  --set extraArgs.balance-similar-node-groups=true \
  --set extraArgs.skip-nodes-with-system-pods=false \
  --set extraArgs.scale-down-delay-after-add=5m \
  --set extraArgs.scale-down-unneeded-time=5m \
  --set extraArgs.expander=least-waste \
  --set nodeSelector."workload"=stable \
  --wait

# CA Pod 확인
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-cluster-autoscaler
```

---

## 5. Node Termination Handler (모니터링/알람 담당자)

```bash
# SQS Queue URL 확인 (terraform output 또는 AWS CLI)
SQS_URL=$(aws sqs get-queue-url \
  --queue-name fiveline-sqs-nth \
  --query QueueUrl --output text)

helm install aws-node-termination-handler eks/aws-node-termination-handler \
  --namespace kube-system \
  --set enableSqsTerminationDraining=true \
  --set queueURL=$SQS_URL \
  --set awsRegion=ap-northeast-2 \
  --set "serviceAccount.annotations.eks\.amazonaws\.com/role-arn"=arn:aws:iam::<AWS_ACCOUNT_ID>:role/fiveline-iam-nth \
  --set podTerminationGracePeriod=120 \
  --set nodeTerminationGracePeriod=120 \
  --set tolerations[0].operator=Exists \
  --wait

# NTH는 DaemonSet으로 전 노드에 배포 (Spot 포함 toleration 자동 설정)
kubectl get ds -n kube-system aws-node-termination-handler
```

---

## 6. Prometheus + kube-state-metrics + node-exporter (모니터링/알람 담당자)

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install fiveline-prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set prometheus.prometheusSpec.scrapeInterval=30s \
  --set prometheus.prometheusSpec.evaluationInterval=30s \
  --set prometheus.prometheusSpec.retention=15d \
  --set "prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage"=20Gi \
  --set kubeStateMetrics.enabled=true \
  --set nodeExporter.enabled=true \
  --set grafana.enabled=false \
  --set alertmanager.enabled=false \
  --set "prometheus.prometheusSpec.nodeSelector.workload"=stable \
  --wait

# node-exporter는 DaemonSet으로 전 노드에 배포되어야 함 (Spot 포함)
# kube-prometheus-stack 기본 설정으로 toleration 자동 적용됨
kubectl get ds -n monitoring
```

---

## 7. Grafana (모니터링/알람 담당자)

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Grafana admin 비밀번호는 Secrets Manager에서 관리 권장
# 교육 환경에서는 직접 설정 가능

helm install fiveline-grafana grafana/grafana \
  --namespace monitoring \
  --set adminPassword=<GRAFANA_ADMIN_PASSWORD> \
  --set persistence.enabled=true \
  --set persistence.size=5Gi \
  --set "serviceAccount.annotations.eks\.amazonaws\.com/role-arn"=arn:aws:iam::<AWS_ACCOUNT_ID>:role/fiveline-iam-grafana \
  --set "nodeSelector.workload"=stable \
  --set "datasources.datasources\.yaml.apiVersion"=1 \
  --set "datasources.datasources\.yaml.datasources[0].name"=Prometheus \
  --set "datasources.datasources\.yaml.datasources[0].type"=prometheus \
  --set "datasources.datasources\.yaml.datasources[0].url"=http://fiveline-prometheus-kube-prom-prometheus.monitoring:9090 \
  --set "datasources.datasources\.yaml.datasources[0].isDefault"=true \
  --set sidecar.dashboards.enabled=true \
  --wait
```

---

---

# Part 4: 플레이스홀더 전체 목록

| 플레이스홀더               | 설명                                  | 값을 얻는 방법                                                                   |
| -------------------------- | ------------------------------------- | -------------------------------------------------------------------------------- |
| `<AWS_ACCOUNT_ID>`         | AWS 12자리 계정 ID                    | `aws sts get-caller-identity --query Account --output text`                      |
| `<IMAGE_TAG>`              | ECR 이미지 태그                       | GitHub Actions가 `sha-$(git rev-parse --short HEAD)` 형식으로 자동 생성          |
| `<ACM_CERT_ARN>`           | ALB용 TLS 인증서 ARN (ap-northeast-2) | 보안 담당자 `acm.tf` 적용 후 `terraform output` 또는 `aws acm list-certificates` |
| `<WAF_REGIONAL_ARN>`       | REGIONAL WAF WebACL ARN               | 보안 담당자 `waf.tf` 적용 후 `aws wafv2 list-web-acls --scope REGIONAL`          |
| `<PUBLIC_SUBNET_2A_ID>`    | 퍼블릭 서브넷 ID (2a)                 | `terraform output public_subnet_ids` 또는 AWS 콘솔 VPC                           |
| `<PUBLIC_SUBNET_2C_ID>`    | 퍼블릭 서브넷 ID (2c)                 | `terraform output public_subnet_ids` 또는 AWS 콘솔 VPC                           |
| `<VPC_ID>`                 | VPC ID                                | `terraform output vpc_id`                                                        |
| `<GITHUB_ORG>`             | GitHub 조직명                         | GitHub 레포 URL 확인                                                             |
| `<GRAFANA_ADMIN_PASSWORD>` | Grafana admin 비밀번호                | 팀 내 공유 또는 Secrets Manager에서 관리                                         |
| IRSA Role ARN들            | 각 컴포넌트별 IAM Role ARN            | 각 담당자가 Terraform 적용 후 `terraform output` 또는 AWS 콘솔 IAM               |

---

# Part 5: 전체 적용 순서

```
[EKS 클러스터 kubeconfig 설정]
aws eks update-kubeconfig --region ap-northeast-2 --name fiveline-eks
        |
        v
[Helm 설치] — 의존성 순서 준수
  1. External Secrets Operator (CRD 등록 필수 선행)
  2. AWS Load Balancer Controller
  3. ArgoCD
  4. Cluster Autoscaler
  5. Node Termination Handler
  6. kube-prometheus-stack (Prometheus)
  7. Grafana
        |
        v
[fiveline_terraform/k8s/ 적용]
  kubectl apply -f k8s/fluent-bit/configmap.yaml
  kubectl apply -f k8s/fluent-bit/daemonset.yaml
  # Fluent Bit DaemonSet 전 노드 Running 확인
        |
        v
[fiveline_manifest 초기 적용 (ArgoCD 등록 전 1회)]
  kubectl apply -f base/namespace.yaml
  kubectl apply -f base/external-secrets/      # SecretSynced 확인 후 다음 단계
  kubectl apply -f base/user-service/serviceaccount.yaml
  kubectl apply -f base/product-service/serviceaccount.yaml
  kubectl apply -f base/order-service/serviceaccount.yaml
  kustomize build overlays/deploy | kubectl apply -f -   # Deployment/Service/HPA/PDB
  kubectl apply -f base/ingress.yaml           # ALB 자동 생성 (2~5분 대기)
        |
        v
[ArgoCD Application 등록]
  kubectl apply -f k8s/argocd/fiveline-application.yaml
  # 이후부터는 fiveline_manifest 레포 변경 → ArgoCD가 자동 감지
        |
        v
[검증]
  kubectl get pods -n fiveline          # 전체 Pod Running 확인
  kubectl get hpa -n fiveline           # Metrics 연동 확인
  kubectl get ingress -n fiveline       # ALB DNS 주소 확인
  kubectl get externalsecret -n fiveline # SecretSynced 확인
  curl https://<ALB_DNS>/api/health     # 엔드투엔드 확인
```
