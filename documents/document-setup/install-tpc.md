# Install wazuh clusters 

# Wazuh dashboard nodes

| Wazuh Role  | IP Addres      | Hostname                 | vCPU    | RAM       | Disk  |  
|-------------| ---------------|--------------------------|---------|-----------|-------|
| indexer     | 10.48.16.15    | tpc-waz-index01          | 8vCPU   | 12G RAM   | 100G  | 
| indexer     | 10.48.16.16    | tpc-waz-index02	        | 8vCPU   | 12G RAM   | 100G  |
| indexer     | 10.48.16.17    | tpc-waz-index03	        | 8vCPU   | 12G RAM   | 100G  |
| master      | 10.48.16.18    | tpc-waz-master01	        | 4vCPU   | 4G RAM    | 30G   |
| worker02    | 10.48.16.19    | tpc-waz-worker02	        | 4vCPU   | 4G RAM    | 30G   |
| worker03    | 10.32.192.10   | tpchc-waz-worker03	      | 4vCPU   | 4G RAM    | 30G   |
| dashboard   | 10.48.16.11    | tpc-waz-dash01  	        | 4vCPU   | 4G RAM    | 30G   |
| nginx       | 10.48.16.10    | tpc-waz-nginx            | 1vCPU   | 2G RAM    | 30G   |
| ansible     | 10.32.192.9    | hc-waz-ansible           | 4vCPU   | 4G RAM    | 30G   |

*** chuẩn bị các node theo cấu hình 
  # update patch
      apt update

    # resize disk all nodes
        echo 1>/sys/class/block/sda/device/rescan
        #Fix disk
        parted -l
        #increase disk
        df -h | grep -v containerd | grep -v kubelet
        growpart /dev/sda 3
        lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
        resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv
        df -h | grep -v containerd | grep -v kubelet
    

# I. Install wazuh-ansible
***Make sure that all K8s root's home nodes have Ansible's public ssh key added.***
## 1. Clone wazuh git repo
```shell
mkdir -p /opt/wazuh && cd /opt/wazuh/ 
#git clone https://github.com/wazuh/wazuh-ansible.git
git clone --branch v4.7.4 https://github.com/wazuh/wazuh-ansible.git
cd wazuh-ansible
ls
```

## 2. Add ssh key to Ansible
```shell
cp ssh/id_rsa* /root/.ssh/
chmod 400 /root/.ssh/id_rsa
eval "$(ssh-agent -s)"
ssh-add /root/.ssh/id_rsa
```
## 3. Add ansible's public key to all node
***Do on all k8s node (Master and Worker node)***
```shell
echo "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQDgl4Co26TD4HO+ag++Ktv90m2rQg+wdIBkuRYn3r3+4vEXac3lUPM0ZvppeZ2UbvUIJjJ04CIJHJleLP8dUqj4qJb9VwQOw4ymdt50GVTud9cFX38UGqJtBwhz1TUf14/2qBXCVAfXYtTXHoGtgCL8HzCWaT1gtelXtF5Xl7Jhtc+OGjq025sC0OmHmejVfPOYifVaJbyQ4lScw8xGDPdSfFFICM1X2ZhDfppwwMSWeDWQCRLwRqJpTPzqjBrUQcYPpoZIV1g2PdO5EV8pLIFYBGrfMsmkyd06wgDCkRG6YczqwOh/JnPRtKSKwo3o3aBcn/pVcIGRX/5oc70EwO+TEKPeVmDLZ8raxZiOLiaBsMqWjWpbMURtrHCWXHw9eCfU98cT0BulH5lJ7sKCCC0oeTAX00JS6YRNXDc7YR/P9ptsiwTvR9fIQDfRuvRDjz+WxHo75zvZH9PfmkZ9L3Y2CqVsePTALTsvJE/8mOFGAIbtENlNRoMs5KlVPJWzizIKrN5dKMI4R5nvP0SuOGgMffSmiWEu2PeBiwsxwovXl1Znhz64AqBxBYRoAy7mGtuWZPGmLxtu7be2yr/Ghl5iOFdxkpZZRU4sHW6fgPUUi4OMOXeOBeqbDU+UG70VuaWcYAFRUB8/UbwFuNWwGz1R2KK9fbu++lsF/C/WZy5cZw== root@ansible" > /root/.ssh/authorized_keys
chmod 600 /root/.ssh/authorized_keys
```
## 4. Set file hosts name and ip for ansible nodes
```shell
vim /etc/hosts

10.48.16.11 tpc-waz-dash01
10.48.16.15 tpc-waz-index01
10.48.16.16 tpc-waz-index02
10.48.16.17 tpc-waz-index03
10.48.16.18 tpc-waz-master01
10.48.16.19 tpc-waz-worker02
10.32.192.10 tpchc-waz-worker03
10.32.192.9 hc-waz-ansible
10.48.16.10 tpc-waz-nginx
```
## 5. Install certificate for all nodes

