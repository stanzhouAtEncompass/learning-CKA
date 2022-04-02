# Other
## JSON Path
is a query language
- DATA
``` json
{
   "car": {
      "color": "blue",
      "price": "$20,000"
   }
}
```

- Query
Get car details
car

- Result
``` json
{
  "color": "blue",
  "price": "$20,000"
}
```
### JSON PATH-Dictionaries
![Dic](https://i.imgur.com/Y8vGqwp.png)

### JSON PATH-Lists
DATA:

``` json
[
 "car",
 "bus",
 "truck",
 "bike"
]
```
Query:
Get the 1st element
$[0]

Get the 1st and 4th element
$[0,3]

Result:
[ "car" ]
[ "car", "bike" ]

### JSON PATH-Dictionary & Lists
DATA:
``` json
{
  "car": {
     "color": "blue"
     "price": "$20,000"
     "wheels": [
       {
          "model": "X345ERT",
          "location": "front-right"
       },
       {
          "model": "X346GRX",
          "location": "front-right"
       },
       {
          "model": "X236DEM",
          "location": "rear-right"
       },
       {
          "model": "X987XMV",
          "location": "rear-right"
       },
     ]
  }
}
```
QUERY: #$ is root element
Get the model of the 2nd wheel
`$.car.wheels[1].model`
or
`$.car.wheels[?(@.location == "rear-right")].model`
Result:
X346GRX

### JSON PATH-Criteria
DATA:

``` json
[
 12,
 43,
 23,
 12,
 56,
 43,
 93,
 32,
 45,
 63,
 27,
 8,
 78
]
```
QUERY: list all number > 40
Get all numbers greater than 40
`$[ Check if each item in the array > 40 ]`
   Check if => ? ()
`$[?( each each item in the array > 40 )]`
   each iteam in the list => @
`$[?( @ > 40 )]`
@ == 40   @ in [40,43,45]
@ != 40   @ nin [40,43,45]



RESULT:
``` json
[
 43,
 56,
 43,
 93,
 45,
 63,
 78
]
```
## Wild card
DATA:
``` json
{
  "car": {
     "color": "blue"
     "price": "$20,000"
   },
  "bus": {
     "color": "white"
     "price": "$120,000"
   }
}
QUERY:
Get all colors
`$.*.color`
Get all prices
`$.*.price`
RESULT:
[ "blue", "white" ]

---
DATA:
``` json
[
  {
     "model": "X345ERT",
     "location": "front-right"
  },
  {
     "model": "X346GRX",
     "location": "front-right"
  },
  {
     "model": "X236DEM",
     "location": "rear-right"
  },
  {
     "model": "X987XMV",
     "location": "rear-right"
  }
]
```
QUERY:
Get 1st wheel's model
`$[0].model`
Get 4st wheel's model
`$[3].model`
Get all wheel's model
`$[*].model`
---
DATA:
![Data](https://i.imgur.com/5wBzVjT.png "Data")
QUERY:
Get car's 1st wheel model
`$.car.wheels[0].model`
Get car's all wheel model
`$.car.wheels[*].model`
Get bus's wheel models
`$.bus.wheels[*].model`
Get all wheels' models
`$.*.wheels[*].model`
Result:
![result](https://i.imgur.com/uauISKe.png "result")

## List
QUERY:
``` jason
$[0:8]
START:END
$[0:8:2]
START:END:STEP
# Get the last element
$[-1:0]
$[-1:]
# Get the last 3 elements
$[-3:]

```
## Why json path?
- Large data sets
  - 100s of Nodes
  - 1000s of PODs, Deployments, ReplicaSets
## Kubectl
`kubectl get nodes`
kubectl talk to kube-apiserver, kube-apiserver speaks json language, it return the result with json format.
`k get nodes -o wide` # show more details
## How to JSON path in kubectl?
1. Indentify the kubectl command
2. Familarize with JSON output

``` json
k get nodes -o json
k get pods -o json
```
3. Form the JSON PATH query
`.items[0].spec.containers[0].image`
4. Use the JSON PATH query with kubectl command
`k get pods -o=jsonpath='{ .items[0].spec.containers[0].image }`
![JSON Path example](https://i.imgur.com/daHu8ey.png "examples")
## Loops -Range
`k get nodes -o=jsonpath='{.items[*].metadata.name{"\n"}{.items[*].status.capacity.cpu}'`
FOR EACH NODE
   PRINT NODE NAME \t PRINT CPU COUNT \n
END FOR
``` json
'{range .items[*]}
   {.metadata.name}{"\t"} {.status.capacity.cpu}{"\n"}
{end}'
```

`k get nodes -o=custom-colums=<COLUMN NAME>:<JSON PATH>`

``` shell
k get nodes -o=custom-colums=NODE:.metadata.name,CPU:.status.capacity.cpu
NODE     CPU
master    4
node01    4
```
## JSON PATH for Sort
```
k get nodes -o=custom-colums=NODE:.metadata.name, CPU=.status.capacity.cpu
NODE   CPU
master  4
node01  4
k get nodes --sort-by= .metadata.name
k get nodes --sort-by= .status.capacity.cpu
```
## LAB
The order of container could be different. The redis-container need not be the second container all the time. So, re-develop a JSON PATH query to get the restart count of redis-container, but this time use a criteria/condition to get the restart count of the container with the container name redis-container .

Scroll down if you can't see the status section

Source Data:

``` json
{
  "apiVersion": "v1",
  "kind": "Pod",
  "metadata": {
    "name": "nginx-pod",
    "namespace": "default"
  },
  "spec": {
    "containers": [
      {
        "image": "nginx:alpine",
        "name": "nginx"
      },
      {
        "image": "redis:alpine",
        "name": "redis-container"
      }
    ],
    "nodeName": "node01"
  },
  "status": {
    "conditions": [
      {
        "lastProbeTime": null,
        "lastTransitionTime": "2019-06-13T05:34:09Z",
        "status": "True",
        "type": "Initialized"
      },
      {
        "lastProbeTime": null,
        "lastTransitionTime": "2019-06-13T05:34:09Z",
        "status": "True",
        "type": "PodScheduled"
      }
    ],
    "containerStatuses": [
      {
        "image": "nginx:alpine",
        "name": "nginx",
        "ready": false,
        "restartCount": 4,
        "state": {
          "waiting": {
            "reason": "ContainerCreating"
          }
        }
      },
      {
        "image": "redis:alpine",
        "name": "redis-container",
        "ready": false,
        "restartCount": 2,
        "state": {
          "waiting": {
            "reason": "ContainerCreating"
          }
        }
      }
    ],
    "hostIP": "172.17.0.75",
    "phase": "Pending",
    "qosClass": "BestEffort",
    "startTime": "2019-06-13T05:34:09Z"
  }
}
```
Expected Output:

``` json
[
  2
]
```

Answer:
`$.status.containerStatuses[?(@.name == 'redis-container')].restartCount**

**advanced kubectl commands**
Use a JSON PATH query to identify the context configured for the aws-user in the my-kube-config context file and store the result in /opt/outputs/aws-context-name.
`kubectl config view --kubeconfig=my-kube-config -o jsonpath="{.contexts[?(@.context.user=='aws-user')].name}" > /opt/outputs/aws-context-name`
