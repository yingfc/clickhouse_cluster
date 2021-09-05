# Clickhouse Cluster

Clickhouse cluster with 2 shards and 2 replicas built with docker-compose.

## Run

Run single command, and it will copy configs for each node and
run clickhouse cluster `test_cluster` with docker-compose
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

## Profiles

- `default` - no password
- `admin` - password `123`

## Test it

Login to clickhouse01 console (first node's ports are mapped to localhost)
```sh
clickhouse-client -h localhost --password xxx
```

Or open `clickhouse-client` inside any container
```sh
docker exec -it clickhouse01 clickhouse-client -h localhost
```

Create a test database and table (sharded and replicated)
```sql
CREATE DATABASE test_db ON CLUSTER 'test_cluster';

CREATE TABLE test_db.events ON CLUSTER 'test_cluster' (
    time DateTime,
    uid  Int64,
    type LowCardinality(String)
)
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{cluster}/{shard}/table', '{replica}')
PARTITION BY toDate(time)
ORDER BY (uid);

CREATE TABLE test_db.events_distr ON CLUSTER 'test_cluster' AS test_db.events
ENGINE = Distributed('test_cluster', test_db, events, uid);
```

Load some data
```sql
INSERT INTO test_db.events_distr VALUES
    ('2020-01-01 10:00:00', 100, 'view'),
    ('2020-01-01 10:05:00', 101, 'view'),
    ('2020-01-01 11:00:00', 100, 'contact'),
    ('2020-01-01 12:10:00', 101, 'view'),
    ('2020-01-02 08:10:00', 100, 'view'),
    ('2020-01-03 13:00:00', 103, 'view');
```

Check data from current shard
```sql
SELECT * FROM test_db.events;
```

Check data from all cluster
```sql
SELECT * FROM test_db.events_distr;
```