---
layout: post
title: "NoSQL: Not only a fairy tale"
comments: true
published: true
categories: nosql redis riak couchdb
author: Sebastian Cohnen
---

*This is a cross-post from Sebastian Cohnens [personal blog](http://tisba.de/2012/06/28/nosql-not-only-a-fairy-tale/).*

At this year's [NoSQL Matters](http://www.nosql-matters.org/) conference in Cologne, which was also the first one of its kind, I held a talk together with [Timo Derstappen](https://twitter.com/teemow) about the evolution of [adcloud](http://adcloud.com)'s adserver. We took the audience on a journey spanning multiple years of changing requirements and the resulting architectural decisions made during that time. A focus was on NoSQL databases involved in the adserver, of course :)

<!--more-->

Before I start, I'd like to quickly thank the organizers of the NoSQL Matters conference. It was a very interesting meetup with awesome speakers, great catering and nice location.

The reminder of this post is a transcript of the talk. Be warned, that this post is lengthy and quite detailed :) You'll also find the slides at the end or directly at [speakerdeck](https://speakerdeck.com/u/tisba/p/nosql-not-only-a-fairy-tale). The talk was also recorded but the video is not available yet.

**Disclaimer**: I'm not an employee of adcloud, I just work for them. If I say *we* in this article, I am referring to Timo and me or the adcloud dev team (depending on the context).

