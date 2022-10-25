---
hide:
  - footer
---

# Controller

Controller ensures that the specified number of pod replicas are running at any point of time. 



1. Replication Controller
2. ReplicaSet
3. Deployment(Rolling-Update & Rollback)
4. DaemonSet
5. StatefulSet
6. Job
7. CronJob  



## 1. Replication Controller

#### ReplicationController YAML Example

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: nginx-rep
spec:
  replicas: 3
  selector:
    app: webui              # must match with one of pod labels
    version: "2.1"  # can be multiple, if multiple labels then selector uses "AND" condition
  template:                 # pod template
    metadata:
      name: nginx-pod
      labels:               # pod labels
        app: webui      
        version: "2.1"   # can have multiple lables
    spec:
      containers:
        - name: nginx-container
          image: nginx
```

#### Edit number of scales

```bash
kubectl edit rc <rc-name>
# or 
kubectl scale rc <rc-name> --replicas=N
```

#### Delete Replication Controller

```bash 
# this will delete the entire running pod..
kubectl delete rc <rc-name>
# --cascade=false will keep pods alive
kubectl delete rc <rc-name> --cascade=false
```

## 2. ReplicaSet

Almost identical to replication controller, but provides diverse selector options.  


#### ReplicaSet YAML Example
``` yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-rep
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webui
    matchExpressions:
      - {key: version, operator: In, values: ["2.1", "2.2"]}
    # - {key: version, operator: Exists }
  template:                 # pod template
    metadata:
      name: nginx-pod
      labels:               # pod labels
        app: webui      
        version: "2.1"   # can have multiple lables
    spec:
      containers:
        - name: nginx-container
          image: nginx
```

`Operator` type:  
* In
* NotIn
* Exists: if key matches, include every values
* DoesNotExist
  



## 3. Deployment
---
Deployment controls `ReplicaSet`  

`Deployment` -> `ReplicaSet` -> `Pod`  

`Deployment` is exactly same as `ReplicaSet` unless you define `rollingupdate`.  

#### Deployment YAML Example

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-dep
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webui
  template:                 # pod template
    metadata:
      name: nginx-pod
      labels:               # pod labels
        app: webui      
        version: "2.1"   # can have multiple lables
    spec:
      containers:
        - name: nginx-container
          image: nginx
```

#### How Deployment works example  

`Deployment` -> `ReplicaSet` -> `Pod`  
```bash
$ kubectl get deploy,rs,pod    
NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-dep   3/3     3            3           30s

NAME                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-dep-546b4b7cbc   3         3         3       30s

NAME                             READY   STATUS    RESTARTS   AGE
pod/nginx-dep-546b4b7cbc-d29qt   1/1     Running   0          30s
pod/nginx-dep-546b4b7cbc-f7sw9   1/1     Running   0          30s
pod/nginx-dep-546b4b7cbc-q75cg   1/1     Running   0          30s

```

#### Delete Deployment

```bash 
kubectl delete deployments <dp-name>
# or
kubectl delete -f path/to/dep.yml
```

#### Delete ReplicaSet
If `ReplicaSet` is deleted, then `Deployment` will re-schedule `ReplicaSet` again, then `ReplicaSet` will create `pod`.   


## Rolling update & Rolling Back(Deployment)
---
There are 3 ways to apply `rolling-update`.  
1. `set` command
2. YAML apply
3. `edit` command
### Rolling update

#### 1. Rolling-update via `set` command
`set` command help you make changes to existing application resources.  Run the following command to update `deployment` while deployment is already up and run.  
```bash
kubectl set image deployment <deploy-name> <container-name>=<new-ver-image> --record
```

#### 2. Rolling-update via YAML file
Modify the `image version` and `annotations`, then `kubectl apply` to perform rolling-update.  Applying the modified yaml file is nicer way for rolling-update because it keeps track of version information on yaml file. 
```yaml
kind: Deployment
metadata:
  annotations:
    kubernetes.io/change-cause: version 1.16 # change this
...
spec:
    containers:
    - name: web
        image: nginx:1.16  # change image version
```
The value of `kubernetes.io/change-cause` key gets tracked on `rollout history`.  

Apply rolling-update via YAML after modifying YAML file.  
```bash
$ kubectl apply -f path/to/deploy.yml

$ kubectl rollout history deployment <dep-name>
# ex) 
deployment.apps/nginx-dep 
REVISION  CHANGE-CAUSE
1         version 1.15
2         version 1.16
```

#### 3. Rolling-update via `edit` command

``` bash
$ kubectl edit deployment <dep-name> --record

# history check ex)
$ kubectl rollout history deploy nginx-dep  
deployment.apps/nginx-dep 
REVISION  CHANGE-CAUSE
1         version 1.15
2         kubectl edit deployment nginx-dep --record=true
```


#### Rolling-update status check
Check status of rolling-update while rolling-update is being executed.  
```bash
kubectl rollout status deployment <deploy-name>
```

#### Rolling-update pause/resume
Pause/resume rolling-update while rolling-update is being executed.  
```bash
kubectl rollout pause deployment <deploy-name>
# resume
kubectl rollout resume deployment <deploy-name>
```


