You are a senior DevOps infrastructure automation expert.

Create a production-ready Ansible role and playbook to deploy a 3-node Praefect cluster on Ubuntu 24.04 LTS.

Target nodes:

- gl-prd-core-01 = 172.16.0.183
- gl-prd-core-02 = 172.16.0.147
- gl-prd-core-03 = 172.16.0.145

Assumptions:

- Gitaly is already installed and listening on port 8075 on all three nodes.
- PostgreSQL cluster (Patroni) is running and reachable.
- No TLS for now.
- GitLab Omnibus is installed.
- Redis is external.

Cluster requirements:

- 3 Praefect nodes (one per core node)
- Cluster name: gitlab-praefect-cluster
- Virtual storage name: default
- Replication factor: 3
- Praefect listen address: 0.0.0.0:2305
- Shared auth token across all nodes
- PostgreSQL backend DB name: praefect_production
- PostgreSQL connection via Patroni leader or DB HA endpoint

Technical requirements:

1. Create clean Ansible project structure:

   inventories/
     production/
       hosts.yml
       group_vars/
         praefect.yml
   roles/
     praefect/
       defaults/
       tasks/
       templates/
       handlers/
   playbooks/
     praefect.yml

2. Inventory must:

   - Define group "praefect_cluster"
   - Assign ansible_host per node
   - Assign praefect_node_name per host

3. Role must:

   - Ensure Omnibus installed
   - Enable praefect in gitlab.rb
   - Disable:
       postgresql
       redis
       nginx
       puma
       sidekiq
   - Configure praefect:
       praefect['enable'] = true
       praefect['listen_addr']
       praefect['auth_token']
       praefect['virtual_storages']
   - Define virtual storage named "default"
   - Register all gitaly nodes dynamically using hostvars
   - Configure PostgreSQL backend connection
   - Use Patroni endpoint variable (no hardcoded IP)
   - Run gitlab-ctl reconfigure via handler
   - Be fully idempotent

4. Praefect configuration must dynamically build:

   - Gitaly node list
   - Storage names
   - DB connection string
   - Shared token

5. Ensure PostgreSQL database exists:

   - Create praefect_production database if not present
   - Create praefect user if not present
   - Grant proper permissions

6. Verification tasks:

   - Wait for port 2305
   - Validate service:

       gitlab-ctl status praefect
       ss -tulpen | grep 2305

   - Provide cluster validation examples:

       gitlab-rake praefect:metadata:check
       gitlab-rake gitlab:gitaly:check

7. Constraints:

   - Do NOT hardcode IPs
   - Use groups['gitaly_cluster']
   - Use hostvars for IP references
   - Use centralized variable for praefect_auth_token
   - Fully idempotent
   - Production safe re-runs

8. Output required:

   - Full directory tree
   - All YAML files
   - gitlab.rb template
   - Example execution command:

     ansible-playbook -i inventories/production/hosts.yml playbooks/praefect.yml

