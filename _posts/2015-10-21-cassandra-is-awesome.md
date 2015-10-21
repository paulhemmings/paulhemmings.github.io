---
title: Datastax Flight Exercise
layout: post
---

### Flight exercise.

##### Install the stack.

This requires registration, and is free for non-commercial use. I downloaded the version that ran on OSX. During installation you are given the option to create a node for analytics, search or both. I chose both, as this was going to be a single node in a single cluster, and therefore wouldn't use up too many resources.

http://www.datastax.com/download

#### Install the Dev Center.

This desktop application allowed me to create and execute CQL against the Cassandra instance. It also provided a rich UI showing the current key spaces within the node.

https://academy.datastax.com/downloads/ops-center#devCenter

This was very useful later when I needed to generate the "flights" table in the single node cluster.


#### Upload CSV into Cassandra.

The first task is to upload the flight data into Cassandra. To do this I had two options.

 * Create an ETL application
 * Make use of existing tools provided within the Datastax enterprise stack.

I chose to make use of the existing tool, Sqoop. I could then always return to this task and create an application if time permitted.

#### Using Sqoop

Sqoop is pre-installed with Datastax Enterprise. There is also helpful documentation covering a demo that shows how to use Sqoop to import a CSV file into Cassandra, via a MySQL database.

http://docs.datastax.com/en/datastax_enterprise/4.8//datastax_enterprise/ana/anaSqpDemo.html

I already had an instance of MySQL up and running so the only task required was to download the MySQL Java JDBC driver required to allow it to interact with Sqoop. This driver is provided by Oracle, so yes - you'll need to register.

http://dev.mysql.com/downloads/connector/j/

This will supply you with a JAR file named: mysql-connector-java-5.1.37-bin.jar

Next step is to copy this into the lib folder of Sqoop, which on my machine was:

/Users/paulhemmings/dse/resources/sqoop/lib

#### Create the new schema/database in MySQL

Now all I needed was to initialize my MySQL instance to accept the CSV data. For this I would need to create:

* A new schema
* A new user with access to the table

Here is some example SQL which performs these tasks

````
mysql> CREATE SCHEMA `flights` DEFAULT CHARACTER SET utf8 COLLATE utf8_bin ;
mysql> CREATE USER 'flights'@'localhost' IDENTIFIED BY 'flights';
mysql> GRANT ALL ON flights.* TO 'flights'@'localhost';
mysql> flush privileges;
````

#### Create the flights table in the new schema.

This table needs to match the CSV data. As we have been provided with the CQL to generate the Cassandra table, this can be re-used.

note:: MySQL requires sized for the varchar. Set to 1024 for now. Can always convert to text if need be

````
CREATE TABLE flights (
ID int PRIMARY KEY,
YEAR int,
DAY_OF_MONTH int,
FL_DATE timestamp,
AIRLINE_ID int,
CARRIER varchar(1024),
FL_NUM int,
ORIGIN_AIRPORT_ID int,
ORIGIN varchar(1024),
ORIGIN_CITY_NAME varchar(1024),
ORIGIN_STATE_ABR varchar(1024),
DEST varchar(1024),
DEST_CITY_NAME varchar(1024),
DEST_STATE_ABR varchar(1024),
DEP_TIME int,
ARR_TIME int,
ACTUAL_ELAPSED_TIME int,
AIR_TIME int,
DISTANCE int)
````

#### Create the flights keyspace, and flights table in Cassandra

Here is the CQL I used to create a new keyspace called "flights"

````
CREATE KEYSPACE flights
WITH replication = {
	'class' : 'SimpleStrategy',
	'replication_factor' : 1
};
````

Here is the CQL to generate the Cassandra table. Notice that this schema sets the DEP_TIME to ARR_TIME columns to be of data type INT. The data types for these columns had to match those within MySQL table, otherwise the import would fail.

````
CREATE TABLE flightsNoTimestamps (
ID int PRIMARY KEY,
YEAR int,
DAY_OF_MONTH int,
FL_DATE timestamp,
AIRLINE_ID int,
CARRIER varchar,
FL_NUM int,
ORIGIN_AIRPORT_ID int,
ORIGIN varchar,
ORIGIN_CITY_NAME varchar,
ORIGIN_STATE_ABR varchar,
DEST varchar,
DEST_CITY_NAME varchar,
DEST_STATE_ABR varchar,
DEP_TIME int,
ARR_TIME int,
ACTUAL_ELAPSED_TIME int,
AIR_TIME int,
DISTANCE int);
````

#### Import the CSV into MySQL

The SQL to load the CSV file into the new table was helpfully provided in the Sqoop demo documentation.

````
LOAD DATA LOCAL INFILE '[path]/flights_from_pg.csv'
         INTO TABLE flights.flights
         FIELDS TERMINATED BY ','
         ENCLOSED BY '"'
         LINES TERMINATED BY '\n';
