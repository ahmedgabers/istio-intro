# Setup Lab

### Download the Ebook (FREE)

[Introducing Istio Service Mesh for Microservices](https://developers.redhat.com/books/introducing-istio-service-mesh-microservices/)

### Clone the Repository

```
git clone https://github.com/ahmedgabers/istio-intro istio-intro
export TUTORIAL_HOME="$(pwd)/istio-intro"
cd $TUTORIAL_HOME
```

### Install CLI Tools

[kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/), [Minikube](https://minikube.sigs.k8s.io/docs/start/), [stern](https://github.com/wercker/stern), [Git](https://git-scm.com/downloads), and [kubectx and kubens](https://github.com/ahmetb/kubectx)


### Start Minikube

Linux:

```
minikube start --memory=4096 --cpus=2 --vm-driver=virtualbox -p istio
minikube profile istio
```

Windows:

```
minikube start --memory=4096 --cpus=2 --vm-driver=hyperv -p istio
minikube profile istio
```

### Istio Installation


Linux:

```
curl -L https://github.com/istio/istio/releases/download/1.6.13/istio-1.6.13-linux-amd64.tar.gz | tar xz
```

Windows:

```
curl -L https://github.com/istio/istio/releases/download/1.6.13/istio-1.6.13-win.zip
```

Both:

```
cd istio-1.6.13
export ISTIO_HOME=`pwd`
export PATH=$ISTIO_HOME/bin:$PATH

```

Install Istio:

```
istioctl manifest apply --set profile=demo --set values.global.proxy.privileged=true --set meshConfig.outboundTrafficPolicy.mode=ALLOW_ANY
```

```bash
✔ Istio core installed
✔ Istiod installed
✔ Egress gateways installed
✔ Ingress gateways installed
✔ Addons installed
✔ Installation complete
```

```
kubectl config set-context $(kubectl config current-context) --namespace=istio-system
kubectl get pods -w
```

```bash
NAME                                        READY     STATUS      RESTARTS   AGE
grafana-b54bb57b9-wdtw6                1/1     Running   0          79s
istio-egressgateway-68587b7b8b-qzz8x   1/1     Running   0          79s
istio-ingressgateway-55bdff67f-t97xd   1/1     Running   0          79s
istio-tracing-9dd6c4f7c-mvwnz          1/1     Running   0          79s
istiod-76bf8475c-cnkvn                 1/1     Running   0          101s
kiali-d45468dc4-kslcb                  1/1     Running   0          78s
prometheus-74d44d84db-j22tf            2/2     Running   0          78s
```

### Deploy Sample Application


```
kubectl create namespace tutorial
kubectl config set-context $(kubectl config current-context) --namespace=tutorial
istioctl version
```

Customer:

```
kubectl apply -f <(istioctl kube-inject -f customer/kubernetes/Deployment.yml) -n tutorial
kubectl create -f customer/kubernetes/Service.yml -n tutorial
kubectl create -f customer/kubernetes/Gateway.yml -n tutorial
```

Preference:

```
kubectl apply -f <(istioctl kube-inject -f preference/kubernetes/Deployment.yml)  -n tutorial
kubectl create -f preference/kubernetes/Service.yml -n tutorial

```

Recommendation:

```
kubectl apply -f <(istioctl kube-inject -f recommendation/kubernetes/Deployment.yml) -n tutorial
kubectl create -f recommendation/kubernetes/Service.yml -n tutorial
```

Patch Services to expose on NodePort:

```
kubectl patch service/istio-ingressgateway -p '{"spec":{"type":"NodePort"}}' -n istio-system
kubectl patch service/grafana -p '{"spec":{"type":"NodePort"}}' -n istio-system
kubectl patch service/jaeger-query -p '{"spec":{"type":"NodePort"}}' -n istio-system
kubectl patch service/prometheus -p '{"spec":{"type":"NodePort"}}' -n istio-system
kubectl patch service/kiali -p '{"spec":{"type":"NodePort"}}' -n istio-system
```

```
export INGRESS_HOST=$(minikube ip -p istio)
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
```

Validate:

```
kubectl get pods -n tutorial
kubectl get svc istio-ingressgateway -n istio-system
curl $GATEWAY_URL/customer
stern "customer-\w" -c customer
stern "preference-\w" -c preference
stern "recommendation-v1-\w" -c recommendation-v1
```

Generate Traffic:

In a separate terminal window

```
while true; do curl $GATEWAY_URL/customer; sleep 0.3;done;
```
