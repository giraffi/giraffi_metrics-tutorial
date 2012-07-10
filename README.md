# Giraffi Metrics tutorial

Here you can find the Getting Started with the **Giraffi Metrics**, resource monitoring service that collects, publishes and retrieves metrics with low latency. We show a sample client app that retrieves metrics over WebSocket and renders timeline charts.

  
# Getting Started

## Scenario

1. A procuder ([collectd](http://collectd.org/)) gathers and sends metrics continuously to the Giraffi Metrics (which behaves as Message Broker). 
2. A client app retrieves metrics from the Giraffi Metrics (which behaves as Messaging gateway) over WebSocket.
3. A client app starts rendering timeline charts parsing the retrieved metrics.

## Requirements

* user_id: Unique number provided by the Giraffi Metrics.
* apikey: Authentication token provided by the Giraffi Metrics.
* collectd: The system statistics collection daemon. collectd-5.0 or higher is required.
* [Plugin:AMQP](http://collectd.org/wiki/index.php/Plugin:AMQP): transmits or receives values collected by collectd over the [AMQP](http://www.amqp.org/).


## Setup the producer

###Install collectd

Please refer to the [collectd installation guide](http://collectd.org/download.shtml). The version of collectd must be higher than or equal to 5.0. 

###Configuration

The configuration file can be found, e.g. `/opt/collectd/etc/collectd.conf`.   
 First in the `Global` section, set `Interval` to a value between 1 and 60 seconds at least.
```sh
##############################################################################
# Global                                                                     #   
#----------------------------------------------------------------------------#
# Global settings for the daemon.                                            #   
##############################################################################

#Hostname    ""  
#FQDNLookup   true
#BaseDir     "/opt/collectd/var/lib/collectd"
#PIDFile     "/opt/collectd/var/run/collectd.pid"
#PluginDir   "/opt/collectd/lib/collectd"collectd
#TypesDB     "/opt/collectd/share/collectd/types.db"
Interval      10  # Set to 10 seconds
#Timeout      2   
#ReadThreads  5
```

Then enable the AMQP plug-in in `LoadPlugin` section.

```sh
##############################################################################
# LoadPlugin section                                                         #
#----------------------------------------------------------------------------#
# Lines beginning with a single `#' belong to plugins which have been built  #
# but are disabled by default.                                               #
#                                                                            #
# Lines begnning with `##' belong to plugins which have not been built due   #
# to missing dependencies or because they have been deactivated explicitly.  #
##############################################################################
	
LoadPlugin amqp
##LoadPlugin apache
#LoadPlugin apcups

snip..
```

And configure the AMQP plug-in settings with a server-specific AMQP Routing Key and user credentials.

```sh
##############################################################################
# Plugin configuration                                                       #
#----------------------------------------------------------------------------#
# In this section configuration stubs for each plugin are provided. A desc-  #
# ription of those options is available in the collectd.conf(5) manual page. #
##############################################################################

## Send values to an AMQP broker
<Plugin "amqp">
 <Publish "name">
    Host "broker.giraffi.jp"
    Port "15671"
    VHost "/"
    User "256"
    Password "d229c5cf-370b-4ab3-b34c-9adbba9aa438"
    Exchange "collectd.json.topic"
    RoutingKey "giraffi.collectd.256"
    Persistent false
    StoreRates false
    Format "JSON"
  </Publish>
</Plugin>
```

Note: Only *Publish* block is used with the AMQP plug-in for the Giraffi Metrics. 

`Host` Hostname or IP-address of the AMQP broker.  
`Port` Service name or port number on which the AMQP broker accepts connections.  
`VHost` Name of the virtual host on the AMQP broker to use.  
`User` `user_id` provided by the Giraffi Metrics.  
`Password` `apikey` provided by the Giraffi Metrics.  
`Exchange` The exchange to send values to. Currently the available exchange is only `collectd.json.topic`.  
`RoutingKey` The routing key to set on all outgoing messages. The syntax is valid with only "giraffi.collectd.`user_id `"  
`Persistent` Selects the delivery method to use. If set to true, delivery is guaranteed.   
`StoreRates` Determines whether or not COUNTER, DERIVE and ABSOLUTE data sources are converted to a rate  
`Format` Sepcifies the format (*Command* or *JSON*) in which messages are sent to the broker. Must set to `JSON`.

Finally run `collectd`, it starts gathering and publishing metrics.
```sh
$ sudo /opt/collectd/sbin/collectd -t /opt/collectd/etc/collectd.conf  # Tests config and exit
$ sudo /opt/collectd/sbin/collectd -C /opt/collectd/etc/collectd.conf  # Makes run with the specified config
```
## Start retrieving metrics

Once you set up the producer, now it's time to retrieve metrics from the [Streaming API](https://github.com/giraffi/giraffi_metrics_tutorial/wiki/Streaming-API) over WebSocket. The Streaming API returns the results in a so-called [CSV](http://en.wikipedia.org/wiki/Comma-separated_values) format, each row is located on a separate line, delimited by a line break (LF) and its fields, separated by commas, never contain line breaks (LF), double quotes and commas.


### Setup the client
```sh
$ git://github.com/giraffi/giraffi_metrics_tutorial.git
```	
	
### Retrieve metrics over WebSocket

1. Open `giraffi_metrics_tutorial/index.html` with the WebSocket compliant browser. 
2. Enter `Apikey` and `Src` (*hostname* where the collectd daemon is gathering resource info). 
3. And then click `Start` to connect to the Giraffi Metrics.
4. The app starts retrieving metrics and displays timeline charts.


### Change settings

Edit the following lines in `giraffi_metrics_tutorial/index.html` to change settings (endpoint uri, query string, etc.). 


```javascript
 22 $(function () {
 23   // ************** Settings **************
 24   var GIRAFFI_URL = "wss://ws.giraffi.jp:4443/",
 25       SINGLE_LINE_QUERY_STRING = "fields=time,val&tags=load,shortterm",
 26       MULTI_LINE_QUERY_STRING = "fields=time,val,tags&tags=cpu,user",
 27       start = document.getElementById("start");
 28   // **************************************
```

### Querying

When retrieving metrics, you can use query parameters for filtering the results.

`fields` An array parameter that specifies the fields to return. The available field names are val, time and tags.  
`tags` An array parameter that filters which type of metrics to return. The results contain at least one of the specified tags.  
`src` A string parameter that specifies the source/server where the producer gathers and sends data from.

```sh
wss://ws.giraffi.jp:4443/d229c5cf-370b-4ab3-b34c-9adbba9aa438/?fields=time,val&tags=load,shortterm&src=hoge.example.com # Returns a single row.
wss://ws.giraffi.jp:4443/d229c5cf-370b-4ab3-b34c-9adbba9aa438/?fields=time,val,tags&tags=cpu,user&src=hoge.example.com # Returns multiple rows.
```
