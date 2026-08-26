# 빌링 서비스 배포 구조 설계안

작성일: 2026-08-18 (갱신: 2026-08-25 — 실제 앱 코드와 대조해 매니페스트 전면 수정)
상태: **매니페스트 수정 완료, 클러스터 미검증** — 최초 작성본은 실제 앱 코드를 보지 않고 만들어져
백엔드가 기동조차 하지 않는 상태였다(`BILLING_ADMIN_PASSWORD` 미주입 → `ProdAccountGuard` 기동 거부).
2026-08-25에 앱 코드와 대조해 아래 항목을 바로잡았다. `kubectl kustomize` 렌더 검증은 통과했으나
**클러스터 배포·이미지 빌드는 아직 수행하지 않았다.**

> **최종 결정**: FE는 **Caddy**가 정적 빌드를 서빙(포트 80, SPA 폴백), BE는 Spring Actuator
> 헬스체크, DB는 Helm 대신 StatefulSet 직접 운영. 라우팅은 **단일 호스트 경로 분기**
> (`billing.axeng.site` 하나) — 앱이 same-origin SPA 전제라 서브도메인 분리는 동작하지 않는다.

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
- **단일 호스트 경로 분기** (`axeng.site` 도메인 컨벤션 재사용):

  | 경로 | 백엔드 |
  |---|---|
  | `/api` (Prefix) | `billing-be-service:8080` |
  | `/actuator/health` (Prefix) | `billing-be-service:8080` |
  | `/` (Prefix) | `billing-fe-service:80` |

  nginx는 설정 순서와 무관하게 최장 prefix location을 고르므로 `/api`가 `/`를 이긴다.
  `rewrite-target`은 쓰지 않는다 — 백엔드가 `/api/v1/...` 전체 경로를 기대하고, 이 annotation은
  ingress-nginx를 regex location 모드로 바꿔 매칭 의미를 흔든다.
- TLS: 기존과 동일하게 `nginx` ingressClass + `cert-manager.io/cluster-issuer: letsencrypt-prod`
- DB(`billing-db-service`)는 **Ingress에 노출하지 않음** — BE Pod에서만 `billing-db-service.billing.svc.cluster.local:5432`로 접근
- **서브도메인 분리(`billing-api.axeng.site`)는 폐기했다.** 앱이 same-origin SPA 전제로
  만들어져 있어 CORS를 열어도 동작하지 않는다:
  - `frontend-admin/src/api/http.ts`의 `fetch`에 `credentials` 옵션이 없다 → 기본값
    `same-origin` → cross-origin이면 `SameSite=None`으로 바꿔도 세션 쿠키가 실리지 않는다.
  - `application.yml`이 세션 쿠키를 `same-site: strict`로 두고, OPS-001 R4가 이를 CSRF
    방어선으로 명시한다. `docs/manual/10-운영-배포.md`도 "다른 오리진에서 이 API를 부르는
    구성은 전제 밖"이라고 못박는다.
  - `src/main/java` 전체에 CORS 설정이 없다.
- FE 이미지의 `Caddyfile`은 `reverse_proxy app:8080`(compose 서비스명)을 가리키지만, k8s에서는
  ingress가 `/api`를 BE로 먼저 보내므로 그 핸들러에 도달하지 않는다. **Caddyfile은 고치지
  않는다** — 고치면 `docker-compose.prod.yml`(백엔드 서비스명이 실제로 `app`)이 깨진다.

## 6. Secret 관리

이전 `kube-prometheus-stack`에서 Grafana 비밀번호가 평문으로 커밋된 이슈를 반복하지 않기 위해:

- 템플릿 2개를 커밋 (키 이름만, 실값 없음):
  - `base/db/secret.example.yaml` — `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB`
  - `base/be/secret.example.yaml` — `BILLING_ADMIN_USERNAME`, `BILLING_ADMIN_PASSWORD`
