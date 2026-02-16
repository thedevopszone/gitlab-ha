# gitlab-ha

High-availability infrastructure for GitLab, managed with Ansible.

## Components

- **etcd** — 3-node cluster providing distributed key-value storage

## etcd Cluster

### Architecture

| Host           | IP             | Role          |
|----------------|----------------|---------------|
| gl-prd-core-01 | 172.16.0.183  | etcd member   |
| gl-prd-core-02 | 172.16.0.147  | etcd member   |
| gl-prd-core-03 | 172.16.0.145  | etcd member   |

- Cluster name: `gitlab-etcd-cluster`
- Client port: `2379`
- Peer port: `2380`
- Data directory: `/var/lib/etcd`
- Configuration: `/etc/etcd/etcd.conf.yml`

### Checking if etcd Is Running

#### Service status on a single node

```bash
ssh gl-prd-core-01 'systemctl status etcd'
```

Or check all three nodes at once via Ansible:

```bash
ansible etcd_cluster -i inventories/production/hosts.yml -b -m shell -a "systemctl is-active etcd"
```

Expected output when healthy:

```
gl-prd-core-01 | CHANGED | rc=0 >> active
gl-prd-core-02 | CHANGED | rc=0 >> active
gl-prd-core-03 | CHANGED | rc=0 >> active
```

#### Cluster endpoint health

Run from any cluster node:

```bash
export ETCDCTL_API=3
etcdctl \
  --endpoints=http://172.16.0.183:2379,http://172.16.0.147:2379,http://172.16.0.145:2379 \
  endpoint health
```

Or remotely via Ansible:

```bash
ansible gl-prd-core-01 -i inventories/production/hosts.yml -b -m shell -a \
  "ETCDCTL_API=3 etcdctl --endpoints=http://172.16.0.183:2379,http://172.16.0.147:2379,http://172.16.0.145:2379 endpoint health"
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
   ansible-playbook -i inventories/production/hosts.yml playbooks/etcd.yml
   ```

   The role is idempotent. Only nodes with changed configuration files will be restarted (via handlers).

3. Verify the cluster is healthy after the run (see checks above).

#### Upgrading the etcd package

The role installs etcd via `apt`. To upgrade to the latest version available in the Ubuntu repository:

1. Run the playbook with `state: latest` by overriding the variable at the command line:

   ```bash
   ansible-playbook -i inventories/production/hosts.yml playbooks/etcd.yml
   ```

   > **Note:** The role uses `state: present` by default, which does not upgrade an already-installed package. To perform an upgrade, temporarily change `state: present` to `state: latest` in `roles/etcd/tasks/main.yml`, or run the upgrade manually one node at a time as described below.

2. **Rolling upgrade (recommended):** Upgrade one node at a time to maintain quorum throughout. For each node:

   ```bash
   # Upgrade a single node
   ansible gl-prd-core-01 -i inventories/production/hosts.yml -b -m apt \
     -a "name=etcd-server state=latest update_cache=yes"

   # Restart etcd on that node
   ansible gl-prd-core-01 -i inventories/production/hosts.yml -b -m systemd \
     -a "name=etcd state=restarted"

   # Verify cluster health before moving to the next node
   ansible gl-prd-core-01 -i inventories/production/hosts.yml -b -m shell -a \
     "ETCDCTL_API=3 etcdctl --endpoints=http://172.16.0.183:2379,http://172.16.0.147:2379,http://172.16.0.145:2379 endpoint health"
   ```

   Repeat for `gl-prd-core-02` and `gl-prd-core-03`.

3. After all nodes are upgraded, confirm the running version:

   ```bash
   ansible etcd_cluster -i inventories/production/hosts.yml -b -m shell -a "etcd --version"
   ```

### Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `systemctl status etcd` shows `inactive` | Service crashed or was stopped | `systemctl start etcd`, check logs with `journalctl -u etcd` |
| `endpoint health` shows one node unhealthy | Network issue or node down | Check connectivity on port 2380 between nodes, restart etcd on the affected node |
| `member list` shows only 2 members | A member was removed or failed to join | Re-add the member with `etcdctl member add`, then re-run the playbook |
| Cluster won't start after full outage | All members trying to join as `new` | Ensure `etcd_initial_cluster_state` is set to `existing` if data directories are intact, or wipe `/var/lib/etcd` on all nodes and redeploy with `new` |
