+++
title = "NRQL Bonanza"
outputs = ["Reveal"]
+++

# NRQL Bonanza 
# 🎉

Get more out of your data with NRQL!

---

<img src="https://che55er.io/img/me.png" width="300">

### Carl Chesser

* Sr. Principal Engieer @ Oracle (formerly Cerner)
* 18 years in software development 
* Used New Relic ~ 5 years

---

# About this Talk
## Exploring NRQL 🔍

* Assume you know about the New Relic Query Langauge (NRQL)
* Short exploration around learning more about your data
* Includes some short exploration of NerdGraph (GraphQL)

---

# Data Dictionary

https://docs.newrelic.com/attribute-dictionary/

![](nr-data-dictionary.png)

---

Search for attributes across entities

![](nr-data-dictionary-search.png)

---

Data Dictionary in NerdGraph!

```graphql{}
{
  docs {
    dataDictionary {
      events(names: "Transaction") {
        attributes {
          name
          definition(format: PLAIN)
          docsUrl
          events
          units {
            label
          }
        }
        dataSources {
          name
        }
        definition(format: PLAIN)
      }
    }
  }
}
```

---

Helpful if you are wanting to extract the dictionary data into other sources for consumption...

![](nr-nerdgraph-data-dictionary.png)

---

What about entities that describe usage of New Relic?

# 🤷‍♂️

---

# Usage

Use NRQL to understand more about your data

* `NrConsumption`
* `NrDailyUsage` once-a-day tracking of usage by product line
* `bytecountestimate` function on entities

---

# Usage

[`NrConsumption`](https://docs.newrelic.com/attribute-dictionary/?event=NrConsumption) to look usage metrics and consuming account name.

_Helpful if you have a multiple account organization._

```sql{1-4|5}
SELECT sum(GigabytesIngested) 
FROM NrConsumption
WHERE productLine = 'DataPlatform'
SINCE 3 days ago 
FACET usageMetric, consumingAccountName
TIMESERIES 1 hour
```

---

# Usage

[`NrConsumption`](https://docs.newrelic.com/attribute-dictionary/?event=NrConsumption) to compare metrics ingestion over 4 weeks.

```sql{}
SELECT sum(GigabytesIngested) AS avgGbIngestTimeseries
FROM NrConsumption 
WHERE productLine = 'DataPlatform' 
AND usageMetric = 'MetricsBytes'
SINCE 4 weeks ago COMPARE WITH 4 weeks ago TIMESERIES 1 day 
```

---

# Usage

[`NrDailyUsage`](https://docs.newrelic.com/attribute-dictionary/?event=NrDailyUsage) to look at counts of application (what for spikes of growth)

```sql{}
SELECT uniqueCount(apmAppName) 
FROM NrDailyUsage
FACET apmAppName
SINCE 30 days ago TIMESERIES 1 day
```

---

# Usage

`NrDailyUsage` to look at counts of application, by categorizing on an application name prefix.

```sql{1-2|3-4}
SELECT uniqueCount(apmAppName) 
FROM NrDailyUsage
-- Facet by application name prefix if you have (dash)
FACET capture(apmAppName, r'(?P<appName>.*)\-.*') 
SINCE 30 days ago TIMESERIES 1 day
```

---

# Usage Estimation

`bytecountestimate`, as the name implies, [is an estimate](https://docs.newrelic.com/docs/accounts/accounts-billing/new-relic-one-pricing-billing/usage-queries-alerts/#byte-count-estimate).

Evaluate metrics by source and metric name:

```sql{}
SELECT bytecountestimate()/10e8 
  AS 'GB Estimate'
FROM Metric 
SINCE 4 weeks ago
FACET newrelic.source, 
      metricName 
LIMIT 2000
```

---

# Usage Estimation

Evaluate the rate of APM transaction and distributed trace data by application and
event type (table) in a timeseries to look at spikes.

```sql{3-6|7-8}
SELECT rate(bytecountestimate()/10e8, 1 minute) 
  AS 'GB per minute' 
FROM Transaction, 
     TransactionError, 
     TransactionTrace, 
     Span
-- eventType() function allows you facet by entities
FACET appName, eventType()
TIMESERIES 1 day 
SINCE 3 days ago 
LIMIT 10
```

---

# Usage

Break down metrics by prefix name

```sql{}
SELECT count(newrelic.timeslice.value) 
FROM Metric 
WHERE appName LIKE 'foo%'
-- Ignore internal NR supportability metrics
AND prefix != 'Supportability'
WITH METRIC_FORMAT '{prefix}/{metric_name}' 
SINCE 1 hour ago LIMIT MAX FACET prefix TIMESERIES 
```

---

# Usage

Focus on metric names based on the prefix identified

```sql{1-3|4-7}
SELECT count(newrelic.timeslice.value) 
FROM Metric 
WHERE appName LIKE 'foo%'
-- Focus in on metric_names with External prefix
AND prefix = 'External'
WITH METRIC_FORMAT '{prefix}/{metric_name}' 
SINCE 1 hour ago LIMIT MAX FACET metric_name TIMESERIES 
```

---

What about entities that are not in the dictionary?

# 🤷‍♂️

---

# Discover

Get a listing of all the event types through this NRQL command:

```sql{}
SHOW EVENT TYPES
```

---

# Discover

Curious about what all columns are on an entity? Use [`keyset()`](https://docs.newrelic.com/docs/query-your-data/nrql-new-relic-query-language/get-started/nrql-syntax-clauses-functions/#keyset)!

```sql{}
-- Returns a JSON structure all of 
-- the attributes for the NrUsage entity
SELECT keyset()
FROM NrUsage
```

---

# Beyond the Dictionary

Use NRQL to understand more about your data

* `ApplicationAgentContext` for agent deployment context
* `NrdbQuery` for what queries are being used

---

# Applications Instrumented

Understand your applications and agents versions that are deployed.

```sql{}
SELECT count(*)
FROM ApplicationAgentContext
-- Replace 'foo' appName qualification 
-- for the entity you are searching
where appName LIKE 'foo%'
FACET appName, entity.guid, agent.version
SINCE 1 days ago
```

---

# Shadow Table

`NrdbQuery` helps track NRQL usage in your account.
What if I need to inspect what fields people are using?
Ex. deprecating one field for another ([7.5.0 agent upgrade](https://docs.newrelic.com/docs/release-notes/agent-release-notes/java-release-notes/java-agent-750/))...

```
SELECT count(*) 
FROM NrdbQuery
WHERE query 
  RLIKE r'.*(\s+|\.)(http.responseCode|response.status|response.statusMessage).*' 
AND query NOT LIKE '%NrdbQuery%'
AND query RLIKE r'(?i).*FROM Transaction.*'
FACET user, query 
SINCE 90 days AGO LIMIT MAX
```

---

# Shadow Table

`NrdbQuery` to help capture metrics being used via NRQL...

```
SELECT count(*) FROM NrdbQuery
-- Include metric name teams may be using in NRQL
WHERE query RLIKE '.*MemoryPool.*'
FACET user, query SINCE 7 days AGO
```

---

Inspect across account with NRQL via GraphQL

# 🚀

```graphql{}
{
  actor {
    account(id: 123) {
      nrql(query: "SELECT count(*) FROM ApplicationAgentContext WHERE agent.language = 'java' AND agent.version RLIKE r'[8-9]\.\d+\.\d+' SINCE 1 WEEK AGO FACET entity.guid, appName LIMIT MAX") {
        results
      }
    }
  }
} 
```

---

## Thank you!

[che55er.io](https://che55er.io)