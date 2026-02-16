You are a senior DevOps infrastructure automation expert.

Create a production-ready Ansible role and playbook to install GitLab Omnibus (Application Layer only) on Ubuntu 24.04 LTS and connect it to:

- External PostgreSQL cluster managed by Patroni
- External Redis cluster with Sentinel
- External Praefect cluster for Gitaly

Target nodes:

- gl-prd-app-01
- gl-prd-app-02

Assumptions:

- PostgreSQL cluster is already running and reachable
- Redis Sentinel cluster is already running
- Praefect cluster is already running
- Internal trusted network (no TLS for internal services)
- External URL: https://gitlab.example.local

Cluster requirements:

- 2 Application nodes
- Stateless Rails layer
- No local PostgreSQL
- No local Redis
- No local Gitaly
- No internal NGINX (Loadbalancing handled separately)
- Shared secrets across both nodes

Technical requirements:

1. Create clean Ansible project structure:

   inventories/
     production/
       hosts.yml
       group_vars/
         gitlab_app.yml
   roles/
     gitlab_app/
       defaults/
       tasks/
       templates/
       handlers/
   playbooks/
     gitlab_app.yml

2. Inventory must:

   - Define group "gitlab_app"
   - Assign ansible_host per node

3. Role must:

   - Install GitLab Omnibus (latest stable)
   - Ensure required packages installed
   - Configure gitlab.rb via template
   - Disable:
       postgresql
       redis
       gitaly
   - Enable:
       puma
       sidekiq
   - Configure external_url
   - Configure DB connection:
       gitlab_rails['db_host']
       gitlab_rails['db_port']
       gitlab_rails['db_username']
       gitlab_rails['db_password']
       gitlab_rails['db_database']
   - Configure Redis Sentinel:
       gitlab_rails['redis_sentinels']
       gitlab_rails['redis_master_name']
   - Configure Praefect:
       gitlab_rails['gitaly_token']
       gitlab_rails['gitaly_address']
   - Run gitlab-ctl reconfigure via handler
   - Be fully idempotent

4. Configuration constraints:

   - Use group_vars for:
       db_host
       redis_sentinels
       praefect_nodes
       shared secrets
   - Do NOT hardcode IP addresses
   - Use dynamic hostvars for cluster references
   - Both app nodes must share identical:
       gitlab-secrets.json
       secret_key_base
       gitaly token

5. Generate shared secrets file:

   - Ensure secrets are generated once
   - Reused across both nodes
   - Managed via Ansible
   - Not regenerated on every run

6. Systemd:

   - Ensure gitlab-runsvdir enabled
   - Ensure gitlab services started
   - Restart only on config change

7. Verification tasks:

   - Wait for port 8080 or 80/443 (depending on config)
   - Validate:

       gitlab-ctl status
       curl http://localhost/users/sign_in

   - Provide final access test example:

       curl -k https://gitlab.example.local

8. Security notes:

   - No internal TLS required for DB/Redis/Praefect
   - Only external HTTPS considered
   - Ensure proper firewalling

9. Output required:

   - Full directory tree
   - hosts.yml
   - group_vars/gitlab_app.yml
   - gitlab.rb template
   - tasks/main.yml
   - handlers/main.yml
   - Example execution command:

     ansible-playbook -i inventories/production/hosts.yml playbooks/gitlab_app.yml

