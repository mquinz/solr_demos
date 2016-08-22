# Using Solr with Cassandra

## Overview of DSE Search

DSE Search (Solr + Cassandra) is a beneficial pairing of the indexing and searching, and faceting benefits of Solr with the scalabiity and reliablity of Cassandra.   In DSE search, all data is stored in Cassandra while the indices are maintained by Solr/Lucene.
Solr queries sent to DSE Search - either via HTTP(s) or via CQL - are evaluated by the Solr/Lucene libraries and the primary keys of the result set are passed to Cassandra in order to obtain the data to be included in the result set.

![integration](./resources/integration_2.png)

### Additional resources
A free [self-paced tutorial on DSE Search](https://academy.datastax.com/courses/ds310-datastax-enterprise-search) is available from the DataStax Academy.  It includes slides and a self-contained Virtual Machine for exercises.


## Benefits

### No ETL

In typical Open Source Solr (OSS), indices run the risk of being stale because of the separation of the database and the Solr indices. There can be a significant amount of latency between the time that data is added to the database and the time that it is added to Solr.  And as new data is added to the database or existing records are removed, separate processes are usually required to keep the indices in-sync with the database.  However, due to the tight integration between Solr and Cassandra, the danger of serving stale data is mitigated because there is

Any changes to a Cassandra data record in a Solr-backed Cassandra table will automatically trigger an indexing event where the record is re-indexed using the Solr libraries.  Usually these records are discoverable (indexed) in search queries within a few seconds (configurable).  

![Alt text](./resources/integration.png)

### Reduced storage costs and fewer tables

Cassandra tables are always created with a partition key / clustering key combination designed to solve a particular query.  A slightly different query against the same data might require duplicating the data in a nearly identical table, varying only by the structure of the primary key.  While there are obviously costs of storing this data redundantly, the additional amount of effort required to keep the multiple copies in sync cannot be ignored.

By creating a Solr core using data from a single table, many different queries can be utilized - regardless of whether or not they use the primary key as part of the WHERE clause.   For example, let's start with a simple table to store music tracks (songs) where the query is is to find all tracks for a given album.

```javascript

CREATE TABLE music.tracks_by_album (
    album_title text,
    album_year int,
    track_number int,
    album_genre text static,
    performer text static,
    track_title text,
    PRIMARY KEY ((album_title, album_year), track_number)
)

```
 If we wanted to query this table by artist, we would have to create a nearly identical table that used artist as part of the primary key.


 ```javascript

 CREATE TABLE music.tracks_by_artist(
     album_title text,
     album_year int,
     track_number int,
     album_genre text static,
     performer text static,
     track_title text,
     PRIMARY KEY (performer, album_title, album_year, track_number)
 )

 ```
And when asked to allow queries by genre, we would create yet another nearly identical table.
If we wanted to query this table by artist, we would have to create a nearly identical table that used artist as part of the primary key.


```javascript

CREATE TABLE music.tracks_by_genre(
    album_title text,
    album_year int,
    track_number int,
    album_genre text static,
    performer text static,
    track_title text,
    PRIMARY KEY (album_genre, album_title, album_year, track_number)
)

```
By creating a single 'tracks' table and pairing it with a Solr core, we can eliminate the need for duplicate tables while greatly increasing the flexibility we have for querying data.   We will use this table to illustrate other


```javascript

CREATE TABLE music.tracks (
    album_title text,
    album_year int,
    track_number int,
    album_genre text ,
    performer text ,
    track_title text,
    PRIMARY KEY ((album_title, album_year), track_number)
)

```

### Flexibility via CQL / Solr Integration

While the Solr engine can be accessed via HTTP(s) for regular REST/JSON API calls from SolrJ and/or other clients, solr queries can be integrated with Cassandra CQL queries.  This is a huge benefit because it simplifies & standardizes the process for handling C* result sets.  


![Alt text](./resources/dse_search_ref.png)

Solr query syntax can be used in the CQL *WHERE* clause.  At first glance this appears to merely be an ideal way to add flexibility to Cassandra queries because it does not require the partition key.  However, It also allows the full power of Solr to be utilized for queries without  having to use a separate Solr HTTP/REST API because the Solr integration is handled by Cassandra and the Cassandra CQL driver.  This eliminates the need for a separate code base for handling persistence, serialization, queries, and exception handling.   

For example, the following query will return all rows for a given artist using a non-key search field.
```javascript

CQL> select * from music.tracks where solr_query='{"q":"performer:Elvis Presley"}' limit 5;

```

### Pagination

Classic CQL queries do not allow an offset parameter to specify the starting row for result sets. This makes it difficult to provide paging capabilities in the API.   However, Solr does allow a start parameter to make this task much simpler.

```javascript
CQL> Select * from ... WHERE solr_query={..., start:100} and LIMIT = 20;
```


### Reduced Data Storage and Increased Reusability

It is not unusual for nearly identical tables to be created in Cassandra where each of the tables have slightly different primary key columns.  For example, the rx_by_npi table and the rx_by_npi_patient tables contain identical data columns, but slightly different keys because they each support different queries.  By using a single table indexed by a Solr core, both queries (and many others) can be made.   

#### Comparing Queries

This is a classic CQL query to obtain all rx records for a given NPI for a given date range.

```javascript
CQL> select * from esidata.rx_by_npi where npi_nbr=12345 AND  rx_received_dte > 20140115 AND rx_received_dte < 20140619];

```

Here is a hybrid CQL/Solr query to obtain all rx records for a given NPI for a given date range for a given patient. This query works even though patient_id is *NOT* part of the primary key.

```javascript
CQL> select * from esidata.rx_by_patient_id where solr_query='{"q":"*:*", "fq":["npi_nbr:10", "specialty_patient_id:5", "rx_received_dte:[  20140115 TO 20150619]"]}' limit 5;
```


## Create the Cassandra Table

```javascript

CREATE TABLE music.tracks (
    album_title text,
    album_year int,
    track_number int,
    album_genre text ,
    performer text ,
    track_title text,
    PRIMARY KEY ((album_title, album_year), track_number)
) WITH CLUSTERING ORDER BY (track_number ASC)

```

### Insert some test records

```javascript

insert into music.tracks (album_title, album_year, album_genre, performer, track_number,track_title) values ('Birth of the Cool',1954,'jazz','Miles Davis',1,'Jeru');  
insert into music.tracks (album_title, album_year, album_genre, performer, track_number,track_title) values ('Birth of the Cool',1954,'jazz','Miles Davis',2,'Move');  
insert into music.tracks (album_title, album_year, album_genre, performer, track_number,track_title) values ('Birth of the Cool',1954,'jazz','Miles Davis',3,'Godchild');  

insert into music.tracks (album_title, album_year, album_genre, performer, track_number,track_title) values ('Coltrane',1957,'jazz','John Coltrane',1,'Bakai');  
insert into music.tracks (album_title, album_year, album_genre, performer, track_number,track_title) values ('Coltrane',1957,'jazz','John Coltrane',2,'Violets for your furs');  

insert into music.tracks (album_title, album_year, album_genre, performer, track_number,track_title) values ('Tommy',1969,'classic rock','The Who',1,'Overture');  
insert into music.tracks (album_title, album_year, album_genre, performer, track_number,track_title) values ('Tommy',1969,'classic rock','The Who',15,'Go to the Mirror');  

insert into music.tracks (album_title, album_year, album_genre, performer, track_number,track_title) values ('Kind of Blue',1959,'jazz','Miles Davis',1,'So What');  
insert into music.tracks (album_title, album_year, album_genre, performer, track_number,track_title) values ('Kind of Blue',1959,'jazz','Miles Davis',2,'Freddy Freeloader');  
insert into music.tracks (album_title, album_year, album_genre, performer, track_number,track_title) values ('Kind of Blue',1959,'jazz','Miles Davis',3,'Blue in Green');  


```

### Verify

```javascript

cqlsh> select * from music.tracks;

 album_title       | album_year | track_number | album_genre  | performer     | solr_query | track_title
-------------------+------------+--------------+--------------+---------------+------------+-----------------------
          Coltrane |       1957 |            1 |         jazz | John Coltrane |       null |                 Bakai
          Coltrane |       1957 |            2 |         jazz | John Coltrane |       null | Violets for your furs
             Tommy |       1969 |            1 | classic rock |       The Who |       null |              Overture
             Tommy |       1969 |           15 | classic rock |       The Who |       null |      Go to the Mirror
      Kind of Blue |       1959 |            1 |         jazz |   Miles Davis |       null |               So What
      Kind of Blue |       1959 |            2 |         jazz |   Miles Davis |       null |     Freddy Freeloader
      Kind of Blue |       1959 |            3 |         jazz |   Miles Davis |       null |         Blue in Green
 Birth of the Cool |       1954 |            1 |         jazz |   Miles Davis |       null |                  Jeru
 Birth of the Cool |       1954 |            2 |         jazz |   Miles Davis |       null |                  Move
 Birth of the Cool |       1954 |            3 |         jazz |   Miles Davis |       null |              Godchild

(10 rows)

```
## Creating A Solr Core

DSE simplifies the process of creating a Solr core. A core generally consists of 2 primary resource files and 0-n supplemental resource files.  The dse create_core command will generate generic versions of the schema.xml file and the solrconfig.xml file.  Any other, supplemental files such as stopwords or dictionaries, must be manually created & uploaded.  Instructions for this can be found [here](https://docs.datastax.com/en/datastax_enterprise/4.8/datastax_enterprise/srch/srchUpld.html)

```javascript
dsetool create_core music.tracks generateResources=true reindex=true

```

### Sample schema.xml file

```javascript

<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<schema name="autoSolrSchema" version="1.5">
  <types>
  	<fieldType class="org.apache.solr.schema.TextField" name="TextField">
  	<analyzer>
  		<tokenizer class="solr.StandardTokenizerFactory"/>
  		<filter class="solr.LowerCaseFilterFactory"/>
  		</analyzer>
  	</fieldType>
  	<fieldType class="org.apache.solr.schema.TrieIntField" name="TrieIntField"/>
  	<fieldType class="org.apache.solr.schema.StrField" name="StrField"/>
  </types>
  <fields>
  	<field indexed="true" multiValued="false" name="album_genre" stored="true" type="TextField"/>
  	<field indexed="true" multiValued="false" name="track_title" stored="true" type="TextField"/>
  	<field indexed="true" multiValued="false" name="album_year" stored="true" type="TrieIntField"/>
  	<field indexed="true" multiValued="false" name="performer" stored="true" type="TextField"/>
  	<field indexed="true" multiValued="false" name="track_number" stored="true" type="TrieIntField"/>
  	<field indexed="true" multiValued="false" name="album_title" stored="true" type="StrField"/>
  </fields>
  <uniqueKey>(album_title,album_year,track_number)</uniqueKey>
</schema>

```
## CQL Queries Using Solr index

The following query will fail because the album_genre is not part of the primary key.


```javascript

select * from music.tracks where album_genre = 'classic rock' limit 5;

```
However, the tight integration of CQL with Solr allows the query to work

```javascript

cqlsh> select * from music.tracks where solr_query='{"q":"album_genre:\"classic rock\""}' limit 5;

 album_title | album_year | track_number | album_genre  | performer | solr_query | track_title
-------------+------------+--------------+--------------+-----------+------------+------------------
       Tommy |       1969 |            1 | classic rock |   The Who |       null |         Overture
       Tommy |       1969 |           15 | classic rock |   The Who |       null | Go to the Mirror

(2 rows)

```
The following query uses the filter query (fq) syntax to utilize Solr's filter cache. It also demonstrates using multiple attributes in a query.

```javascript

cqlsh> select * from music.tracks where  solr_query='{"q":"*:*", "fq":["album_genre:jazz", "performer:\"Miles Davis\""]}' limit 5;

 album_title       | album_year | track_number | album_genre | performer   | solr_query | track_title
-------------------+------------+--------------+-------------+-------------+------------+-------------------
 Birth of the Cool |       1954 |            2 |        jazz | Miles Davis |       null |              Move
      Kind of Blue |       1959 |            1 |        jazz | Miles Davis |       null |           So What
 Birth of the Cool |       1954 |            1 |        jazz | Miles Davis |       null |              Jeru
 Birth of the Cool |       1954 |            3 |        jazz | Miles Davis |       null |          Godchild
      Kind of Blue |       1959 |            2 |        jazz | Miles Davis |       null | Freddy Freeloader

(5 rows)

```

```javascript
cqlsh> select * from music.tracks where solr_query='{"q":"*:*", "fq":"album_year:[ 1954 TO 1995]"}' limit 5;

 album_title       | album_year | track_number | album_genre  | performer     | solr_query | track_title
-------------------+------------+--------------+--------------+---------------+------------+-----------------------
 Birth of the Cool |       1954 |            2 |         jazz |   Miles Davis |       null |                  Move
          Coltrane |       1957 |            2 |         jazz | John Coltrane |       null | Violets for your furs
             Tommy |       1969 |            1 | classic rock |       The Who |       null |              Overture
      Kind of Blue |       1959 |            1 |         jazz |   Miles Davis |       null |               So What
 Birth of the Cool |       1954 |            1 |         jazz |   Miles Davis |       null |                  Jeru

(5 rows)
```

## Bonus - Fast & Accurate counts

Almost every new Cassandra user attempts to count the records in a Cassandra table using a query similar to the following - after increasing the limit - and then waits for a timeout.


```javascript

cqlsh> SELECT count(*) FROM music.tracks limit 1000000 ;

```

But Cassandra is able to utilize the Solr indices to provide extremely fast and accurate counts.

```javascript

cqlsh> SELECT count(*) FROM music.tracks WHERE solr_query = '{"q":"*:*"}' ;

 count
-------
    10

(1 rows)
```

If simple counts are not enough, try using facets.


### Faceting

Solr facets are an extremely fast and simple way to obtain basic counts and/or histograms from result sets.  

 The following CQL query will return a map of histograms in the form of a <k,v> map where the map keys are column data, and the map values are counts    This is similar to running an SQL count() - groupby() query.


```javascript

cqlsh> select * from music.tracks where solr_query='{"q":"*:*","facet":{"field":["album_genre","album_year"]} }';

 facet_fields
----------------------------------------------------------------------------------------------------
 {"album_genre":{"jazz":8,"classic":2,"rock":2},"album_year":{"1954":3,"1959":3,"1957":2,"1969":2}}

(1 rows)

```

## Tuning A Solr Core

The generic schema.xml file is perfectly fine for a POC, but it should be tweaked and tuned before going into production.  This tuning process generally involves evaluating each field to determine the optimal values for the data type and whether or not the field is to be indexed.   


### Overview

[solr_core.pdf](./resources/solr_core.pdf)


### More details about creating and maintaining a Solr Core

[CQLSH Copy Documentation](hhttp://docs.datastax.com/en/cql/3.1/cql/cql_reference/copy_r.html)


### A simple overview of the entire process is available here
[Excellent DSE Search Tutorial](https://datastax.jira.com/wiki/display/~caroline.george/Solr+Tutorial+-+Getting+Started+with+DSE+Search)
