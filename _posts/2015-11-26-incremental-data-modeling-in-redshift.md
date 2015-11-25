---
layout: post
title: Incremental data modeling in Redshift
author: Christophe
category: Data Modeling
---

Data modeling is a topic we think a lot about at Snowplow. It's a critical step in the pipeline because it's where business logic gets applied to the data.

The Snowplow trackers capture data in its rawest form. Events describe what has happened. No business logic gets applied at this stage. This is important because it means we end up with an event stream that is an unbiased log of all that has happened up to this point. If the segmentation algorithm changes, it can be run on historical as well as future events. The business logic that gets applied might (or should) change as the organization becomes more data sophisticated.

While the event stream itself is a gold mine for data scientists, it's not the format that is most useful to other users within the business. Most Snowplow users will have a step in between that create tables that transform the event-level data and/or join it with out datasets, apply business logic such as identity stitching, and roll it up to different levels that have meaning. This can happen in Looker in the form of PDTs. Snowplow users can also run SQL queries on a regular basis using our open source SQL Runner application. I will refer to these as the derived tables throughout the blogpost.

These queries will at first be run as follows:

- drop or delete from the existing derived tables
- insert into or create the tables again

This is perfect to get started. The tables are recomputed each time – which makes it possible to iterate with each run. However, as event volumes grow large, the queries will start to take longer and longer. While it's possible to do the data modeling in other environment (we are interested in Spark), there are also benefits to keep it in Redshift as long as possible. We have been using different approaches to incremental data modeling with many clients. The basic principle is simple: don't recompute data that hasn't been changed.

Let's get started with some example models!

## Example models

I will use a simple example model. This SQL statement aggregates events into sessions.

{% highlight sql %}
DROP TABLE IF EXISTS derived.sessions;
CREATE TABLE derived.sessions
  DISTKEY (domain_userid)
  SORTKEY (domain_userid, domain_sessionidx)
AS (
  SELECT
    domain_userid,
    domain_sessionidx,
    MIN(derived_tstamp) AS tstamp
    COUNT(DISTINCT page_url) AS distinct_pages_visited
  FROM atomic.events
  GROUP BY 1,2
);
{% endhighlight %}

Let's assume we have a table `derived.sessions` that has been recomputing with each run. We now want to keep that table around and add in all new events.

## Use a landing schema as a manifest

Snowplow ships with some examples data models – including `web-incremental`. This data model works as follows:

- storageloader loads events into a `landing` schema rather than `atomic`
- events in

This approach has its limitations:

- if a run fails on the SQL step, events will not be moved to atomic until the run goes through (other applications that consume the data in atomic might have to wait longer for the data)
- events are split between batches and we never consider data to be golden

-- our trackers cache events until they have arrived at the collector
-- information we receive today might change what we know about yesterday - we want to retain the flexbility to update our understanding of the past (this is still a hard sell in many places)

- need to duplicate all tables and update them

For instance. You might want to roll up events into a visitors table (with one row per visitors). When a user identifies on one device - you might realise that what were 2 separate visitors rows are the same user -- this fits in our identity stitching story.

## Use the ETL timestamp to create a manifest

This is a basic example for aggregation. But event-level data might be different -- easier.

The basic implementation

- run deduplication (cfr. dedupe post and release post)
- select all event ID where ETL timestamp not in manifest
- run the ID stitching

- create an events enriched table belonging

open question: also include all users or not? -- this is likely to be the slowest option



first method:

all relevant events in enriched_events - then aggregatate and delete and insert all

second method:

this batch first: aggregate then combine with existing rows using the first/last value approach then recompute on all else (fewer columns) and join these back o and delete/insert all
