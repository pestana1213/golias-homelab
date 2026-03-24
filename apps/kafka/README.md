# Kafka (Strimzi)

3-node Kafka cluster in KRaft mode (no ZooKeeper), managed by the Strimzi operator.

## Listeners

| Name     | Port  | Type     | TLS |
|----------|-------|----------|-----|
| plain    | 9092  | internal | no  |
| tls      | 9093  | internal | yes |
| external | 9094  | nodeport | no  |

External bootstrap: `192.168.1.225:30748`

## Produce messages

```bash
kubectl run kafka-producer -ti --image=quay.io/strimzi/kafka:0.51.0-kafka-4.2.0 \
  --rm=true --restart=Never -n kafka -- bin/kafka-console-producer.sh \
  --bootstrap-server kafka-kafka-bootstrap:9092 \
  --topic golias_general_metrics_topic
```

## Consume messages

```bash
kubectl run kafka-consumer -ti --image=quay.io/strimzi/kafka:0.51.0-kafka-4.2.0 \
  --rm=true --restart=Never -n kafka -- bin/kafka-console-consumer.sh \
  --bootstrap-server kafka-kafka-bootstrap:9092 \
  --topic golias_general_metrics_topic \
  --from-beginning
```
