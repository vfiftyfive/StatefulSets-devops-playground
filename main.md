# StatefulSet Workshop

## Objective

The objective of this workshop is to explore Kubernetes StatefulSets and their use cases. Traditionally, Kubernetes applications are developed to run as Web applications. They don't often need to write persistent data to disk or need a stable identity. In the event of a node failure in Kubernetes, the controller plane  spawns new Pods to match the expected number of replicas. This is achieved by leveraging Kubernetes Deployment and ReplicaSets. The Pods they create are stateless and replaceable. When a Pod is created, it takes an arbitraty identity with an auto-generated unique name.
On the opposite, StatefulSets are Kubernetes first class objects providing Pods with a stable identity, and ensuring they can write to individual persistent volumes. They have additional characteristics we're going to highlight in this workshop.

## Create your first application
In this section we review some of the Kubernetes foundations for creating and deploying basic applications.

### Pods
An easy way to create a pod is to use `kubectl run` in dry-run mode and redirect the result to a file. You can then alter the configuration as needed.

1. `Task 1`: Create a Pod with the `nginx` image exposing container port 80.

```
kubectl run nginx --image=nginx --port=80 --dry-run=client -oyaml > nginx.yaml
#To see other options, used kubectl run --help

```
2. `Task 2`: Add a Label `owner: yourname` and deploy the Pod

```
vi nginx.yaml
```
```YAML
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    #ADD THE FOLLOWING LINE
    owner: nic
    run: nginx
  name: nginx
spec:
  containers:
  - image: nginx
    name: nginx
    ports:
    - containerPort: 80
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```
```
kubectl apply -f nginx.yaml
```
Verify the label has been applied:
```
kubectl get pod nginx --show-labels
```
```
NAME    READY   STATUS    RESTARTS   AGE   LABELS
nginx   1/1     Running   0          82s   owner=nic,run=nginx
```

### Replica Sets and Deployments
Pods can be combined in a highly available Kubernetes object that ensures the intended number of running replicas is always met. This is achieved via the `ReplicaSet` controller. It is responsible for reconciling the current state of the cluster with the configuration stored in the Kubernetes database. That is, in the event of a node failure, the pods running on that node and part of a `ReplicaSet` will be restarted on an other node available, if it matches the conditions to run them. 
The `ReplicaSet` controller leverages Pod Templates to deploy individual pods when required. The `template` object in Kubernetes defines the `metadata` and `spec` to be used by the Pod when created. It is merely a wrapper around the `Pod` object. An example can be found below:

```YAML
template:
  metadata:
    labels:
      app: nginx
      owner: nic
  spec:
    containers:
      - name: mywebserver
        image: nginx
        ports:
          - containerPort: 80
```

`ReplicaSets` can be scaled imperatively using `kubectl scale`, or by editing their configuration. However, it is  recommended to use `Deployments` to manage `ReplicaSets`. The `Deployment` object in Kubernetes allows you to rollout new versions of your application in a smooth way by maintaining the expected level of availability. It also enables easy rollback and provides version history, making it the perfect ally to deploy stateless applications.

1. `Task 3`: Create a Deployment with the `nginx` image exposing port 80.
```
kubectl create deployment nginx-deploy --image=nginx --port=80 -o yaml --dry-run=client > nginx-deploy.yaml
```
2. `Task 4`: Create a Deployment with 2 replicas, and add a Label `owner: yourname` in the Pod template.
```
vi nginx-deploy.yaml
```
```YAML
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: nginx-deploy
  name: nginx-deploy
spec:
  #CHANGE THE FOLLOWING LINE TO CONTROL # OF REPLICA
  replicas: 2
  selector:
    matchLabels:
      app: nginx-deploy
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        #ADD THE FOLLOWING LINE
        owner: nic
        app: nginx-deploy
    spec:
      containers:
      - image: nginx
        name: nginx
        ports:
        - containerPort: 80
        resources: {}
status: {}
```
```
kubectl apply -f nginx-deploy.yaml
```
Verify the label has been applied:
```
kubectl get pod --show-labels
```
```
NAME                            READY   STATUS    RESTARTS   AGE   LABELS
nginx                           1/1     Running   0          25h   owner=nic,run=nginx
nginx-deploy-67f467bcd4-hpg5d   1/1     Running   0          12m   app=nginx-deploy,owner=nic,pod-template-hash=67f467bcd4
nginx-deploy-67f467bcd4-mpgh4   1/1     Running   0          12m   app=nginx-deploy,owner=nic,pod-template-hash=67f467bcd4
```
3. `Task 5`: Scale the number of replicas to 4.
```
kubectl scale deploy nginx-deploy --replicas=4
```
Verify the Deployment has been scaled successfully:
```
kubectl get pod --show-labels
```
```
NAME                            READY   STATUS    RESTARTS   AGE   LABELS
nginx                           1/1     Running   0          26h   owner=nic,run=nginx
nginx-deploy-67f467bcd4-hpg5d   1/1     Running   0          15m   app=nginx-deploy,owner=nic,pod-template-hash=67f467bcd4
nginx-deploy-67f467bcd4-mpgh4   1/1     Running   0          15m   app=nginx-deploy,owner=nic,pod-template-hash=67f467bcd4
nginx-deploy-67f467bcd4-rwqcn   1/1     Running   0          20s   app=nginx-deploy,owner=nic,pod-template-hash=67f467bcd4
nginx-deploy-67f467bcd4-v7j8x   1/1     Running   0          20s   app=nginx-deploy,owner=nic,pod-template-hash=67f467bcd4
```
4. `Task 6`: Update the application with a new container image.
```
vi nginx-deploy.yaml
```

