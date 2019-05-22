

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

#### Calculator app
```bash
kubectl create ns calc
kubectl label namespace calc istio-injection=enabled
kubectl apply -f loop/deploy_calc.yaml -n calc
```
- port forward (as specified below)
  - visit url: http://localhost:8080
  - expect: calculator to do the opposite of the specified operation




## Port forwards
```bash
kubectl port-forward -n grafana deployment/kubecon-eu-grafana 3000
kubectl port-forward -n bookinfo deploy/productpage-v1 9080
k port-forward -n calc deploy/example-service1 8080
kubectl port-forward -n loop-system deployment/loop 5678
```

# Demos

## Squash
### Outline
- observe a bug in a go-java microservice app
- attach debuggers to each of the running processes
- determine the bug

### Initial condition expectations
- example-service2 image is example-service2-java:v0.2.2

### Steps
- open a (go) debugger on service 1
- open a (java) debugger on service 2
- set breakpoints on each
- run a calculation, follow the code execution
- patch the new image
```bash
kubectl patch deployment -n calc example-service2 -p '{"spec":{"template":{"spec":{"containers":[{"name":"example-service2","image":"soloio/example-service2ise:0.1.0"}]}}}}'
```
- verify that the addition/subtraction bug has gone away

## Loop
### Outline
- replace the broken service with a "fixed" version
- observe that the app has a new bug
- create a tap to capture traffic with a 500 response code
- attach squash to the "sandboxed" process
- replay the traffic to the "sandboxed" service
- determine the bug
### Initial condition expectations
- example-service2 image is example-service2ise:v0.1.1
- there are no envoyconfig or tap crds
- `loopctl list` returns an empty list
### Steps
```bash
loopctl -h
loopctl list
# create some traffic, including 500s
loopctl list # note that only the 500s are captured
loopctl replay --id 1
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
supergloo apply routingrule trafficshifting \
    --namespace glooshot \
    --name reviews-resilient \
    --dest-upstreams glooshot.bookinfo-reviews-9080 \
    --target-mesh glooshot.istio-istio-system \
    --destination glooshot.bookinfo-reviews-v3-9080:1
```
- reload the page for 30 seconds, notice that the error is contained, and the reviews service still renders
```bash
kubectl get exp -n bookinfo abort-ratings-metric-repeat -o yaml
kubectl get report -n bookinfo abort-ratings-metric-repeat -o yaml
```


# Restore to pre-demo state
## Squash demo portion
- patch back the old image
```bash
kubectl patch deployment -n calc example-service2 -p '{"spec":{"template":{"spec":{"containers":[{"name":"example-service2","image":"soloio/example-service2-java:v0.2.2"}]}}}}'
```
- delete any lingering squash plank pods
```bash
kubectl delete pods -n squash-debugger --all
```

## Loop demo portion
- remove envoyfilter and tap crds
```bash
kubectl delete envoyfilters -n loop-system --all
kubectl delete taps -n loop-system --all
```
- clear out the records, deleting the pod will do
```bash
kubectl delete pods -n loop-system --all
```
- verify that list is empty
```bash
loopctl list
```

## Glooshot demo portion
- restore vulnerable service route and delete resilient service route
```bash
kubectl delete routingrule -n bookinfo reviews-resilient
supergloo apply routingrule trafficshifting \
    --namespace glooshot \
    --name reviews-vulnerable \
    --dest-upstreams glooshot.bookinfo-reviews-9080 \
    --target-mesh glooshot.istio-istio-system \
    --destination glooshot.bookinfo-reviews-v4-9080:1
```
- delete experiment and report crds
```bash
kubectl delete experiments -n glooshot --all
kubectl delete reports -n glooshot --all
```




## Notes
- Loop's custom envoy seems to be interferring with supergloo's ability to translate faults into istio resources. The effect of the vulnerability experiment was not shown when I had loop set up and running in the same cluster.