Rollout history check  
```bash
kubectl rollout history deployment <deploy-name>

# ex) 
deployment.apps/nginx-dep
REVISION  CHANGE-CAUSE
1         kubectl create --filename=controller/deployment.yml --record=true
2         kubectl set image deployment nginx-dep web=nginx:1.15 --record=true
3         kubectl set image deployment nginx-dep web=nginx:1.16 --record=true
4         kubectl set image deployment nginx-dep web=nginx:1.17 --record=true
```

#### How to control Rolling-update
```yaml
spec:
  progressDeadlineSeconds: 600
  replicas: 3
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: webui
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
```

`progressDeadlineSeconds`: rollback the update if update takes too long.  
`revisionHistoryLimit`: the number of old ReplicaSets to retain to allow rollback.  

`maxSurge`: the maximum number of Pods that can be created over the desired number of Pods.   
if `replicas=3` and `maxSurge=50%`,  3 x 50% = 2(round up).  2 + 3(replicas) = 5.  Thus, 2 additional pods gets created during rolling-update.  

`maxUnavailable`: the maximum number of Pods that can be unavailable during the update process.
if `replicas=3` and `maxUnavailable=50%`, then 2 pods get terminated during rolling-update.  


#### Rollback
```bash
# rollback to the previous version
kubectl rollout undo deploy <deploy-name>
# select the version to rollback(use the index from history)
kubectl rollout undo deploy <deploy-name> --to-revision=N
```

## 4. DaemonSet

`DaemonSet` ensures that a particular pod gets placed every node(one pod at a node). The pod gets automatically scheduled/deleted as a new Node gets createed/deleted. 

#### `DaemonSet` Example
**IMPORTANT: No replicas attribute**
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nginx-daemonset
spec:
  selector:
    matchLabels:
      app: webui
  template:                 # pod template
    metadata:
      name: nginx-pod
      labels:               # pod labels
        app: webui      
    spec:
      containers:
        - name: web
          image: nginx:1.15
```

#### DaemonSet rolling-update via `apply` command

Edit YAML file(image version), then use `apply` command 
```bash
$ kubectl apply -f controller/daemonset.yml
```

**IMPORTANT**  
`DaemonSet` rolling-update behaves a bit different from `deployment` rolling-update. it doesn't create a new pod then terminate, but instead it terminates the pod then create a new pod. 


rollout undo and checking history work in the same manner as `Deployment`.
#### DaemonSet delete
```bash
kubectl delete ds <daemonset-name>
```

## 5. StatefulSet

Manages the deployment and scaling of a set of Pods, and provides guarantees about **the ordering and uniqueness** of these Pods.

If you want to use storage volumes to provide persistence for your workload, you can use a StatefulSet as part of the solution.  

`statefulSet` doesn't ensure a pod to be placed at each node like `daemonset`, but it ensures the order and name of pod. 

#### `statefulSet` example YAML

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: nginx-stateful
spec:
  replicas: 3
  serviceName: sf-service
  podManagementPolicy: Parallel # OrderedReady # optional
  selector:
    matchLabels:
      app: webui
  template:                 # pod template
    metadata:
      name: nginx-pod
      labels:               # pod labels
        app: webui      
    spec:
      containers:
        - name: web
          image: nginx:1.15
```

#### Pod's name example

```bash
$ kubectl get pod

nginx-stateful-0   1/1     Running   0          4m58s
nginx-stateful-1   1/1     Running   0          4m57s
nginx-stateful-2   1/1     Running   0          4m56s
```

If you delete nginx-stateful-1, then a pod with the exact same name gets scheduled again, but it doesn't guarantee to place the pod on the same node. 


#### scale up & down
Check how pod's name changes when scale goes up/down.  

``` bash
kubectl scale statefulset <sf-name> --replicas=N

## ex) from scale 3 to 5
NAME               READY   STATUS    RESTARTS   AGE   IP          NODE      NOMINATED NODE   READINESS GATES
nginx-stateful-0   1/1     Running   0          78s   10.44.0.1   worker1   <none>           <none>
nginx-stateful-1   1/1     Running   0          77s   10.36.0.4   worker2   <none>           <none>
nginx-stateful-2   1/1     Running   0          75s   10.44.0.2   worker1   <none>           <none>
nginx-stateful-3   1/1     Running   0          12s   10.36.0.5   worker2   <none>           <none>
nginx-stateful-4   1/1     Running   0          10s   10.44.0.3   worker1   <none>           <none>



# ex) from scale 5 to 2
NAME               READY   STATUS    RESTARTS   AGE    IP          NODE      NOMINATED NODE   READINESS GATES
nginx-stateful-0   1/1     Running   0          117s   10.44.0.1   worker1   <none>           <none>
nginx-stateful-1   1/1     Running   0          116s   10.36.0.4   worker2   <none>           <none>
```

#### `StatefulSet` Rolling-update
Rolling-update terminates the pod then create a new pod like `DaemonSet`.
It guarantees the pods to be placed on the same node with version changes. 

