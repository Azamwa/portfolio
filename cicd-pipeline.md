# CI/CD 파이프라인 재구성: GitHub + GCP Cloud Build → GitLab Self-hosted Runner + Blue-Green 배포

## 한줄 요약

GitHub + GCP Cloud Build → GitLab CI/CD Self-hosted Runner 전환, Blue-Green 무중단 배포 파이프라인 재구성

---

## 배경

사내 코드 저장소를 GitHub에서 GitLab(self-hosted)으로 이관하면서, 기존 GCP Cloud Build 기반 CI/CD도 함께 전환해야 하는 상황이었습니다. 기존 Cloud Build에서 운영하던 Blue-Green 배포 전략을 GitLab Runner 환경에 맞게 재구성하고, 배포 파이프라인 전체를 .gitlab-ci.yml 단일 파일로 통합하는 것을 목표로 했습니다.

| 구분        | Before                   | After                                                  |
| ----------- | ------------------------ | ------------------------------------------------------ |
| 코드 저장소 | GitHub                   | GitLab (self-hosted)                                   |
| CI/CD       | GCP Cloud Build (관리형) | GitLab Runner (self-hosted)                            |
| 배포 방식   | Blue-Green (Cloud Build) | **Blue-Green (GitLab Runner, 단일 YAML 관리)**         |
| 인프라      | GCP                      | GCP (Compute Engine, Artifact Registry, Load Balancer) |

---

## 아키텍처

```
Developer → GitLab Push
               ↓
         GitLab Runner (Self-hosted, Docker-in-Docker)
               ↓
   ┌───────────────────────────┐
   │  7-Step Blue-Green Deploy │
   │                           │
   │  1. Secret Manager 환경변수 로드
   │  2. 현재 Active 감지 (Blue/Green)
   │  3. Standby 인스턴스 기동
   │  4. Docker 이미지 빌드 & Push
   │  5. Standby 컨테이너 교체
   │  6. Load Balancer 트래픽 전환
   │  7. 이전 인스턴스 정리
   └───────────────────────────┘
               ↓
         GCP Compute Engine
    ┌──────────┬──────────┐
    │  Blue    │  Green   │
    │ (Active) │(Standby) │
    └──────────┴──────────┘
               ↑
         GCP Load Balancer ← 사용자 트래픽
```

---

## 주요 구현 내용

### 1. GitLab Runner + Docker-in-Docker 환경 구성

- 사내 서버에 GitLab Runner를 self-hosted로 설치하여 CI/CD 실행 환경 직접 운영
- Runner 자체가 Docker 컨테이너로 동작하므로, 내부에서 Docker 빌드를 수행하기 위해 **Docker-in-Docker(DinD)** 구조 적용
- `docker:24.0.9-dind` 서비스를 사이드카로 띄워 빌드 컨텍스트 분리

### 2. Blue-Green 무중단 배포 파이프라인

7단계 배포 흐름을 `.gitlab-ci.yml` 단일 파일로 구성했습니다.

**핵심 로직:**

```bash
# 현재 Active 서버 감지 → Standby 결정
CURRENT=$(gcloud compute ssh ... --command="cat /etc/nginx/conf.d/upstream.conf")
# blue가 active면 → green에 배포, 반대면 → blue에 배포
```

```bash
# 이미지 태그: {env}-{color}-{commit SHA}
# 예: dev-green-6632c85f
docker buildx build --platform=linux/amd64 \
  --build-arg NEXT_PUBLIC_VERSION=$CI_COMMIT_SHORT_SHA \
  -t "$FULL_IMAGE_NAME" --push .
```

```bash
# Standby 서버에 새 컨테이너 배포 후 Load Balancer 트래픽 전환
gcloud compute backend-services update ... --backends=NEW_INSTANCE_GROUP
```

**배포 후 정리:**

- 이전 Active 인스턴스 자동 종료 (비용 절감)
- 미사용 Docker 이미지 정리

### 3. 브랜치별 환경 분리

```yaml
stages:
  - deploy-dev # develop 브랜치 → dev 환경
  - deploy-prod # main 브랜치 → prod 환경
```

- YAML 앵커(`&deploy_logic` / `*deploy_logic`)를 활용해 dev/prod 공통 배포 로직을 중복 없이 관리
- 환경별 변수(인스턴스명, Secret 경로 등)만 override하는 구조

### 4. 보안 및 시크릿 관리

- GCP Service Account Key는 GitLab CI/CD Variables에 base64 인코딩 + Masked 처리
- OAuth 키, Redis 접속 정보 등 민감한 환경변수는 **GCP Secret Manager**에서 배포 시점에 동적 로드
- `NEXT_PUBLIC_*` 환경변수는 빌드타임에 주입이 필요하므로 Docker build arg로 별도 처리

### 5. 배포 버전 추적 체계

- **이미지 태그:** `dev-green-6632c85f` (환경-컬러-커밋SHA)
- **앱 푸터:** `v 6632c85f` 표시
- 배포된 버전과 GitLab 커밋을 1:1로 추적 가능

---

## 사용 기술

| 영역      | 기술                                                                 |
| --------- | -------------------------------------------------------------------- |
| CI/CD     | GitLab CI/CD, GitLab Runner (self-hosted)                            |
| 컨테이너  | Docker, Docker-in-Docker, Docker Buildx                              |
| 클라우드  | GCP Compute Engine, Artifact Registry, Load Balancer, Secret Manager |
| 배포 전략 | Blue-Green Deployment                                                |
| 앱        | Next.js (Docker 멀티스테이지 빌드)                                   |

---

## 성과

- **무중단 배포 달성**: Blue-Green 전략으로 배포 중 서비스 중단 제로
- **즉시 롤백 가능**: 문제 발생 시 이전 버전 인스턴스로 트래픽 즉시 전환
- **비용 최적화**: Standby 인스턴스를 배포 시에만 기동하고, 이후 자동 종료
- **단계적 전환**: dev 환경 먼저 검증 → 안정화 확인 후 prod 전환하는 점진적 이관 전략 수행
