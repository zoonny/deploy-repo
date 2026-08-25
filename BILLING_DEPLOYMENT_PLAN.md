# 빌링 서비스 배포 구조 설계안

작성일: 2026-08-18 (갱신: 원격 pull 반영 → 매니페스트 구현 완료)
상태: **구현 완료** — `billing-deploy/` 매니페스트 작성 및 `kubectl kustomize` 빌드 검증 완료. 실제 클러스터 배포/이미지 빌드는 별도 진행 필요.

> **최종 결정**: FE는 nginx 직접 서빙(포트 80), BE는 Spring Actuator(`/actuator/health`) 헬스체크 사용, DB는 Helm 대신 StatefulSet 직접 운영으로 확정.

> **변경 이력**: 원격 저장소에서 커밋 `8bbbd90 (Update ingress configuration for new host)`을 pull — `dashboard-deploy/base/ingress.yaml`의 호스트가 `dashboard.dev-x.store` → `dashboard.axeng.site`로 변경됨. 이 설계안의 도메인 컨벤션을 `axeng.site`로 갱신함. (※ `platform/kube-prometheus-stack/ingress.yaml`은 아직 `dev-x.store`로 남아있어 두 도메인이 혼재된 상태 — 기존 인프라 쪽 별개 이슈이며 이번 갱신 범위에는 포함하지 않음)

## 1. 목표

- React 기반 FE(프론트엔드), Spring 기반 BE(백엔드/API), PostgreSQL DB를 각각 Pod로 배포
- Ingress를 통해 FE/BE를 외부에서 접속 가능하게 함 (DB는 내부 전용)
- 기존 `deploy-repo`의 GitOps 컨벤션(`dashboard-deploy`)과 일관된 구조 유지

## 2. 기존 컨벤션 검토 결과

| 패턴 | 사용처 | 특징 |
|---|---|---|
| `dashboard-deploy` (Kustomize) | 서비스 앱 배포 | base/overlay 구조, ArgoCD `Application` 1개, CI가 이미지 태그 커밋 |
| `platform` (Helm) | 클러스터 공통 인프라 | 원격 Helm chart를 ArgoCD가 직접 참조 |

빌링 서비스는 "우리가 만드는 애플리케이션"이므로 `dashboard-deploy` 패턴(Kustomize)을 따르는 것이 자연스럽습니다. DB도 Helm chart(Bitnami 등)를 쓸 수도 있지만, 이 앱 배포 계열은 지금까지 Helm 의존성이 없었기 때문에 **1차로는 직접 작성한 StatefulSet**을 제안합니다. (운영 중 백업/복제 요구가 커지면 Bitnami `postgresql` Helm chart로 전환하는 옵션을 열어둡니다.)

## 3. 제안 디렉토리 구조

새 최상위 폴더 `billing-deploy/`를 이 저장소(`deploy-repo`)에 추가합니다.

```
billing-deploy/
├── argocd/
│   └── application-dev.yaml        # ArgoCD Application: billing-dev
├── base/
│   ├── fe/
│   │   ├── deployment.yaml         # React (정적 빌드, nginx로 서빙 가정), port 80
│   │   ├── service.yaml            # ClusterIP 80
│   │   └── kustomization.yaml
│   ├── be/
│   │   ├── deployment.yaml         # Spring Boot, port 8080
│   │   ├── service.yaml            # ClusterIP 8080
│   │   ├── configmap.yaml          # 비민감 설정 (profile, log level 등)
│   │   └── kustomization.yaml
│   ├── db/
│   │   ├── statefulset.yaml        # postgres:16-alpine, volumeClaimTemplates
│   │   ├── service.yaml            # ClusterIP(headless) 5432, ingress 없음
│   │   ├── secret.example.yaml     # 템플릿만 커밋 (실값 미포함)
│   │   └── kustomization.yaml
│   ├── ingress.yaml                 # fe/be 서브도메인 라우팅
│   └── kustomization.yaml           # fe + be + db + ingress 통합
└── overlays/
    └── dev/
        ├── namespace.yaml           # "billing" 네임스페이스
        ├── kustomization.yaml       # ../../base 참조, 이미지 태그 override (CI가 갱신)
        └── README 안내: db secret은 최초 1회 수동 apply
```

## 4. ArgoCD Application 설계

`billing-deploy/argocd/application-dev.yaml` (dashboard-dev와 동일 패턴):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: billing-dev
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/zoonny/deploy-repo.git   # 이 저장소 자체를 소스로 사용
    targetRevision: main
    path: billing-deploy/overlays/dev
  destination:
    server: https://kubernetes.default.svc
    namespace: billing
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

> 참고: `dashboard-deploy`의 Application은 별도 저장소(`dashboard-deploy.git`)를 `repoURL`로 쓰고 있어 이 모노레포(`deploy-repo`) 구조와 다소 어긋납니다. 빌링 서비스는 "여기에" 만든다는 요청에 따라 **이 저장소 자체를 소스로 사용**하는 것으로 제안했습니다. 기존 dashboard-deploy 쪽 repoURL 불일치는 별개 이슈로 남겨둡니다.

## 5. 네트워킹 / Ingress 설계

- 네임스페이스: 전용 `billing` (dashboard의 `dev`와 분리 — DB 라이프사이클/쿼터를 독립적으로 관리하기 위함)
- 서브도메인 분리 (dashboard가 최근 전환한 `axeng.site` 도메인 컨벤션 재사용):
  - `billing.axeng.site` → `billing-fe-service:80` (React FE)
  - `billing-api.axeng.site` → `billing-be-service:8080` (Spring BE API)
