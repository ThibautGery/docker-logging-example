Centralize logs from Docker applications
========================================

The purpose of the article is to show how we can centralize logs from Docker
application in a database where we can query them.


This article will be built around an example where our application will only be
nginx and our database will be Elasticsearch and Kibana to show beautiful graphs
and diagrams. The code of the example is available on [github](https://github.com/ThibautGery/Docker-logging-example).

We need to collect and transport our logs from a distributed system to a
centralize remote location in real time with reliability. That way, we can get
an aggregate vision of the system in almost real time.

The logging system is plugged at the container level because the application
should be loosely coupled with the logging system. Depending on the environment
(development, pre-production, production) you might not send logs to the same
system: a file for development, elasticseach for pre-production, elasticseach
and HDFS for production.

Architecture
------------

### Choosing our middleware

We need a tool to extract the logs from the Docker container and push them in [Elasticsearch](https://www.elastic.co/products/elasticsearch). In order to do that, we
can use multiple tools like [Logstash](https://www.elastic.co/products/logstash)
or [Fluentd](http://www.fluentd.org/).

[Logstash](https://www.elastic.co/products/logstash) is built by
[Elastic](https://www.elastic.co/) and is well integrated
with Elasticsearch and Kibana. It has lots of plugins.
[Fluentd](http://www.fluentd.org/) describes itself as an open source data
collector for unified logging layer. Docker provides a [driver](https://docs.docker.com/engine/reference/logging/fluentd/) to push logs
directly into Fluentd. It also has a lot of plugins like [one](https://github.com/uken/fluent-plugin-elasticsearch) to connect to
Elasticsearch.

I chose Fluentd because Docker pushes it, and [Kubernetes](http://kubernetes.io/)
(an important Docker project) uses it. Furthermore Elasticsearch's plugin
plays well with Kibana.

### Infrastructure

We could use two types of infrastructure: either a classical architecture with servers or cloud. I chose
to use the classic one for simplicity.

Therefore two servers are needed, one for our application and Fluentd and one for our
database, Elasticsearch and Kibana.

### Process
The process can be described in 6 steps

 1. Users connect to our application (nginx) and this generates
 logs
 2. Our containerized application sends its logs to stdout and stderr
 3. Docker intercepts logs from the container and uses its native Fluentd output
 driver to send them to the Fluentd container running locally
 4. Fluentd parses and structures logs
 5. Structured data are sent to elasticsearch in batches, you might have to
 wait a minute or two for the data to arrive in Elasticsearch. You can
 parameterize this behavior with the
 [Buffer plugins](http://docs.fluentd.org/articles/buffer-plugin-overview)
 6. Data is exposed to administrators through graphs and diagrams with
 Kibana

![Fluentd Docker process schema](http://i.imgur.com/SW57gC7.png)


### Application

The application is a simple nginx, I packaged a [new image](https://hub.Docker.com/r/thibautgery/Docker-nginx) since
the official one uses a custom logger that is not appropriate for our purpose.
You can run the app using `Docker-compose up` with the following Configuration

```
$ cat ./Docker-compose.yml
nginx:
  image: thibautgery/Docker-nginx
  ports:
    - 8080:80
```

### Fluentd

Fluentd is a middleware to collect logs, they flow through steps and are identified
with tags. Here is a simple configuration with two steps to receive logs through
HTTP and print them to stdout:

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

In this sample, each step defines how to treat data:

 * The first step defines how to capture the data, here on port 8888 using HTTP.
 * The second step defines how to output the data, in this case by printing it on stdout.

The data is streamed through Fluentd. Each chunk of data is tagged with a label.
This tag is used to route the data between the different steps.

In the previous example the tag is specified after the key `match` : `app.access`.
The tag of the incoming data is the URL of the request.
For example running `curl http://localhost:8888/myapp.access?json={"event":"data"}`
will output `{"event":"data"}` to stdout.

You can learn more in this [slideshare](http://www.slideshare.net/treasure-data/the-basics-of-fluentd).


Each step is a plugin. There are more than 150 plugins divided in 6 categories.
The most important ones are:

 * [Input plugins](http://docs.fluentd.org/articles/input-plugin-overview)
 to accept and parse data
 * [Output plugins](http://docs.fluentd.org/articles/output-plugin-overview)
 to send the data to external systems, in our example Elasticsearch

Configuration
-------------

### Docker pushes its logs to Fluentd

First of all, the Fluentd agent can be run anywhere, but for the sake of simplicity we will
run it on the same node as the application.
The [official image](https://hub.Docker.com/r/fluent/fluentd/) can be found on
the Docker hub.



We need to configure the Fluentd agent :

```
$ cat ./conf/Fluentd
<source>
  @type forward
</source>


<match nginx.Docker.**>
  @type stdout
</match>
```
Fluentd will accept connections on the 24224 port and will print logs on stdout
thanks to two default plugins [in_forward](http://docs.fluentd.org/articles/in_forward)
and [out_stdout](http://docs.fluentd.org/articles/out_stdout)

You can run Fluentd with `Docker-compose -f Docker-compose-fluentd.yml up`

```
$ cat ./Docker-compose-fluentd.yml
Fluentd:
  image: fluent/Fluentd
  restart: always
  ports:
    - 24224:24224
  volumes:
    - ./conf:/Fluentd/etc
```

The default logging format option for Docker is json-file. We can use the [log-driver](https://docs.Docker.com/engine/reference/run/#logging-drivers-log-driver)
option to specify Fluentd. By default it will connect to `localhost` on the
`24224` port.

```
$ cat ./Docker-compose.yml
nginx:
  image: thibautgery/Docker-nginx
  ports:
    - 8080:80
  log_driver: Fluentd
  log_opt:
    Fluentd-tag: "nginx.Docker.{{.Name }}"
```
The Docker driver uses a default tag for Fluentd: `Docker._container-id_ `.
We override it to be `nginx.Docker._container-name_` with the `log_opt`,
`Fluentd-tag: "nginx.Docker.{{.Name }}"`. The tag in the Docker driver must
match the one in Fluentd. You should be able to see the Nginx logs in Fluentd
container log.

Right now, our system is useless. We need to send logs to a distant database,
Elasticsearch.


### Fluentd pushes its log to Elasticsearch

Fluentd will need the [fluent-plugin-elasticsearch](https://github.com/uken/fluent-plugin-elasticsearch)
in order to send data to Elasticsearch. I have packaged the image [here](https://hub.Docker.com/r/thibautgery/fluent.d-es)

We need to update the Fluentd agent configuration

```
$ cat ./conf/Fluentd
<source>
  @type forward
</source>


<match nginx.Docker.**>
  type elasticsearch
  hosts http://elasticsearch.host.com:9200
  logstash_format true
</match>
```
Don't forget to change the hosts to point to your Elasticsearch instance.

The `logstash_format true` configuration is meant to write data into an
ElasticSearch compatible format with what Logstash writes. By doing this, one could
take advantage of Kibana.

You can run Fluentd with:
```
$ cat ./Docker-compose-Fluentd.yml
Fluentd:
  image: thibautgery/fluent.d-es
  ports:
    - 24224:24224
  volumes:
    - ./conf:/Fluentd/etc
```
Then you can run the application and query it with your favorite browser to see
some lines in Elasticsearch in the Logstash index. Since Fluentd buffers the data
before sending them by batch, you might have to wait a minute or two.

![Kibana result unstructured](http://i.imgur.com/cXrFzEO.png)


Unfortunately, only the meta data of Docker are sent (like the docker name, label, id...)
but the field log is the logs from nginx and they are not structured. For
example, we cannot query all failed HTTP requests (status code >= 400)

This line of log need to be parsed.

### Structure the application logs


Fluentd will need the [fluent-plugin-parser](https://github.com/tagomoris/fluent-plugin-parser)
in order to format a specific field a second time. I have packaged the image with it [here](https://hub.Docker.com/r/thibautgery/fluent.d-es)

We need to update the configuration :

```
$ cat ./conf/Fluentd

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



Here we use the tag's concept to route the data to the correct steps:

![Flow diagram of fluentd](http://i.imgur.com/9YeOcod.png)

  1. the data arrives with the tag setup by docker driver :
`nginx.docker._container-name_`
  2. Fluentd sends it to the second step
  3. the tag is modified to `docker._container-name_`
  4. data goes through the third step
  5. data is sent to Elasticsearch

Then you can run the application and query it with your favorite browser to see
the data correctly formatted in Kibana.

You can create diagrams from these data :
![Kibana result structured](http://i.imgur.com/SYVSPLH.png)

Run it
------

Your system will collect logs from your application and send them to
Elasticsearch. The Docker engine will require to have Fluentd up and running to
start your container.

Even if Fluentd dies, your containers using Fluentd will continue to
work properly. Furthermore, if Fluentd stops for short periods of time, you will
not lose any logs because the Docker engine buffers unsent messages so that they will be
sent when Fluentd is back online.

Finally, since Docker 1.9 you can show labels and environment variable with the
logging driver of Docker. In our example, we added : `service: nginx` and it will
be shown in Kibana.

Conclusion
----------

So far, we have seen how to collect and structure logs from Docker to push them in
Elasticsearch. You can easily change the Elasticsearch plugin to the [Mongo](https://github.com/fluent/fluent-plugin-mongo) or
[HDFS](https://github.com/fluent/fluent-plugin-webhdfs/) plugin and push logs to the database of your choice.
You can also add an alerting system like [Zabbix](https://github.com/fujiwara/fluent-plugin-zabbix)

You can add nodes to your infrastructure and add several containers in one node.
Keep in mind that this article doesn't cover everything :

 * how to monitor the Docker running Fluentd ?
 * how to keep high availability in the monitoring system ?

You can run everything in two commands with the [Ansible scripted repository](https://github.com/ThibautGery/Docker-logging-example)
