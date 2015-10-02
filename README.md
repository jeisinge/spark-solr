Lucidworks Spark/Solr Integration
========

Tools for reading data from Solr as a Spark RDD and indexing objects from Spark into Solr using SolrJ.

Features
========

* Send objects from a Spark Streaming application into Solr
* Read the results from a Solr query as a Spark RDD
* Read a large results set from Solr using distributed deep paging as a Spark RDD

Example Applications
========

First, build the Jar for this project:

`mvn clean package -DskipTests`

This will build 2 jars in the `/target` directory: spark-solr-1.0-SNAPSHOT.jar and spark-solr-1.0-SNAPSHOT-shaded.jar. 
The first is what you'd want to use if you were using spark-solr in your own project. The second is what you'd use to 
submit one of the included example apps to Spark.

Now, let's populate a SolrCloud index with tweets (be sure to update the command shown below with your Twitter API credentials):

Start Solr running in Cloud mode and create a collection named “socialdata” partitioned into two shards:

```
bin/solr -c && bin/solr create -c socialdata -shards 2
```

The remaining sections in this document assume Solr is running in cloud mode on port 8983 with embedded ZooKeeper listening on localhost:9983.

Also, to ensure you can see tweets as they are indexed in near real-time, you should enable auto soft-commits using Solr’s Config API.
Specifically, for this exercise, we’ll commit tweets every 2 seconds.

```
curl -XPOST http://localhost:8983/solr/socialdata/config \
 -d '{"set-property":{"updateHandler.autoSoftCommit.maxTime":"2000"}}'
```

Now, let’s populate Solr with tweets using Spark streaming:

```
$SPARK_HOME/bin/spark-submit --master $SPARK_MASTER \
 --conf "spark.executor.extraJavaOptions=-Dtwitter4j.oauth.consumerKey=? -Dtwitter4j.oauth.consumerSecret=? -Dtwitter4j.oauth.accessToken=? -Dtwitter4j.oauth.accessTokenSecret=?" \
 --class com.lucidworks.spark.SparkApp \
 ./target/spark-solr-1.0-SNAPSHOT-shaded.jar \
 twitter-to-solr -zkHost localhost:9983 -collection socialdata
```

Replace $SPARK_MASTER with the URL of your Spark master server. If you don’t have access to a Spark cluster, you can run the Spark job in local mode by passing:

```
--master local[2]
```

However, when running in local mode, there is no executor, so you’ll need to pass the Twitter credentials in the `spark.driver.extraJavaOptions` parameter instead of `spark.executor.extraJavaOptions`.
Tweets will start flowing into Solr; be sure to let the streaming job run for a few minutes to build up a few thousand tweets in your socialdata collection. You can kill the job using ctrl-C.

NOTE: If you don't have Twitter API credentials, please see:
https://databricks-training.s3.amazonaws.com/realtime-processing-with-spark-streaming.html#twitter-credential-setup

Working at the Spark Shell
========

Let’s start up the Spark Scala REPL shell to do some interactive data exploration with our indexed tweets:

```
cd $SPARK_HOME
ADD_JARS=$PROJECT_HOME/target/spark-solr-1.0-SNAPSHOT-shaded.jar bin/spark-shell
```

$PROJECT_HOME is the location where you cloned the spark-solr project. You should see a message like this from Spark during shell initialization:

```
15/05/27 10:07:53 INFO SparkContext: Added JAR file:/spark-solr/target/spark-solr-1.0-SNAPSHOT-shaded.jar at http://192.168.1.3:57936/jars/spark-solr-1.0-SNAPSHOT-shaded.jar with timestamp 1432742873044
```

Let’s load the socialdata collection into Spark by executing the following Scala code in the shell:

```
val tweets = sqlContext.load("solr",
 Map("zkHost" -> "localhost:9983", "collection" -> "socialdata")
 ).filter("provider_s='twitter'")
```

