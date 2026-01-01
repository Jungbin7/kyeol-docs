# 🚀 jung-kyeol 플랫폼 엔지니어링 명세서 (V2)

- **상태**: 실무 아키텍처 확정 및 구현 대기
- **설계자**: jung (Senior Cloud Architect / Tech Lead)
- **리전**: ap-northeast-1 (Tokyo)
- **표준 프리픽스**: `jung-` (모든 리소스 네이밍 강제)

---

## 1. 아키텍처 핵심 전략 (ISMS-P 준수)

### 1.1 네트워크 격리 및 PG 전용 경로
- **Dual VPC**: `mgmt` (관제)와 `dev` (서비스) VPC를 완전 물리 격리.
- **Dual NAT (Egress 격리)**:
  - **RNAT (Regional NAT)**: 일반 Pod 및 시스템 통신용 (IP 유동적).
  - **PG-NAT (Dedicated NAT)**: 결제 관련 트래픽 전용. **고정 EIP** 할당 후 PG사 화이트리스트 등록.
- **4-Tier Subnet**: Public, App(Private), Cache(Private), Data(Private)로 계층화하여 데이터 자원 노출 차단.

### 1.2 데이터 레이어 구축 순서 (Data Integrity)
1. **Valkey (Cache)**: 세션 및 성능 최적화용. App 서브넷에서만 접근 가능한 SG 적용.
2. **RDS (Database)**: PostgreSQL 16+. 모든 데이터의 근간. Storage Encryption(KMS) 필수 적용.

---

## 2. 상세 리소스 명세

### 2.1 인프라 (Terraform)
| 구분 | 리소스 이름 예시 | 주요 설정 |
| :--- | :--- | :--- |
| **VPC** | `jung-kyeol-dev-vpc` | 10.10.0.0/16, ap-northeast-1a/c |
| **EKS** | `jung-kyeol-dev-eks` | v1.29, IRSA(OIDC) 활성화 |
| **Valkey** | `jung-kyeol-dev-valkey` | cache.t3.micro, Port 6379, Private 전용 |
| **RDS** | `jung-kyeol-dev-rds` | db.t3.small, Port 5432, KMS 암호화 |
| **WAF** | `jung-kyeol-waf` | SQLi/XSS 방어 규칙 (us-east-1 생성) |
| **CF** | `jung-kyeol-cf` | OAC 적용, S3 직접 접근 차단 |

### 2.2 보안 및 규제 준수 (ISMS-P)
- **인증**: GitHub Actions와 AWS 간 **OIDC** 신뢰 관계 구축 (Access Key 금지).
- **로깅**: CloudTrail 및 VPC Flow Logs 활성화 (`mgmt` VPC로 통합 수집).
- **접근 통제**: 모든 SG에서 `0.0.0.0/0` Ingress 금지 (ALB 제외).

---

## 3. 리포지토리별 작업 정의 (Task List)

### 📂 kyeol-infra-terraform
- [ ] `envs/dev/main.tf` 내 리전(`ap-northeast-1`) 및 프리픽스(`jung-`) 수정.
- [ ] `vpc` 모듈 내 PG 전용 NAT 및 라우팅 로직 추가.
- [ ] `valkey` 모듈 호출 및 `rds` 모듈 연동 (Valkey -> RDS 순서).
- [ ] WAF v2 및 CloudFront 배포 정의.

### 📂 kyeol-platform-gitops
- [ ] `mgmt` 클러스터에 ArgoCD (Helm) 설치 자동화.
- [ ] `dev` 클러스터에 AWS Load Balancer Controller 및 External-DNS 배포.
- [ ] 모든 Add-on의 IRSA(IAM Role) 연결 확인.

### 📂 kyeol-app-gitops
- [ ] Saleor API, Dashboard, Storefront 배포 매니페스트 분리.
- [ ] 결제 관련 Pod가 PG 전용 서브넷 노드에 배치되도록 `nodeSelector` 적용.
- [ ] Valkey/RDS 엔드포인트를 ConfigMap/Secret으로 주입.

---

## 4. 실무 운영 가이드

### 4.1 팀원 협업 (충돌 방지)
- 모든 팀원은 각자의 `owner_prefix`를 사용하여 테라폼을 실행한다. (예: `jung-`, `kim-`, `lee-`)
- S3 Backend 버킷 이름: `${owner_prefix}-kyeol-tfstate`

### 4.2 장애 대응 (Rollback)
- **인프라**: `terraform destroy` 대신 `git revert` 후 `apply` 원칙.
- **앱**: ArgoCD History를 통한 UI 기반 즉시 Rollback.

---

## 5. 테크리드 최종 확인 사항
- [ ] 모든 버킷 이름에 `jung-`이 붙어 있는가?
- [ ] RDS/Valkey가 Public Subnet에 있지 않은가?
- [ ] PG 전용 NAT의 EIP가 고정되어 있는가?
- [ ] WAF가 CloudFront 앞단에서 정상 작동하는가?

