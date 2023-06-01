# Setting up Confluent for Kubernetes on minikube

This is intended to be a walkthrough of the Confluent for Kubernetes quickstart.  

We will start by looking at the components that are installed by default and then look to start making modifications and additions with a view to demonstrating the workflow.

#### Install `minikube`

Start by installing minikube - follow the guide here: https://minikube.sigs.k8s.io/docs/start/

#### Install `helm`

Then install helm (https://helm.sh/) on OS X we're going to use brew for this:

```bash
brew install helm
```

#### Install `cubectx`

Let's also install `cubectx`:

```bash
brew install kubectx
```

#### Install `krew`

Install the `kubectl` plugin manager:

```bash
brew install krew
```

### Configuring `minikube`

Ensure `minikube` is successfully installed:

```bash
minikube version
```

You should see something like this:

```bash
minikube version: v1.30.1
commit: 08896fd1dc362c097c925146c4a0d0dac715ace0
```

Configuring some settings based on Memory / CPU capacity (or availability):

```bash
minikube config set memory 20000MB
minikube config set cpus 6
```

Check for updates

```bash
minikube update-check
```

```bash
minikube start
```

Note the configuration resources on startup of Minikube:

```
Creating docker container (CPUs=8, Memory=20000MB) ...
```

Get the status of Minikube:

```bash
minikube status
```

You should see:

```bash
minikube
type: Control Plane
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured
```

### Start Confluent for Kubernetes

Create our `confluent` namespace:

```bash
kubectl create namespace confluent
```

You should see:

```
namespace/confluent created
```

Then run the following to set the `confluent` namespace as the current context:

```bash
kubectl config set-context --current --namespace confluent
```

You should see:

```
Context "minikube" modified.
```

Add the helm repo:

```bash
helm repo add confluentinc https://packages.confluent.io/helm
```

Update the helm repo:

```bash
helm repo update
```

You should see:

```
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "confluentinc" chart repository
Update Complete. ⎈Happy Helming!⎈
```

Install CfK (the Confluent Operator)

```bash
helm upgrade --install confluent-operator confluentinc/confluent-for-kubernetes
```

You should see:

```
Release "confluent-operator" does not exist. Installing it now.
NAME: confluent-operator
LAST DEPLOYED: Tue May 30 16:27:26 2023
NAMESPACE: confluent
STATUS: deployed
[...]
```

### What are we deploying?

See: https://github.com/confluentinc/confluent-kubernetes-examples/tree/master/quickstart-deploy for more details.

Deploy the latest yaml file:

```bash
kubectl apply -f https://raw.githubusercontent.com/confluentinc/confluent-kubernetes-examples/master/quickstart-deploy/confluent-platform.yaml
```

You should see:

```
zookeeper.platform.confluent.io/zookeeper created
kafka.platform.confluent.io/kafka created
connect.platform.confluent.io/connect created
ksqldb.platform.confluent.io/ksqldb created
controlcenter.platform.confluent.io/controlcenter created
schemaregistry.platform.confluent.io/schemaregistry created
kafkarestproxy.platform.confluent.io/kafkarestproxy created
```

Get the status:

```bash
kubectl get pods
```

Or:

```bash
kubectl get pods --output=wide
```

### Service List

```bash
minikube service list
```

You should see:

```
|-------------|---------------------------|--------------|-----|
|  NAMESPACE  |           NAME            | TARGET PORT  | URL |
|-------------|---------------------------|--------------|-----|
| confluent   | confluent-operator        | No node port |     |
[...]
```

Get the service logs:

```bash
minikube logs kafka | less
```

Also try:

```bash
minikube logs kafka-0
```

Get the Kafka logs from the container:

```bash
kubectl logs kafka-0 | less
```

Describe the service:

```bash
kubectl describe services kafka-0
```

### Remember the namespace context!

Your containers are all created in the `confluent` namespace, so ensure you're using that context - you can check this by running:

```bash
kubectl config view --minify --flatten
```

You should see:

```bash
contexts:
- context:
    namespace: confluent
```

If you don't see this, ensure you're set to the correct context:

```bash
kubectl config set-context --current --namespace confluent
```

And if so, running `kubectl get pods` should show you 14 running containers:

```
kubectl get pods --output=wide
```

### Using `cubectx`

As we've got `cubectx` installed, we can run:

```bash
kubectx
```

This will give us a list of the top-level contexts (e.g. `minikube`).  To get the namespace context, we can run:

```bash
kubens -c
```

This will return the namespace context:

```
confluent
```

### Check C3

Describe the service:

```bash
kubectl describe services controlcenter-0
```

Look at the endpoints:

```bash
minikube service -n confluent controlcenter --url
```

In order to be able to access the container, we need to configure port forwarding on port 9021:

```bash
kubectl port-forward controlcenter-0 9021:9021
```

Check the setup by going to http://localhost:9021/clusters

#### Re-deploy C3

If you need to re-deploy C3 after any configuration changes:

```bash
kubectl delete pod controlcenter-0
kubectl apply -f controlcenter.yaml
```


### Let's look at Zookeeper

Describe `zookeeper-0`:

```bash
kubectl describe services zookeeper-0
```

Connect to the `zookeeper-0` container using Zookeeper shell:

```bash
kubectl exec zookeeper-0 zookeeper-shell localhost:2181
```

See all paths (using a single command that closes afterwards):

```bash
kubectl exec zookeeper-0 zookeeper-shell 0.0.0.0:2181 ls /
```

Another example:

```bash
kubectl exec zookeeper-0 zookeeper-shell 0.0.0.0:2181 get /kafka-confluent/controller
```

And let's ssh into the instance:

```bash
kubectl exec -it zookeeper-0 -- bash
```

Connect using `zookeeper-shell`

```bash
zookeeper-shell localhost:2181
```

List paths:

```bash
ls /
```

Figure out who the controller is:

```bash
get /kafka-confluent/controller
```

You should see:

```json
{"version":2,"brokerid":0,"timestamp":"1685460739018","kraftControllerEpoch":-1}
```

Get the Cluster ID:

```bash
get /kafka-confluent/cluster/id
```

You should see:

```json
{"version":"1","id":"Sfr7YvlkRHSHrmSrwq9mug"}
```

And list all (3) brokers:

```bash
ls /kafka-confluent/brokers/ids
```

Get some information about broker 1:

```bash
get /kafka-confluent/brokers/ids/1
```

### Tests on the `kafka-0` broker

```bash
kubectl describe services kafka-0
```

Get the cluster ID from the broker:

```bash
kubectl exec kafka-0 -- kafka-cluster cluster-id --bootstrap-server localhost:9092
```

Use `kafka-configs` to describe all broker entity information:

```shell
kubectl exec kafka-0 -- kafka-configs --bootstrap-server localhost:9092 --entity-type brokers --describe --all
```

### Let's test it

We will start by creating a topic:

```shell
kubectl exec kafka-0 -- kafka-topics --bootstrap-server localhost:9092 --topic ab-demo-topic --replication-factor 3 --partitions 1 --create --config min.insync.replicas=2
```

You should see:

```bash
Created topic ab-demo-topic.
```


Let's try to produce to the topic:

```bash
kubectl exec kafka-0 -- kafka-console-producer --bootstrap-server localhost:9092 --topic ab-demo-topic
```

Note that it quits immediately.   Let's try this instead:

```bash
seq 100 | kubectl exec kafka-0 -- kafka-console-producer --bootstrap-server localhost:9092 --topic ab-demo-topic
```

Try to consume from the topic:

```bash
kubectl exec kafka-0 -- kafka-console-consumer --bootstrap-server  localhost:9092 --from-beginning --topic ab-demo-topic
```

This doesn't seem to work either.  Let's ssh into the instance:

```bash
kubectl exec kafka-0 -it -- bash
```

From here we can produce:

```bash
kafka-console-producer --bootstrap-server localhost:9092 --topic ab-demo-topic
```

And try this:

```bash
seq 100 | kafka-console-producer --bootstrap-server localhost:9092 --topic ab-demo-topic
```

Now let's try to consume from the topic to confirm that the messages are there:

```bash
kafka-console-consumer --bootstrap-server  localhost:9092 --from-beginning --topic ab-demo-topic
```

Ok - so that's great.   But we're manually creating topics.   Let's look at how we can create Confluent for Kubernetes resources.

### Creating a topic (`topic.yaml`)

```yaml
apiVersion: platform.confluent.io/v1beta1
kind: KafkaTopic
metadata:
  name: ab-example-topic
  namespace: confluent
spec:
  replicas: 1
  partitionCount: 3
  kafkaClusterRef:
    name: kafka
  configs:
    min.insync.replicas: "1"
    retention.ms: "86400000"
```

Then apply it using `kubectl apply`:

```bash
kubectl apply -f topic.yaml
```

You should see:

```bash
kafkatopic.platform.confluent.io/ab-example-topic created
```

Read more on topic management here: https://docs.confluent.io/operator/current/co-manage-topics.html

### `kubectl explain`

We can use `kubectl explain` to get more information about a given resource specification:

```bash
kubectl explain kafkarestproxy.spec
```

```
kubectl explain kafkarestproxy.spec.dependencies.kafka
```

Let's look at the `KafkaTopic` specification:

```bash
kubectl explain KafkaTopic.spec
```

You will see:

```
KIND:     KafkaTopic
VERSION:  platform.confluent.io/v1beta1

RESOURCE: spec <Object>

DESCRIPTION:
     spec defines the desired state of the KafkaTopic.
```

### Viewing Service information (as JSON)

```bash
kubectl get services -n confluent -ojson | jq
```

### List Persistent Volumes

```bash
kubectl get pv --sort-by=.spec.capacity.storage
```

### Show only running instances

```bash
kubectl get pods --field-selector=status.phase=Running
```

### Scaling Instances

Kubernetes will allow you to easily scale your infrastructure using a command (and potentially, thresholds):

```bash
kubectl scale zookeeper zookeeper --replicas=2
```

You will see:

```
zookeeper.platform.confluent.io/zookeeper scaled
```

### Schema Registry

Let's find out about Schema Registry

```bash
kubectl describe services schemaregistry-0
```

Note that the external port is 8081:

```
Port:              external  8081/TCP
TargetPort:        8081/TCP
Endpoints:         10.244.0.12:8081
```

```bash
kubectl port-forward schemaregistry-0 8081:8081
```

http://localhost:8081/subjects

Or:

```bash
curl --silent -X GET http://localhost:8081/subjects/ | jq
```

```bash
curl --silent -X GET http://localhost:8081/config | jq
```

```bash
curl -s -XGET http://localhost:8081/schemas/types | jq
```


### ReST Proxy

```bash
kubectl describe services kafkarestproxy-0
```

External port is 8082:

```
Port:              external  8082/TCP
TargetPort:        8082/TCP
Endpoints:         10.244.0.11:8082
```

```bash
kubectl port-forward kafkarestproxy-0 8082:8082
```

```bash
curl --silent -X GET "http://localhost:8082/topics" | jq
```

Let's view `ab-demo-topic` to see the current configuration:

```bash
curl --silent -X GET "http://localhost:8082/topics/ab-demo-topic" | jq
```

Let's view partition information:

```bash
curl --silent -X GET "http://localhost:8082/topics/ab-demo-topic/partitions" | jq
```

curl --silent -X GET http://localhost:8082/topics | jq

### Ubuntu Tools Instance

Useful to have an instance containing a set of tools that you can use to interrogate your pods.

We're going to use this as our base: https://github.com/ableasdale/confluent-dockerfiles/blob/main/mTLS/tools/Dockerfile

We're going to build the `Dockerfile`:

```bash
docker build . -t alexjbleasdale/ubuntu-tools:0.0.2
```

And then push it:

```bash
docker push alexjbleasdale/ubuntu-tools:0.0.2
```

Then we create our resource for it:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
    - name: my-container
      image: alexjbleasdale/ubuntu-tools:0.0.2
      env:
        - name: CONFLUENT_HOME
          value: /usr
        - name: JAVA_HOME
          value: /usr/lib/jvm/java-17-openjdk-amd64/
        - name: TERM
          value: xterm-256color
      command: ["sleep", "infinity"]
      resources:
        requests:
          memory: "64Mi"
          cpu: "100m"
        limits:
          memory: "128Mi"
          cpu: "200m"
      volumeMounts:
        - name: my-volume
          mountPath: /var/my-data
  volumes:
    - name: my-volume
      emptyDir: {}

```

Then instantiate it:

```bash
kubectl apply -f ubuntu-tools.yml
```

Now let's ssh into it:

```bash
kubectl exec ubuntu-pod -it -- bash
```

So we can now run:

```bash
java
kcat
httpie
jq
confluent
```

... and so on ...

To redeploy, simply run:

```bash
kubectl delete pod ubuntu-pod
kubectl apply -f ubuntu-tools.yml
```

### Using the Kafka tools on the Ubuntu Pod

```bash
/opt/kafka/bin/kafka-console-producer.sh
```

TODO: 

/opt/kafka/bin/kafka-console-producer.sh --bootstrap-server kafka-0.confluent.svc.cluster.local:9092 --topic ab-test-topic 
doesn't work

kubectl exec ubuntu-pod -- /opt/kafka/bin/kafka-console-producer.sh --bootstrap-server kafka-0:9092 --topic ab-test-topic

To resolve the host by "hostname":

```bash
ping kafka-0.confluent.svc.cluster.local
```


This seems to work if you specify the IP for the service for `kafka-0`

```bash
kubectl exec ubuntu-pod -- /opt/kafka/bin/kafka-console-producer.sh --bootstrap-server 10.106.55.186:9092 --topic ab-test-topic
```

Also - note that you can get the FQDN from a host by running:

```bash
hostname --fqdn
kafka-0.kafka.confluent.svc.cluster.local
```

So this works:

```bash
/opt/kafka/bin/kafka-console-producer.sh --bootstrap-server kafka-0.kafka.confluent.svc.cluster.local:9092 --topic ab-test-topic 
```

hostname --fqdn
controlcenter-0.controlcenter.confluent.svc.cluster.local

### Convenience methods


### Support Bundle

```bash
kubectl confluent support-bundle --namespace <namespace>
```

See: https://docs.confluent.io/operator/current/co-troubleshooting.html#support-bundle


### Sidecar Pod

TODO - old?

```yaml
apiVersion: v1

kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: centos
  name: centos
  namespace: confluent
spec:
  containers:
  - image: centos:8
    name: centos
    command: ["/bin/sleep", "3650d"]
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
```

### Teardown

If confguration changes cause major issues, minikube can be stopped and removed:

```bash
minikube stop
minikube delete
minikube start
```

This will allow you to start fresh with a new deployment.

## Building the setup piece by piece...

After restarting with a **clean** install of minikube (see the Teardown section).

We need to run:

```bash
kubectl create namespace confluent
kubectl config set-context --current --namespace confluent
helm repo add confluentinc https://packages.confluent.io/helm
helm repo update
helm upgrade --install confluent-operator confluentinc/confluent-for-kubernetes
```

Following the above steps will set up the namespace and run the CfK Operator.  If you run `kubectl get pods` now, you will see:

```
NAME                                  READY   STATUS    RESTARTS   AGE
confluent-operator-7cc4fdc656-sfvhs   1/1     Running   0          30s
```

We're going to deploy the components one-by-one...  Using the yaml files in this GIthub repository: https://github.com/ableasdale/confluent-cfk-examples/tree/main/walkthrough

#### Zookeeper

The basic resource definition (`zookeeper.yaml`) looks like this:

```yaml
apiVersion: platform.confluent.io/v1beta1
kind: Zookeeper
metadata:
  name: zookeeper
  namespace: confluent
spec:
  replicas: 3
  image:
    application: confluentinc/cp-zookeeper:7.4.0
    init: confluentinc/confluent-init-container:2.6.0
  dataVolumeCapacity: 10Gi
  logVolumeCapacity: 10Gi
  
```

```bash
kubectl apply -f zookeeper.yaml
```

You will see:

```
zookeeper.platform.confluent.io/zookeeper created
```

Running `kubectl get pods` should show the three zookeeper instances:

```
NAME                                  READY   STATUS    RESTARTS   AGE
zookeeper-0                           0/1     Running   0          106s
zookeeper-1                           1/1     Running   0          106s
zookeeper-2                           1/1     Running   0          106s
```

Looking at the yaml file for zookeeper, we can see that the specification requires three replicas and that each instance has a data volume and a log volume.   

We can confirm this by reviewing the Persistent Volumes (PV):

```bash
kubectl get pv --sort-by=.spec.capacity.storage
```

You will see something similar to this:

```
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                          STORAGECLASS   REASON   AGE
pvc-06c1c94b-dc0f-4b69-965a-2022f79926c2   10Gi       RWO            Delete           Bound    confluent/txnlog-zookeeper-2   standard                4m36s
pvc-49e6e1e5-6673-43a4-b209-c73f934a0f0e   10Gi       RWO            Delete           Bound    confluent/txnlog-zookeeper-0   standard                4m36s
pvc-583de40d-3795-481b-a04c-8a4294b67c4a   10Gi       RWO            Delete           Bound    confluent/data-zookeeper-1     standard                4m36s
pvc-6828292c-48da-4fa2-a45d-d923c28b3bbc   10Gi       RWO            Delete           Bound    confluent/data-zookeeper-2     standard                4m36s
pvc-6f997abf-1a09-4ff9-bca5-4f49db5e3347   10Gi       RWO            Delete           Bound    confluent/txnlog-zookeeper-1   standard                4m36s
pvc-f05c7020-81de-4b13-98b7-68b813af9576   10Gi       RWO            Delete           Bound    confluent/data-zookeeper-0     standard                4m36s
```

Let's kill a Pod and see what happens:

```bash
kubectl delete pod zookeeper-0
```

`kubectl` confirms that the pod was deleted:

```
pod "zookeeper-0" deleted
```

Now let's run `kubectl get pods` and note that another Pod is immediately initialised to replace the deleted one (note the age difference): 

```bash
kubectl get pods
NAME                                  READY   STATUS            RESTARTS   AGE
confluent-operator-7cc4fdc656-sfvhs   1/1     Running           0          141m
zookeeper-0                           0/1     PodInitializing   0          2s
zookeeper-1                           1/1     Running           0          140m
zookeeper-2                           1/1     Running           0          140m
```

What happens if we scale down from the yaml default `replicas: 3` configuration and configure the resource for 1 replica?

```bash
kubectl scale zookeeper zookeeper --replicas=1
```

Immediately if we do this and we look at the pods, we now see a single pod.

Let's reinstate quorum by re-applying `zookeeper.yaml`:

```bash
kubectl apply -f zookeeper.yaml
```

Note that you now see three Zookeeper nodes.

### Kafka Brokers

Let's add the brokers (`brokers.yaml`):

```yaml
apiVersion: platform.confluent.io/v1beta1
kind: Kafka
metadata:
  name: kafka
  namespace: confluent
spec:
  replicas: 3
  image:
    application: confluentinc/cp-server:7.4.0
    init: confluentinc/confluent-init-container:2.6.0
  dataVolumeCapacity: 100Gi
  metricReporter:
    enabled: true
  dependencies:
    zookeeper:
      endpoint: zookeeper.confluent.svc.cluster.local:2181
```

We can introduce them by running: 

```bash
kubectl apply -f brokers.yaml
```

Let's confirm that we have 3 brokers:

```
kubectl get pods
NAME                                  READY   STATUS            RESTARTS   AGE
kafka-0                               0/1     Init:0/1          0          3s
kafka-1                               0/1     Init:0/1          0          3s
kafka-2                               0/1     PodInitializing   0          3s
```

Note from the `yaml` file that the spec contains a `dataVolumeCapacity` property (set to `100Gi`):

```bash
kubectl get pv --sort-by=.spec.capacity.storage
```

Now we see the volumes backing the 3 brokers listed:

```
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                          STORAGECLASS   REASON   AGE
pvc-041ec3c6-5868-4227-a91a-69fd8f31d1a3   100Gi      RWO            Delete           Bound    confluent/data0-kafka-0        standard                111s
pvc-3c649bb7-f778-44b5-b3e4-525e553e3d44   100Gi      RWO            Delete           Bound    confluent/data0-kafka-1        standard                111s
pvc-60a77fbf-5093-4cb1-b5f5-ece34288192a   100Gi      RWO            Delete           Bound    confluent/data0-kafka-2        standard                111s
```

Let's make sure the brokers are connected to zookeeper:

```bash
kubectl exec -it zookeeper-0 -- bash
zookeeper-shell localhost:2181
```

We can run a few tests in Zookeeper to confirm that all brokers are registered as expected:

```bash
Connecting to localhost:2181
Welcome to ZooKeeper!
JLine support is disabled

WATCHER::

WatchedEvent state:SyncConnected type:None path:null
get /kafka-confluent/controller
{"version":2,"brokerid":0,"timestamp":"1685560188067","kraftControllerEpoch":-1}
get /kafka-confluent/cluster/id
{"version":"1","id":"TBOvRxhYTBumiqiGheNurA"}
ls /kafka-confluent/brokers/ids
[0, 1, 2]
```

### Schema Registry

Let's review the `yaml` file (`schema-registry.yaml`):

```yaml
apiVersion: platform.confluent.io/v1beta1
kind: SchemaRegistry
metadata:
  name: schemaregistry
  namespace: confluent
spec:
  replicas: 1
  image:
    application: confluentinc/cp-schema-registry:7.4.0
    init: confluentinc/confluent-init-container:2.6.0
    
```

Note that Schema Registry does not create any additional data volumes; all data is stored in topics managed by the brokers.

Let's apply the `yaml`:

```bash
kubectl apply -f schema-registry.yaml
```

Running `kubectl get pods` will show us the newly instantiated Schema Registry instance:

```
NAME                                  READY   STATUS            RESTARTS   AGE
schemaregistry-0                      0/1     PodInitializing   0          77s
```

Remember that we can use `kubectl describe services` to find out more about the network setup:

```bash
kubectl describe services schemaregistry-0
```

Note that the external port is 8081:

```
Port:              external  8081/TCP
TargetPort:        8081/TCP
Endpoints:         10.244.0.12:8081
```

We can set up port forwarding for that port:

```bash
kubectl port-forward schemaregistry-0 8081:8081
```

And run a `curl` call against the instance:

```bash
curl -s -XGET http://localhost:8081/schemas/types | jq
```

This will confirm that the ReST API is functioning - while there are no schemas yet, we can see a list of supported types to show that everything is working:

```json
[
  "JSON",
  "PROTOBUF",
  "AVRO"
]
```


### ksqlDB

Here's some minimal `yaml` configuration for ksqlDB (`ksqldb.yaml`):

```yaml
apiVersion: platform.confluent.io/v1beta1
kind: KsqlDB
metadata:
  name: ksqldb
  namespace: confluent
spec:
  replicas: 1
  image:
    application: confluentinc/cp-ksqldb-server:7.4.0
    init: confluentinc/confluent-init-container:2.6.0
  dataVolumeCapacity: 10Gi
  
```

Let's apply the configuration:

```bash
kubectl apply -f ksqldb.yaml
```

We see a new entry when we get our list of pods:

```bash
kubectl get pods
NAME                                  READY   STATUS            RESTARTS   AGE
ksqldb-0                              0/1     PodInitializing   0          29s
```

Note from the `yaml` file that the spec contains a `dataVolumeCapacity` property (set to `10Gi`):

```bash
kubectl get pv --sort-by=.spec.capacity.storage
```

Now we see the volume bound to the ksqlDB instance:

```
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                          STORAGECLASS   REASON   AGE
pvc-fb9716a9-a9b4-41fd-895c-b57316958e37   10Gi       RWO            Delete           Bound    confluent/data-ksqldb-0        standard                4m41s
```

Let's describe the service:

```bash
kubectl describe services ksqldb-0
```

We can set up port forwarding for that port:

```bash
kubectl port-forward ksqldb-0 8088:8088
```

Let's access the web UI (using cURL):

```bash
curl -s -XGET http://localhost:8088/info | jq
```

```json
{
  "KsqlServerInfo": {
    "version": "7.4.0",
    "kafkaClusterId": "TBOvRxhYTBumiqiGheNurA",
    "ksqlServiceId": "confluent.ksqldb_",
    "serverStatus": "RUNNING"
  }
}
```

### Kafka Connect

Introducing a Connect node into the group of pods (`connect.yaml`):

```yaml
apiVersion: platform.confluent.io/v1beta1
kind: Connect
metadata:
  name: connect
  namespace: confluent
spec:
  replicas: 1
  image:
    application: confluentinc/cp-server-connect:7.4.0
    init: confluentinc/confluent-init-container:2.6.0
  dependencies:
    kafka:
      bootstrapEndpoint: kafka:9071
      
```

Applying the configuration:

```bash
kubectl apply -f connect.yaml
```

Let's see what port we can use:

```bash
kubectl describe services connect-0
```

We can see port 8083 is the ReST API port:

```
Port:              external  8083/TCP
TargetPort:        8083/TCP
```

Set up port forwarding:

```bash
kubectl port-forward connect-0 8083:8083
```

Get the initial cluster info:

```bash
curl -s -XGET http://localhost:8083 | jq
```

```json
{
  "version": "7.4.0-ce",
  "commit": "aee97a585bd06866",
  "kafka_cluster_id": "TBOvRxhYTBumiqiGheNurA"
}
```

Let's get a list of connector plugins:

```bash
curl -s -XGET http://localhost:8083/connector-plugins | jq
```

Note that there are only 3 source connectors installed; others would need to be added by extending this image:

```json
[
  {
    "class": "org.apache.kafka.connect.mirror.MirrorCheckpointConnector",
    "type": "source",
    "version": "7.4.0-ce"
  },
  {
    "class": "org.apache.kafka.connect.mirror.MirrorHeartbeatConnector",
    "type": "source",
    "version": "7.4.0-ce"
  },
  {
    "class": "org.apache.kafka.connect.mirror.MirrorSourceConnector",
    "type": "source",
    "version": "7.4.0-ce"
  }
]
```

### Confluent Control Center (C3)

Here's the resource description (`controlcenter.yaml`):

```yaml
apiVersion: platform.confluent.io/v1beta1
kind: ControlCenter
metadata:
  name: controlcenter
  namespace: confluent
spec:
  replicas: 1
  image:
    application: confluentinc/cp-enterprise-control-center:7.4.0
    init: confluentinc/confluent-init-container:2.6.0
  dataVolumeCapacity: 10Gi
  dependencies:
    schemaRegistry:
      url: http://schemaregistry.confluent.svc.cluster.local:8081
    ksqldb:
    - name: ksqldb
      url: http://ksqldb.confluent.svc.cluster.local:8088
    connect:
    - name: connect
      url: http://connect.confluent.svc.cluster.local:8083

```

Add the Control Center into the pods:

```bash
kubectl apply -f controlcenter.yaml
```

Let's configure port forwarding on port 9021:

```bash
kubectl port-forward controlcenter-0 9021:9021
```

### ReST Proxy

Here's the contents of `restproxy.yaml`:

```yaml
apiVersion: platform.confluent.io/v1beta1
kind: KafkaRestProxy
metadata:
  name: kafkarestproxy
  namespace: confluent
spec:
  replicas: 1
  image:
    application: confluentinc/cp-kafka-rest:7.4.0
    init: confluentinc/confluent-init-container:2.6.0
  dependencies:
    schemaRegistry:
      url: http://schemaregistry.confluent.svc.cluster.local:8081
      
```

We're going to apply the resource:

```bash
kubectl apply -f restproxy.yaml
```

Let's confirm the port for ReST Proxy is 8082:

```bash
kubectl describe services kafkarestproxy-0
```

Set up port forwarding for testing:

```bash
kubectl port-forward kafkarestproxy-0 8082:8082
```

We can use cURL to inspect the cluster information:

```
curl -s -XGET localhost:8082/v3/clusters | jq
```

Let's get a list of topics:

```bash
curl -s -XGET localhost:8082/topics | jq
```

### Create a topic

We're going to use the `topic.yaml` to create the topic:

```yaml
apiVersion: platform.confluent.io/v1beta1
kind: KafkaTopic
metadata:
  name: ab-example-topic
  namespace: confluent
spec:
  replicas: 1
  partitionCount: 3
  kafkaClusterRef:
    name: kafka
  configs:
    min.insync.replicas: "1"
    retention.ms: "86400000"

```

This can be applied to create the topic:

```bash
kubectl apply -f topic.yaml
```

We can use ReST Proxy to access the details:

```bash
curl -s -XGET localhost:8082/topics/ab-example-topic | jq
```

### Ubuntu Tools container

Here's the definition for `ubuntu-tools.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-pod
spec:
  containers:
    - name: ubuntu-container
      image: alexjbleasdale/ubuntu-tools:0.0.2
      env:
        - name: CONFLUENT_HOME
          value: /usr
        - name: JAVA_HOME
          value: /usr/lib/jvm/java-17-openjdk-amd64/
        - name: TERM
          value: xterm-256color
      command: ["sleep", "infinity"]
      resources:
        requests:
          memory: "64Mi"
          cpu: "100m"
        limits:
          memory: "128Mi"
          cpu: "200m"
      volumeMounts:
        - name: my-volume
          mountPath: /var/my-data
  volumes:
    - name: my-volume
      emptyDir: {}
```

Apply the `yaml` file:

```bash
kubectl apply -f ubuntu-tools.yaml
```

Connect to the tools instance (`ubuntu-pod`):

```bash
kubectl exec ubuntu-pod -it -- bash
```


### JMXTerm (TODO - not working!)

Connect to the `kafka-0` instance:

```bash
kubectl describe pod kafka-0 | grep ID
```

Note that the **Container ID** in this case is the second one listed (starting with b18d):

```
    Container ID:  docker://5c29397fb0821783e61713ec77a5178877c6abb27b1de180cd5e4007895d23a0
    Image ID:      docker-pullable://confluentinc/confluent-init-container@sha256:546dc13c76cdf36968961b5ed930aa74af3c26a95ad1a72a8874fa937edd4ab6
      CAAS_POD_ID:       kafka-0 (v1:metadata.name)
    Container ID:  docker://b18df0804ebcf1b034a6ff044727da28c7af5793f6c5448389064692dd35e816
    Image ID:      docker-pullable://confluentinc/cp-server@sha256:adf4cfdf99b1b526f7270aa951ad2891192a06d0ee6260e53ee66d441176fd4d
      CAAS_POD_ID:    kafka-0 (v1:metadata.name)
```

We're going to attempt to `ssh` to the instance as root:

```bash
docker exec -it -u root b18df0804ebcf1b034a6ff044727da28c7af5793f6c5448389064692dd35e816 /bin/bash
```

Note that this doesn't seem to work anymore.  Let's use a `kubectl` extension called `exec-as` instead:

```bash
kubectl krew install exec-as
```

Now let's try to `ssh` (as root) into the instance:

```bash
kubectl exec-as -u root kafka-0 -- /bin/bash
```

Doesn't seem to work either...  Try:

```bash
minikube ssh docker container ls | grep kafka-0
```

This gives us a docker container ID:

```bash
b18df0804ebc   confluentinc/cp-server                      "/bin/sh -xc /mnt/co…"   3 hours ago         Up 3 hours                   k8s_kafka_kafka-0_confluent_e3d7e8aa-3909-4786-baa2-c75cae84545b_0
919b597e381b   registry.k8s.io/pause:3.9                   "/pause"                 3 hours ago         Up 3 hours                   k8s_POD_kafka-0_confluent_e3d7e8aa-3909-4786-baa2-c75cae84545b_0
```

Let's try to connect to it:

```bash
minikube ssh "docker container exec -it -u 0 b18df0804ebc /bin/bash"
```

Let's install JMXTerm:

```bash
cd /tmp/
wget https://github.com/jiaqi/jmxterm/releases/download/v1.0.4/jmxterm-1.0.4-uber.jar
```

And let's run JMXTerm:

```bash
java -jar jmxterm-1.0.4-uber.jar
```

Okay - can't connect to the JVMs!  Not sure why...?  Is Kafka running?!?

```bash
kubectl logs kafka-0
```

### Monitoring with Jolokia

What about using Jolokia for JMX metrics? 

```bash
kubectl describe services kafka-0
```

Note that port 7777 is associated with Jolokia - let's set up port forwarding and take a look:

```bash
kubectl port-forward kafka-0 7777:7777
```

Let's try to access something in Jolokia:

```
curl -s -XGET http://localhost:7777/jolokia/version | jq
```

```json
{
  "request": {
    "type": "version"
  },
  "value": {
    "agent": "1.7.1",
    "protocol": "7.2",
    "config": {
      "listenForHttpService": "true",
      "maxCollectionSize": "0",
      "authIgnoreCerts": "false",
      "agentId": "10.244.0.17-1-9573584-jvm",
      "agentType": "jvm",
      "policyLocation": "classpath:/jolokia-access.xml",
      "agentContext": "/jolokia",
      "mimeType": "text/plain",
      "discoveryEnabled": "true",
      "streaming": "true",
      "historyMaxEntries": "10",
      "allowDnsReverseLookup": "true",
      "maxObjects": "0",
      "debug": "false",
      "serializeException": "false",
      "multicastGroup": "239.192.48.84",
      "maxDepth": "15",
      "authMode": "basic",
      "authMatch": "any",
      "canonicalNaming": "true",
      "allowErrorDetails": "true",
      "realm": "jolokia",
      "includeStackTrace": "true",
      "multicastPort": "24884",
      "useRestrictorService": "false",
      "debugMaxEntries": "100"
    },
    "info": {
      "product": "jetty",
      "vendor": "Eclipse",
      "version": "9.4.51.v20230217"
    }
  },
  "timestamp": 1685571199,
  "status": 200
}
```

```bash
curl -s -XGET http://localhost:7777/jolokia/list | jq
```

This will give you a (very long) list of MBeans that you can query.

```bash
curl -s -XGET http://localhost:7777/jolokia/read/java.lang:type=Memory/HeapMemoryUsage/used | jq
```

```json
{
  "request": {
    "path": "used",
    "mbean": "java.lang:type=Memory",
    "attribute": "HeapMemoryUsage",
    "type": "read"
  },
  "value": 496541184,
  "timestamp": 1685615331,
  "status": 200
}
```

## Notes

confluent kafka topic consume -b confluent-audit-log-events
confluent kafka broker describe --all --bootstrap kafka-0:9021

confluent kafka topic produce test-topic --bootstrap kafka-0:9092

kubectl describe pod kafka-0

kubectl exec kafka-0 -it -- bash