# Fluent::Plugin::Papertrail

[![Gem Version](https://badge.fury.io/rb/fluent-plugin-papertrail.svg)](https://badge.fury.io/rb/fluent-plugin-papertrail) [![CircleCI](https://circleci.com/gh/solarwinds/fluent-plugin-papertrail/tree/master.svg?style=shield)](https://circleci.com/gh/solarwinds/fluent-plugin-papertrail/tree/master)

## Description

This repository contains the Fluentd Papertrail Output Plugin.

## Installation

Install this gem when setting up fluentd:
```ruby
gem install fluent-plugin-papertrail
```

## Usage

### Setup

This plugin connects to Papertrail log destinations over TCP+TLS. This connection method should be enabled by default in standard Papertrail accounts, see:
```
papertrailapp.com > Settings > Log Destinations
```

To configure this in fluentd:
```xml
<match whatever.*>
  type papertrail
  papertrail_host <your papertrail hostname>
  papertrail_port <your papertrail port>
</match>
```

### Configuring a record_transformer

This plugin expects the following fields to be set for each Fluent record:
```
    message   The log
    program   The program/tag
    severity  A valid syslog severity label
    facility  A valid syslog facility label
    hostname  The source hostname for papertrail logging
```

The following example is a `record_transformer` filter, from the Kubernetes assets [in the Solarwinds fluentd-deployment repo](https://github.com/solarwinds/fluentd-deployment/blob/master/docker/conf/kubernetes.conf), that is used along with the [fluent-plugin-kubernetes_metadata_filter](https://github.com/fabric8io/fluent-plugin-kubernetes_metadata_filter) to populate the required fields for our plugin:
```yaml
<filter kubernetes.**>
  type kubernetes_metadata
</filter>

<filter kubernetes.**>
  type record_transformer
  enable_ruby true
  <record>
    hostname ${record["kubernetes"]["namespace_name"]}-${record["kubernetes"]["pod_name"]}
    program ${record["kubernetes"]["container_name"]}
    severity info
    facility local0
    message ${record['log']}
  </record>
</filter>
```

If you don't set `hostname` and `program` values in your record, they will default to the environment variable `FLUENT_HOSTNAME` or `'unidentified'` and the fluent tag, respectively.

### Advanced Configuration
This plugin inherits a few useful config parameters from Fluent's `BufferedOutput` class.

Parameters for flushing the buffer, based on size and time, are `buffer_chunk_limit` and `flush_interval`, respectively. This plugin overrides the inherited default `flush_interval` to `1`, causing the fluent buffer to flush to Papertrail every second. 

If the plugin fails to write to Papertrail for any reason, the log message will be put back in Fluent's buffer and retried. Retrying can be tuned and inherits a default configuration where `retry_wait` is set to `1` second and `retry_limit` is set to `17` attempts.

If you want to change any of these parameters simply add them to a match stanza. For example, to flush the buffer every 60 seconds and stop retrying after 2 attempts, set something like:
```xml
<match whatever.*>
  type papertrail
  papertrail_host <your papertrail hostname>
  papertrail_port <your papertrail port>
  flush_interval 60
  retry_limit 2
</match>
```

### Advanced idle time configuration
Papertrail can sometimes kill idle connections. This plugin is built to try and avoid the standard re-connection scenarios and has sensible defaults to reduce the number of failed messages which will be sent. You can configure this behaviour with the following settings:
 - Using TCP keep-alive `use_keep_alive`. This means that the TCP sockets created will try to use TCP keep-alives if the socket is idle for a period of time. Default true. There are configurable settings to tune this behaviour:
   - Keep idle time `keep_alive_keep_idle`. This value in seconds is passed directly to the `TCP_KEEPIDLE` socket option, which is the amount of idle time on a socket before sending a keep alive probe. Default 300s
   - Keep alive count `keep_alive_keep_cnt`. This value is passed directly to the `TCP_KEEPCNT` socket option, which is the number of keep alive probes tried before the TCP socket is marked as disconnected. Default 3
   - Keep alive interval `keep_alive_keep_interval`. This value in seconds is passed directly the `TCP_KEEPINTVL` socket option, which is the time between each successful keep alive probe. Default 300s
   - TCP user timeout `tcp_user_timeout`. This value in **milliseconds** is passed directly to the `TCP_USER_TIMEOUT` socket option, which is the amount of time to wait for an unacknowledged packet before terminating the connection. Default 10,000ms
- Controlling the socket re-creation timeout `socket_recreation_timeout`. If a socket sees no log messages for this period of time (in seconds), then it is re-created before trying to send any new messages. Default 1800 seconds (30 minutes), Papertrail will kill any connections which haven't sent application data in the last 59 minutes

## Kubernetes Annotations

If you're running this plugin in Kubernetes with the kubernetes_metadata_filter plugin enabled you can redirect logs to alternate Papertrail destinations by adding annotations to your Pods or Namespaces:

```
solarwinds.io/papertrail_host: 'logs0.papertrailapp.com'
solarwinds.io/papertrail_port: '12345'
```

If both the Pod and Namespace have annotations for any running Pod, the Pod's annotation is used.

## Development

This plugin is targeting Ruby 2.4 and Fluentd v1.0, although it should work with older versions of both.

We have a [Makefile](Makefile) to wrap common functions and make life easier.

### Install Dependencies
`make bundle`

### Test
`make test`

### Release in [RubyGems](https://rubygems.org/gems/fluent-plugin-papertrail)
To release a new version, update the version number in the [GemSpec](fluent-plugin-papertrail.gemspec) and then, run:

`make release`

## Contributing

Bug reports and pull requests are welcome on GitHub at: https://github.com/solarwinds/fluent-plugin-papertrail

## License

The gem is available as open source under the terms of the [Apache License](LICENSE).

# Questions/Comments?

Please [open an issue](https://github.com/solarwinds/fluent-plugin-papertrail/issues/new), we'd love to hear from you.
