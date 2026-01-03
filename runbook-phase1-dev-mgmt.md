# 🚀 jung-kyeol Phase-1 런북 (DEV + MGMT)

이 문서는 **Tokyo 리전(ap-northeast-1)** 및 **jung-** 프리픽스를 사용하는 신규 아키텍처(V2)를 기준으로 작성되었습니다.

---

## 0. 사전 준비

### 0.1. 필수 도구

| 도구 | 최소 버전 | 설치 확인 |
|------|----------|----------|
| AWS CLI | 2.x | `aws --version` |
| Terraform | 1.5.0+ | `terraform --version` |
| kubectl | 1.28+ | `kubectl version --client` |
| kustomize | 5.0+ | `kustomize version` |
| helm | 3.12+ | `helm version` |
| git | 2.x | `git --version` |
| docker | 24.x+ | `docker --version` |

### 0.2. 필수 입력값 (Placeholders)

| Placeholder | 설명 | 예시 |
|-------------|------|------|
| `ACCOUNT_ID` | AWS 계정 ID | `827913617839` |
| `HOSTED_ZONE_ID` | Route53 Hosted Zone ID | `Z01193062JD31QR7P4APO` |
| `ACM_ARN` | ALB용 ACM 인증서 ARN (Tokyo) | `arn:aws:acm:ap-northeast-1:827913617839:certificate/f7d794c7-e3c3-4cd2-9ef1-bd4aa7d34834` |
| `REPO_OWNER` | GitHub 사용자/조직 | `Jungbin7` |
| `AWS_REGION` | AWS 리전 | `ap-northeast-1` |

---

## 1. GitHub 레포지토리 초기화 및 푸시

> ⚠️ 모든 코드의 `Owner` 태그와 프리픽스가 `jung`으로 수정되었는지 확인하십시오.

```bash
# 1. kyeol-infra-terraform
cd kyeol-infra-terraform
git init
git remote add origin https://github.com/Jungbin7/kyeol-infra-terraform.git
git add . && git commit -m "Initial commit: jung-kyeol V2" && git branch -M main && git push -u origin main

# 2. kyeol-platform-gitops
cd kyeol-platform-gitops
git init
git remote add origin https://github.com/Jungbin7/kyeol-platform-gitops.git
git add . && git commit -m "Initial commit: Platform Addons" && git branch -M main && git push -u origin main

# 3. kyeol-app-gitops
cd kyeol-app-gitops
git init
git remote add origin https://github.com/Jungbin7/kyeol-app-gitops.git
git add . && git commit -m "Initial commit: Saleor Apps" && git branch -M main && git push -u origin main
```

---

## 2. Bootstrap: 인프라 기반 조성

```bash
cd kyeol-infra-terraform/envs/bootstrap
terraform init
terraform apply -auto-approve
```

### 검증
- S3 버킷 확인: `jung-kyeol-tfstate-827913617839-ap-northeast-1`
- DynamoDB 확인: `jung-kyeol-tfstate-lock`

---

## 3. MGMT 환경 배포 (관제 클러스터)

```bash
cd kyeol-infra-terraform/envs/mgmt
terraform init
terraform apply -auto-approve
```

---

## 4. DEV 환경 배포 (서비스 클러스터)

```bash
cd kyeol-infra-terraform/envs/dev
terraform init
terraform apply -auto-approve
```

### ✅ 핵심 리소스 검증
1. **ALB LBC**: `jung-kyeol-dev-alb-controller-role` 확인
2. **Valkey**: `jung-kyeol-dev-cache` 복제 그룹 상태 확인
3. **RDS**: `jung-kyeol-dev-rds` 인스턴스 확인

---

## 5. kubeconfig 설정 및 ArgoCD 설치

```bash
# 컨텍스트 업데이트
aws eks update-kubeconfig --region ap-northeast-1 --name jung-kyeol-mgmt-eks
aws eks update-kubeconfig --region ap-northeast-1 --name jung-kyeol-dev-eks

# MGMT에 ArgoCD 설치
kubectl config use-context arn:aws:eks:ap-northeast-1:827913617839:cluster/jung-kyeol-mgmt-eks
cd kyeol-platform-gitops
kubectl apply -k argocd/bootstrap/
```

---

## 6. DEV 클러스터 Addons 설치 (ArgoCD 연동)

```bash
# DEV 클러스터를 MGMT ArgoCD에 등록
# (MGMT 컨텍스트에서 실행)
argocd cluster add arn:aws:eks:ap-northeast-1:827913617839:cluster/jung-kyeol-dev-eks --name dev-cluster

# Root App 배포
kubectl apply -f argocd/app-of-apps/root-app.yaml
```

---

## 7. App 배포 및 검증

### 7.1. ECR 이미지 푸시 (GitHub Actions)
`kyeol-storefront` 레포의 `.github/workflows/build-push-ecr.yml`이 실행되도록 푸시합니다.

### 7.2. Saleor 앱 배포
```bash
# MGMT 컨텍스트에서 실행 (ArgoCD Application 등록)
kubectl apply -f kyeol-app-gitops/argocd/applications/saleor-dev.yaml
```

### 7.3. 최종 확인
- **Ingress**: `kubectl get ingress -n kyeol` (dev 컨텍스트)
- **DNS**: `https://dev.mgz-g2-u3.shop` 접속 확인
- **CloudFront**: `https://kyeol.mgz-g2-u3.shop` (Phase-2)
