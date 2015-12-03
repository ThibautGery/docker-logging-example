Centralize logs from docker applications
========================================

The purpose of the article is to show how we can centralize logs from docker
application in a database where we can run query on them.


This article will be build around an example where our application will only be
nginx and our database will be Elasticsearch and Kibana to show beautiful graphs
and diagrams. The code of the example is available on [github](https://github.com/ThibautGery/docker-logging-example).


Architecture
------------

### Tools

We now need a tool to extract the logs from the docker engine, running the
containers and push them in the
[Elasticsearch](https://www.elastic.co/products/elasticsearch). There is
multiple tools like [Logstash](https://www.elastic.co/products/logstash)
or [Fluentd](http://www.fluentd.org/).

[Logstash](https://www.elastic.co/products/logstash) is build by
[Elastic](https://www.elastic.co/) and is well integrated
with Elasticsearch and Kibana. It has lots of plugins.
[Fluentd](http://www.fluentd.org/) describe itself is an open source data
collector for unified logging layer. Docker provides a [driver](https://docs.docker.com/engine/reference/logging/fluentd/) to push log
directly into Fluentd. It also has lots of plugins like [one](https://github.com/uken/fluent-plugin-elasticsearch) to connect to
Elasticsearch.

I choose fluentd because it's used in some important docker project
like [Kubernetes](http://kubernetes.io/) and the Elasticsearch plugin
play well with Kibana.

We now need two server, one for our application and Fluentd and one for our
database, Elasticsearch and Kibana.

### Flux

Our containerized application will send its logs from the stdout and stderr to
Fluentd running locally in a container. Fluentd will parse and structure them.
Finally it will send them to elasticsearch.

==>

==> INSERER UN SCHEMA DE L'ARCHITECTURE

==>


### Security

If your network cannot be trusted (on internet) then you can add basic
authentication with Nginx in front of Elasticsearch and add the credentials in
the Elasticsearch URL.

### Application

The application is a simple nginx, it's not the official image because the
access log are not using the default format so I packaged a [new image](https://hub.docker.com/r/thibautgery/docker-nginx)
You can run the app using `docker-compose up` with the following Configuration

```
nginx:
  image: thibautgery/docker-nginx
  ports:
    - 8080:80
```

Configuration
-------------

### Docker push its logs to fluentd

First of all, we need to run fluentd on the same machine and we are going to used
the [official image](https://hub.docker.com/r/fluent/fluentd/) on the docker hub.

Fluentd use plugins to manage the data and tags to identify the data.

It has multiple kind of plugins but the most important one are:

 * [Input plugins](http://docs.fluentd.org/articles/input-plugin-overview)
 to accept and parse data
 * [Output plugins](http://docs.fluentd.org/articles/output-plugin-overview)
 to modify the data

We need to configure it in `./conf/fluent.conf`:

```
<source>
  @type forward
</source>


<match nginx.docker.**>
  @type stdout
</match>
```
Fluentd will accept connection on the 24224 port and will print it on stdout
thanks to two default plugin [in_forward](http://docs.fluentd.org/articles/in_forward)
and [out_stdout](http://docs.fluentd.org/articles/out_stdout)

You can run fluentd with:

```
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
nginx:
  image: thibautgery/docker-nginx
  ports:
    - 8080:80
  log_driver: fluentd
  log_opt:
    fluentd-tag: "nginx.docker.{{.Name }}"
```
By default docker use docker.{{ container id }}. We override it to be
nginx.docker.{{ container name }}. It's important that the tag match the one in
fluentd You should be able to see the nginx logs in fluentd container log.

Right now, our system is useless. We need to send the log to a distant database,
Elasticsearch.


### Fluentd push its log to Elasticsearch

Fluentd will need a [plugin](https://github.com/uken/fluent-plugin-elasticsearch)
in order to send data to Elasticsearch. I have package the image [here](https://hub.docker.com/r/thibautgery/fluent.d-es)

We need to update the configuration in `./conf/fluent.conf`:

```
<source>
  @type forward
</source>


<match nginx.docker.**>
  type elasticsearch
  hosts http://elasticsearch.host.com:9200
  logstash_format true
</match>
```
Don't forget to change the the hosts to point to your elasticsearch. We juste use
`logstash_format true` to push the data, as Logstash would do, to be able to use
Kibana.

You can run fluentd with:
```
fluentd:
  image: thibautgery/fluent.d-es
  ports:
    - 24224:24224
  volumes:
    - ./conf:/fluentd/etc
```
Then you can run the application and query it with your favorite browser to see
a few line in elasticsearch in the logstash index.

Unfortunately, the data are not indexed. The nginx line of log is still not
structured: for example, we cannot query all the failed HTTP request (status code >= 400)

This line of log need to be parsed.

### Structure the application logs


Fluentd will need a [plugin](https://github.com/tagomoris/fluent-plugin-parser)
in order to format a specific field a second time. I have package the image with it [here](https://hub.docker.com/r/thibautgery/fluent.d-es)

We need to update the configuration in `./conf/fluent.conf`:

```
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
 * parse the field logs
 * parse it using the pre-build Regex of Nginx
 * remove the prefix `nginx` on the tag `nginx.docker.**`
 * keep the previous informations in the message and emit it as `docker.**`

Then you can run the application and query it with your favorite browser to see
the data correctly formated in Kibana.


Conclusion
----------

We have cover how to collect and structure logs from docker to push them in
Elasticsearch. You can easily change the Elasticsearch plugin to the [Mongo](https://github.com/fluent/fluent-plugin-mongo) or
[HDFS](https://github.com/fluent/fluent-plugin-webhdfs/) plugin and push the log to the database of your choice.

Keep in mind that this article doesn't cover everything :

 * how to monitor the docker running fluentd ?
 * how to keep high availability in the monitoring system ?


 You can run everything in two commands with the [Ansible scripted repository](https://github.com/ThibautGery/docker-logging-example)
