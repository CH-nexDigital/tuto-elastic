# start-elastic

This is a simple tutorial to begin with Elasticsearch, Logstash and Kibana combined with Confluent Kafka.

## Prerequisites

- Java 8
- Confluent Kafka
- HTTPIE

## Installations
In this tutorial, we will install services with the Debian package manager. If you are using Debian or Ubuntu, follow the following steps to install Elasticsearch, Kibana and Logstash.

For other operating systems or installation methods, please refer to this [page](https://www.elastic.co/guide/en/elasticsearch/reference/current/install-elasticsearch.html) for Elasticsearch installation; this [page](https://www.elastic.co/guide/en/kibana/current/install.html) for installing Kibana and this [page](https://www.elastic.co/guide/en/logstash/current/installing-logstash.html) for Logstash.


```bash
$ wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -

$ sudo apt-get install apt-transport-https

$ echo "deb https://artifacts.elastic.co/packages/7.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-7.x.list

$ sudo apt-get update

# Install Elasticsearch
$ sudo apt-get install elasticsearch

# Install Kibana
$ sudo apt-get install kibana

#Install Logstash
$ sudo apt-get install logstash
```



## Start services

To start and stop Kibana and Logstash services depends on whether your system uses systemd or SysV init.
Check which one is used by your system with this following command:
```bash
$ ps -p 1
```

Once you know which init process is used, run the following commands:
```bash
# If you use systemd
sudo systemctl start elasticsearch.service
sudo systemctl start kibana.service
sudo systemctl start logstash.service

# If you use SysV
sudo -i service elasticsearch start
sudo -i serivce kibana start
sudo -i service logstash start
```

## Use the services

Normally, if you installed the service with above commands you will get a configuration like:

| Service       | Adresses       |
| ------------- | -------------- |
| Elasticsearch | localhost:9200 |
| Kibana        | localhost:5601 |
| Logstash      | localhost:9600 |

Try to hit those adresses to verify if you have a functional environment.
```bash
# Ping Elasticsearch
$ http :9200/

# Ping Kibana
$ http :5601/

# Ping Logstash
$ http :9600/
```

You should obtain a results looking like:

```json
clairehuang@clairehuang-VirtualBox:~$ http :9200
HTTP/1.1 200 OK
content-encoding: gzip
content-length: 312
content-type: application/json; charset=UTF-8

{
    "cluster_name": "elasticsearch",
    "cluster_uuid": "ZreObPJdTj2Z1CuVqVC5mw",
    "name": "clairehuang-VirtualBox",
    "tagline": "You Know, for Search",
    "version": {
        "build_date": "2019-04-29T12:56:03.145736Z",
        "build_flavor": "default",
        "build_hash": "e4efcb5",
        "build_snapshot": false,
        "build_type": "deb",
        "lucene_version": "8.0.0",
        "minimum_index_compatibility_version": "6.0.0-beta1",
        "minimum_wire_compatibility_version": "6.7.0",
        "number": "7.0.1"
    }
}
```

```json
clairehuang@clairehuang-VirtualBox:~$ http :5601
HTTP/1.1 302 Found
Date: Thu, 16 May 2019 12:36:04 GMT
cache-control: no-cache
connection: close
content-length: 0
content-type: text/html; charset=utf-8
kbn-name: kibana
kbn-xpack-sig: 8ffb32c48c2c02774f12f7f8f79dbf93
location: /app/kibana
```

```json
clairehuang@clairehuang-VirtualBox:~$ http :9600
HTTP/1.1 200 OK
Content-Length: 273
Content-Type: application/json
X-Content-Type-Options: nosniff

{
    "build_date": "2019-04-29T13:58:53Z",
    "build_sha": "54853601666de7b5da02f555d7b59087b3afc1aa",
    "build_snapshot": false,
    "host": "clairehuang-VirtualBox",
    "http_address": "127.0.0.1:9600",
    "id": "a1094d8d-9001-4279-a9c6-0d35aef6fadf",
    "name": "clairehuang-VirtualBox",
    "version": "7.0.1"
}
```

## Ingest data into Elasticsearch with Logstash
Let's connect Confluent Kafka and Elasticsearch with Logstash. Note that Confluent has its own Kafka Connect Elasticsearch Connector but it is only working with Elasticsearch 2.x, 5.x, or 6.x. Normally, if you followed the installation steps of this tutorial, you will get a version like 7.x, so you have to use Logstash instead of Kafka Connect Elasticsearch connector.

One scenario of this part is that you have a data source that streams to Kafka and you want to index those data in Elasticsearch to make them searchable.
In this part, we will simplily consider a console producer that write messages to Kafka. We want those messages to be searchable.

### Step 1: Configure Logstash
First, we have to configure Logstash. Create a configuration file at **/etc/logstash/conf.d/logstash-kafka-test.conf**

```apache
input {
    kafka {
        id => "my_plugin_id"
        topics => ["test"]
        bootstrap_servers => "localhost:9092"
        }
}
output {
    elasticsearch {
        hosts => ["localhost:9200"]
        ilm_rollover_alias => "kafka"
        ilm_pattern => "1"
        }
}
```
> Please modify the **bootstrap_servers** if you have a different Kafka configuration.

Now, let's restart Logstash and this new created plugin will be running.
```bash
#If you use systemd
$ sudo systemctl restart logstash.service

#If you use SysV
$ sudo -i service logstash restart
```
### Step 2: Produce messages in Kafka
Produce some messages with the console producer.
```bash
$ kafka-console-producer --broker-list locahost:9092 --topic test
> hello world
> hipopotamus
> Los Angeles
> Pokémon Detective Pikachu
```

### Step 3: Search messages in Elasticsearch
Your logstash plugin is already running. When you finished writing to Kafka, your data will be already searchable in Elasticsearch.
Try the following command to check that you have successfully ingested Kafka data to Elasticsearch with Logstash:
```bash
$ http :9200/_cat/indices
```
![Indices](img/indices.PNG?raw=true)
You may notice that your data have been successfully indexed to **kafka-1**. Now, try to search one word in this index.
```bash
$ http :9200/kafka-1/_search?q=pikachu
```
Your result should look like:
```http
clairehuang@clairehuang-VirtualBox:~$ http :9200/_cat/indices
HTTP/1.1 200 OK
content-encoding: gzip
content-length: 231
content-type: text/plain; charset=UTF-8

green  open .kibana_1            KmTc0cQNRkGdv6p_B9KjZA 1 0 2 0 10.6kb 10.6kb
yellow open logstash-test-1      cMh-O41cQi2RKnUzkJq1ZA 1 1 0 0   283b   283b
green  open .kibana_task_manager WOgGnUsCStmCiGgwCZlPpg 1 0 2 0 29.5kb 29.5kb
yellow open kafka-1              Xn0Sz-2oQvuSiDmyurAuOg 1 1 4 0 14.3kb 14.3kb

```

If you want to pass several key words at time, just put a coma between your searching key words like:
```bash
$ http :9200/kafka-1/_search?q=pikachu,hello
```
You will now get two responses: one that matches "hello" and the other for "pikachu".

```json
clairehuang@clairehuang-VirtualBox:~$ http :9200/kafka-1/_search?q=pikachu,hello
HTTP/1.1 200 OK
content-encoding: gzip
content-length: 318
content-type: application/json; charset=UTF-8

{
    "_shards": {
        "failed": 0,
        "skipped": 0,
        "successful": 1,
        "total": 1
    },
    "hits": {
        "hits": [
            {
                "_id": "JTrqv2oBummLXKIp2SAp",
                "_index": "kafka-1",
                "_score": 1.5135659,
                "_source": {
                    "@timestamp": "2019-05-16T09:13:20.244Z",
                    "@version": "1",
                    "message": "hello world"
                },
                "_type": "_doc"
            },
            {
                "_id": "KDrsv2oBummLXKIpayB6",
                "_index": "kafka-1",
                "_score": 1.5135659,
                "_source": {
                    "@timestamp": "2019-05-16T09:15:03.300Z",
                    "@version": "1",
                    "message": "Pokémon Detective Pikachu"
                },
                "_type": "_doc"
            }
        ],
        "max_score": 1.5135659,
        "total": {
            "relation": "eq",
            "value": 2
        }
    },
    "timed_out": false,
    "took": 1
}


```

## Stop services
```bash
# If you use systemd
sudo systemctl stop elasticsearch.service
sudo systemctl stop kibana.service
sudo systemctl stop logstash.service

# If you use SysV
sudo -i service elasticsearch stop
sudo -i serivce kibana stop
sudo -i service logstash stop
```