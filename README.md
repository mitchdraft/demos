

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

### Deploy Grafana, configure dashboards
```bash
kubectl create ns grafana
kubectl apply -f install/tmpgrafana.yaml -n grafana
# get password:
kubectl get secret -n grafana kubecon-eu-grafana -o jsonpath='{.data.admin-password}'|base64 --decode
## maybe it's: vlAMl8lkU9bmeiiToiYbnZkFixRIXJwRhejj6pIm
```
- add data source:
  - Prometheus
  - URL: http://glooshot-prometheus-server.glooshot.svc.cluster.local
  - click "Save and test"
- create dashboards
  - "+" -> "Create" -> "import"
  - paste from `./glooshot/dashboard.json`

### Deploy loop

```bash
kubectl apply -f res/loop.yaml
```

### install the apps

#### bookinfo
```bash
kubectl create ns bookinfo
kubectl label namespace bookinfo istio-injection=enabled
kubectl apply -f glooshot/bookinfo.yaml -n bookinfo
```
- configure it for the demo
  - port forward (as specified below)
  - visit url: http://localhost:9080/productpage?u=normal
  - before: page should rotate between versions of the reviews app (different star styles)
  - after: stars should always be red (vulnerable v4 of app in use)
```bash
supergloo apply routingrule trafficshifting \
    --namespace glooshot \
    --name reviews-vulnerable \
    --dest-upstreams glooshot.bookinfo-reviews-9080 \
    --target-mesh glooshot.istio-istio-system \
    --destination glooshot.bookinfo-reviews-v4-9080:1
```




## Port forwards
```bash
kubectl port-forward -n grafana deployment/kubecon-eu-grafana 3000
kubectl port-forward -n bookinfo deploy/productpage-v1 9080
kubectl port-forward -n loop-system deployment/loop 5678
```
