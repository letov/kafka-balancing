# Kafka balancing

## Задание 1. Балансировка партиций и диагностика кластера

1. Создайте новый топик balanced_topic с 8 партициями и фактором репликации 3.
```
kafka-topics.sh --bootstrap-server localhost:9092 --topic balanced_topic --create --partitions 8 --replication-factor 3
``` 
2. Определите текущее распределение партиций.
```
kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic balanced_topic
```
```
Topic: balanced_topic   TopicId: 1DaugQlHSR-Hy2nzVcPkgg PartitionCount: 8       ReplicationFactor: 3    Configs: 
        Topic: balanced_topic   Partition: 0    Leader: 2       Replicas: 2,0,1 Isr: 2,0,1
        Topic: balanced_topic   Partition: 1    Leader: 0       Replicas: 0,1,2 Isr: 0,1,2
        Topic: balanced_topic   Partition: 2    Leader: 1       Replicas: 1,2,0 Isr: 1,2,0
        Topic: balanced_topic   Partition: 3    Leader: 0       Replicas: 0,1,2 Isr: 0,1,2
        Topic: balanced_topic   Partition: 4    Leader: 1       Replicas: 1,2,0 Isr: 1,2,0
        Topic: balanced_topic   Partition: 5    Leader: 2       Replicas: 2,0,1 Isr: 2,0,1
        Topic: balanced_topic   Partition: 6    Leader: 0       Replicas: 0,1,2 Isr: 0,1,2
        Topic: balanced_topic   Partition: 7    Leader: 1       Replicas: 1,2,0 Isr: 1,2,0
```
3. Создайте JSON-файл reassignment.json для перераспределения партиций.
```
echo '{
  "version": 1,
  "partitions": [
    {"topic": "balanced_topic", "partition": 0, "replicas": [0, 1], "log_dirs": ["any", "any"]},
    {"topic": "balanced_topic", "partition": 1, "replicas": [1, 2], "log_dirs": ["any", "any"]},
    {"topic": "balanced_topic", "partition": 2, "replicas": [2, 0], "log_dirs": ["any", "any"]},
    {"topic": "balanced_topic", "partition": 3, "replicas": [2, 0], "log_dirs": ["any", "any"]},
    {"topic": "balanced_topic", "partition": 4, "replicas": [0, 2], "log_dirs": ["any", "any"]},
    {"topic": "balanced_topic", "partition": 5, "replicas": [1, 0], "log_dirs": ["any", "any"]},
    {"topic": "balanced_topic", "partition": 6, "replicas": [2, 0], "log_dirs": ["any", "any"]},
    {"topic": "balanced_topic", "partition": 7, "replicas": [0, 1], "log_dirs": ["any", "any"]}
  ]
}' > /tmp/reassignment.json
```
4. Перераспределите партиции.
```
kafka-reassign-partitions.sh \
--bootstrap-server localhost:9092 \
--broker-list "0,1,2" \
--topics-to-move-json-file "/tmp/reassignment.json" \
--generate 
```
```
kafka-reassign-partitions.sh \
--bootstrap-server localhost:9092 \
--reassignment-json-file "/tmp/reassignment.json" \
--execute
```
5. Проверьте статус перераспределения.
```
Successfully started partition reassignments for balanced_topic-0,balanced_topic-1,balanced_topic-2,balanced_topic-3,balanced_topic-4,balanced_topic-5,balanced_topic-6,balanced_topic-7
```
6. Убедитесь, что конфигурация изменилась.
```
kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic balanced_topic
```
```
Topic: balanced_topic   TopicId: 1DaugQlHSR-Hy2nzVcPkgg PartitionCount: 8       ReplicationFactor: 2    Configs: 
        Topic: balanced_topic   Partition: 0    Leader: 0       Replicas: 0,1   Isr: 0,1
        Topic: balanced_topic   Partition: 1    Leader: 1       Replicas: 1,2   Isr: 1,2
        Topic: balanced_topic   Partition: 2    Leader: 2       Replicas: 2,0   Isr: 2,0
        Topic: balanced_topic   Partition: 3    Leader: 2       Replicas: 2,0   Isr: 0,2
        Topic: balanced_topic   Partition: 4    Leader: 0       Replicas: 0,2   Isr: 2,0
        Topic: balanced_topic   Partition: 5    Leader: 1       Replicas: 1,0   Isr: 0,1
        Topic: balanced_topic   Partition: 6    Leader: 2       Replicas: 2,0   Isr: 0,2
        Topic: balanced_topic   Partition: 7    Leader: 0       Replicas: 0,1   Isr: 1,0
```
7. Смоделируйте сбой брокера:
```
docker stop kafka-1
```
8. Проверьте состояние топиков после сбоя.
```
Topic: balanced_topic   TopicId: 1DaugQlHSR-Hy2nzVcPkgg PartitionCount: 8       ReplicationFactor: 2    Configs: 
        Topic: balanced_topic   Partition: 0    Leader: 0       Replicas: 0,1   Isr: 0
        Topic: balanced_topic   Partition: 1    Leader: 2       Replicas: 1,2   Isr: 2
        Topic: balanced_topic   Partition: 2    Leader: 2       Replicas: 2,0   Isr: 2,0
        Topic: balanced_topic   Partition: 3    Leader: 2       Replicas: 2,0   Isr: 0,2
        Topic: balanced_topic   Partition: 4    Leader: 0       Replicas: 0,2   Isr: 2,0
        Topic: balanced_topic   Partition: 5    Leader: 0       Replicas: 1,0   Isr: 0
        Topic: balanced_topic   Partition: 6    Leader: 2       Replicas: 2,0   Isr: 0,2
        Topic: balanced_topic   Partition: 7    Leader: 0       Replicas: 0,1   Isr: 0
```
9. Запустите брокер заново.
10. Проверьте, восстановилась ли синхронизация реплик.
```
Topic: balanced_topic   TopicId: 1DaugQlHSR-Hy2nzVcPkgg PartitionCount: 8       ReplicationFactor: 2    Configs: 
        Topic: balanced_topic   Partition: 0    Leader: 0       Replicas: 0,1   Isr: 0,1
        Topic: balanced_topic   Partition: 1    Leader: 2       Replicas: 1,2   Isr: 2,1
        Topic: balanced_topic   Partition: 2    Leader: 2       Replicas: 2,0   Isr: 2,0
        Topic: balanced_topic   Partition: 3    Leader: 2       Replicas: 2,0   Isr: 0,2
        Topic: balanced_topic   Partition: 4    Leader: 0       Replicas: 0,2   Isr: 2,0
        Topic: balanced_topic   Partition: 5    Leader: 0       Replicas: 1,0   Isr: 0,1
        Topic: balanced_topic   Partition: 6    Leader: 2       Replicas: 2,0   Isr: 0,2
        Topic: balanced_topic   Partition: 7    Leader: 0       Replicas: 0,1   Isr: 0,1
```