## Preface
Before we can dive right into our story we are about to tell you, we first have to explain some terms related to the stuff we are doing at adcloud. At adcloud we are doing [performance advertising](http://en.wikipedia.org/wiki/Performance-based_advertising). To simplify things for this talk: We are serving ads.

We have two important data entities:

{% img right /images/posts/nosql-fairy-tale/website-placement.png %}

* **placements** are container on websites that can display ads to the user. A placement knows its size, style and the amount of ads that it holds.
* one, two or more **ads** can be displayed within a placement. An ad consists of some text, an image and a link.
* there is an n:m relationship between placements and ads, describing which ads can run on what placements. An attribute of this relationship is what we call **ad priority** or just priority. The priority specifies how likely it is for an ad to be displayed on a given placement.

**Placements** and **ads** are rather static data, whereas the **priority** changes all the time in order to optimize the performance.

To follow this talk you need to know that there are two distinct systems at adcloud (there are more actually, but they can be ignored for now): **Platform** and **adserver**.

* The **Platform** is a typical web application working primarily as the administrative back office. It is the place where all the business related data is managed, reports are generated, etc. Beside the traditional web stack, there are also tons of background workers to process and analyse data. There is only very little NoSQL here.
* The **adserver** on the other hand is a dedicated and decoupled system, which serves ads and collects tracking data from the users (like clicks on ads). NoSQL databases are heavily used within the adserver.

{% img center /images/posts/nosql-fairy-tale/system-context.png %}

These systems are highly decoupled. The Platform *publishes* placement, ad and priority data to the adservers. The adserver reports statistical and tracking data back to the Platform for analysis.

Since only the adserver heavily uses NoSQL systems, we will focus on it in this talk.



## AWS S3: Simple Storage Service
The first version of the "adserver" wasn't actually a server at all. We simply used S3. We combined all placement and ad data, including priorities, together in one document, added some JavaScript and pushed everything to S3. This was the initial version of the publishing process.

When a user visited a website, we delivered the JavaScript with the embedded data via a [CDN](http://www.akamai.com/cotendo) (fed by S3) and executed it on the client side. The JavaScript contained some logic to choose from the available ads, based on the ads' priorities; and in the end, it rendered the ads.

The problem with this simple approach was that, due to the heavy denormalization, the publishing process was rather expensive. As we already mentioned, placements and ads don't change, but priorities change all the time. With our denormalized approach, we couldn't update our data incrementally.

Eventually, the time required to fully publish everything was taking too long, and we need to publish the calculated priorities as quickly as possible to maximize the yield.



## The relaxed Knight

{% img right /images/posts/nosql-fairy-tale/couchdb-logo.png %}

In 2009, we took a closer look at CouchDB. We were pretty excited about the idea of using HTTP/REST with JavaScript map/reduce. The multi-master replication setup looked exactly like what we were looking for, in terms of having a scalable system.

The first design idea was to just replace S3 with CouchDB. We thought about normalizing the data (split by update frequency) to enable incremental updates, to speed up the publishing process. But in the end we realized that modeling n:m relationships with CouchDB's MapReduce is a hard problem to solve. Another issue was that CouchDB's views are incremental and persisted &mdash; which is quite useless if you are constantly changing the data underneath.

In the end, we added a application server written in node.js to the stack. We used node.js to assemble the normalized data served by CouchDB. We also added nginx with response caching in front of node.js. Additionally, we cached some data directly within the node.js process to speed up requests which were not already cached by nginx.

The request flow was like this:

* incoming request
* given a nginx cache miss...
* fetch placement data and priorities for this placement
* process the placement data and preselect ads based on the priority
* fetch the rest of the ad data from CouchDB
* build and send the response back to the client

With this in place, we were able to effectively cut the publishing time in half. We also laid out the foundation for upcoming features which required more business logic to serve ads.


### How to measure consistency with CouchDB in a multi-master setup?
We don't have strong requirements regarding consistency for most of our data. But we still need to make sure that an adserver with its CouchDB replication is not too far behind the others.

To solve this, we wrote a tracer document (containing a timestamp and the hostname) to the local CouchDB in each adserver instance. On the other hand, each adserver asked the ELB (Elastic Load Balancer) from time to time what other adservers were out there. Based on those hostnames, every adserver read the tracer documents of all the other servers and calculated the replication delay.

In case an adserver was too far behind, it removed itself from the ELB to allow the replication to catch up again. If the database became back in sync, the adserver automatically re-added itself to the ELB and started serving requests again.



## New Feature Requests
We are now in early 2011. This was also about the time when I started to work for adcloud on a regular base as a freelancer.

The problem we were facing is that we know that eventually all requests to the adservers for serving ads are going to be unique and therefor full-response caching is no longer an option. The main reason for the uniqueness of the requests are targeting features, like IP-based location-awareness e.g.

A quick test immediately showed that CouchDB was not able to provide the necessary throughput and latency we need to disable the caching in nginx. We also realized that caching things in node.js is pretty a bad idea &mdash; stop world garbage collection is a no-go for good and stable service quality in terms of latency.

{% img right /images/posts/nosql-fairy-tale/redis-logo.png %}

The solution was rather simple from an architectural perspective. We simply replaced our caching backend implemented from objects in node.js to Redis. Some cache objects and structures for lookups were changed to fully utilize native data types Redis provides. The migration was rather simple.

When we start an adserver, we have a cache warmup phase where we pre-fill all placement and ad data into Redis. Once we've finished that we start to accept requests and serve everything out of Redis. But since data is constantly being updated (the Platform publishes data), we needed to keep the data in Redis up to date. We did this with background workers that would handle this while the web worker can continue to serve (stale) data without creating a big impact on the performance.

The results of this step was pretty amazing. We were able to disable the full-response caching entirely while adding new features and maintaining a good and solid service quality and performance. We really love Redis for this!



## Scalability
We are now in mid- to late-2011, and scalability was the next big topic on our roadmap. This step focussed on our usage of CouchDB, since our scalability was limited by it.

If you take a look at how we used CouchDB and what we did not require at all, you quickly see that there is a mismatch.

We updated **>10k documents/h** (with only about 20k documents total in our database). We had just a single source of updates, the Platform. And we had multi-master replication in a 4-node cluster in place.

On the other hand, we did not require CouchDB's bullet-proof append-only durability, since all the data was essentially transient to the adserver since we constantly got new data. Another thing we did not need was the multi-version concurrency control (MVCC) that CouchDB offers: We only had one source of updates; there was no way that conflicts could happen.

The resulting issue was that we had huge problems with load caused by replication processes. CouchDB's multi-master replication was the one thing hindering us from scaling out. When we added more instances, we got more replication going on, and in the end even more load on each machine &mdash; this is not something you have in mind when scaling out a system.

Compaction was painful too: We accumulated a two-digit gigabyte number of data per server per day due to append-only. For our purpose, we didn't need our database to be rock-solid in terms of durability. And it was getting harder and harder to find a time window in the night when we could run the compaction.

We all realized that this was not CouchDB's fault. We were simply trying to apply the wrong use case to CouchDB. In our case we could not utilize CouchDB's strengths and instead triggered its weak points. Maybe we could have known better in the beginning, but nonetheless opting for CouchDB over S3 back then wasn't a bad idea at all. It helped a lot in terms of evolving the system and the company.

The solution to this situation has a touch of irony. With Redis in place, we replaced CouchDB for placement- and ad-data with S3. Since we weren't using any CouchDB-specific features, we simply published all the documents to S3 buckets instead. We still did the Redis cache warming upfront and data updates in the background. So by decoupling the application from the persistence layer using Redis, we also removed the need for a super fast database backend. We didn't care that S3 is slower than a local CouchDB, since we updated everything asynchronously.

In the end, S3 simply fits our needs. We could remove lots of code like the replication delay check. One additional benefit is that we have fewer moving parts now and less state on our application servers, which is also always a good thing too.



## Once again, more features
Of course there are always new feature requests on the horizon. We are now in early 2012. Before we can dive into our next challenge, we have to take a quick look at the status quo.

The first S3-based adserver did the entire ad selection on the client side (the user's browser). To a certain degree this was still the case with the CouchDB-node.js solution. One difference is that we did a pre-selection of ads there.

The challenge now is the goal to prepare the system for [Real-time bidding](http://en.wikipedia.org/wiki/Real-time_bidding) (RTB). RTB is quite a hot topic in the ad industry. Without going too much into the details what RTB is, let me try to describe it briefly. RTB establishes a auction, where you can bid on a single impression for a given user. For example: When we see a user, we are asking a number of third party services what they are willing to pay for this user/impression. We then need to compare the bids with the revenue we would generate from our own ads and decide whether to serve our ads or the 3rd party ads.<br />
*(Please excuse me if this is not 100% accurate &mdash; I hope you get the idea. I'm a tech person and not that super deep into the business mechanics behind the ad industry.)*

To enable RTB in our adservers, we need to bring the entire decision logic onto the server-side in order to tell which ads we want to deliver. And we have to do it rather fast. Since we need to query external systems and we have a network roundtrip to our final response to the request, we only have little time processing the bids. We are speaking of ~25ms here to choose from hundreds of possible ads.

The solution to this still excites me quite a bit: Redis. We know and trust Redis' performance. We know it has [sorted sets](http://redis.io/commands#sorted_set). We also have kind of sets of ads to display, and they are sorted by a priority. Why not utilize Redis for this selection logic?

We now heavily use Redis' sorted sets. The score is our ad priority which helps us selecting ads in the end. We build lot's of sets of ads that we can choose from for a given placement. But we also create sets of ads that mutually exclude each other &mdash; you don't want to see a McDonalds ad next to Burger King. For the selection we use mainly `ZUNIONSTORE` and `ZRANGEBYSCORE`. We first create a temporary set of ads that we can choose from, where we removed ads that cannot be shown at all. Then we apply our selection logic to determine which ads to pick. Ads with a higher priority (higher member score) are more likely to be shown.

What can I say? Our use case was very easy to model with Redis and it became a deeply integrated part of our business logic. Besides laying the foundation for Real-time bidding, we also improved the performance and quality for our users, by removing code in the client-side JavaScript and by reducing the payload that we need to send to the client. Overall we reduced the size of data that needs to be transfered by over 75%!



## Conclusion
We now have reached the end of our journey. We have shown you five generations of adcloud's adserver and you might have noticed that we did some radical changes in the architecture. We started out with just S3, moved to CouchDB, then we added Redis, replaced CouchDB with S3 and lately we are using Redis even more...

There are three importent drivers for our design decisions: features, quality & performance and scalability.

If we can deliver a new feature, that give us an advantage over competitors, than this is great. This step ahead of others financed the company very early on and enabled us to continuously grow and improve our systems. Service quality and performance is of course also a very important matter &mdash; only if the quality is good enough you can keep the competitive advantage. And last but not least, scalability. We need to enable our system and architecture to scale as the business grows.

When it comes to NoSQL we had to learn things the hard way sometimes. And one thing is for sure: There is no "one fits all" solution to the problems we are facing. We pick a tool that fits our needs and try to get the best out of it. If we are stuck, cannot scale any further, or we are hindered from implementing new features, we then consider re-evaluating the status quo. Don't be afraid of questioning existing systems.

Adcloud started with a very small team of four developers. Beside the adserver was the Platform which was to be build. Having a very simple solution utilizing S3 was a good decision back then. Going with very small, incremental steps was the only way to go back then. And even now the developer team in Cologne has grown significantly we still try to go by the same principles.



## Slides

<script async class="speakerdeck-embed" data-id="4fc63943c41f0f002101150c" data-ratio="1.3333333333333333" src="//speakerdeck.com/assets/embed.js"></script>