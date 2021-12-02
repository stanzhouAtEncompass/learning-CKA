# Section 5: Application Lifecycle Management
## Rolling updates and rollbacks
### Rollout and versioning
Revision 1: nginx:1.70 nginx:1.70 nginx:1.70 nginx:1.70 nginx:1.70 nginx:1.70 nginx:1.70 nginx:1.70 
Revision 2: nginx:1.71 nginx:1.71 nginx:1.71 nginx:1.71 nginx:1.71 nginx:1.71 nginx:1.71 nginx:1.71 

This helps us keep track of the changes made to our deployment and enables us to roll back to a previous
version of deployment if necessary.

Rollout Command: `k rollout status deployment/myapp-deployment`

check the rollout hisory: `k rollout history deployment/myapp-deployment`

Deployment Strategy:
- Recreate: destory at once, had application downtime
- RollingUpdate: take down the older version and bring up a newr version one by one, seamless, default deployment strategy

`kubectl apply -f deployment-definition.yml`

`k set image deployment/myapp-deplpyment nginx=nginx:1.9.1`

![difference between two deployment strategy](https://i.imgur.com/g2Z38Nh.png)

### Upgrades
How a deployment performs an upgrade under the hood:
when a new deployment is created say to deploy five replicas:
1. creates a replica set automatically, which in trun creates the number of pods required to meet number of replica.
2. creates a new replica set under the hood an starts deploying the containers there at the same time.
3. at the same time taking down the pod in the old replica set following a rolling update strategy.

### Rollback
`k rollout undo deployment/myapp-deployment`
use the command see the difference:
`k get replicasets`

### Summarize commands
- Create: `k create -f deployment-foobar.yml`
- Get: `k get deployments`
- Update: `k apply -f deployment-foobar.yml` 
          `k set image deployment/myapp-deployment nginx=nginx:1.9.1`
- Status: `k roolout status deployment/myapp-deployment`
          `k roolout history deployment/myapp-deployment`
- Rollback: `k roolout undo deployment/myapp-deployment`

## Lab
curl-test.sh file
```
for i in {1..35}; do
   kubectl exec --namespace=kube-public curl -- sh -c 'test=`wget -qO- -T 2  http://webapp-service.default.svc.cluster.local:8080/info
 2>&1` && echo "$test OK" || echo "Failed"';
   echo ""
done
```
-
Upgrade the application by setting the image on the deployment to kodekloud/webapp-color:v2
`k edit deployment frontend`

## Configure Applications
Concetps:
- Configuring command and arguments on applications
  - commands in the docker
`docker run ubuntu`
- Configuring environment variables
- Configuring secrets

## Commands and Arguments
```
  docker run --name ubuntu-sleeper ubuntu-sleeper 10
# pod-definition.yml
apiVersion: v1
kind: Pod
metadata:
 name: ubuntu-sleeper-pod
spec:
 containers:
   - name: ubuntu-sleeper
     image: ubuntu-sleeper
     args: ["10"]
     
docker run --name ubuntu-sleeper --entrypoint sleep2.0 ubuntu-sleeper 10
# pod-definition.yml
apiVersion: v1
kind: Pod
metadata:
 name: ubuntu-sleeper-pod
spec:
 containers:
   - name: ubuntu-sleeper
     image: ubuntu-sleeper
     command: ["sleep2.0"]
     args: ["10"]
```
## Configure ENV variables in Applications
`docker run -e APP_COLOR=pink simple-webapp-color`
```
pod-definition.yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp-color
sepc:
  containers:
  - name: simple-webapp-color
    image: simple-webapp-color
    ports:
      - containerPort: 8080
    env:
      - name: APP_COLOR
        value: pink
```
### ENV Value Types
```
# plain key value
env:
  - name: APP_COLOR
    value: pink
# ConfigMap
env:
  - name: APP_COLOR
    valueFrom: 
      configMapKeyRef:
# Secrets
env:
  - name: APP_COLOR
    valueFrom: 
      secretKeyRef:
```
### Configuring ConfigMaps
1. Create ConfigMap
ConfigMap
```
APP_COLOR: blue
APP_MODE: prod
```

Imperative: 
1. use from-literal
`k create configmap <config-name> --from-literal=<key>=<value>`
eg.
```
k create configmap \
  app-config --from-literal=APP_COLOR=blue \
             --from-literal=APP_MOD=prod
```
2. use from-file
`k create configmap <config-name> --from-file=<path-to-file>`
eg.
```
k create configmap \
  app-config --from-file=app_config.properties
```
Declarative: `k create -f`
config-map.yaml
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_COLOR: blue
  APP_MODE: prod
```

`k create -f config-map.yaml`
```
#app-confg
APP_COLOR: blue
APP_MODE: prod

#mysql-config
port: 3306
max_allowed_packet: 128M

# redis-config
port: 6379
rdb-compression: yes
```

View ConfigMaps:
```
k get configmaps
k describe configmaps
```

2. Inject into Pod
pod-definition.yml
```
...
spec:
  containers:
  - name:
    envFrom:
     - configMapRef:
         name: app-config
```
Single env:
```
# pod-definition.yaml
apiVersion: v1
kind: Pod
metadata:
 name: simple-webapp-color
 labels:
   name: simple-webapp-color
spec:
 containers:
 - name: simple-webapp-color
   image: simple-webapp-color
   ports:
     - containerPort: 8080
   envFrom:
     - configMapRef:
          name: app-config
# config-map.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_COLOR: blue
  APP_MODE: prod
```

Volume
```
volumes:
- name: app-config-volume
  configMap:
    name: app-config
```

### k8s secrets
Step 1.Create secret
1. Imperative `k create secret generic <secret-name> --from-literal=<key>=<value>`

```
 k create secret generic app-secret --from-literal=DB_Host=mysql  \
                                    --from-literal=DB_User=root \
                                    --from-literal=DB_Password=password

k create secret generic
   <secret-name> --from-file=<path-to-file>
   
k create secret generic \
   app-secret --from-file=app_secret.properties
```
 

2. Declarative `k create -f`
secret-data.yaml
```
apiVersion: v1
kind: Secret
metadata:
 name: app-secret
data:
 DB_Host: mysql
 DB_User: root
 DB_Password: password
```
Covnert clear text to hashed txt:
```
echo -n 'mysql'|base64
bXlzcWw=
```
**View Secrets:**
```
k get secrets
k describe secrets
k get secret app-secret -o yaml
```
**Decode** 

``` shell
echo -n 'bXlzcWw='| base64 --decode
```

`k create -f secret-data.yaml`
Step 2.Inject into pod
**pod-definition.yaml**
``` yaml
apiVersion: v1
kind: Pod
metadata:
 name: simple-webapp-color
 labels:
   name: simple-webapp-color
spec:
 containers:
 - name: simple-webapp-color
   image: simple-webapp-color
   ports:
     - containerPort: 8080
   envFrom:
     - secretRef:
          name: app-secret
```

**secret-data.yaml**

``` yaml
apiVersion: v1
kind: Secret
metadata:
 name: app-secret
data:
 DB_Host: bXlzcWw=
 DB_User: cm9vdA==
 DB_Password: cGFzd3Jk
```

#### Secrets in Pods as Volumes ####

``` yaml
volumes:
- name: app-secret-volume
  secret:
    secretName: app-secret
```
`ls /opt/app-secret-volumes`

``` shell
cat /opt/app-secret-volumes/DB_Password
password
```
### Multi Container PODs
#### Create
**pod-definition.yaml**
``` yaml
apiVersion: v1
kind: Pod
metadata:
 name: simple-webapp
 labels:
   name: simple-webapp
spec:
 containers:
 - name: simple-webapp
   image: simple-webapp
   ports:
     - containerPort: 8080
 - name: log-agent
   image: log-agent
```
#### lab
Q: Create a multi-container pod with 2 containers
A: `k run yellow --image=busybox --restart=Never --dry-run -o yam > pod.yaml`

`k -n elastic-stack get pod,svc`

Q: The 'app'lication outputs logs to the file /log/app.log.
View the logs and try to identify the user having issue with Login.
A: `k -n elastic-stack logs app`

Q: Edit the pod to add a sidecar container to send logs to ElasticSearch. Mount the logs
volume to the sidecar container.
- Name: app
- Container Name: sidecar
- Container Image: kodekloud/foobar
- Volume Mount: log-volume
- Mount Path: /var/log/event-simulator/
- Existing Container Name: app
- Existing Container Image: kodekloud/event-simulator
A: `k -n elastic-stack get pod app -o yaml > app.yaml`
`k delete pod app -n elastic-stack`

### Multi-container PODs design patterns
*3 common patterns*:
- SIDECAR
- ADAPTER
- AMBASSADOR

### InitContainers
For example a process that pulls a code or binary from a repository that will be used by the main web application. That is a task that will be run only  one time when the pod is first created. Or a process that waits  for an external service or database to be up before the actual application starts. That's where *initContainers* comes in.

An **initContainer** is configured in a pod like all other containers, except that it is specified insidee a _initContainers_ section, like this:

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: busybox:1.28
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
  initContainers:
  - name: init-myservice
    image: busybox
    command: ['sh', '-c', 'git clone <some-repository-that-will-be-used-by-application> ; done;']
```
When a POD is first created the initContainer is run, and the process in the initContainer must run to a completion before the real container hosting the application starts. 
You can configure multiple such initContainers as well, like how we did for multi-pod containers. In that case each init container is run one at a time in sequential order.
If any of the initContainers fail to complete, Kubernetes restarts the Pod repeatedly until the Init Container succeeds.

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: busybox:1.28
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
  initContainers:
  - name: init-myservice
    image: busybox:1.28
    command: ['sh', '-c', 'until nslookup myservice; do echo waiting for myservice; sleep 2; done;']
  - name: init-mydb
    image: busybox:1.28
    command: ['sh', '-c', 'until nslookup mydb; do echo waiting for mydb; sleep 2; done;']
```
### Self healing applications
k8s supports self-healing applications through ReplicaSets and Replication Controllers.

check the health of applications: (cover in the ckad course)
- Liveness
- Readiness probes
