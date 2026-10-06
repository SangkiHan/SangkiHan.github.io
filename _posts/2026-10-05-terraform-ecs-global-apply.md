---
layout: post
title: "Terraform 실행 준비 — AWS 로그인, state 버킷, global 적용"
date: 2026-10-05 12:00:00 +0900
categories: [DevOps, Terraform 구축]
tags: [Terraform, AWS, S3, Backend, Troubleshooting]
---

[지난 포스트](/posts/terraform-ecs-structure)에서 코드 구조를 잡았다. 이번엔 실제로 AWS에 연결하고 `global` 스택을 적용한다. 코드가 이미 있어도 **실행 환경을 맞추는 단계**에서 의외로 시간이 많이 걸렸다.

---

## 전체 순서

1. 검증: `fmt`, `validate` (AWS 불필요)
2. AWS CLI 로그인
3. state를 저장할 S3 버킷 만들기
4. `terraform init`, `plan`, `apply` (global)

---

## 1단계: AWS 없이 먼저 검증

`apply` 전에, AWS 자격증명 없이도 코드를 검증할 수 있다. 진입점인 `global`과 `envs/dev`에서 실행한다.

```bash
terraform fmt -recursive -check      # 포맷 검사
terraform init -backend=false        # backend 없이 provider만 받는다
terraform validate                   # 문법·참조 검증
```

`envs/dev`가 모듈 8개를 전부 호출하기 때문에, `envs/dev`에서 `validate`가 통과하면 모듈 내부까지 검증된 것이다. 모듈 폴더를 따로 돌릴 필요는 없다.

다만 `validate`가 잡는 건 문법, 타입, 참조 이름뿐이다. IAM 정책 내용이나 AWS가 값을 받아들이는지는 `plan`과 `apply`에서야 드러난다. 이 한계는 5편에서 직접 겪는다.

---

## 2단계: AWS CLI 로그인

```bash
aws login --profile terraform-study
```

브라우저가 열리고 콘솔 로그인 정보로 **임시 자격증명**이 발급된다. 액세스 키를 만들 필요가 없어서 가장 간단했다. 처음 실행하면 리전을 물어보는데, 서울인 `ap-northeast-2`를 입력했다.

로그인 후 이렇게 확인한다.

```bash
aws sts get-caller-identity --profile terraform-study
```

이 결과에서 눈여겨볼 점이 있었다. `Arn`이 `...:root`로 나왔다. **루트 계정으로 로그인한 것**이다.

> 루트 계정은 모든 권한을 가져서, 유출되면 계정 전체가 위험하다. 연습용 개인 계정이라 진행했지만, 계속 쓸 계정이라면 `AdministratorAccess` 권한의 IAM 사용자나 SSO를 만들어 쓰는 게 맞다.
{: .prompt-warning }

### 프로파일은 꼭 지정해야 하나?

아니다. 코드의 `provider`에는 `profile = var.aws_profile`이 있고 기본값이 `null`이다. 프로파일을 안 주면 Terraform이 AWS 기본 자격증명 체인(`default` 프로파일, 환경변수 등)을 쓴다. 계정이 여러 개일 때만 프로파일로 구분하면 된다.

---

## 3단계: state 버킷 만들기

Terraform은 만든 리소스를 state 파일에 기록한다. 혼자 쓰면 로컬 파일로도 되지만, 이 파일을 잃으면 Terraform이 자기가 만든 걸 모르게 된다. 그래서 **S3에 저장**한다. S3 버킷은 Terraform으로 만들 수 없다(닭과 달걀 문제). state를 저장할 곳이 먼저 있어야 하기 때문이다. 이것만 CLI로 직접 만든다.

```bash
aws s3api create-bucket --bucket terraform-study-tfstate-<ACCOUNT_ID>-ap-northeast-2 \
  --profile terraform-study --region ap-northeast-2 \
  --create-bucket-configuration LocationConstraint=ap-northeast-2

# 버전 관리 (state가 망가졌을 때 되돌리기 위해)
aws s3api put-bucket-versioning --bucket ... --versioning-configuration Status=Enabled

# 암호화, 퍼블릭 접근 차단도 함께 설정
```

### 에러 1: zsh에서 변수가 쪼개지지 않는다

