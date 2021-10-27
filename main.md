# Kubernetes Application Patterns Workshop

## Objective

The objective of this workshop is to explore Kubernetes StatefulSets and their use cases. Traditionally, Kubernetes applications are developed to run as Web applications. They don't often need to write persistent data to disk or need a stable identity. In the event of a node failure in Kubernetes, the controller plane  spawns new Pods to match the expected number of replicas. This is achieved by leveraging Kubernetes Deployment and ReplicaSets. The Pods they create are stateless and replaceable. When a Pod is created, it takes an arbitraty identity with an auto-generated unique name.
On the opposite, StatefulSets are Kubernetes first class objects providing Pods with a stable identity, and ensuring they can write to individual persistent volumes. They have additional characteristics we're going to highlight in this workshop.

# Create your first application
In this section we review some of the Kubernetes foundations for creating and deploying basic applications.

## Pods
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
Add sections as required:
```YAML
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    #ADD THE FOLLOWING LINE AND REPLACE "nic" WITH YOUR OWN NAME
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
output:
```
NAME    READY   STATUS    RESTARTS   AGE   LABELS
nginx   1/1     Running   0          82s   owner=nic,run=nginx
```

## Replica Sets and Deployments
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
Add sections as required:
```YAML
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: nginx-deploy
  name: nginx-deploy
spec:
  #ADD THE FOLLOWING LINE AND REPLACE "nic" WITH YOUR OWN NAME
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
output:
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
output:
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
Add sections as required:
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
output:
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
output:
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
output:
```
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
```
command:
```
kubectl rollout undo deploy nginx-deploy 
```
7. Task 9: Check the original service has been restored.
```
kubectl exec -it client -- curl nginx-deploy.default.svc.cluster.local:8000 | grep title
```
output:
```
<title>Welcome to nginx!</title>
```

### Passing data to applications
Kubernetes defines various patterns to pass data to the application. We are going to use volumes abd the downward API to display dynamic content. 

