# gitlab-ha

High-availability infrastructure for GitLab, managed with Ansible.

## Components

| Component  | Role     | Description                                      |
|------------|----------|--------------------------------------------------|
| etcd       | `etcd`   | 3-node distributed key-value store (DCS)         |
| PostgreSQL | `patroni`| 3-node PostgreSQL 16 cluster managed by Patroni  |

## Nodes

| Host           | IP            | etcd | PostgreSQL / Patroni |
|----------------|---------------|:----:|:--------------------:|
| gl-prd-core-01 | 172.16.0.183  |  x   |          x           |
| gl-prd-core-02 | 172.16.0.147  |  x   |          x           |
| gl-prd-core-03 | 172.16.0.145  |  x   |          x           |

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
│           └── patroni.yml               # Patroni / PostgreSQL variables
├── roles/
│   ├── etcd/
│   │   ├── defaults/main.yml
│   │   ├── tasks/main.yml
│   │   ├── templates/
│   │   │   ├── etcd.conf.yml.j2
│   │   │   └── etcd.service.j2
│   │   └── handlers/main.yml
│   └── patroni/
│       ├── defaults/main.yml
│       ├── tasks/main.yml
│       ├── templates/
│       │   ├── patroni.yml.j2
│       │   └── patroni.service.j2
│       └── handlers/main.yml
└── playbooks/
    ├── etcd.yml
    └── patroni.yml
```

## Deployment Order

The services must be deployed in the following order because Patroni depends on etcd for leader election and cluster state:

```
1. etcd       →  ansible-playbook playbooks/etcd.yml
2. patroni    →  ansible-playbook playbooks/patroni.yml
```

> Both playbooks use the default inventory defined in `ansible.cfg`. You can also specify it explicitly with `-i inventories/production/hosts.yml`.

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
