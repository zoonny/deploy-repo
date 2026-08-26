# dashboard-deploy

### k8s 모니터링 시스템

```shell
# kube-system: metrics-server
# monitoring: prometheus / grafana / alertmanager / loki / alloy argocd: argocd-notifications

# metrics-server
kubectl apply -f platform/metrics-server/application.yaml

kubectl edit deployment metrics-server -n kube-system
# find args
# - --kubelet-insecure-tls  # 이 라인을 추가하세요

kubectl top nodes
kubectl top pods -A
kubectl get apiservice | grep metrics

# kube-prometheus-stack
kubectl apply -f platform/kube-prometheus-stack/application.yaml

kubectl get pods -n monitoring
kubectl get pvc -n monitoring
kubectl get svc -n monitoring -o wide

# kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80
# kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090

kubectl apply -f platform/kube-prometheus-stack/ingress.yaml

kubectl get ingress -n monitoring
kubectl describe ingress -n monitoring
kubectl get secret -n monitoring
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller
kubectl get endpoints -n monitoring

# kubectl get prometheus -n monitoring
# kubectl get statefulset -n monitoring
# kubectl get pods -n monitoring | grep prometheus
# kubectl get svc -n monitoring kube-prometheus-stack-prometheus -o yaml
# kubectl get endpoints -n monitoring kube-prometheus-stack-prometheus -o yaml
# kubectl get pvc -n monitoring
# kubectl describe prometheus -n monitoring
# kubectl logs -n monitoring deploy/kube-prometheus-stack-operator | tail -n 200

kubectl delete namespace monitoring
kubectl delete all --all -n monitoring

kubectl get all -n monitoring

```

### billing 서비스 (fe / be / db)

```shell
# 1) 시크릿 3개를 먼저 수동 생성 (git에 실값은 없다 — db/be는 값 없는 템플릿만 존재:
#    billing-deploy/base/db/secret.example.yaml, billing-deploy/base/be/secret.example.yaml.
#    ghcr-secret은 템플릿도 없고 아래 커맨드가 유일한 정의다)
kubectl create namespace billing

kubectl create secret generic billing-db-secret -n billing \
  --from-literal=POSTGRES_USER=billing \
  --from-literal=POSTGRES_PASSWORD='<실값>' \
  --from-literal=POSTGRES_DB=billing

# BILLING_ADMIN_PASSWORD가 없으면 BE는 기동을 거부한다 (OPS-001 R2, ProdAccountGuard).
# DB 시크릿과 분리한 이유: billing-db-secret은 postgres 컨테이너가 envFrom으로
# 통째로 가져가므로 앱 관리자 비밀번호가 섞이면 안 된다.
kubectl create secret generic billing-admin-secret -n billing \
  --from-literal=BILLING_ADMIN_USERNAME=admin \
  --from-literal=BILLING_ADMIN_PASSWORD='<실값>'

# GHCR 이미지 pull 시크릿. billing-fe/billing-be 패키지가 private이라
# (CI가 GITHUB_TOKEN으로 푸시 → 기본 private) 없으면 ImagePullBackOff가 난다.
# PAT는 classic + read:packages 스코프. GITHUB_TOKEN은 CI 런 동안만 유효하므로 쓰면 안 된다.
# 앱이 아니라 kubelet이 쓰는 시크릿이라 fe/be deployment의 imagePullSecrets가 참조한다.
kubectl create secret docker-registry ghcr-secret -n billing \
  --docker-server=ghcr.io \
  --docker-username=zoonny \
  --docker-password='<PAT: read:packages>'

# PAT 만료 시 pull이 조용히 깨진다. 회전은 삭제 후 재생성 + 파드 재시작:
#   kubectl delete secret ghcr-secret -n billing
#   (위 create를 새 PAT로 다시 실행)
#   kubectl rollout restart deploy/billing-be deploy/billing-fe -n billing

# 2) ArgoCD Application 등록 (시크릿 생성 후에! 안 그러면 CreateContainerConfigError)
kubectl apply -f billing-deploy/argocd/application-dev.yaml

kubectl get pods -n billing
kubectl get pvc -n billing
kubectl get ingress -n billing

# 단일 호스트 경로 라우팅 — FE와 API가 같은 오리진이어야 한다.
# 세션 쿠키가 SameSite=Strict이고 SPA의 fetch에 credentials 옵션이 없어서,
# 서브도메인을 분리하면 CORS를 열어도 로그인이 동작하지 않는다 (OPS-001 R4).
#   https://billing.axeng.site/                 -> React FE (Caddy 정적 서빙 + SPA 폴백)
#   https://billing.axeng.site/api/**           -> Spring BE
#   https://billing.axeng.site/actuator/health  -> Spring BE (prod는 health만 공개, R5)
#
# /swagger-ui·/v3/api-docs는 ingress에 노출하지 않는다 — port-forward로만 접근.
# 주의: FE 서비스에 직접 port-forward해서 /api를 치면 Caddy가 502를 낸다.
#       Caddyfile의 upstream(app:8080)은 compose 전용이고, k8s에서는 ingress가
#       /api를 BE로 먼저 보내므로 그 핸들러에 도달하지 않는다.

# DB는 ingress로 노출하지 않음. 내부 접속만 필요:
kubectl exec -it -n billing billing-db-0 -- psql -U billing -d billing

# 백업 (임시 수동 절차 — CronJob 미도입. 사유는 BILLING_DEPLOYMENT_PLAN.md 참고)
kubectl exec -n billing billing-db-0 -- pg_dump -U billing -d billing --no-owner \
  | gzip > billing-$(date +%Y%m%d-%H%M%S).sql.gz
```