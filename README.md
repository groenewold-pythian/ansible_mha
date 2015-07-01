# Ansible_MHA
#####The playbook install MySQL instances, set up replication and configure MHA 
Currently it supports the following:
 - Linux user management 
 - Automated master-slave replication setup. You can add new slaves to a cluster online(xtrabackup stream) 
 - Semi-automated failover support with MHA
 - my.cnf file creation based on available ram with templates
    - custom my.cnf setup 
    - MySQL user management

###How to install

Requirements:
  - ssh client
  - ansible
  - vagrant
  - virtualbox

add your credentials to the encrypted system.yml (default password: 42)
```
ansible-vault edit group_vars/system.yml
```
you can change the password with
```
ansible-vault rekey group_vars/system.yml
```
###Spin up instances 
Spin up 4 instances, we will use them as:
 - 1 master
 - 2 slaves 
 - 1 MHA manager node

Set up ssh passwordless authentication from the host you are running ansible

Vagrant box: chef/centos-6.5

Define at least these tags:
 - Name = anything

Edit the sample_cluster inventory file and change the ip addresses
```
~/github/ansible_mha$ cat clusters/sample_cluster
#ROLES
======
[system]
52.28.26.105   mycnf=plsc_s1 mysql_role=master  nickname=dev-plsc-m
52.28.23.4    mycnf=plsc_s2 mysql_role=slave  nickname=dev-plsc-s1
52.28.19.44   mysql_role=slave  nickname=dev-plsc-s2

52.28.3.145   mysql_role=mha_manager

[system:vars]
cluster_id=dev-plscdemo

[databases:vars]
master_host=52.28.26.105

# Group to install and configure MySQL
[databases]
52.28.26.105  mysql_role=master server_id=1
52.28.23.4   mysql_role=slave server_id=2 backup_src=52.28.26.105
52.28.19.44  mysql_role=slave server_id=3 backup_src=52.28.26.105
```

run the playbook 
```sh
ansible-playbook -i clusters/sample_cluster setup.yml -e bootstrap_enabled=true --ask-vault-pass
```

#####All servers has read_only=1 set by default!!!

###MHA examples
run these command on the ansible host:
#####Connectivity test
```
ansible  -i clusters/sample_cluster system --limit  [ip_of_mha_manager_node]   -a "masterha_check_ssh --conf=/etc/mha_dev-plscdemo.cnf"
```
#####Replication test
```
ansible  -i clusters/sample_cluster system --limit  [ip_of_mha_manager_node]   -a "masterha_check_repl  --conf=/etc/mha_dev-plscdemo.cnf"
```

#####Online failover
You can see the current topology in the output of the replication test. 
```sh
ansible  -i clusters/sample_cluster system --limit  [ip_of_mha_manager_node]   -a "masterha_master_switch --master_state=alive --orig_master_is_new_slave --conf=/etc/mha_dev-plscdemo.cnf  --interactive=0 --new_master_host=[ip address of the chosen slave ]"
```
#####Failover with a dead master
```
ansible  -i clusters/sample_cluster system --limit  [ip_of_mha_manager_node]   -a "masterha_master_switch --master_state=dead --conf=/etc/mha_dev-plscdemo.cnf  --dead_master_host=[ip of the dead master] --interactive=0 --remove_dead_master_conf"
```

###TAGS
#####'users' - User management
If you want to add a new Linux user to the servers simply copy its public key to roles/common/files/users/ -  filename will be the username. 
Once the file is deployed the users can connect to the servers with their own username and can sudo.
If you want to delete a user, simply remove the file and run the playbook again.
Run the following to apply the changes:
```
ansible-playbook -i clusters/sample_cluster setup.yml --tags users 
```


