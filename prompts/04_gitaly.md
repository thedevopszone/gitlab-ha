You are a senior DevOps infrastructure automation expert.

Create a production-ready Ansible role and playbook to deploy a 3-node Gitaly cluster on Ubuntu 24.04 LTS.

Target nodes:

- gl-prd-core-01 = 172.16.0.183
- gl-prd-core-02 = 172.16.0.147
- gl-prd-core-03 = 172.16.0.145

Assumptions:

- GitLab Omnibus is already installed on these nodes.
- PostgreSQL and Redis are external.
- Praefect will be configured later.
- Internal trusted network (no TLS for now).

Cluster requirements:

- 3 independent Gitaly nodes
- Each node provides one storage backend
- Storage path: /var/opt/gitlab/git-data
- Gitaly listen address: 0.0.0.0:8075
- Unique auth token shared across nodes
- No Praefect configuration in this step

Technical requirements:

1. Create clean Ansible project structure:

   inventories/
     production/
       hosts.yml
       group_vars/
         gitaly.yml
   roles/
     gitaly/
       defaults/
       tasks/
       templates/
       handlers/
   playbooks/
     gitaly.yml

2. Inventory must:

   - Define group "gitaly_cluster"
   - Assign ansible_host per node
   - Assign gitaly_node_name per host

3. Role must:

   - Ensure GitLab Omnibus package is installed
   - Create storage directory:
       /var/opt/gitlab/git-data
   - Set correct permissions for git user
   - Generate gitlab.rb snippet dynamically via template
   - Enable only Gitaly component
   - Disable:
       postgresql
       redis
       nginx
       puma
       sidekiq
   - Set:
       gitaly['listen_addr']
       gitaly['auth_token']
       git_data_dirs
   - Run gitlab-ctl reconfigure via handler
   - Be fully idempotent

4. Configuration constraints:

   - Use shared variable for gitaly_auth_token
   - Do NOT hardcode IPs
   - Use hostvars and inventory data
   - Each node must define its own storage name
   - Prepare config to be compatible with future Praefect cluster

5. Verification tasks:

   - Wait for port 8075
   - Validate socket or port listening
   - Provide example test commands:

       ss -tulpen | grep 8075
       gitlab-ctl status gitaly

6. Output required:

   - Full directory tree
   - hosts.yml
   - group_vars/gitaly.yml
   - role files
   - gitlab.rb template
   - Example execution command:

     ansible-playbook -i inventories/production/hosts.yml playbooks/gitaly.yml

