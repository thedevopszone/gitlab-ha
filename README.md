# gitlab-ha

High-availability infrastructure for GitLab, managed with Ansible.

## Components

| Component  | Role             | Description                                      |
|------------|------------------|--------------------------------------------------|
| etcd       | `etcd`           | 3-node distributed key-value store (DCS)         |
| PostgreSQL | `patroni`        | 3-node PostgreSQL 16 cluster managed by Patroni  |
| Redis      | `redis_sentinel` | 3-node Redis with Sentinel for automatic failover|
| Gitaly     | `gitaly`         | 3-node Git storage backend (GitLab Omnibus)      |

## Nodes

| Host           | IP            | etcd | PostgreSQL / Patroni | Redis / Sentinel | Gitaly      |
|----------------|---------------|:----:|:--------------------:|:----------------:|:-----------:|
| gl-prd-core-01 | 172.16.0.183  |  x   |          x           |   x (master)     | x (gitaly-1)|
| gl-prd-core-02 | 172.16.0.147  |  x   |          x           |   x (replica)    | x (gitaly-2)|
| gl-prd-core-03 | 172.16.0.145  |  x   |          x           |   x (replica)    | x (gitaly-3)|

## Prerequisites

- Ubuntu 24.04 LTS on all target nodes
- Ansible installed on the control node
- SSH access with sudo privileges to all target nodes

## Project Structure

```
gitlab-ha/
├── ansible.cfg
├── inventories/
│   └── production/
│       ├── hosts.yml                      # all host/group definitions
│       └── group_vars/
│           ├── etcd_cluster.yml           # etcd cluster variables
│           ├── patroni.yml               # Patroni / PostgreSQL variables
│           ├── redis.yml                 # Redis / Sentinel variables
│           └── gitaly.yml                # Gitaly cluster variables
├── roles/
│   ├── etcd/
│   │   ├── defaults/main.yml
│   │   ├── tasks/main.yml
│   │   ├── templates/
│   │   │   ├── etcd.conf.yml.j2
│   │   │   └── etcd.service.j2
│   │   └── handlers/main.yml
│   ├── patroni/
│   │   ├── defaults/main.yml
│   │   ├── tasks/main.yml
│   │   ├── templates/
│   │   │   ├── patroni.yml.j2
│   │   │   └── patroni.service.j2
│   │   └── handlers/main.yml
│   ├── redis_sentinel/
│   │   ├── defaults/main.yml
│   │   ├── tasks/main.yml
│   │   ├── templates/
│   │   │   ├── redis.conf.j2
│   │   │   └── sentinel.conf.j2
│   │   └── handlers/main.yml
│   └── gitaly/
│       ├── defaults/main.yml
│       ├── tasks/main.yml
│       ├── templates/
│       │   └── gitlab.rb.j2
│       └── handlers/main.yml
└── playbooks/
    ├── etcd.yml
    ├── patroni.yml
    ├── redis.yml
    └── gitaly.yml
```

## Deployment Order

The services must be deployed in the following order because Patroni depends on etcd for leader election and cluster state:

```
1. etcd       →  ansible-playbook playbooks/etcd.yml
2. patroni    →  ansible-playbook playbooks/patroni.yml
3. redis      →  ansible-playbook playbooks/redis.yml
4. gitaly     →  ansible-playbook playbooks/gitaly.yml
```

> All playbooks use the default inventory defined in `ansible.cfg`. You can also specify it explicitly with `-i inventories/production/hosts.yml`.

---

## etcd Cluster

### Architecture