- 실제 값은 git에 커밋하지 않고, 최초 배포 시 수동으로 생성:
  ```shell
  kubectl create secret generic billing-db-secret -n billing \
    --from-literal=POSTGRES_USER=billing \
    --from-literal=POSTGRES_PASSWORD='<실값>' \
    --from-literal=POSTGRES_DB=billing

  kubectl create secret generic billing-admin-secret -n billing \
    --from-literal=BILLING_ADMIN_USERNAME=admin \
    --from-literal=BILLING_ADMIN_PASSWORD='<실값>'

  kubectl create secret docker-registry ghcr-secret -n billing \
    --docker-server=ghcr.io \
    --docker-username=zoonny \
    --docker-password='<PAT: read:packages>'
  ```
- **`ghcr-secret`은 성격이 다르다** — 위 두 개는 앱 컨테이너가 env로 읽지만, 이건 **kubelet**이
  이미지를 받을 때 쓴다(`type: kubernetes.io/dockerconfigjson`). `base/{fe,be}/deployment.yaml`의
  `imagePullSecrets`가 이름으로 참조하며, 파드와 **같은 네임스페이스**에 있어야 한다.
  템플릿(`*.example.yaml`)을 두지 않은 이유: `.dockerconfigjson`은 손으로 편집할 값이 아니라
  `kubectl create secret docker-registry`가 조립하는 값이다. PAT는 **classic + `read:packages`** —
  `GITHUB_TOKEN`은 CI 런 동안만 유효해서 클러스터에 넣으면 만료 시 pull이 조용히 깨진다.
- **admin 시크릿을 DB 시크릿과 분리한 이유**: `base/db/statefulset.yaml`이
  `envFrom: secretRef: billing-db-secret`으로 **모든 키**를 postgres 컨테이너 환경에 넣는다.
  앱 관리자 자격증명이 DB 컨테이너에 섞이면 안 되고, 회전 주기도 다르다(DB는 `ALTER ROLE`
  필요, admin은 파드 재시작).
- BE Deployment는 두 Secret을 env로 주입한다. **키 이름은 `DB_HOST`/`DB_PORT`/`DB_NAME`/
  `DB_USER`/`DB_PASSWORD` + `BILLING_ADMIN_USERNAME`/`BILLING_ADMIN_PASSWORD`** —
  `application-prod.yml`이 읽는 노브가 이것뿐이다. `SPRING_DATASOURCE_*`를 주면 relaxed
  binding으로 동작은 하지만 앱이 선언한 계약(compose와 동일한 `DB_*`)과 두 벌이 된다.
- `BILLING_ADMIN_PASSWORD`가 없으면 prod 프로파일은 기동을 거부한다(OPS-001 R2) —
  최초 작성본이 이걸 빠뜨려 백엔드가 CrashLoopBackOff로 떨어지는 상태였다.
- 추후 secret 회전/재현성이 중요해지면 sealed-secrets 또는 SOPS 도입을 검토 (현재 저장소엔 아직 도구 없음, 1차는 수동 방식으로 기존 README 관례를 따름)

## 7. 이미지 / CI 연동

- FE: `ghcr.io/zoonny/billing-fe`
- BE: `ghcr.io/zoonny/billing-be`
- DB: 공식 `postgres:16-alpine` (직접 빌드 불필요, Docker Hub public이라 pull secret 대상 아님)
- **FE/BE의 GHCR 패키지는 private으로 유지**하고 `ghcr-secret`으로 pull한다 (§9-6)
- `overlays/dev/kustomization.yaml`의 `images:` 필드로 FE/BE 태그 관리 → dashboard-deploy와 동일하게 CI가 빌드 후 태그를 커밋하고 ArgoCD selfHeal로 반영

## 8. 리소스 / 헬스체크 초안

| 컴포넌트 | replicas | requests | limits | probe |
|---|---|---|---|---|
| FE | 2 | 100m / 128Mi | 250m / 256Mi | `/` (Caddy, SPA 폴백이라 항상 200) |
| BE | **1** | 250m / 512Mi | 500m / 1Gi | startup·readiness `/actuator/health`, liveness `/actuator/health/liveness` |
| DB | 1 | 250m / 512Mi | 500m / 1Gi, PVC 10Gi (`local-path`) | `pg_isready -U billing -d billing` exec probe |

- BE `startupProbe`(최대 200초)를 따로 둔 이유: 톰캣은 Flyway 마이그레이션이 끝난 뒤에야 접속을
  받으므로 콜드 JVM에서는 readiness 기본 예산(약 50초)이 빠듯하다.
