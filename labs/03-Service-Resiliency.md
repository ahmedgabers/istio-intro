# Retry

We will make pod recommendation-v2 fail 100% of the time

```
kubectl exec -it -n tutorial $(kubectl get pods -n tutorial|grep recommendation-v2|awk '{ print $1 }'|head -1) -c recommendation /bin/bash
```

```
curl localhost:8080/misbehave
exit
```

This is a special endpoint that will make our application return only `503`s

Generate traffic in a separate terminal window:

```
while true; do curl $GATEWAY_URL/customer; sleep 0.3;done;
```

Open Kiali and notice that v2 receives requests, but that failing request is never returned to the user 

```
istioctl dashboard kiali
```

Now, make the pod v2 behave well again

```
kubectl exec -it -n tutorial $(kubectl get pods -n tutorial|grep recommendation-v2|awk '{ print $1 }'|head -1) -c recommendation /bin/bash
```

```
curl localhost:8080/behave
exit
```

Check traffic

# Timeout

Lets introduce some wait time in recommendation v2 by making it a slow performer with a 3 second delay by running the command

```
kubectl patch deployment recommendation-v2 -p '{"spec":{"template":{"spec":{"containers":[{"name":"recommendation", "image":"quay.io/rhdevelopers/istio-tutorial-recommendation:v2-timeout"}]}}}}' -n tutorial
```

Add the timeout rule

```
kubectl create -f istiofiles/virtual-service-recommendation-timeout.yml -n tutorial
```

You will see it return v1 after waiting about 1 second. You don’t see v2 anymore

Clean up

```
kubectl patch deployment recommendation-v2 -p '{"spec":{"template":{"spec":{"containers":[{"name":"recommendation", "image":"quay.io/rhdevelopers/istio-tutorial-recommendation:v2"}]}}}}' -n tutorial
kubectl delete -f istiofiles/virtual-service-recommendation-timeout.yml -n tutorial
```

# Circuit Breaking

Load test without circuit breaker using siege. We’ll have 10 clients sending 4 concurrent requests each

```
siege -r 10 -c 4 -v $GATEWAY_URL/customer
```

We will make pod recommendation-v2 fail 100% of the time

```
kubectl exec -it -n tutorial $(kubectl get pods -n tutorial|grep recommendation-v2|awk '{ print $1 }'|head -1) -c recommendation /bin/bash
```
```
curl localhost:8080/misbehave
exit
```

Check the logs using stern or ```kubectl logs```

```
stern 'recommendation-v2-\w' -c recommendation
```

Scale up the recommendation v2 service to two instances

```
kubectl scale deployment recommendation-v2 --replicas=2 -n tutorial
```

Now, you’ve got one instance of recommendation-v2 that is misbehaving and another one that is working correctly. Let’s redirect all traffic to recommendation-v2

```
kubectl create -f istiofiles/destination-rule-recommendation-v1-v2.yml -n tutorial
kubectl create -f istiofiles/virtual-service-recommendation-v2.yml -n tutorial
```

Load test and check the logs

```
siege -r 10 -c 4 -v $GATEWAY_URL/customer
stern 'recommendation-v2-\w' -c recommendation
```

You will notice the requests are still able to reach the failing service, eventhough automatic retries are working as expected

This is where the Circuit Breaker comes into the scene

```
kubectl replace -f istiofiles/destination-rule-recommendation_cb_policy_version_v2.yml -n tutorial
siege -r 10 -c 4 -v $GATEWAY_URL/customer
stern 'recommendation-v2-\w' -c recommendation
```

You should be able to see requests no longer reaching the failing pod

# Clean up

```
kubectl delete -f istiofiles/destination-rule-recommendation_cb_policy_version_v2.yml -n tutorial
kubectl delete -f istiofiles/virtual-service-recommendation-v2.yml -n tutorial
kubectl scale deployment recommendation-v2 --replicas=1 -n tutorial
kubectl delete pod -l app=recommendation,version=v2
```
