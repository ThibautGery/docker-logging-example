Centralize logs from docker applications
========================================

The purpose of the article is to show how we can centralize logs from docker
application in a database where we can query them.


This article will be built around an example where our application will only be
nginx and our database will be Elasticsearch and Kibana to show beautiful graphs
and diagrams. The code of the example is available on [github](https://github.com/ThibautGery/docker-logging-example).


Architecture
------------

### Choosing our middleware

We need a tool to extract the logs from the docker container and push them in [Elasticsearch](https://www.elastic.co/products/elasticsearch). To do that, we
can use multiple tools like [Logstash](https://www.elastic.co/products/logstash)
or [Fluentd](http://www.fluentd.org/).

[Logstash](https://www.elastic.co/products/logstash) is build by
[Elastic](https://www.elastic.co/) and is well integrated
with Elasticsearch and Kibana. It has lots of plugins.
[Fluentd](http://www.fluentd.org/) describe itself is an open source data
collector for unified logging layer. Docker provides a [driver](https://docs.docker.com/engine/reference/logging/fluentd/) to push log
directly into Fluentd. It also has lots of plugins like [one](https://github.com/uken/fluent-plugin-elasticsearch) to connect to
Elasticsearch.

I chose fluentd because docker pushes it, [Kubernetes](http://kubernetes.io/)
(an important docker project) uses it. Furthermore Elasticsearch's plugin
plays well with Kibana.

### Infrastructure

We could use two types of infrastructure: classic with servers or cloud. I chose
to use the classic one for simplicity.

We now need two server, one for our application and Fluentd and one for our
database, Elasticsearch and Kibana.

### Process
The process can be described in 6 steps

 1. Users connect to our application (nginx) and this generates
 logs
 2. Our containerized application sends its logs to stdout and stderr
 3. Fluentd driver redirects logs to the Fluentd container running
 locally
 4. Fluentd parses and structures logs
 5. Structured data are sent to elasticsearch by batch, you might have to
 wait a minute or two for the data to arrive in Elasticsearch. You can parameterize this behavior with the [Buffer plugins](http://docs.fluentd.org/articles/buffer-plugin-overview)
 6. Data is exposed to administrators through graphs and diagrams with
 Kibana

![fluentd docker process schema](http://i.imgur.com/cTTH3Fi.jpg)


### Application

The application is a simple nginx, it is not the official image because the
access log are not using the default format so I packaged a [new image](https://hub.docker.com/r/thibautgery/docker-nginx)
You can run the app using `docker-compose up` with the following Configuration

```
$ cat ./docker-compose.yml
nginx:
  image: thibautgery/docker-nginx
  ports:
    - 8080:80
```

### Fluentd

Fluentd is a middleware for logs, they flow through steps and are identified with
tags. Here is a simple configuration to receive logs though HTTP and print them
to stdout:

```
$ cat ./fluentd/fluentd.sample
<source>
  @type http
  port 8888
</source>

<match myapp.access>
  @type stdout
</match>
```

Each steps define how to treat data:

 * The first step define how to capture the data, here on port 8888 using HTTP.
 * The second step define how the output the data, here print it on stdout.

You can capture and parse different stream of data, and tags help you
to identify the data in the system. In the previous example the tag is specified
after match `app.access`. Each data is tagged with a string and that way we can
apply different steps to different kind of data.

For example running `curl http://localhost:8888/myapp.access?json={"event":"data"}`
will output `{"event":"data"}``to stdout


Each steps is a plugin. There is more than 150 plugins divided in 6 categories.
The most important ones are:

 * [Input plugins](http://docs.fluentd.org/articles/input-plugin-overview)
 to accept and parse data
 * [Output plugins](http://docs.fluentd.org/articles/output-plugin-overview)
 to send the data to extern system, in our example Elasticsearch

Configuration
-------------

### Docker pushes its logs to Fluentd

First of all, the fluentd agent can be run anywhere but, for simplicity we will
run it one the same node as the application.
The [official image](https://hub.docker.com/r/fluent/fluentd/) can be found on
the docker hub.



We need to configure the Fluentd agent :

```
$ cat ./conf/fluentd
<source>
  @type forward
</source>


<match nginx.docker.**>
  @type stdout
</match>
```
Fluentd will accept connection on the 24224 port and will print it on stdout
thanks to two default plugins [in_forward](http://docs.fluentd.org/articles/in_forward)
and [out_stdout](http://docs.fluentd.org/articles/out_stdout)

You can run fluentd with `docker-compose -f docker-compose-fluentd.yml up`

```
$ cat ./docker-compose-fluentd.yml
fluentd:
  image: fluent/fluentd
  ports:
    - 24224:24224
  volumes:
    - ./conf:/fluentd/etc
```

The default logging option of docker is json-file. We can use the [log-driver](https://docs.docker.com/engine/reference/run/#logging-drivers-log-driver)
option to specify fluentd. By default it will connect to `localhost` on the
`24224` port.

```
$ cat ./docker-compose.yml
nginx:
  image: thibautgery/docker-nginx
  ports:
    - 8080:80
  log_driver: fluentd
  log_opt:
    fluentd-tag: "nginx.docker.{{.Name }}"
```
By default the docker-fluentd driver use a default tag: `docker._container-id_ `.
We override it to be `nginx.docker._container-name_` with the `log_opt`,
`fluentd-tag: "nginx.docker.{{.Name }}"`. The tag in the docker driver must
match the one in Fluentd. You should be able to see the Nginx logs in Fluentd
container log.

Right now, our system is useless. We need to send logs to a distant database,
Elasticsearch.


### Fluentd pushes its log to Elasticsearch

Fluentd will need the [fluent-plugin-elasticsearch](https://github.com/uken/fluent-plugin-elasticsearch)
in order to send data to Elasticsearch. I have package the image [here](https://hub.docker.com/r/thibautgery/fluent.d-es)

We need to update the fluentd agent configuration

```
$ cat ./conf/fluentd
<source>
  @type forward
</source>


<match nginx.docker.**>
  type elasticsearch
  hosts http://elasticsearch.host.com:9200
  logstash_format true
</match>
```
Don't forget to change the the hosts to point to your elasticsearch.

The `logstash_format true` configuration is meant to make writing data into
ElasticSearch compatible to what Logstash writes. By doing this, one could
take advantage of Kibana.

If you use an unsecured network (like internet), you can put basic authentication
in front on your elasticsearch and put the credentials in the url in the fluentd
configuration.

You can run fluentd with:
```
$ cat ./docker-compose-fluentd.yml
fluentd:
  image: thibautgery/fluent.d-es
  ports:
    - 24224:24224
  volumes:
    - ./conf:/fluentd/etc
```
Then you can run the application and query it with your favorite browser to see
some lines in Elasticsearch in the Logstash index. Since Fluentd buffer the data
before sending it by batch, you might have to wait a minute or two.

Unfortunately, the data are not indexed. The Nginx line of log is not
structured: for example, we cannot query all failed HTTP requests (status code >= 400)

This line of log need to be parsed.

### Structure the application logs


Fluentd will need the [fluent-plugin-parser](https://github.com/tagomoris/fluent-plugin-parser)
in order to format a specific field a second time. I have package the image with it [here](https://hub.docker.com/r/thibautgery/fluent.d-es)

We need to update the configuration :

```
$ cat ./conf/fluentd

<source>
  @type forward
</source>

<match nginx.docker.**>
  type parser
  key_name log
  format nginx
  remove_prefix nginx
  reserve_data yes
</match>

<match docker.**>
  type elasticsearch
  hosts http://elasticsearch.host.com:9200
  logstash_format true
</match>
```

The second block of configuration will :

 * use the plugin parser
 * parse the field `logs`
 * parse it using the pre-build Regex of Nginx
 * remove the prefix `nginx` on the tag `nginx.docker.**`
 * keep the previous informations in the message and emit it as `docker.**`

Here we use the system of tag to route the data to the correct steps:
  * the data arrives with `nginx.docker._container-name_` and go through the second step
  * the tag is modified to `docker._container-name_` and go through the third step

Then you can run the application and query it with your favorite browser to see
the data correctly formated in Kibana.

Run it
------

Your system will collect your logs from your application and send it to
Elasticsearch. The docker engine will require to have Fluentd up and running to
start your container.

Nevertheless if Fluentd dies, your containers using Fluentd will continue to
work properly. Furthermore, if Fluentd stop for short period of time, you will
not loose any logs because the docker engine buffer the messages not sent. They
are sent again when Fluentd is, once again, available.


Conclusion
----------

We have cover how to collect and structure logs from docker to push them in
Elasticsearch. You can easily change the Elasticsearch plugin to the [Mongo](https://github.com/fluent/fluent-plugin-mongo) or
[HDFS](https://github.com/fluent/fluent-plugin-webhdfs/) plugin and push logs to the database of your choice.
You can also add an alerting system like [Zabbix](https://github.com/fujiwara/fluent-plugin-zabbix)

You can add node to your infrastructure and add several containers in one node.
Keep in mind that this article doesn't cover everything :

 * how to monitor the docker running fluentd ?
 * how to keep high availability in the monitoring system ?

You can run everything in two commands with the [Ansible scripted repository](https://github.com/ThibautGery/docker-logging-example)
