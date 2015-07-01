# Kafka Multi Node Cluster using Ansible.
This work is inspired from https://github.com/lloydmeta/ansible-kafka-cluster. You can configure provisioned ubuntu VMs with Zookeeper(s) and Kafka node(s) in few seconds and get the Kafka multinode cluster up and running for you.

## How to use
In this version, its assumed that you have provisioned VMs either on cloud or you local infrastructure before you choose to run ansible playbook.

### Pre-requisite
1. Install Ansible on the development machine/VM. There are number of ways, I followed using Python package. ```$ sudo pip install ansible```
2. ```$ git clone https://github.com/sanjeevmaheve/ansible-kafka-cluster.git```

### Running the playbook
Ensure you have the following information correctly placed with in ansible configuration and host files.

```
skm@ubuntu-node14:~/project/ansible/ansible-kafka-cluster$ cat hosts 
[zookeepers]
192.168.0.168 zookeeper_id=1 ansible_ssh_user=<username>
192.168.0.169 zookeeper_id=2 ansible_ssh_user=<username>

[kafka-nodes]
192.168.0.172 kafka_broker_id=1 kafka_hostname=kafka-1 ansible_ssh_user=<username>
192.168.0.173 kafka_broker_id=2 kafka_hostname=kafka-2 ansible_ssh_user=<username>
```

Replace username above with the user name on the destination machines where the modules would be installed and configured by ansible. Next, change the main YAML file (incase needed); it looks like this:

```
skm@ubuntu-node14:~/project/ansible/ansible-kafka-cluster$ cat site.yml 
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

```
Please note that the hosts defined in site.yml comes from hosts file above. Next, lets run the main playbook (assuming you made the above mentioned changes.)
```
skm@ubuntu-node14:~/project/ansible/ansible-kafka-cluster$ ansible-playbook -i hosts site.yml --sudo -K
```
OR run the playbook based on the defined tags.

#### Configure "Zookeeper" cluster (first-step)
```
skm@ubuntu-node14:~/project/ansible/ansible-kafka-cluster$ ansible-playbook -i hosts site.yml --tags "zookeeper" --sudo -K
SUDO password: 

PLAY [zookeepers] ************************************************************* 

GATHERING FACTS *************************************************************** 
ok: [192.168.0.169]
ok: [192.168.0.168]

TASK: [ansible-zookeeper | create group] ************************************** 
ok: [192.168.0.169]
ok: [192.168.0.168]

TASK: [ansible-zookeeper | create user] *************************************** 
ok: [192.168.0.169]
ok: [192.168.0.168]

TASK: [ansible-zookeeper | Setting internal variable] ************************* 
ok: [192.168.0.168]
ok: [192.168.0.169]

TASK: [ansible-zookeeper | Setting internal variable] ************************* 
ok: [192.168.0.168]
ok: [192.168.0.169]

TASK: [ansible-zookeeper | Download Zookeeper {{ zookeeper.version }}] ******** 
ok: [192.168.0.169]
ok: [192.168.0.168]

TASK: [ansible-zookeeper | Unpack the tar] ************************************ 
ok: [192.168.0.169]
changed: [192.168.0.168]

TASK: [ansible-zookeeper | Overwrite default config file] ********************* 
ok: [192.168.0.169]
changed: [192.168.0.168]

TASK: [ansible-zookeeper | Change mode of configuration file] ***************** 
changed: [192.168.0.169]
changed: [192.168.0.168]

TASK: [ansible-zookeeper | Copy zookeeper to real destination] **************** 
changed: [192.168.0.169]
changed: [192.168.0.168]

TASK: [ansible-zookeeper | Check for zookeeper data dir and create if not present] *** 
ok: [192.168.0.169]
ok: [192.168.0.168]

TASK: [ansible-zookeeper | shell mkdir -p {{ zookeeper.data_dir }}] *********** 
changed: [192.168.0.169]
changed: [192.168.0.168]

TASK: [ansible-zookeeper | Overwrite myid file.] ****************************** 
changed: [192.168.0.169]
changed: [192.168.0.168]

TASK: [ansible-zookeeper | Link {{ zookeeper.install_dir }}/zookeeper to this version] *** 
changed: [192.168.0.169]
changed: [192.168.0.168]

TASK: [ansible-zookeeper | Install the zookeeper service handler] ************* 
changed: [192.168.0.168]
changed: [192.168.0.169]

TASK: [ansible-zookeeper | Ensure the state of the zookeeper service is started] *** 
changed: [192.168.0.169]
changed: [192.168.0.168]

NOTIFIED: [ansible-zookeeper | start zookeeper] ******************************* 
changed: [192.168.0.169]
changed: [192.168.0.168]

PLAY RECAP ******************************************************************** 
192.168.0.168              : ok=17   changed=10   unreachable=0    failed=0   
192.168.0.169              : ok=17   changed=8    unreachable=0    failed=0   

skm@ubuntu-node14:~/project/ansible/ansible-kafka-cluster$
```

