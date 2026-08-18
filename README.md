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