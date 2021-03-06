kafka-ganglia
=============

Quick and dirty how-to on monitoring kafka metrics using ganglia. 



with Kafka:

Make sure to edit kafka-run-class.sh to include the following:

    KAFKA_JMX_OPTS="-Dcom.sun.management.jmxremote=true -Dcom.sun.management.jmxremote.authenticate=false  -Dcom.sun.management.jmxremote.ssl=false "

If you want to add security you'll have to set to true, enabling users would done inside JMXTrans' config file.

Inside kafka-server-start.sh set the JMX port to:

    export JMX_PORT=${JMX_PORT:-9999}



with JMXTrans:
To install:

http://code.google.com/p/jmxtrans/

Download the .deb package.
    dpkg -i jmxtrans_239-1_amd64.deb (replace the version number)
    
Enter in the JVM heap size you want: 512 is the default. The more JVM's you need to monitor, the more memory you will probably need.
If you are getting OutOfMemoryError's, then increase this value by editing /etc/default/jmxtrans. 

The application is installed in:

    /usr/share/jmxtrans
Configuration options are stored in:

    /etc/default/jmxtrans
The init script in:

    /etc/init.d/jmxtrans (this wraps the jmxtrans.sh discussed below)
Add your .json files to: 

    /usr/share/jmxtrans  (it's supposed to read from /var/lib/jmxtrans, but that's a lie.)


To run the jmxtrans:

    ./jmxtrans.sh start [optional path to one json file] 

To stop jmxtrans:

    ./jmxtrans.sh stop 

Options you may want to configure:

    JSON_DIR - Location of your .json files. Defaults to '.'
    LOG_DIR - Location of where the log files get written. Defaults to '.'
    SECONDS_BETWEEN_RUNS - How often jobs run. Defaults to 60. 


________________________________________________________________________________________________________



JMXTrans uses JSON configuration files to specify what hosts to poll JMX, which metrics/attributes to read, and then which Ganglia host to send to.  Every server being monitored will need its own JSON entry.



######################################################################################

This is my example, monitoring a Kafka instance and sending UDP data to
a Ganglia host all on the same local machine.
   
This will send metric values to Ganglia, which will then generate round robin database
files for each attribute.

For example, with: 
    
    "obj" : "kafka:type=kafka.SocketServerStats",
      "resultAlias": "Kafka",
      "attr" : [ "AvgProduceRequestMs"

Ganglia will output: "Kafka.AvgProduceRequestMs.rrd" into /var/lib/ganglia/rrds/{Gmetad Host}/{IP}


    {
      "servers" : [ {
        "port" : "9999",        #Kafka JMX Port
        "host" : "127.0.0.1",   #Kafka server
        "queries" : [ {
          "outputWriters" : [ {
            "@class" :
      "com.googlecode.jmxtrans.model.output.GangliaWriter",
            "settings" : {
          "groupName" : "jvmheapmemory",
          "port" : 8649,         #port to send UDP, confirm on Ganglia host's gmond.conf
          "host" : "127.0.0.1"   #Ganglia host
          }
      } ],
        "obj" : "java.lang:type=Memory",
        "resultAlias": "heap",
        "attr" : [ "HeapMemoryUsage", "NonHeapMemoryUsage" ]
      }, {
        "outputWriters" : [ {
          "@class" :
      "com.googlecode.jmxtrans.model.output.GangliaWriter",
            "settings" : {
          "groupName" : "kafka stats",
          "port" : 8649,
          "host" : "127.0.0.1"
          }
      } ],
        "obj" : "kafka:type=kafka.SocketServerStats",
        "resultAlias": "Kafka",
        "attr" : [ "ProduceRequestsPerSecond", "FetchRequestsPerSecond", "BytesReadPerSecond", "BytesWrittenPerSecond"  ]
      }, {
        "outputWriters" : [ {
          "@class" :
        "com.googlecode.jmxtrans.model.output.GangliaWriter",
          "settings" : {
          "groupName" : "jvmGC",
          "port" : 8649,
          "host" : "127.0.0.1",
          "typeNames" : [ "name" ]
          }
       } ],
        "obj" : "java.lang:type=GarbageCollector,name=*",
        "resultAlias": "GC",
        "attr" : [ "CollectionCount", "CollectionTime" ]
       } ],
         "numQueryThreads" : 2
      } ]
    }





Once Ganglia is generating RRDs for each metric, if you want to edit the file names you'll have to edit
the JSON files, restart JMXTrans, and then erase the existing RRDs.  New ones are written every
few seconds, so it's a good idea to do this to prevent junk files from piling up.
 


The Ganglia Webfront automatically looks for RRDs on the /var/lib/ganglia/rrds/ path, so on the Ganglia
Webfront page using the "Metric" dropdown, you can select/display each RRDs data, but it's very plain.  
It will be listed as the RRD's file name.
You can also write a report graph.php to display multiple metrics comparatively from the RRDs.
Ganglia Web provides a sample report template as a base located in /var/www/ganglia-web/graph.d

#### Example: requests_report.php  

    <?php

    /* Pass in by reference! */
    function graph_requests_report ( &$rrdtool_graph ) {

    global $context, 
           $cpu_nice_color,  
           $cpu_user_color,
           $hostname,
           $range,
           $rrd_dir,
           $size;

    $title = 'Kafka Requests';
    if ($context != 'host') {
       $rrdtool_graph['title'] = $title;
    } else {
       $rrdtool_graph['title'] = shortenFQDN($hostname) . " $title last $range";

    }

    $rrdtool_graph['lower-limit']    = '0';
    $rrdtool_graph['vertical-label'] = 'requests/msec';
    $rrdtool_graph['extras']         = '--rigid';
 
    if($context != "host" )
                {
                    $rrdtool_graph['series'] ="DEF:'produced'='${rrd_dir}/Kafka.ProduceRequestsPerSecond.rrd':'sum':AVERAGE "
                   ."DEF:'consumed'='${rrd_dir}/Kafka.FetchRequestsPerSecond.rrd':'sum':AVERAGE "
                   ."AREA:'consumed'#$cpu_user_color:'Consumed' "
                   ."STACK:'produced'#$cpu_nice_color:'Produced' "
                }

    return $rrdtool_graph;
    }


    ?>


You also have to modify Ganglia Web's conf.php. Include at the end:

    $optional_graphs = array('requests_report');
    
You also need to modify get_context.php to the list of report graphs. Include at end:

    "requests_report" => 1,