On line 1, we use the sqlContext object loaded into the shell automatically by Spark to load a DataSource named “solr”. Behind the scenes, Spark locates the solr.DefaultSource class in the project JAR file we added to the shell using the ADD_JARS environment variable.
On line 2, we pass configuration parameters needed by the Solr DataSource to connect to Solr using a Scala Map. At a minimum, we need to pass the ZooKeeper connection string (zkHost) and collection name. By default, the DataSource matches all documents in the collection, but you can pass a Solr query to the DataSource using the optional “query” parameter. This allows to you restrict the documents seen by the DataSource using a Solr query.
On line 3, we use a filter to only select documents that come from Twitter (provider_s=’twitter’).
At this point, we have a Spark SQL DataFrame object that can read tweets from Solr. In Spark, a DataFrame is a distributed collection of data organized into named columns (see: https://databricks.com/blog/2015/02/17/introducing-dataframes-in-spark-for-large-scale-data-science.html). Conceptually, DataFrames are similar to tables in a relational database except they are partitioned across multiple nodes in a Spark cluster.

It’s important to understand that Spark does not actually load the socialdata collection into memory at this point. We’re only setting up to perform some analysis on that data; the actual data isn’t loaded into Spark until it is needed to perform some calculation later in the job. This allows Spark to perform the necessary column and partition pruning operations to optimize data access into Solr.
Every DataFrame has a schema. You can use the printSchema() function to get information about the fields available for the tweets DataFrame:

```
tweets.printSchema()
```

Behind the scenes, our DataSource implementation uses Solr’s Schema API to determine the fields and field types for the collection automatically.

```
scala> tweets.printSchema()
 root
 |-- _indexed_at_tdt: timestamp (nullable = true)
 |-- _version_: long (nullable = true)
 |-- accessLevel_i: integer (nullable = true)
 |-- author_s: string (nullable = true)
 |-- createdAt_tdt: timestamp (nullable = true)
 |-- currentUserRetweetId_l: long (nullable = true)
 |-- favorited_b: boolean (nullable = true)
 |-- id: string (nullable = true)
 |-- id_l: long (nullable = true)
 ...
```

Next, let’s register the tweets DataFrame as a temp table so that we can use it in SQL queries:

```
tweets.registerTempTable("tweets")
```

For example, we can count the number of retweets by doing:

```
sqlContext.sql("SELECT COUNT(type_s) FROM tweets WHERE type_s='echo'").show()
```

If you check your Solr log, you’ll see the following query was generated by the Solr DataSource to process the SQL statement (note I added the newlines between parameters to make it easier to read the query):

```
 q=*:*&
 fq=provider_s:twitter&
 fq=type_s:echo&
 distrib=false&
 fl=type_s,provider_s&
 cursorMark=*&
 start=0&
 sort=id+asc&
 collection=socialdata&
 rows=1000
```

There are a couple of interesting aspects of this query. First, notice that the provider_s field filter we used when we declared the DataFrame translated into a Solr filter query parameter (fq=provider_s:twitter). Solr will cache an efficient data structure for this filter that can be reused across queries, which improves performance when reading data from Solr to Spark.
In addition, the SQL statement included a WHERE clause that also translated into an additional filter query (fq=type_s:echo). Our DataSource implementation handles the translation of SQL clauses to Solr specific query constructs. On the backend, Spark handles the distribution and optimization of the logical plan to execute a job that accesses data sources.
Even though there are many fields available for each tweet in our collection, Spark ensures that only the fields needed to satisfy the query are retrieved from the data source, which in this case is only type_s and provider_s. In general, it’s a good idea to only request the specific fields you need access to when reading data in Spark.
The query also uses deep-paging cursors to efficiently read documents deep into the result set. If you’re curious how deep paging cursors work in Solr, please read: https://lucidworks.com/blog/coming-soon-to-solr-efficient-cursor-based-iteration-of-large-result-sets/. Also, matching documents are streamed back from Solr, which improves performance because the client side (Spark task) does not have to wait for a full page of documents (1000) to be constructed on the Solr side before receiving data. In other words, documents are streamed back from Solr as soon as the first hit is identified.
The last interesting aspect of this query is the distrib=false parameter. Behind the scenes, the Solr DataSource will read data from all shards in a collection in parallel from different Spark tasks. In other words, if you have a collection with ten shards, then the Solr DataSource implementation will use 10 Spark tasks to read from each shard in parallel. The distrib=false parameter ensures that each shard will only execute the query locally instead of distributing it to other shards.
However, reading from all shards in parallel does not work for Top N type use cases where you need to read documents from Solr in ranked order across all shards. You can disable the parallelization feature by setting the parallel_shards parameter to false. When set to false, the Solr DataSource will execute a standard distributed query. Consequently, you should use caution when disabling this feature, especially when reading very large result sets from Solr.

Beyond SQL, the Spark API exposes a number of functional operations you can perform on a DataFrame. For example, if we wanted to determine the top authors based on the number of posts, we could use the following SQL:

```
sqlContext.sql("select author_s, COUNT(author_s) num_posts from tweets where type_s='post' group by author_s order by num_posts desc limit 10").show()
```

```
tweets.filter("type_s='post'").groupBy("author_s").count().orderBy(desc("count")).limit(10).show()
```

Another subtle aspect of working with DataFrames is that you as a developer need to decide when to cache the DataFrame based on how expensive it was to create it. For instance, if you load 10’s of millions of rows from Solr and then perform some costly transformation that trims your DataFrame down to 10,000 rows, then it would be wise to cache the smaller DataFrame so that you won’t have to re-read millions of rows from Solr again. On the other hand, caching the original millions of rows pulled from Solr is probably not very useful, as that will consume too much memory. The general advice I follow is to cache DataFrames when you need to reuse them for additional computation and they require some computation to generate.

Tuning the Solr SparkSQL DataSource
========

The Solr DataSource supports a number of optional parameters to allow you to optimize performance when reading data from Solr. Let's start with the most basic definition of the Solr DataSource and build up the options as we progress through this section:

```
var solr = sqlContext.read.format("solr").option("zkhost", "localhost:9983").option("collection","socialdata").load()
```

query
-------------

Probably the most obvious option is to specify a Solr query that limits the rows you want to load into Spark.
For instance, if we only wanted to load documents that mention "solr", we would do:

```
option("query","body_t:solr")
```

If you don't specify the "query" option, then all rows are read using the match all documents query (*:*).

fields
-------------

You can use the "fields" option to specify a subset of fields to retrieve for each document in your results:

```
option("fields","id,author_s,favorited_b,...")
```

By default, all fields for each document are pulled back from Solr.

rows
-------------

You can use the "rows" option to specify the number of rows to retrieve from Solr per request. Behind the scenes, the implemenation uses deep paging cursors and response streaming, so it is usually safe to specify a large number of rows. By default, the implementation uses 1000 but if your documents are smaller, you can increase this to 5000. Using too large a value can put pressure on the Solr JVM's garbage collector.

```
option("rows","5000")
```

split_field
-------------

If your Spark cluster has more available executor slots than the number of shards, then you can increase parallelism when reading from Solr by splitting each shard into sub ranges using a split field. A good candidate for the split field is the _version_ field that is attached to every document by the shard leader during indexing.

```
option("split_field","_version_")
```

Behind the scenes, the DataSource implementation tries to split the shard into evenly sized splits using filter queries. You can also split on a string-based keyword field but it should have sufficient variance in the values to allow for creating enough splits to be useful. In other words, if your Spark cluster can handle 10 splits per shard, but there are only 3 unique values in a keyword field, then you will only get 3 splits.

splits_per_shard
-------------

The "splits_per_shard" option provides a hint to the shard split planner on how many splits to create per shard. This should be based on the number of available executor slots in your Spark cluster divided by the number of shards in the collection you're querying. For instance, if you're querying into a 5 shard collection and your Spark cluster has 20 available executor slots to run the job, then you'll want to use:

```
option("splits_per_shard","4")
```

Keep in mind that this is only a hint to the split calculator and you may end up with a slightly different number of splits than what was requested.

parallel_shards
-------------

By default, each shard in your collection is read independently in different Spark tasks, which means your Spark job will not see rows in global sort order. This will not work for top N type queries where you need results sorted by Solr across all shards.
You can set the "parallel_shards" option to "false" to disable this feature to support top N style queries, however, that will reduce read performance in your Spark job, so use it wisely.


Reading data from Solr as a Spark RDD
========

The `com.lucidworks.spark.SolrRDD` class transforms the results of a Solr query into a Spark RDD.

```
SolrRDD solrRDD = new SolrRDD(zkHost, collection);
JavaRDD<SolrDocument> solrJavaRDD = solrRDD.queryShards(jsc, solrQuery);
```

Once you've converted the results in an RDD, you can use the Spark API to perform analytics against the data from Solr. For instance, the following code extracts terms from the tweet_s field of each document in the results:

```
JavaRDD<String> words = solrJavaRDD.flatMap(new FlatMapFunction<SolrDocument, String>() {
  public Iterable<String> call(SolrDocument doc) {
    Object tweet_s = doc.get("tweet_s");
    String str = tweet_s != null ? tweet_s.toString() : "";
    str = str.toLowerCase().replaceAll("[.,!?\n]", " ");
    return Arrays.asList(str.split(" "));
  }
});
```

Writing data to Solr from Spark Streaming
========

The `com.lucidworks.spark.SolrSupport` class provides static helper functions for send data to Solr from a Spark
 streaming application. The TwitterToSolrStreamProcessor class provides a good example of how to use the SolrSupport
 API. For sending documents directly to Solr, you need to build-up a `SolrInputDocument` in your
 Spark streaming application code. 

```
    String zkHost = cli.getOptionValue("zkHost", "localhost:9983");
    String collection = cli.getOptionValue("collection", "collection1");
    int batchSize = Integer.parseInt(cli.getOptionValue("batchSize", "10"));
    SolrSupport.indexDStreamOfDocs(zkHost, collection, batchSize, docs);
```

Developing a Spark Application
========

The `com.lucidworks.spark.SparkApp` provides a simple framework for implementing Spark applications in Java. The
class saves you from having to duplicate boilerplate code needed to run a Spark application, giving you more time to
focus on the business logic of your application. To leverage this framework, you need to develop a concrete class
that either implements RDDProcessor or extends StreamProcessor depending on the type of application you're developing.

RDDProcessor
-------------

Implement the `com.lucidworks.spark.SparkApp$RDDProcessor` interface for building a Spark application that operates
 on a JavaRDD, such as one pulled from a Solr query (see: SolrQueryProcessor as an example)

StreamProcessor
-------------

Extend the `com.lucidworks.spark.SparkApp$StreamProcessor` abstract class to build a Spark streaming application.
See com.lucidworks.spark.example.streaming.oneusagov.OneUsaGovStreamProcessor or
com.lucidworks.spark.example.streaming.TwitterToSolrStreamProcessor for examples of how to write a StreamProcessor.


Authenticating with Kerberized Solr
========

For background on Solr security, see: https://cwiki.apache.org/confluence/display/solr/Security

The SparkApp framework allows you to pass the path to a JAAS authentication configuration file using the -solrJaasAuthConfig option.

For example, if you need to authenticate using the "solr" Kerberos principal, you need to create a JAAS config file that sets the location of your Kerberos keytab file, such as:

```
Client {
  com.sun.security.auth.module.Krb5LoginModule required
  useKeyTab=true
  keyTab="/opt/lucidworks-hdpsearch/job/solr.keytab"
  storeKey=true
  useTicketCache=true
  debug=true
  principal="solr";
};
```

To use this configuration to authenticate to Solr, you simply need to pass the path using the -solrJaasAuthConfig option, such as:

```
spark-submit --master yarn-server \
  --class com.lucidworks.spark.SparkApp \
  $SPARK_SOLR_PROJECT/target/lucidworks-spark-rdd-2.0.3.jar \
  hdfs-to-solr -zkHost $ZK -collection spark-hdfs \
  -hdfsPath /user/spark/testdata/syn_sample_50k \
  -solrJaasAuthConfig=/opt/lucidworks-hdpsearch/job/jaas-client.conf
```

