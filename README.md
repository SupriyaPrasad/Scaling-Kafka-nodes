## SCALING UP AND DOWN OF KAFKA NODES USING ANSIBLE:

This playbook is for adding / removing kafka broker nodes from an existing zookeeper-kafka cluster. The initial kafka multinode cluster can be built using https://github.com/sanjeevmaheve/ansible-kafka-cluster. 

## How to use: 
In this version, its assumed that you have provisioned VMs with Ubuntu OS either on cloud or you local infrastructure before you choose to run ansible playbook.
The kafka version we are currently using is **0.8.2.1**

## Pre- requisite:
Clone the scale up-scale down playbook using https://github.com/SupriyaPrasad/Scaling-Kafka-nodes.git

## Running the playbook:

Ensure the `remove-kafka` role under the roles directory.

Ensure the `site.yml` file to look like :
```
---
- hosts: zookeepers
  roles:
    - {
        role: ansible-java,
        when: accept_oracle_licence
      }
    - {
        role: ansible-zookeeper,
        zookeeper_version: 3.4.6
      }
- hosts: kafka-nodes
  roles:
    - {
        role: ansible-java,
        when: accept_oracle_licence
      }
    - {
        role: ansible-kafka
      }
- hosts: remove-nodes
  roles:
    - {
        role: remove-kafka
      }

```
Ensure the inventory file looks like :

```
[zookeepers]
192.168.0.170 zookeeper_id=1 ansible_ssh_user=<username>
192.168.0.171 zookeeper_id=2 ansible_ssh_user=<username>

[kafka-nodes]
192.168.0.168 kafka_broker_id=1 kafka_hostname=kafka-1 ansible_ssh_user=<username>
192.168.0.175 kafka_broker_id=2 kafka_hostname=kafka-2 ansible_ssh_user=<username>

[remove-nodes]
192.168.0.168 kafka_broker_id=1 kafka_hostname=kafka-1 ansible_ssh_user=<username>
```
Replace username above with the user name on the destination machines where the modules would be installed and configured by ansible. 

Also edit the ` '/etc/hosts' ` file to make sure it contains the respective information of the nodes to be added.
```
127.0.0.1       localhost
127.0.0.1       sfuser-virtual-machine

# The following lines are desirable for IPv6 capable hosts
192.168.0.168 sfuser-virtual-machine kafka-1
192.168.0.175 sfuser-virtual-machine kafka-2
192.168.0.154 sfuser-virtual-machine kafka-3
```

## Scaling up of broker nodes.
Make sure you add the respective broker information which should be added under the [kafka-nodes] section. 
```
[kafka-nodes]
192.168.0.154 kafka_broker_id=3 kafka_hostname=kafka-3 ansible_ssh_user=<username>
192.168.0.168 kafka_broker_id=1 kafka_hostname=kafka-1 ansible_ssh_user=<username>
192.168.0.175 kafka_broker_id=2 kafka_hostname=kafka-2 ansible_ssh_user=<username>
```
**Since ansible is idempotent, it will install kafka and bring up the service only on the nodes which does not have kafka on it.**

To add broker nodes, we run the following command:
``` 
ansible-playbook -i hosts site.yml --tags "java,kafka" --sudo -K 
```

Once you run the above command, the ansible playbook starts running on the nodes which dont have kafka installed on them and bring them up. 

we can verify that the service is up and running using the command ` netstat -tulnp ` which will give the following output
```
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
.
.
tcp6       0      0 192.168.0.175:9092      :::*                    LISTEN      -               
.
.
```
the port 9092 shows that kafka is up and running. 

## Scaling Down Kafka nodes: 
Enter the IP and the other required information of the node/nodes which have to be removed from the cluster. 
```
[remove-nodes]
192.168.0.168 kafka_broker_id=1 kafka_hostname=kafka-1 ansible_ssh_user=<username>
```
Run the following command to remove the node from the cluster:
```
ansible-playbook -i hosts site.yml --tags "remove" --sudo -K
```
After running this, you will see that the 9092 port does not show up when we run `netstat -tulnp` .

## Tests to verify no data loss between producer and consumer during scale up and scale down

Consider that we have created a topic by name "new" with a replication-factor as 2 and partition as 2. the description of the topic look something like this:
```
sfuser@sfuser-virtual-machine:/usr/local/kafka$ bin/kafka-topics.sh --describe --zookeeper 192.168.0.170:2181 --topic new
Topic:new	PartitionCount:2	ReplicationFactor:2	Configs:
	Topic: new	Partition: 0	Leader: 6	Replicas: 1,6	Isr: 6,1
	Topic: new	Partition: 1	Leader: 6	Replicas: 6,1	Isr: 6,1
```
As we see that the leader for p0 and p1 is 6 and the ISR i.e the in-sync replica(follower leader) is 6,1 which means that when the leader goes down, kafka picks up the next in-sync replica which is 1. 

**Producer**: I have created a python API as shown below which produces sequence numbers for that particular topic. I have included all the broker nodes so that kafka will pick up another node when we bring down a node.
```
# -*- coding: utf-8 -*-
from kafka import SimpleProducer, KafkaClient
from datetime import datetime
import pytz
from pytz import timezone
#import datetime
import time
import socket

# To send messages synchronously
kafka = KafkaClient('192.168.0.168:9092, 192.168.0.169:9092,192.168.0.172:9092, 192.168.0.173:9092, 192.168.0.175:9092, 192.168.0.154:9092')
BATCH_SEND_DEFAULT_INTERVAL = 5
BATCH_SEND_MSG_COUNT = 10

producer = SimpleProducer(kafka, async=True,
                          batch_send_every_n=BATCH_SEND_DEFAULT_INTERVAL,
                          batch_send_every_t=BATCH_SEND_MSG_COUNT)
fmt = "%y-%m%d %H:%M%S"
producer = SimpleProducer(kafka, async=True)
for i in range(0,1000):
        ts = time.time()
# Note that the application is responsible for encoding messages to type bytes
        now_utc=datetime.now(timezone('Asia/Kolkata'))
        producer.send_messages(b'new', bytes(now_utc) + "-->" + bytes(i))
        print now_utc.strftime(fmt), "-->", i
        time.sleep(1)

```
Save this python script in the /usr/local/kafka/bin directory. Run the script using python <<filename>>.py
**Consumer**: using the default shell script to start the consumer for that particular topic.
```
bin/kafka-console-consumer.sh --zookeeper 192.168.0.170:2181 --topic new
```

**Findings:**

1. Start the producer and consumer on different nodes and bring down the leader of a particular partition. 
      In this case, the producer will run seemlessly. However the consumer will throw some warnings until kafka picks up           another leader from the ISR. once it picks up the leader, the consumer will start picking up messages from where it          stopped without experiencing any loss of data.
      Note: the consumer does not receive messages in the order. Therefore to make sure no message has been lost, i took the       numbers and sorted it. On doing so i found that no message was lost.
2. Start the producer and consumer on differnt nodes and bring down a non-leader of a particular partition.
      In this case, the producer and the consumer will work seemlessly.
