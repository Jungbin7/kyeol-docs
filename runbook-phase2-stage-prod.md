# KYEOL Phase-2 런북 (STAGE + PROD + Dashboard)

Phase-2에서 STAGE, PROD 환경을 활성화하고 Saleor Dashboard를 배포하기 위한 실행 순서입니다.

---

## 0. Phase-2 개요

### Phase-1과의 차이

| 항목 | Phase-1 | Phase-2 |
|------|---------|---------|
| 환경 | DEV, MGMT | DEV, STAGE, PROD |
| Storefront | DEV만 | DEV, STAGE, PROD |
| Dashboard | 없음 | DEV, STAGE, PROD |
| RDS | Single-AZ | Multi-AZ (STAGE/PROD) |
| EKS Nodes | 최소 | 단계적 증가 |
| 캐시 | 비활성 (DEV) | 활성 (STAGE/PROD) |

---

## 1. 사전 준비

### 1.1. Phase-1 완료 확인

```bash
# DEV 환경 확인
aws eks describe-cluster --name jung-kyeol-dev-eks --region ap-northeast-1 --query "cluster.status"

# MGMT ArgoCD 확인
kubectl get pods -n argocd
```

### 1.2. ACM 인증서 발급 (STAGE/PROD용)

> ⚠️ 이미 `*.mgz-g2-u3.shop` 와일드카드 인증서가 발급되어 있으므로 이를 재사용합니다.

---

## 2. STAGE/PROD 환경 배포

### 2.1. Terraform 설정

```bash
# STAGE 배포
cd kyeol-infra-terraform/envs/stage
terraform init
terraform apply -auto-approve

# PROD 배포
cd kyeol-infra-terraform/envs/prod
terraform init
terraform apply -auto-approve
```

---

## 3. ArgoCD를 통한 앱 배포

### 3.1. Storefront/Dashboard 동기화

```bash
# MGMT 컨텍스트에서 실행
kubectl apply -f kyeol-app-gitops/argocd/applications/saleor-stage.yaml
kubectl apply -f kyeol-app-gitops/argocd/applications/saleor-prod.yaml
kubectl apply -f kyeol-app-gitops/argocd/applications/dashboard-dev.yaml
```

---

## 4. 최종 검증 주소

| 환경 | Storefront URL | Dashboard URL |
|------|---------------|---------------|
| DEV | `https://dev.mgz-g2-u3.shop` | `https://dev-dashboard.mgz-g2-u3.shop` |
| STAGE | `https://stage.mgz-g2-u3.shop` | `https://stage-dashboard.mgz-g2-u3.shop` |
| PROD | `https://mgz-g2-u3.shop` | `https://dashboard.mgz-g2-u3.shop` |
