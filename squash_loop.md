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


### Deploy loop

```bash
kubectl apply -f res/loop.yaml
```

### install the apps

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
- apply the resources
```bash
kubectl apply -f loop/tap.yaml
kubectl apply -f loop/envoyfilter.yaml
```
- use loop
```bash
loopctl -h
loopctl list
# create some traffic, including 500s
loopctl list # note that only the 500s are captured
loopctl replay --id 1
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
