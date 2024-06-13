1. Setup Cluster
2. Export kubeconfig: `export KUBECONFIG=/path/to/configfile
3. Use get nodes command to see everything is fine.
4. Create Dockerfile
5. Build Image
```
docker build --no-cache --platform=linux/amd64 -t ttl.sh/amol/demo:10h .
```
6. Push on anonymous or docker registry
```
docker push ttl.sh/amol/demo:10h
```
