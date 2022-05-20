# Confluent for Kubernetes (CfK) Examples

## Simple Broker and Zookeeper

Start the `confluent-operator` pod (see the quick start guide for more information)

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