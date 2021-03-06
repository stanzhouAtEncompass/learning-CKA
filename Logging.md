# Section 4: Logging & Monitoring
## Monitor
- Mertics server
```
# for minikube
minikube addons enable metrics-server
# others
git clone https://github.com/kubernetes-incubator/metrics-serve
k create -f deploy/1.8+/
```
### view
```
k top node
k top pod
```
- Prometheus
- Elastic Stack
- Datadog
- dynatrace

## Logs - Docker
```
docker run -d kodekloud/event-simulator
docker logs -f ecf
```

## Logs - Kubernetes
```
k create -f event-simulator.yaml
#event-simulator.yaml
apiVersion: v1
kind: Pod
metadata:
 name: event-simulator-pod
spec:
 containers:
 - name: event-simulator
   image: kodekloud/event-simulator
 - name: image-processor
   image: some-image-processor
   
k logs -f event-simulator-pod event-simulator
```
