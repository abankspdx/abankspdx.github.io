---
title: "Druid Kubernetes Pain"
date: 2023-01-04T4:09:44-08:00
draft: false
---

## Druid? Yes. Kubernetes? Maybe. Networking? Scary.

I got the bright idea to try and deploy an Apache Druid  (https://druid.apache.org/) cluster to Kubernetes recently. I've never worked with either technology so I thought it'd be fun.

I was wrong.

## Druid

Druid seems great. Understanding the overall model of the software was pretty straightforward. There are a couple node types that each serve a specific purpose in the cluster, some of which being stateful (obviously, it's a database). Like many popular technologies, Druid relies on Zookeeper (https://zookeeper.apache.org/) to maintain cluster configuration/information. This is in theory fine, as Kubernetes' guarantee is that "Just one more node" shouldn't be a problem at all.

With the Zookeeper node (I opted to only deploy 1, because this is just for fun), that comes to 7 node types. 

 - Router
 - Broker
 - Coordinator
 - Middle Manager
 - Historical
 - Postgres
 - Zookeeper

The router was optional, but I really like having a web UI available to me, and since Druid is unfortunately incompatible with both of the database tools I use regularly (TablePlus (https://tableplus.com/) and DataGrip (https://www.jetbrains.com/datagrip/)) I figured it would be extra helpful. After all, it's just one more node, right?

## Kubernetes. Pain.

I started learning the basics of Kubernetes. Services. Deployments. StatefulSets. PersistentVolumes (and PersistentVolumeClaims). A lot of new words for Just A Couple Nodes. No worries, lots of resources available online.

So I started with Zookeeper. This was pretty straight forward. Zookeeper could be run as a standalone node and didn't require any crazy networking, and was stateless, so it went quick.
Service:

```service.yaml
apiVersion: v1
kind: Service
metadata:
  name: zk-cs
  labels:
    app: zk
spec:
  ports:
    - port: 2181
  selector:
    app: zk
```
Deployment:

```deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: zk-deployment
spec:
  selector:
    matchLabels:
      app: zk
  replicas: 1
  template:
    metadata:
      labels:
        app: zk
    spec:
      containers:
        - name: zookeeper
          image: zookeeper:3.5
          env:
            - name: ZOO_MY_ID
              value: "1"
          ports:
            - containerPort: 2181
          resources:
            limits:
              memory: "128Mi"
              cpu: "500m"
      restartPolicy: Always
```

After this, I've gotten my feet wet. Some `kubectl apply -f`'s and I've got some stuff going. Pretty nifty.

Until...

I did the exact same thing with the router. A `service.yaml` and a `deployment.yaml`. The only differences I could see was the port number, the container config and the number of replicas. I ran 3 instances of the router, just kind of to see if I could, but repeatedly got `cannot connect to Zookeeper`-type errors.

I could figure it out. I tried everything. I learned a lot about Kubernetes networking, and getting access to logs, and DNS, but never made any progress.
Everything I could find on the internet, including a number of reference implementations, suggested that my services should've worked. After probaly 6 hours I rewrote them both, from scratch, including deployments. _Pretty much_ the same code but not identical. I have to assume it was something name related that I just couldn't see (I'd been working on this until probably 5am), because post-rewrite everything worked fine.

The Router:

Service:

```service.yaml
apiVersion: v1
kind: Service
metadata:
  name: router
  labels:
    app: router
spec:
  ports:
    - port: 8888
  selector:
    app: router
```

PersistentVolumeClaim:
```persistentvolumeclaim.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: router
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Mi
status: {}

```

Deployment:
```deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: router
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2
      maxUnavailable: 1
  selector:
    matchLabels:
      app: druid
  replicas: 3
  template:
    metadata:
      labels:
        app: druid
    spec:
      containers:
        - name: router
          image: apache/druid:24.0.2
          args:
            - router
          env:
            - name: AWS_REGION
              value: us-west-2
            - name: DRUID_LOG4J
              value: <?xml version="1.0" encoding="UTF-8" ?><Configuration status="WARN"><Appenders><Console name="Console" target="SYSTEM_OUT"><PatternLayout pattern="%d{ISO8601} %p [%t] %c - %m%n"/></Console></Appenders><Loggers><Root level="info"><AppenderRef ref="Console"/></Root><Logger name="org.apache.druid.jetty.RequestLog" additivity="false" level="DEBUG"><AppenderRef ref="Console"/></Logger></Loggers></Configuration>
            - name: DRUID_SINGLE_NODE_CONF
              value: micro-quickstart
            - name: druid_coordinator_balancer_strategy
              value: cachingCost
            - name: druid_emitter_logging_logLevel
              value: debug
            - name: druid_extensions_loadList
              value: '["druid-histogram", "druid-datasketches", "druid-lookups-cached-global", "postgresql-metadata-storage", "druid-multi-stage-query", "druid-s3-extensions", "druid-parquet-extensions"]'
            - name: druid_indexer_fork_property_druid_processing_buffer_sizeBytes
              value: 256MiB
            - name: druid_indexer_logs_directory
              value: /opt/shared/indexing-logs
            - name: druid_indexer_logs_type
              value: file
            - name: druid_indexer_runner_javaOptsArray
              value: '["-server", "-Xmx1g", "-Xms1g", "-XX:MaxDirectMemorySize=3g", "-Duser.timezone=UTC", "-Dfile.encoding=UTF-8", "-Djava.util.logging.manager=org.apache.logging.log4j.jul.LogManager", "-Daws.region=us-west-2"]'
            - name: druid_metadata_storage_connector_connectURI
              value: jdbc:postgresql://postgres.druid.svc.cluster.local:5432/druid
            - name: druid_metadata_storage_connector_password
              value: FoolishPassword
            - name: druid_metadata_storage_connector_user
              value: druid
            - name: druid_metadata_storage_host
            - name: druid_metadata_storage_type
              value: postgresql
            - name: druid_processing_numMergeBuffers
              value: "2"
            - name: druid_processing_numThreads
              value: "2"
            - name: druid_storage_storageDirectory
              value: /opt/shared/segments
            - name: druid_storage_type
              value: local
            - name: druid_zk_service_host
              value: zk-cs.druid.svc.cluster.local
          ports:
            - containerPort: 8888
          volumeMounts:
            - mountPath: /opt/druid/var
              name: router
          resources:
            limits:
              memory: "512Mi"
              cpu: "500m"
      restartPolicy: Always
      volumes:
        - name: router
          persistentVolumeClaim:
            claimName: router
```

To my job, after running the deployment and the pods becoming accessible, I saw:

```
2023-01-05T01:06:35,500 INFO [main-SendThread(zk-cs.druid.svc.cluster.local:2181)] org.apache.zookeeper.ClientCnxn - Socket connection established, initiating session, client: /172.17.0.16:57282, server: zk-cs.druid.svc.cluster.local/10.99.26.238:2181
2023-01-05T01:06:35,859 INFO [main-SendThread(zk-cs.druid.svc.cluster.local:2181)] org.apache.zookeeper.ClientCnxn - Session establishment complete on server zk-cs.druid.svc.cluster.local/10.99.26.238:2181, sessionid = 0x10000372dd80008, negotiated timeout = 30000
```

Huzzah! The router could communicate with Zookeeper!

## Postgres

I'd forgotten (it had been like 20 hours of debugging over 2 days at this point) that I needed to deploy Postgres for any of this to work. Luckily postgres could be a single node, and there were dozens of reference impls online for running Postgres from Kubernetes. The only new thing I found was creating `PersistentVolume` separate from `PersistentVolumeClaim` which, maybe I just don't understand but, certainly seem like they could be a single yaml file.

Anyway.

Service:
```service.yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
  labels:
    app: postgres
spec:
  ports:
    - port: 5432
  selector:
    app: postgres
```

PV:
```pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: postgres-pv-volume
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce  
  hostPath:
    path: "/mnt/data"
```

PVC:
```pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pv-claim
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

Deployment:
```deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      volumes:
        - name: postgres-pv-storage
          persistentVolumeClaim:
            claimName: postgres-pv-claim
      containers:
        - name: postgres
          image: postgres:11
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_PASSWORD
              value: FoolishPassword
            - name: POSTGRES_USER
              value: druid
            - name: POSTGRES_DB
              value: druid
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          volumeMounts:
            - mountPath: /var/lib/postgresql/data
              name: postgres-pv-storage
          resources:
            limits:
              memory: "512Mi"
              cpu: "500m"
```

I remembered to match the Postgres env vars that I'd configured in Druid config, back to the configuration for Postgres I used in the deployment (specifically the DB/user/password). Things started up normally and seemed fine.

## Kubernetes, but with significantly less pain

Okay so, I'd deployed two services and they could communicate. The rest should be mostly copy and paste, right? The only hiccup would be that Historical and MiddleManager nodes need to be StatefulSets instead of Deployments. Please be easy.

So after deploying almost-identical service and deployments for the Coordinator and the Broker, it was time to take a shot at the Historical node. I read a bunch about StatefulSets and gave it a whirl.

Service:
```service.yaml
apiVersion: v1
kind: Service
metadata:
  name: historical
  labels:
    app: historical
spec:
  ports:
    - port: 8083
  selector:
    app: historical
```

StatefulSet:
```statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: historical
  labels:
    app: druid
spec:
  serviceName: historical
  replicas: 2
  selector:
    matchLabels:
      app: druid
  template:
    metadata:
      labels:
        app: druid
    spec:
      volumes:
        - name: druid-shared
          emptyDir: {}
        - name: historical-var
          emptyDir: {}
      containers:
        - name: historical
          image: apache/druid:24.0.2
          args:
            - historical
          ports:
            - containerPort: 8083
          volumeMounts:
            - mountPath: /opt/shared
              name: druid-shared
            - mountPath: /opt/druid/var
              name: historical-var
          env:
            - name: AWS_REGION
              value: us-west-2
            - name: DRUID_LOG4J
              value: <?xml version="1.0" encoding="UTF-8" ?><Configuration status="WARN"><Appenders><Console name="Console" target="SYSTEM_OUT"><PatternLayout pattern="%d{ISO8601} %p [%t] %c - %m%n"/></Console></Appenders><Loggers><Root level="info"><AppenderRef ref="Console"/></Root><Logger name="org.apache.druid.jetty.RequestLog" additivity="false" level="DEBUG"><AppenderRef ref="Console"/></Logger></Loggers></Configuration>
            - name: DRUID_SINGLE_NODE_CONF
              value: micro-quickstart
            - name: druid_coordinator_balancer_strategy
              value: cachingCost
            - name: druid_emitter_logging_logLevel
              value: debug
            - name: druid_extensions_loadList
              value: '["druid-histogram", "druid-datasketches", "druid-lookups-cached-global", "postgresql-metadata-storage", "druid-multi-stage-query", "druid-s3-extensions", "druid-parquet-extensions"]'
            - name: druid_indexer_fork_property_druid_processing_buffer_sizeBytes
              value: 256MiB
            - name: druid_indexer_logs_directory
              value: /opt/shared/indexing-logs
            - name: druid_indexer_logs_type
              value: file
            - name: druid_indexer_runner_javaOptsArray
              value: '["-server", "-Xmx1g", "-Xms1g", "-XX:MaxDirectMemorySize=3g", "-Duser.timezone=UTC", "-Dfile.encoding=UTF-8", "-Djava.util.logging.manager=org.apache.logging.log4j.jul.LogManager", "-Daws.region=us-west-2"]'
            - name: druid_metadata_storage_connector_connectURI
              value: jdbc:postgresql://postgres.druid.svc.cluster.local:5432/druid
            - name: druid_metadata_storage_connector_password
              value: FoolishPassword
            - name: druid_metadata_storage_connector_user
              value: druid
            - name: druid_metadata_storage_host
            - name: druid_metadata_storage_type
              value: postgresql
            - name: druid_processing_numMergeBuffers
              value: "2"
            - name: druid_processing_numThreads
              value: "2"
            - name: druid_storage_storageDirectory
              value: /opt/shared/segments
            - name: druid_storage_type
              value: local
            - name: druid_zk_service_host
              value: zk-cs.druid.svc.cluster.local
```

After a `kubectl apply -f` for the Service and the StatefulSet, everything looked great!
```
2023-01-05T01:06:22,538 INFO [main-SendThread(zk-cs.druid.svc.cluster.local:2181)] org.apache.zookeeper.ClientCnxn - Opening socket connection to server zk-cs.druid.svc.cluster.local/10.99.26.238:2181. Will not attempt to authenticate using SASL (unknown error)
2023-01-05T01:06:22,552 INFO [main-SendThread(zk-cs.druid.svc.cluster.local:2181)] org.apache.zookeeper.ClientCnxn - Socket connection established, initiating session, client: /172.17.0.14:43664, server: zk-cs.druid.svc.cluster.local/10.99.26.238:2181
2023-01-05T01:06:22,564 INFO [main-SendThread(zk-cs.druid.svc.cluster.local:2181)] org.apache.zookeeper.ClientCnxn - Session establishment complete on server zk-cs.druid.svc.cluster.local/10.99.26.238:2181, sessionid = 0x10000372dd80004, negotiated timeout = 30000

```
!!

Seems like we're in business.

At this point, I could also see the nodes showing up in the Druid UI (after learning about `kubectl port-forward` to be able to use the Router from outside the Kubernetes network)

From here I basically copied and pasted the service and statefulset yaml from the historical, and changed the word `historical` to `middelmanager` and was done. When I finally saw:

![image](https://abankspdx.github.io/druid.png)

I was HYPE. Also, Druid Doctor is a great tool (that I learned about much too late in this process)

## Closing Thoughts

This process took entirely too long. I learned enough about Kubernetes to feel frustrated by my experience and didn't really learn enough about Druid itself, just its deployment strategy and architecture. Soon I'll be dumping a whole bunch of NBA data into this cluster (after maybe deploying it to Digital Ocean?) to play around with. We'll see. I feel proud that I completed it, even if I think it should've been easier.

Also, it is totally unclear to me why the Druid nodes needed to be StatefulSets when Postgres is also stateful (maybe the most stateful?) and was done via Deployment? I followed the guides I saw online, but I still do not understand.

Anyway, if you made it this far, thanks for reading!