- BE liveness를 집계 `/actuator/health`가 아니라 `livenessState`로 분리한 이유: 집계값은 `db`
  인디케이터를 포함해서, DB가 잠깐 흔들리면 kubelet이 파드를 죽이고 재기동 시 Flyway가 실패해
  CrashLoopBackOff로 번진다. liveness는 "JVM이 멈췄나"를 물어야지 "의존성이 건강한가"를
  물으면 안 된다.
- DB probe에 `$(POSTGRES_USER)`를 쓰면 안 된다 — `$(VAR)` 치환은 컨테이너 `command`/`args`에만
  적용되고 probe `exec.command`에서는 리터럴로 전달된다(최초 작성본의 버그).

## 9. 결정 사항 (모두 확정됨)

1. FE: **Caddy**(`caddy:2-alpine`)가 React 빌드 산출물을 서빙 (포트 80, SPA 폴백).
   `readinessProbe`/`livenessProbe`는 `GET /`. — 최초 작성본은 nginx로 적었으나
   `frontend-admin/Dockerfile`의 런타임 이미지는 Caddy다.
2. BE: Spring Actuator 사용. startup·readiness는 `GET /actuator/health`, liveness는
   `GET /actuator/health/liveness` (8080).
   **ConfigMap으로 `MANAGEMENT_ENDPOINTS_WEB_EXPOSURE_INCLUDE`를 주지 않는다** — env가
   `application-prod.yml`의 `include: health`를 이겨서 `/actuator/info`까지 다시 열린다
   (OPS-001 R5 위반). 노출 정책은 앱의 prod 프로파일이 단독으로 정한다. ConfigMap에는
   `MANAGEMENT_ENDPOINT_HEALTH_PROBES_ENABLED: "true"`만 두는데, 이는 노출을 늘리지 않고
   이미 열린 health의 하위 그룹(liveness/readiness)을 보장할 뿐이다.
3. DB: StatefulSet 직접 운영 (`postgres:16-alpine`, `volumeClaimTemplates`로 PVC 10Gi,
   `local-path` StorageClass). Service는 **일반 ClusterIP** — 복제·피어 디스커버리가 없고
   BE는 서비스 이름으로만 붙으므로 headless가 필요 없다. 고정 VIP가 파드 IP 변경에 더 강하다
   (JVM DNS 캐시 30초).
4. 도메인: **`billing.axeng.site` 하나**로 FE·BE를 모두 서빙(경로 분기, §5).
   `billing-api.axeng.site`는 폐기 — same-origin 전제 때문에 동작 불가.
5. BE `replicas: 1` + `strategy: Recreate`. 세션이 JVM 힙에 있고(`InMemoryUserDetailsManager`,
   세션 저장소 없음) 2개면 요청이 다른 파드로 튈 때마다 로그아웃된다. 가용성을 우선하려면
   `replicas: 2` + ingress 쿠키 어피니티(`nginx.ingress.kubernetes.io/affinity: cookie`)가
   대안이지만, 롤아웃·축출 때의 세션 유실은 여전히 남으므로 권장하지 않는다.
6. **GHCR 패키지는 private 유지 + `imagePullSecrets: ghcr-secret`.** CI가 `GITHUB_TOKEN`으로
   푸시하면 패키지가 private으로 생기는데, 최초 작성본은 `imagePullSecrets`가 없어 그대로면
   `ImagePullBackOff`였다. 대안이던 "패키지를 public 전환"(dashboard의 방식)은 이미지를 공개하게
   되므로 채택하지 않았다 — **dashboard와 컨벤션이 갈라지는 지점이니 주의.** 시크릿은
   `base/{fe,be}/deployment.yaml`의 파드 스펙에 직접 건다. `billing` ns의 `default`
   ServiceAccount에 붙이는 방법도 있지만, git에 남지 않는 암묵적 설정이라 배제했다.
   `ghcr-secret`은 수동 생성이라 ArgoCD 추적 라벨이 없고, 따라서 `prune: true`의 대상이 아니다
   (기존 두 시크릿과 동일).

## 10. 구현된 파일 목록

