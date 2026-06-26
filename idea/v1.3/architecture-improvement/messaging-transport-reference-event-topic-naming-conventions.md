# Kafka Topic Naming Conventions of Tim former project

## Suggestion

> > Once you create a topic in Kafka, it is `impossible to rename` them.
> > And `hard to migration` old data to new topic

- Do not use fields that possible change future. Example:
  - Is Producer or Consumer.
  - Team name.
  - Topic owner.
  - Application name not in common case, Prefer `Domain service mame`.
  - Version number like `v1`/`v2`, it's will cause lead to the fact that countless topics are created quickly.
- `Prod Env` need `disable auto-create topic` by application, all topic should be manually create.
  - Not disable delete topic this time.

## Topic Rule

```
<scope>.<message type>.<domain name>.<domain data>
```

- topic separate with `.` and lowercase.

### Scope

- public
  - Everyone can consume for self async operation
- private
  - Just use of self this topic for async operation

### Message type

- queuing
  - For classical queuing use cases.
- tracking
  - For tracking events such as user clicks, page views, ad views, etc.
- etl/db
  - For ETL/Database feed
- streaming
  - For intermediate topics created by stream processing pipelines.
- push
  - For data that’s being pushed from other source
- state
  - For data that's state changed, use for notify other need be known
- user/test
  - For user-specific data such as scratch and test topics

### Domain Name

Such like:

- asset-iot
- tag
- site

### Domain Data Name

Such like:

- geolocation
- status
- battery

## Reference

- [kafka-topic-naming-conventions](https://cnr.sh/essays/how-paint-bike-shed-kafka-topic-naming-conventions)
- [kafka-topic-naming-conventions-best-practices](https://medium.com/@kiranprabhu/kafka-topic-naming-conventions-best-practices-6b6b332769a3)
- [kafka-topic-naming-conventions-5-recommendations-with-examples](https://www.kadeck.com/blog/kafka-topic-naming-conventions-5-recommendations-with-examples)
