# Flank - Streaming Demo

**Status:** *Work in progress*

---

# Introduction

The purpose of this repo is to enable the easy and quick setup of a FLANK demo use-case. We are using Flink/Nifi/Kafka and Kudu as well as some other tools to showcase a streaming data use-case from edge to visualization.

It is setup very flexible and can be reused for different scenarios. Below is using a fleet control example. The main goal is a 10-20 minute **demo** to customers. The sensors can be renamed to suit the scenario and the geolocation data can also be updated/edited to fit to the location of the customer.

# Overview

Below is the demo flow including Nifi to collect the data simulating sensors at the edge. These could be on vehicles or wearables etc. We are then combining different datasources and enrich the data to display realtime streaming data on a map.

![](assets/20220623_093924_image.png)

![](assets/20220623_093538_image.png)

# Pre-requisites

Follow the instructions [here](https://github.com/asdaraujo/edge2ai-workshop) to setup an 'edge2ai' one node cluster. Alternatively you can follow my tutorial [here](https://www.reginahack.com/tutorial-cdp-private-cloud-base-dev-cluster/) to do the setup manually. I've been using AWS and recommend the EC2 instance type to be *r5a.4xlarge*.

If you followed the edge2ai guide, you will also have a web instance that provides links to the different applications. For the purpose of this doc, we will use this as the basis. The next steps are:

* Get IP from the ‘web’ instance and go to the web portal
* Login with the username/password as defined per setup
* Download ssh key from web portal and update permission

# Step 1: Start the data stream

## Nifi (optional)

*You can use NiFi to create the sensor stream or connect via ssh and start a generator script via console (see below).*

In NiFi click on the top right burger menu and select *Templates*. Import the following Nifi flow:

[20220622_083349_Streaming-Demo_NiFi_Flow.xml](assets/20220622_083349_Streaming-Demo_NiFi_Flow.xml)

You will then need to update the IPs of your processors as well as the record reader and writer, to reflect your elastic IP/your cluster's IP.

Your canvas should now look like below:

![](assets/20220622_083109_nifi.png)

Go to the Schema Registry and follow lab 1 from [here](https://github.com/asdaraujo/edge2ai-workshop/blob/trunk/workshop_nifi.adoc#lab-1---registering-a-schema-in-schema-registry) to add the Schema to the Registry.

You can find the schema text [here](https://raw.githubusercontent.com/cloudera-labs/edge2ai-workshop/master/sensor.avsc).

The result will look as follows:

![](assets/20220627_080213_image.png)

## Weather data

1. Download the scripts

```
ssh -i workshop.pem centos@IP

sudo wget https://github.com/zBrainiac/kafka-producer/releases/download/0.0.1/kafka-producer-0.0.1.0.jar -P /opt/cloudera/parcels/FLINK/lib/flink/examples/streaming
```

2. Start iot sensor stream (JSON) - (or use NiFi as described above)

```
cd /opt/cloudera/parcels/FLINK/lib/flink/examples/streaming &&

java -classpath kafka-producer-0.0.1.0.jar producer.KafkaIOTSensorSimulator edge2ai-0.dim.local:9092 1000
```

3. Start Weather data stream (CSV)

```
cd /opt/cloudera/parcels/FLINK/lib/flink/examples/streaming

java -classpath kafka-producer-0.0.1.0.jar producer.KafkaLookupWeatherCondition edge2ai-0.dim.local:9092
```

# Step 2: Create the Kudu Tables

1. Create *sensors*

```sql
DROP TABLE if exists sensors;
CREATE TABLE sensors (
sensor_id INT,
sensor_ts TIMESTAMP,
sensor_0 DOUBLE,
sensor_1 DOUBLE,
sensor_2 DOUBLE,
sensor_3 DOUBLE,
sensor_4 DOUBLE,
sensor_5 DOUBLE,
sensor_6 DOUBLE,
sensor_7 DOUBLE,
sensor_8 DOUBLE,
sensor_9 DOUBLE,
sensor_10 DOUBLE,
sensor_11 DOUBLE,
is_healthy INT,
PRIMARY KEY (sensor_ID, sensor_ts)
)
PARTITION BY HASH PARTITIONS 16
STORED AS KUDU
TBLPROPERTIES ('kudu.num_tablet_replicas' = '1');
```

2. Create *GeoLocation* data

```sql
DROP TABLE if exists RefData_GeoLocation;
CREATE TABLE if not exists RefData_GeoLocation (
sensor_id INT,
city STRING,
lat DOUBLE,
lon DOUBLE,
PRIMARY KEY (sensor_ID)
)
PARTITION BY HASH PARTITIONS 16
STORED AS KUDU
TBLPROPERTIES ('kudu.num_tablet_replicas' = '1');
```

3. Add your custom location data

   [austria](assets/20220623_111410_austria.sql)

   [hungary](assets/20220623_111508_hungary.sql)

   [switzerland](assets/20220623_093403_switzerland.sql)
4. Create target table *sensors_joined*

   Update the column names to suit your example! Below are car/truck sensors.

```sql
DROP TABLE if exists sensors_joined ;
CREATE TABLE sensors_joined(
sensor_id BIGINT,
sensor_ts BIGINT,
liquid_level BIGINT,
airflow BIGINT,
temperature BIGINT,
humidity BIGINT,
speed BIGINT,
brake_wear BIGINT,
tire BIGINT,
AirTemperature2m STRING,
city STRING,
lat DOUBLE,
lon DOUBLE,
PRIMARY KEY (sensor_id, sensor_ts)
)
PARTITION BY HASH PARTITIONS 16
STORED AS KUDU
TBLPROPERTIES ('kudu.num_tablet_replicas' = '1');
```

# Step 3: Create virtual tables in SQL Stream Builder

1. Create Weather condition upsert

```sql

DROP TABLE IF EXISTS `weather_condition_upsert`;
CREATE TABLE `weather_condition_upsert` (
`stationid` INT,
`eventDate` BIGINT,
`tre200s0` STRING,
`rre150z0` STRING,
`sre000z0` STRING,
`gre000z0` STRING,
`ure200s0` STRING,
`tde200s0` STRING,
`dkl010z0` STRING,
`fu3010z0` STRING,
`fu3010z1` STRING,
`prestas0` STRING,
`pp0qffs0` STRING,
`pp0qnhs0` STRING,
`ppz850s0` STRING,
`ppz700s0` STRING,
`dv1towz0` STRING,
`fu3towz0` STRING,
`fu3towz1` STRING,
`ta1tows0` STRING,
`uretows0` STRING,
`tdetows0` STRING,
PRIMARY KEY (`stationid`) NOT ENFORCED
) WITH (
'connector' = 'upsert-kafka',
'topic' = 'kafka_LookupWeatherCondition',
'properties.bootstrap.servers' = 'edge2ai-0.dim.local:9092',
'properties.group.id' = 'kafka_LookupWeatherCondition_upsert',
'key.format' = 'raw',
'value.format' = 'csv'
);

```

2. Create *iot_enriched_source*

*Virtual Tables on SSB are a way to associate a Kafka topic with a schema so that we can use that as a table in our queries.*

---

We will use a Source Virtual Table now to read from the topic.

To create our Source Virtual Table, click on Console (on the left bar) > Tables > Add table > Apache Kafka.

![](assets/20220623_072941_image.png)

On the Kafka Source window, enter the following information:

Virtual table name: `iot_enriched_source`

Kafka Cluster:      `Local Kafka`

Topic Name:         `iot`

Data Format:        `JSON`

![](assets/20220623_073120_image.png)

![]()

Ensure the Schema tab is selected. Scroll to the bottom of the tab and click Detect Schema. SSB will take a sample of the data flowing through the topic and will infer the schema used to parse the content. Alternatively you could also specify the schema in this tab.

![](assets/20220623_073202_image.png)

Click on the Event Time tab, define your time handling. You can specify Watermark Definitions when adding a Kafka table. Watermarks use an event time attribute and have a watermark strategy, and can be used for various time-based operations.
The Event Time tab provides the following properties to configure the event time field and watermark for the Kafka stream:

1. Input Timestamp Column: name of the timestamp column in the Kafka table from where the event time column is mapped. If you wanna use a column from the event message you have to unselect the box Use Kafka Timestamp first.
2. Event Time Column: new name of the timestamp column where the watermarks are going to be mapped

Watermark seconds : number of seconds used in the watermark strategy. The watermark is defined by the current event timestamp minus this value.

Input Timestamp Column: `sensor_ts`

Event Time Column:      `event_ts`

Watermark Seconds:      `3`

![](assets/20220623_073253_image.png)

If we need to manipulate the source data to fix, cleanse or convert some values, we can define transformations for the data source to perform those changes. These transformations are defined in Javascript.
The serialized record read from Kafka is provided to the Javascript code in the record.value variable. The last command of the transformation must return the serialized content of the modified record.
The sensor_0 data in the iot topic has a pressure expressed in micro-pascal. Let’s say we need the value in pascal scale. Let’s write a transformation to perform that conversion for us at the source.
Click on the Transformations tab and enter the following code in the Code field:

```javascript
// Kafka payload (record value JSON deserialize to JavaScript object)

var payload = JSON.parse(record.value);

payload['sensor_0'] = Math.round(payload.sensor_0 * 1000);

JSON.stringify(payload);
```

![](assets/20220623_073345_image.png)

Click on the Properties tab, enter the following value for the Consumer Group property and click Save changes.

![](assets/20220623_073425_image.png)

Setting the Consumer Group properties for a virtual table will ensure that if you stop a query and restart it later, the second query execute will continue to read the data from the point where the first query stopped, without skipping data. However, if multiple queries use the same virtual table, setting this property will effectively distribute the data across the queries so that each record is only read by a single query. If you want to share a virtual table with multiple distinct queries, ensure that the Consumer Group property is unset.

3. Add Kudu as Data Catalog in SBB

Click on Data Providers and +Register Catalog.

Name: `kudu_source`

Catalog Type: `kudu`

Kudu Masters: `edge2ai-0.dim.local:7051`

![](assets/20220623_073600_image.png)

Then click ‘**validate**’ and ‘**Add Table**’ and then go back to the **Console > SQL**.

Make sure to add the EXACT same column names as in step 2.4!

```sql
INSERT INTO `kudu_source`.`default_database`.`default.sensors_joined`
SELECT
iot.`sensor_id`,
iot.`sensor_ts`,
iot.`sensor_0` as liquid_level,
iot.`sensor_1` as airflow,
iot.`sensor_2` as temperature,
iot.`sensor_3` as humidity,
iot.`sensor_4` as speed,
iot.`sensor_5` as brake_wear,
iot.`sensor_6` as tire,
Replace(weather.`tre200s0`, '''', '') as AirTemperature2m,
Replace(geo.city, '''', ''),
geo.`lat`,
geo.`lon`
FROM iot_enriched_source AS iot
JOIN  weather_condition_upsert AS weather
ON iot.`sensor_id` = weather.`stationid`
LEFT JOIN `kudu_source`.`default_database`.`default.refdata_geolocation` AS geo
ON iot.`sensor_id` = geo.`sensor_id`;
```

![](assets/20220623_073722_image.png)

Start the job and make sure there are no errors.

![](assets/20220623_073802_image.png)

Check in Flink that the job is running.

![](assets/20220623_073826_image.png)

# Step 4: Show the results on a map

Go to Data Viz via the webinterface of the web instance and click on ‘data’ to add a new connection.

1. Add new Data Connection

Basic:

Connection type: `Impala`

Connection name: `ssb_1`

Hostname or IP address: `YOUR HOST IP ADDRESS`

Advanced:

Connection mode: `Binary`

Socket type: `Normal`

Authentication mode: `NoSasl`

![](assets/20220623_092457_image.png)

![](assets/20220623_092553_image.png)

You need to get a Google API Id.

![](assets/20220623_093001_image.png)

Then add the key in DataViz under **Site Settings > Google API Id** and save the settings.

![](assets/20220623_092650_image.png)

You can now either create the map yourself or import a ready made dashboard:

[Import Fleet Control Dashboard](assets/20220623_092822_visuals_fleet-control.json)

Import the Dashboard by clicking on the ... under **Data** and then click on **Import Visual Artifacts**.

![](assets/20220627_081450_image.png)

Below is an example of what it will look like:

![](assets/20220623_093125_image.png)
