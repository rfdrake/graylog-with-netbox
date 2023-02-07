# Graylog with a Netbox ip lookup

This is an example of using Graylog with a Netbox ipam lookup for each
message.  The message types I collected are TACACS, router logs, and DHCP logs.

You may need to tweak the grok rules and extractors a bit to fit your environment.

The results of this is that each message should have a new field called "site"
which is the site associated with an IP prefix in netbox.

You might want to tweak the query for other data if, say, Tenant is more
important than the site.

You could probably enrich the messages with any netbox field, like Location,
Tenant, and Site together.

# Caching is essential

Netbox does it's best, but graylog sees around 500 messages/sec.  The prefix
cache I have is showing around 140 lookups/s (it varies).  There is a 98.8%
hit rate, and netbox is getting between 1 and 5 lookups/s.

If you were to try to send all these to netbox I believe it would fall over.
It's made for light API use and heavy interactive use, not really for a log
server slamming it all day.

I've never seen a performance issue caused by the graylog looks we have.

# Docker compose file

You can use it as an example, but I recommend upgrading to the new version of
mongodb and graylog.  You would need to change "mygrayloghost.example.com" to
your actual domain.

# filebeat + syslog-ng

We used filebeat to pull the logs from text files into graylog.  We used
logstash output for reasons I can't remember, but I believe it was the only
one I could get to log the right format.  I remember wanting to write directly
from syslog-ng to elastic, or from filebeat directly to elastic, but both of
those approaches caused problems.

We didn't really do any log enrichment with filebeat.  I've included most of
the config in filebeat.yml.

# Separating devices using syslog-ng

You can receive syslog messages on different ports, so use 2514, or
3514, etc, instead of the base 514.

You can also use syslog facilities (local1-local7), but not all devices
support them and you will run out if you have lots of device types.

Another option is filtering on an IP range using netmask("...");  This is good
if you want your logs broken out into directories by property.  The downside
being that you will need to edit syslog-ng more often to add more netmask
ranges.

# A note about log rotation

Log rotation is fine on a standalone host, but you don't want to do it on a
central syslog server.  Stopping and starting the syslog server each night
gives you a window where log messages may be lost.  There is also the chance
that configuration errors will cause the server to never restart and you won't
have logs until the problem is resolved.

Syslog-ng has the ability to specify a filename with a date in it.  For
example:

```
destination smartzone {
    file("/log/smartzone/smartzone.${R_YEAR}-${R_MONTH}-${R_DAY}.log" create_dirs(yes));
};
```

This will automatically role to the new file each night at midnight.  Filebeat
will notice the new file and start pushing it to graylog.

You can write a cron job that will cleanup old files.  This is what we used:

```
30 0 * * * syslog find /log/ -type f -mtime +730 -exec rm {} \;
```

# A note about compression

I don't like gzipping logfiles each day.  It makes them harder to grep and
other things.  I think the better solution is to use a filesystem that
supports compression.  We use zfs for our logging filesystem.  It isn't quite
as good as compressing with gzip -9 or bzip2 but it ends up being completely
managable.

# A note about time

Syslog doesn't do a great job of recording timestamps.  Some devices ship logs
with no timestamp.  Once the message reaches the syslog-ng server they have a time
added to the message.  Other devices ship logs with a builtin timestamp.  In
those cases syslog-ng will still add another header with it's time.

Then filebeat processes this stuff and adds another header, which is a
timestamp of when filebeat processed the message.  There are multiple problems
with this.

1. Syslog timestamps aren't all consistent, so you might end up with
something you can't parse.
2. Syslog timestamps don't always include timezones, so you have no idea what
the actual time is.
3. With few exceptions you don't know the accuracy of the source clock so
you're taking the source device's word for it that the event happened at that
time.

With all of that said, you have multiple choices of which time to use as the
"true" value.  In 99% of the cases it won't matter, but in rare cases there
might be legal problems with getting it wrong.

My suggestions are:

1.  Force every device and host in the network to use UTC and update with reliable
NTP servers.
2.  Place filebeat (or whatever you use for log ingestion) on the source
device if you can.
3. Use the filebeat processed time as the actual arrival time (but attempt to
preserve the source timestamp if possible, in case it's needed later)


