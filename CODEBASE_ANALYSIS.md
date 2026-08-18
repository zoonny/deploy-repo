# deploy-repo 코드베이스 분석

작성일: 2026-08-18

## 1. 저장소 개요

이 저장소는 Kubernetes 클러스터에 애플리케이션과 모니터링 스택을 배포하기 위한 **GitOps(ArgoCD) 매니페스트 저장소**입니다. 애플리케이션 코드는 없고, ArgoCD `Application` 리소스, Helm values, Kustomize 매니페스트만 존재합니다.

```
deploy-repo/
├── README.md                     # 운영 커맨드 치트시트 (한글)
├── dashboard-deploy/              # 사용자 서비스(dashboard) 배포 정의
│   ├── argocd/application-dev.yaml
│   ├── base/                      # Kustomize base
│   └── overlays/dev/              # dev 환경 오버레이 (CI가 이미지 태그 갱신)
└── platform/                      # 클러스터 공통 플랫폼 컴포넌트
    ├── metrics-server/
    ├── kube-prometheus-stack/     # Prometheus / Grafana / Alertmanager
    ├── loki/                      # (미구현, 빈 파일)
    ├── alloy/                     # (미구현, 빈 파일)
    └── argocd-notifications/      # (미구현, 빈 파일)
```

크게 두 축으로 구성되어 있습니다.

- **`dashboard-deploy`**: 실제 서비스(`dashboard`)를 Kustomize base/overlay 구조로 배포. ArgoCD가 Git 저장소(`dashboard-deploy` 리포, 경로 `overlays/dev`)를 소스로 사용.
- **`platform`**: 클러스터에 필요한 공통 인프라(metrics-server, kube-prometheus-stack 등)를 각각 ArgoCD `Application`으로 정의하고, 소스는 원격 Helm chart repo를 직접 참조.

## 2. dashboard-deploy (애플리케이션 배포)

- `argocd/application-dev.yaml`: ArgoCD `Application`. 소스는 GitHub의 `dashboard-deploy` 리포 `overlays/dev` 경로, 대상은 `dev` 네임스페이스. `automated: {prune:true, selfHeal:true}` + `CreateNamespace=true`로 완전 자동 동기화.
- `base/`: Kustomize 베이스
  - `deployment.yaml`: `ghcr.io/zoonny/dashboard` 이미지, replica 2, 포트 3000, requests/limits 지정.
  - `service.yaml`: ClusterIP, 80→3000.
  - `ingress.yaml`: nginx ingress + cert-manager(`letsencrypt-prod`)로 `dashboard.dev-x.store` TLS 종단.
- `overlays/dev/`: `namespace.yaml`(dev 생성) + base 참조, `images:` 필드로 이미지 태그를 고정.
  - Git 로그(`Update dev image to <sha>` 커밋들)를 보면, **CI가 빌드 후 이 `newTag` 값을 자동 커밋**하고 ArgoCD가 selfHeal로 반영하는 전형적인 GitOps 배포 흐름임을 알 수 있습니다.

## 3. platform (클러스터 공통 컴포넌트)

### 3.1 metrics-server (수정됨, 아직 커밋 안 됨)

- `application.yaml`: 이전에는 `helm.valueFiles: [values.yaml]`로 로컬 `values.yaml`을 참조했으나, 이번 변경으로 **inline `helm.values`**로 바뀌며 `args: []`, resource requests/limits를 직접 명시하도록 변경됨.
- `values.yaml`: `args:` 오타(`args\:` → `args:`) 수정 + `limits` 추가.

⚠️ **주의할 점**: ArgoCD `Application`의 source가 원격 Helm 저장소(`kubernetes-sigs.github.io/metrics-server`) 하나뿐이라, `valueFiles: values.yaml`은 실제로는 (별도 `sources`/`ref` 구성 없이는) 참조가 불가능해 무시되었을 가능성이 높습니다. README에도 배포 후 `kubectl edit deployment metrics-server`로 `--kubelet-insecure-tls`를 **수동으로** 추가하라는 절차가 있는데, 이는 `values.yaml`의 `args`가 실제 배포에 반영되지 않고 있었다는 정황 증거입니다. 이번 변경으로 `application.yaml`에 `args: []`를 inline으로 명시했지만, 여전히 `--kubelet-insecure-tls`가 빠져 있어 README의 수동 조치가 계속 필요합니다.

### 3.2 kube-prometheus-stack (수정됨, 아직 커밋 안 됨)

