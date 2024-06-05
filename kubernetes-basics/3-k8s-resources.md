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
Container which start before the main container, this container always runs to completion. when it done with its execution then only main container will start.
```
apiVersion: v1
kind: Pod
metadata:
  name: init-pod
spec:
  volumes:
  - name: shared-data
    emptyDir: {}
  initContainers:
  - name: init-container
    image: busybox
    command: ['sh', '-c', 'wget -O /usr/share/data/index.html http://kubesimplify.com']
    volumeMounts:
    - name: shared-data
      mountPath: /usr/share/data
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: shared-data
      mountPath: /usr/share/nginx/html   # index.html file will get copied at this location use below exec command to see it
```
###Exec into the pod
```
kubectl exec --stdin --tty init-pod -- /bin/bash
```

### Multiple Init Container
Scenario: Main Container will not start its execution until the init container finished there execution.
here, Here 2 init container will finish its execution when db service & my-service will get created. then main container will start.
```
apiVersion: v1
kind: Pod
metadata:
  name: init-demo-2
spec:
  initContainers:
  - name: check-db-service
    image: busybox
    command: ['sh', '-c', 'until nslookup db.default.svc.cluster.local; do echo waiting for db service; sleep 2; done;']
  - name: check-myservice
    image: busybox
    command: ['sh', '-c', 'until nslookup myservice.default.svc.cluster.local; do echo waiting for db service; sleep 2; done;']
  containers:
  - name: main-container
    image: busybox
    command: ['sleep', '3600']
```

db Service:
```
apiVersion: v1
kind: Service
metadata:
  name: db
spec:
  selector:
    app: demo1
  ports:
  - protocol: TCP
    port: 3306
    targetPort: 3306
```
my-service
```
apiVersion: v1
kind: Service
metadata:
  name: myservice
spec:
  selector:
    app: demo2
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

### Sidecar Container 
change has been in v1.28 kubernetes release, if we created initContainer with `restartPolicy: Always` then it will work as a sidecar container.
```
apiVersion: v1
kind: Pod
metadata:
  name: side-container-pod
spec:
  volumes:
  - name: shared-data
    emptyDir: {}
  initContainers:
  - name: meminfo-container
    image: alpine
    restartPolicy: Always
    command: ['sh', '-c', 'sleep 5; while true; do cat /proc/meminfo > /usr/share/data/index.html; sleep 10; done;']
    volumeMounts:
    - name: shared-data
      mountPath: /usr/share/data
  containers:
  - name: nginx-container
    image: nginx
    volumeMounts:
    - name: shared-data
      mountPath: /usr/share/nginx/html
```
get pods result
```
side-container-pod   2/2     Running                 0              21s
```



