---
layout: page 
title: Yahoo!
permalink: /yahoo/
---

![Timesense Team](/images/timesense-paintball.png)

I was a primary member of the small but awesome *Timesense* engineering team. We built systems to rapidly detect real-world events, like earthquakes and celebrity deaths,
from large, noisy streams of user-generated content.

As a full-time engineer *(2012 - 2013)*, I helped build a new system that improved our earlier event-detection latency by 6x. I worked with streaming/low-latency technologies like [Storm](http://storm-project.net), [Kafka](http://kafka.apache.org), and [HBase](http://hbase.apache.org/).

I also led the organization of a science and math talk series. I gave a talk myself, [on random forests](https://speakerdeck.com/emaadmanzoor/reviving-failed-classifiers-with-random-forests).

More memorably, I worked on numerous blue-sky prototypes during Yahoo!'s quarterly Hack Days. I still miss the adrenaline, and the night I dozed off in a conference room.

![Timesense Hack](/images/timesense-hack.png)

As in intern *(Fall 2011)*, I extended the trend detection system to be multithreaded and centrally configurable via a simple XML file. This cut down the deployment time of new locales from impossible to a few minutes.

I also implemented a research prototype to detect geographically niche search trends. It was basically an implementation of [this KDD '12 paper](http://dl.acm.org/citation.cfm?id=2339594) by our science-team counterparts.

I was one of the 3 interns (out of 14 from my university) to be offered a full-time position.

Some resources describing our early work back in 2010, using language
models to classify trending queries:

   * [Towards recency ranking in web search](http://www.wsdm-conference.org/2010/proceedings/docs/p11.pdf). WSDM 2010.
   * [Yahoo! Real Time Search](http://www.slideshare.net/davtchev/yahoo-real-time-search-smx-march-2010-3320691). SMX, 2010.

We also expose the (currently very limited) Timesense API via
[YQL](http://developer.yahoo.com/yql/console/#h=select+*+from+timesense.trending+where+locale%3D'en-US').

On a related note, [Pranay Agarwal](http://www.cse.iitd.ac.in/~cs5080220/index.html)
collaborated with us on some very interesting approaches to Twitter
trend prediction for his [masters thesis](http://www.cse.iitd.ac.in/~cs5080220/TwitterNews.html). Check him out!
