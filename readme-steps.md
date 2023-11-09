# Start Flink (Using Docker)
docker compose up --build -d
docker compose run sql-client

# Shut Down Docker
docker compose down -v

# Deploying-ETL-pipeline-using-Flink-SQL
```bash
confluent login --save
confluent environment list
confluent environment use env-dgw817
confluent kafka cluster describe lkc-7qngqj

confluent kafka topic create pageviews --cluster <cluster id>  --environment <environment id>
confluent kafka topic describe pageviews --cluster lkc-7qngqj  --environment env-dgw817
```

### Create a table backed by Kafka
```bash
CREATE TABLE pageviews_kafka (
  `url` STRING,
  `user_id` STRING,
  `browser` STRING,
  `ts` TIMESTAMP(3)
) WITH (
  'connector' = 'kafka',
  'topic' = 'pageviews',
  'properties.group.id' = 'demoGroup',
  'scan.startup.mode' = 'earliest-offset',
  'properties.bootstrap.servers' = 'BOOTSTRAP_SERVER',
  'properties.security.protocol' = 'SASL_SSL',
  'properties.sasl.mechanism' = 'PLAIN',
  'properties.sasl.jaas.config' = 'org.apache.kafka.common.security.plain.PlainLoginModule required username="API_KEY" password="API_SECRET";',
  'value.format' = 'json',
  'sink.partitioner' = 'fixed'
);
```

### Create a source table
```bash
CREATE TABLE `pageviews` (
  `url` STRING,
  `user_id` STRING,
  `browser` STRING,
  `ts` TIMESTAMP(3)
)
WITH (
  'connector' = 'faker',
  'rows-per-second' = '10',
  'fields.url.expression' = '/#{GreekPhilosopher.name}.html',
  'fields.user_id.expression' = '#{numerify ''user_##''}',
  'fields.browser.expression' = '#{Options.option ''chrome'', ''firefox'', ''safari'')}',
  'fields.ts.expression' =  '#{date.past ''5'',''1'',''SECONDS''}'
);
```

### View data from source table
```bash
SELECT * FROM pageviews;
```

### Use INSERT INTO to create a long-lived Flink job (have to be off the VPN for insertion to work)
```bash
INSERT INTO pageviews_kafka SELECT * FROM pageviews;
```

### Verify that data is getting into Kafka
```bash
SELECT * FROM pageviews_kafka;
SELECT browser, COUNT(*) FROM pageviews_kafka GROUP BY browser;
```

# Streaming Analytics with Flink SQL
### add processing time
```bash
ALTER TABLE pageviews ADD `proc_time` AS PROCTIME();
```
# Specifying a watermark strategy
CREATE TABLE `pageviews` (
  ```bash
  `url` STRING,
  `user_id` STRING,
  `browser` STRING,
  `ts` TIMESTAMP(3),
  WATERMARK FOR `ts` AS ts - INTERVAL '5' SECOND
)
WITH (
  'connector' = 'faker',
  'rows-per-second' = '10',
  'fields.url.expression' = '/#{GreekPhilosopher.name}.html',
  'fields.user_id.expression' = '#{numerify ''user_##''}',
  'fields.browser.expression' = '#{Options.option ''chrome'', ''firefox'', ''safari'')}',
  'fields.ts.expression' =  '#{date.past ''5'',''1'',''SECONDS''}'
);

SELECT
  window_start, count(url) AS cnt
FROM TABLE(
  TUMBLE(TABLE pageviews, DESCRIPTOR(ts), INTERVAL '1' SECOND))
GROUP BY window_start;
```

# Pattern matching with MATCH_RECOGNIZE
```bash
SELECT *
FROM pageviews
    MATCH_RECOGNIZE (
      PARTITION BY user_id
      ORDER BY ts
      MEASURES
        A.browser AS browser1,
        B.browser AS browser2,
        A.ts AS ts1,
        B.ts AS ts2
      PATTERN (A B) WITHIN INTERVAL '1' SECOND
      DEFINE
        A AS true,
        B AS B.browser <> A.browser
    );

or using pageviews_kafka by change sink.partitioner from fixed to default:

CREATE TABLE pageviews_kafka (
  `url` STRING,
  `user_id` STRING,
  `browser` STRING,
  `ts` TIMESTAMP(3),
  WATERMARK FOR `ts` AS ts - INTERVAL '5' SECOND
) WITH (
  'connector' = 'kafka',
  'topic' = 'pageviews',
  'properties.group.id' = 'demoGroup',
  'scan.startup.mode' = 'earliest-offset',
  'properties.bootstrap.servers' = 'BOOTSTRAP_SERVER',
  'properties.security.protocol' = 'SASL_SSL',
  'properties.sasl.mechanism' = 'PLAIN',
  'properties.sasl.jaas.config' = 'org.apache.kafka.common.security.plain.PlainLoginModule required username="API_KEY" password="API_SECRET";',
  'value.format' = 'json',
  'sink.partitioner' = 'default'
);

SELECT *
FROM pageviews_kafka
    MATCH_RECOGNIZE (
      PARTITION BY user_id
      ORDER BY ts
      MEASURES
        A.browser AS browser1,
        B.browser AS browser2,
        A.ts AS ts1,
        B.ts AS ts2
      PATTERN (A B) WITHIN INTERVAL '1' SECOND
      DEFINE
        A AS true,
        B AS B.browser <> A.browser
    );

```
