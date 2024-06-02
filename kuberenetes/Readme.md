
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
  ```

- If we desire, we can update our kubernetes server configuration to use a specific namespace

```bash
kubectl config set-context --current --namespace=mynamespace

###REVERT to Default
kubectl config set-context --current --namespace=default
```
- now by default the pods created will be created in mynamspace 


 


