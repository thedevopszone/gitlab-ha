You are a senior DevOps engineer and infrastructure automation expert.

Create a production-ready Ansible role and playbook to deploy a 3-node etcd cluster on Ubuntu 24.04 LTS.

Target nodes:

- gl-prd-core-01 = 172.16.0.183
- gl-prd-core-02 = 172.16.0.147
- gl-prd-core-03 = 172.16.0.145

Cluster requirements:

- Cluster name: gitlab-etcd-cluster
- Initial cluster state: new
- Peer port: 2380
- Client port: 2379
- No TLS (internal trusted network only)
- Full quorum (all 3 nodes voting members)

Technical requirements:

1. Create a clean Ansible project structure:

   inventories/
     production/
       hosts.yml
       group_vars/
         etcd.yml
   roles/
     etcd/
       defaults/
       tasks/
       templates/
       handlers/
   playbooks/
     etcd.yml

2. Inventory must:

   - Define group "etcd_cluster"
   - Assign ansible_host per node
   - Assign etcd_member_name per host

3. Role requirements:

   - Ensure etcd is installed via apt (Ubuntu 24.04 compatible)
   - Create etcd system user (non-login)
   - Create /var/lib/etcd with correct ownership
   - Generate configuration file via template
   - Use systemd service management
   - Enable and start etcd
   - Use handlers for restart
   - Be idempotent

4. etcd configuration requirements:

   Each node must dynamically configure:

   - ETCD_NAME
   - ETCD_DATA_DIR
   - ETCD_LISTEN_PEER_URLS
   - ETCD_LISTEN_CLIENT_URLS
   - ETCD_INITIAL_ADVERTISE_PEER_URLS
   - ETCD_ADVERTISE_CLIENT_URLS
   - ETCD_INITIAL_CLUSTER
   - ETCD_INITIAL_CLUSTER_STATE
   - ETCD_INITIAL_CLUSTER_TOKEN

   Use hostvars and groups['etcd_cluster'] to dynamically construct the initial cluster string.

   Do NOT hardcode IPs inside templates.

5. Systemd:

   - Provide a custom unit file if necessary
   - Ensure service restarts on config change
   - Ensure proper ordering (After=network.target)

6. Add verification tasks:

   - Wait for port 2379
   - Provide example commands:

     export ETCDCTL_API=3
     etcdctl --endpoints=http://172.16.0.183:2379,http://172.16.0.147:2379,http://172.16.0.145:2379 endpoint health
     etcdctl member list

7. Output:

   - Complete directory tree
   - All YAML files
   - All templates
   - Example execution command:

     ansible-playbook -i inventories/production/hosts.yml playbooks/etcd.yml