```shell
cd /opt/wazuh
curl -sO https://packages.wazuh.com/4.7/wazuh-certs-tool.sh
curl -sO https://packages.wazuh.com/4.7/config.yml
vim config.yml
```
# change content config.yml
```shell
### content config.yml ##
nodes:
  # Wazuh indexer nodes
  indexer:
    - name: node-1      # Importance hold node name equa with the certificate name of node sample: node-1.pem , node-1-key.pem
      ip: 10.48.16.15
    - name: node-2
      ip: 10.48.16.16
    - name: node-3
      ip: 10.48.16.17

  # Wazuh server nodes
  # If there is more than one Wazuh server
  # node, each one must have a node_type
  server:
    - name: node-4
      ip: 10.48.16.18
      node_type: master
    - name: node-5
      ip: 10.48.16.19
      node_type: worker
    - name: node-7
      ip: 10.32.192.10
      node_type: worker
    #- name: wazuh-2
    #  ip: "<wazuh-manager-ip>"
    #  node_type: worker
    #- name: wazuh-3
    #  ip: "<wazuh-manager-ip>"
    #  node_type: worker

  # Wazuh dashboard nodes
  dashboard:
    - name: node-6
      ip: 10.48.16.11
```
# gen certificate for all nodes cluster
```shell
  cd /opt/wazuh
  bash wazuh-certs-tool.sh -A

  # cp wazuh-certificates/*.* wazuh-ansible/indexer/certificates/
  mkdir wazuh-ansible/indexer/certificates/ -p
  cp /opt/wazuh/wazuh-certificates /opt/wazuh/wazuh-ansible/indexer/certificates/ -r
```
# (option) advance certificate with exits Root CA

