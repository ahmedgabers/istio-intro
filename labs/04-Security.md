# Egress Blocking

Let's deploy V3

```
kubectl apply -f <(istioctl kube-inject -f recommendation/kubernetes/Deployment-v3.yml) -n tutorial
kubectl get pods -w -n tutorial
```

Configure Istio to allow only registered traffic

```
kubectl get configmap istio -n istio-system -o yaml | sed 's/mode: ALLOW_ANY/mode: REGISTRY_ONLY/g' | kubectl replace -n istio-system -f -
```

```
kubectl create -f istiofiles/destination-rule-recommendation-v1-v2-v3.yml -n tutorial
kubectl create -f istiofiles/virtual-service-recommendation-v3.yml

curl $GATEWAY_URL/customer
```

Service will return a 500 error

```
customer => Error: 503 - preference => Error: 500 - <html><head><title>Error</title></head><body>Internal Server Error</body></html>
```

Let’s fix it by registering a service entry to allow access to ```worldclockapi``` only

```
kubectl create -f istiofiles/service-entry-egress-worldclockapi.yml -n tutorial
kubectl get serviceentry
curl $GATEWAY_URL/customer
```

Shell into the pod

```
kubectl exec -it -n tutorial $(kubectl get pods -n tutorial -o jsonpath="{.items[*].metadata.name}" -l app=recommendation,version=v3) -c recommendation /bin/bash
curl http://worldclockapi.com/api/json/cet/now

exit
```

#### Clean up

```
kubectl delete -f istiofiles/service-entry-egress-worldclockapi.yml -n tutorial
kubectl delete -f istiofiles/destination-rule-recommendation-v1-v2-v3.yml -n tutorial
kubectl delete -f istiofiles/virtual-service-recommendation-v3.yml
kubectl delete all -n tutorial -l app=recommendation,version=v3
kubectl get configmap istio -n istio-system -o yaml | sed 's/mode: REGISTRY_ONLY/mode: ALLOW_ANY/g' | kubectl replace -n istio-system -f -
```

# mTLS

mTLS is enabled by default in permissive mode

To check if mTLS is enabled or not, open up Kiali

```
istioctl dashboard kiali
```

In the Graph section, under Display, toggle on Security and check if the lock sign appears on the arrows

Check the mTLS by sniffing traffic between services

```
CUSTOMER_POD=$(kubectl get pod | grep cust | awk '{ print $1}' ) 
kubectl exec -it $CUSTOMER_POD -c istio-proxy /bin/bash 
```

```
ifconfig 
sudo tcpdump -vvvv -A -i eth0 '((dst port 8080) and (net 172.17.0.X))' 
```
Change the ip address to the Pod's IP

Generate traffic in a separate terminal window

```
while true; do curl $GATEWAY_URL/customer; sleep 0.3;done;
```

Traffic should appear encrypted

Now, let’s disable TLS

```
kubectl apply -f istiofiles/disable-mtls.yml
```

And again check tcpdump output

Now traffic is displayed unencrypted

On Kiali, the lock sign should disappear as well

#### Clean up

```
kubectl apply -f istiofiles/enable-mtls.yml
```

# Access Control

We’ll create an authorization path that will only allow the following communication path: customer → preference → recommendation

Any other path will result to a 403 forbidden HTTP error

#### Denny All

```
kubectl apply -f istiofiles/authorization-policy-deny-all.yaml -n tutorial
curl $GATEWAY_URL/customer
```

```
RBAC: access denied
```

#### Allow Customer

Let’s permit the interaction to customer service

```
kubectl apply -f istiofiles/authorization-policy-allow-customer.yaml -n tutorial
curl $GATEWAY_URL/customer
```

```
customer => Error: 403 - RBAC: access denied
```

Now customer is reached but not preference

#### Allow Preference

```
kubectl apply -f istiofiles/authorization-policy-allow-preference.yaml -n tutorial
curl $GATEWAY_URL/customer
```

```
customer => Error: 503 - preference => Error: 403 - RBAC: access denied
```
The preference service is accessed but not recommendation

#### Allow Recommendation

Let’s end up by allowing all the path

```
kubectl apply -f istiofiles/authorization-policy-allow-recommendation.yaml -n tutorial
curl $GATEWAY_URL/customer
```

Succeed!

#### Validate Other Paths

Now let’s assume that someone gets access to recommendation service

```
kubectl exec -it -n tutorial $(kubectl get pods -n tutorial|grep recommendation|awk '{ print $1 }'|head -1) -c recommendation /bin/bash
```

```
curl preference:8080
RBAC: access denied
exit
```

So you can see that preference service can only be accessed by customer service and not any other service such as recommendation

#### Clean Up

```
./scripts/clean.sh
```