- Cluster name: `gitlab-etcd-cluster`
- Client port: `2379`
- Peer port: `2380`
- Data directory: `/var/lib/etcd`
- Configuration: `/etc/etcd/etcd.conf.yml`
- v2 API: enabled (required by Patroni's `python-etcd` client)

### Deploying etcd

```bash
ansible-playbook playbooks/etcd.yml
```

The role installs `etcd-server` and `etcd-client` via apt, creates a dedicated system user, deploys the configuration from template, and starts the cluster. The playbook is idempotent and safe to re-run.

### Checking etcd Health

#### Service status on all nodes

```bash
ansible etcd_cluster -b -m shell -a "systemctl is-active etcd"
```

Expected output when healthy:

```
gl-prd-core-01 | CHANGED | rc=0 >> active
gl-prd-core-02 | CHANGED | rc=0 >> active
gl-prd-core-03 | CHANGED | rc=0 >> active
```

#### Cluster endpoint health

```bash
export ETCDCTL_API=3
etcdctl \
  --endpoints=http://172.16.0.183:2379,http://172.16.0.147:2379,http://172.16.0.145:2379 \
  endpoint health
```

All three endpoints should report `is healthy`.

#### Member list

```bash
export ETCDCTL_API=3
etcdctl \
  --endpoints=http://172.16.0.183:2379,http://172.16.0.147:2379,http://172.16.0.145:2379 \
  member list -w table
```

Verify all three members show `STATUS = started` and `IS LEARNER = false`.

#### Endpoint status (leader, DB size, raft index)

```bash
export ETCDCTL_API=3
etcdctl \
  --endpoints=http://172.16.0.183:2379,http://172.16.0.147:2379,http://172.16.0.145:2379 \
  endpoint status -w table
```

#### Logs

```bash
ssh gl-prd-core-01 'journalctl -u etcd --no-pager -n 50'
```

### Updating etcd

#### Updating the configuration

1. Edit the relevant variables in `inventories/production/group_vars/etcd_cluster.yml` or the template in `roles/etcd/templates/etcd.conf.yml.j2`.

2. Run the playbook:

   ```bash
   ansible-playbook playbooks/etcd.yml
   ```

   The role is idempotent. Only nodes with changed configuration files will be restarted (via handlers).

3. Verify the cluster is healthy after the run (see checks above).

#### Upgrading the etcd package

The role installs etcd via `apt` with `state: present`, which does not upgrade an already-installed package.

**Rolling upgrade (recommended):** Upgrade one node at a time to maintain quorum:

```bash
# Upgrade a single node
ansible gl-prd-core-01 -b -m apt -a "name=etcd-server state=latest update_cache=yes"

# Restart etcd on that node
ansible gl-prd-core-01 -b -m systemd -a "name=etcd state=restarted"

# Verify cluster health before moving to the next node
ansible gl-prd-core-01 -b -m shell -a \
  "ETCDCTL_API=3 etcdctl --endpoints=http://172.16.0.183:2379,http://172.16.0.147:2379,http://172.16.0.145:2379 endpoint health"
```

Repeat for `gl-prd-core-02` and `gl-prd-core-03`.

After all nodes are upgraded, confirm the version:

```bash
ansible etcd_cluster -b -m shell -a "etcd --version"
```

### etcd Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `systemctl status etcd` shows `inactive` | Service crashed or was stopped | `systemctl start etcd`, check logs with `journalctl -u etcd` |
| `endpoint health` shows one node unhealthy | Network issue or node down | Check connectivity on port 2380 between nodes, restart etcd on the affected node |
| `member list` shows only 2 members | A member was removed or failed to join | Re-add the member with `etcdctl member add`, then re-run the playbook |
| Cluster won't start after full outage | All members trying to join as `new` | Set `etcd_initial_cluster_state: existing` if data directories are intact, or wipe `/var/lib/etcd` on all nodes and redeploy with `new` |

---

## PostgreSQL / Patroni Cluster

### Architecture

| Setting              | Value                                  |
|----------------------|----------------------------------------|
| Cluster name         | `gitlab-pg-cluster`                    |
| PostgreSQL version   | 16                                     |
| PostgreSQL port      | `5432`                                 |
| Patroni REST API     | `8008`                                 |
| Data directory       | `/var/lib/postgresql/16/main`          |
| WAL archive          | `/var/lib/postgresql/wal_archive`      |
| Patroni config       | `/etc/patroni/patroni.yml`             |
| DCS backend          | etcd (v2 API)                          |
| Replication user     | `replicator`                           |
| Superuser            | `postgres`                             |
| Listen addresses     | `0.0.0.0` (all interfaces)             |
| Network CIDR (HBA)   | `172.16.0.0/16`                        |

Patroni handles all PostgreSQL lifecycle management: initialization, configuration, startup, failover, and replication. The native `postgresql` systemd service is disabled.

### Deploying PostgreSQL / Patroni

> **Prerequisite:** The etcd cluster must be running before deploying Patroni.

```bash
ansible-playbook playbooks/patroni.yml
```

#### What the playbook does

1. Adds the official PostgreSQL APT repository (PGDG)
2. Installs PostgreSQL 16 server and client packages
3. Installs Patroni, `python3-etcd`, and `python3-psycopg2`
4. Stops and disables the native `postgresql` systemd service
5. Creates the WAL archive and Patroni configuration directories
6. Deploys `/etc/patroni/patroni.yml` from template (per-node configuration)
7. Deploys the `patroni.service` systemd unit
8. Enables and starts Patroni (which bootstraps PostgreSQL on the first node and joins replicas)
9. Waits for ports 5432 and 8008 to become available
10. Prints cluster status via `patronictl list`

The playbook is fully idempotent. Configuration changes are applied through handlers that restart Patroni only when files actually change.

### Checking Cluster Health

#### Patroni cluster status

The primary tool for cluster health is `patronictl`. Run on any cluster node:

```bash
patronictl -c /etc/patroni/patroni.yml list
```

Expected healthy output:

```
+ Cluster: gitlab-pg-cluster -------+---------+---------+----+-----------+
| Member         | Host         | Role    | State   | TL | Lag in MB |
+----------------+--------------+---------+---------+----+-----------+
| gl-prd-core-01 | 172.16.0.183 | Leader  | running |  1 |           |
| gl-prd-core-02 | 172.16.0.147 | Replica | running |  1 |         0 |
| gl-prd-core-03 | 172.16.0.145 | Replica | running |  1 |         0 |
+----------------+--------------+---------+---------+----+-----------+
```

Verify that:
- Exactly **one** node shows `Role = Leader`
- All nodes show `State = running`
- All replicas are on the same `TL` (timeline) as the leader
- `Lag in MB` is `0` or close to `0`

Or check all nodes remotely via Ansible:

```bash
ansible postgres_cluster -b -m shell -a "patronictl -c /etc/patroni/patroni.yml list"
```

#### Patroni REST API

Each node exposes a REST API on port 8008. Query it for JSON-formatted health status:

```bash
# Returns HTTP 200 on the leader, 503 on replicas
curl -s http://172.16.0.183:8008

# Explicit role endpoints
curl -s http://172.16.0.183:8008/leader    # 200 if this node is leader
curl -s http://172.16.0.183:8008/replica   # 200 if this node is a replica
curl -s http://172.16.0.183:8008/health    # 200 if PostgreSQL is running
```

Check the REST API on all nodes at once:

```bash
ansible postgres_cluster -b -m shell -a "curl -s http://{{ ansible_host }}:8008"
```

#### Systemd service status

```bash
# Single node
ssh gl-prd-core-01 'systemctl status patroni'

# All nodes
ansible postgres_cluster -b -m shell -a "systemctl is-active patroni"
```

#### PostgreSQL replication status

Connect to the **leader** and query replication state:

```bash
psql -h 172.16.0.183 -U postgres -c "SELECT client_addr, state, sent_lsn, replay_lsn, replay_lag FROM pg_stat_replication;"
```

Expected output shows two rows (one per replica), both with `state = streaming`.

#### Check if a node is primary or standby

```bash
psql -h 172.16.0.183 -U postgres -c "SELECT pg_is_in_recovery();"
```

Returns `f` (false) on the leader, `t` (true) on replicas.

#### Patroni logs

```bash
ssh gl-prd-core-01 'journalctl -u patroni --no-pager -n 100'
```

Or tail logs in real-time:

```bash
ssh gl-prd-core-01 'journalctl -u patroni -f'
```

### Updating Patroni / PostgreSQL Configuration

#### Changing Patroni or PostgreSQL parameters

1. Edit the variables in `inventories/production/group_vars/patroni.yml`, or modify the template directly at `roles/patroni/templates/patroni.yml.j2`.

   Common tunables (in `group_vars/patroni.yml`):

   | Variable                         | Default     | Description                        |
   |----------------------------------|-------------|------------------------------------|
   | `postgresql_shared_buffers`      | `256MB`     | PostgreSQL shared memory           |
   | `postgresql_max_wal_senders`     | `5`         | Max concurrent WAL sender processes|
   | `postgresql_max_replication_slots`| `5`        | Max replication slots              |
   | `postgresql_wal_keep_size`       | `1GB`       | WAL retention size                 |
   | `postgresql_network_cidr`        | `172.16.0.0/16` | CIDR allowed in pg_hba         |
   | `postgresql_superuser_password`  | `changeme-super` | Superuser password (use vault) |
   | `postgresql_replication_password`| `changeme-repl`  | Replication password (use vault)|

2. Run the playbook:

   ```bash
   ansible-playbook playbooks/patroni.yml
   ```

   Only nodes with changed config files will be restarted via handlers.

3. Verify the cluster is healthy after the run.

> **Note on DCS-stored parameters:** Some PostgreSQL parameters (like `max_wal_senders`) are managed in Patroni's bootstrap DCS section. Once the cluster is bootstrapped, changing them in the template alone is not enough. Use `patronictl edit-config` to modify DCS-stored parameters on a running cluster:
>
> ```bash
> patronictl -c /etc/patroni/patroni.yml edit-config
> ```

#### Changing pg_hba rules

`pg_hba.conf` is fully managed by Patroni. Rules are defined in the `patroni.yml.j2` template under both `bootstrap.pg_hba` and `postgresql.pg_hba`. To add application access rules:

1. Edit `roles/patroni/templates/patroni.yml.j2` and add entries to the `postgresql.pg_hba` list.
2. Re-run the playbook. Patroni will reload PostgreSQL with the new HBA rules.

### Upgrading Packages

#### Rolling Patroni upgrade

Upgrade Patroni one node at a time, starting with replicas, then the leader:

```bash
# 1. Upgrade on a replica first
ansible gl-prd-core-02 -b -m apt -a "name=patroni state=latest update_cache=yes"
ansible gl-prd-core-02 -b -m systemd -a "name=patroni state=restarted"

# 2. Verify the replica rejoined
ansible gl-prd-core-01 -b -m shell -a "patronictl -c /etc/patroni/patroni.yml list"

# 3. Repeat for the second replica
ansible gl-prd-core-03 -b -m apt -a "name=patroni state=latest update_cache=yes"
ansible gl-prd-core-03 -b -m systemd -a "name=patroni state=restarted"

# 4. Verify again, then upgrade the leader
ansible gl-prd-core-01 -b -m apt -a "name=patroni state=latest update_cache=yes"
ansible gl-prd-core-01 -b -m systemd -a "name=patroni state=restarted"

# 5. Final health check
ansible gl-prd-core-01 -b -m shell -a "patronictl -c /etc/patroni/patroni.yml list"
```

#### Minor PostgreSQL version upgrade (e.g. 16.x to 16.y)

Minor version upgrades are safe and do not require `pg_upgrade`. Follow the same rolling pattern as Patroni upgrades:

```bash
# Upgrade on a replica
ansible gl-prd-core-02 -b -m apt \
  -a "name=postgresql-16,postgresql-client-16 state=latest update_cache=yes"
ansible gl-prd-core-02 -b -m systemd -a "name=patroni state=restarted"

# Verify, then repeat for other nodes
ansible gl-prd-core-01 -b -m shell -a "patronictl -c /etc/patroni/patroni.yml list"
```

After all nodes are upgraded:

```bash
ansible postgres_cluster -b -m shell -a "psql -U postgres -c 'SELECT version();'"
```

#### Major PostgreSQL version upgrade (e.g. 16 to 17)

Major version upgrades require `pg_upgrade` and careful planning. This is **not** handled by the Ansible role and must be done manually. High-level steps:

1. Stop writes to the cluster.
2. Install the new PostgreSQL major version packages on all nodes.
3. Stop Patroni on all nodes.
4. Run `pg_upgrade` on the leader node.
5. Update `postgresql_version` in group vars and role defaults.
6. Re-deploy Patroni configuration with the updated version.
7. Reinitialize replicas from the upgraded leader.

Refer to the [Patroni documentation](https://patroni.readthedocs.io/en/latest/) and [PostgreSQL pg_upgrade docs](https://www.postgresql.org/docs/current/pgupgrade.html) for detailed instructions.

### Failover and Switchover

#### Automatic failover

Patroni handles automatic failover. If the leader becomes unreachable, a replica is promoted within the `ttl` window (30 seconds by default). No manual intervention is required.

#### Manual switchover

To perform a planned switchover (e.g. for maintenance on the current leader):

```bash
# Interactive switchover
patronictl -c /etc/patroni/patroni.yml switchover

# Non-interactive switchover to a specific node
patronictl -c /etc/patroni/patroni.yml switchover \
  --leader gl-prd-core-01 \
  --candidate gl-prd-core-02 \
  --force
```

After the switchover, verify the new topology:

```bash
patronictl -c /etc/patroni/patroni.yml list
```

#### Reinitializing a failed replica

If a replica falls too far behind or its data becomes corrupted:

```bash
patronictl -c /etc/patroni/patroni.yml reinit gitlab-pg-cluster gl-prd-core-03
```

This wipes the replica's data directory and performs a fresh `pg_basebackup` from the leader.

### Maintenance Operations

#### Restarting PostgreSQL via Patroni

Never restart PostgreSQL directly with `systemctl restart postgresql`. Always go through Patroni:

```bash
# Restart PostgreSQL on a specific member (Patroni-managed)
patronictl -c /etc/patroni/patroni.yml restart gitlab-pg-cluster gl-prd-core-02

# Restart PostgreSQL on all members (rolling)
patronictl -c /etc/patroni/patroni.yml restart gitlab-pg-cluster
```

#### Reloading PostgreSQL configuration

```bash
patronictl -c /etc/patroni/patroni.yml reload gitlab-pg-cluster gl-prd-core-01
```

#### Pausing Patroni auto-management

To perform manual maintenance without Patroni interfering:

```bash
# Pause automatic failover
patronictl -c /etc/patroni/patroni.yml pause

# Resume automatic failover
patronictl -c /etc/patroni/patroni.yml resume
```

> **Warning:** While paused, Patroni will not perform automatic failover. Remember to resume after maintenance.

### Credentials

Passwords are stored in `inventories/production/group_vars/patroni.yml` in plain text by default. For production use, encrypt them with Ansible Vault:

```bash
# Encrypt the group vars file
ansible-vault encrypt inventories/production/group_vars/patroni.yml

# Run the playbook with vault
ansible-playbook playbooks/patroni.yml --ask-vault-pass
```

### Key File Locations (on nodes)

| File                               | Description                          |
|------------------------------------|--------------------------------------|
| `/etc/patroni/patroni.yml`         | Patroni configuration                |
| `/etc/systemd/system/patroni.service` | Patroni systemd unit              |
| `/var/lib/postgresql/16/main/`     | PostgreSQL data directory            |
| `/var/lib/postgresql/wal_archive/` | WAL archive directory                |
| `/var/run/postgresql/`             | PostgreSQL unix socket directory     |

### Patroni Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `patronictl list` shows no leader | etcd unreachable or split-brain | Check etcd health first; verify all Patroni nodes can reach etcd on port 2379 |
| Replica shows `Lag in MB` growing | Network issue or high write load | Check network between nodes; check `pg_stat_replication` on leader; consider increasing `wal_keep_size` |
| Patroni won't start, logs show `waiting on etcd` | etcd v2 API not enabled | Ensure `enable-v2: true` is set in `/etc/etcd/etcd.conf.yml` and restart etcd |
| Replica stuck in `start failed` | Stale data directory | Reinitialize the replica: `patronictl reinit gitlab-pg-cluster <member>` |
| `systemctl status patroni` shows `failed` | Configuration error or port conflict | Check `journalctl -u patroni -n 50`; verify `/etc/patroni/patroni.yml` syntax |
| PostgreSQL starts but clients can't connect | pg_hba rules too restrictive | Check `postgresql_network_cidr` variable; re-run playbook after adjusting |
| Timeline divergence after failover | Replica was promoted, old leader rejoined without rewind | Ensure `use_pg_rewind: true` is set (default); reinitialize the affected node if needed |

---

## Redis / Sentinel Cluster

### Architecture

| Setting                | Value                    |
|------------------------|--------------------------|
| Redis port             | `6379`                   |
| Sentinel port          | `26379`                  |
| Sentinel master name   | `mymaster`               |
| Quorum                 | `2`                      |
| Down-after-milliseconds| `5000`                   |
| Failover timeout       | `10000`                  |
| Append-only file       | enabled                  |
| Max memory             | `256mb`                  |
| Max memory policy      | `noeviction`             |
| Configuration          | `/etc/redis/redis.conf`  |
| Sentinel configuration | `/etc/redis/sentinel.conf`|
| Data directory         | `/var/lib/redis`         |

The cluster runs one Redis instance and one Sentinel instance per node. `gl-prd-core-01` is configured as the initial master; the other two nodes start as replicas. Sentinel handles automatic failover — if the master goes down, Sentinel promotes a replica within `down-after-milliseconds + failover-timeout` (roughly 15 seconds).

### Deploying Redis / Sentinel

> **Note:** Redis is independent of etcd and Patroni. It can be deployed at any time, but the recommended order keeps it after the database layer is up.

```bash
ansible-playbook playbooks/redis.yml
```

#### What the playbook does

1. Installs `redis-server` and `redis-sentinel` via apt
2. Deploys `/etc/redis/redis.conf` from template (per-node: master gets no `replicaof`, replicas point to the master dynamically)
3. Deploys `/etc/redis/sentinel.conf` from template (identical monitoring config, per-node `announce-ip`)
4. Enables and starts both `redis-server` and `redis-sentinel` systemd services
5. Waits for ports 6379 and 26379 to become available on every node
6. Prints replication info and Sentinel status

The playbook is fully idempotent. Configuration changes trigger handler-based restarts only when files actually change.

### Checking Cluster Health

#### Quick check — all services running

```bash
ansible redis_cluster -b -m shell -a "systemctl is-active redis-server && systemctl is-active redis-sentinel"
```

Expected output when healthy:

```
gl-prd-core-01 | CHANGED | rc=0 >> active
active
gl-prd-core-02 | CHANGED | rc=0 >> active
active
gl-prd-core-03 | CHANGED | rc=0 >> active
active
```

#### Replication status

Run on any node (or target the current master):

```bash
redis-cli -h 172.16.0.183 -p 6379 INFO replication
```

Healthy master output includes:

```
role:master
connected_slaves:2
slave0:ip=172.16.0.147,port=6379,state=online,offset=...,lag=0
slave1:ip=172.16.0.145,port=6379,state=online,offset=...,lag=0
```

Verify that:
- `role` is `master` on exactly one node
- `connected_slaves` is `2`
- Both replicas show `state=online` and `lag=0` or `lag=1`

To check replication from all nodes at once:

```bash
ansible redis_cluster -b -m shell -a "redis-cli -h {{ ansible_host }} INFO replication | head -5"
```

#### Sentinel status

```bash
# List monitored masters
redis-cli -p 26379 SENTINEL masters

# Get current master address
redis-cli -p 26379 SENTINEL get-master-addr-by-name mymaster

# List known replicas
redis-cli -p 26379 SENTINEL replicas mymaster

# List known sentinels
redis-cli -p 26379 SENTINEL sentinels mymaster
```

Key values to verify in `SENTINEL masters` output:

| Field                | Expected value |
|----------------------|----------------|
| `name`               | `mymaster`     |
| `flags`              | `master`       |
| `num-slaves`         | `2`            |
| `num-other-sentinels`| `2`            |
| `quorum`             | `2`            |

#### Data read/write test

```bash
# Write to master
redis-cli -h 172.16.0.183 SET healthcheck "ok"

# Read from each replica (should return "ok")
redis-cli -h 172.16.0.147 GET healthcheck
redis-cli -h 172.16.0.145 GET healthcheck

# Clean up
redis-cli -h 172.16.0.183 DEL healthcheck
```

#### Logs

```bash
# Redis server logs
ssh gl-prd-core-01 'journalctl -u redis-server --no-pager -n 50'

# Sentinel logs
ssh gl-prd-core-01 'journalctl -u redis-sentinel --no-pager -n 50'
```

### Updating Redis Configuration

#### Changing parameters

1. Edit the variables in `inventories/production/group_vars/redis.yml`, or modify the templates directly in `roles/redis_sentinel/templates/`.

   Common tunables (in `group_vars/redis.yml`):

   | Variable                         | Default       | Description                           |
   |----------------------------------|---------------|---------------------------------------|
   | `redis_maxmemory`                | `256mb`       | Max memory before eviction policy     |
   | `redis_maxmemory_policy`         | `noeviction`  | Eviction policy when maxmemory hit    |
   | `redis_appendonly`               | `yes`         | Enable append-only persistence        |
   | `redis_appendfsync`              | `everysec`    | AOF fsync frequency                   |
   | `redis_sentinel_down_after_ms`   | `5000`        | Time before marking master as down    |
   | `redis_sentinel_failover_timeout`| `10000`       | Failover timeout in milliseconds      |
   | `redis_sentinel_quorum`          | `2`           | Sentinels needed to agree on failover |

2. Run the playbook:

   ```bash
   ansible-playbook playbooks/redis.yml
   ```

   Only nodes with changed configuration files will be restarted via handlers.

3. Verify the cluster is healthy after the run (see checks above).

> **Note on Sentinel config rewrites:** Sentinel modifies its own configuration file at runtime to track discovered replicas and other sentinels. Re-running the playbook re-templates `sentinel.conf` and restarts Sentinel, which then re-discovers the cluster topology within seconds. This is safe and expected.

### Upgrading Redis Packages

The role installs Redis via `apt` with `state: present`, which does not upgrade an already-installed package.

#### Rolling upgrade (recommended)

Upgrade one node at a time, starting with replicas, then the master. This maintains availability throughout the upgrade.

**Step 1 — Upgrade replicas first:**

```bash
# Upgrade the first replica
ansible gl-prd-core-02 -b -m apt -a "name=redis-server,redis-sentinel state=latest update_cache=yes"
ansible gl-prd-core-02 -b -m systemd -a "name=redis-server state=restarted"
ansible gl-prd-core-02 -b -m systemd -a "name=redis-sentinel state=restarted"

# Verify it rejoined as replica
redis-cli -h 172.16.0.147 INFO replication | grep role
# Expected: role:slave

# Upgrade the second replica
ansible gl-prd-core-03 -b -m apt -a "name=redis-server,redis-sentinel state=latest update_cache=yes"
ansible gl-prd-core-03 -b -m systemd -a "name=redis-server state=restarted"
ansible gl-prd-core-03 -b -m systemd -a "name=redis-sentinel state=restarted"

# Verify it rejoined
redis-cli -h 172.16.0.145 INFO replication | grep role
# Expected: role:slave
```

**Step 2 — Failover away from the master, then upgrade it:**

```bash
# Trigger a failover so the master moves to an already-upgraded replica
redis-cli -p 26379 SENTINEL failover mymaster

# Wait a few seconds, then confirm the new master
redis-cli -p 26379 SENTINEL get-master-addr-by-name mymaster

# Now upgrade the old master (which is now a replica)
ansible gl-prd-core-01 -b -m apt -a "name=redis-server,redis-sentinel state=latest update_cache=yes"
ansible gl-prd-core-01 -b -m systemd -a "name=redis-server state=restarted"
ansible gl-prd-core-01 -b -m systemd -a "name=redis-sentinel state=restarted"
```

**Step 3 — Final verification:**

```bash
# Check all nodes are running the new version
ansible redis_cluster -b -m shell -a "redis-server --version"

# Verify cluster health
redis-cli -p 26379 SENTINEL masters
redis-cli -h $(redis-cli -p 26379 SENTINEL get-master-addr-by-name mymaster | head -1) INFO replication
```

### Failover

#### Automatic failover

Sentinel monitors the master with `down-after-milliseconds: 5000`. If the master becomes unreachable for 5 seconds, Sentinels vote (quorum = 2 of 3) and promote a replica. The `failover-timeout` is 10 seconds. Total time from master failure to promotion is roughly 5–15 seconds.

No manual intervention is required.

#### Manual failover

To trigger a planned failover (e.g. for maintenance on the current master):

```bash
# Trigger failover
redis-cli -p 26379 SENTINEL failover mymaster

# Wait ~5 seconds, then verify the new master
redis-cli -p 26379 SENTINEL get-master-addr-by-name mymaster
```

#### Testing failover

1. Identify the current master:

   ```bash
   redis-cli -p 26379 SENTINEL get-master-addr-by-name mymaster
   ```

2. Stop Redis on the master node (e.g. `gl-prd-core-01`):

   ```bash
   ssh gl-prd-core-01 'sudo systemctl stop redis-server'
   ```

3. Wait 5–15 seconds, then check that Sentinel promoted a replica:

   ```bash
   redis-cli -p 26379 SENTINEL get-master-addr-by-name mymaster
   ```

   The returned IP should be one of the replicas.

4. Verify the cluster is functional:

   ```bash
   # Write to the new master
   NEW_MASTER=$(redis-cli -p 26379 SENTINEL get-master-addr-by-name mymaster | head -1)
   redis-cli -h $NEW_MASTER SET failover-test "passed"
   redis-cli -h $NEW_MASTER GET failover-test
   ```

5. Bring the old master back:

   ```bash
   ssh gl-prd-core-01 'sudo systemctl start redis-server'
   ```

   It will automatically rejoin as a replica of the new master.

6. Clean up:

   ```bash
   redis-cli -h $NEW_MASTER DEL failover-test
   ```

### Key File Locations (on nodes)

| File                               | Description                          |
|------------------------------------|--------------------------------------|
| `/etc/redis/redis.conf`           | Redis server configuration           |
| `/etc/redis/sentinel.conf`        | Sentinel configuration (runtime-modified) |
| `/var/lib/redis/`                 | Data directory (RDB + AOF files)     |
| `/var/log/redis/redis-server.log` | Redis server log                     |
| `/var/log/redis/redis-sentinel.log`| Sentinel log                        |

### Redis Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `systemctl status redis-server` shows `inactive` | Service crashed or was stopped | `systemctl start redis-server`, check logs with `journalctl -u redis-server` |
| `connected_slaves` is `0` or `1` on master | Replica down or network issue | Check `systemctl status redis-server` on replicas; verify connectivity on port 6379 between nodes |
| `SENTINEL masters` shows `flags` containing `s_down` or `o_down` | Master unreachable by sentinels | Check if Redis is running on the master node; check network connectivity on ports 6379 and 26379 |
| `num-other-sentinels` is less than `2` | A Sentinel instance is down | Check `systemctl status redis-sentinel` on all nodes; restart any stopped sentinels |
| Replica shows `master_link_status:down` | Master unreachable from replica | Verify the master is running; check network between the replica and master on port 6379 |
| Sentinel keeps failing over repeatedly | Flapping — master is unstable | Check master node health (CPU, memory, disk); increase `down-after-milliseconds` if the master is slow but functional |
| After failover, old master doesn't rejoin as replica | Sentinel couldn't reconfigure it | Restart Redis on the old master: `systemctl restart redis-server`; Sentinel will reconfigure it |
| `MISCONF` errors on writes | AOF/RDB save failed (disk full or permissions) | Check disk space on `/var/lib/redis`; verify `redis` user owns the data directory |

---

## Gitaly Cluster

### Architecture

| Setting                | Value                                  |
|------------------------|----------------------------------------|
| GitLab Omnibus version | 18.8.4-ee                              |
| Gitaly port            | `8075`                                 |
| Listen address         | `0.0.0.0:8075`                         |
| Storage path           | `/var/opt/gitlab/git-data/repositories`|
| Configuration          | `/etc/gitlab/gitlab.rb`                |
| Auth token             | Shared across all nodes                |
| Omnibus role           | `gitaly_role`                          |

Each node runs a standalone Gitaly instance serving one named storage. The nodes are configured using the GitLab Omnibus `roles ['gitaly_role']` mechanism, which enables only Gitaly and disables all other GitLab components (PostgreSQL, Redis, nginx, Puma, Sidekiq, etc.).

| Node           | Storage Name | Address              |
|----------------|-------------|----------------------|
| gl-prd-core-01 | `gitaly-1`  | `172.16.0.183:8075`  |
| gl-prd-core-02 | `gitaly-2`  | `172.16.0.147:8075`  |
| gl-prd-core-03 | `gitaly-3`  | `172.16.0.145:8075`  |

> **Note:** These are independent Gitaly nodes. Praefect (Gitaly Cluster with replication) will be configured in a later step.

### Deploying Gitaly

> **Prerequisite:** The GitLab Omnibus APT repository does not need to be pre-configured — the role handles it automatically.

```bash
ansible-playbook playbooks/gitaly.yml
```

#### What the playbook does

1. Installs prerequisites (`curl`, `gnupg`, `apt-transport-https`)
2. Adds the official GitLab APT repository signing key and sources list
3. Installs the `gitlab-ee` Omnibus package
4. Deploys `/etc/gitlab/gitlab.rb` from template (per-node configuration)
5. Runs `gitlab-ctl reconfigure` (creates the `git` user, storage directories, and starts Gitaly)
6. Verifies storage directory ownership (`git:git`, mode `0700`)
7. Waits for port 8075 to become available on every node
8. Validates Gitaly is running via `gitlab-ctl status gitaly`

The playbook is fully idempotent. Configuration changes trigger `gitlab-ctl reconfigure` only when `gitlab.rb` actually changes.

### Checking Cluster Health

#### Quick check — service status on all nodes

```bash
ansible gitaly_cluster -b -m shell -a "gitlab-ctl status gitaly"
```

Expected output when healthy:

```
gl-prd-core-01 | CHANGED | rc=0 >> run: gitaly: (pid 35799) 120s; run: log: (pid 35710) 134s
gl-prd-core-02 | CHANGED | rc=0 >> run: gitaly: (pid 34816) 120s; run: log: (pid 34732) 134s
gl-prd-core-03 | CHANGED | rc=0 >> run: gitaly: (pid 34767) 121s; run: log: (pid 34695) 134s
```

All three nodes should show `run: gitaly` with an active PID.

#### Verify port is listening

```bash
ansible gitaly_cluster -b -m shell -a "ss -tlnp | grep 8075"
```

Expected output (one line per node):

```
gl-prd-core-01 | CHANGED | rc=0 >> LISTEN 0  4096  *:8075  *:*  users:(("gitaly",pid=...,fd=...))
gl-prd-core-02 | CHANGED | rc=0 >> LISTEN 0  4096  *:8075  *:*  users:(("gitaly",pid=...,fd=...))
gl-prd-core-03 | CHANGED | rc=0 >> LISTEN 0  4096  *:8075  *:*  users:(("gitaly",pid=...,fd=...))
```

#### Check all GitLab-managed services on a node

```bash
ssh gl-prd-core-01 'sudo gitlab-ctl status'
```

On a Gitaly-only node, only `gitaly` and its log process should be listed as `run`. All other services should be absent or disabled.

#### Gitaly health check (from GitLab application server)

Once a GitLab application server is configured to use these Gitaly nodes:

```bash
gitlab-rake gitlab:gitaly:check
```

#### Storage directory verification

```bash
ansible gitaly_cluster -b -m shell -a "ls -la /var/opt/gitlab/git-data/"
```

The `git-data` directory and its `repositories` subdirectory should be owned by `git:git` with mode `0700`.

#### Gitaly logs

```bash
# Tail logs on a specific node
ssh gl-prd-core-01 'sudo gitlab-ctl tail gitaly'

# Last 50 lines
ssh gl-prd-core-01 'sudo gitlab-ctl tail gitaly | head -50'

# All nodes at once
ansible gitaly_cluster -b -m shell -a "gitlab-ctl tail gitaly 2>&1 | tail -5"
```

### Updating Gitaly Configuration

#### Changing parameters

1. Edit the variables in `inventories/production/group_vars/gitaly.yml`, or modify the template directly at `roles/gitaly/templates/gitlab.rb.j2`.

   Common tunables (in `group_vars/gitaly.yml`):

   | Variable                 | Default                    | Description                              |
   |--------------------------|----------------------------|------------------------------------------|
   | `gitaly_auth_token`      | `changeme-gitaly-secret`   | Shared auth token (use vault)            |
   | `gitaly_listen_addr`     | `0.0.0.0`                  | Listen address for Gitaly                |
   | `gitaly_port`            | `8075`                     | Gitaly gRPC port                         |
   | `gitaly_storage_path`    | `/var/opt/gitlab/git-data` | Parent directory for repository storage  |
   | `gitlab_omnibus_package` | `gitlab-ee`                | Omnibus package name (`gitlab-ee` or `gitlab-ce`) |
   | `gitlab_external_url`    | `http://gitlab.example.com`| External URL (required by Omnibus)       |

2. Run the playbook:

   ```bash
   ansible-playbook playbooks/gitaly.yml
   ```

   Only nodes with changed `gitlab.rb` files will run `gitlab-ctl reconfigure` (via handlers).

3. Verify the cluster is healthy after the run (see checks above).

> **Note on `gitlab.rb` format (GitLab 18.x):** The template uses `gitaly['configuration']` with a Ruby hash instead of the legacy `git_data_dirs()` function, which was removed in GitLab 18.0. The storage `path` must include the `/repositories` suffix. See [GitLab migration docs](https://docs.gitlab.com/omnibus/settings/configuration.html#migrating-from-git_data_dirs) for details.

#### Changing the auth token

The auth token must match across all Gitaly nodes **and** the GitLab application server's `gitlab_rails['repositories_storages']` configuration.

1. Update `gitaly_auth_token` in `inventories/production/group_vars/gitaly.yml`.
2. Re-run the playbook to apply to all nodes simultaneously:

   ```bash
   ansible-playbook playbooks/gitaly.yml
   ```

3. Update the GitLab application server configuration to use the new token.

#### Adding a storage name / node

1. Add the new host to the `gitaly_cluster` group in `inventories/production/hosts.yml` with a unique `gitaly_node_name`.
2. Run the playbook — it only affects nodes with changed configuration.
3. Register the new storage on the GitLab application server.

### Upgrading GitLab Omnibus (Gitaly)

The role installs `gitlab-ee` via `apt` with `state: present`, which does not upgrade an already-installed package.

#### Checking current version

```bash
ansible gitaly_cluster -b -m shell -a "dpkg -l gitlab-ee | tail -1 | awk '{print \$3}'"
```

#### Rolling upgrade (recommended)

Upgrade one node at a time to maintain availability. Since each node serves an independent storage, there is no leader/replica relationship to consider, but a rolling approach ensures at least some nodes remain available throughout.

**Step 1 — Upgrade the first node:**

```bash
# Upgrade the package
ansible gl-prd-core-01 -b -m apt -a "name=gitlab-ee state=latest update_cache=yes"

# Reconfigure to apply any new defaults
ansible gl-prd-core-01 -b -m shell -a "gitlab-ctl reconfigure"

# Verify Gitaly is running
ansible gl-prd-core-01 -b -m shell -a "gitlab-ctl status gitaly"

# Verify port is listening
ansible gl-prd-core-01 -b -m shell -a "ss -tlnp | grep 8075"
```

**Step 2 — Repeat for remaining nodes:**

```bash
# Second node
ansible gl-prd-core-02 -b -m apt -a "name=gitlab-ee state=latest update_cache=yes"
ansible gl-prd-core-02 -b -m shell -a "gitlab-ctl reconfigure"
ansible gl-prd-core-02 -b -m shell -a "gitlab-ctl status gitaly"

# Third node
ansible gl-prd-core-03 -b -m apt -a "name=gitlab-ee state=latest update_cache=yes"
ansible gl-prd-core-03 -b -m shell -a "gitlab-ctl reconfigure"
ansible gl-prd-core-03 -b -m shell -a "gitlab-ctl status gitaly"
```

**Step 3 — Final verification:**

```bash
# Confirm all nodes are on the same version
ansible gitaly_cluster -b -m shell -a "dpkg -l gitlab-ee | tail -1 | awk '{print \$3}'"

# Verify all nodes are healthy
ansible gitaly_cluster -b -m shell -a "gitlab-ctl status gitaly"
```

> **Warning:** Major GitLab version upgrades (e.g. 17.x to 18.x) may introduce breaking changes to `gitlab.rb` syntax. Always review the [GitLab upgrade path](https://docs.gitlab.com/ee/update/) and test on a staging environment first. The `grafana['enable']` and `git_data_dirs()` removals in 18.0 are examples of such breaking changes.

### Credentials

The Gitaly auth token is stored in `inventories/production/group_vars/gitaly.yml` in plain text by default. For production use, encrypt it with Ansible Vault:

```bash
# Encrypt the group vars file
ansible-vault encrypt inventories/production/group_vars/gitaly.yml

# Run the playbook with vault
ansible-playbook playbooks/gitaly.yml --ask-vault-pass
```

### Key File Locations (on nodes)

| File                                       | Description                          |
|--------------------------------------------|--------------------------------------|
| `/etc/gitlab/gitlab.rb`                    | GitLab Omnibus configuration         |
| `/etc/gitlab/gitlab-secrets.json`          | Auto-generated secrets (do not edit) |
| `/var/opt/gitlab/git-data/`                | Git storage parent directory         |
| `/var/opt/gitlab/git-data/repositories/`   | Repository storage path              |
| `/var/log/gitlab/gitaly/`                  | Gitaly log directory                 |

### Gitaly Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `gitlab-ctl status gitaly` shows `down` | Service crashed or was stopped | Run `gitlab-ctl start gitaly`; check logs with `gitlab-ctl tail gitaly` |
| `ss -tlnp | grep 8075` shows nothing | Gitaly not listening | Check `gitlab-ctl status gitaly`; run `gitlab-ctl reconfigure`; verify `gitlab.rb` syntax |
| `gitlab-ctl reconfigure` fails with "Reading unsupported config value" | Deprecated or removed setting in `gitlab.rb` | Remove the unsupported setting (e.g. `grafana['enable']` was removed in 18.x); check [GitLab deprecations](https://docs.gitlab.com/ee/update/deprecations) |
| `gitlab-ctl reconfigure` fails with "Removed configurations found" | Using `git_data_dirs()` on GitLab 18.x | Migrate to `gitaly['configuration']` with `storage:` array; see [migration docs](https://docs.gitlab.com/omnibus/settings/configuration.html#migrating-from-git_data_dirs) |
| `gitlab-rake gitlab:gitaly:check` fails | Auth token mismatch or network issue | Verify `gitaly_auth_token` matches between Gitaly nodes and the GitLab application server; check port 8075 connectivity |
| Storage directory has wrong permissions | Ownership changed or `reconfigure` not run | Run `gitlab-ctl reconfigure`; verify `git:git` ownership on `/var/opt/gitlab/git-data` |
| Package install fails with "No package matching gitlab-ee" | GitLab APT repository not configured | Re-run the playbook — it adds the repository automatically; or add manually with `curl https://packages.gitlab.com/install/repositories/gitlab/gitlab-ee/script.deb.sh \| bash` |
