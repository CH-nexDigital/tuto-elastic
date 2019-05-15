# start-elastic

A simple tutorial for Elasticsearch. This tutorial will show a simple use case of Elasticsearch, combined with Kafka.

## Prerequisites

- Java 8
- Kafka

## Installations
In this tutorial, we will install services with Debian Package. If you are under Linux or Debian, follow the following steps to install Elasticsearch, Kibana and Logstash. 

For other operating systems or installation methods, please refer to this [page](https://www.elastic.co/guide/en/elasticsearch/reference/current/install-elasticsearch.html) for Elasticsearch installation; this [page](https://www.elastic.co/guide/en/kibana/current/install.html) for installing Kibana and this [page]() for Logstash.

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