# Elastic Search HA Cluster

## Setup using Ansible and Vagrant

### Prerequisites
1. virtualbox v6+
2. vagrant v2.2.7
3. ansible v2.9.6

#### If you are new to Elasticsearch, I would recommend you to read the following articles before proceeding

https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html, 
https://www.elastic.co/pdf/architecture-best-practices.pdf


#### Steps to procure VMs using Vagrant (Skip this if you already have VMs procured)
1. Procure VMs using Vagrant . Number of VMs can differ based on requirement.
2. This playbook sets up a 4 node cluster and hence 4 vm entries are made in the Vagrantfile.
3. Add or remove entries in the Vagrantfile based on your requirement
4. Set the network of each vm to be a private network and assign an ip from the reserved space.
5. Run your vagrant file

```
vagrant up
```

#### Writing your playbook
1.Set up your inventory first which will be the hosts file or inventory.yml

```
[master_nodes]
node01 ansible_host=192.168.50.2 ansible_user=vagrant ansible_private_key_file=/Users/aravindvvaideesh/elastic-ansible/.vagrant/machines/node0/virtualbox/private_key

[data_nodes]
node02 ansible_host=192.168.50.3 ansible_user=vagrant ansible_private_key_file=/Users/aravindvvaideesh/elastic-ansible/.vagrant/machines/node1/virtualbox/private_key
node03 ansible_host=192.168.50.4 ansible_user=vagrant ansible_private_key_file=/Users/aravindvvaideesh/elastic-ansible/.vagrant/machines/node2/virtualbox/private_key
```
2. From the VMs procured previously, group them into master_nodes and data_nodes as per your requirement.
3. Give in your vm's host, username and private key files. Mentioned above is just a sample template.
4. Import elasticsearch role using ansible-galaxy

```
ansible-galaxy install elastic.elasticsearch,7.7.0
```

5. Now that you have the role imported, set up your site.yml using the imported role

 ```
 - hosts: master_nodes
  roles:
    - role: elastic.elasticsearch


- hosts: data_nodes
  roles:
    - role: elastic.elasticsearch
 ```
 
 This is an ideal and clean way of writing a site.yml. All the configurations should be moved to group variables (group_vars).
 
 6. Moving to the configurations, all variables that will be common to both master and data node groups will go into all.yml
 
**group_vars/all.yml**
```
es_major_version: "7.x"
es_version: "7.6.0"
es_heap_size: "128m"
es_cluster_name: "production"

```

7. Variables specific to master nodes group will go into master_nodes.yml

**group_vars/master_nodes.yml**
```
es_config:
  cluster.initial_master_nodes: "{{ groups['master_nodes'] | map('extract', hostvars, ['ansible_eth1', 'ipv4', 'address']) | join(',') }}"
  cluster.name: "production"
  network.host: "{{ ansible_eth1.ipv4.address }}, _local_"
  discovery.seed_hosts: "{{ groups['master_nodes'] | map('extract', hostvars, ['ansible_eth1', 'ipv4', 'address']) | join(':9300,') }}:9300"
  http.port: 9200
  transport.port: 9300
  node.data: false
  node.master: true
  bootstrap.memory_lock: false
 ```
 Here, I am making use of the ansible fact **ansible_eth1.ipv4.address** which will return the ip address of the host. This could differ based on the network interface.
 
 To get the list of master nodes I am using both ansible facts and ansible magic variables like hostvars which is basically host variables.
 
 8. Variables specific to data nodes group will go into data_nodes.yml
 
**group_vars/data_nodes.yml**
```
es_config:
  cluster.initial_master_nodes: "{{ groups['master_nodes'] | map('extract', hostvars, ['ansible_eth1', 'ipv4', 'address']) | join(',') }}"
  cluster.name: "production"
  network.host: "{{ ansible_eth1.ipv4.address }}, _local_"
  discovery.seed_hosts: "{{ groups['master_nodes'] | map('extract', hostvars, ['ansible_eth1', 'ipv4', 'address']) | join(':9300,') }}:9300"
  http.port: 9200
  transport.port: 9300
  node.data: true
  node.master: false
  bootstrap.memory_lock: false
 ```
 
Ensure to modify the config variables as per your requirement. Notice the **node.data** and **node.master** variables changes for the respective groups. 

#### Running your playbook

```
ansible-playbook -i ./hosts ./site.yml
```


Once the ansible installation completes, your elasticsearch cluster is ready. To verify your cluster please hit the following url either in a browser window on your local system or using curl from one of the VMs.

**http://<ip_address_of_one_node>/_cat/nodes?v**

If the cluster has been successfully set up you should see something like this on the browser window.


ip         | heap.percent | ram.percent | cpu | load_1m | load_5m | load_15m | node.role | master | name
---        |    ---       | ---         |    --- |  ---    |   ---   |  ---     |   ---     |    ---     |  --- |
192.168.50.5 |  75        |     80      |      0      |   0.24  |  0.15    |   0.06    |    dil |    -   |   node04
192.168.50.2 |     52     |     82      | 4  |    0.08 |   0.10 |    0.04 | ilm |    -  |     node01
192.168.50.4 |     77     |     82      | 1  |  0.07  |   0.09  |    0.04 | ilm |   *   |     node03
192.168.50.3 |     55     |     83 |   1 |    0.29 |   0.26 |  0.10 | ilm |   -   |    node02

Here the node that is the current master will have the " * " set to the master column.


 
