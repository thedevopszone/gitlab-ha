You are a senior infrastructure automation engineer.

Create a production-ready Ansible role and playbook to deploy a 3-node PostgreSQL 16 cluster managed by Patroni on Ubuntu 24.04 LTS.

Target nodes:

- gl-prd-core-01 = 172.16.0.183
- gl-prd-core-02 = 172.16.0.147
- gl-prd-core-03 = 172.16.0.145

Assume:
- A working etcd cluster is already running on these three nodes (ports 2379/2380).
- No TLS required for now (internal trusted network).
- PostgreSQL version: 16.

Cluster requirements:

- Cluster name: gitlab-pg-cluster
- Patroni must use etcd as DCS backend
- 3 full voting members
- Automatic leader election
- Streaming replication
- WAL archiving enabled (local only for now)
- Replication user: replicator
- Superuser: postgres
- Data directory: /var/lib/postgresql/16/main
- Listen on all interfaces (0.0.0.0)
- PostgreSQL port: 5432
- Patroni REST API port: 8008

Technical requirements:

1. Create clean Ansible project structure:

   inventories/
     production/
       hosts.yml
       group_vars/
         patroni.yml
   roles/
     patroni/
       defaults/
       tasks/
       templates/
       handlers/
   playbooks/
     patroni.yml

2. Inventory must:

   - Define group "postgres_cluster"
   - Assign ansible_host per node
   - Assign patroni_member_name per host

3. Role must:

   - Install PostgreSQL 16
   - Install Patroni (via apt + pip if necessary)
   - Install required Python dependencies
   - Create replication user
   - Create proper data directories
   - Generate patroni.yml from template
   - Disable native PostgreSQL systemd service
   - Manage PostgreSQL only via Patroni
   - Enable and start Patroni systemd service
   - Be fully idempotent
   - Use handlers for restart

4. Patroni configuration must:

   - Dynamically build etcd endpoints using hostvars
   - Define bootstrap section
   - Configure initdb options
   - Configure authentication (trust for internal only)
   - Define replication settings
   - Define PostgreSQL parameters:
       shared_buffers
       wal_level = replica
       hot_standby = on
       max_wal_senders
       max_replication_slots
       wal_keep_size
   - Configure REST API:
       listen = 0.0.0.0:8008
       connect_address = node_ip:8008

5. PostgreSQL access:

   - pg_hba.conf managed by Patroni
   - Allow replication traffic between cluster nodes
   - Allow application connections from internal network

6. Add verification tasks:

   - Wait for port 5432
   - Wait for port 8008
   - Provide example validation commands:

     patronictl list
     curl http://<node>:8008
     psql -h <leader_ip> -U postgres

7. Important constraints:

   - Do NOT hardcode IPs in templates.
   - Use groups['postgres_cluster'] and hostvars.
   - Ensure correct dependency order.
   - Ensure safe re-runs (idempotency).

8. Output:

   - Full directory tree
   - All YAML files
   - patroni.yml template
   - Example ansible-playbook execution command:

     ansible-playbook -i inventories/production/hosts.yml playbooks/patroni.yml

