# Clickhouse Cluster

Clickhouse cluster with 2 shards and 2 replicas built with docker-compose.

## Run

Run single command, and it will copy configs for each node and
run Clickhouse cluster `company_cluster` with docker-compose

```sh
make config up
```

Containers will be available in docker network `172.23.0.0/24`

| Container    | Address
| ------------ | -------
| zookeeper    | 172.23.0.10
| clickhouse01 | 172.23.0.11
| clickhouse02 | 172.23.0.12
| clickhouse03 | 172.23.0.13
| clickhouse04 | 172.23.0.14

## Test it

Login to clickhouse01 console (first node's ports are mapped to localhost)
```sh
clickhouse-client -h localhost
```

Create a test database and table (sharded and replicated)
```sql
CREATE DATABASE company_db ON CLUSTER '{cluster}';

CREATE TABLE company_db.events_local ON CLUSTER '{cluster}' (
    timestamp DateTime,
    uid  Int64,
    type LowCardinality(String)
)
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{cluster}/{shard}/{table}', '{replica}')
PARTITION BY toDate(time)
ORDER BY (uid);

CREATE TABLE company_db.events ON CLUSTER '{cluster}' AS company_db.events_local
ENGINE = Distributed('{cluster}', company_db, events_local, intHash32(uid));
```

Load some data via Distributed table (CH will take care of sharding):
```sql
INSERT INTO company_db.events VALUES
    ('2020-01-01 10:00:00', 1005, 'view'),
    ('2020-01-01 10:05:00', 1015, 'view'),
    ('2020-01-01 11:00:00', 1005, 'contact'),
    ('2020-01-01 12:10:00', 1015, 'view'),
    ('2020-01-02 08:10:00', 1005, 'view'),
    ('2020-01-03 13:00:00', 1035, 'view');
```

Check data from current shard
```sql
SELECT * FROM company_db.events_local;
```

Check data from all cluster
```sql
SELECT * FROM company_db.events;
```
