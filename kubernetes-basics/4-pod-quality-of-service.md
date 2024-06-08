##
### Quality of service classes
##
1. Guaranteed - pods are guaranteed that have stricted resource limits & are least likely to face eviction.They are guaranteed not to be killed until they exceed their limits or there are no lower-priority Pods that can be preempted from the Node. 
```
apiVersion: v1
kind: Pod
metadata:
  name: nginx-guaranteed
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      requests:
        memory: "256Mi"
        cpu: "500m"
      limits:
        memory: "256Mi"
        cpu: "500m"
```
verify by describing pod

`QoS Class:                   Guaranteed`




2. Burstable - Pods that are Burstable have some lower bound rresource guarantees based on the request, but do not require a specific limit.(aleast request or limit-memory/cpu)
```
apiVersion: v1
kind: Pod
metadata:
  name: nginx-burstable
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      requests:
        memory: "256Mi"
      limits:
        memory: "512Mi"
```
`QoS Class:                   Burstable`


3. Pods in the BestEffort QoS class can use node resources that aren't specifically assigned to Pods in other QoS classes. The kubelet prefers to evict BestEffort Pods if the node comes under resource pressure.
```
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: mysql
  name: mysql
spec:
  containers:
  - image: mysql
    name: mysql
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```
`QoS Class:                   BestEffort`



##
### Limit Range
##

Define the max-min limit/request of memory & cpu, below example shows the limit specified for `type: Pod` & `type: Container`. if nothing is specified it will pick up the default.

```
apiVersion: v1
kind: LimitRange
metadata:
  name: example-limitrange
  namespace: default
spec:
  limits:
  - type: Pod
    max:
      cpu: "2"
      memory: "1Gi"
    min:
      cpu: "200m"
      memory: "100Mi"
  - type: Container
    max:
      cpu: "1"
      memory: "500Mi"
    min:
      cpu: "100m"
      memory: "50Mi"
    default:
      cpu: "300m"
      memory: "200Mi"
    defaultRequest:
      cpu: "200m"
      memory: "100Mi"
```

Under specified cpu example:

```
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: nginx
  name: nginx
spec:
  containers:
  - image: nginx
    name: nginx
    resources:
       requests:
          cpu: "30m"
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}

```




ERROR: 
`Error from server (Forbidden): error when creating "test.yml": pods "nginx" is forbidden: [minimum cpu usage per Pod is 200m, but request is 30m, minimum cpu usage per Container is 100m, but request is 30m]`


##
### PodDisruptionBudget
##
Pod disruption budget means you can set the min number of pods available or max number of pod unavailability using the label. this helps during the rolling update, and restrict nodes to evict the pods.(example we can not drain the nodes fully this makes sure application will be accessible to the users all time.

```
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: nginx-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: nginx
```
drain node command:
```
k drain node01 --ignore-daemonsets
```

ERROR: `error when evicting pods/"nginx-bf5d5cf98-h9b2h" -n "default" (will retry after 5s): Cannot evict pod as it would violate the pod's disruption budget.`
