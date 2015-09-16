---
layout: page
group: documentation
subgroup: data modeling
breadcrumb: sessionization
subbreadcrumb: advanced server-side sessionization in sql
rank: 3
title: Reconciling Snowplow and Google Analytics session numbers
description: Reconciling Snowplow and Google Analytics session numbers.
permalink: /documentation/data-modeling/sessionization/reconciling-snowplow-and-ga/
---

Most Snowplow users aggregate events into sessions using `domain_sessionidx`, which is generated by our Javascript tracker and increments when a user was not active for 30 minutes. When comparing with Google Analytics, Snowplow users might find that both tools reports slightly different session numbers. This is because Google Analytics also [expires sessions](https://support.google.com/analytics/answer/2731565) at midnight and when a user changes campaigns (i.e arrives via one campaign, leaves, and then comes back via a different campaign).

To reconcile these numbers, it's possible to create a new session index in SQL which increments when a user has not been active for 30 minutes or when a user leaves and comes back via a different referrer.

{% highlight sql %}
WITH step_1 AS (

  SELECT

    domain_userid,
    domain_sessionidx, -- to compare with the custom session index

    collector_tstamp,
    dvce_tstamp,

    LAG(dvce_tstamp, 1) OVER (PARTITION BY domain_userid ORDER BY dvce_tstamp) AS previous_dvce_tstamp,
    NVL(page_referrer, '') || NVL(mkt_medium, '') || NVL(mkt_source, '') || NVL(mkt_term, '') || NVL(mkt_content, '') || NVL(mkt_campaign, '') AS referrer,

    refr_source,
    refr_medium,
    refr_term,

    mkt_source,
    mkt_medium,
    mkt_term,
    mkt_content,
    mkt_campaign

  FROM atomic.events
  WHERE collector_tstamp::date = 'YYYY-MM-DD' -- restrict the dataset
  ORDER BY domain_userid, dvce_tstamp

), step_2 AS (

  SELECT

    *, SUM(CASE WHEN refr_medium = 'internal' OR referrer IS NULL OR referrer = '' THEN 0 ELSE 1 END) OVER (PARTITION BY domain_userid ORDER BY dvce_tstamp ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS new_referrer

  FROM step_1
  ORDER BY domain_userid, dvce_tstamp

), step_3 AS (

  SELECT

  *, FIRST_VALUE(referrer) OVER (PARTITION BY domain_userid, new_referrer ORDER BY dvce_tstamp ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS referrer_partition

  FROM step_2
  ORDER BY domain_userid, dvce_tstamp

), step_4 AS (

  SELECT

    *, LAG(referrer_partition, 1) OVER (PARTITION BY domain_userid ORDER BY dvce_tstamp) AS previous_referrer_partition

  FROM step_3
  ORDER BY domain_userid, dvce_tstamp

), step_5 AS (

  SELECT

    *,
    CASE
      WHEN ((EXTRACT(EPOCH FROM (dvce_tstamp - previous_dvce_tstamp)) < 60*30) AND (referrer_partition = previous_referrer_partition OR (referrer_partition IS NULL AND previous_referrer_partition IS NULL))) THEN 0
      ELSE 1
    END AS new_session

  FROM step_4
  ORDER BY domain_userid, dvce_tstamp

), step_6 AS (

  SELECT

    *, SUM(new_session) OVER (PARTITION BY domain_userid ORDER BY dvce_tstamp ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS new_sessionidx

  FROM step_5
  ORDER BY domain_userid, dvce_tstamp

)

SELECT
  domain_userid,
  new_sessionidx
FROM step_6
GROUP BY 1,2
ORDER BY 1,2
{% endhighlight %}

The SQL is explained in more detail in our documentation on [server-side sessionization in SQL](../basic-sessionization-in-sql). The main addition are steps 2 to 4, which partition events on referrer. This is used to select events where the user arrived via a different external referrer.

## Differences in allocation to source/medium

The above query should return session numbers that are closer to what is reported by Google Analytics. However, it's also possible that the number of sessions per source/medium differs between Snowplow and Google Analytics, even when the total matches. This is because Google Analytics uses [data from past sessions](https://support.google.com/analytics/answer/6205762#flowchart) when no referrer information is available. This following SQL query replicates this logic:

{% highlight sql %}
WITH step_7 AS (

  SELECT
    domain_userid,
    new_sessionidx,

    collector_tstamp,
    dvce_tstamp,

    refr_source,
    refr_medium,
    mkt_source,
    mkt_medium

  FROM step_6
  WHERE new_session = 1
  ORDER BY domain_userid, new_sessionidx

), step_8 AS (

  SELECT

    *, SUM(CASE WHEN mkt_source IS NULL THEN 0 ELSE 1 END) OVER (PARTITION BY domain_userid ORDER BY new_sessionidx ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS mkt_partition

  FROM step_7
  ORDER BY domain_userid, new_sessionidx

), step_9 AS (

  SELECT
    *,
    FIRST_VALUE(dvce_tstamp) OVER (PARTITION BY domain_userid, mkt_partition ORDER BY new_sessionidx ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS partition_dvce_tstamp,
    FIRST_VALUE(mkt_source) OVER (PARTITION BY domain_userid, mkt_partition ORDER BY new_sessionidx ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS partition_mkt_source,
    FIRST_VALUE(mkt_medium) OVER (PARTITION BY domain_userid, mkt_partition ORDER BY new_sessionidx ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS partition_mkt_medium
  FROM step_8
  ORDER BY domain_userid, new_sessionidx

), step_10 AS (

  SELECT

    *,
    CASE
      WHEN mkt_source IS NOT NULL THEN 'mkt-' || mkt_source
      WHEN refr_source IS NOT NULL THEN 'refr-' || refr_source
      WHEN partition_mkt_source IS NOT NULL AND (EXTRACT(EPOCH FROM (dvce_tstamp - partition_dvce_tstamp)) < 60*60*24*183) THEN 'mkt-' || partition_mkt_source
      ELSE 'direct'
    END AS ga_source,
    CASE
      WHEN mkt_source IS NOT NULL THEN 'mkt-' || mkt_medium
      WHEN refr_source IS NOT NULL THEN 'refr-' || refr_medium
      WHEN partition_mkt_source IS NOT NULL AND (EXTRACT(EPOCH FROM (dvce_tstamp - partition_dvce_tstamp)) < 60*60*24*183) THEN 'mkt-' || partition_mkt_medium
      ELSE 'direct'
    END AS ga_medium

  FROM step_9
  ORDER BY domain_userid, new_sessionidx

)

-- count source:

SELECT
  ga_source,
  count(*)
FROM step_10
GROUP BY 1
ORDER BY 2 DESC

-- count medium:

SELECT
  ga_medium,
  count(*)
FROM step_10
GROUP BY 1
ORDER BY 2 DESC
{% endhighlight %}

Steps 8 and 9 create a new partition each time `mkt_source` is set. When both `mkt_source` and `refr_source` are `NULL`, previous campaign data is used if it is within the timeout period.