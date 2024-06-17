
## To check which Cgroup version is being used
```bash
stat -fc %T /sys/fs/cgroup 

cgroup2fs
```
### K3s Installation
- k3s is a convenient lightweight distribution that can be used for setting up a multinode Kubernetes cluster. Let's start by initialising our control-plane node.

```bash
curl -sfLk https://get.k3s.io | INSTALL_K3S_VERSION=v1.28.4+k3s1 K3S_TOKEN=KCNA INSTALL_K3S_EXEC="--disable traefik --kubelet-arg=eviction-hard=imagefs.available<1%,nodefs.available<1%" sh -
```
As part of the k3s setup process, it will create a symbolic link from the k3s binary to /usr/local/bin/kubectl

```bash
ls -altrh /usr/local/bin/kubectl
cat /etc/rancher/k3s/k3s.yaml
```

### File movement

For consistency with  distributions, we will move this to ~/.kube/config - k3s will work as expected with this file in both locations -

```bash 
mkdir -p ~/.kube; mv /etc/rancher/k3s/k3s.yaml ~/.kube/config
```

```bash
ssh worker-1 'curl -sfLk https://get.k3s.io | INSTALL_K3S_VERSION=v1.28.4+k3s1 K3S_URL=https://control-plane:6443 K3S_TOKEN=KCNA INSTALL_K3S_EXEC="--kubelet-arg=eviction-hard=imagefs.available<1%,nodefs.available<1%" sh -'

ssh worker-2 'curl -sfLk https://get.k3s.io | INSTALL_K3S_VERSION=v1.28.4+k3s1 K3S_URL=https://control-plane:6443 K3S_TOKEN=KCNA INSTALL_K3S_EXEC="--kubelet-arg=eviction-hard=imagefs.available<1%,nodefs.available<1%" sh -'
```
```
NAME            STATUS   ROLES                  AGE    VERSION
control-plane   Ready    control-plane,master   101m   v1.28.4+k3s1
worker-2        Ready    <none>                 101m   v1.28.4+k3s1
worker-1        Ready    <none>                 101m   v1.28.4+k3s1
```

#### Let us run the simple nginx pod
```bash 
kubectl run nginx --image=nginx
```

#### Deleting the Pod
```bash
kubectl delete pod/nginx pod/ubuntu --now
```
#### To see the  the yaml declaration that the cli is using, let's tee this to a file called nginx.yaml at the same time -

```bash
kubectl run nginx --image=nginx --dry-run=client -o yaml | tee nginx.yaml
```
```yml 
# output
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
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
root@control

```
#### To understand what restartpolicy is we can use the below command
```bash
kubectl explain pod.spec.restartPolicy
```
```
# output

KIND:       Pod
VERSION:    v1

FIELD: restartPolicy <string>

DESCRIPTION:
    Restart policy for all containers within the pod. One of Always, OnFailure,
    Never. In some contexts, only a subset of those values may be permitted.
    Default to Always. More info:
    https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#restart-policy
    
    Possible enum values:
     - `"Always"`
     - `"Never"`
     - `"OnFailure"`

```

#### create this in kubernetes using the imperative approach of kubectl create with -f to specify the filename nginx.yaml 

```bash
kubectl create -f nginx.yaml
```

#### Declaritive approach to create the pod
- we are applying what this entity should look like. You will get a warning initially if you do this to an entity that was created 
```bash
kubectl apply -f nginx.yaml
```

#### Combining two differnet yaml file into a single for Apply/ Create
- Currently we have two separate files, but a yaml file can contain multiple declarations, let's combine the two files with a --- yaml separator, we will use tee to save this to combined.yaml.
```bash 
{ cat nginx.yaml; echo "---"; cat ubuntu.yaml; } | tee combined.yaml
```
- Now we can apply this file to create the pod

```bash
kubectl apply -f combined.yaml
```
- This method will create 2 different pods 

### Creating 2 container in the same pod 
- use case : If you want to use the ubuntu server as the sidecar 

```bash
cat <<EOF > combined.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: mypod
  name: mypod
spec:
  containers:
  - image: nginx
    name: webserver
    resources: {}
  - image: ubuntu
    name: sidecar
    args:
    - /bin/sh
    - -c
    - while true; do echo "\$(date +'%T') - Hello from the sidecar"; sleep 5; if [ -f /tmp/crash ]; then exit 1; fi; done
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
EOF
```



