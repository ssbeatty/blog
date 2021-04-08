---
title: 使用docker搭建Kafka集群
date: 2021-04-08T15:40:00+08:00
lastmod: 2021-04-08T15:40:00+08:00
author: sasaba
cover: /img/使用docker搭建Kafka集群.jpg
images:
  - /img/使用docker搭建Kafka集群.jpg
categories:
  - 云计算
tags:
  - 技术
  - 配置
---

怎么用docker搭建kafka集群，用于测试。

<!--more-->
  工作中需要用到一个kafka集群做开发调试任务，因此记录下搭建配置文件。


### 配置
> docker-compose.yaml  
```yaml
version: '2'
services:
    zookeeper:
        image: wurstmeister/zookeeper
        container_name: zookeeper
        restart: always
        ports:
          - "2181:2181"
          
    kafka1:
        image: wurstmeister/kafka
        container_name: kafka1
        restart: always
        ports:
          - "9092:9092"
        environment:
          - KAFKA_BROKER_ID=0
          - KAFKA_ZOOKEEPER_CONNECT=zookeeper:2181
          - KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://{宿主机IP}:9092
          - KAFKA_LISTENERS=PLAINTEXT://0.0.0.0:9092

    kafka2:
        image: wurstmeister/kafka
        container_name: kafka2
        restart: always
        ports:
          - "9093:9093"
        environment:
          - KAFKA_BROKER_ID=1
          - KAFKA_ZOOKEEPER_CONNECT=zookeeper:2181
          - KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://{宿主机IP}:9093
          - KAFKA_LISTENERS=PLAINTEXT://0.0.0.0:9093

    kafka3:
        image: wurstmeister/kafka
        container_name: kafka3
        restart: always
        ports:
          - "9094:9094"
        environment:
          - KAFKA_BROKER_ID=2
          - KAFKA_ZOOKEEPER_CONNECT=zookeeper:2181
          - KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://{宿主机IP}:9094
          - KAFKA_LISTENERS=PLAINTEXT://0.0.0.0:9094
```

### 使用
  附上go的测试脚本，基于Shopify/sarama。

> 生产者

```go
package main

import (
	"fmt"
	"github.com/Shopify/sarama"
)

func main() {
	config := sarama.NewConfig()
	config.Producer.RequiredAcks = sarama.WaitForAll          //赋值为-1：这意味着producer在follower副本确认接收到数据后才算一次发送完成。
	config.Producer.Partitioner = sarama.NewRandomPartitioner //写到随机分区中，默认设置8个分区
	config.Producer.Return.Successes = true
	msg := &sarama.ProducerMessage{}
	msg.Topic = `nginx_log`
	msg.Value = sarama.StringEncoder("this is a good test")
	client, err := sarama.NewSyncProducer([]string{"10.1.1.81:9092"}, config)
	if err != nil {
		fmt.Println("producer close err, ", err)
		return
	}
	defer client.Close()
	pid, offset, err := client.SendMessage(msg)

	if err != nil {
		fmt.Println("send message failed, ", err)
		return
	}
	fmt.Printf("分区ID:%v, offset:%v \n", pid, offset)
}
```

> 消费者
```go
package main

import (
	"fmt"
	"github.com/Shopify/sarama"

)

func main() {
	config := sarama.NewConfig()
	config.Consumer.Return.Errors = true
	config.Version = sarama.V0_11_0_2

	// consumer
	consumer, err := sarama.NewConsumer([]string{"10.1.1.81:9092"}, config)
	if err != nil {
		fmt.Printf("consumer_test create consumer error %s\n", err.Error())
		return
	}

	defer consumer.Close()

	partition_consumer, err := consumer.ConsumePartition("test", 0, sarama.OffsetOldest)
	if err != nil {
		fmt.Printf("try create partition_consumer error %s\n", err.Error())
		return
	}
	defer partition_consumer.Close()

	for {
		select {
		case msg := <-partition_consumer.Messages():
			fmt.Printf("msg offset: %d, partition: %d, timestamp: %s, value: %s\n",
				msg.Offset, msg.Partition, msg.Timestamp.String(), string(msg.Value))

		case err := <-partition_consumer.Errors():
			fmt.Printf("err :%s\n", err.Error())
		}
	}

}
```