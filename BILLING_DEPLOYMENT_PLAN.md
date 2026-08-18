# 빌링 서비스 배포 구조 설계안

작성일: 2026-08-18
상태: **설계안 (아직 매니페스트 미구현)** — 리뷰 후 실제 파일 작성 진행

## 1. 목표

- React 기반 FO(프론트오피스), Spring 기반 BO(백오피스/API), PostgreSQL DB를 각각 Pod로 배포
- Ingress를 통해 FO/BO를 외부에서 접속 가능하게 함 (DB는 내부 전용)
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
│   ├── fo/
│   │   ├── deployment.yaml         # React (정적 빌드, nginx로 서빙 가정), port 80
│   │   ├── service.yaml            # ClusterIP 80
│   │   └── kustomization.yaml
│   ├── bo/
│   │   ├── deployment.yaml         # Spring Boot, port 8080
│   │   ├── service.yaml            # ClusterIP 8080
│   │   ├── configmap.yaml          # 비민감 설정 (profile, log level 등)
│   │   └── kustomization.yaml
│   ├── db/
│   │   ├── statefulset.yaml        # postgres:16-alpine, volumeClaimTemplates
│   │   ├── service.yaml            # ClusterIP(headless) 5432, ingress 없음
│   │   ├── secret.example.yaml     # 템플릿만 커밋 (실값 미포함)
│   │   └── kustomization.yaml
│   ├── ingress.yaml                 # fo/bo 서브도메인 라우팅
│   └── kustomization.yaml           # fo + bo + db + ingress 통합
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
    repoURL: git@github.com:zoonny/deploy-repo   # 이 저장소 자체를 소스로 사용
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
- 서브도메인 분리 (기존 `dev-x.store` 도메인 컨벤션 재사용):
  - `billing.dev-x.store` → `billing-fo-service:80` (React FO)
  - `billing-api.dev-x.store` → `billing-bo-service:8080` (Spring BO API)
- TLS: 기존과 동일하게 `nginx` ingressClass + `cert-manager.io/cluster-issuer: letsencrypt-prod`
- DB(`billing-db-service`)는 **Ingress에 노출하지 않음** — BO Pod에서만 `billing-db-service.billing.svc.cluster.local:5432`로 접근
- FO → BO 호출은 브라우저가 `billing-api.dev-x.store`를 직접 호출하는 구조 가정 (BO에 CORS 허용 필요). FO 소스가 API base URL을 빌드 시점 env로 주입받는 구조라면 이 방식이 가장 단순합니다.

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
- BO Deployment는 이 Secret을 env로 주입 (`SPRING_DATASOURCE_USERNAME/PASSWORD`, `SPRING_DATASOURCE_URL`)
- 추후 secret 회전/재현성이 중요해지면 sealed-secrets 또는 SOPS 도입을 검토 (현재 저장소엔 아직 도구 없음, 1차는 수동 방식으로 기존 README 관례를 따름)

## 7. 이미지 / CI 연동

- FO: `ghcr.io/zoonny/billing-fo`
- BO: `ghcr.io/zoonny/billing-bo`
- DB: 공식 `postgres:16-alpine` (직접 빌드 불필요)
- `overlays/dev/kustomization.yaml`의 `images:` 필드로 FO/BO 태그 관리 → dashboard-deploy와 동일하게 CI가 빌드 후 태그를 커밋하고 ArgoCD selfHeal로 반영

## 8. 리소스 / 헬스체크 초안

| 컴포넌트 | requests | limits | probe |
|---|---|---|---|
| FO | 100m / 128Mi | 250m / 256Mi | `/` (nginx) |
| BO | 250m / 512Mi | 500m / 1Gi | `/actuator/health` (Spring Actuator 필요) |
| DB | 250m / 512Mi | 500m / 1Gi, PVC 10Gi (`local-path`) | `pg_isready` exec probe |

## 9. 아직 확정 안 된 부분 (구현 전 확인 필요)

1. FO가 nginx로 정적 서빙되는 컨테이너인지, Node 서버(`next start` 등)로 3000 포트를 쓰는 방식인지 — Dockerfile 구조에 따라 `deployment.yaml`/`service.yaml` 포트가 달라짐.
2. BO에 Spring Actuator가 포함되어 있는지 (헬스체크 엔드포인트).
3. DB를 StatefulSet 직접 관리할지, Bitnami Helm chart로 전환할지 — 백업/복제 요구 수준에 따라 결정.
4. 도메인은 `dev-x.store` 서브도메인을 그대로 재사용할지, 별도 도메인을 쓸지.

## 10. 다음 단계

이 설계안에 이견이 없으면, 위 구조대로 실제 Deployment/Service/StatefulSet/Ingress/Kustomization/ArgoCD Application 매니페스트 파일을 작성하겠습니다.
