---
layout: post
title: Spark Streaming Microbatch Metrics, Programmatically via the REST API
tags: distributed-systems 
---
*TLDR; [metric collection script](https://github.com/lenhattan86/ccra/blob/master/flintrock-setup/get_spark_streaming_batch_statistics.py).*
 
The Spark Streaming web UI shows a number of interesting metrics over time.
[Tan](http://www3.cs.stonybrook.edu/~tnle/) and I were specifically interested
in the (micro)batch start times, processing times and scheduling delays, which we
could find no reported way of obtaining programmatically. We were running Spark
2.0.0 on YARN 2.7.2 in cluster-mode.

All I could find with was this [StackOverflow question](http://stackoverflow.com/questions/34507578/how-to-fetch-spark-streaming-job-statistics-using-rest-calls-when-running-in-yar)
that suggested scraping the Spark UI webpage (as, horribly, [some have done](https://github.com/amitsing89/pythonscripts/blob/master/sparkMonitoring.py))
or hitting the JSON API endpoint at `/api/v1/`. This endpoint, unfortunately again,
does not provide the same metrics we see on the Spark Streaming web UI.

Not directly.

It turns out that you can use the `/jobs/` endpoint to reconstruct the metrics you
see on the Spark Streaming web UI: the batch start time, processing delay and
scheduling delay. The key to how this reconstruction is done lies in the
[BatchInfo class definition](https://github.com/apache/spark/blob/d6dc12ef0146ae409834c78737c116050961f350/streaming/src/main/scala/org/apache/spark/streaming/scheduler/BatchInfo.scala) 
in the Spark codebase.

I wrote a [script](https://github.com/lenhattan86/ccra/blob/master/flintrock-setup/get_spark_streaming_batch_statistics.py)
that parses the JSON from this endpoint and reconstructs these metrics, given the
application ID (the one YARN generates for you on submission) and the YARN master
URL. All times are in seconds. A sample execution is:

```bash
python get_spark_streaming_batch_statistics.py \
  --master ec2-52-40-144-150.us-west-2.compute.amazonaws.com \
  --applicationId application_1469205272660_0006
```

Sample output (batch start-time, processing time, scheduling delay):

```
  18:36:55 3.991 3783.837
  18:36:56 4.001 3786.832
  18:36:57 3.949 3789.862
  ...
```

The script creates a map from each batch to its start time and all the jobs it
contains, along with the jobs' start and completion times. Simple arithmetic then
generates the required metrics. It is easy to modify the script to print the
actual timestamps instead of the delay, if one wishes to.

Do file an [issue](https://github.com/lenhattan86/ccra/issues) with any
questions or contributions you have.
