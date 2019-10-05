---
title: "Output Kafka Logs to Syslog"
date: 2019-05-26T23:00:00+02:00
draft: true
---

I have been strugling with Kafka recently, or at least log4j.
The default configuration output a lot of logs to stdout. In my actual configuration, stdout logs from an application are captured by systemd-journald and then given to rsyslog.
This can come handy when the application doesn't know how to log with syslog. 
However, I had an issue with Kafka and journald where logs were also written to the filesystem until the partition is full.
So output log directly to rsyslog was necessary.

## A bit of history

I read a lot of posts on stackoverflow about logging with log4j2 and syslog. I also read the doc of log4j about the `SyslogAppender`. At first it appears to be really simple, however, Kafka doesn't implement log4j2 but log4j `1.2.17`.

Yes, this version is no more supported [since 2015](https://logging.apache.org/log4j/1.2/) but Kafka still use it, even in its [2.3 release](https://github.com/apache/kafka/blob/2.3/gradle/dependencies.gradle#L67).

In this version, the `SyslogAppender` was not really working for me. Logs are sent to syslog but in a incorrect format. Some of the options you can find on solutions on Internet are also only available on log4j2.

## Activate rsyslog UDP input

Because log4j is unable to connect to a `UNIX` socket (even in log4j2), you will need to activate `UDP` listening on rsyslog.

Yes `UDP`. You can't use `TCP`, this feature is only available on log4j2.

So add at least this to your rsyslog configuration to listen on `127.0.0.1:514` with `UDP`.

{{< highlight shell >}}
module(load="imudp")
input(type="imudp" address="127.0.0.1" port="514")
{{< /highlight >}}

Restart your rsyslog instance and you should see rsyslog listening on port 514:
{{< highlight shell >}}
# netstat -nlptu | grep [r]syslog
udp        0      0 127.0.0.1:514           0.0.0.0:*                           484/rsyslogd
{{< /highlight >}}

You can also test that rsyslog is working with UDP with `logger`:
{{< highlight shell >}}
$ logger -d -n 127.0.0.1 -t test "Test logging"

$ tail /var/log/syslog
May 26 10:20:00 kafka-server test Test logging
{{< /highlight >}}

## Configuring log4j on Kafka to send logs to rsyslog

Now that rsyslog is listening on an UDP port we can configure Kafka accordingly.

### Adding the SyslogAppender

What we are going to do is adding an appender to Kafka's `log4j.properties`.

So open this file and add this configuration first:
{{< highlight cfg "linenos=table" >}}
log4j.appender.syslog=org.apache.log4j.net.SyslogAppender
log4j.appender.syslog.threshold=INFO
log4j.appender.syslog.syslogHost=127.0.0.1:514
log4j.appender.syslog.facility=LOCAL6
log4j.appender.syslog.facilityPrinting=false
log4j.appender.syslog.header=false
log4j.appender.syslog.layout=org.apache.log4j.PatternLayout
log4j.appender.syslog.layout.conversionPattern= %d{MMM dd HH:mm:ss} your_hostname kafka:{"date":"%d","level":"%p","category":"%c","message":"%m"}%n
{{< /highlight >}}

What do we have here?

**`threshold`** is here to filter what kind of log level you will be sending to syslog.

**`facility`** is to define the facility of your syslog messages. Here we sent everything on `LOCAL6` but you can change it to your needs.

**`facilityPrinting`** and **`header`** are set to false because we will be formatting our message correctly with `PatternLayout`.
`facilityPrinting` add the facility you defined in the message sent to syslog.
`header` add syslog header: date and time, hostname, topic but with some mistakes (at least in this version).

**`layout.conversionPattern`** is customisable to your needs but you must to keep this portion at the beginning
{{< highlight text >}}
 %d{MMM dd HH:mm:ss} your_hostname kafka:
{{< /highlight >}}
and `%n` at the end (to put a new line character).
The whitespace before `%d` is important here.
`your_hostname` must match the hostname of your server.

I'm also using a json like logging but you can format it the way you want. Options are available [here](https://logging.apache.org/log4j/1.2/apidocs/org/apache/log4j/PatternLayout.html).

### Send logs to your syslog appender

Last bit of configuration is to tell log4j to forward logs to your appender.

You can for example send everything to syslog (no stdout) like this:
{{< highlight cfg >}}
log4j.rootLogger=INFO, syslog
{{< /highlight >}}

Or just add syslog to the current configuration:
{{< highlight cfg >}}
log4j.rootLogger=INFO, stdout, syslog
{{< /highlight >}}

### Restart Kafka

Restart kafka to apply your newly log4j configuration!

Here is a log sample with my configuration when arriving on our central rsyslog:
{{< highlight logs >}}
2019-05-26T22:17:04+00:00 kafka-server kafka.info {"date":"2019-05-26 22:17:04,763","level":"INFO","category":"kafka.coordinator.group.GroupMetadataManager","message":"[GroupMetadataManager brokerId=1]
 Scheduling loading of offsets and group metadata from __consumer_offsets-7"}
2019-05-26T22:17:04+00:00 kafka-server kafka.info {"date":"2019-05-26 22:17:04,763","level":"INFO","category":"kafka.coordinator.group.GroupMetadataManager","message":"[GroupMetadataManager brokerId=1]
 Scheduling loading of offsets and group metadata from __consumer_offsets-13"}
2019-05-26T22:17:04+00:00 kafka-server kafka.info {"date":"2019-05-26 22:17:04,763","level":"INFO","category":"kafka.coordinator.group.GroupMetadataManager","message":"[GroupMetadataManager brokerId=1]
 Scheduling loading of offsets and group metadata from __consumer_offsets-19"}
{{< /highlight >}}

# Conclusion

Maybe Kafka will migrate to log4j2 one day but in the mean time this configuration is working as I want.

I regret that there isn't a lot of documentation about how to output logs the way we want either on Kafka website or log4j (1.2). I had hard time figuring out why I wasn't able to forward logs, turns out it was because the `SyslogAppender` doesn't have the correct format (tcpdump helped here).

I hope that this article was helpful to you, do not hesitate to share it!
