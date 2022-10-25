# Kubernetes Notes

Refer to a README file in each folder for further notes and example.


## Edit

``` bash
# edit a current running pod
kubectl edit pod pod-name
```

## Get output


``` bash
# use --dry-run=client to preview the object only(doesnt submit) then write info into yaml file
kubectl run redis --image=redis --dry-run=client -o yaml > pod-def.yml

# write a current running pod info into yaml file
kubectl get pod pod-name -o yaml > pod-def.yml
```



## Accessing a pod from another namespace
#### Example URL
```bash
db-service.dev.svc.cluster.local
```
`db-service` : service name(deployment, pod, etc)  
`dev` : namespace  
`svc` : service  
`cluster.local` : domain




## Labels and Selector

You can use the labels for retrieving and filtering the data from the Kubernetes API.

#### Assign labels to pod and search
```bash
# To assgin label 
k label pod <pod-name> <label-key>=<label-val> <label-key2>=<label-val2>
# to overwrite
k label pod <pod-name> <label-key>=<label-val> --overwrite

# To check pod labels
kubectl get pod --show-labels
# or search pod with label 
kubectl get pod -L <key-name>
kubectl get pod -L <key-name>=<val-name>
k get pod --selector <key-name>=<val-name>

# delete specific pod with label 
k delete pod --selector <key-name>=<val-name>

# to delete label from pod
k label pod <pod-name> <label-kery>-
```
#### Label Definition at each level
`Selector labels` and `Pod labels` must be the same
``` yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
  labels:
    app: dep-label # Deployment labels <--this label is to manage the deployment itself. this may be used to filter the deployment based on this label.   Check dep label by k get deploy <dep-name> --show-labels
spec:              
  replicas: 2
  selector:
    matchLabels:
      app: my-nginx    # Selector labels:  <--  this is where we tell replication controller to manage the pods. This field defines how the Deployment finds which Pods to manage. Must match with Pod labels
  template:
    metadata:
      labels:
        app: my-nginx   # Pod labels: <--this is the label of the pod, this must be same as Selector labels. Check pod label by k get pod <pod-name> --show-labels
    spec:                
      containers:
      - name: my-nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
        resources:
          limits:
            memory: "128Mi" #128 MB
            cpu: "200m" #200 millicpu (.2 cpu or 20% of the cpu)

```



## Anotation

Annotations are the second way of attaching metadata to the Kubernetes resources.  

Annotations are NOT used to identify and select objects
Labels are for Kubernetes, while annotations are for humans.

#### Anotation Example
``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  annotations:
    builder: sungkim
    buildDate: 1111111
    imageRegistry: https://hub.docker.com
spec:
  containers:
    - name: nginx
      image: nginx
      ports:
        - containerPort: 8080
```

```bash 
# To check annotation
k describe pod <pod-name>
```
## Scale
```bash
# After creatuib of deployment, use scale command. This doesn't change the corresponding yaml file. 
kubectl scale deployment nginx --replicas=4
```



## Service

Imperative command example:
#### Create a service name redis-sevice to expose pod redis
``` bash
# assume pod exists already. this maps service to pod redis
kubectl expose pod redis --port=6379 --name redis-service --dry-run=client -o yaml
```
#### Create a pod and service at the same time and expose service to pod
Example:  
``` bash
# pod and service creatation at the same time
# create a pod and service called httpd and expose service to the pod at once.
# and the target port for the service is 80
$ kubectl run httpd --image=httpd:alpine --port 80 --expose --dry-run=client -o yaml
```

Corresponding YAML file:
``` yaml
apiVersion: v1
kind: Service
metadata:
  name: httpd
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    run: httpd  # --expose option ensures the selector matches the labels from pod(**)
---
---
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: httpd   # this part **
  name: httpd
spec:
  containers:
  - image: httpd:alpine
    name: httpd
    ports:
    - containerPort: 80
```




## Log

To view the entire logs of replicated pods
``` bash
k logs -l <label-key>=<val>
k logs -l app=webui -f
```