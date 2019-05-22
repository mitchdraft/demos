### if an old rr exists, remove it
```bash
kubectl delete routingrule -n glooshot reviews-resilient
```

### route to the vulnerable app
```bash
supergloo apply routingrule trafficshifting \
    --namespace glooshot \
    --name reviews-vulnerable \
    --dest-upstreams glooshot.default-reviews-9080 \
    --target-mesh glooshot.istio-istio-system \
    --destination glooshot.default-reviews-v4-9080:1
```

### apply the experiment
- review the page
```bash
kubectl apply -f fault-abort-ratings.yaml
```
- produce traffic until the experiment terminates
### inspect the results
```bash
k get results abort-ratings-metric -o yaml 
```

## Let's deploy a more resilient app and repeat the experiment
```bash
kubectl get pod -l app=reviews
kubectl delete routingrule -n glooshot reviews-vulnerable
supergloo apply routingrule trafficshifting \
    --namespace glooshot \
    --name reviews-resilient \
    --dest-upstreams glooshot.default-reviews-9080 \
    --target-mesh glooshot.istio-istio-system \
    --destination glooshot.default-reviews-v3-9080:1
```

### apply the experiment again with the more resilient app
- review the page
```bash
kubectl apply -f fault-abort-ratings-repeat.yaml
```
- produce traffic until the experiment terminates
### inspect the results for the more resilient app
```bash
k get results abort-ratings-metric-repeat -o yaml 
```
