# 单节点部署
* pv-pvc.yml
```yml
---
apiVersion: v1
kind: PersistentVolume
metadata:
  labels:
    type: "es-pv" 
  name: elasticsearch-pv
spec:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 3Gi
  nfs:
    path: /hzero/elasticsearch
    server: 192.168.1.13
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: elasticsearch-pvc
  namespace: hzero-devops
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi
  selector:
    matchLabels:
      type: "es-pv"  
``` 
* deployment.yml
```yml
kind: Deployment
apiVersion: apps/v1beta2
metadata:
  labels:
    elastic-app: elasticsearch
    role: master
  name: elasticsearch
  namespace: hzero-devops
spec:
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      elastic-app: elasticsearch
      role: master
  template:
    metadata:
      labels:
        elastic-app: elasticsearch
        role: master
    spec:
      securityContext:
        fsGroup: 1000
      initContainers:
      - name: "sysctl"
        image: "busybox"
        imagePullPolicy: "Always"
        command: ["sysctl", "-w", "vm.max_map_count=262144"]
        securityContext:
          privileged: true
      - name: "chown"
        image: docker.elastic.co/elasticsearch/elasticsearch:6.4.0
        imagePullPolicy: IfNotPresent
        command:
        - /bin/bash
        - -c
        - chown -R elasticsearch:elasticsearch /usr/share/elasticsearch/data &&
          chown -R elasticsearch:elasticsearch /usr/share/elasticsearch/logs
        securityContext:
          runAsUser: 0
        volumeMounts:
        - mountPath: /usr/share/elasticsearch/data
          name: elasticsearch
      containers:
        - name: elasticsearch-master
          image: docker.elastic.co/elasticsearch/elasticsearch:6.4.0
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 9200
              protocol: TCP
            - containerPort: 9300
              protocol: TCP
          env:
            - name: "cluster.name"
              value: "elasticsearch-cluster"
            - name: "node.master"
              value: "true"
            - name: "node.data"
              value: "true"
            - name: "node.ingest"
              value: "false"
            - name: "ES_JAVA_OPTS"
              value: "-Xms512m -Xmx512m"
            - name: PROCESSORS
              valueFrom:
                resourceFieldRef:
                  resource: limits.cpu
          resources:
            limits:
              cpu: "1"
              # memory: "2048Mi"
            requests:
              cpu: "25m"
            memory: "1536Mi"  
          volumeMounts:
          - mountPath: /usr/share/elasticsearch/data
            name: elasticsearch
      volumes:
      - name: elasticsearch
        persistentVolumeClaim:
          claimName: elasticsearch-pvc
---
kind: Service
apiVersion: v1
metadata:
  labels:
    elastic-app: elasticsearch
  name: elasticsearch
  namespace: hzero-devops
spec:
  ports:
    - port: 9200
      targetPort: 9200
  selector:
    elastic-app: elasticsearch
    role: master
```

# 多节点部署

* 3个master pvc
```yml
helm install ./pv-pvc \
--set pv.name=data-elasticsearch-master-0 \
--set nfs.path=/hzero/es-master-0 \
--set pvc.name=data-elasticsearch-master-0 \
--name es-master-pvc-0

helm install ./pv-pvc \
--set pv.name=data-elasticsearch-master-1 \
--set nfs.path=/hzero/es-master-1 \
--set pvc.name=data-elasticsearch-master-1 \
--name es-master-pvc-1

helm install ./pv-pvc \
--set pv.name=data-elasticsearch-master-2 \
--set nfs.path=/hzero/es-master-2 \
--set pvc.name=data-elasticsearch-master-2 \
--name es-master-pvc-2
```

* 2个data pvc

```
helm install ./pv-pvc \
--set pv.name=data-elasticsearch-data-0 \
--set nfs.path=/hzero/es-data-0 \
--set pvc.name=data-elasticsearch-data-0 \
--name es-data-pvc-0

helm install ./pv-pvc \
--set pv.name=data-elasticsearch-data-1 \
--set nfs.path=/hzero/es-data-1 \
--set pvc.name=data-elasticsearch-data-1 \
--name es-data-pvc-1
```
* 部署elasticsearch，默认部署3个master，2个client，2个data。可根据自己的需求自定义
```
helm repo add incubator https://kubernetes-charts-incubator.storage.googleapis.com/
helm install incubator/elasticsearch --name elasticsearch  --namespace hzero-devops

```