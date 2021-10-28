# Kubernetes Application Patterns Workshop

## Objective

The objective of this workshop is to explore Kubernetes StatefulSets and their use cases. Traditionally, Kubernetes applications are developed to run as Web applications. They don't often need to write persistent data to disk or need a stable identity. In the event of a node failure in Kubernetes, the controller plane  spawns new Pods to match the expected number of replicas. This is achieved by leveraging Kubernetes Deployment and ReplicaSets. The Pods they create are stateless and replaceable. When a Pod is created, it takes an arbitrary identity with an auto-generated unique name.
On the other hand, StatefulSets are Kubernetes first class objects providing Pods with a stable identity, and ensuring they can write to individual persistent volumes. They have additional characteristics we're going to highlight in this workshop.

# Step 0 - Bring your Kubernetes cluster up with `kubeadm`
On your Master node, run:
```
sudo kubeadm init --pod-network-cidr=10.244.0.0/16

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
On each of your worker nodes, run:
```
sudo kubeadm join <Cluster-IP>:6443 --token <Token> \
        --discovery-token-ca-cert-hash sha256:<hash>
```
If you lose your cluster token, create a new one on the master:
```
kubeadm token create --print-join-command
```

# Additional Configuration
Configure `Vim` settings as below to avoid any tabulation issue when doing copy/paste from the guide to your console.
```
vi ~/.vimrc
```
```
syntax on
set background=dark
set tabstop=2
set shiftwidth=2
set expandtab
```
Feel free to use `Vim` instead of `Vi` in the rest of the document.

# Create your first application
In this section we review some of the Kubernetes foundations required for creating and deploying basic applications.

## Pods
An easy way to create a pod is to use `kubectl run` in dry-run mode and redirect the result to a file. You can then alter the configuration as needed.

1. `Task 1`: Create a Pod with the `nginx` image exposing container port 80.

```
kubectl run nginx --image=nginx --port=80 --dry-run=client -oyaml > nginx.yaml
#To see other options, used kubectl run --help

```
2. `Task 2`: Add a Label `owner: yourname` and deploy the Pod.

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
Output:
```
NAME    READY   STATUS    RESTARTS   AGE   LABELS
nginx   1/1     Running   0          82s   owner=nic,run=nginx
```

## Replica Sets and Deployments
`Pods` can be combined in a highly available Kubernetes object that ensures the intended number of running replicas is always met. This is achieved via the `ReplicaSet` controller. It is responsible for reconciling the current state of the cluster with the configuration stored in the Kubernetes database. That is, in the event of a node failure, the `Pods` running on that node and part of a `ReplicaSet` will be restarted on another node available, if it matches the conditions to run them. 
The `ReplicaSet` controller leverages Pod Templates to deploy individual pods when required. The `template` object in Kubernetes defines the `metadata` and `spec` to be used by the `Pod` when created. It is merely a wrapper around the `Pod` object. An example can be found below:

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
  #ADD THE FOLLOWING LINE
  replicas: 2
  selector:
    matchLabels:
      app: nginx-deploy
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        #ADD THE FOLLOWING LINE AND REPLACE "nic" WITH YOUR OWN NAME
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
Output:
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
Output:
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
5. `Task 7`: Rollout and test the new application.
```
kubectl apply -f nginx-deploy.yaml
```
```
kubectl rollout status deploy nginx-deploy
```
Output:
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
Output:
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
Output:
```
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
```
Command:
```
kubectl rollout undo deploy nginx-deploy 
```
7. Task 9: Check the original service has been restored.
```
kubectl exec -it client -- curl nginx-deploy.default.svc.cluster.local:8000 | grep title
```
Output:
```
<title>Welcome to nginx!</title>
```

### Passing data to applications
Kubernetes defines various patterns to pass data to applications. We are going to use volumes and the downward API to display dynamic content. 

1. `Task 9`: Add an environment variable in the Pod Template. It must set your name, based on the label previously configured.
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
2. `Task 10`: Force the replacement of all `Pods` in the deployment and test the app again.
```
kubectl replace --force -f nginx-deploy.yaml
```
Wait until all Pods have have been replaced and run:
```
kubectl exec -it client -- curl nginx-deploy.default.svc.cluster.local:8000
```
Output:
```
       _       __        __          _   _               _ _ _
 _ __ (_) ___  \ \      / /_ _ ___  | | | | ___ _ __ ___| | | |