1. `Task 9`: Add an environment variable in the pod template. It must be set your name, based on the label previously configured.
```
vi nginx-deploy.yaml
```
Add sections as required:
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
output:
```
       _       __        __          _   _               _ _ _
 _ __ (_) ___  \ \      / /_ _ ___  | | | | ___ _ __ ___| | | |
| '_ \| |/ __|  \ \ /\ / / _` / __| | |_| |/ _ \ '__/ _ \ | | |
| | | | | (__    \ V  V / (_| \__ \ |  _  |  __/ | |  __/_|_|_|
|_| |_|_|\___|    \_/\_/ \__,_|___/ |_| |_|\___|_|  \___(_|_|_)
```

Of course, the output should match your name :-)

Let's explore another pattern for passing data to the application. Now we're going to use an `init container` to pass data to the main container by writing to a shared volume.

3. `Task 11`: Add an init container to the Pod template that writes the message to a shared file, and replace the existing Pods in the Deployment.
```
vi nginx-deploy.yaml
```
Add sections as required:
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
      #ADD THE FOLLOWING INIT CONTAINER SECTION
      initContainers:
      - name: copymessage
        image: alpine
        command: ['sh', '-c', 'echo $HOSTNAME Was Here > /mnt/message/text']
        volumeMounts:
        - name: message
          mountPath: /mnt/message
      containers:
      - image: vfiftyfive/flask_ascii
        name: nginx
        env:
        - name: OWNER
          valueFrom:
            fieldRef:
              fieldPath: metadata.labels['owner']
        #ADD THE FOLLOWING VOLUMEMOUNTS SECTION
        volumeMounts:
        - name: message
          mountPath: /mnt/message
        imagePullPolicy: Always
        ports:
        - containerPort: 80
        resources: {}
      #ADD THE FOLLOWING VOLUMES SECTION
      volumes:
      - name: message
        emptyDir: {}
status: {}
```
```
kubectl replace --force -f nginx-deploy.yaml
```
`emptyDir` gives every individual Pod a temporary local in-memory storage. The message displayed should now be different for every Pod, as it is based on individual Pod hostname. Let's check this by using the client Pod again.

4. `Task 12`: Query the Kubernetes service multiple times and record the result.
```
kubectl exec -it client -- curl nginx-deploy.default.svc.cluster.local:8000
```
output:
```
             _                     _            _                    __
 _ __   __ _(_)_ __ __  __      __| | ___ _ __ | | ___  _   _       / /_
| '_ \ / _` | | '_ \\ \/ /____ / _` |/ _ \ '_ \| |/ _ \| | | |_____| '_ \
| | | | (_| | | | | |>  <_____| (_| |  __/ |_) | | (_) | |_| |_____| (_) |
|_| |_|\__, |_|_| |_/_/\_\     \__,_|\___| .__/|_|\___/ \__, |      \___/
       |___/                             |_|            |___/
 _  _        ____ _____ ____  _  _       _ ____  ___              _
| || |   ___| ___|___  | ___|| || |   __| | ___|/ _ \       _ __ | |__   __ _
| || |_ / __|___ \  / /|___ \| || |_ / _` |___ \ (_) |_____| '_ \| '_ \ / _` |
|__   _| (__ ___) |/ /  ___) |__   _| (_| |___) \__, |_____| | | | | | | (_| |
   |_|  \___|____//_/  |____/   |_|  \__,_|____/  /_/      |_| |_|_| |_|\__, |
                                                                        |___/
  ___      _  __        __          _   _
 / _ \  __| | \ \      / /_ _ ___  | | | | ___ _ __ ___
| (_) |/ _` |  \ \ /\ / / _` / __| | |_| |/ _ \ '__/ _ \
 \__, | (_| |   \ V  V / (_| \__ \ |  _  |  __/ | |  __/
   /_/ \__,_|    \_/\_/ \__,_|___/ |_| |_|\___|_|  \___|
```
Repeat the query. Every request leads to a different result. This is because every Pod backing the service VIP has a now a different message configured.

In summary, while `emptyDir` provides distinct storage for every Pod in a deployment, its lifecycle is tied to the Pod. That is, if the Pod stops, the volume is deleted.
When persistent data is required, Kubernetes provides the `PersistentVolume` object. But remember how we define Pod configuration in a `Deployment`: we use a template. So unless we use `RWX` (Read/Write Multiple, aka NFS or shared storage), `PersistentVolumes` won't work well with Deployments. The first Pod will claim the volume, and all remaining ones will fail to start.

# Exploring StatefulSets

In this section we take a closer look at Kubernetes `StatefulSets`. They were originally called "PetSets", which clearly makes you understand their use case. `StatefulSets` were introduced in Kubernetes 1.5
and have the following properties:
- Each Pod replica gets a stable hostname with a unique index. For example if you create a `StatefulSet` named "mymongodb" configured with 3 Pod replicas, these pods will be named: `mymongodb-0`, `mymongodb-1`, and `mymongodb-2`.
- Each Pod replica is created in order, from the lowest to highest index. They are not created in parallel, Pod `n` is only created after Pod `n-1` is up and running. This also applies when scaling up or upgrading `StatefulSets`.
- When a `StatefulSet` is scaled down or deleted, the reversed ordered is used to process the operation, from the highest to lowest Pod index.

Like `Deployments`, `StatefulSets` allows for phased rollout. This guarantees a minimum of Pod replicas will remain up and running while the others get updated. This is done using the rules mentioned in the previous paragraph. However, there are a couple of additional parameters you can leveraged depending on your goals:

- `Parallel Deployments`: When `.spec.podManagementPolicy` is set to `Parallel`, the control plane doesn't wait for Pods to become ready or deleted before proceeding with the remaining ones. If you don't need sequential processing for your `StatefulSets`, this option will speed up operations.
- `Partitioned Updates`: You can set a particular ordinal index as the Pods partition limit. For example, if you set the number "5" as value, All pods with an index >= 5 will be updated, while the other ones are not touched. This remains true even if these Pods are deleted. The control plane will recreate them at the previous version.

Now let's get some hands-on and deploy MongoDB as a `StatefulSet` of 3 replicas. We'll configure MongoDB manually. It is usually recommended to manage Databases deployed on Kubernetes with specific Kubernetes Operators, but this would add another layer complexity to this lab. I'm not sure everyone is ready for this yet!

Before deploying the `StatefulSet`, we are going to install Ondat. In the same way we have added a Pod template in the `Deployment` configuration, we'll need to add a `volumeClaimTemplate` in the `StatefulSet` configuration. This is because we want every MongoDB instance to mount its own persisten volume and not share it with other Pods. Ondat provides an automated way to provision these volumes by simply referencing a `StorageClass` in the `volumeClaimTemplate`. It also adds features labels to the persistent volumes provisioned, such as replication, encryption, thing provisioning, compresstion, etc. We are also going to explore these capabilities.

1. `Task 13`: [Install Ondat](https://docs.ondat.io/docs/self-eval/)
```
curl -sL https://storageos.run | bash
```
This creates a new `StorageClass` named `fast`

2. `Task 14`: Deploy MongoDB `StatefulSet` and the corresponding headless service.
```
vi mongodb-sts.yaml
```
```YAML
apiVersion: v1
kind: Service
metadata:
  name: mongodb
spec:
  ports:
  - port: 27017
    name: peer
  clusterIP: None
  selector:
    app: mongo
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb
  labels:
    app: mongo
spec:
  serviceName: "mongodb"
  replicas: 3
  selector:
    matchLabels:
      app: mongo
  template:
    metadata:
      labels:
        app: mongo
    spec:
      containers:
      - name: mongodb
        image: mongo:3.4.1
        volumeMounts:
        - name: database
          mountPath: /data/db
        command:
        - mongod
        - --replSet
        - rs0
        ports:
        - containerPort: 27017
          name: peer
  volumeClaimTemplates:
  - metadata:
      name: database
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: "fast"
      resources:
        requests:
          storage: 1Gi
```
```
kubectl apply -f mongodb-sts.yaml
```
3. `Task 15`: Check Pods, StatefulSets, PVC, PV.
```
kubectl get pods,sts,svc,pvc,pv  
```
output:
```                                                                                                               
NAME            READY   STATUS    RESTARTS   AGE
pod/mongodb-0   1/1     Running   0          2m27s
pod/mongodb-1   1/1     Running   0          2m9s
pod/mongodb-2   1/1     Running   0          2m4s

NAME                       READY   AGE
statefulset.apps/mongodb   3/3     2m38s

NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)     AGE
service/kubernetes     ClusterIP   10.43.0.1       <none>        443/TCP     20d
service/mongodb        ClusterIP   None            <none>        27017/TCP   2m38s
service/nginx-deploy   ClusterIP   10.43.211.190   <none>        8000/TCP    26h

NAME                                       STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/database-mongodb-0   Bound    pvc-f6a9ee16-a81b-4001-a2ce-a6710bf235cd   1Gi        RWO            fast           47m
persistentvolumeclaim/database-mongodb-1   Bound    pvc-8c362db2-e53e-4d3f-afb5-7870afbbb946   1Gi        RWO            fast           47m
persistentvolumeclaim/database-mongodb-2   Bound    pvc-448601a9-98f2-43d7-b8c8-bb3641ee8ff1   1Gi        RWO            fast           47m

NAME                          CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                STORAGECLASS      AGE
persistentvolume/pvc-448...   1Gi        RWO            Delete           Bound    default/database-mongodb-2           fast              29m
persistentvolume/pvc-8c3...   1Gi        RWO            Delete           Bound    default/database-mongodb-1           fast              29m
persistentvolume/pvc-f6a...   1Gi        RWO            Delete           Bound    default/database-mongodb-0           fast              29m
```
Also check the Ondat volumes:
```
kubectl exec -it -n kube-system cli -- storageos get volumes -n default
```
output:
```
NAMESPACE  NAME                                      SIZE     LOCATION                    ATTACHED ON        REPLICAS  AGE
default    pvc-39b61ee9-a475-435b-8ca4-816adc5f7504  1.0 GiB  nic-temp-master-1 (online)  nic-temp-master-1  0/0       1 day ago
default    pvc-74405f09-65ef-4e54-af52-43c619612ab2  1.0 GiB  nic-temp-worker1 (online)   nic-temp-worker1   0/0       1 day ago
default    pvc-118e869e-513a-41f0-b789-adcc9b89775b  1.0 GiB  nic-temp-worker4 (online)   nic-temp-worker4   0/0       1 day ago
```
4. `Task 16`: Set MongoDB Replication
```
kubectl exec -it mongodb-0 mongo
```
Once connected to the database, type:
```
>rs.initiate({ _id: "rs0", members: [ { _id:0, host: "mongodb-0.mongodb:27017" } ]});
...
```
```
>rs.add("mongodb-1.mongodb:27017");
...
```
```
>rs.add("mongodb-2.mongodb:27017");
...
```
Check the output of `rs.status()`. You should have the 3 members listed.

The next step is to deploy an application that will make use of this database. For this, we're going to deploy a web app that displays information of random Marvel characters. The list of characters is retrieved from the Marvel APIs, and stored in a new MongoDB collection named "characters". 
Let's first run a Kubernetes Job to perform fill the database with characters data:

5. `Task 17`: Add MongoDB documents

First, let's create a boilerplate for the Kubernetes `Job` object using the container image that is going to populate the database.
```
kubectl create job add-data-to-mongodb --image=vfiftyfive/marvel_init_db --dry-run=client -o yaml > job.yaml
```
```
vim job.yaml
```
Add sections as required:
```YAML
apiVersion: batch/v1
kind: Job
metadata:
  creationTimestamp: null
  name: add-data-to-mongodb
spec:
  template:
    metadata:
      creationTimestamp: null
    spec:
      containers:
      #ADD THE ENV SECTION TO CONFIGURE THE JOB PROCESS
      #AND SET THE TARGET DATABASE INSTANCE
      - env:
        - name: MONGO_HOST
          value: mongodb-0.mongodb.default.svc.cluster.local
        - name: OFFSET
          value: "200"
        image: vfiftyfive/marvel_init_db
        name: add-data-to-mongodb
        resources: {}
      restartPolicy: Never
  backoffLimit: 4
status: {}
```
```
kubectl apply -f job.yaml
```
output:
```
NAME                        READY   STATUS      RESTARTS   AGE
add-data-to-mongodb-trw8w   0/1     Completed   0          49s
```
The job status will initially be displayed as `Running` and will change to `Completed` once the data has been ingested.

6. `Task 18`: Check the data is present in the database
```
kubectl run -it --rm --image vfiftyfive/utilities:first mongo-client -- mongosh "mongodb://mongodb-0.mongodb.default.svc.cluster.local" 
```
output:
```
...
rs0 [direct: primary] test>
```
commands:
```
> use marvel
...
> show collections
...
> db.characters.find({}).count()
```
outputs:
```
128
```
128 documents have been added to our database. If you want to further check the content, you can run:
```
> db.characters.find({})
...
#Then you can type "it" + 'enter key' to proceed with the next page. Repeat until "no cursor" is displayed instead of "it"
```

7. `Task 19`: Deploy the frontend application
For this, we create a `Deployment` boilerplate and add extra configuration elements. The applications is using `gunicorn` with `Flask`, providing a stateless HTTP frontend to present data from the MongoDB characters collection.
```
kubectl create deployment marvel-frontend --port 80 --image vfiftyfive/flask_marvel --dry-run=client -o yaml > marvel_deployment.yaml
```
```
vi marvel_deployment.yaml
```
Add sections as required:
```YAML
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: marvel-frontend
  name: marvel-frontend
spec:
  #CHANGE REPLICAS TO 3
  replicas: 3
  selector:
    matchLabels:
      app: marvel-frontend
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: marvel-frontend
    spec:
      containers:
      #ADD THE ENV SECTION TO SET THE TARGET DATABASE INSTANCE
      - env:
        - name: MONGO_HOST
          value: mongodb-0.mongodb.default.svc.cluster.local
        image: vfiftyfive/flask_marvel
        name: flask-marvel-fds72
        ports:
        - containerPort: 80
        resources: {}
status: {}
```
As done before, let's expose this awesome application as a Kuberenetes service
```
kubectl expose deploy marvel-frontend --type=ClusterIP --port=8080 --target-port=80
```
8. `Task 20`: Check Kubernetes Service
```
kubectl get svc marvel-frontend
```
output:
```
NAME              TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
marvel-frontend   ClusterIP   10.43.44.204   <none>        8080/TCP   2m8s
```
9. `Task 21`: Check Application is connected to the backend MongoDB collection
There are 2 ways to access the application. 
- If you run `kubectl` locally on your computer:
```
kubectl port-forward svc/marvel-frontend 8080
```
Then open your browser and navigate to the URL: `http://localhost:8080`
![](https://nic-ondat.s3.eu-west-2.amazonaws.com/Screen+Shot+2021-10-27+at+2.50.28+PM.png){:height="50%" width="50%"}



Set ondat PVC replicas



