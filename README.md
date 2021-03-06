# Confluent for Kubernetes (CfK) Examples

## Simple Broker and Zookeeper

Start the `confluent-operator` pod (see the quick start guide for more information):

```bash
kubectl create namespace confluent
kubectl config set-context --current --namespace confluent
helm repo add confluentinc https://packages.confluent.io/helm
helm repo update
helm upgrade --install confluent-operator confluentinc/confluent-for-kubernetes
```

Start Zookeeper and the broker:

```bash
cd simple-broker-and-zookeeper
kubectl apply -f zookeeper.yaml
kubectl apply -f broker.yaml
```

Confirm the pods (`kafka-0` and `zookeeper-0`) are up:

```bash
kubectl get pods
```

You should see:

```bash
NAME                                  READY   STATUS    RESTARTS   AGE
confluent-operator-64c5c5756d-66f4m   1/1     Running   0          101m
kafka-0                               0/1     Running   0          6s
zookeeper-0                           1/1     Running   0          58s
```

### Sanity check for Zookeeper

To test zookeeper we will ssh into the instance and run `zookeeper-shell`:

```bash
kubectl --namespace=confluent exec -it zookeeper-0 -- bash
zookeeper-shell localhost:2181
ls /
ls /kafka-confluent/brokers
get /kafka-confluent/controller
```

You should see evidence of a controller broker:

```bash
{"version":1,"brokerid":0,"timestamp":"1653049092079"}
```

## Sanity check for the Kafka broker

```bash
kubectl --namespace=confluent exec -it kafka-0 -- bash
kafka-console-producer --bootstrap-server localhost:9092 --topic test
```

Produce some content and Ctrl+C when done.

Check the `test` topic:

```bash
kafka-topics --bootstrap-server localhost:9092 --describe --topic test
```

Consume from the `test` topic:

```bash
kafka-console-consumer --bootstrap-server localhost:9092 --topic test
```

## Tail the logs on the broker / zookeeper

```bash
kubectl logs --follow kafka-0
kubectl logs --follow zookeeper-0
```

## Create the example topic

```bash
kubectl apply -f example-topic.yaml
```

## Confirm on the broker that the topic was created

```bash
kubectl --namespace=confluent exec -it kafka-0 -- bash
kafka-topics --bootstrap-server localhost:9092 --describe --topic example-topic
```

You should see:

```bash
Topic: example-topic	PartitionCount: 4	ReplicationFactor: 1	Configs: min.insync.replicas=1,segment.bytes=1073741824,retention.ms=86400000,message.format.version=2.6-IV0
	Topic: example-topic	Partition: 0	Leader: 0	Replicas: 0	Isr: 0	Offline:
	Topic: example-topic	Partition: 1	Leader: 0	Replicas: 0	Isr: 0	Offline:
	Topic: example-topic	Partition: 2	Leader: 0	Replicas: 0	Isr: 0	Offline:
	Topic: example-topic	Partition: 3	Leader: 0	Replicas: 0	Isr: 0	Offline:
```

## Adding mTLS

Create the secret from `secret.txt` in the `single-broker-with-tls-and-zookeeper` directory:

```bash
cd single-broker-with-tls-and-zookeeper
kubectl create secret generic credential \
  --from-file=my_credentials=secret.txt
```

Create the CA

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=foo.bar.com"
```

then:

```bash
kubectl create secret tls ca-pair-sslcerts --key="tls.key" --cert="tls.crt"
```

Or:

```bash
cat tls.crt | base64
cat tls.key | base64
```

And add the output into `tls.yaml` and apply to create the secret

Spin up the broker (and Zookeeper if needed):

```bash
kubectl apply -f ../simple-broker-and-zookeeper/zookeeper.yaml
kubectl apply -f broker.yaml
```

## Older pieces

```bash
kubectl create secret tls ca-pair-sslcerts \
  --cert=/path/to/ca.pem \
  --key=/path/to/ca-key.pem
```

```bash
kubectl get secret confluent-operator-licensing -o jsonpath='{.data}'
```
### Debugging

```bash
kubectl --namespace=confluent exec -it kafka-0 -- bash
more /opt/confluentinc/etc/kafka/kafka.properties
```

### Something failed? Check the Operator!

```bash
kubectl logs confluent-operator-64c5c5756d-vwb9s
```

### List recent events in time order 

```bash
kubectl get events --sort-by=.metadata.creationTimestamp
```

### Nuke Minikube

```bash
minikube delete && minikube start --vm-driver kvm2
```

### get rolebindings

```bash
kubectl get confluentrolebindings
kubectl api-resources | grep bindings
```