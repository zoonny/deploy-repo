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
# 1) DB 시크릿을 먼저 수동으로 생성 (git에는 값이 없는 템플릿만 존재: billing-deploy/base/db/secret.example.yaml)
kubectl create secret generic billing-db-secret -n billing \
  --from-literal=POSTGRES_USER=billing \
  --from-literal=POSTGRES_PASSWORD='<실값>' \
  --from-literal=POSTGRES_DB=billing

# 2) ArgoCD Application 등록
kubectl apply -f billing-deploy/argocd/application-dev.yaml

kubectl get pods -n billing
kubectl get pvc -n billing
kubectl get ingress -n billing

# billing.axeng.site      -> React FE (nginx 직접 서빙)
# billing-api.axeng.site  -> Spring BE (Actuator: /actuator/health)

# DB는 ingress로 노출하지 않음. 내부 접속만 필요:
kubectl exec -it -n billing billing-db-0 -- psql -U billing -d billing
```