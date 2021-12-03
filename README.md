logging-elk
=========

Role for logging playbook. It will install logging setup with ELK ( Elasticsearch, Logstash and Kibana). The playbook has two options for log collector - filebbeat and rsyslog. 

Role Variables
--------------

All variables are defined in defaults/main.yaml

1. namespace -  logging_namespace: logging

2. operation_loggingsetup - Possible value - install, upgrade and uninstall. This will create the namespace, storage class, PVC, PV, index template etc etc. 
This is always set to install, but once the PVC etc are created it will not modify them again and again, hence retaining the data. Set it to uninstall only when you have to delete the setup completely, including the namespace, data etc.
The PV's are created with deletion policy, which basically means when we will delete the PVC, PV will automatically be deleted.
So be careful when setting this flag to uninstall.

3. operation - Possible value - install and uninstall. The default value is install. This flag will install ELK deployments. Use uninstall to delete the ELK and log collector setup. Setting this flag to uninstall will not delete PVC.

4. logcollector - Possible value - rsyslog and filebeat. The default value is rsyslog, which means if not specified, the setup will be deployed with rsyslog as log collector and forwarder. 
Rsyslog makes changes in docker daemon file, restart docker, rsyslog.conf files, restart rsyslog. If rsyslog is used we have to manually delete the pods in ces namespace after the playbook has run to get latest applied docker changes into effect. This part has not been included in playbook, to avoaid any issues in application setup.
If the flag is set to filebeat, filebeat will be deployed as a daemonset. So on each node on filebeat pod will be running. No need to restart the application pods in this one.

Dynamic Inventory Creation 
--------------------------

1. Create dynamic Inventory via ip address

ansible-playbook -i hosts dynamicinventory.yml -e"{"K8s_Master_IP": $k8s_master_ip}"

The above command will create dynamic inventory in current directory named hosts_logging. The rest of the logging playbook will use that as inventory file

2. Delete Dynamic Inventory

rm hosts_logging

Example Playbook
----------------

1. Install with filebeat as log collector

ansible-playbook -i hosts_logging logging-elk.yml -e"{"operation": install}" -e"{"logcollector": filebeat}"

2. Uninstall with filebeat as log collector

ansible-playbook -i hosts_logging logging-elk.yml -e"{"operation": uninstall}" -e"{"logcollector": filebeat}" 

3. Completely uninstall the logging setup including namespace, SC, PVc etc etc 

ansible-playbook -i hosts_logging logging-elk.yml -e"{"operation": uninstall}" -e"{"operation_loggingsetup": uninstall}" -e"{"logcollector": filebeat}"

4. Install with rsyslog as log collector 

ansible-playbook -i hosts_logging logging-elk.yml -e"{"operation": install}" -e"{logcollector: rsyslog}" 

5. Uninstall with rsyslog as log collector

ansible-playbook -i hosts_logging logging-elk.yml -e"{"operation": uninstall}" -e"{logcollector: rsyslog}"

6. Completely uninstall the logging setup including namespace, SC, PVC etc etc

ansible-playbook -i hosts_logging logging-elk.yml -e"{"operation": uninstall}" -e"{"operation_loggingsetup": uninstall}" -e"{"logcollector": rsyslog}"

NOTE: Recommended when changing mode of logcollector, basically switching between them, one should entirely delete the setup and PVs etc and recreate as both ways uses some reserved fields which will conflict leading in elasticsearch to not accept fresh data.


NOTE
----

1. Since rsyslog configuration changes needs to be applied on all VMs and not just master, the inventory host file should contain details of all master nodes under [kube-master] group and worked node details under [kube-node]. A final k8s group whose children are kube-master and kube-node

Sample inventory file 

[kube-master]
master-node-1 ansible_host=1.1.1.1 ip=1.1.1.1 access_ip=1.1.1.1
master-node-2 ansible_host=2.2.2.2 ip=2.2.2.2 access_ip=2.2.2.2
[kube-node]
worker-node-1 ansible_host=3.3.3.3 ip=3.3.3.3 access_ip=3.3.3.3
worker-node-2 ansible_host=4.4.4.4 ip=4.4.4.4 access_ip=4.4.4.4
worker-node-3 ansible_host=5.5.5.5 ip=5.5.5.5 access_ip=5.5.5.5
[k8s:children]
kube-master
kube-node


The logging-elk is applied on final k8 hosts group to include all nodes.