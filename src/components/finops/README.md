# FinOps Analyzer

KRR CLI와 FastAPI를 함께 실행하는 분석 서비스입니다. Kubernetes의 리소스 사용 이력을
Thanos/Prometheus에서 읽어 권장값을 계산하고, 정책 평가 결과를 DB와 Telegram gateway에 전달합니다.

## 원본과 이식 범위

- 원본: https://github.com/After-mak/practical-project/tree/c4e0d8ea441daf55d13e11c74dee6eb67725de5f/finops
- 원본의 FastAPI 앱, KRR Dockerfile, 의존성, KRR 결과 파서 테스트를 가져왔습니다.
- Dockerfile의 COPY 경로를 이 폴더를 기준으로 바꿨습니다.
- 기본 Thanos와 Telegram gateway 주소를 현재 클러스터의 monitoring namespace에 맞췄습니다.
- Helm chart는 기존 MGPZZ/GitOps의 charts/finops를 사용합니다.

## 이미지 빌드와 배포

App 저장소 루트에서:

```bash
docker build -t krr-analyzer:local src/components/finops
```

github_action 브랜치에 src 또는 workflow 변경을 push하면 .github/workflows/ecr-images.yml이
이 폴더를 빌드해 Actions의 AWS 계정에 있는 krr-analyzer ECR 저장소로 push합니다.
KRR 이미지 빌드 전에 Python 3.11에서 tests를 실행하며, 실패하면 해당 이미지 빌드를 중단합니다.
AWS Secrets는 기존 MINKI_AWS_ACCESS_KEY_ID와 MINKI_AWS_SECRET_ACCESS_KEY를 사용합니다.
태그는 latest, sha-<전체 커밋 SHA>, 그리고 현재 배포에 맞춘 v0.1.36입니다.

향후 버전을 올릴 때는 workflow의 KRR matrix.version과 GitOps의 charts/finops/values.yaml에
있는 image.tag를 함께 변경하세요. GitOps 배포 브랜치는 test_alb입니다.
이 workflow는 GitOps를 자동으로 수정하지 않습니다. 같은 태그에 새 이미지를 push해도
기존 Running Pod는 자동으로 교체되지 않습니다.

## 실행과 검증

이 폴더에서 Python 3.11 가상환경을 만든 뒤:

```bash
python -m pip install -r requirements-dev.txt
python -m pytest tests -q
```

외부 서비스 없이 API를 확인하는 로컬 실행 예시:

```bash
MOCK_INTEGRATION=true AUTO_SEND_TELEGRAM=false uvicorn app.main:app --host 127.0.0.1 --port 8000
```

실제 모드의 KRR Linux 실행파일은 Docker 이미지 빌드 시 설치됩니다. Kubernetes에서는
기존 Helm chart가 ServiceAccount/RBAC, DB Secret, Telegram Secret, 환경변수를 연결합니다.

- GET /health/live: API 프로세스 상태
- GET /health/ready: KRR 실행파일 및 필요한 Telegram 설정 확인
- POST /analyze: 단일 워크로드 분석
- POST /analyze/namespace: namespace 전체 분석
- GET /recommendation/{namespace}/{deployment_name}: 최근 권장값 조회

실제 분석에는 정상적인 Thanos/Prometheus 지표가 필요합니다. 현재 클러스터에서 확인한
Thanos S3 버킷 문제는 이미지 빌드와 별도로 복구해야 합니다.