- TLS: 기존과 동일하게 `nginx` ingressClass + `cert-manager.io/cluster-issuer: letsencrypt-prod`
- DB(`billing-db-service`)는 **Ingress에 노출하지 않음** — BE Pod에서만 `billing-db-service.billing.svc.cluster.local:5432`로 접근
- FE → BE 호출은 브라우저가 `billing-api.axeng.site`를 직접 호출하는 구조 가정 (BE에 CORS 허용 필요). FE 소스가 API base URL을 빌드 시점 env로 주입받는 구조라면 이 방식이 가장 단순합니다.

## 6. Secret 관리

이전 `kube-prometheus-stack`에서 Grafana 비밀번호가 평문으로 커밋된 이슈를 반복하지 않기 위해:

- `base/db/secret.example.yaml`에 키 이름만 있는 템플릿을 커밋 (`POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB`)
- 실제 값은 git에 커밋하지 않고, 최초 배포 시 수동으로 생성:
  ```shell
  kubectl create secret generic billing-db-secret -n billing \
    --from-literal=POSTGRES_USER=billing \
    --from-literal=POSTGRES_PASSWORD='<실값>' \
    --from-literal=POSTGRES_DB=billing
  ```
- BE Deployment는 이 Secret을 env로 주입 (`SPRING_DATASOURCE_USERNAME/PASSWORD`, `SPRING_DATASOURCE_URL`)
- 추후 secret 회전/재현성이 중요해지면 sealed-secrets 또는 SOPS 도입을 검토 (현재 저장소엔 아직 도구 없음, 1차는 수동 방식으로 기존 README 관례를 따름)

## 7. 이미지 / CI 연동

- FE: `ghcr.io/zoonny/billing-fe`
- BE: `ghcr.io/zoonny/billing-be`
- DB: 공식 `postgres:16-alpine` (직접 빌드 불필요)
- `overlays/dev/kustomization.yaml`의 `images:` 필드로 FE/BE 태그 관리 → dashboard-deploy와 동일하게 CI가 빌드 후 태그를 커밋하고 ArgoCD selfHeal로 반영

## 8. 리소스 / 헬스체크 초안

| 컴포넌트 | requests | limits | probe |
|---|---|---|---|
| FE | 100m / 128Mi | 250m / 256Mi | `/` (nginx) |
| BE | 250m / 512Mi | 500m / 1Gi | `/actuator/health` (Spring Actuator 필요) |
| DB | 250m / 512Mi | 500m / 1Gi, PVC 10Gi (`local-path`) | `pg_isready` exec probe |

## 9. 결정 사항 (모두 확정됨)

1. FE: nginx가 React 빌드 산출물을 직접 서빙 (포트 80). `readinessProbe`/`livenessProbe`는 `GET /`.
2. BE: Spring Actuator 사용. `readinessProbe`/`livenessProbe`는 `GET /actuator/health` (8080). `management.endpoints.web.exposure.include=health,info`는 ConfigMap(`billing-be-config`)으로 주입.
3. DB: StatefulSet 직접 운영 (`postgres:16-alpine`, `volumeClaimTemplates`로 PVC 10Gi, `local-path` StorageClass). Headless Service(`clusterIP: None`)로 안정적인 파드 DNS 제공.
4. 도메인: `billing.axeng.site`(FE), `billing-api.axeng.site`(BE) — dashboard가 최근 전환한 `axeng.site` 컨벤션을 따름.

## 10. 구현된 파일 목록

```
billing-deploy/
├── argocd/application-dev.yaml
├── base/
│   ├── fe/{deployment,service,kustomization}.yaml
│   ├── be/{deployment,service,configmap,kustomization}.yaml
│   ├── db/{statefulset,service,kustomization}.yaml, secret.example.yaml (템플릿, kustomization 미포함)
│   ├── ingress.yaml
│   └── kustomization.yaml
└── overlays/dev/{namespace,kustomization}.yaml
```

`kubectl kustomize billing-deploy/overlays/dev`로 렌더링 검증 완료.
저장소 루트에 `.gitignore`를 추가해 `*secret*.yaml` 패턴(실값 시크릿)이 실수로 커밋되지 않도록 했습니다. `*.example.yaml`은 예외로 허용됩니다.
README.md에 "billing 서비스 (fe / be / db)" 섹션을 추가해 DB 시크릿 수동 생성 → ArgoCD Application 적용 → 확인 커맨드 순서를 문서화했습니다.

## 11. 남은 작업 (이 설계 범위 밖)

- `billing-fe`, `billing-be` 실제 애플리케이션 코드/Dockerfile 작성 및 `ghcr.io/zoonny/billing-fe`, `ghcr.io/zoonny/billing-be` 이미지 빌드·푸시 (현재 `overlays/dev/kustomization.yaml`은 `newTag: latest` placeholder).
- ArgoCD에 이 저장소(`https://github.com/zoonny/deploy-repo.git`)가 실제로 소스로 등록되어 있는지 확인 (dashboard-deploy처럼 별도 레포로 분리할지도 재확인 필요).
- BE의 CORS 설정 (FE가 `billing-api.axeng.site`를 브라우저에서 직접 호출).
- DNS에 `billing.axeng.site` / `billing-api.axeng.site` 레코드 등록, cert-manager 인증서 발급 확인.
- CI 파이프라인에서 `overlays/dev/kustomization.yaml`의 이미지 태그를 자동 갱신하도록 연결 (dashboard-deploy와 동일 패턴).
