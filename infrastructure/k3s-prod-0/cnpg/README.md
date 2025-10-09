# Benchmarking CloudNativePG with JuiceFS

We use `pgbench` for benchmarking. We successively test three different circumstances:

1. Small database with a scale factor of `--scale=32`, and `shared_buffers=1GB`. Guaranteeing that all pages are always in memory.
2. Large database with a scale factor of `--scale=100` and `shared_buffers=1GB`. Here, the database files takes up somewhere around `1.6GB`. The `pgbench` randomly accesses across the entire database file, thus forcing dirty pages to constatly flush to backend block stroage.
3. Large database with a scale factor of `--scale=90` and `shared_buffers=1.5GB`. Now, the database files takes up roughly `1.5GB`, similar to the page size.

## Small database

We create a Postgres cluster with the following configuration:

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: jfs-cluster2
  namespace: default
  labels:
    zty89.com/env: database
spec:
  instances: 3
  resources:
    limits:
      cpu: 4000m
      memory: 4Gi
    requests:
      cpu: 2000m
      memory: 2Gi
  postgresql:
    synchronous:
      method: any
      number: 1
      dataDurability: required
    parameters:
      checkpoint_timeout: "30min"
      max_wal_size: "16GB"
      wal_level: "replica"
      shared_buffers: "1GB"
  storage:
    pvcTemplate:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 10Gi
      storageClassName: juicefs-0
      volumeMode: Filesystem
  walStorage:
    size: 20Gi
  monitoring:
    enablePodMonitor: false
```

This limits the cluster to `16GB` of WAL file with `4GB` of headroom for abrupt commits. We setup the tables with the following command:

```zsh
 ~/gitops/production ❯ pgbench -h 192.168.3.2 -p 31000 -U app -d app -i --scale=32
Password: 
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
vacuuming...                                                                                 
creating primary keys...
done in 24.09 s (drop tables 0.00 s, create tables 0.20 s, client-side generate 12.59 s, vacuum 3.95 s, primary keys 7.34 s).
```

We run the small database benchamrk:

```zsh
 ~/gitops/production ❯ pgbench -h 192.168.3.2 -p 31000 -U app -d app --jobs=2 --client=10 --time=300
Password: 
pgbench (17.6, server 17.5 (Debian 17.5-1.pgdg110+1))
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 32
query mode: simple
number of clients: 10
number of threads: 2
maximum number of tries: 1
duration: 300 s
number of transactions actually processed: 608475
number of failed transactions: 0 (0.000%)
latency average = 4.928 ms
initial connection time = 156.676 ms
tps = 2029.252945 (without initial connection time)
```

and a second time with more elaborate per-command latency

```zsh
 ~/gitops/production ❯ pgbench -h 192.168.3.2 -p 31000 -U app -d app -r --jobs=2 --client=10 --time=300
Password: 
pgbench (17.6, server 17.5 (Debian 17.5-1.pgdg110+1))
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 32
query mode: simple
number of clients: 10
number of threads: 2
maximum number of tries: 1
duration: 300 s
number of transactions actually processed: 566549
number of failed transactions: 0 (0.000%)
latency average = 5.292 ms
initial connection time = 169.717 ms
tps = 1889.533025 (without initial connection time)
statement latencies in milliseconds and failures:
         0.001           0  \set aid random(1, 100000 * :scale)
         0.000           0  \set bid random(1, 1 * :scale)
         0.000           0  \set tid random(1, 10 * :scale)
         0.000           0  \set delta random(-5000, 5000)
         0.488           0  BEGIN;
         0.681           0  UPDATE pgbench_accounts SET abalance = abalance + :delta WHERE aid = :aid;
         0.576           0  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
         0.618           0  UPDATE pgbench_tellers SET tbalance = tbalance + :delta WHERE tid = :tid;
         0.812           0  UPDATE pgbench_branches SET bbalance = bbalance + :delta WHERE bid = :bid;
         0.665           0  INSERT INTO pgbench_history (tid, bid, aid, delta, mtime) VALUES (:tid, :bid, :aid, :delta, CURRENT_TIMESTAMP);
         1.445           0  END;
```

The database handles roughly `1900` r/w combined transactions per second.

## Large database w/ limited buffer

We use the same configuration for cluster, and init the database as follows:

```zsh
 ~/gitops/production ❯ pgbench -h 192.168.3.2 -p 31000 -U app -d app -i --scale=100
