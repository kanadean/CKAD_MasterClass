# Multi-Container PODs

Follow the instructions below to execute in the minikube Cluster.
If you don't have minikube installed you can refer to below video to install minukube.

https://youtu.be/25pjepyr21M

Or, 
Refer Minukube installation page at,

https://minikube.sigs.k8s.io/docs/start/

## Multi-Container Pod patterns

### Sidecar Pattern
Main container supported by other container/s with main purpose of helping main container with tasks such as statistics and logs distribution etc

### Ambassador Pattern (Proxy Pattern)
Several/single Main containers listening on localhost that sits behind proxy container taking request from network and distrubuting to containers behind proxy (One proxy container, 1+ application service containers)

### Adapter Pattern
Main application service containers with additional container to adapt the output of main container in desired format (e.g csv to json log etc.)

## Init Containers
It is special type of multi-container pod where Init container is started first and main containers is started only after init contianer completes. The pupose of Init Container is to initialize the environment for the main contianer to operate. 

Ref: https://kubernetes.io/docs/concepts/workloads/pods/init-containers/

### Create Init Container POD

Initiate pod with 2 Init containers

```
mkdir -p /tmp/ckad
cat <<-"EOF" > /tmp/ckad/initPod.yaml

apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app.kubernetes.io/name: MyApp
spec:
  containers:
  - name: myapp-container
    image: busybox:1.28
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
  initContainers:
  - name: init-myservice
    image: busybox:1.28
    command: ['sh', '-c', "until nslookup myservice.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local; do echo waiting for myservice; sleep 2; done"]
  - name: init-mydb
    image: busybox:1.28
    command: ['sh', '-c', "until nslookup mydb.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local; do echo waiting for mydb; sleep 2; done"]
EOF
```
Apply the yaml
```
kubectl apply -f /tmp/ckad/initPod.yaml
```

Check the status of the pod
```
kubectl get pods -n default
```

Check the pod description
```
kubectl describe pod myapp-pod -n default
```

Initiate the two dummy services Init containers are waiting on
```
kubectl expose pod myapp-pod --name mydb --port 100
kubectl expose pod myapp-pod --name myservice --port 200

kubectl get svc -n default
```

Check the status of the pod again
```
kubectl get pods -n default
```


Cleanup
```
kubectl delete -f /tmp/ckad/initPod.yaml
kubectl delete svc mydb -n default
kubectl delete svc myservice -n default
```
