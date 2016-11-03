# About this Repo

This repo forked from [Odoo](https://registry.hub.docker.com/_/odoo/).
Cluster taken from [paunin/postgres-docker-cluster](https://github.com/paunin/postgres-docker-cluster)
- - - -

## 1- Steps without pgpool

Steps for run in Google Cloud Shell.

1.1- Create new cluster according to **http://kubernetes.io/docs/hellonode/**

1.2- Clone this repo in directory **odoo-docker**
```
https://github.com/gorozcoh/docker.git odoo-docker
```

1.3- In folder **odoo-docker/10.0** build image ($PROJECT_ID = odoo-10-1):
```
docker build -t gcr.io/odoo-10-1/odoo-10:v1 .
```

1.4- Push image to Google Container Registry (GCR):
```
gcloud docker push gcr.io/odoo-10-1/odoo-10:v1
```

1.5- Adjust **image** in file **odoo-docker/10.0/odoo-deployment.yaml** according to step 1.2:
```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: odoo-10-deployment
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: odoo-10
    spec:
      containers:
      - name: odoo-10
        image: gcr.io/odoo-10-1/odoo-10:v1
        ports:
        - containerPort: 8069
```

1.6- Create namespace **app**:
```
kubectl create ns app
```

1.7- Create **node1**:
```
kubectl create -f /odoo-docker/paunin-cluster/node1-master.yaml
```
>**Note:** Watch kubernetes pod logs until Repmgr daemon started successfully after line **>>> Starting repmgr daemon...**

```
kubectl logs database-node1

>>> Starting repmgr daemon...
[2016-11-03 08:58:29] [NOTICE] looking for configuration file in current directory
[2016-11-03 08:58:29] [NOTICE] looking for configuration file in /etc
[2016-11-03 08:58:29] [NOTICE] configuration file found at: /etc/repmgr.conf
[2016-11-03 08:58:29] [INFO] connecting to database 'user=replication_user password=replication_pass host=database-node5-service dbname=replication_db port=5432 connect_timeout=2'
[2016-11-03 08:58:29] [INFO] connected to database, checking its state
[2016-11-03 08:58:29] [INFO] connecting to master node of cluster 'pg_cluster'
[2016-11-03 08:58:29] [INFO] retrieving node list for cluster 'pg_cluster'
[2016-11-03 08:58:29] [INFO] checking role of cluster node '1'
[2016-11-03 08:58:29] [INFO] checking cluster configuration with schema 'repmgr_pg_cluster'
[2016-11-03 08:58:29] [INFO] checking node 5 in cluster 'pg_cluster'
[2016-11-03 08:58:29] [INFO] reloading configuration file and updating repmgr tables
[2016-11-03 08:58:29] [INFO] starting continuous standby node monitoring
```

1.8- Create **node2** and **node4**:
```
kubectl create -f /odoo-docker/paunin-cluster/node2.yaml && kubectl create -f ./k8s/database-service/node4.yaml
```
>**Note:** Watch kubernetes pod logs until Repmgr daemon started successfully after line **>>> Starting repmgr daemon...**

1.9- Create **node3** and **node5**:
```
kubectl create -f /odoo-docker/paunin-cluster/node3.yaml && kubectl create -f ./k8s/database-service/node5.yaml
```
>**Note:** Watch kubernetes pod logs until Repmgr daemon started successfully after line **>>> Starting repmgr daemon...**

1.10- In node1 verify cluster topology:
```
kubectl exec -t database-node1 gosu postgres repmgr cluster show
```

```
Role      | Name  | Upstream | Connection String
----------+-------|----------|------------------------------------------------------------------------------------------------------------
* master  | node1 |          | user=replication_user password=replication_pass host=database-node1-service dbname=replication_db port=5432
  standby | node2 | node1    | user=replication_user password=replication_pass host=database-node2-service dbname=replication_db port=5432
  standby | node4 | node1    | user=replication_user password=replication_pass host=database-node4-service dbname=replication_db port=5432
  standby | node3 | node2    | user=replication_user password=replication_pass host=database-node3-service dbname=replication_db port=5432
  standby | node5 | node4    | user=replication_user password=replication_pass host=database-node5-service dbname=replication_db port=5432
```
1.11- Create load balancer service:
```
kubectl create -f odoo-load-balancer.yaml
```
1.12- Create deployment:
```
kubectl create -f odoo-deployment.yaml
```
1.13- Get IP for load balancer service (odoo-service):
```
NAME                     CLUSTER-IP      EXTERNAL-IP      PORT(S)    AGE
database-node1-service   10.55.254.178   <none>           5432/TCP   5h
database-node2-service   10.55.255.197   <none>           5432/TCP   5h
database-node3-service   10.55.243.9     <none>           5432/TCP   5h
database-node4-service   10.55.240.111   <none>           5432/TCP   5h
database-node5-service   10.55.255.90    <none>           5432/TCP   5h
kubernetes               10.55.240.1     <none>           443/TCP    5h
odoo-service             10.55.254.50    XX.XX.XX.XX      80/TCP     5h
```
1.14- In web browser open odoo
```
http://XX.XX.XX.XX
```
- - - -
## 2- Steps with pgpool
>**Note:** Odoo currently does not work when this procedure is followed.

2.1- Create new cluster according to **http://kubernetes.io/docs/hellonode/**

2.2- In folder **odoo-docker/10.0** modify file **odoo.conf** changing line:
```
...
db_host = database-pgpool-service
...
```
2.3- Repeat steps 1.3 and 1.4 saving image to **GCR** with version v2

2.4 Repeat step 1.5 using new image v2

2.5 Repeat steps 1.6 to 1.9.

2.6 Create pod and service for **pgpool**:
```
kubectl create -f /odoo-docker/paunin-cluster/pgpool.yaml
```
>**Note:** Watch kubernetes pod logs until Repmgr daemon started successfully.

2.7 Repeat steps 1.10 to 1.14

>**Note:** Creation de new database from web browser fails. View log files. Pending...
