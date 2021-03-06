# MongoDB Deployment 

Our first step is to create a deployment for MongoDB (mongodb-deployment.yml). Here I'm using the latest version of mongoDB. 

Looking at dockerhub (https://hub.docker.com/_/mongo), the default port for mongodb is 27017. Furthermore, it needs two environment variables:
* MONGO_INITDB_ROOT_USERNAME
* MONGO_INITDB_ROOT_PASSWORD

Since these two are sensitive information, I have created them as Kubernetes Secret (mongodb-secret.yml). Then I can reference them in the former mongodb-deployment.yml.

e.g.:
```
env:
    - name: MONGO_INITDB_ROOT_USERNAME
      valueFrom:
        secretKeyRef:
        name: mongodb-secret
        key: mongodb-username

```
Note:
* _mongodb-secret_ is the name of my Kubernetes Secret (refer to mongodb-secret.yml)
* _mongodb-username_ is an entry in my Secret

Now, we can start minikube:
```
➜  mongoexpress-kubernetes git:(master) minikube start
😄  minikube v1.17.1 on Darwin 10.14.6
🎉  minikube 1.18.1 is available! Download it: https://github.com/kubernetes/minikube/releases/tag/v1.18.1
💡  To disable this notice, run: 'minikube config set WantUpdateNotification false'

✨  Using the docker driver based on existing profile
👍  Starting control plane node minikube in cluster minikube
🔄  Restarting existing docker container for "minikube" ...
🐳  Preparing Kubernetes v1.20.2 on Docker 20.10.2 ...
🔎  Verifying Kubernetes components...
🌟  Enabled addons: storage-provisioner, default-storageclass
🏄  Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default
```

and lets deploy our Secret first, as our deployment depends on it:
```
➜  mongoexpress-kubernetes git:(master) kubectl apply -f mongodb-secret.yml 
secret/mongodb-secret created
➜  mongoexpress-kubernetes git:(master) kubectl get secret
NAME                  TYPE                                  DATA   AGE
default-token-sd2hq   kubernetes.io/service-account-token   3      9d
mongodb-secret        Opaque                                2      35s
```

followed by deploying our MongoDB deployment:
``` 
➜  mongoexpress-kubernetes git:(master) kubectl apply -f mongodb-deployment.yml 
deployment.apps/mongodb-deployment created

➜  mongoexpress-kubernetes git:(master) kubectl get deployment -o wide
NAME                 READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS    IMAGES   SELECTOR
mongodb-deployment   1/1     1            1           13s   mongodb-pod   mongo    app=mongodb

➜  mongoexpress-kubernetes git:(master) kubectl get replicaset          
NAME                          DESIRED   CURRENT   READY   AGE
mongodb-deployment-fd7bd6dc   1         1         1       2m28s
```
lets check the pods for this deployment:
```
➜  mongoexpress-kubernetes git:(master) kubectl get pod -o wide
NAME                                READY   STATUS    RESTARTS   AGE   IP           NODE       NOMINATED NODE   READINESS GATES
mongodb-deployment-fd7bd6dc-bdt8c   1/1     Running   0          24s   172.17.0.2   minikube   <none>           <none>
```

and if we are curious about whats happening inside that pod :
```
➜  mongoexpress-kubernetes git:(master) kubectl logs mongodb-deployment-fd7bd6dc-bdt8c 
about to fork child process, waiting until server is ready for connections.
forked process: 30

{"t":{"$date":"2021-03-06T11:12:44.370+00:00"},"s":"I",  "c":"CONTROL",  "id":20698,   "ctx":"main","msg":"***** SERVER RESTARTED *****"}
{"t":{"$date":"2021-03-06T11:12:44.373+00:00"},"s":"I",  "c":"CONTROL",  "id":23285,   "ctx":"main","msg":"Automatically disabling TLS 1.0, to force-enable TLS 1.0 specify --sslDisabledProtocols 'none'"}
{"t":{"$date":"2021-03-06T11:12:44
...
...
{"t":{"$date":"2021-03-06T11:12:47.782+00:00"},"s":"I",  "c":"FTDC",     "id":20625,   "ctx":"initandlisten","msg":"Initializing full-time diagnostic data capture","attr":{"dataDirectory":"/data/db/diagnostic.data"}}
{"t":{"$date":"2021-03-06T11:12:47.785+00:00"},"s":"I",  "c":"NETWORK",  "id":23015,   "ctx":"listener","msg":"Listening on","attr":{"address":"/tmp/mongodb-27017.sock"}}
{"t":{"$date":"2021-03-06T11:12:47.785+00:00"},"s":"I",  "c":"NETWORK",  "id":23015,   "ctx":"listener","msg":"Listening on","attr":{"address":"0.0.0.0"}}
{"t":{"$date":"2021-03-06T11:12:47.785+00:00"},"s":"I",  "c":"NETWORK",  "id":23016,   "ctx":"listener","msg":"Waiting for connections","attr":{"port":27017,"ssl":"off"}}
```

and we can always get the description:
```
➜  mongoexpress-kubernetes git:(master) kubectl describe pod mongodb-deployment-fd7bd6dc-bdt8c 
Name:         mongodb-deployment-fd7bd6dc-bdt8c
Namespace:    default
Priority:     0
Node:         minikube/192.168.49.2
Start Time:   Sat, 06 Mar 2021 22:12:39 +1100
Labels:       app=mongodb
              pod-template-hash=fd7bd6dc
Annotations:  <none>
Status:       Running
IP:           172.17.0.2
IPs:
  IP:           172.17.0.2
Controlled By:  ReplicaSet/mongodb-deployment-fd7bd6dc
Containers:
  mongodb-pod:
    Container ID:   docker://2cab8eb3ee19f85963e95cee2e7b4ee803640d70c0a0446ff9eb3ed061e48d3a
    Image:          mongo
    Image ID:       docker-pullable://mongo@sha256:51f6fdbfc622d91e276ade7e6cf6491aa36ff2bd9b158dadb732f9e4a05f33ad
    Port:           27017/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Sat, 06 Mar 2021 22:12:44 +1100
    Ready:          True
    Restart Count:  0
    Environment:
      MONGO_INITDB_ROOT_USERNAME:  <set to the key 'mongodb-username' in secret 'mongodb-secret'>  Optional: false
      MONGO_INITDB_ROOT_PASSWORD:  <set to the key 'mongodb-password' in secret 'mongodb-secret'>  Optional: false
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-sd2hq (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  default-token-sd2hq:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-sd2hq
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                 node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age    From               Message
  ----    ------     ----   ----               -------
  Normal  Scheduled  4m52s  default-scheduler  Successfully assigned default/mongodb-deployment-fd7bd6dc-bdt8c to minikube
  Normal  Pulling    4m52s  kubelet            Pulling image "mongo"
  Normal  Pulled     4m48s  kubelet            Successfully pulled image "mongo" in 3.944508519s
  Normal  Created    4m48s  kubelet            Created container mongodb-pod
  Normal  Started    4m48s  kubelet            Started container mongodb-pod

```

and if we are really really curious, we can login to the container:
```
➜  mongoexpress-kubernetes git:(master) kubectl exec -it mongodb-deployment-fd7bd6dc-bdt8c  -- /bin/bash
root@mongodb-deployment-fd7bd6dc-bdt8c:/# hostname
mongodb-deployment-fd7bd6dc-bdt8c

root@mongodb-deployment-fd7bd6dc-bdt8c:/# ps -afeww
UID          PID    PPID  C STIME TTY          TIME CMD
mongodb        1       0  0 11:12 ?        00:00:03 mongod --auth --bind_ip_all
root         125       0  0 11:18 pts/0    00:00:00 /bin/bash
root         138     125  0 11:19 pts/0    00:00:00 ps -afeww
```

## MongoDB Service

Now, the way we are going to hook up our front end (MongoExpress) with MongoDB is through an internal service.
So lets create that inside our mongodb-deployment.yml. 

Note that a kubernetes YML file can contain more than one component. In this case, it is quite common to create service in the same YML file as the deployment (as they are related to each other).

Also, our selector (```app: mongodb```) in the service:
```
spec:
  selector:
    app: mongodb
```
is referring to the label that we have chosen in our mongodb-deployment.yml:

```
metadata:
  name: mongodb-deployment
  labels:
    app: mongodb
```

with this, lets apply our changes.
```
➜  mongoexpress-kubernetes git:(master) kubectl apply -f mongodb-deployment.yml
deployment.apps/mongodb-deployment unchanged
service/mongodb-service created
```

lets check our service.
```
➜  mongoexpress-kubernetes git:(master) ✗ kubectl describe service mongodb-service
Name:              mongodb-service
Namespace:         default
Labels:            <none>
Annotations:       <none>
Selector:          app=mongodb
Type:              ClusterIP
IP:                10.96.1.244
Port:              <unset>  27017/TCP
TargetPort:        27017/TCP
Endpoints:         172.17.0.2:27017
Session Affinity:  None
Events:            <none>

```

Great! Our service is connected to our container (172.17.0.2 - remember, this is the IP of our container when we describe our deployment earlier). 

With this, we are done with the backend!

# MongoExpress Deployment

Now we are going to add a front end to our MongoDB backend. First thing first, lets create the deployment for mongo express. We are creating ```mongoexpress-deployment.yml```. 

First thing first, lets go to Dockerhub to see the image for mongo-express (https://hub.docker.com/_/mongo-express). A few things :
* mongo express is listening on port 8081
* We need to pass in ME_CONFIG_MONGODB_ADMINUSERNAME and ME_CONFIG_MONGODB_ADMINPASSWORD for the username & password 
* We need to pass in ME_CONFIG_MONGODB_SERVER to point to the DB URL>

Now, for the username & password, we can just use the secret we created earlier. So similar to our MongoDB deployment, with the caveat of different environment variable names:
```
    - name: ME_CONFIG_MONGODB_ADMINUSERNAME
      valueFrom:
        secretKeyRef:
        name: mongodb-secret
        key: mongodb-username
    - name: ME_CONFIG_MONGODB_ADMINPASSWORD
      valueFrom:              
        secretKeyRef:
        name: mongodb-secret
        key: mongodb-password
```

For the DB URL though, as it is not so sensitive, we are going to create configMap for this: ```mongodb-configmap.yml```.

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: mongodb-configmap
data:
  database-url: mongodb-service
```

One important thing here - notice that our database URL is actually referring to ```mongodb-service```. What is this? It is none other than __the name of our mongodb internal service__! (the service we created earlier for our mongodb deployment). This is a very important thing. We dont refer to IP address ! We refer to the name of the service. It is kind of like a DNS name.


Now that we have this config map, we can refer it in our mongoexpress deployment:

```
- name: ME_CONFIG_MONGODB_SERVER
    valueFrom:
    configMapKeyRef:
    name: mongodb-configmap
    key: database-url 
```

Note that ```mongodb-configmap``` is the name of our configmap.

With these, lets deploy our new stuffs!

```
➜  mongoexpress-kubernetes git:(master) kubectl apply -f mongodb-configmap.yml 
configmap/mongodb-configmap created

➜  mongoexpress-kubernetes git:(master) ✗ kubectl get configmap -o wide
NAME                DATA   AGE
kube-root-ca.crt    1      9d
mongodb-configmap   1      2m48s
➜  mongoexpress-kubernetes git:(master) ✗ kubectl describe configmap mongodb-configmap
Name:         mongodb-configmap
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
database-url:
----
mongodb-service
Events:  <none>
```

and deploy our new shiny mongoexpress deployment:
```
➜  mongoexpress-kubernetes git:(master) ✗ kubectl apply -f mongoexpress-deployment.yml 
deployment.apps/mongoexpress-deployment created

➜  mongoexpress-kubernetes git:(master) ✗ kubectl get pod
NAME                                       READY   STATUS    RESTARTS   AGE
mongodb-deployment-fd7bd6dc-bdt8c          1/1     Running   0          37m
mongoexpress-deployment-5b85d8f894-pccw8   1/1     Running   0          20s

➜  mongoexpress-kubernetes git:(master) ✗ kubectl logs mongoexpress-deployment-5b85d8f894-pccw8
Waiting for mongodb-service:27017...
Welcome to mongo-express
------------------------


Mongo Express server listening at http://0.0.0.0:8081
Server is open to allow connections from anyone (0.0.0.0)
basicAuth credentials are "admin:pass", it is recommended you change this in your config.js!
Database connected
Admin Database connected
```


## MongoExpress Service

The final thing is to make an external service, so that our mongo express is accessible from outside.
Again, we can just add it in the same deployment file as our mongo express deployment.

```

```