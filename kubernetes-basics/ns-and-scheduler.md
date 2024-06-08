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