Password: 
dropping old tables...
creating tables...
generating data (client-side)...
vacuuming...                                                                                   
creating primary keys...
done in 75.74 s (drop tables 0.21 s, create tables 0.04 s, client-side generate 41.47 s, vacuum 13.97 s, primary keys 20.05 s).
```

We then commence testing as follows

```zsh
 ~/gitops/production ❯ pgbench -h 192.168.3.2 -p 31000 -U app -d app --jobs=2 --client=10 --time=300
Password: 
pgbench (17.6, server 17.5 (Debian 17.5-1.pgdg110+1))
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 100
query mode: simple
number of clients: 10
number of threads: 2
maximum number of tries: 1
duration: 300 s
number of transactions actually processed: 168857
number of failed transactions: 0 (0.000%)
latency average = 17.985 ms
initial connection time = 153.188 ms
tps = 556.013962 (without initial connection time)
```

and a more detailed second attempt

```zsh
 ~/gitops/production ❯ pgbench -h 192.168.3.2 -p 31000 -U app -d app -r --jobs=2 --client=10 --time=30 -r
Password: 
pgbench (17.6, server 17.5 (Debian 17.5-1.pgdg110+1))
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 100
query mode: simple
number of clients: 10
number of threads: 2
maximum number of tries: 1
duration: 30 s
number of transactions actually processed: 23035
number of failed transactions: 0 (0.000%)
latency average = 12.942 ms
initial connection time = 198.202 ms
tps = 772.652496 (without initial connection time)
statement latencies in milliseconds and failures:
         0.002           0  \set aid random(1, 100000 * :scale)
         0.001           0  \set bid random(1, 1 * :scale)
         0.000           0  \set tid random(1, 10 * :scale)
         0.000           0  \set delta random(-5000, 5000)
         0.723           0  BEGIN;
         4.704           0  UPDATE pgbench_accounts SET abalance = abalance + :delta WHERE aid = :aid;
         1.007           0  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
         0.848           0  UPDATE pgbench_tellers SET tbalance = tbalance + :delta WHERE tid = :tid;
         0.987           0  UPDATE pgbench_branches SET bbalance = bbalance + :delta WHERE bid = :bid;
         0.847           0  INSERT INTO pgbench_history (tid, bid, aid, delta, mtime) VALUES (:tid, :bid, :aid, :delta, CURRENT_TIMESTAMP);
         3.787           0  END;
```

during the third attemp for consistency test, we observe instances begin to crash and exit under high pressure

```zsh
 ~/gitops/production ❯ pgbench -h 192.168.3.2 -p 31000 -U app -d app -r --jobs=2 --client=10 --time=300 -r
Password: 
pgbench (17.6, server 17.5 (Debian 17.5-1.pgdg110+1))
starting vacuum...end.
pgbench: error: client 7 aborted in command 5 (SQL) of script 0; perhaps the backend died while processing
pgbench: error: client 1 aborted in command 5 (SQL) of script 0; perhaps the backend died while processing
pgbench: error: client 5 aborted in command 5 (SQL) of script 0; perhaps the backend died while processing
pgbench: error: client 2 aborted in command 5 (SQL) of script 0; perhaps the backend died while processing
pgbench: error: client 0 aborted in command 5 (SQL) of script 0; perhaps the backend died while processing
pgbench: error: client 4 aborted in command 5 (SQL) of script 0; perhaps the backend died while processing
pgbench: error: client 3 aborted in command 5 (SQL) of script 0; perhaps the backend died while processing
pgbench: error: client 6 aborted in command 5 (SQL) of script 0; perhaps the backend died while processing
pgbench: error: client 8 aborted in command 5 (SQL) of script 0; perhaps the backend died while processing
pgbench: error: client 9 aborted in command 5 (SQL) of script 0; perhaps the backend died while processing
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 100
query mode: simple
number of clients: 10
number of threads: 2
maximum number of tries: 1
duration: 300 s
number of transactions actually processed: 137267
number of failed transactions: 0 (0.000%)
latency average = 20.452 ms
initial connection time = 173.305 ms
tps = 488.952870 (without initial connection time)
statement latencies in milliseconds and failures:
         0.001           0  \set aid random(1, 100000 * :scale)
         0.001           0  \set bid random(1, 1 * :scale)
         0.001           0  \set tid random(1, 10 * :scale)
         0.000           0  \set delta random(-5000, 5000)
         0.858           0  BEGIN;
         8.161           0  UPDATE pgbench_accounts SET abalance = abalance + :delta WHERE aid = :aid;
         1.200           0  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
         0.977           0  UPDATE pgbench_tellers SET tbalance = tbalance + :delta WHERE tid = :tid;
         1.431           0  UPDATE pgbench_branches SET bbalance = bbalance + :delta WHERE bid = :bid;
         1.031           0  INSERT INTO pgbench_history (tid, bid, aid, delta, mtime) VALUES (:tid, :bid, :aid, :delta, CURRENT_TIMESTAMP);
         4.217           0  END;
