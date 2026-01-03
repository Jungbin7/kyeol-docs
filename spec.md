<!-- docs/spec.md -->

# KYEOL Saleor 멀티환경 클라우드 아키텍처 스펙 (Anti-Gravity Reference)
- 기준일: 2026-01-03
- 목적: VSCode Agent/MCP가 “흔들리지 않도록” 변경 불가 기준(고정 스펙)만 제공

---

## 0) 프로젝트 목표(고정)
- Saleor 기반 이커머스 멀티환경(DEV/STAGE/PROD/MGMT) 아키텍처 구현
- 환경별 동일 UI/서비스를 운영하고, 도메인으로 환경 분리
- MSP 운영 수준(IaC + GitOps + CI/CD + 보안/운영 고려)으로 재현 가능해야 함

---

## 1) 환경/도메인 정책(고정)

### 1.1 환경
- `dev`, `stage`, `prod`, `mgmt` 총 4개 환경
- Phase-1에서는 **DEV + MGMT만 적용 대상**
  - STAGE/PROD는 **코드 템플릿만 생성**, apply/runbook에서는 비활성/미사용 처리

### 1.2 도메인
- 메인 도메인: `mgz-g2-u3.shop`
- 서비스 도메인(CloudFront 앞단)
  - DEV: `dev.mgz-g2-u3.shop`
  - STAGE: `stage.mgz-g2-u3.shop`
  - PROD: `mgz-g2-u3.shop`
- API 도메인 (ALB Ingress)
  - DEV: `dev-api.mgz-g2-u3.shop`
  - STAGE: `stage-api.mgz-g2-u3.shop`
  - PROD: `api.mgz-g2-u3.shop`
- 규칙: **ALB DNS를 직접 사용하지 않고 ExternalDNS를 통해 도메인 연결**

---

## 2) 핵심 원칙(고정)

### 2.1 ALB 생성 원칙
- ALB는 Terraform으로 직접 생성하지 않는다.
- **EKS + AWS Load Balancer Controller + Ingress**로 ALB가 자동 생성되어야 한다.

### 2.2 DNS 자동화 원칙
- Route53 레코드는 **ExternalDNS**가 자동 관리한다.
- 특히 `*.mgz-g2-u3.shop` 레코드를 ExternalDNS가 자동 생성/갱신해야 한다.

### 2.3 NAT 원칙
- VPC별 NAT는 **VPC당 1개**로 구성 (DEV 기준)
- **Regional NAT Gateway + Elastic IP 수동 지정(manual mode)**
- PROD 환경에서는 결제 API 전용 NAT(PG NAT)를 추가 고려

### 2.4 CloudFront 원칙
- CloudFront 배포판은 **DEV/STAGE/PROD 총 3개**
- CloudFront용 WAFv2는 **us-east-1** 리전에 생성

---

## 3) 리전/프로바이더/인증서(고정)

### 3.1 리전
- 기본 리전: **ap-northeast-1 (Tokyo)**
- AZ 표기는 `a`, `c` 사용:
  - `ap-northeast-1a`
  - `ap-northeast-1c`

### 3.2 ACM 인증서
- CloudFront 인증서: **us-east-1** (필수)
- ALB(Ingress) 인증서: **ap-northeast-1**
- 권장: `*.mgz-g2-u3.shop` 와일드카드 인증서 사용

---

## 4) CIDR/서브넷(고정 표)

### 4.1 DEV (컴퓨트 1AZ + Public만 2AZ, cache 있음)
- VPC: `10.10.0.0/16`
- Public: 2AZ (a, c)
- App/Data/Cache: 1AZ (a)
- Subnets:
  - public-a: `10.10.0.0/24`
  - public-c: `10.10.1.0/24`
  - app-private-a: `10.10.4.0/22`
  - data-private-a: `10.10.9.0/24`
  - cache-private-a: `10.10.8.0/26`

### 4.2 MGMT (2AZ)
- VPC: `10.40.0.0/16`
- Subnets:
  - public-a: `10.40.0.0/24`
  - public-c: `10.40.1.0/24`
  - ops-private-a: `10.40.4.0/22`
  - ops-private-c: `10.40.12.0/22`

---

## 5) 네이밍 컨벤션(고정)

### 5.1 기본 규칙
- 기본 컨벤션: `jung-kyeol-[env]-[resource]-[detail]`
- 환경(env): `prod`, `stage`, `dev`, `mgmt`
- 소문자 + 하이픈(-)

### 5.2 프리픽스
- 모든 리소스에 **`jung-`** 프리픽스를 반드시 붙인다. (사용자 요청 사항)

### 5.3 예시
- VPC: `jung-kyeol-dev-vpc`
- EKS: `jung-kyeol-dev-eks`
- S3 (State): `jung-kyeol-tfstate-827913617839-ap-northeast-1`

---

## 6) EKS 필수 애드온(고정)
- AWS Load Balancer Controller (IRSA)
- ExternalDNS (IRSA)
- metrics-server

---

## 7) 산출물(고정)
- IaC: `kyeol-infra-terraform`
- Platform GitOps: `kyeol-platform-gitops`
- App GitOps: `kyeol-app-gitops`
- Frontend: `kyeol-storefront`, `kyeol-saleor-dashboard`
