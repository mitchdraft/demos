

## Setup
- run these steps before beginning the specific demos
  - setup glooshot
  - install istio
  - set prometheus stats
  - deploy grafana
  - configure grafana dashboards
  - install loop
- install the apps
  - bookstore
    - [ ] setup new namespace for it, test this
  - calculator
    - annotate for specific envoy
    - restart pods

### Setup glooshot
```bash
# glooshot init
cd /Users/mitch/go/src/github.com/solo-io/glooshot
REGISTER_GLOOSHOT=1 ./_output/glooshot-darwin-amd64 register
./_output/glooshot-darwin-amd64 init -f _output/helm/charts/glooshot-tute2e7.tgz
```

### Install Istio
```bash
supergloo install istio \
    --namespace glooshot \
    --name istio-istio-system \
    --installation-namespace istio-system \
    --mtls=false \
    --auto-inject=true
```

### Set Prometheus stats
```bash
supergloo set mesh stats \
    --target-mesh glooshot.istio-istio-system \
    --prometheus-configmap glooshot.glooshot-prometheus-server
```
- verify that the stats are there
```bash
kubectl get cm -n glooshot glooshot-prometheus-server -o yaml | rg istio
```
- way to do that from a script (sets exit value to 0 or 1 depending on result)
```bash
test 0 -lt `kubectl get cm -n glooshot glooshot-prometheus-server -o yaml | rg istio|wc -l`
```

### Deploy Grafana
```bash
kubectl create ns grafana
kubectl apply -f install/tmpgrafana.yaml -n grafana
```



## Port forwards
```bash
kubectl port-forward -n grafana deployment/kubecon-eu-grafana 3000
```