```
billing-deploy/
├── argocd/application-dev.yaml
├── base/
│   ├── fe/{deployment,service,kustomization}.yaml
│   ├── be/{deployment,service,configmap,kustomization}.yaml, secret.example.yaml (템플릿, kustomization 미포함)
│   ├── db/{statefulset,service,kustomization}.yaml, secret.example.yaml (템플릿, kustomization 미포함)
│   ├── ingress.yaml
│   └── kustomization.yaml
└── overlays/dev/{namespace,kustomization}.yaml
```

`kubectl kustomize billing-deploy/overlays/dev`로 렌더링 검증 완료.
저장소 루트에 `.gitignore`를 추가해 `*secret*.yaml` 패턴(실값 시크릿)이 실수로 커밋되지 않도록 했습니다. `*.example.yaml`은 예외로 허용됩니다.
README.md에 "billing 서비스 (fe / be / db)" 섹션을 추가해 시크릿 3개(`billing-db-secret`, `billing-admin-secret`, `ghcr-secret`) 수동 생성 → ArgoCD Application 적용 → 확인 커맨드 순서를 문서화했습니다.

## 11. 남은 작업 (이 설계 범위 밖)

- **클러스터 실배포 미검증** — 로컬에 kubecontext가 없어 `kubectl kustomize` 렌더까지만 확인했다.
  서버 dry-run·스모크 체크는 수행하지 않았다.
- **`ghcr-secret` 생성** — classic PAT(`read:packages`) 발급 후 README §billing의 커맨드 실행.
  매니페스트의 `imagePullSecrets`만 머지되고 시크릿이 없으면 파드가 `ImagePullBackOff`로 남는다.
  PAT에 만료일을 걸었다면 회전 일정을 별도로 관리해야 한다 (README에 회전 절차 기재).
- `ghcr.io/zoonny/billing-fe`, `ghcr.io/zoonny/billing-be` 이미지 최초 빌드·푸시.
  현재 `overlays/dev/kustomization.yaml`은 `newTag: latest` placeholder인데,
  `imagePullPolicy: IfNotPresent`와 조합되면 노드가 처음 받은 `latest`에 고정되어 배포가
  조용히 무효화된다 — **CI 첫 실행이 반드시 실제 SHA로 교체해야 한다.**
- ArgoCD에 이 저장소(`https://github.com/zoonny/deploy-repo.git`)가 실제로 소스로 등록되어 있는지 확인 (dashboard-deploy처럼 별도 레포로 분리할지도 재확인 필요).
- DNS에 `billing.axeng.site` A 레코드 등록, cert-manager 인증서 발급 확인.
  (`billing-api.axeng.site`는 폐기 — 레코드가 있다면 제거 가능)
- 배포 후 `/actuator/health/liveness`가 200인지 확인. 404면 liveness를 `/actuator/health` +
  `failureThreshold: 6`으로 되돌리고 ConfigMap의 `MANAGEMENT_ENDPOINT_HEALTH_PROBES_ENABLED`를
  제거한다 — liveness 경로가 404면 kubelet이 파드를 재시작 루프에 빠뜨린다.
- 클러스터에 `billing-db-service`가 이미 headless로 존재하면 `spec.clusterIP` 불변 제약으로
  apply가 실패한다 — 먼저 `kubectl delete svc billing-db-service -n billing`.
- **NetworkPolicy 보류**: 레포 전체에 하나도 없고 적용 여부가 CNI에 달렸다(plain flannel은
  조용히 무시 — 없는 것보다 나쁜 거짓 안심). CNI 시행 여부 확인 후 별도 변경으로.
- **백업 CronJob 보류**: `local-path` PVC에 덤프를 쓰면 원본과 같은 노드 디스크에 저장된다 —
  백업이 아니라 사본이다. 오프노드 대상(S3/NFS)이 생기기 전까지는 README의 수동 절차를 쓴다.
  `scripts/db-backup.sh`는 `docker exec` 기반 compose 산출물이고 OPS-001 R8은 compose 배포가 충족한다.
- CI 파이프라인 연결은 `billing` 레포의 `.github/workflows/deploy.yml`이 담당한다
  (`kustomize edit set image`로 `overlays/dev/kustomization.yaml` 갱신 → ArgoCD selfHeal).
