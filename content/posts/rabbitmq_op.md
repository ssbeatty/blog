---
title: RabbitMQ基础操作
date: 2021-01-26T09:50:05+08:00
lastmod: 2021-01-26T09:50:05+08:00
author: sasaba
cover: /some/Buy-Me-A-Coffee.jpg
images:
  - /some/Buy-Me-A-Coffee.jpg
categories:
  - 消息队列
tags:
  - Python
  - Mq
draft: true
---

RabbitMQ基础操作的python实现与例子。

<!--more-->

## 基础操作

### hello world

##### client

```python
import pika
connection = pika.BlockingConnection(
    pika.ConnectionParameters(
        host="192.168.8.101",
        port=5672,
        # 设置用户
        credentials=pika.credentials.PlainCredentials("admin", "199564"),
        # 设置虚拟机 相当于独立的mq
        virtual_host="/"
    ),
)
```



##### 生产者

```python
from mq.client import connection
# 打开一个通道
channel = connection.channel()

# queue, durable=False 是否持久, exclusive=False 是否独占(临时队列), auto_delete=False 自动删除
channel.queue_declare(queue="hello", durable=True)

# exchange 交换机 默认是'', routing_key 使用默认交换机要设置为队列
channel.basic_publish(exchange='', routing_key='hello',
                      body=b'Test message.')
connection.close()
```

##### 消费者

```python
from mq.client import connection

def message_callback(ch, method, properties, body):
    print("body is {}".format(body))

# 打开一个通道
channel = connection.channel()
# queue, durable=False 是否持久, exclusive=False 是否独占(临时队列), auto_delete=False 自动删除
channel.queue_declare(queue="hello", durable=True)
# queue, auto_ack 自动回复, on_message_callback 消息方法, consumer_tag 消费者标识
# 不自动回复的的话会一直存在
channel.basic_consume(queue="hello", auto_ack=True, on_message_callback=message_callback, consumer_tag="c1")
channel.start_consuming()
```

可以查看队列的消息数目

