# Kubernetes Scripts
A collection of scripts for various tasks in [Kubernetes](https://kubernetes.io/).

## Usage
Each script has a `usage` function. See usage with
```shell script
$ <script> --help
```

## Scripts
* [findEmptyNamespaces.sh](findEmptyNamespaces.sh): Loop over all namespaces in a cluster and find empty ones.
* [getPodsTopCSV.sh](getPodsTopCSV.sh): Get a pod's cpu and memory usage (optionally per container) written as CSV formatted file.
* [getResourcesCSV.sh](getResourcesCSV.sh): Get all pods resources requests and limits per container in a CSV format with values normalized. 
CSV format is very automation friendly and is great for pasting in an Excel or Google sheet for further processing.
* [getRestartingPods.sh](getRestartingPods.sh): Get all pods (all or single namespace) that have restarts detected in one or more containers. Formatted in CSV.
* [podReady](podReady.sh): Simple script to check if pod is really ready. Check status is 'Running' and that all containers are ready.
Returns 0 if ready. Returns 1 if not ready.

## One liners
### Kubectl
#### See all custer nodes load (top)
```shell script
kubectl top nodes
```

#### Get cluster events
```shell script
# All cluster
kubectl get events

# Specific namespace events
kubectl get events --namespace=kube-system
```

#### Get all cluster nodes IPs and names
```shell script
# Single call to K8s API
kubectl get nodes -o json | grep -A 12 addresses

# A loop for more flexibility
for n in $(kubectl get nodes -o name); do \
  echo -e "\nNode ${n}"; \
  kubectl get ${n} -o json | grep -A 8 addresses; \
done
```

#### See all cluster nodes CPU and Memory requests and limits
```shell script
kubectl describe nodes | grep -A 3 "Resource .*Requests .*Limits"
``` 

#### Get all labels attached to all pods in a namespace
```shell script
for a in $(kubectl get pods -n namespace1 -o name); do \
  echo -e "\nPod ${a}"; \
  kubectl -n namespace1 describe ${a} | awk '/Labels:/,/Annotations/' | sed '/Annotations/d'; \
done
```

#### Forward local port to a pod or service
```shell script
# Forward localhost port 8080 to a specific pod exposing port 8080
kubectl port-forward -n namespace1 web 8080:8080

# Forward localhost port 8080 to a specific web service exposing port 80
kubectl port-forward -n namespace1 svc/web 8080:80
```

#### Port forwarding
* A great tool for port forwarding all services in a namespace + adding aliases to `/etc/hosts` is [kubefwd](https://github.com/txn2/kubefwd).
Note that this requires root or sudo to allow temporary editing of `/etc/host`.
```shell script
# Port forward all service in namespace1
kubefwd svc -n namespace1
```

#### Extract and decode a secret's value
```shell script
# Get the value of the postgresql password
kubectl get secret -n namespace1 my-postgresql -o jsonpath="{.data.postgres-password}" | base64 --decode
```

#### Copy secret from `namespace1` to `namespace2`
```shell script
kubectl get secret my-secret --namespace namespace1 -o yaml | sed "/namespace:/d" | kubectl apply --namespace=namespace2 -f -
```

#### Create an Ubuntu pod
A one liner to create an Ubuntu pod that will just wait forever. 
```shell script
# Create the pod
cat <<ZZZ | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: my-ubuntu-pod
spec:
  containers:
  - name: my-ubuntu-container
    image: ubuntu:20.04
    command:
    - 'bash'
    - '-c'
    - 'while true; do sleep 5; done'
ZZZ

# Shell into the pod
kubectl exec -it my-ubuntu-pod bash

# Delete the pods once done
kubectl delete pod my-ubuntu-pod
```

#### Start a shell in a temporary pod
Note - Pod will terminate once exited
```shell script
# Ubuntu
kubectl run my-ubuntu --rm -i -t --restart=Never --image ubuntu -- bash

# CentOS
kubectl run my-centos --rm -i -t --restart=Never --image centos:8 -- bash

# Alpine
kubectl run my-alpine --rm -i -t --restart=Never --image alpine:3.10 -- sh

# Busybox
kubectl run my-busybox --rm -i -t --restart=Never --image busybox -- sh
```

#### Get formatted list of container images in pods
Useful for listing all running containers in your cluster
```shell script
kubectl get pod --all-namespaces \
    -o=jsonpath='{range .items[*]}{.metadata.namespace}, {.metadata.name}, {.spec.containers[].image}{"\n"}'
```
Look into [a few more examples](https://kubernetes.io/docs/tasks/access-application-cluster/list-all-running-container-images) of listing containers

#### Get current replica count on all HPAs (Horizontal Pod Autoscaler)
```shell script
kubectl get hpa --all-namespaces -o=custom-columns=NAME:.metadata.name,REPLICAS:.status.currentReplicas | sort -k2 -n -r
```

### Helm
**NOTE:** It is recommended to move to [Helm v3](https://helm.sh/docs/), which does not use tiller anymore.

#### Helm v2 on an RBAC enabled cluster
This will give tiller `cluster-admin` role
```shell script
kubectl -n kube-system create sa tiller && \
    kubectl create clusterrolebinding tiller --clusterrole cluster-admin --serviceaccount=kube-system:tiller && \
    helm init --service-account tiller
```

#### Helm template
View the templates generated by `helm install`. Useful for seeing the actual templates generated by helm before deploying.<br>
Can also be used for deploying the templates generated when cannot use Tiller
```shell script
helm template <chart>
```

#### Debug helm install
* Debug a `helm install`. Useful for seeing the actual values resolved by helm before deploying
```shell script
helm install --debug --dry-run <chart>
```

#### Rolling restart
Roll a restart across all resources managed by a Deployment, DaemonSet or StatefulSet with **zero downtime**
```shell script
# Deployment
kubectl -n <namespace> rollout restart deployment <deployment-name>

# DaemonSet
kubectl -n <namespace> rollout restart daemonset <daemonset-name>

# StatefulSet
kubectl -n <namespace> rollout restart statefulsets <statefulset-name>
```
