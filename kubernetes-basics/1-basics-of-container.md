##
### Basics of Containers ### 
##

Containers are nothing but the linux process. Contaiers are isolated by the linux namegroups. To see linux namespace used by container follow below:

  
1. Create any simple docker container
```
docker run -d --name my-container --memory 512m --cpus 1 nginx 
```
2. Check pid of above container
```
ps aux | grep '[n]ginx' | sort -n -k 2 | head -n 1 | awk '{print $2}'
```

3. Check the linux namespaces for the above container pid
```
lsns -p pid
```
result looks something like below
```
ubuntu $ lsns -p 2803
        NS TYPE   NPROCS   PID USER COMMAND
4026531835 cgroup    119     1 root /sbin/init
4026531837 user      119     1 root /sbin/init
4026532341 mnt         2  2803 root nginx: master process nginx -g daemon off;
4026532342 uts         2  2803 root nginx: master process nginx -g daemon off;
4026532343 ipc         2  2803 root nginx: master process nginx -g daemon off;
4026532344 pid         2  2803 root nginx: master process nginx -g daemon off;
4026532346 net         2  2803 root nginx: master process nginx -g daemon off;
```
4. Check the control groups
```
systemd-cgls --no-pager
```
above will give result something below:
```
Control group /:
-.slice
├─1005 bpfilter_umh
├─init.scope 
│ └─1 /sbin/init
└─system.slice 
  ├─docker-34eeb3fa062ecf1512f8f02398a34570f6adb4042978608fbcee43f3ba3116de.scope 
```
5. To see memory we allocated for nginx container in 1st step
```
cat /sys/fs/cgroup/memory/system.slice/docker-34eeb3fa062ecf1512f8f02398a34570f6adb4042978608fbcee43f3ba3116de.scope/memory.stat
```