## 6. Job Controller
`Job Controller` are primarily used for batch processing.  

In a nutshell, 
* Job(task) completed -> Pod in completed status, but alive
* Job(task) fails -> Pod gets recreated the specified time. Pods can be either terminated or alive depending on options on `restartPolicy`


Check [pod's lifecyle](../pod/README.md#pod-lifecyle) and how `restartPolicy` works in pod. `restartPolicy` manages container, not pod. 

### When a pod is failed 

An entire Pod can also fail. ex) when the pod is kicked off the node (node is upgraded, rebooted, deleted, etc.), or if a container of the Pod fails and the `.spec.template.spec.restartPolicy = "Never"`. When a Pod fails, then the `Job controller` starts a new Pod. This means that your application needs to handle the case when it is restarted in a new pod.

### `backoffLimit` policy
You can fail a Job after some amount of retries. Set `.spec.backoffLimit` to specify the number of retries(pod recreation) before considering a Job as failed. Default is `.spec.backoffLimit=6`.  

### Job Controller YAML example

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: nginx-job
spec:
  # completions: 5    # run N time
  # parallelism: 2
  activeDeadlineSeconds: 5 # pod terminated after activeDeadlineSeconds
  template:                 # pod template
    spec:
      containers:
        - name: centos-container
          image: centos:7
          command: ["bash"]
          args:
            - "-c"
            - "echo 'Hello'; sleep 10; echo 'bye'"
      restartPolicy: Never
  backoffLimit: 2
```


### When a task completed

Even if a task completed without a problem, the pod is still alive.  `restartPolicy: Always` is invalid value on Job.  

Output example
```bash
NAME              READY   STATUS      RESTARTS   AGE     IP          NODE      NOMINATED NODE   READINESS GATES
nginx-job-skmg2   0/1     Completed   0          5m33s   10.44.0.1   worker1   <none>           <none>
```


### When a task fails and `restartPolicy: Never` (Recommanded)
`restartPolicy: Never` will not restart **container** even if it fails, thus the job(task) stays as failed, and 'Job Controller` will restart(create a new pod) the pod until the job backoff limit has been reached. **Pods are still alive.**  
```yaml
spec:
    template:
        spec:
            ...
            restartPolicy: Never
        backoffLimit: 2
```

Result
```bash

$ kubectl get pod <pod-name>

# example 
NAME              READY   STATUS       RESTARTS   AGE   IP          NODE      NOMINATED NODE   READINESS GATES
nginx-job-8xfmt   0/1     StartError   0          4s    10.44.0.1   worker1   <none>           <none>
nginx-job-z6kdp   0/1     StartError   0          17s   10.44.0.1   worker1   <none>           <none>
nginx-job-z6pj4   0/1     StartError   0          20s   10.44.0.1   worker1   <none>           <none>
```


### When a task fails and `restartPolicy: OnFailure`

`restartPolicy: OnFailure` restarts **container** when fails. Once the job backoff limit has been reached, your Pod running the Job will be **terminated(pod dies)**. This makes debugging the job's executable mroe difficult. 

```yaml
spec:
    template:
        spec:
            ...
            restartPolicy: OnFailure
        backoffLimit: 2
```
Result 
```bash
$ kubectl get pod <pod-name>

# example 
NAME              READY   STATUS       RESTARTS   AGE   IP          NODE      NOMINATED NODE   READINESS GATES
```


## 7. CronJob

A CronJob creates Jobs on a repeating schedule just like crontab on Linux.  

**IMPORTANT**  
All CronJob schedule: times are based on the timezone of the **kube-controller-manager**.



```yaml title="example.yaml"
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nginx-cronjob
spec:
  schedule: "* * * * *"
  concurrencyPolicy: Allow # Forbid  # allow cronjob to run concurrently if needed
  startDeadlineSeconds: 500
  successfulJobsHistoryLimit: 3 # only keep 3 succesful pod history run by cronjob when kubctl get pod <pod-name>
  jobTemplate: 
    spec:
      template:
        spec:
          containers:
            - name: centos-container
              image: centos:7
              command: ["bash"]
              args:
                - "-c"
                - "echo 'Hello'; sleep 10; echo 'bye'"
          restartPolicy: Never
      backoffLimit: 3
```


#### Cron schedule syntax
```
# ┌───────────── minute (0 - 59)
# │ ┌───────────── hour (0 - 23)
# │ │ ┌───────────── day of the month (1 - 31)
# │ │ │ ┌───────────── month (1 - 12)
# │ │ │ │ ┌───────────── day of the week (0 - 6) (Sunday to Saturday;
# │ │ │ │ │                                   7 is also Sunday on some systems)
# │ │ │ │ │                                   OR sun, mon, tue, wed, thu, fri, sat
# │ │ │ │ │
# * * * * *
```

!!! note "Example" 

    every weekday at 3:00 am  
    0 3 * * 1-5  
    every weekend at 3:00 am  
    0 3 * * 0,6  
    every 5 minute  
    */5 * * * *
