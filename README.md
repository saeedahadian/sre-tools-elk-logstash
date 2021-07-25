# Logstash

Logstash is a useful tool to process and extract informations from
simple logs.

## Configuration

There are 2 types of configurations in Logstash:

+ Configurations that are related to the Logstash itself e.g. node name,
  log level, etc. These configurations are placed inside the
  `./configs/settings` directory.

+ Configurations related to the pipelines and how incoming logs should
  be processed. These configurations are available inside the
  `./configs/pipeline` directory.

### Pipeline Configuration

The configuration for each specific pipeline is put into the
`./configs/pipeline` directory. Don't forget to also update
`./configs/sesettings/pipeline` file and add your newly added pipelines
to it.

## Load Balancing

We used NginX for load balancing between logstash instances. It's
important to use `stream` block instead of `http` block in NginX
configuration to enable load-balancing in TCP/UDP layer. That's because
filebeat uses a protocol called _lumberjack_ to communicate with
logstash. This protocol sits on the top of TCP.
