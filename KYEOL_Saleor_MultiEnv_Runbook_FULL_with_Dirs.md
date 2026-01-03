# KYEOL Saleor 멀티환경 클라우드 아키텍처 런북 (FULL 버전)
- 기준일: 2026-01-03
- 프리픽스: **jung-**
- 도메인: **mgz-g2-u3.shop**
- 리전: **ap-northeast-1 (Tokyo)**

---

## 0. 프로젝트 목표
- Saleor 기반 이커머스 멀티환경(DEV/STAGE/PROD/MGMT) 아키텍처 구현
- 환경별 동일 UI/서비스를 운영하고, 도메인으로 환경 분리
- **IaC (Terraform) + GitOps (ArgoCD) + CI/CD (GitHub Actions)** 완벽 통합

---

## 1. 환경/도메인 정책

| 환경 | 구분 | 서비스 URL | API URL |
|------|------|------------|---------|
| **DEV** | 개발 | `https://dev.mgz-g2-u3.shop` | `https://dev-api.mgz-g2-u3.shop` |
| **STAGE** | 스테이징 | `https://stage.mgz-g2-u3.shop` | `https://stage-api.mgz-g2-u3.shop` |
| **PROD** | 운영 | `https://mgz-g2-u3.shop` | `https://api.mgz-g2-u3.shop` |

---

## 2. 핵심 아키텍처 원칙

### 2.1 로드밸런서 및 DNS 자동화
- **AWS Load Balancer Controller**: Ingress 리소스를 감지하여 ALB를 자동 생성.
- **ExternalDNS**: Ingress/Service 리소스를 감지하여 Route53 레코드를 자동 생성/갱신.

### 2.2 보안 및 규제 준수 (ISMS-P)
- **OIDC (OIDC Provider)**: GitHub Actions와 EKS IRSA를 위해 IAM OIDC 제공자 구성.
- **WAFv2**: CloudFront 앞단에 WAF를 배치하여 보안 강화 (us-east-1 리전에 생성).
- **PG 전용 NAT (PROD 전용)**: 결제 API 트래픽 격리를 위한 전용 NAT Gateway 구성.

---

## 3. 리소스 사이징 (기준)

| 리소스 | DEV | STAGE | PROD | MGMT |
|--------|-----|-------|------|------|
| **EKS Nodes** | t3.medium x 2 | t3.medium x 2-4 | t3.medium x 3-5 | t3.medium x 2 |
| **RDS (PostgreSQL)** | db.t3.small (Single) | db.t3.medium (Multi-AZ) | db.t3.medium (Multi-AZ) | N/A |
| **Cache (Valkey)** | cache.t3.micro (Single) | cache.t3.small (Replication) | cache.t3.small (Replication) | N/A |

---

## 4. 디렉토리 구조 표준

### 4.1 `kyeol-infra-terraform` (IaC)
```text
kyeol-infra-terraform/
├── envs/
│   ├── bootstrap/      # tfstate 저장소
│   ├── dev/            # 개발 환경
│   ├── mgmt/           # 관리 환경 (ArgoCD)
│   ├── stage/          # 스테이징 (템플릿)
│   └── prod/           # 운영 (템플릿)
├── modules/            # VPC, EKS, RDS, Valkey, ECR 등 공통 모듈
└── global/
    └── us-east-1/      # CloudFront ACM 및 글로벌 리소스
```

### 4.2 `kyeol-platform-gitops` (플랫폼 애드온)
```text
kyeol-platform-gitops/
├── argocd/
│   ├── app-of-apps/    # 루트 앱 및 프로젝트 정의
│   └── applications/   # 개별 애드온 (LBC, ExternalDNS 등)
└── clusters/
    └── dev/            # 클러스터별 설정 (Kustomize/Helm)
```

### 4.3 `kyeol-app-gitops` (애플리케이션 배포)
```text
kyeol-app-gitops/
├── apps/
│   └── saleor/
│       ├── base/       # 공통 매니페스트
│       └── overlays/   # 환경별 오버레이 (dev, stage, prod)
└── argocd/
    └── applications/   # Saleor 앱 정의
```

---

## 5. 배포 순서 (V2 가이드)

1. **Bootstrap**: `terraform apply` (State S3/DynamoDB 생성)
2. **MGMT 인프라**: `terraform apply` (MGMT EKS 생성)
3. **DEV 인프라**: `terraform apply` (DEV EKS + RDS + Valkey 생성)
4. **ArgoCD 설치**: MGMT 클러스터에 ArgoCD 배포
5. **클러스터 등록**: DEV 클러스터를 MGMT ArgoCD에 연결
6. **플랫폼 애드온**: Root App 배포하여 LBC, ExternalDNS 자동 설치
7. **CI/CD**: GitHub Actions로 ECR에 이미지 푸시
8. **앱 배포**: Saleor App Application 동기화
9. **검증**: `https://dev.mgz-g2-u3.shop` 접속 확인
