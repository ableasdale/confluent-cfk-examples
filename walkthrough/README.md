# Setting up Confluent for Kubernetes on minikube

This is intended to be a walkthrough of the Confluent for Kubernetes quickstart.  

We will start by looking at the components that are installed by default and then look to start making modifications and additions with a view to demonstrating the workflow.

Start by installing minikube - follow the guide here: https://minikube.sigs.k8s.io/docs/start/

Then install helm (https://helm.sh/) on OS X we're going to use brew for this:

```bash
brew install helm
```

Let's also install `cubectx`:

```bash
brew install kubectx
```

### Configuring `minikube`

Ensure minikube is successfully installed:

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
docker build . -t alexjbleasdale/ubuntu-tools:0.0.1
```

And then push it:

```bash
docker push alexjbleasdale/ubuntu-tools:0.0.1
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
      image: your-custom-image:tag
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

### Convenience methods

TODO - docs / support bundle

### Sidecar Pod

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


## notes

confluent kafka topic consume -b confluent-audit-log-events
confluent kafka broker describe --all --bootstrap kafka-0:9021

confluent kafka topic produce test-topic --bootstrap kafka-0:9092