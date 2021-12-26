# Logstash

Logstash is an intermediary service for processing some input with different filters and sending its outputs to elasticsearch or other data collecting services. It mainly is used for processing services' log files to extract useful informations such as log timestamp, log severity, etc.

## Usage

Based on the shell you're using, copy either of `.env.sample` or `.env.fish.sample` into `.env` and replace its required fields with values that matches your service.

```bash
cp .env.sample .env
```

If you are connecting to elasticsearch over HTTPS, you need to also provide an external volume called `elastic-certs` that includes SSL certificates for connecting to your elasticsearch server. The certificate is a `.pem` file that should be inside the address that you provide in your pipeline configuration.

```text
# pipeline/beats.conf

output {
  elasticsearch {
    hosts => ["${ELASTICHOST:http://10.1.19.1:9200}"]
    index => "%{[@metadata][beat]}-%{[@metadata][version]}-%{+YYYY.MM.dd}"
    user => "logstash_writer"
    password => "${LOGSTASH_WRITER_PASSWORD}"
    ssl => true
    cacert => "${CERTS_DIR}/kibana/elasticsearch-ca.pem"  # CERTIFICATE SHOULD BE HERE!
  }

  stdout {  }
}
```

Then everything is ready and you run the service.

```bash
docker-compose up -d
```

## Configuration

There are 2 types of configurations in Logstash:

+ Configurations that are related to the Logstash itself e.g. node name, log level, etc. These configurations are placed inside the `./configs/settings` directory.

+ Configurations related to the pipelines and how incoming logs should be processed. These configurations are available inside the `./configs/pipeline` directory.

### Pipeline Configuration

The configuration for each specific pipeline is put into the `./configs/pipeline` directory. Don't forget to also update `./configs/settings/pipeline` file and add your newly added pipelines to it.

## Load Balancing

There are 2 ways to implement load-balancing on Logstash services:

### Client-side Load Balancing

A good way to load-balance your requests between different instances of Logstash is to run separate instances of Logstash on different ports and use filebeat's inherent load-balance feature to balance requests between these instances. Your filebeat configuration should be like this:

```yaml
# filebeat.yml

output.logstash:
  enabled: true
  hosts: 
    - "${LOGSTASH_HOST:logstash-host.com}:9202"
    - "${LOGSTASH_HOST:logstash-host.com}:9203"
  loadbalance: true
  worker: 2
  compression_level: 3
  escape_html: false
  ssl.enabled: false
```

As you can see, multiple logstash hosts are given and the `loadbalance` feature is set to true.

### Server-side Load Balancing

Another way to balance requests is to use an NginX instance besides logstash services on the server. While using this method, you have to be careful to use `stream` block instead of `http` block in your NginX configuration so that load-balancing will be enabled in TCP/UDP layer instead of the default HTTP layer. That's because filebeat uses a protocol called _lumberjack_ to communicate with logstash. This protocol sits on top of TCP.

NginX configuration with `stream` block should be something like this:

```conf
# /etc/nginx.conf

stream {
    upstream logstash {
        server logstash-1:5044;
        server logstash-2:5044;
    }

    server {
        listen       5044;
        listen  [::]:5044;

        proxy_pass logstash;
    }
}
```

You might need to change this configuration based on the number of your logstash services and their access ports.
