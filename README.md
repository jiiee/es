# elasticsearch单节点部署
## pv-pvc.yml
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
## deployment.yml
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

# elasticsearch多节点部署

## 3个master pvc
```bash
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

## 2个data pvc

```bash
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
```bash
helm repo add incubator https://kubernetes-charts-incubator.storage.googleapis.com/
helm install incubator/elasticsearch --name elasticsearch  --namespace hzero-devops
```

# 部署kibana
```bash
helm install stable/kibana --namespace hzero-devops \
--set env.ELASTICSEARCH_URL=http://elasticsearch-client:9200 --name kibana
```
# 部署logstash
## pvc
```bash
helm install ./pv-pvc \
--set pv.name=data-logstash-0 \
--set nfs.path=/hzero/logstash-0 \
--set pvc.name=data-logstash-0 \
--name logstash-pvc-0
```
```bash
helm install incubator/logstash --namespace hzero-devops --name logstash \
--set elasticsearch.host=elasticsearch-client
```

# 部署filebeat
* filebeat.yml
```yml
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: filebeat
  labels:
    app: filebeat
spec:
  template:
    metadata:
      labels:
        app: filebeat
      name: filebeat
    spec:
      # filebeat has to run as root to read logs, since the logs
      # on the host system are all owned by root. this is a little
      # scary, but at least all the host mounts are read-only, to minimize
      # possible damage.
      securityContext:
        runAsUser: 0
      containers:
      - name: filebeat
        image: docker.elastic.co/beats/filebeat:6.4.1
        # override the configuration path. because kubernetes can only mount directories
        # and not individual files
        command: [ "/usr/share/filebeat/filebeat"]
        args: [ "-e", "-path.config", "/usr/share/filebeat/config"]
        resources:
          requests:
            cpu: 50m
            memory: 50Mi
          limits:
            cpu: 50m
            memory: 100Mi
        env:
          - name: LOGSTASH_HOSTS
            value: logstash:5044
          - name: LOG_LEVEL
            value: info
          - name: FILEBEAT_HOST
            valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
        volumeMounts:
        - name: config-volume
          mountPath: /usr/share/filebeat/config
        - name: varlog
          mountPath: /var/log/hostlogs
          readOnly: true
        - name: varlogcontainers
          mountPath: /var/log/containers
          readOnly: true
        - name: varlogpods
          mountPath: /var/log/pods
          readOnly: true
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        - name: vartmp
          mountPath: /var/tmp/filebeat
      terminationGracePeriodSeconds: 30
      # allow filebeat to also be scheduled on master nodes, so we can pick up kubernetes logs
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      volumes:
      # mount the root of var log so that system level logs (like kube-apiserver and etcd) can be read
      - name: varlog
        hostPath:
          path: /var/log
      # mount /var/tmp as a persistent place to store the filebeat registry
      - name: vartmp
        hostPath:
          path: /var/tmp
      # mount /var/log/containers to get friendly named symlinks to actual logs
      - name: varlogcontainers
        hostPath:
          path: /var/log/containers
      # mount /var/log/pods as its where pod logs are collected
      - name: varlogpods
        hostPath:
          path: /var/log/pods
      # mount /var/lib/docker/containers which is where the logs _actually_ are
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      # mount the configmap with the filebeat config file
      - name: config-volume
        configMap:
          name: logging-configmap
          items:
            - key: filebeat.yml
              path: filebeat.yml
```
* logging-configmap.yaml
```yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: logging-configmap
data:
  filebeat.yml: |
    filebeat.registry_file: /var/tmp/filebeat/filebeat_registry # store the registry on the host filesystem so it doesn't get lost when pods are stopped
    filebeat.prospectors:
    # process all docker container logs, which are stored as json
    - input_type: log
      symlinks: true
      json.message_key: log
      json.keys_under_root: true
      json.add_error_key: true
      multiline.pattern: '^\s'
      multiline.match: after
      fields:
          host: ${FILEBEAT_HOST:${HOSTNAME}}
          type: kube-logs
      fields_under_root: true
      paths:
        - /var/log/containers/config-server*.log
        - /var/log/containers/api-gateway*.log
        - /var/log/containers/gateway-helper*.log
        - /var/log/containers/manager-service*.log
        - /var/log/containers/oauth-server*.log
        - /var/log/containers/mysql*.log
        - /var/log/containers/redis*.log
    # process system logs, such as kube-apiserver, kube-controller-manager, etc
    - input_type: log
      fields:
          host: ${FILEBEAT_HOST:${HOSTNAME}}
          type: kube-logs
      fields_under_root: true
      paths:
        - /var/log/hostlogs/not*.log
    output.logstash:
      hosts: ["logstash:5044"]
```