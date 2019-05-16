# start-elastic

A simple tutorial to begin with Elasticsearch, Logstash, Kibana combined with Confluent Kafka.

## Prerequisites

- Java 8
- Confluent Kafka
- HTTPIE

## Installations
In this tutorial, we will install services with Debian Package. If you are under Linux or Debian, follow the following steps to install Elasticsearch, Kibana and Logstash.

For other operating systems or installation methods, please refer to this [page](https://www.elastic.co/guide/en/elasticsearch/reference/current/install-elasticsearch.html) for Elasticsearch installation; this [page](https://www.elastic.co/guide/en/kibana/current/install.html) for installing Kibana and this [page](https://www.elastic.co/guide/en/logstash/current/installing-logstash.html) for Logstash.

Please adapt the command lines used in this tutorial if you have different installations.


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

How to start and stop Kibana and Logstash services depends on whether your system uses systemd or SysV init.
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

|Service|Adresses|
|-----------------|-----------------|
|Elasticsearch|localhost:9200|
|Kibana|localhost:5601|
|Logstash|localhost:9600|

Try to hit those adresses to verify if you have a functional environment.
```bash
# Ping Elasticsearch
$ http :9200/

# Ping Kibana
$ http :5601/

# Ping Logstash
$ http :9600/
```

Basically, you must obtain results like:

![Elasticsearch](img/elasticsearch.PNG?=true)
![Kibana](img/kibana.PNG?raw=true)
![Logstash](img/logstash.PNG?raw=true)

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