#### we see the logs of the pod using the kubectl logs command

```bash
kubectl logs pod/nginx
```

#### This will provide extra information about deployed pods

```bash
kubectl get pods -o wide
```
### Get the pod ip and store it somewhere ###
```bash
NGINX_IP=$(kubectl get pods -o wide | awk '/nginx/ { print $6 }'); echo $NGINX_IP
```
##### Ping from control-plane and workers###
```bash
ping -c 3 $NGINX_IP

ssh worker-1 ping -c 3 $NGINX_IP

ssh worker-2 ping -c 3 $NGINX_IP
```


##### Port Forwarding ####
- We can also try the kubectl port-forward functionality, go back to our terminal tab, then let's forward pod/nginx to localhost:8080 on the control-plane and 80 in the pod, when you have finished using the port-forward, you'll need to use ctrl-c to close

```bash 
kubectl port-forward pod/nginx 8080:80
```
#### Get the complete information about POD

- we can describe the pod for further information, in particular the IP address and each of the containers
```bash 
kubectl describe pod/mypod

kubectl logs pod/mypod -c sidecar
```
#### Get info on number of namespaces present
```bash
kubectl get namespaces/ns
```


#### Get info on Resources running on the each name space

```bash 
kubectl get all -A 
```

#### Get all the Api resource in and out of the namespace 
```bash 
kubectl api-resources | more
```
```
NAME                              SHORTNAMES   APIVERSION                             NAMESPACED   KIND
bindings                                       v1                                     true         Binding
componentstatuses                 cs           v1                                     false        ComponentStatus
configmaps                        cm           v1                                     true         ConfigMap
endpoints                         ep           v1                                     true         Endpoints
events                            ev           v1                                     true         Event
limitranges                       limits       v1                                     true         LimitRange
namespaces                        ns           v1                                     false        Namespace
nodes                             no           v1                                     false        Node
persistentvolumeclaims            pvc          v1                                     true         PersistentVolumeClaim
persistentvolumes                 pv           v1                                     false        PersistentVolume
pods                              po           v1                                     true         Pod
```

#### Creating the namespace 

```bash 
kubectl create namespace mynamespace 
# To run and check the pods in namespace 

kubectl -n mynamespace run nginx --image=nginx

kubectl -n mynamespace get pods

#Delete NS
kubectl delete namespace/myawesomenamespace --now
```

## Setting the config 

- By definition you will see that our context is using the default namespace, either with a defined key set to default or no key present

```bash 
kubectl config view
-------------------
contexts:
- context:
    cluster: default
    namespace: default
    user: default
  name: default
----------------------
  ```

- If we desire, we can update our kubernetes server configuration to use a specific namespace

```bash
kubectl config set-context --current --namespace=mynamespace

###REVERT to Default
kubectl config set-context --current --namespace=default
```
- now by default the pods created will be created in mynamspace 

# Deployment and Replica sets

### Deployment

```bash 
kubectl create deployment nginx --image=nginx --dry-run=client -o yaml | tee nginx-deployment.yaml | kubectl apply -f -
```
- The kubectl command's output is first sent to both a file and the terminal via tee
This output, appearing on the terminal (stdout), then acts as input (stdin) for the following command, as we're using a pipe
Finally, we use a dash with kubectl to specify that it should read this input from stdin.


```bash

apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: nginx
    type:webserver
  name: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
        resources: {}
status: {}

```


```bash
kubectl get deployment

---------------------------------------------------
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   1/1     1            1           4m53s
---------------------------------------------------
#More information

kubectl get deployment/nginx -o yaml

```

```bash
kubectl get replicaset

---------------------------------------------------
NAME               DESIRED   CURRENT   READY   AGE
nginx-7854ff8877   1         1         1       5m1s
nginx-54658f9cf8   3         3         3       4m
nginx-b9b667b8b    0         0         0       23m
---------------------------------------------------
```
#### Roll Out Status
```bash
kubectl rollout status deployment/nginx
-------------------------------------------
deployment "nginx" successfully rolled out
-------------------------------------------

kubectl apply -f nginx-deployment.yaml && kubectl rollout status deployment/nginx

-----------------------------------------------------------------------------------------------
Waiting for deployment "nginx" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "nginx" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "nginx" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "nginx" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "nginx" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "nginx" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "nginx" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "nginx" rollout to finish: 1 old replicas are pending termination...
deployment "nginx" successfully rolled out
-----------------------------------------------------------------------------------------------
```