- `application.yaml`(변경 전은 빈 파일, 이번에 최초 작성): Grafana(admin/`New123!@#`, PVC 10Gi), Prometheus(retention 10d/4GB, PVC 5Gi, `local-path` StorageClass), kube-state-metrics, nodeExporter를 inline helm values로 정의. `targetRevision: 69.*`.
- `values.yaml`: `application.yaml`과 **거의 동일한 내용을 중복 보유** (alertmanager 섹션은 주석 처리 상태). 위 metrics-server 사례와 동일하게, Application이 이 파일을 실제로 로드하는 경로가 없어 **참고용 사본**에 가깝습니다.
- `ingress.yaml` (신규, untracked): `grafana.dev-x.store`, `prometheus.dev-x.store` 두 호스트를 nginx ingress + cert-manager TLS로 라우팅. README에 `kubectl apply`로 수동 적용하는 절차가 있음 (kube-prometheus-stack Helm chart의 `ingress.enabled: false`로 두고 별도 관리).
- `application.yaml.bk` (신규, untracked): 백업 파일로 보이며, 내용은 alertmanager(Slack webhook 포함) 섹션이 활성화된 이전 버전. **저장소에 남겨둘 이유가 없는 임시 파일**로 보입니다.

⚠️ **보안 이슈**: Grafana 관리자 비밀번호(`New123!@#`)가 `application.yaml`/`values.yaml`/`.bk` 파일에 **평문으로 커밋**되어 있습니다. Git 히스토리에 남으므로, Kubernetes `Secret` + ArgoCD `sops`/`sealed-secrets` 또는 최소한 `.gitignore` 처리된 별도 secret 파일로 분리하는 것을 권장합니다. `application.yaml.bk`에 있던 Slack webhook URL(`https://hooks.slack.com/services/REPLACE/ME`)은 placeholder이지만, 실제 값으로 채워질 경우도 동일한 문제가 됩니다.

### 3.3 loki / alloy / argocd-notifications (플레이스홀더)

- `platform/loki/{application,values}.yaml`
- `platform/alloy/{application,values,config.alloy}`
- `platform/argocd-notifications/{application,notifications-cm,notifications-secret}.yaml`

모두 `init platform` 커밋(00edeaa)에서 **빈 파일(0바이트)**로 생성된 이후 한 번도 채워진 적이 없습니다. README에는 `loki`, `alloy`(로그 수집), `argocd-notifications`가 모니터링 스택의 일부로 언급되어 있어, 로깅 파이프라인과 ArgoCD 알림 설정은 **아직 구현 전 단계**임을 나타냅니다.

## 4. 현재 워킹 디렉토리 변경사항 요약 (커밋 전)

| 파일 | 상태 | 내용 |
|---|---|---|
| `README.md` | 수정 | 운영 명령어를 대폭 보강 (배포/확인/삭제 커맨드, 트러블슈팅용 주석 명령 다수) |
| `platform/kube-prometheus-stack/application.yaml` | 수정 | 빈 파일 → Grafana/Prometheus 등 inline values로 최초 작성 |
| `platform/kube-prometheus-stack/values.yaml` | 수정 | 동일 내용의 참고용 사본 작성 |
| `platform/metrics-server/application.yaml` | 수정 | `valueFiles` 참조 → inline `values`로 전환, resource limits 추가 |
| `platform/metrics-server/values.yaml` | 수정 | YAML 문법 오타 수정 + limits 추가 |
| `platform/kube-prometheus-stack/application.yaml.bk` | 신규(untracked) | 백업 파일 (정리 대상으로 보임) |
| `platform/kube-prometheus-stack/ingress.yaml` | 신규(untracked) | Grafana/Prometheus ingress 정의 |

## 5. 배포 흐름 요약

1. **애플리케이션(dashboard)**: CI → `overlays/dev/kustomization.yaml`의 이미지 태그 갱신 커밋 → ArgoCD `selfHeal`로 자동 반영.
2. **플랫폼 컴포넌트**: 각 컴포넌트별 ArgoCD `Application`이 원격 Helm chart를 직접 참조 (`targetRevision`은 대부분 `x.*` 형태의 느슨한 버전 고정 — 자동 마이너 업그레이드 위험 있음).
3. **수동 단계 다수 존재**: metrics-server의 `--kubelet-insecure-tls` 추가, kube-prometheus-stack ingress 적용 등은 README의 수동 `kubectl` 절차에 의존 — GitOps 원칙(선언적 관리)과는 다소 어긋나는 부분.

## 6. 권장 사항 (요약)

1. `application.yaml.bk` 삭제 또는 `.gitignore` 처리.
2. Grafana admin password를 平文 커밋에서 Secret 기반으로 분리.
3. `values.yaml`이 실제로 ArgoCD에 의해 로드되지 않는다면(다중 `sources`+`ref` 구성이 없다면) 혼란을 막기 위해 제거하거나, 반대로 다중 소스 구성을 추가해 실제로 참조되도록 통일.
4. README에 남아있는 수동 `kubectl edit` 단계(`--kubelet-insecure-tls`)를 `application.yaml`의 inline values에 포함시켜 완전 선언적으로 전환.
5. `targetRevision`의 와일드카드(`3.*`, `69.*`) 사용 시 의도치 않은 업그레이드를 막기 위해 필요하면 더 좁은 범위로 고정 검토.
6. `loki`/`alloy`/`argocd-notifications`는 현재 미구현 상태이므로, README의 "monitoring: ... loki / alloy" 문구와 실제 상태 간 괴리를 인지하고 우선순위에 따라 채우거나 문서에 "TODO" 명시.
