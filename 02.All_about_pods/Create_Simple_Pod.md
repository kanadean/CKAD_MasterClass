# Create Simple POD

Follow the instructions below to execute in the minikube Cluster.
If you don't have minikube installed you can refer to below video to install minukube.

https://youtu.be/25pjepyr21M

Or, 
Refer Minukube installation page at,

https://minikube.sigs.k8s.io/docs/start/




## Verify output of below commands to ensure minikube is working fine
```
minikube status
kubectl get nodes
kubectl get pods -n kube-system
```

## Create a sample busybox pod (dry-run)
Dry-run is way to see how the command will behave without actually executing the command
```
kubectl run test-busybox --image=busybox:latest --dry-run=client --command -- sleep 3600 
```

## Create a sample busybox pod 
Create an actual pod
```
kubectl run test-busybox --image=busybox:latest --command -- sleep 3600 
```

Check if pod is running
```
kubectl get pods -n default
```

## Inspect the pod created
Execute following commands to view the output

```
kubectl describe pod test-busybox -n default
kubectl get pods test-busybox -n default -o yaml > /tmp/busybox.yaml
```

## Edit busybox.yaml to use in declarative command

- Edit busybox.yaml in the editor and set "name: test-busybox" as "name: test-busybox-declarative" under 'metadata'
- Execute below command to create a new pod using declrative method
```
kubectl apply -f /tmp/busybox.yaml
```
- Check the pod created
```
kubectl get pods
```

## Edit busybox.yaml to change the image to nginx

- generate the yaml file again
```
kubectl get pods test-busybox-declarative -n default -o yaml > /tmp/busybox.yaml
```
- Edit busybox.yaml in the editor and set "name: test-busybox" as "image: nginx:latest" under 'spec'
- Execute below command to create a new pod using declrative method
```
kubectl apply -f /tmp/busybox.yaml
```
- Check the pod created
```
kubectl get pods
```


## dry-run trick to generate pod definition boilerplate

It is not necessary to create yaml file from scratch to use in declarative commands. 
Below command can be used to generate pod definition yaml and edited as needed before invoking 'kubectl apply'
```
kubectl run test-busybox --image=busybox:latest --command -- sleep 3600 --dry-run=client -o yaml > /tmp/testPod.yaml
```