![](https://sasaba-1256963938.cos.ap-shanghai.myqcloud.com/uPic/20200903164933.png)

### 工作模式

##### 交换机类型

- fanout/发布订阅类型
- direct/Routing工作模式
- topic/通配符工作模式
- headers/headers工作模式

##### 工作队列模式

实际上就是hello world的版本，hello world多个消费者时，会采用轮训的方式依次把消息给消费者。

##### 发布订阅

```python
# 生产者
from mq.client import connection

QUEUE_INFORM_EMAIL = "queue_inform_email"
QUEUE_INFORM_SMS = "queue_inform_sms"
EXCHANGE_FANOUT_INFORM = "exchange_fanout_inform"

# 打开一个通道
channel = connection.channel()

# 声明两个队列
channel.queue_declare(queue=QUEUE_INFORM_SMS, durable=True)
channel.queue_declare(queue=QUEUE_INFORM_EMAIL, durable=True)

# 声明一个交换机
# exchange 名称, exchange_type="fanout"
channel.exchange_declare(EXCHANGE_FANOUT_INFORM, "fanout", durable=True)

# 交换机队列绑定
# queue 队列, exchange 交换机, routing_key  路由key 交换机根据路由key的值把消息发到指定队列 发布订阅为''
channel.queue_bind(QUEUE_INFORM_SMS, EXCHANGE_FANOUT_INFORM, "")
channel.queue_bind(QUEUE_INFORM_EMAIL, EXCHANGE_FANOUT_INFORM, "")


channel.basic_publish(exchange=EXCHANGE_FANOUT_INFORM, routing_key='',
                      body=b'Test 2222')
connection.close()
```

```python
# 消费者email
from mq.client import connection

QUEUE_INFORM_EMAIL = "queue_inform_email"
EXCHANGE_FANOUT_INFORM = "exchange_fanout_inform"

def message_callback(ch, method, properties, body):
    print("body is {}".format(body))

# 打开一个通道
channel = connection.channel()
# 声明两个队列
channel.queue_declare(queue=QUEUE_INFORM_EMAIL, durable=True)
# 声明一个交换机
# exchange 名称, exchange_type="fanout"
channel.exchange_declare(EXCHANGE_FANOUT_INFORM, "fanout", durable=True)
# 交换机队列绑定
# queue 队列, exchange 交换机, routing_key  路由key 交换机根据路由key的值把消息发到指定队列 发布订阅为''
channel.queue_bind(QUEUE_INFORM_EMAIL, EXCHANGE_FANOUT_INFORM, "")

channel.basic_consume(queue=QUEUE_INFORM_EMAIL, auto_ack=True, on_message_callback=message_callback, consumer_tag="c1")
channel.start_consuming()


# 消费者sms
from mq.client import connection

QUEUE_INFORM_SMS = "queue_inform_sms"
EXCHANGE_FANOUT_INFORM = "exchange_fanout_inform"

def message_callback(ch, method, properties, body):
    print("body is {}".format(body))


# 打开一个通道
channel = connection.channel()

# 声明两个队列
channel.queue_declare(queue=QUEUE_INFORM_SMS, durable=True)

# 声明一个交换机
# exchange 名称, exchange_type="fanout"
channel.exchange_declare(EXCHANGE_FANOUT_INFORM, "fanout", durable=True)

# 交换机队列绑定
# queue 队列, exchange 交换机, routing_key  路由key 交换机根据路由key的值把消息发到指定队列 发布订阅为''
channel.queue_bind(QUEUE_INFORM_SMS, EXCHANGE_FANOUT_INFORM, "")

channel.basic_consume(queue=QUEUE_INFORM_SMS, auto_ack=True, on_message_callback=message_callback, consumer_tag="c1")
channel.start_consuming()
```

查看交换机绑定的队列

![交换机](https://sasaba-1256963938.cos.ap-shanghai.myqcloud.com/uPic/20200903174747.png)

##### 路由模式

> 跟发布订阅的区别主要是使用direct，并且需要指定路由。

```python
# pub
from mq.client import connection

QUEUE_INFORM_EMAIL = "queue_inform_email"
QUEUE_INFORM_SMS = "queue_inform_sms"
EXCHANGE_ROUTING_INFORM = "exchange_routing_inform"
ROUTING_EMAIL = "routing_email"
ROUTING_SMS = "routing_sms"

# 打开一个通道
channel = connection.channel()

# 声明两个队列
channel.queue_declare(queue=QUEUE_INFORM_SMS, durable=True)
channel.queue_declare(queue=QUEUE_INFORM_EMAIL, durable=True)

channel.exchange_declare(EXCHANGE_ROUTING_INFORM, "direct", durable=True)

# 交换机队列绑定
# queue 队列, exchange 交换机, routing_key  路由key 交换机根据路由key的值把消息发到指定队列 发布订阅为''
channel.queue_bind(QUEUE_INFORM_SMS, EXCHANGE_ROUTING_INFORM, ROUTING_SMS)
channel.queue_bind(QUEUE_INFORM_EMAIL, EXCHANGE_ROUTING_INFORM, ROUTING_EMAIL)


channel.basic_publish(exchange=EXCHANGE_ROUTING_INFORM, routing_key=ROUTING_SMS,
                      body=b'Test 22223')
connection.close()
```

```python
# sub email
from mq.client import connection


def message_callback(ch, method, properties, body):
    print("body is {}".format(body))


QUEUE_INFORM_EMAIL = "queue_inform_email"
EXCHANGE_ROUTING_INFORM = "exchange_routing_inform"
ROUTING_EMAIL = "routing_email"

# 打开一个通道
channel = connection.channel()

# 声明两个队列
channel.queue_declare(queue=QUEUE_INFORM_EMAIL, durable=True)

channel.exchange_declare(EXCHANGE_ROUTING_INFORM, "direct", durable=True)

# 交换机队列绑定
# queue 队列, exchange 交换机, routing_key  路由key 交换机根据路由key的值把消息发到指定队列 发布订阅为''
channel.queue_bind(QUEUE_INFORM_EMAIL, EXCHANGE_ROUTING_INFORM, ROUTING_EMAIL)

channel.basic_consume(queue=QUEUE_INFORM_EMAIL, auto_ack=True, on_message_callback=message_callback, consumer_tag="email")
channel.start_consuming()

# sub sms
from mq.client import connection


def message_callback(ch, method, properties, body):
    print("body is {}".format(body))


QUEUE_INFORM_SMS = "queue_inform_sms"
EXCHANGE_ROUTING_INFORM = "exchange_routing_inform"
ROUTING_SMS = "routing_sms"

# 打开一个通道
channel = connection.channel()

# 声明两个队列
channel.queue_declare(queue=QUEUE_INFORM_SMS, durable=True)

channel.exchange_declare(EXCHANGE_ROUTING_INFORM, "direct", durable=True)

# 交换机队列绑定
# queue 队列, exchange 交换机, routing_key  路由key 交换机根据路由key的值把消息发到指定队列 发布订阅为''
channel.queue_bind(QUEUE_INFORM_SMS, EXCHANGE_ROUTING_INFORM, ROUTING_SMS)

channel.basic_consume(queue=QUEUE_INFORM_SMS, auto_ack=True, on_message_callback=message_callback, consumer_tag="sms")
channel.start_consuming()
```

查看绑定的路由

![](https://sasaba-1256963938.cos.ap-shanghai.myqcloud.com/uPic/20200904103457.png)

##### 通配符 Topic

> 使用topic，并且需要指定通配符路由。# 表示匹配一个或多个词，*只匹配一个词。
>
> 跟mqtt类似。

```python
# pub
from mq.client import connection

QUEUE_INFORM_EMAIL = "queue_inform_email"
QUEUE_INFORM_SMS = "queue_inform_sms"
EXCHANGE_ROUTING_INFORM = "exchange_topic_inform"

ROUTING_EMAIL = "inform.#.email.#"
ROUTING_SMS = "inform.#.sms.#"

# 打开一个通道
channel = connection.channel()

# 声明两个队列
channel.queue_declare(queue=QUEUE_INFORM_SMS, durable=True)
channel.queue_declare(queue=QUEUE_INFORM_EMAIL, durable=True)

channel.exchange_declare(EXCHANGE_ROUTING_INFORM, "topic", durable=True)

# 交换机队列绑定
# queue 队列, exchange 交换机, routing_key  路由key 交换机根据路由key的值把消息发到指定队列 发布订阅为''
channel.queue_bind(QUEUE_INFORM_SMS, EXCHANGE_ROUTING_INFORM, ROUTING_SMS)
channel.queue_bind(QUEUE_INFORM_EMAIL, EXCHANGE_ROUTING_INFORM, ROUTING_EMAIL)


channel.basic_publish(exchange=EXCHANGE_ROUTING_INFORM, routing_key="inform.sms.email",
                      body=b'Test 22223')
connection.close()
```

```python
# sub mail
from mq.client import connection


def message_callback(ch, method, properties, body):
    print("body is {}".format(body))


QUEUE_INFORM_EMAIL = "queue_inform_email"
EXCHANGE_ROUTING_INFORM = "exchange_topic_inform"
ROUTING_EMAIL = "inform.#.email.#"

# 打开一个通道
channel = connection.channel()

# 声明两个队列
channel.queue_declare(queue=QUEUE_INFORM_EMAIL, durable=True)

channel.exchange_declare(EXCHANGE_ROUTING_INFORM, "topic", durable=True)

# 交换机队列绑定
# queue 队列, exchange 交换机, routing_key  路由key 交换机根据路由key的值把消息发到指定队列 发布订阅为''
channel.queue_bind(QUEUE_INFORM_EMAIL, EXCHANGE_ROUTING_INFORM, ROUTING_EMAIL)

channel.basic_consume(queue=QUEUE_INFORM_EMAIL, auto_ack=True, on_message_callback=message_callback, consumer_tag="email")
channel.start_consuming()


# sub sms
from mq.client import connection


def message_callback(ch, method, properties, body):
    print("body is {}".format(body))


QUEUE_INFORM_SMS = "queue_inform_sms"
EXCHANGE_ROUTING_INFORM = "exchange_topic_inform"
ROUTING_SMS = "inform.#.sms.#"

# 打开一个通道
channel = connection.channel()

# 声明两个队列
channel.queue_declare(queue=QUEUE_INFORM_SMS, durable=True)

channel.exchange_declare(EXCHANGE_ROUTING_INFORM, "topic", durable=True)

# 交换机队列绑定
# queue 队列, exchange 交换机, routing_key  路由key 交换机根据路由key的值把消息发到指定队列 发布订阅为''
channel.queue_bind(QUEUE_INFORM_SMS, EXCHANGE_ROUTING_INFORM, ROUTING_SMS)

channel.basic_consume(queue=QUEUE_INFORM_SMS, auto_ack=True, on_message_callback=message_callback, consumer_tag="sms")
channel.start_consuming()
```



















