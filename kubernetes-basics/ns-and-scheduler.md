##
### Namespace
##

Namespace in kubernetes helps us to group the resource. namespace can group the resources of two nodes also.
create namespace
```
k create ns dev
k create ns app
```
Create deployments in above namespaces
```
k create deploy demo --image nginx -n dev
k create deploy demo --image nginx -n app
```
get deployments
```
k get deploy # this will not return anything as we have nothing create in default namespace
```
get deployment in app namespace
```
k get deploy -n app
```
change the context
```
kubectl config set-context --current --namespace=app
```
now to get the pods of app namespace
```
kubectl get pods
```
##
### Resource Quota
##
Allocate resource quota to particular namespace


create namespace
```
kubectl create namespace uat-namepspace
```
resource quota
```
apiVersion: v1
kind: ResourceQuota
metadata:
  name: demo-quota
  namespace: uat-namespace
spec:
  hard:
    requests.cpu: "500m" # total amount of CPU that can be requested
    requests.memory: "200Gi" # total amount of memory that can be requested
    limits.cpu: "1" # total amount of CPU limit across all pods
    limits.memory: 400Gi # total amount of memory limit across all pods
    pods: "10" # total number of pods that can be created
```
Get the quota allocated
```
kubectl api-resources # shorthand name
```
```
controlplane $ k get quota -n uat-namespace
NAME         AGE   REQUEST                                                      LIMIT
demo-quota   62s   pods: 0/10, requests.cpu: 0/500m, requests.memory: 0/200Gi   limits.cpu: 0/1, limits.memory: 0/400Gi
```
create deployment with 11 pods in uat-namespace, scheduler will not schedule this as pods=10 set in resource quota.

Note: Create resource quota and limit range and test the scheduling behavious in same namespace and check QOS class.


##
###  Label and Selector
##

Create pod 
```
k run nginx --image nginx`
```
label nginx pod with app=testing
```
k label pod nginx app=testing
```
show labels
```
k get pod nginx --show-labels
```

### Equity & Set Based Condition for labels
app!=testing
```
k get pod -l app!=testing
```
in condition
```
k get pod -l 'app in (testing,nginx)'
```

### Scheduling gate: 

If scheduling gate is applied over the pod then scheduler will not schedule this pod.

```
apiVersion: v1
kind: Pod
metadata:
  name: demo-pod
spec:
  schedulingGates:
  - name: amol
  containers:
  - name: pause
    image: registry.k8s.io/pause:3.6
```
OUTPUT:



`NAME       READY   STATUS            RESTARTS   AGE   IP       NODE     NOMINATED NODE   READINESS GATES
demo-pod   0/1     SchedulingGated   0          10s   <none>   <none>   <none>           <none>
`

### Topology Constraint
You can use topology spread constraints to control how Pods are spread across your cluster among failure-domains such as regions, zones, nodes, and other user-defined topology domains. This can help to achieve high availability as well as efficient resource utilization.

Equaly distribution of workload(pods) over the all nodes using the pod topology spreads.


Scheduling concepts(plugins):https://github.com/kubernetes/kubernetes/tree/master/pkg/scheduler/framework/plugins

```
apiVersion: v1
kind: Pod
metadata:
  name: example-pod
spec:
  # Configure a topology spread constraint
  topologySpreadConstraints:
    - maxSkew: <integer>
      minDomains: <integer> # optional
      topologyKey: <string>
      whenUnsatisfiable: <string>
      labelSelector: <object>
      matchLabelKeys: <list> # optional; beta since v1.27
      nodeAffinityPolicy: [Honor|Ignore] # optional; beta since v1.26
      nodeTaintsPolicy: [Honor|Ignore] # optional; beta since v1.26
  ### other Pod fields go here
```

topology example
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-app
spec:
  replicas: 4
  selector:
    matchLabels:
      app: demo-app
  template:
    metadata:
      labels:
        app: demo-app
    spec:
      containers:
      - name: app-container
        image: nginx
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: "kubernetes.io/hostname"
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: demo-app
```
`maxSkey`: degree to which pods can be evenly distributed over the nodes.

` topologyKey: "kubernetes.io/hostname"` you can see using command
```
 k get nodes --show-labels
```

apply the above manifest and scale the deployment
```
kubectl scale deploy demo-app --replicas 6
```

cardon the control plane: scheduler will not schedule the pods on control plane(disabled scheduling on controlplane)
```
kubectl cordon controlplane
```
`controlplane   Ready,SchedulingDisabled   control-plane`
```
kubectl scale deploy demo-app --replicas 7 
```
scale up again
```
kubectl scale deploy demo-app --replicas 8
```
this 8th one will remain in pending state as we are not scheduling it on controlplane, topology constrain condition will not satify and pod will not schedule.

result:


`demo-app-5967484fb9-csmqq   0/1     Pending           0          13s     <none>        <none>     `

ERROR:


`
Warning  FailedScheduling  116s  default-scheduler  0/2 nodes are available: 1 node(s) didn't match pod topology spread constraints, 1 node(s) were unschedulable. preemption: 0/2 nodes are available: 1 No preemption victims found for incoming pod, 1 Preemption is not helpful for scheduling
`


now, uncordon control plane and get the pods, you will find it will schedule on control plane.
```
k uncordon controlplane
```


### Priority Class
it is an 32 bit integer value, higher the value higher priority.
default priority classes
```
k get priorityclasses -A
```
 for demo purpose create the deploy with high workload of around 160 replicas, so that some of the pod will remain in pending state
```
k create deploy nginx --image nginx --replicas 160
```

Now create priority class 
```
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: demo-priority
value: 1000000
globalDefault: false
description: "This priority class should be used higher priority."
```
Create pod with above priority class  `demo-priority`
```
apiVersion: v1
kind: Pod
metadata:
  name: high-priority-pod
spec:
  priorityClassName: demo-priority
  containers:
  - name: busybox
    image: busybox
    command: ["sleep", "3600"]
    resources:
      requests:
        cpu: "300m"
        memory: "300Mi"
```
result can be seen below, due to its high priority scheduled the evict the low priority pod and schedule the high priority pod.
```
high-priority-pod           1/1     Running             0          55s
```


### Node Affinity:
Scheduling technique of workload on a particular nodes. Affinity - what we want, and on which basis we want to schedule the workload. below is the pod spec section that shows affinity.

```
spec:
  containers:
  - name: my-container
    image: nginx
  affinity:
      nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
              nodeSelectorTerms:
              - matchExpressions:
                - key: "label-on-the-node"
                  operator: In
                  value: "value-of-above-label"
          preferedDuringSchedulingIgnoredDuringExecution:   # If above label matched with more than one nodes then this will preferred by scheduler.
               nodeSelectorTerms:
               - matchExpressions:
                 - key: "another-label-on-node"
                   operator: In
                   value: "value-of-label"
```

Demo:
```
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: mycontainer
    image: nginx
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: "topology.kubernetes.io/region"
            operator: In
            values:
            - "us-east-1"
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: "disktype"
            operator: In
            values:
            - "ssd"
```

above pod will remain in pending state as it will not find above labels on the nodes.


`controlplane $ k get pods
NAME    READY   STATUS    RESTARTS   AGE
mypod   0/1     Pending   0          21s`

Create labels:
```
k label node node01 topology.kubernetes.io/region=us-east-1
```
Now, the above pod will run on the node01.

demo: 2 
delete the above pod.
```
kubectl delete pod mypod
```
apply same label for controlplane as well
```
k label node controlplane topology.kubernetes.io/region=us-east-1
```
create the label of preferred affinity section for controlplane
```
k label node controlplane disktype=ssd
```
apply the mypod manifest again and create pod
```
k get pods -o wide
```
pod will be scheduled on controlplane this time, as it checked preferred section now

Result:

`controlplane $ k get pods -o wide
NAME    READY   STATUS    RESTARTS   AGE   IP            NODE           NOMINATED NODE   READINESS GATES
mypod   1/1     Running   0          14s   192.168.0.4   controlplane   <none>           <none>
`

### Taints and Tolerations
node affinity attracts the pods to the node where as taint restricts the pod to get scheduled on the nodes. only pod having the tolerations can be scheduled on the node on which we applied the taint.

first, disable the scheduling on controlplane
```
k cordon controlplane
```
### NoSchedule Effect:
Strictly restrict the pod to schedule on node which has the node.


apply taint on node01
```
kubectl taint node node01 app=demo:NoSchedule
```

create pod with no tolerations
```
k run nginx --image nginx
```

Result due to no tolerations on pod:

`controlplane $ k get pods -o wide
NAME    READY   STATUS    RESTARTS   AGE   IP       NODE     NOMINATED NODE   READINESS GATES
nginx   0/1     Pending   0          44s   <none>   <none>   <none>           <none>`

now create pod with tolerations:
```
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: demo
  name: demo
spec:
  tolerations:
  - key: "app"
    operator: "Equal"
    value: "demo"
    effect: "NoSchedule"
  containers:
  - image: nginx
    name: demo
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```
now you can observe it is scheduled on the node01.



### PreferNoSchedule
Even if pod does not have any toleration and if scheduler will not able to find any node to schedule the pod then it will prefer the node which has toleration effect as PreferNoSchedule and will schedule the pod.



remove the existing taint of node01
```
kubectl taint nodes node01 app-
```
apply taint with effect PreferNoSchedule
```
k run nginx --image nginx
```
```
k taint node node01 app=demo:PreferNoSchedule
```
create pod without toleration and see the result:
```
k run nginx --image nginx
```

### NoExecute

Apply taint on node01

```
k taint node node01 app=demo:NoExecute
```

Now you can see the executing pod which does not have this toleration will evict.



