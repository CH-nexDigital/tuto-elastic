# start-elastic

This is a simple tutorial to begin with Elasticsearch, Logstash and Kibana combined with Confluent Kafka.

## Table of Contents
- [start-elastic](#start-elastic)
  - [Table of Contents](#table-of-contents)
  - [Prerequisites](#prerequisites)
  - [Installations](#installations)
  - [Start services](#start-services)
  - [Use the services](#use-the-services)
  - [Example 1 : Ingest data into Elasticsearch with Logstash](#example-1--ingest-data-into-elasticsearch-with-logstash)
    - [Step 1: Configure Logstash](#step-1-configure-logstash)
    - [Step 2: Produce messages in Kafka](#step-2-produce-messages-in-kafka)
    - [Step 3: Search messages in Elasticsearch](#step-3-search-messages-in-elasticsearch)
      - [Simple Search](#simple-search)
      - [Multiple keywords search](#multiple-keywords-search)
  - [Example 2: Ingest data to Kafka](#example-2-ingest-data-to-kafka)
      - [Handling multiple pipelines](#handling-multiple-pipelines)
      - [Multi-fields Match and Selection](#multi-fields-match-and-selection)
    - [Retrieve Data in Kafka](#retrieve-data-in-kafka)
  - [Visualization on Kibana](#visualization-on-kibana)
    - [Connect Elasticsearch index to Kibana](#connect-elasticsearch-index-to-kibana)
    - [Create Visualization](#create-visualization)
  - [Stop services](#stop-services)

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
$ http :9200

# Ping Kibana
$ http :5601

# Ping Logstash
$ http :9600
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

## Example 1 : Ingest data into Elasticsearch with Logstash
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
$ kafka-console-producer --broker-list localhost:9092 --topic test
> hello world
> hipopotamus
> Los Angeles
> Pokémon Detective Pikachu
> hello Pikachu
> Pikachu hello
```

### Step 3: Search messages in Elasticsearch
Your logstash plugin is already running. When you finished writing to Kafka, your data will be already searchable in Elasticsearch.
Try the following command to check that you have successfully ingested Kafka data to Elasticsearch with Logstash:
```bash
$ http :9200/_cat/indices
```
You may notice that your data have been successfully indexed to **kafka-1**.

```http
clairehuang@clairehuang-VirtualBox:~$ http :9200/_cat/indices
HTTP/1.1 200 OK
content-encoding: gzip
content-length: 187
content-type: text/plain; charset=UTF-8

green  open .kibana_1            mtyWm9wVQ5WDKON8-WATPg 1 0 2 0 10.6kb 10.6kb
green  open .kibana_task_manager 5tIYgcmCTfmpr9OH5b38cg 1 0 2 0 45.5kb 45.5kb
yellow open kafka-1              XxFrrN9sSHKPO53EZu-X8A 1 1 6 0 17.6kb 17.6kb
```

#### Simple Search
Now, try to search one word in this index.
```bash
$ http :9200/kafka-1/_search?q=hipopotamus
```
Your result should look like:
```json
clairehuang@clairehuang-VirtualBox:~$ http :9200/kafka-1/_search?q=hipopotamus
HTTP/1.1 200 OK
content-encoding: gzip
content-length: 265
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
                "_id": "JnnnxGoBqXpp3LBZj9vH",
                "_index": "kafka-1",
                "_score": 1.9365597,
                "_source": {
                    "@timestamp": "2019-05-17T08:27:51.003Z",
                    "@version": "1",
                    "message": "hipopotamus"
                },
                "_type": "_doc"
            }
        ],
        "max_score": 1.9365597,
        "total": {
            "relation": "eq",
            "value": 1
        }
    },
    "timed_out": false,
    "took": 17
}

```

#### Multiple keywords search
You may notice that a score is attributed to each result. Let's see what happens when there are several matching results. We will pass several key words separated by coma in the request just like:
```bash
$ http :9200/kafka-1/_search?q=hello,Pikachu
```
You will now get four responses. The two that contain both "hello" and "Pikachu" have greater score than the two that only contain one key word.
The more one document contains searching key words, the greater is its score.
```json
clairehuang@clairehuang-VirtualBox:~$ http :9200/kafka-1/_search?q=hello,Pikachu
HTTP/1.1 200 OK
content-encoding: gzip
content-length: 373
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
                "_id": "KXnnxGoBqXpp3LBZ6du9",
                "_index": "kafka-1",
                "_score": 1.7427702,
                "_source": {
                    "@timestamp": "2019-05-17T08:28:14.036Z",
                    "@version": "1",
                    "message": "hello Pikachu"
                },
                "_type": "_doc"
            },
            {
                "_id": "KnnnxGoBqXpp3LBZ99uC",
                "_index": "kafka-1",
                "_score": 1.7427702,
                "_source": {
                    "@timestamp": "2019-05-17T08:28:17.562Z",
                    "@version": "1",
                    "message": "Pikachu hello"
                },
                "_type": "_doc"
            },
            {
                "_id": "JXnnxGoBqXpp3LBZb9vU",
                "_index": "kafka-1",
                "_score": 0.8713851,
                "_source": {
                    "@timestamp": "2019-05-17T08:27:42.735Z",
                    "@version": "1",
                    "message": "hello world"
                },
                "_type": "_doc"
            },
            {
                "_id": "KHnnxGoBqXpp3LBZztvJ",
                "_index": "kafka-1",
                "_score": 0.8713851,
                "_source": {
                    "@timestamp": "2019-05-17T08:28:07.129Z",
                    "@version": "1",
                    "message": "Pokémon Detective Pikachu"
                },
                "_type": "_doc"
            }
        ],
        "max_score": 1.7427702,
        "total": {
            "relation": "eq",
            "value": 4
        }
    },
    "timed_out": false,
    "took": 1
}
```
## Example 2: Ingest data to Kafka
Now, we want to make searchable CSV files and ingest data to Kafka using Logstash. 
In this example, we will consider only one CSV file. Clone this repo first, then create **sincedb** file for further use:
```bash
$ git clone https://github.com/nexDchuang/tuto-elastic
$ cd tuto-elastic/data
$ touch sincedb
```
**sincedb** file is used to track the current position in each file. It is useful if you don't want to ingest all data from beginning when Logstash restart but only those that were not ingested.

As above, create a new configuration file at **/etc/logstash/conf.d/logstash-kafka-airports.conf**:
```apache
input {
    file {
        mode => "tail"
        path => "/path/to/tuto-elastic/data/airports.csv"
        start_position => "beginning"
        sincedb_path => "/path/to/tuto-elastic/data/sincedb"
    }
}
filter {
    csv {
        separator => ","
        columns => ["Airport_ID","Name","City","Country","IATA","ICAO","Latitude","Longitude","Altitude","Timezone","DST","Tz_database_time_zone","Type","Source"]
    }
    prune {
        blacklist_names => ["message","host","path"]
    }
}
output {
    elasticsearch {
        hosts => ["localhost:9200"]
        ilm_rollover_alias => "logstash-airports"
        ilm_pattern => "1"
    }
    kafka {
        topic_id => "logstash-airports"
        bootstrap_servers => "localhost:9092"
        codec => json
    }
}
```
>Don't forget to modify **path** and **sincedb_path** to the path where you cloned the repository.

#### Handling multiple pipelines
Usually, when you install Logstash there is only one pipeline for all configuration files in **/etc/logstash/conf.d** repository. If you got multiple flows, you can handle them using one configuration file and defining conditions. If you put several configuration files without any additional configurations, your data will be mixed up and you can will find unexpected behaviors. The following shows that if you put all your flows in one pipeline without any distinction, you can get your CSV file data in the **kafka-1** index of the previous example. In addition to that, your data can be corrupted with mis-matching fields and values.

```http
clairehuang@clairehuang-VirtualBox:~/Desktop$ http :9200/_cat/indices
HTTP/1.1 200 OK
content-encoding: gzip
content-length: 151
content-type: text/plain; charset=UTF-8

green  open .kibana_1            mtyWm9wVQ5WDKON8-WATPg 1 0 2 0 10.6kb 10.6kb
green  open .kibana_task_manager 5tIYgcmCTfmpr9OH5b38cg 1 0 2 0 45.5kb 45.5kb
yellow open logstash-airports-1 80HuuCLGSnmXPwxQZg39Aw 1 1 7542 0   3.2mb   3.2mb
yellow open kafka-1             XxFrrN9sSHKPO53EZu-X8A 1 1 4669 0 324.8kb 324.8kb
```

To avoid that, what I suggest is if your flows are simple and share some common characterics, you can group them into one pipeline. Even though increasing pipeline is CPU consuming, it is important to deploy multiple pipelines if you want proper configuration. In this section, we will learn how to deploy multiple pipelines.

The pipelines configuration can be found at **/etc/logstash/pipelines.yml**, modify the file to match the following lines:
```yaml
# This file is where you define your pipelines. You can define multiple.
# For more information on multiple pipelines, see the documentation:
#   https://www.elastic.co/guide/en/logstash/current/multiple-pipelines.html

- pipeline.id: kafka
  path.config: "/etc/logstash/conf.d/logstash-kafka-test.conf"

- pipeline.id: airports
  path.config: "/etc/logstash/conf.d/logstash-kafka-airports.conf"
```

Then, restart the logstash service to take the modifications into consideration.
```bash
# If you use systemd
$ sudo systemctl restart logstash.service

#If you use SysV
$ sudo -i service logstash restart
```
You should have a result like:
```http
clairehuang@clairehuang-VirtualBox:~/Desktop$ http :9200/_cat/indices
HTTP/1.1 200 OK
content-encoding: gzip
content-length: 237
content-type: text/plain; charset=UTF-8

yellow open kafka-1              j3f7QyawRWiws3ePl36RUQ 1 1    0 0   230b   230b
green  open .kibana_1            gbnHPfVaT4Sh8hAyc2rU5Q 1 0    2 0 10.5kb 10.5kb
green  open .kibana_task_manager Y04rMxaVRjOPMpXu3w6zWw 1 0    2 4 44.9kb 44.9kb
yellow open logstash-airports-1  Fhor9__sSMGaeALtbpQeYg 1 1 7543 0  3.2mb  3.2mb
```

#### Multi-fields Match and Selection

Now, let's find all Airports in Paris. We will search on **Country** and **City** fields and show only **Country**,**City** and **Name** fields in the result. 

```bash
http GET :9200/logstash-airports/_search --json <<< '{ 
  "query": { 
    "bool": { 
      "must": [
        { "match": { "Country": "France" }}, 
        { "match": { "City": "Paris" }}  
      ]
    }
  },
  "_source":["Country","City","Name"]
}'
```
The result should be like:
```json
HTTP/1.1 200 OK
content-encoding: gzip
content-length: 305
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
                "_id": "7tdmxmoBEHaHvh_Z2xCw",
                "_index": "logstash-airports-1",
                "_score": 12.103136,
                "_source": {
                    "City": "Paris",
                    "Country": "France",
                    "Name": "Paris-Le Bourget Airport"
                },
                "_type": "_doc"
            },
            {
                "_id": "8NdmxmoBEHaHvh_Z2xCw",
                "_index": "logstash-airports-1",
                "_score": 12.103136,
                "_source": {
                    "City": "Paris",
                    "Country": "France",
                    "Name": "Charles de Gaulle International Airport"
                },
                "_type": "_doc"
            },
            {
                "_id": "9NdmxmoBEHaHvh_Z2xCw",
                "_index": "logstash-airports-1",
                "_score": 12.103136,
                "_source": {
                    "City": "Paris",
                    "Country": "France",
                    "Name": "Paris-Orly Airport"
                },
                "_type": "_doc"
            }
        ],
        "max_score": 12.103136,
        "total": {
            "relation": "eq",
            "value": 3
        }
    },
    "timed_out": false,
    "took": 1
}
```
Now, you know the basic searches. For more information about searching, please refer to the [Elasticsearch documentation page](https://www.elastic.co/guide/en/elasticsearch/reference/current/search.html).

### Retrieve Data in Kafka

Try the following command to check that the CSV file has been successfully ingested to Kafka.
```bash
$ kafka-console-consumer --bootstrap-server localhost:9092 --topic logstash-airports --from-beginning

# Use --max-messages flag if you only want to check the first 10 messages
kafka-console-consumer --bootstrap-server localhost:9092 --topic logstash-airports --from-beginning --max-messages 10

```
Messages in Kafka should look like:
```json
{"Source":"OurAirports","Name":"Kangerlussuaq Airport","City":"Sondrestrom","Altitude":"165","Longitude":"-50.7116031647","Timezone":"-3","Tz_database_time_zone":"America/Godthab","IATA":"SFJ","@version":"1","@timestamp":"2019-05-17T12:43:04.679Z","Latitude":"67.0122218992","DST":"E","ICAO":"BGSF","Type":"airport","Country":"Greenland","Airport_ID":"9"}
{"Source":"OurAirports","Name":"Keflavik International Airport","City":"Keflavik","Altitude":"171","Longitude":"-22.605600357056","Timezone":"0","Tz_database_time_zone":"Atlantic/Reykjavik","IATA":"KEF","@version":"1","@timestamp":"2019-05-17T12:43:04.681Z","Latitude":"63.985000610352","DST":"N","ICAO":"BIKF","Type":"airport","Country":"Iceland","Airport_ID":"16"}
{"Source":"OurAirports","Name":"Reykjavik Airport","City":"Reykjavik","Altitude":"48","Longitude":"-21.9405994415","Timezone":"0","Tz_database_time_zone":"Atlantic/Reykjavik","IATA":"RKV","@version":"1","@timestamp":"2019-05-17T12:43:04.682Z","Latitude":"64.1299972534","DST":"N","ICAO":"BIRK","Type":"airport","Country":"Iceland","Airport_ID":"18"}
{"Source":"OurAirports","Name":"Pond Inlet Airport","City":"Pond Inlet","Altitude":"181","Longitude":"-77.9666976929","Timezone":"-5","Tz_database_time_zone":"America/Toronto","IATA":"YIO","@version":"1","@timestamp":"2019-05-17T12:43:04.695Z","Latitude":"72.6832962036","DST":"A","ICAO":"CYIO","Type":"airport","Country":"Canada","Airport_ID":"75"}
{"Source":"OurAirports","Name":"Schefferville Airport","City":"Schefferville","Altitude":"1709","Longitude":"-66.8052978515625","Timezone":"-5","Tz_database_time_zone":"America/Toronto","IATA":"YKL","@version":"1","@timestamp":"2019-05-17T12:43:04.696Z","Latitude":"54.805301666259766","DST":"A","ICAO":"CYKL","Type":"airport","Country":"Canada","Airport_ID":"80"}
{"Source":"OurAirports","Name":"San Pedro Airport","City":"San Pedro","Altitude":"26","Longitude":"-6.660820007324219","Timezone":"0","Tz_database_time_zone":"Africa/Abidjan","IATA":"SPY","@version":"1","@timestamp":"2019-05-17T12:43:04.740Z","Latitude":"4.746719837188721","DST":"N","ICAO":"DISP","Type":"airport","Country":"Cote d'Ivoire","Airport_ID":"258"}
{"Source":"OurAirports","Name":"Yamoussoukro Airport","City":"Yamoussoukro","Altitude":"699","Longitude":"-5.36558008194","Timezone":"0","Tz_database_time_zone":"Africa/Abidjan","IATA":"ASK","@version":"1","@timestamp":"2019-05-17T12:43:04.740Z","Latitude":"6.9031701088","DST":"N","ICAO":"DIYO","Type":"airport","Country":"Cote d'Ivoire","Airport_ID":"259"}
{"Source":"OurAirports","Name":"Nnamdi Azikiwe International Airport","City":"Abuja","Altitude":"1123","Longitude":"7.263169765472412","Timezone":"1","Tz_database_time_zone":"Africa/Lagos","IATA":"ABV","@version":"1","@timestamp":"2019-05-17T12:43:04.740Z","Latitude":"9.006790161132812","DST":"N","ICAO":"DNAA","Type":"airport","Country":"Nigeria","Airport_ID":"260"}
{"Source":"OurAirports","Name":"Akure Airport","City":"Akure","Altitude":"1100","Longitude":"5.3010101318359375","Timezone":"1","Tz_database_time_zone":"Africa/Lagos","IATA":"AKR","@version":"1","@timestamp":"2019-05-17T12:43:04.740Z","Latitude":"7.246739864349365","DST":"N","ICAO":"DNAK","Type":"airport","Country":"Nigeria","Airport_ID":"261"}
{"Source":"OurAirports","Name":"Benin Airport","City":"Benin","Altitude":"258","Longitude":"5.5995001792907715","Timezone":"1","Tz_database_time_zone":"Africa/Lagos","IATA":"BNI","@version":"1","@timestamp":"2019-05-17T12:43:04.741Z","Latitude":"6.316979885101318","DST":"N","ICAO":"DNBE","Type":"airport","Country":"Nigeria","Airport_ID":"262"}
```
As you can see, it is hard to visualize our data. In next section, we will learn how to use Kibana to easily visualize data.

## Visualization on Kibana
First, you have to navigate to <http://localhost:5601> with your web browser.

### Connect Elasticsearch index to Kibana
Kibana uses index patterns to retrieve data from Elasticsearch indices for things like visualizations.
First, on the left Menu Bar, click on **Management** (the last button) or navigate directly to <http://localhost:5601/kibana#/management>.
Then, on click on **Kibana > Index Patterns > Create index pattern**.

In **Step 1 of 2: Define index pattern**, enter **logstash-airports-1** and for the next step, select **@timestamp** as **The Time field Name**. Create the index pattern.

To visualize your data, navigate to <http://localhost:5601/kibana#/discover> directly or by clicking the **Discover** button in the left menu bar.
If you finished the previous section during the last 15 minutes, you will see the data traffic directly when you navigate to the Discover page. Otherwise, you have to configure the time filter to find the time period during when the data was introduced to Elasticsearch.

Typically, you will find:
![Discover Data](img/discover.PNG?raw=true)

### Create Visualization

Let's create a Pie Chart that indicates the airports distribution through countries.
Click on **Visualize** button of the left side bar, then click on **Create a visualization** and select **Pie** chart. Choose **logstash-airports-1** as the source of this chart. Once you validated, select **Split Slices** as buckets type. After that, choose **Terms** in **Aggregation** field and fill the form as follow:

![Pie Chart Configuration](img/pie-conf.PNG?raw=true)

When you finished, click on the **Apply Changes** button to get you pie chart like bellow:
![Pie Chart](img/pie.PNG?raw=true)

You can save this visualization and add it to the Kibana Dashboard.

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