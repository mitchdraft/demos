
# Glooshot: Bookinfo

### Setup glooshot
```bash
# glooshot init
cd /Users/mitch/go/src/github.com/solo-io/glooshot
REGISTER_GLOOSHOT=1 ./_output/glooshot-darwin-amd64 register
## or: REGISTER_GLOOSHOT=1 go run cmd/cli/main.go register
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
## or: NdnA0y79uLGY1uwmAwZrq9Nm75jx8ufjGoms9eBn
```
- add data source:
  - Prometheus
  - URL: http://glooshot-prometheus-server.glooshot.svc.cluster.local
  - click "Save and test"
- create dashboards
  - "+" -> "Create" -> "import"
  - paste from `./glooshot/dashboard.json`

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
    --namespace bookinfo \
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

## Glooshot
### Outline
- review the service mesh bookinfo app
- apply a chaos experiment to the ratings service
- observe the reviews service fails (a cascading failure)
- note that the faults are removed after the experiment ends
- review the experiment results
- review the grafana logs
- deploy a more resilient version of the app
- apply a repeat experiment
- observe that the experiment passes, the cascading failure vulnerability has been fixed
- review the grafana logs, there are no 500s
### Steps
- apply experiment
```bash
kubectl apply -f glooshot/fault-abort-ratings.yaml
```
- reload the page a few times, observe the error
- wait for the experiment to fail, print out the crds for the exp and report
```bash
kubectl get exp -n bookinfo abort-ratings-metric -o yaml
kubectl get report -n bookinfo abort-ratings-metric -o yaml
```
- deploy a more resilient version of the service:
```bash
kubectl delete routingrule -n bookinfo reviews-vulnerable
supergloo apply routingrule trafficshifting \
    --namespace bookinfo \
    --name reviews-resilient \
    --dest-upstreams glooshot.bookinfo-reviews-9080 \
    --target-mesh glooshot.istio-istio-system \
    --destination glooshot.bookinfo-reviews-v3-9080:1
```
- reapply the experiment
```bash
kubectl apply -f glooshot/fault-abort-ratings-repeat.yaml
```
- reload the page for 30 seconds, notice that the error is contained, and the reviews service still renders
```bash
kubectl get exp -n bookinfo abort-ratings-metric-repeat -o yaml
kubectl get report -n bookinfo abort-ratings-metric-repeat -o yaml
```

# Restore to pre-demo state
- delete experiment and report crds
```bash
kubectl delete experiments -n bookinfo --all
kubectl delete reports -n bookinfo --all
kubectl delete routingrule -n bookinfo reviews-resilient
```
- restore vulnerable service route and delete resilient service route
```bash
supergloo apply routingrule trafficshifting \
    --namespace bookinfo \
    --name reviews-vulnerable \
    --dest-upstreams glooshot.bookinfo-reviews-9080 \
    --target-mesh glooshot.istio-istio-system \
    --destination glooshot.bookinfo-reviews-v4-9080:1
```