````

Rather surprisingly, this worked first time. Almost. You'll notice that the DEP_TIME to AIR_TIME columns in the "flights" table have been created as INT data types, instead of timestamp, as defined in the requirements. This was because several columns within the CSV failed to import into MySQL when the columns were of that type. I reviewed the data in the raw CSV file, and made the change to the column data types accordingly, to ensure all data would be loaded successfully.

And here's the output of the successful load

````
1048576 row(s) affected Records:
1048576  Deleted: 0
Skipped: 0
Warnings: 0	10.058 sec
````

#### Configure Sqoop to load the data from MySQL.

Now we need to configure Sqoop to load data from MySQL into Cassandra. For this we need to tell it the source schema and table, as well as the destination keyspace and table. The options also need to describe the mappings for each column.

NOTE:: Cassandra columns are lower case.

The configuration is held within an *import.options* file.

````
cql-import
--table
flights
--cassandra-keyspace
flights
--cassandra-table
flightsNoTimestamps
--cassandra-column-mapping
id:ID,year:YEAR,day_of_month:DAY_OF_MONTH,fl_date:FL_DATE,airline_id:AIRLINE_ID,carrier:CARRIER,fl_num:FL_NUM,origin_airport_id:ORIGIN_AIRPORT_ID,origin:ORIGIN,origin_city_name:ORIGIN_CITY_NAME,origin_state_abr:ORIGIN_STATE_ABR,dest:DEST,dest_city_name:DEST_CITY_NAME,dest_state_abr:DEST_STATE_ABR,dep_time:DEP_TIME,arr_time:ARR_TIME,actual_elapsed_time:ACTUAL_ELAPSED_TIME,air_time:AIR_TIME,distance:DISTANCE
--connect
jdbc:mysql://127.0.0.1/flights
--username
flights
--password
flights
--cassandra-host
127.0.0.1
````

### Execute Sqoop

Run the sqoop using the import.options just created.

````
$ cd /Users/paulhemmings/dse/bin
$ ./dse sqoop --options-file /Users/paulhemmings/dse/demos/sqoop/import.options
````

#### Check there's data in that table.

Sqoop completed without any notification to say if the import was successful. I made a quick check by running a CQL statement within DevCenter:

````SELECT COUNT(*) FROM flightsnotimestamps;````

It worked first time. Interestingly enough the DevCenter imposes a limit on how many items it can return. This includes the value of the count. So even though there were just over a million rows in the table, the count would not return a number greater than this limit. Later I find a way to get a true count.

#### Analysis time.

The exercise required some stats to be extracted from the flight data. As I'm far more proficient in SQL than I am CQL I thought I'd get these values from my MySQL instance first. I could then use these to double check any values I retrieved from CQL.

Q How many records were loaded?

A 1048576

````select count(*) from flights````

Q. List all flights leaving a particular airport, sorted by time.

A.

````select * from flights where ORIGIN = 'GGG' order by fl_date, dep_time````

Q. How many flights originated from the ‘HNL’ airport code on 2012-01-25?

A. 288

````select count(*) from flights where ORIGIN = "HNL" and fl_date = '2012-01-25'````

Q. How many airport codes start with the letter ‘A’?

A. 22 (over complicated I'm sure but wanted to make sure there were no airports in ORIGIN that were not in DEST, or vice versa)

````
select a.airport, b.airport
from
(
 select distinct(ORIGIN) as airport
 from flights
 where ORIGIN like 'A%'
) a
left outer
join
(
 select distinct(DEST) as airport
 from flights
 where DEST like 'A%'
) b
on a.airport = b.airport
````

Q. What originating airport had the most flights on 2012-01-23?

A. ATL, with 2155 flights

````
select ORIGIN, count(*)
from flights
where FL_DATE = '2012-01-23'
group
by ORIGIN
order
by count(*) desc
````

#### Analysis using Cassandra.

The exercise asks for the stats to be retrieved using either Analystics (DSE Spark, Hadoop), or Search (DSE SOLR, Native SOLR)

##### Spark

Here's a link to the official Datastax documentation for Spark.

http://docs.datastax.com/en/datastax_enterprise/4.8//datastax_enterprise/spark/sparkSCcontext.html

Here's how to start Spark on your local machine.

````
$ cd /Users/paulhemmings/dse/bin
$ ./dse spark
````

Here's some example queries

````
scala> :showSchema flights flightsnotimestamps
scala> sc.cassandraTable("flights", "flights").select("id").where("ORIGIN = ?", "hnl")
````

And here is how I fwas able to get an accurate count of the rows in my Cassandra table.

````scala> csc.sql("SELECT origin from flights.flightsNoTimestamps").collect().length````
