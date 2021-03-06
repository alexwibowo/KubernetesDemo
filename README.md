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