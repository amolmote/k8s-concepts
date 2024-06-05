##
### Pod lifecycle
##
1. Pending - finding the best suitable node, waiting for PV to be ready and bound PVC to it.
2. ContainerCreating - Pulling image, starting it, attach network
3. Running
4. Error - failed to pull the image/did not find the node with suitable resource etc
5. CrashLoopBackOff - Process dying too many times
6. Succeeded - Exited after performing its task

```
kubectl run myapp --image nginx
```
### Dry Run Command
```
kubectl run my-pod --image nginx --dry-run=client -o yaml
```

Simple Pod definition file
```
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: my-pod
  name: my-pod
spec:
  containers:
  - image: nginx
    name: my-pod
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

Pod logs
```
kubectl logs pod_name
```


### Init container
Container which start before the main container, this container already runs to completion. when it done with its execution then only main container will start.



