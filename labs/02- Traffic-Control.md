# Simple Routing Rules

### Deploy V2

```
kubectl apply -f <(istioctl kube-inject -f recommendation/kubernetes/Deployment-v2.yml) -n tutorial
kubectl get pods -w -n tutorial
```

### Generate Traffic in a separate terminal window

```
while true; do curl $GATEWAY_URL/customer; sleep 0.3;done;
```

### Scale Deployment

```
kubectl scale --replicas=2 deployment/recommendation-v2 -n tutorial
```

# Blue/Green Deployment

### All users to recommendation:v2

```
kubectl create -f istiofiles/destination-rule-recommendation-v1-v2.yml -n tutorial
kubectl create -f istiofiles/virtual-service-recommendation-v2.yml -n tutorial
```

Watch traffic logs and notice v2 being returned.


### All users to recommendation:v1

```
kubectl replace -f istiofiles/virtual-service-recommendation-v1.yml -n tutorial
kubectl get virtualservice -n tutorial
kubectl get virtualservice -o yaml -n tutorial
```
### All users to recommendation v1 and v2

```
kubectl delete -f istiofiles/virtual-service-recommendation-v1.yml -n tutorial
```

# Canary deployment

```
kubectl get pods -l app=recommendation -n tutorial
```

Create the virtualservice that will send 90% of requests to v1 and 10% to v2
```
kubectl create -f istiofiles/virtual-service-recommendation-v1_and_v2.yml -n tutorial
```

In another terminal, change the mixture to be 75/25
```
kubectl replace -f istiofiles/virtual-service-recommendation-v1_and_v2_75_25.yml -n tutorial
```

Clean up
```
kubectl delete -f istiofiles/virtual-service-recommendation-v1_and_v2_75_25.yml -n tutorial
kubectl delete -f istiofiles/destination-rule-recommendation-v1-v2.yml -n tutorial
```
