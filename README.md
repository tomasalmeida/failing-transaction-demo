# Failing transaction demo

The idea of this demo show that a failed transaction blocks the consumption in the consumer. 

## Start the clusters

```shell
docker-compose up --build -d
```

### Create the topic `product`

```shell
docker-compose exec mainKafka-1 kafka-topics --bootstrap-server mainKafka-1:19092 --topic product --create --partitions 3 --replication-factor 1
```

## Start consumers

### Read commited

```shell
docker-compose exec mainKafka-1 \
        kafka-console-consumer --bootstrap-server mainKafka-1:19092 \
        --consumer-property "isolation.level=read_committed" \
        --property print.partition=true \
        --property print.key=true \
        --topic product \
        --group committed_group
```

### Read uncommited

```shell
docker-compose exec mainKafka-1 \
        kafka-console-consumer --bootstrap-server mainKafka-1:19092 \
        --consumer-property "isolation.level=read_uncommitted" \
        --property print.partition=true \
        --property print.key=true \
        --topic product \
        --group uncommitted_group
```

## Step 1: Produce transaction data

Produce data. We are using a versioned application provide by confluent in the following [link](https://developer.confluent.io/courses/architecture/transactions-hands-on/).

Last argument is the `transactional.id` and also used as key

```shell
docker exec -it java_app \
java -jar build/libs/transactional-producer.jar build/resources/main/client-main.properties tt1
```
Press enter five times. The output of the command will be similar to

```shell
*** Begin Transaction ***
*** transactional.id tt3 ***
Sent key: tt3, value: 2024-04-02T15:15:16.710308

Sent key: tt3, value: 2024-04-02T15:15:18.847528

Sent key: tt3, value: 2024-04-02T15:15:19.100039

Sent key: tt3, value: 2024-04-02T15:15:19.324631

Sent key: tt3, value: 2024-04-02T15:15:19.819573

*** Commit Transaction ***
```

> **NOTE:**: the data is shown in both consumers

## Step 2: Produce transaction data and do not commit the transaction

Produce data again but this time we will not commit the transaction. For example, just press enter two times.

```shell
docker exec -it java_app \
java -jar build/libs/transactional-producer.jar build/resources/main/client-main.properties tt1
```

**IMPORTANT:** Stop the producer without committing the transaction to simulate a crash

## Step 3: Produce transaction data and commit the transaction in the same partition

```shell
docker exec -it java_app \
java -jar build/libs/transactional-producer.jar build/resources/main/client-main.properties tt2
```

## Step 4: Produce transaction data and commit the transaction in other partition

```shell
docker exec -it java_app \
java -jar build/libs/transactional-producer.jar build/resources/main/client-main.properties tt3
```

Hit enter 5 times to commit the transaction.

### Conclusions

Check the lag

```shell
docker-compose exec mainKafka-1 kafka-consumer-groups --bootstrap-server mainKafka-1:19092 --describe --all-groups
```

1. Data is shown only in the uncommitted consumer. Other consumer is blocked in the partition with the open transaction
   2. Lag is increasing as there is data not consumed.
3. Others partitions are not blocked


### Shutdown

1. Stop all consumers
2. Stop and clean the cluster

```shell
 docker-compose down -v && docker system prune
```