#### Deployments have a history, lets check the initial rollout history

```bash
kubectl rollout history deployment/nginx
------------------------------------------
deployment.apps/nginx 
REVISION  CHANGE-CAUSE
1         <none>
------------------------------------------
```
##### Annotation
- At the moment there is only 3 section in our history, we can annote this to make use of this record -
#```bash
<pre lang=bash>
<B>kubectl annotate deployment/nginx kubernetes.io/change-cause="Initial deployment"</b>
--------------------------------------------------------------------------------------
deployment.apps/nginx 
REVISION  CHANGE-CAUSE
1         initial deployment
2         second  stable  deployment
3         Third  alpine  deployment
</pre>
 scale the number of replicas, we will run a watch command straight after to see this as it progresses.
>> press ctrl-c to exit 

```bash

**kubectl scale deployment/nginx --replicas=10; watch kubectl get pods -o wide**

```



- We can change the file with increased or decrecased numbe of pods and then deploy pod

```bash
kubectl apply -f nginx-deployment.yaml && kubectl rollout status deployment/nginx
```
#### Desribe 

- We can check the history of deployment and also you will now see references to the new and old replicaSets and much more
```bash
kubectl describe deployment/nginx

------------------------------------------------------------------------
 Type    Reason             Age                From                   Message
  ----    ------             ----               ----                   -------
  Normal  ScalingReplicaSet  58m                deployment-controller  Scaled up replica set nginx-7854ff8877 to 1
  Normal  ScalingReplicaSet  55m                deployment-controller  Scaled up replica set nginx-7854ff8877 to 2 from 1
  Normal  ScalingReplicaSet  53m                deployment-controller  Scaled up replica set nginx-7854ff8877 to 3 from 2
  Normal  ScalingReplicaSet  39m                deployment-controller  Scaled up replica set nginx-7854ff8877 to 4 from 3
  Normal  ScalingReplicaSet  27m                deployment-controller  Scaled up replica set nginx-b9b667b8b to 1
  Normal  ScalingReplicaSet  27m                deployment-controller  Scaled down replica set nginx-7854ff8877 to 3 from 4
  Normal  ScalingReplicaSet  27m                deployment-controller  Scaled up replica set nginx-b9b667b8b to 2 from 1
  Normal  ScalingReplicaSet  26m                deployment-controller  Scaled down replica set nginx-7854ff8877 to 2 from 3
  Normal  ScalingReplicaSet  26m                deployment-controller  Scaled up replica set nginx-b9b667b8b to 3 from 2
  Normal  ScalingReplicaSet  26m                deployment-controller  Scaled down replica set nginx-7854ff8877 to 1 from 2
  Normal  ScalingReplicaSet  46s (x4 over 51s)  deployment-controller  (combined from similar events): Scaled down replica set nginx-54658f9cf8 to 0 from 1
```
---

##### Roll out to specific version 
```bash
kubectl rollout undo deployment/nginx --to-revision=1 && kubectl rollout status deployment/nginx
```

```bash
deployment.apps/nginx rolled back
deployment "nginx" successfully rolled out

kubectl rollout history deployment/nginx
REVISION  CHANGE-CAUSE
1         initial deployment
2         second  stable  deployment
3         Third  alpine  deployment
5         change image to nginx:apples - failed
6         Fourth  perl  deployment
```


```bash
kubectl get replicaset -o wide

---------------------------------------------------------------------------------------------
NAME               DESIRED   CURRENT   READY   AGE     CONTAINERS   IMAGES          SELECTOR
nginx-7854ff8877   0         0         0       64m     nginx        nginx           app=nginx,pod-template-hash=7854ff8877
nginx-b9b667b8b    0         0         0       33m     nginx        nginx:stable    app=nginx,pod-template-hash=b9b667b8b
nginx-54658f9cf8   0         0         0       13m     nginx        nginx:alpine    app=nginx,pod-template-hash=54658f9cf8
nginx-6bc7ff9956   3         3         3       6m59s   nginx        nginx:perl      app=nginx,pod-template-hash=6bc7ff9956
nginx-78bcb9d677   0         0         0       3m52s   nginx        nginx:bananas   app=nginx,pod-template-hash=78bcb9d677
---------------------------------------------------------------------------------------------
```
---