| '_ \| |/ __|  \ \ /\ / / _` / __| | |_| |/ _ \ '__/ _ \ | | |
| | | | | (__    \ V  V / (_| \__ \ |  _  |  __/ | |  __/_|_|_|
|_| |_|_|\___|    \_/\_/ \__,_|___/ |_| |_|\___|_|  \___(_|_|_)
```

Of course, the output should match your name :-)

Let's explore another pattern for passing data to the application. We are now going to use an `init container` to pass data to the main container by writing to a shared volume.

3. `Task 11`: Add an init container to the Pod Template that writes the message to a shared file, and replace the existing `Pods` in the Deployment.
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
      #ADD THE FOLLOWING INITCONTAINERS SECTION
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
`emptyDir` gives every individual Pod a temporary local in-memory storage. The message displayed should now be different for every Pod, as it is based on the individual Pod hostname. Let's check this by using the client Pod again.

4. `Task 12`: Query the Kubernetes service multiple times and record the result.
```
kubectl exec -it client -- curl nginx-deploy.default.svc.cluster.local:8000
```
Output:
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
Repeat the query. Every request leads to a different result. This is because every `Pod` backing the service VIP now has a different message configured.

In summary, while `emptyDir` provides individual storage for every `Pod` in a deployment, its lifecycle is tied to the `Pod`. That is, if the `Pod` stops, the volume is deleted.
When persistent data is required, Kubernetes provides the `PersistentVolumeClaim` object. But remember how we define Pod configuration in a `Deployment`: we use a template. So unless we use `RWX` (Read/Write Multiple, aka NFS or shared storage), `PersistentVolumes` won't work well with Deployments. The first Pod will claim the volume, and all remaining ones will fail to start.

# Exploring StatefulSets

In this section we take a closer look at Kubernetes `StatefulSets`. They were originally called "PetSets", which clearly makes you understand their use case. `StatefulSets` were introduced in Kubernetes 1.5
and have the following properties:
- Each `Pod` replica gets a stable hostname with a unique index. For example if you create a `StatefulSet` named "mymongodb" configured with 3 `Pod` replicas, these `Pods` will be named: `mymongodb-0`, `mymongodb-1`, and `mymongodb-2`.
- Each `Pod` replica is created in order, from the lowest to highest index. They are not created in parallel, `Pod` `n` is only created after `Pod` `n-1` is up and running. This also applies when scaling up or upgrading `StatefulSets`.
- When a `StatefulSet` is scaled down or deleted, the reversed order is used to process the operation, from the highest to lowest `Pod` index.

Like `Deployments`, `StatefulSets` allows for phased rollout. This guarantees a minimum of `Pod` replicas will remain up and running while the others get updated. This is done using the rules mentioned in the previous paragraph. However, there are a couple of additional parameters you can leveraged depending on your goals:

- `Parallel Deployments`: When `.spec.podManagementPolicy` is set to `Parallel`, the control plane doesn't wait for `Pods` to become ready or deleted before proceeding with the remaining ones. If you don't need sequential processing for your `StatefulSets`, this option will speed up operations.
- `Partitioned Updates`: You can set a particular ordinal index as the `Pods` partition limit. For example, if you set the number "5" as value, All `Pods` with an index >= 5 will be updated, while the other ones are not touched. This remains true even if these `Pods` are deleted. The control plane will recreate them at the previous version.

Now let's get hands-on and deploy MongoDB as a `StatefulSet` of 3 replicas. We'll configure MongoDB manually. It is usually recommended to manage Databases deployed on Kubernetes with specific Kubernetes Operators, but this would add another layer of complexity to this lab. I'm not sure everyone is ready for this yet!

Before deploying the `StatefulSet`, we are going to install Ondat. In the same way we have added a Pod Template in the `Deployment` configuration, we'll need to add a `volumeClaimTemplate` in the `StatefulSet` configuration. This is because we want every MongoDB instance to mount its own `PeristentVolume` and not share it with other `Pods`. Ondat provides an automated way to provision these volumes by simply referencing a `StorageClass` in the `volumeClaimTemplate`. It also adds features labels to the persistent volumes provisioned, such as replication, encryption, thin provisioning, compression, etc. We are also going to explore some of these capabilities.

1. `Task 13`: [Install Ondat](https://docs.ondat.io/docs/self-eval/).
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
3. `Task 15`: Check `Pods`, `StatefulSets`, `PersistentVolumesClaims` (`PVC`), `PersistentVolumes` (`PV`).
```
kubectl get pods,sts,svc,pvc,pv  
```
Output:
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
Output:
```
NAMESPACE  NAME              SIZE     LOCATION                    ATTACHED ON        REPLICAS  AGE
default    pvc-39b61ee9-...  1.0 GiB  nic-temp-master-1 (online)  nic-temp-master-1  0/0       1 day ago
default    pvc-74405f09-...  1.0 GiB  nic-temp-worker1 (online)   nic-temp-worker1   0/0       1 day ago
default    pvc-118e869e-...  1.0 GiB  nic-temp-worker4 (online)   nic-temp-worker4   0/0       1 day ago
```
4. `Task 16`: Configure MongoDB Replication.
```
kubectl exec -it mongodb-0 mongo
```
Once connected to the database, type:
```
rs.initiate({ _id: "rs0", members: [ { _id:0, host: "mongodb-0.mongodb:27017" } ]});
```
```
rs.add("mongodb-1.mongodb:27017");
```
```
rs.add("mongodb-2.mongodb:27017");
```
Check the output of `rs.status()`. You should have the 3 members listed. Type `exit` when done.

The next step is to deploy an application that will make use of this database. For this, we're going to deploy a web app that displays information of random Marvel characters. The list of characters is retrieved from the Marvel APIs, and stored in a new MongoDB collection named "characters". 
Let's first run a Kubernetes Job to fill the database with characters data:

5. `Task 17`: Add MongoDB Documents.

First, let's create a boilerplate for the Kubernetes `Job` object using the container image that is going to populate the database.
```
kubectl create job add-data-to-mongodb --image=vfiftyfive/marvel_init_db --dry-run=client -o yaml > job.yaml
```
```
vi job.yaml
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
```
kubectl get jobs
```
Output:
```
NAME                  COMPLETIONS   DURATION   AGE
add-data-to-mongodb   1/1           26s        2m37s
```
The `Pod` status will initially be displayed as `Running` and will change to `Completed` once the data has been ingested.

6. `Task 18`: Check the data is present in the database.
```
kubectl run -it --rm --image vfiftyfive/utilities:first mongo-client -- mongosh "mongodb://mongodb-0.mongodb.default.svc.cluster.local" 
```
Output:
```
...
rs0 [direct: primary] test>
```
Command:
```
> use marvel
...
> show collections
...
> db.characters.find({}).count()
```
Output:
```
128
```
128 documents have been added to our database. If you want to further check the content, you can run:
```
> db.characters.find({})
...
#Then you can type "it" + 'enter key' to proceed with the next page. Repeat until "no cursor" is displayed instead of "it"
```
Type `exit` when you're done.

7. `Task 19`: Deploy the frontend application.

Let's create a `Deployment` boilerplate and add extra configuration elements. The application is using `gunicorn` with `Flask`, providing a stateless HTTP frontend to present data from the MongoDB characters collection.
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
Now let's deploy the application:
```
kubectl apply -f marvel_deployment.yaml
```
And verify all 3 pods have been deployed and are running:
```
kubectl get pods | grep marvel
```
Output:
```
marvel-frontend-69c57f7ff5-m2sd8   1/1     Running     0          1m
marvel-frontend-69c57f7ff5-rfv8s   1/1     Running     0          1m
marvel-frontend-69c57f7ff5-rxn7f   1/1     Running     0          1m
```
As done before, let's expose this awesome application as a Kubernetes service.

8. `Task 20`: Expose the application as Kubernetes `Service`.

If the Kubernetes nodes are all directly accessible on any unprivileged port, you can use `NodePort`:
```
 kubectl expose deploy marvel-frontend --type=NodePort --port=8080 --target-port=80
```
If the Kubernetes nodes are not directly accessible, you can use `ClusterIP`:
```
kubectl expose deploy marvel-frontend --type=ClusterIP --port=8080 --target-port=80
```
Let's validate the configuration:
```
kubectl get svc marvel-frontend
```
Output for `NodePort`
```
NAME              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
marvel-frontend   NodePort    10.43.39.82     <none>        8080:31730/TCP   23m
```
Output for `ClusterIP`
```
NAME              TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
marvel-frontend   ClusterIP   10.43.44.204   <none>        8080/TCP   2m8s
```

9. `Task 21`: Check the application is connected to the backend MongoDB collection.

There are 3 ways to access the application:
- If you have used `NodePort`, then simply connect to any node on the unprivileged port displayed in the service description. In the example of `Task 20`, you can open your browser and connect to `http://<kubernetes-node>:31730`
- If you are running `kubectl` locally on your computer:
```
kubectl port-forward svc/marvel-frontend 8080
```
Then open your browser and navigate to the URL: `http://localhost:8080`

<a href="https://nic-ondat.s3.eu-west-2.amazonaws.com/Screen+Shot+2021-10-27+at+2.50.28+PM.png"><img src="https://nic-ondat.s3.eu-west-2.amazonaws.com/Screen+Shot+2021-10-27+at+2.50.28+PM.png" width="800"/></a>

- If you are running `kubectl` locally on your computer:

The first part is identical, run:
```
kubectl port-forward svc/marvel-frontend 8080
```
Create and `SSH` tunnel between your local machine and the remote `kubectl` machine:
```
ssh -L 8080:localhost:8080 your_user@remote_machine_address -N
```
Output example:
```
The authenticity of host '178.79.180.33 (178.79.180.33)' can't be established.
ECDSA key fingerprint is SHA256:P64TxSvhnttm8176JFdYzVUY4w5NtAx8auUr7IrFdyI.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
```
The command won't return. THIS IS EXPECTED.

You can open your browser and navigate to the URL: `http://localhost:8080`

# Distributed Data Availability

It is now time to try to break things to see how Ondat helps increase availability in a kube-native way. We are first going to add volume replicas to our stateful application and then we'll introduce some failure scenarios.

1. `Task 22`: Configure each Ondat `PVC` volume with 2 replicas.
```
kubectl exec -it -n kube-system cli -- storageos get volumes -n default
```
Output:
```
NAMESPACE  NAME             SIZE     LOCATION                   ATTACHED ON       REPLICAS  AGE
default    pvc-97b3a55f...  1.0 GiB  nic-temp-worker2 (online)  nic-temp-worker2  0/0       6 hours ago
default    pvc-1489ddf1...  1.0 GiB  nic-temp-worker2 (online)  nic-temp-worker2  0/0       6 hours ago
default    pvc-ff049c9d...  1.0 GiB  nic-temp-worker4 (online)  nic-temp-worker4  0/0       6 hours ago
```
There is currently 0 replicat configured.

Command:
```
kubectl label pvc --all storageos.com/replicas="2" --overwrite
```
Ondat utilizes Kubernetes `labels` primitives to enable and configure features. In that particular example, we enabled 2 replicas for every Ondat `PVC` present in the default namespace.
Let's verify this by running the check command again:
```
kubectl exec -it -n kube-system cli -- storageos get volumes -n default
```
Output:
```
NAMESPACE  NAME             SIZE     LOCATION                   ATTACHED ON       REPLICAS  AGE
default    pvc-97b3a55f...  1.0 GiB  nic-temp-worker2 (online)  nic-temp-worker2  2/2       6 hours ago
default    pvc-1489ddf1...  1.0 GiB  nic-temp-worker2 (online)  nic-temp-worker2  2/2       6 hours ago
default    pvc-ff049c9d...  1.0 GiB  nic-temp-worker4 (online)  nic-temp-worker4  2/2       6 hours ago
```
>`LOCATION` shows the node where the primary volume is currently attached

>`ATTACHED ON` shows the node where the `Pod` is currently running

We can notice the primary volume is attached to the same node as the `Pod` is running on. Ondat always tries to co-locate the primary volume with the node where the `Pod` is running. Sometimes this may not be possible. In that case, the volume can be accessed remotely by the `Pod` because Ondat presents a distributed data overlay to the `Pod`. It doens't matter if the `PersistentVolume` is physically local to the `Pod` or not. Ondat takes care of connecting the dots.

The next step is to force the `Pod` and the primary volume to sit on different nodes. We'll then kill the node where the primary volume is located. The expectation is that another volume will instantly be promoted and that a new replica will be created in order to match the required number of replicas - 3 in our case. This should happens without the `Pod` losing access to the data at anytime.

2. `Task 23`: Restart `Pod` on another node.

An easy way to evict the Pod from its node is to cordon the node, making it non-schedulable, and delete the `Pod`. As a result, The `StatefulSet` controller will restart the `Pod` on another available node.
But first, find the `Pod` and attached `PVC` you want to work with.
```
kubectl get pvc
```
Output:
```
NAME                 STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE     VOLUMEMODE
database-mongodb-0   Bound    pvc-1489ddf1-fcb9-417a-9906-c2de157f2247   1Gi        RWO            fast           7h46m   Filesystem
database-mongodb-1   Bound    pvc-ff049c9d-97db-44e7-9193-0b72e4269aea   1Gi        RWO            fast           7h46m   Filesystem
database-mongodb-2   Bound    pvc-97b3a55f-7f3b-45de-a102-b2837a9ce7a7   1Gi        RWO            fast           7h46m   Filesystem
```
In my case I want to work with Pod `database-mongodb-0`, and the attached volume is `pvc-1489ddf1-fcb9-417a-9906-c2de157f2247`. Both are running on node `nic-temp-worker2`, as displayed in the previous task.
```
#Replace the node name below with your node name
kubectl cordon nic-temp-worker2 
```
```
kubectl get nodes
```
Output:
```
NAME                STATUS                     ROLES                      AGE   VERSION
nic-temp-master-1   Ready                      controlplane,etcd,worker   22d   v1.20.11
nic-temp-worker1    Ready                      worker                     22d   v1.20.11
nic-temp-worker2    Ready,SchedulingDisabled   worker                     22d   v1.20.11
nic-temp-worker3    Ready                      worker                     22d   v1.20.11
nic-temp-worker4    Ready                      worker                     22d   v1.20.11
```
Just after typing the following command, refresh the Marvel app.

Command:
```
kubectl delete pod database-mongodb-0
```
The app should freeze a little bit and go back to normal a couple of seconds later. You can monitor what is happening in Kubernetes:
```
kubectl get pods | grep mongo
```
```
NAME                               READY   STATUS              RESTARTS   AGE
mongodb-0                          0/1     ContainerCreating   0          10s
mongodb-1                          1/1     Running             0          8h
mongodb-2                          1/1     Running             0          8h
```
Since we didn't configure the application to connect to all MongoDB nodes (only `mongodb-0`), it has to wait for the new `mongodb-0` `Pod` to be provisioned again. The application should quickly come back up online and reconnect to the `PersistentVolume`, which is now remote.
```
kubectl exec -it -n kube-system cli -- storageos get volumes -n default
```
Output:
```
NAMESPACE  NAME                                      SIZE     LOCATION                   ATTACHED ON       REPLICAS  AGE
default    pvc-97b3a55f-7f3b-45de-a102-b2837a9ce7a7  1.0 GiB  nic-temp-worker2 (online)  nic-temp-worker2  2/2       8 hours ago
default    pvc-1489ddf1-fcb9-417a-9906-c2de157f2247  1.0 GiB  nic-temp-worker2 (online)  nic-temp-worker4  2/2       8 hours ago
default    pvc-ff049c9d-97db-44e7-9193-0b72e4269aea  1.0 GiB  nic-temp-worker4 (online)  nic-temp-worker4  2/2       8 hours ago
```
We can notice that `mongodb-0` is now running on node `nic-temp-worker4`, but the volume is still attached to `nic-temp-worker2`. The `Pod` believes it is accessing its `PersistentVolume` locally on node `nic-temp-worker4`. But in reality the Ondat engine acts as an interface and establishes the connection to the backend volume located on node `nic-temp-worker2`. Without Ondat, the `Pod` wouldn't have been able to restart on another node, since its `PersistentVolume` wouldn't have been reachable.

3. `Task 24`: Uncordon the Kubernetes node. 
```
#Replace the node name below with your node name
kubectl uncordon nic-temp-worker2
```
Validate the node is uncordonned:
```
kubectl get nodes
```
Output:
```
NAME                STATUS   ROLES                      AGE   VERSION
nic-temp-master-1   Ready    controlplane,etcd,worker   22d   v1.20.11
nic-temp-worker1    Ready    worker                     22d   v1.20.11
nic-temp-worker2    Ready    worker                     22d   v1.20.11
nic-temp-worker3    Ready    worker                     22d   v1.20.11
nic-temp-worker4    Ready    worker                     22d   v1.20.11
```
4. `Task 25`: Reboot the node serving the primary volume.

In the previous task, you've identified the node where the primary volume is attached. In my case, this is node `nic-temp-worker2`. Let's reboot this node.
```
ssh user@node
shutdown -r now
```
While the node reboots, refresh the Marvel application. There should not be any lag this time as the `Pod` is still available - However, we just killed the volume attached to it. An available replica is promoted and a new one is created to match the required number of replicas.
```
kubectl exec -it -n kube-system cli -- storageos get volumes -n 
```
```
NAMESPACE  NAME             SIZE     LOCATION                   ATTACHED ON       REPLICAS  AGE
default    pvc-97b3a55f...  1.0 GiB  nic-temp-worker3 (online)                    2/2       8 hours ago
default    pvc-1489ddf1...  1.0 GiB  nic-temp-worker1 (online)  nic-temp-worker4  2/2       8 hours ago
default    pvc-ff049c9d...  1.0 GiB  nic-temp-worker4 (online)  nic-temp-worker4  2/2       8 hours ago
```
The primary volume is now located on `nic-temp-worker1`. We didn't experience any disruption at any time, including while the volume was promoted! The volume continues to be presented to the node `nic-temp-worker4` through a remote protocol enabled by the Ondat engine.

*That was the last task, CONGRATULATIONS FOR MAKING IT TO THE END!!!*

*Author*: Nic Vermandé

*Twitter*: @nvermande

*Linkedin*: https://www.linkedin.com/in/vnicolas/
