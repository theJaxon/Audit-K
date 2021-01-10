# Audit-K
Project made for collecting and filtering Kubernetes audit policy logs using various tools

![K8s](https://img.shields.io/badge/-kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)
![FluentBit](https://img.shields.io/badge/-FluentBit-0E83C8?style=for-the-badge&logo=FluentD&logoColor=white)
![ElasticSearch](https://img.shields.io/badge/-Elasticsearch-005571?style=for-the-badge&logo=Elasticsearch&logoColor=white)
![Kibana](https://img.shields.io/badge/-Kibana-005571?style=for-the-badge&logo=Kibana&logoColor=white)

### Enabling [audit logs in Kubernetes](https://kubernetes.io/docs/tasks/debug-application-cluster/audit/#log-backend):
- By default no audit logs are stored in the cluster, to enable audit logs we have to modify the config file at `/etc/kubernetes/manifests/kube-apiserver.yaml`
- There are 2 types of [audit backends](https://kubernetes.io/docs/tasks/debug-application-cluster/audit/#audit-backends):
  1. Log backend (which i'll be using)
  2. Webhook backend

```yaml
# Create the policy file containing the levels needed to be logged at each stage
vi /etc/kubernetes/audit.yml

# Make a directory to store the logs 
mkdir -pv /etc/kubernetes/audit

# Modify the kube-apiserver configuration
vi /etc/kubernetes/manifests/kube-apiserver.yaml

- --audit-policy-file=/etc/kubernetes/audit.yml
- --audit-log-path=/etc/kubernetes/audit/audit.log
- --audit-log-maxage=5 # Max number of days to keep old logs
- --audit-log-maxsize=5 # Max size of the log file in Megabyte
- --audit-log-maxbackup=5 # Max number of log files to be kept

# Define audit log volumes 
volumes:
- name: audit-policy-v
  hostPath: 
    path: /etc/kubernetes/audit.yml
    type: File
- name: audit-log-v
  hostPath:
    path: /etc/kubernetes/audit/audit.log
    type: FileOrCreate
  
# Mount the volumes 
volumeMounts:
- name: audit-policy-v
  mountPath: /etc/kubernetes/audit.yml
- name: audit-log-v
  mountPath: /etc/kubernetes/audit/audit.log
```

---

### 1. Audit Logs using EFK:
#### FluentBit configuration:
FluentBit is installed on the kubernetes cluster following the [guide](https://docs.fluentbit.io/manual/installation/kubernetes#installation):
```bash
# Create logging namespace
k create ns logging $do > ns.yml

# Create FluentBit ServiceAccount in the logging namespace
k create sa fluent-bit -n logging $do > sa.yml

# Create ClusterRole with reading privileges
k create clusterrole fluent-bit-read --resource=ns,po --verb=get,list,watch $do > cluster-role.yml

# Bind the service account with the cluster role
k create clusterrolebinding fluent-bit-read --serviceaccount=logging:fluent-bit --clusterrole=fluent-bit-read $do > cluster-role-binding.yml 

# Create the ConfigMap the will be used by the DaemonSet
k apply -f https://raw.githubusercontent.com/fluent/fluent-bit-kubernetes-logging/master/output/elasticsearch/fluent-bit-configmap.yaml

# Apply FluentBit to ElasticSearch DaemonSet 
# https://docs.fluentbit.io/manual/installation/kubernetes#fluent-bit-to-elasticsearch
k apply -f https://raw.githubusercontent.com/fluent/fluent-bit-kubernetes-logging/master/output/elasticsearch/fluent-bit-ds.yaml
```

Applying all these steps from the **FluentBit** directory:
```bash
k apply -f FluentBit/
namespace/logging created
serviceaccount/fluent-bit created
clusterrole.rbac.authorization.k8s.io/fluent-bit-read created
clusterrolebinding.rbac.authorization.k8s.io/fluent-bit-read created
configmap/fluent-bit-config created
daemonset.apps/fluent-bit created
```

#### ElasticSearch Configuration:
```yaml
# A configMap is created containing elasticsearch.yml config file to be placed at config directory
k create configmap elasticsearch --from-file=elasticsearch.yml -n logging

# In ElasticSearch deployment the file is placed in /usr/share/elasticsearch/config 
spec:
  volumes:
  - name: elasticsearch-v 
    configMap:
      name: elasticsearch 
  containers:
  - image: docker.elastic.co/elasticsearch/elasticsearch:7.5.2
    name: elasticsearch
    ports:
    - containerPort: 9200
      name: elasticsearch
    volumeMounts:
    - name: elasticsearch-v 
      mountPath: /usr/share/elasticsearch/config/elasticsearch.yml
      subPath: elasticsearch.yml
```

#### Configuring Kibana to work with ElasticSearch:
In kibana configuration elasticsearch is identified through service discovery using
```bash
elasticsearch.hosts: ["http://elasticsearch:9200"]
```

```yaml
# Create Kibana config file
k create configmap kibana --from-file=kibana.yml -n logging

# Use the config file in kibana deployment 
spec:
  volumes:
  - name: kibana-v 
    configMap:
      name: kibana
  containers:
  - image: docker.elastic.co/kibana/kibana:7.5.2
    name: kibana
    ports:
    - containerPort: 5601
      name: kibana
    volumeMounts:
    - name: kibana-v 
      mountPath: /usr/share/kibana/config/kibana.yml
      subPath: kibana.yml
```