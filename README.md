# AWS CDK 템플릿 카탈로그 (v2)

> **모든 템플릿은 AWS CDK v2 스택**으로 제공되며, 파라미터 5개 이하만 입력하면 배포가 완료됩니다.

---

## 1. 템플릿 레벨 & 모듈

### 레벨 01 · Quick‑Start

| 코드 ID              | 핵심 서비스 묶음                                              | 언제 사용하나              | 현실적 주의점                                                                  |
| ------------------ | ------------------------------------------------------ | -------------------- | ------------------------------------------------------------------------ |
| `static-spa`       | **Amplify Hosting** (+ S3, CloudFront, Route 53, ACM)  | 마케팅 사이트, 랜딩 페이지, 블로그 | Amplify가 빌드→배포→CDN까지 자동 처리. 단 CloudFront Functions 호출이 많아지면 소액 과금 발생 가능. |
| `srvless-api-mini` | **API Gateway HTTP API + Lambda + DynamoDB + Cognito** | 프로토타입·해커톤용 API       | 메모리 256 MB, 타임아웃 10 초로 시작하여 과금 폭주 방지.                                    |
| `fargate-web-solo` | **ECS Fargate (1 Service)** + ALB + ECR                | 단일 컨테이너 웹앱           | 0.25 vCPU / 0.5 GB 최소 사양 → 두 달간 < \$30 예상.                               |

### 레벨 02 · Growth‑Ops

| 코드 ID           | 핵심 서비스 묶음                                                                         | 언제 사용하나         | 현실적 주의점                            |
| --------------- | --------------------------------------------------------------------------------- | --------------- | ---------------------------------- |
| `ecs-micro-kit` | VPC(3‑tier) + ALB + **ECS Fargate(다중 Task)** + Aurora Serverless v2 + ElastiCache | 월간 DAU ≥ 10k    | ECS는 EKS보다 러닝커브 낮고 제어판 요금 없음.      |
| `event-fanout`  | S3 → EventBridge → Lambda fan‑out → Step Functions                                | 이미지 / CSV 일괄 변환 | Lambda 동시성(예: 50) 제한으로 첫 달 예산 고정.  |
| `ci-cd-flow`    | CodeCommit(또는 GitHub) → CodePipeline → CodeBuild → CDK deploy                     | 팀 CI/CD 파이프라인   | PR당 약 \$0.004 → 10명 팀도 월 한 자릿수 \$. |

### 레벨 03 · Data & AI

| 코드 ID                 | 핵심 서비스 묶음                                                                    | 언제 사용하나      | 현실적 주의점                                                  |
| --------------------- | ---------------------------------------------------------------------------- | ------------ | -------------------------------------------------------- |
| `lake-core`           | Lake Formation + S3(raw/curated) + Glue + Athena + EventBridge Scheduler     | 로그·BI 분석     | Glue DPU 1 → 2 배 증설 전, Athena CTAS로 충분한지 테스트.            |
| `stream-realtime`     | **Kinesis Data Streams/Firehose → Managed Flink → S3 / Redshift Serverless** | IoT·실시간 대시보드 | 초당 1 MB 미만이면 Kinesis On‑Demand가 더 저렴.                    |
| `bedrock-genai-proto` | API Gateway + Lambda + **Bedrock (Claude/Sonnet)** + DynamoDB                | 챗봇·요약기 PoC   | Bedrock 호출비가 가장 큼 → `max_tokens`, `temperature` 파라미터 노출. |

### 레벨 04 · Advanced

| 코드 ID             | 핵심 서비스 묶음                                            | 언제 사용하나        | 현실적 주의점                                        |
| ----------------- | ---------------------------------------------------- | -------------- | ---------------------------------------------- |
| `eks-foundation`  | **EKS** (managed node 그룹 + Karpenter) + App Mesh     | 다중 팀 / 멀티 클러스터 | EKS 제어판 \$0.10/시간 + 운영 복잡도. 쿠버네티스 경험 있을 때만 추천. |
| `mlops-sagemaker` | **SageMaker Pipelines + Model Registry + CodeBuild** | 주기적 모델 재학습     | Spot Training의 ‘Save %’ 옵션 기본값 ON.             |

---

## 2. 데이터 기반 근거

* **Docker × Datadog 2023 리포트**

  * AWS 고객 70% 이상이 Lambda·Fargate 등 **서버리스** 사용.
  * 컨테이너 사용자 46%가 **Fargate/App Runner** 같은 *서버리스 컨테이너*로 이동.
* **⇒ ECS Fargate + Lambda** 조합은 업계에서 가장 널리 쓰이는 현실적 선택.


---

## 4. 경쟁 비교

|             | CloudSeed(우리)        | AWS Launch Wizard / Proton | Terraform Cloud | Pulumi Cloud     |
| ----------- | -------------------- | -------------------------- | --------------- | ---------------- |
| **온보딩 난이도** | 웹 UI에서 5개 필드 입력 후 배포 | CFN YAML 이해 필요             | HCL 학습 필요       | TypeScript/Go 필요 |
| **템플릿 범위**  | 스타트업 빈번 워크로드 10종     | 컨테이너·서버리스(로드맵)             | 범위 넓지만 빈칸 많음    | 동일               |
| **비용 가시성**  | 실시간 사용량 + 예산 경보      | 별도 Cost Explorer           | 팀 요금 +\$20\~    | 팀 요금 +\$50\~     |
| **업그레이드**   | 구독 플랜만 바꾸면 템플릿 해제    | 서비스별 기능 플래그                | 모놀리식 코드 리팩터     | 동일               |
| **한국어 지원**  | ✅ (문서·Slack KST)     | ❌                          | ❌               | ❌                |

> **결론** : 초기 인력이 적은 스타트업·프리랜서에게는 \*\*“낮은 진입장벽 UI” + “자동 비용 가드레일”\*\*이 실질적 차별점.

---

## 5. 서비스 개요 (예시 이름: CloudSeed)

1. **3단계 온보딩**

   1. AWS IAM Role 연결 (CFN 스택 1개로 Read‑only + Deploy 권한)
   2. 템플릿 선택 & 파라미터 입력 (도메인, Git URL, 예상 트래픽 등)
   3. *Deploy* 클릭 → 평균 7분 내 완료 (CDK Asset S3 캐시)
2. **통합 대시보드** — 인프라 토폴로지·알람·예산·배포 이력 한눈에.
3. **CI/CD 연동** — GitHub App / GitLab CI 토큰 연결 시 PR 단위 *Preview Stack* 자동.
4. **요금제**

   * **Free** : 템플릿 2개, 월 1스택, 모니터링 7일 보존.
   * **Team (\$29)** : Growth 템플릿 전체, 5스택, Slack 알림.
   * **Biz (\$79)** : Data/AI 템플릿, 역할 기반 접근제어, 14일 비용 예측.
   * **Enterprise (견적)** : Advanced 레벨, SSO/OIDC, 전담 SA.
5. **학습 자료** — 5분 튜토리얼, 템플릿별 예상 비용 계산기.
6. **로드맵** — 분기별 AWS 신규 서비스(Zero‑ETL, App Runner LLM Runtime) 템플릿 추가, 구독자에게 버튼 업그레이드 제공.

---

## 6. 요약

* **TOP 10 템플릿**에 *초간단 배포* 집중 → 양보다 질.
* **ECS Fargate + Lambda** 대세 → “쿠버네티스 필수” 편견 해소.
* **CloudSeed**의 핵심 = *One‑click 배포* + *비용 가드레일* — AI·스타트업이 원하는 지점.
