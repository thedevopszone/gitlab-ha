You are a senior DevOps engineer and infrastructure automation expert.

Create a production-ready Ansible role and playbook to deploy a 3-node Redis cluster with Redis Sentinel on Ubuntu 24.04 LTS.

Target nodes:

- gl-prd-core-01 = 172.16.0.183
- gl-prd-core-02 = 172.16.0.147
- gl-prd-core-03 = 172.16.0.145

Cluster requirements:

- 3 Redis nodes
- 1 Redis master
- 2 Redis replicas
- 3 Redis Sentinel instances (one per node)
- Automatic failover
- Internal trusted network (no TLS for now)
- Redis port: 6379
- Sentinel port: 26379

Technical requirements:

1. Create a clean Ansible project structure:

   inventories/
     production/
       hosts.yml
       group_vars/
         redis.yml
   roles/
     redis_sentinel/
       defaults/
       tasks/
       templates/
       handlers/
   playbooks/
     redis.yml

2. Inventory must:

   - Define group "redis_cluster"
   - Assign ansible_host per node
   - Assign redis_node_role (master or replica)
   - Define one node as initial master

3. Role must:

   - Install redis-server via apt
   - Ensure redis service is enabled
   - Configure redis.conf via template
   - Bind to 0.0.0.0
   - Disable protected mode (internal network only)
   - Enable replication on replicas
   - Configure appendonly yes
   - Set reasonable maxmemory policy
   - Be fully idempotent
   - Use handlers for restart

4. Redis replication:

   - Replicas must dynamically reference the master using hostvars
   - No hardcoded IP addresses inside templates
   - Use groups['redis_cluster']

5. Sentinel configuration:

   - One sentinel instance per node
   - Configure sentinel.conf via template
   - Monitor master
   - quorum = 2
   - failover-timeout = 10000
   - down-after-milliseconds = 5000
   - announce-ip dynamically
   - announce-port 26379

6. Systemd:

   - Ensure redis-server.service enabled
   - Ensure redis-sentinel.service enabled
   - Proper After=network.target
   - Restart on failure

7. Verification tasks:

   - Wait for port 6379
   - Wait for port 26379
   - Provide example test commands:

     redis-cli -h <master_ip> INFO replication
     redis-cli -p 26379 SENTINEL masters
     redis-cli -p 26379 SENTINEL get-master-addr-by-name mymaster

   - Provide failover test instructions

8. Important constraints:

   - Do NOT hardcode IPs in templates
   - Use dynamic cluster configuration
   - Clean variable separation
   - Idempotent and production safe

9. Output:

   - Full directory tree
   - All YAML files
   - redis.conf template
   - sentinel.conf template
   - Example ansible-playbook execution command:

     ansible-playbook -i inventories/production/hosts.yml playbooks/redis.yml