```shell
bash wazuh-certs-tool.sh -A /path/to/root-ca.pem /path/to/root-ca.key
```
[Wazuh Certificates deployment](https://documentation.wazuh.com/current/user-manual/manager/certificates.html)

# ////



## II. install ansible in hc-waz-ansible  
    # 1. install ansible
    
    apt-add-repository -y ppa:ansible/ansible
    apt-get update

    apt-get install ansible tree -y

## 2. change playbook index
    cd /opt/wazuh/wazuh-ansible

    vim playbooks/wazuh-indexer.yml
    
      node1:
        name: tpc-waz-index01
        ip: 10.48.16.15
        role: indexer
      node2:
        name: tpc-waz-index02
        ip: 10.48.16.16
        role: indexer
    #  node3:
    #    name: node-3
    #    ip: <node-3 IP>
    #    role: indexer

vim roles/wazuh/wazuh-indexer/defaults/main.yml

single_node: false
.... ~~
indexer_cluster_nodes:
  #- 127.0.0.1
  - 10.48.16.15
  - 10.48.16.16
indexer_discovery_nodes:
  #- 127.0.0.1
  - 10.48.16.15
  - 10.48.16.16

...
generate_certs: true


# Minimum master nodes in cluster, 2 for 3 nodes Wazuh indexer cluster
minimum_master_nodes: 2

# Configure hostnames for Wazuh indexer nodes
# Example es1.example.com, es2.example.com
#domain_name: wazuh.com
domain_name: wazuh.datx.com.vn


## 3. change playbook dashboard
vim playbooks/wazuh-dashboard.yml

---
- hosts: tpc-waz-dash01
  roles:
    - role: ../roles/wazuh/wazuh-dashboard
  vars:
    ansible_shell_allow_world_readable_temp: true
---
- hosts: tpc-waz-dash02
  roles:
    - role: ../roles/wazuh/wazuh-dashboard
  vars:
    ansible_shell_allow_world_readable_temp: true


vim roles/wazuh/wazuh-dashboard/defaults/main.yml

# Dashboard configuration
indexer_http_port: 9200
indexer_api_protocol: https
dashboard_conf_path: /etc/wazuh-dashboard/
dashboard_node_name: tpc-waz-dash01
dashboard_node_name: tpc-waz-dash02
dashboard_server_host: "0.0.0.0"
dashboard_server_port: "443"
dashboard_server_name: "dashboard"
wazuh_version: 5.0.0
indexer_cluster_nodes:
  - 10.48.16.15
  - 10.48.16.16
# The Wazuh dashboard package repository
dashboard_version: "5.0.0"

# API credentials
wazuh_api_credentials:
  - id: "default"
    url: "https://10.48.16.18"
    port: 55000
    username: "wazuh-admin"
    password: "KKwAdtsHIEyBuYIV4xlI0Xly4"

# Dashboard Security
dashboard_security: true
indexer_admin_password: "ZnDWLEXm1crATnLPrTYAwXbHT"
dashboard_user: wazuh-dashboard
dashboard_password: "dmU00dI84KawOYceb5bHYI8d9"
local_certs_path: "{{ playbook_dir }}/indexer/certificates"

## 4. Testing cấu hình tập trung 
touch playbooks/wazuh-indexer-and-dashboard.yml
vim playbooks/wazuh-indexer-and-dashboard.yml

- hosts: wazuh-dashboard
  roles:
    # Roles for Dashboard nodes
  vars:
    instances:
      tpc-waz-dash01:
        name: tpc-waz-dash01
        ip: 10.48.16.11
        role: dashboard
      tpc-waz-dash02:
        name: tpc-waz-dash02
        ip: 10.48.16.12
        role: dashboard

- hosts: wazuh-indexer
  roles:
    # Roles for Indexer nodes
  vars:
    instances:
      tpc-waz-index01:
        name: tpc-waz-index01
        ip: 10.48.16.15
        role: indexer
      tpc-waz-index02:
        name: tpc-waz-index02
        ip: 10.48.16.16
        role: indexer

- hosts: wazuh-master
  roles:
    # Roles for Master node
  vars:
    instances:
      tpc-waz-master01:
        name: tpc-waz-master01
        ip: 10.48.16.18
        role: master

- hosts: wazuh-workers
  roles:
    # Roles for Worker nodes
  vars:
    instances:
      tpc-waz-worker02:
        name: tpc-waz-worker02
        ip: 10.48.16.19
        role: worker
      tpchc-waz-worker03:
        name: tpchc-waz-worker03
        ip: 10.32.192.10
        role: worker

## III. Let's run the playbook.

##Switch to the playbooks folder on the Ansible server and proceed to run the command below:


    ansible-playbook wazuh-indexer-and-dashboard.yml -b -K

      ansible-playbook \
    -i inventory/tpc-wazuh/hosts.yaml \
    --become \
    --become-user=root \
    tpc-waz-index-dash.yml

    ansible-playbook \
    -i inventory/tpc-wazuh/hosts.yaml \
    --become \
    --become-user=root \
    tpc-waz-index-dash_new.yml