처음엔 옵션을 변수에 담아 쓰려고 했다.

```bash
P="--profile terraform-study --region ap-northeast-2"
aws s3api create-bucket --bucket $B $P ...
```

```
aws: [ERROR]: Unknown options: --profile terraform-study --region ap-northeast-2
```

**zsh는 변수를 공백 기준으로 쪼개지 않는다.** bash였다면 `--profile`과 `terraform-study`가 따로 전달됐겠지만, zsh는 `$P` 전체를 하나의 인자로 넘긴다. 해결은 변수를 쓰지 않고 옵션을 직접 쓰는 것이다.

---

## 계정 ID를 코드에 넣지 않는 방법

버킷 이름에 계정 ID가 들어가는데, 이 코드는 공개 레포에 올라간다. 계정 ID를 커밋 이력에 남기고 싶지 않았다. 그래서 `backend.tf`의 `bucket` 줄을 빼고, `init`할 때 파일로 주입했다 (partial configuration).

```hcl
# backend.tf
terraform {
  backend "s3" {
    # bucket 은 계정 ID 가 들어가서 코드에 넣지 않는다.
    key          = "global/terraform.tfstate"
    region       = "ap-northeast-2"
    encrypt      = true
    use_lockfile = true
  }
}
```

```hcl
# backend.hcl  (.gitignore 대상, 로컬에만 있다)
bucket  = "terraform-study-tfstate-<ACCOUNT_ID>-ap-northeast-2"
profile = "terraform-study"
```

```bash
terraform init -backend-config=backend.hcl
```

레포에는 값이 비어 있는 `backend.hcl.example`만 커밋한다.

### 에러 2: backend가 자격증명을 못 찾는다

`init`을 하니 이런 에러가 났다.

```
Error: No valid credential sources found
Error: failed to refresh cached credentials, no EC2 IMDS role found
```

`provider`에 프로파일을 지정해도, **backend는 provider와 별개로 자격증명을 찾는다.** 프로파일을 못 찾은 backend는 EC2 메타데이터까지 뒤지다 실패한 것이다. `backend.hcl`에 `profile`을 추가해서 해결했다. 같은 이유로 5편에서 `terraform_remote_state`에도 프로파일을 넘겨야 했다.

---

## 4단계: global 적용

```bash
cd infra/terraform/global
terraform init -backend-config=backend.hcl
terraform plan
```

`plan` 결과는 이랬다.

```
Plan: 13 to add, 0 to change, 0 to destroy.
```

- ECR repository 6개 (서비스당 1개)
- ECR lifecycle 정책 6개 (오래된 이미지 자동 삭제)
- GitHub OIDC provider 1개

`plan`은 AWS에 아무것도 만들지 않고 "이렇게 만들 예정"만 보여준다. 숫자가 예상과 같은지 확인하고 `apply`를 실행했다.

```bash
terraform apply
```

```
Apply complete! Resources: 13 added, 0 changed, 0 destroyed.
```

여기서 한 가지 실수가 있었다. 확인 프롬프트에 한글 입력 상태로 `ㅛyes`를 쳤다. Terraform은 정확히 `yes`만 받으니 영문 입력인지 확인해야 한다.

---

## apply가 실제로 하는 일

헷갈리기 쉬워서 정리하고 넘어간다.

| 명령 | 성격 | AWS 변경 |
|---|---|---|
| `validate` | 문법 검증 | 없음 |
| `plan` | 미리보기 | 없음 |
| `apply` | **실제 생성** | 있음, 비용 발생 |

`apply`는 검증이 아니라 **실제로 만드는** 단계다. state와 실제 AWS를 비교해 차이를 계산하고, 확인을 받은 뒤 의존 순서대로 AWS API를 호출한다. 만든 리소스의 ID는 S3 state에 기록된다.

---

## 다음 글

`global`이 끝났다. 다음 글에서는 `envs/dev`를 적용하고, 컨테이너 이미지를 올려서 실제로 서비스를 띄운다. 그 과정에서 `validate`와 `plan`으로는 안 잡히던 에러를 여러 개 만났다.

➡️ 시리즈 인덱스: [Terraform으로 AWS ECS 서비스 배포하기](/posts/terraform-ecs-intro)