pgbench: error: Run was aborted; the above results are incomplete.
```

Altogether, we observe roughly `800` r/w combined transactions per second at peak, and around `500` r/w combined transactions per second when under load (i.e. High JuiceFS staging size, degraded object storage speed).

## Large database w/ abundant buffer

For the final test, we use the following configuration file:

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: jfs-cluster2
  namespace: default
  labels:
    zty89.com/env: database
spec:
  instances: 3
  resources:
    limits:
      cpu: 4000m
      memory: 4Gi
    requests:
      cpu: 2000m
      memory: 2Gi
  postgresql:
    synchronous:
      method: any
      number: 1
      dataDurability: required
    parameters:
      checkpoint_timeout: "30min"
      max_wal_size: "16GB"
      wal_level: "replica"
      shared_buffers: "1.5GB"
  storage:
    pvcTemplate:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 10Gi
      storageClassName: juicefs-0
      volumeMode: Filesystem
  walStorage:
    size: 20Gi
  monitoring:
    enablePodMonitor: false
```

Note that we increased the `shared_buffers` to `1.5GB`, which is roughly the same as the database size in the beginning (the history table constantly appends new entries during benchmark, increasing the final database size). We thus initialize the database as follows

```zsh
 ~/gitops/production ❯ pgbench -h 192.168.3.2 -p 31000 -U app -d app -i --scale=90
Password: 
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
vacuuming...                                                                                 
creating primary keys...
done in 58.55 s (drop tables 0.00 s, create tables 0.07 s, client-side generate 37.86 s, vacuum 9.32 s, primary keys 11.29 s).
```

And shortly afters, the benchmark

```zsh
 ~/gitops/production ❯ pgbench -h 192.168.3.2 -p 31000 -U app -d app -r --jobs=2 --client=10 --time=300
Password: 
pgbench (17.6, server 17.5 (Debian 17.5-1.pgdg110+1))
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 90
query mode: simple
number of clients: 10
number of threads: 2
maximum number of tries: 1
duration: 300 s
number of transactions actually processed: 821343
number of failed transactions: 0 (0.000%)
latency average = 3.651 ms
initial connection time = 164.833 ms
tps = 2739.282200 (without initial connection time)
statement latencies in milliseconds and failures:
         0.001           0  \set aid random(1, 100000 * :scale)
         0.000           0  \set bid random(1, 1 * :scale)
         0.000           0  \set tid random(1, 10 * :scale)
         0.000           0  \set delta random(-5000, 5000)
         0.293           0  BEGIN;
         0.431           0  UPDATE pgbench_accounts SET abalance = abalance + :delta WHERE aid = :aid;
         0.362           0  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
         0.383           0  UPDATE pgbench_tellers SET tbalance = tbalance + :delta WHERE tid = :tid;
         0.468           0  UPDATE pgbench_branches SET bbalance = bbalance + :delta WHERE bid = :bid;
         0.395           0  INSERT INTO pgbench_history (tid, bid, aid, delta, mtime) VALUES (:tid, :bid, :aid, :delta, CURRENT_TIMESTAMP);
         1.311           0  END;
```

The benchmark reaches a peak performance of `2700` r/w mixed transactions per second. These experiments shows the importnace of keeping an eye on performance degredations, and increasing memory limit during production.

---

# Appendix

## Storage Class configuration

During the above test, we use the following JuiceFS configuration file:

```yaml
globalConfig:
  mountPodPatch:
    - resources:
        requests:
          cpu: 1000m
          memory: 2Gi
        limits:
          cpu: 2000m
          memory: 4Gi
      mountOptions:
        - backup-meta=86400
        - backup-skip-trash
        - cache-size=50G
    - pvcSelector:
        matchLabels:
          zty89.com/env: "database"
      mountOptions:
        - backup-meta=86400
        - backup-skip-trash
        - cache-size=50G
        - upload-delay=5s
        - writeback_cache
        - writeback
        - buffer-size=1G
        - prefetch=1
        - max-readahead=0
```

All cnpg clusters using juicefs are labeled `zty89.com/env: "database"`, enabling local `writeback` buffer with `upload-delay=5s` of delay for block consolidation.