#### Configure "Kafka" node(s)
```
skm@ubuntu-node:~/project/ansible/ansible-kafka-cluster$ ansible-playbook -i hosts site.yml --tags "kafka" --sudo -K
SUDO password: 

PLAY [kafka-nodes] ************************************************************ 

GATHERING FACTS *************************************************************** 
ok: [192.168.0.173]
ok: [192.168.0.172]

TASK: [ansible-kafka | create group] ****************************************** 
ok: [192.168.0.172]
ok: [192.168.0.173]

TASK: [ansible-kafka | create user] ******************************************* 
ok: [192.168.0.173]
ok: [192.168.0.172]

TASK: [ansible-kafka | Setting internal variable] ***************************** 
ok: [192.168.0.172]
ok: [192.168.0.173]

TASK: [ansible-kafka | Setting internal variable] ***************************** 
ok: [192.168.0.172]
ok: [192.168.0.173]

TASK: [ansible-kafka | check if tar has been downloaded] ********************** 
ok: [192.168.0.173]
ok: [192.168.0.172]

TASK: [ansible-kafka | Ensure Kafka tar is downloaded] ************************ 
skipping: [192.168.0.172]
skipping: [192.168.0.173]

TASK: [ansible-kafka | Ensure tar is extracted] ******************************* 
changed: [192.168.0.173]
changed: [192.168.0.172]

TASK: [ansible-kafka | Ensures data dir "{{ kafka.data_dir }}" exists] ******** 
changed: [192.168.0.172]
changed: [192.168.0.173]

TASK: [ansible-kafka | Copy real config] ************************************** 
changed: [192.168.0.173]
changed: [192.168.0.172]

TASK: [ansible-kafka | Symlink {{ kafka.install_dir }}/kafka to this version] *** 
changed: [192.168.0.173]
changed: [192.168.0.172]

TASK: [ansible-kafka | Lets wait to see if we have Port {{zk_client_port}} is avialable.] *** 
ok: [192.168.0.173]
ok: [192.168.0.172]

TASK: [ansible-kafka | Install the Kafka service handler] ********************* 
changed: [192.168.0.173]
changed: [192.168.0.172]

NOTIFIED: [ansible-kafka | start kafka] *************************************** 
changed: [192.168.0.173]
changed: [192.168.0.172]

PLAY RECAP ******************************************************************** 
192.168.0.172              : ok=13   changed=6    unreachable=0    failed=0   
192.168.0.173              : ok=13   changed=6    unreachable=0    failed=0   

skm@ubuntu-node14:~/project/ansible/ansible-kafka-cluster$
```
## Testing multi-node Kafka setup.

On set of VM’s that would be used as kafka-cluster, ```'/etc/hosts'``` file should list hostnames of all kafka-nodes; something like this
```
127.0.0.1       localhost
127.0.1.1       ubuntu-node17
# List all the kafka nodes here so that each node in cluster know about every other nodes.
192.168.0.172   ubuntu-node17 kafka-1
192.168.0.173   ubuntu-node18 kafka-2
```
Similarly do for ubuntu-node18. As per example above, the ```/etc/hostname``` file on 192.168.0.172 and 192.168.0.173 nodes need to contain:
```
ubuntu-node17
ubuntu-node18
```
entry respectively.

Under kafka role in ansible playbook, I enabled the ```host.name```  property i.e. hostname the broker will bind to. If not set, the server will bind to all interfaces
```
host.name={{ kafka_hostname }}
```

Then create the topic using the following command:
```
skm@ubuntu-node17:/usr/local/kafka$ bin/kafka-topics.sh --create --zookeeper 192.168.0.168:2181 --replication-factor 2 --partitions 2 --topic test-2
Created topic “test-2”.
```

Ensure it is created as expected:
```
skm@ubuntu-node17:/usr/local/kafka$ bin/kafka-topics.sh --describe --zookeeper 192.168.0.168:2181 --topic test-2
Topic:test-2      PartitionCount:2  ReplicationFactor:2     Configs:
      Topic: test-2     Partition: 0      Leader: 2   Replicas: 2,1     Isr: 2,1
      Topic: test-2     Partition: 1      Leader: 1   Replicas: 1,2     Isr: 1,2
```

Then test this using the following commad on ubuntu-node17 and ubuntu-node18 respectively (I.e. Producer on one node and Consumer on another)
```
skm@ubuntu-node17:/usr/local/kafka$ bin/kafka-console-producer.sh --broker-list kafka-1:9092,kafka-2:9092 --sync --topic test-2
skm@ubuntu-node18:/usr/local/kafka$ bin/kafka-console-consumer.sh --zookeeper 192.168.0.168:2181 --from-beginning --topic test-2
```

Now kill ```ubuntu-node18``` and you can see the leaders for the defined topic has been changed accordingly.
```
skm@ubuntu-node17:/usr/local/kafka$ bin/kafka-topics.sh --describe --zookeeper 192.168.0.168:2181 --topic test-2
Topic:test-2      PartitionCount:2  ReplicationFactor:2     Configs:
      Topic: test-2     Partition: 0      Leader: 1   Replicas: 2,1     Isr: 1
      Topic: test-2     Partition: 1      Leader: 1   Replicas: 1,2     Isr: 1
```

Hope it helps.
