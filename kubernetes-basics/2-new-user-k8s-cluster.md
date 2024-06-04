##
## Create New User for Cluster ##
##

Create User For cluster and provide limited access to the resources

1. Generate openssl key
 ```
openssl genrsa -out amol.key 2048
```
2. Create CSR
```
openssl req -new -key amol.key -out amol.csr -subj "/CN=amol/O=group1"
```
3. Read the amol.csr, this is in base64 encoded format. decode this key
```
cat amol.csr | base64 | tr -d '\n'
```
4. Sign CertificateSigningRequest(CSR) with Kubernetes CA
   csr.yml
```
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: amol
spec:
  request: BASE64_CSR     # paste above decoded key 
  signerName: kubernetes.io/kube-apiserver-client
  usages:
  - client auth
```
5. Create CSR
```
kubectl create -f csr.yml
```
6. Approve New User
```
kubectl certificate approve amol
```
7. Create Client Certificate-  amol.crt
```
kubectl get csr amol -o jsonpath='{.status.certificate}' | base64 --decode > amol.crt
```
8. Create Role- role.yml
```
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```
```
kubectl create -f role.yml
```
9. Create Role Binding role-binding.yml
```
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: amol
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```  
```
kubectl create -f role-binding.yml
```
## Setup KubeConfig with New User Details
1. set user
```
kubectl config set-credentials amol --client-certificate=amol.crt --client-key=amol.key
```
2. Get Context: 
```
kubectl config get-contexts
```
3. Set User Context:
```
kubectl config set-context amol-context --cluster=kubernetes --namespace=default --user=amol 
```  
4. Use Context
Before setting the new user context, create pod with admin user context
```
kubectl run demo-pod --image nginx
```
Now, use the new user context and read, create deployment/pod and see the result.
```
kubectl config use-context amol-context
```
Now, get the context and see the * pointer
```
kubectl config get-contexts
```
Now Create deployment and see if user is able to create
```
kubectl create deployment my-app --image nginx --replicas=3
```
Error message:
error: failed to create deployment: deployments.apps is forbidden: User "amol" cannot create resource "deployments" in API group "apps" in the namespace "default"

## Merging multiple KubeConfig files
```
export KUBECONFIG=/path/to/first/config:/path/to/second/config:/path/to/third/config
```
   
