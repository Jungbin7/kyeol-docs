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
| `HOSTED_ZONE_ID` | Route53 Hosted Zone ID | `Z0XXXXXXXXXXXX` |
| `ACM_ARN` | ALB용 ACM 인증서 ARN (Tokyo) | `arn:aws:acm:ap-northeast-1:...` |
| `REPO_OWNER` | GitHub 사용자/조직 | `jungbin7` |
| `AWS_REGION` | AWS 리전 | `ap-northeast-1` |

---

## 1. GitHub 레포지토리 초기화 및 푸시

> ⚠️ 모든 코드의 `Owner` 태그와 프리픽스가 `jung`으로 수정되었는지 확인하십시오.

```bash
# 1. kyeol-infra-terraform
cd kyeol-infra-terraform
git init
git remote add origin https://github.com/jungbin7/kyeol-infra-terraform.git
git add . && git commit -m "Initial commit: jung-kyeol V2" && git branch -M main && git push -u origin main

# 2. kyeol-platform-gitops
cd kyeol-platform-gitops
git init
git remote add origin https://github.com/jungbin7/kyeol-platform-gitops.git
git add . && git commit -m "Initial commit: Platform Addons" && git branch -M main && git push -u origin main

# 3. kyeol-app-gitops
cd kyeol-app-gitops
git init
git remote add origin https://github.com/jungbin7/kyeol-app-gitops.git
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
- S3 버킷 확인: `jung-kyeol-tfstate`
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
1. **PG 전용 NAT**: `jung-kyeol-dev-pg-nat-eip` 확인 (이 IP를 PG사에 전달)
2. **Valkey**: `jung-kyeol-dev-valkey` 클러스터 상태 확인
3. **RDS**: `jung-kyeol-dev-rds` 암호화(KMS) 적용 여부 확인

---

## 5. kubeconfig 설정 및 ArgoCD 설치

```bash
# 컨텍스트 업데이트
aws eks update-kubeconfig --region ap-northeast-1 --name jung-kyeol-mgmt-eks --alias mgmt
aws eks update-kubeconfig --region ap-northeast-1 --name jung-kyeol-dev-eks --alias dev

# MGMT에 ArgoCD 설치
kubectl config use-context mgmt
cd kyeol-platform-gitops
kubectl apply -k argocd/bootstrap/
```

---

## 6. DEV 클러스터 Addons 설치 (ArgoCD 연동)

> ⚠️ 수동 헬름 설치 대신 ArgoCD를 통해 관리합니다.

```bash
# DEV 클러스터를 MGMT ArgoCD에 등록
argocd cluster add dev --name jung-kyeol-dev

# Addons 배포 (ALB Controller, ExternalDNS)
# kyeol-platform-gitops/clusters/dev/root-app.yaml 배포 (가정)
```

---

## 7. App 배포 및 검증

### 7.1. ECR 이미지 푸시 (GitHub Actions)
`kyeol-storefront` 레포의 `.github/workflows/build-push-ecr.yml`이 실행되도록 푸시합니다.

### 7.2. Saleor 앱 배포
```bash
kubectl config use-context dev
kubectl apply -k kyeol-app-gitops/apps/saleor/overlays/dev/
```

### 7.3. 최종 확인
- **Ingress**: `kubectl get ingress -n kyeol`
- **DNS**: `origin-dev-kyeol.msp-g1.click` 접속 확인
- **CloudFront**: 생성된 CF 도메인으로 접속 및 WAF 작동 확인

---

## 11. 트러블슈팅

### PG NAT 고정 IP 확인
결제 Pod 내에서 `curl ifconfig.me` 실행 시, AWS 콘솔의 `pg-nat-eip` 주소와 일치해야 합니다. 일치하지 않으면 라우팅 테이블 및 Subnet 배치를 확인하십시오.