```YAML
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: nginx-deploy
  name: nginx-deploy
spec:
  #CHANGE THE # OF REPLICAS TO 4
  replicas: 4
  selector:
    matchLabels:
      app: nginx-deploy
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        owner: nic
        app: nginx-deploy
    spec:
      containers:
      #CHANGE CONTAINER IMAGE HERE
      - image: vfiftyfive/flask_ascii
        name: nginx
        ports:
        - containerPort: 80
        resources: {}
status: {}
```
5. `Task 7`: Rollout and test the new application
```
kubectl apply -f nginx-deploy.yaml
```
```
kubectl rollout status deploy nginx-deploy
```
```
Waiting for deployment "nginx-deploy" rollout to finish: 3 out of 4 new replicas have been updated...
Waiting for deployment "nginx-deploy" rollout to finish: 3 out of 4 new replicas have been updated...
Waiting for deployment "nginx-deploy" rollout to finish: 3 out of 4 new replicas have been updated...
Waiting for deployment "nginx-deploy" rollout to finish: 3 out of 4 new replicas have been updated...
Waiting for deployment "nginx-deploy" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "nginx-deploy" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "nginx-deploy" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "nginx-deploy" rollout to finish: 3 of 4 updated replicas are available...
deployment "nginx-deploy" successfully rolled out
```
In order to test the application, we need to expose it as a service:
```
kubectl expose deploy nginx-deploy --type=ClusterIP --port=8000 --target-port=80
```
As `ClusterIP` is only accessible within the cluster boundaries, we test the application from another container performing a `curl` request:
```
kubectl run --image=curlimages/curl client --command -- sleep 20000
```
```
kubectl exec -it client -- curl nginx-deploy.default.svc.cluster.local:8000
```
```
 _   _      _ _         _____
| | | | ___| | | ___   |  ___| __ ___  _ __ ___
| |_| |/ _ \ | |/ _ \  | |_ | '__/ _ \| '_ ` _ \
|  _  |  __/ | | (_) | |  _|| | | (_) | | | | | |
|_| |_|\___|_|_|\___/  |_|  |_|  \___/|_| |_| |_|

 _  __     _                          _
| |/ /   _| |__   ___ _ __ _ __   ___| |_ ___  ___
| ' / | | | '_ \ / _ \ '__| '_ \ / _ \ __/ _ \/ __|
| . \ |_| | |_) |  __/ |  | | | |  __/ ||  __/\__ \
|_|\_\__,_|_.__/ \___|_|  |_| |_|\___|\__\___||___/
```
6. `Task 8`: Display application history and revert back to previous version.
```
kubectl rollout history deploy nginx-deploy
```
```
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
```
```
kubectl rollout undo deploy nginx-deploy 
```
7. Task 9: Check the original service has been restored.
```
kubectl exec -it client -- curl nginx-deploy.default.svc.cluster.local:8000 | grep title
```
```
<title>Welcome to nginx!</title>
```

### Passing data to applications
Kubernetes defines various patterns to pass data to the application. We are going to use the downward API and volumes to display dynamic content. 

1. `Task 9`: Add an environment variable in the pod template. It must be set your name, based on the label previously configured.
```
vi nginx-deploy.yaml
```
```YAML
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: nginx-deploy
  name: nginx-deploy
spec:
  replicas: 4
  selector:
    matchLabels:
      app: nginx-deploy
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        owner: nic
        app: nginx-deploy
    spec:
      containers:
      - image: vfiftyfive/flask_ascii
        name: nginx
        #ADD THE ENV SECTION BELOW
        env:
        - name: OWNER
          valueFrom:
            fieldRef:
              fieldPath: metadata.labels['owner']
        imagePullPolicy: Always
        ports:
        - containerPort: 80
        resources: {}
status: {}
```
2. `Task 10`: Force the replacement of all Pods in the deployment and test the app again
```
kubectl replace --force -f nginx-deploy.yaml
```
Wait until all Pods have have been replaced and run:
```
kubectl exec -it client -- curl nginx-deploy.default.svc.cluster.local:8000
```
```
       _       __        __          _   _               _ _ _
 _ __ (_) ___  \ \      / /_ _ ___  | | | | ___ _ __ ___| | | |
| '_ \| |/ __|  \ \ /\ / / _` / __| | |_| |/ _ \ '__/ _ \ | | |
| | | | | (__    \ V  V / (_| \__ \ |  _  |  __/ | |  __/_|_|_|
|_| |_|_|\___|    \_/\_/ \__,_|___/ |_| |_|\___|_|  \___(_|_|_)
```
