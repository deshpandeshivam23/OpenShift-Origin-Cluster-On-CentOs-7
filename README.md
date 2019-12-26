# OpenShift-Origin-Cluster-On-CentOs-7
Deploy MultiNode OpenShift-Origin 3.9 Cluster On CentOs-7


	Deploying OpenShift Origin 3.9  Cluster On CentOs-7 Step-by-Step : 
	


# Lab Setup: 

	Hostname                             IP	       CPUs     RAM	     	HDD		        OS	  Role
	master.openshift.example.com  	192.168.43.20	 2	  2 GB	  /dev/sda (75 GB) 	CentOS-7	Master-Node
	node1.openshift.example.com	192.168.43.21	 1	  1 GB	  /dev/sda (50 GB) 	CentOS-7	Worker-Node 
	node2.openshift.example.com	192.168.43.22	 1	  1 GB	  /dev/sda (50 GB) 	CentOS-7	Worker-Node
	infra.openshift.example.com	192.168.43.23	 1	  1 GB	  /dev/sda (50 GB) 	CentOS-7	infra-Node

	Note : Above Lab Setup is According to my Laptop Configuration. You can Change as per Your requirements.

			
				# 	Preparing Nodes: 	#

Step 1: Set the hostname:
*************************

      # hostnamectl set-hostname master.openshift.example.com	# For master node

  Use the above command to set the hostname for other nodes accordingly. 

       # exec bash 

Step 2: Configure Static IP Address: 
************************************

    [root@master ~]# vim /etc/sysconfig/network-scripts/ifcfg-ens33 

          TYPE="Ethernet"
          BOOTPROTO="none"
          IPADDR=192.168.43.20
          NETMASK=255.255.255.0
          GATEWAY=192.168.0.1
          DEFROUTE="yes"
          IPV6INIT="no"
          NAME="ens33"
          DEVICE="ens33"
          ONBOOT="yes"

:wq (save and exit) 

The above IP settings for "master" Node, use the same settings for other nodes accordingly. 
Don't forget to changes the IP Address for nodes.  

Down the Network Connection, reload the configuration and up it again on all nodes. 

          # nmcli connection down ens33 
          # nmcli connection reload
          # nmcli connection up ens33 


Configure /etc/hosts file for name resolution as following on all nodes: 

    # vim /etc/hosts

          192.168.43.20		master.openshift.example.com	master
          192.168.43.21		node1.openshift.example.com		node1
          192.168.43.22		node2.openshift.example.com		node2
          192.168.43.23		infra.openshift.example.com		infra



Step 3: Use the below command to update the System on all nodes: 

      # yum update -y

Use the below command to compare the kernel on all nodes: 
    
      # uname -r
      
If you have different kernel in above command output, then you need to reboot all the system. otherwise jump to Step 4. 

      # reboot 


Step 4: Once the systems came back UP/ONLINE, Install the following Packages on all nodes:  

    # yum install -y wget git vim net-tools docker-1.13.1 bind-utils iptables-services bridge-utils bash-completion   kexec-tools sos psacct openssl-devel httpd-tools NetworkManager python-cryptography python-devel python-passlib java-1.8.0-openjdk-headless "@Development Tools"

Step 5: Configure Ansible Repository on master Node only. 
*********************************************************

    # vim /etc/yum.repos.d/ansible.repo

      [ansible]
      name = Ansible Repo
      baseurl = https://releases.ansible.com/ansible/rpm/release/epel-7-x86_64/
      enabled = 1
      gpgcheck =  0


      :wq (save and exit) 



Step 6: Start and Enable NetworkManager and Docker Services on all nodes: 
*************************************************************************


      # systemctl start NetworkManager
      # systemctl enable NetworkManager
      # systemctl status NetworkManager

      # systemctl start docker && systemctl enable docker && systemctl status docker



Step 7: Install Ansible Package and Clone Openshift-Ansible Git Repo on Master Machine :
****************************************************************************************


        # yum -y install ansible-2.6* pyOpenSSL
        # git clone https://github.com/openshift/openshift-ansible.git
        # cd openshift-ansible && git fetch && git checkout release-3.9


Step 8: Generate SSH Keys on Master Node and install it on all nodes: 
*********************************************************************

        # ssh-keygen -f ~/.ssh/id_rsa -N '' 
        # for host in master.openshift.example.com node1.openshift.example.com node2.openshift.example.com ifra.openshift.example.com ; do ssh-copy-id -i ~/.ssh/id_rsa.pub $host; done


Step 9: Now Create Your Own Inventory file for Ansible as following on master Node:  

      # vim ~/inventory.ini 

              [OSEv3:children]
              masters
              nodes
              etcd


              [masters]
              master.openshift.example.com openshift_ip=192.168.43.20

              [etcd]
              master.openshift.example.com openshift_ip=192.168.43.20

              [nodes]
              master.openshift.example.com openshift_ip=192.168.43.20 openshift_schedulable=true
              node1.openshift.example.com openshift_ip=192.168.43.21 openshift_schedulable=true openshift_node_labels="{'region': 'primary', 'zone': 'default'}"
              node2.openshift.example.com openshift_ip=192.168.43.22 openshift_schedulable=true openshift_node_labels="{'region': 'primary', 'zone': 'default'}"
              infra.openshift.example.com openshift_ip=192.168.43.23 openshift_schedulable=true openshift_node_labels="{'region': 'infra', 'zone': 'default'}"

              [OSEv3:vars]
              debug_level=4
              ansible_ssh_user=root
              openshift_enable_service_catalog=true
              ansible_service_broker_install=true

              containerized=false
              os_sdn_network_plugin_name='redhat/openshift-ovs-multitenant'
              openshift_disable_check=disk_availability,docker_storage,memory_availability,docker_image_availability

              openshift_node_kubelet_args={'pods-per-core': ['10']}

              deployment_type=origin
              openshift_deployment_type=origin

              openshift_release=v3.9.0
              openshift_pkg_version=-3.9.0
              openshift_image_tag=v3.9.0
              openshift_service_catalog_image_version=v3.9.0
              template_service_broker_image_version=v3.9.0
              osm_use_cockpit=true

              # put the router on dedicated infra1 node
              openshift_hosted_router_selector='region=infra'
              openshift_master_default_subdomain=apps.openshift.example.com
              openshift_public_hostname=master.openshift.example.com

              # put the image registry on dedicated infra1 node
              openshift_hosted_registry_selector='region=infra'


              :wq (save and exit) 



Step 10: Use the below ansible playbook command to check the prerequisites to deply OpenShift Cluster on master Node: 
    
    # cd openshift-ansible
    # ansible-playbook -i ~/inventory.ini playbooks/prerequisites.yml


Step 11: Once prerequisites completed without any error use the below ansible playbook to Deploy OpenShift Cluster on master Node: 

    # ansible-playbook -i ~/inventory.ini playbooks/deploy_cluster.yml


Now you have to wait approx 30-40 Minutes to complete the Installation.( Its Depend on Internet Speed ) 


Step 12: Once the Installation is completed, Create a admin user in OpenShift with Password "Redhat" from master Node:  

Install httpd-tools on master

    # htpasswd -c /etc/origin/master/htpasswd admin

        New password: Redhat
        Re-type new password: Redhat

    # ls -l /etc/origin/master/htpasswd
    # cat /etc/origin/master/htpasswd


Open /etc/origin/master/master-config.yaml file and do the following changes: 

            identityProviders:
            - challenge: true
              login: true
              mappingMethod: claim
              name: allow_all
              provider:
                apiVersion: v1
                kind: HTPasswdPasswordIdentityProvider		## This line
                file: /etc/origin/master/htpasswd			## This line

	
      :wq (save and exit) 


Restart the below services: 

      # systemctl restart origin-master-controllers.service 
      # systemctl restart origin-master-api.service



Step 13: Use the below command to assign cluster-admin Role to admin user: 

      # oc adm policy add-cluster-role-to-user cluster-admin admin


Step 14: Use the below command to login as admin user on CLI: 

      # oc login
          Authentication required for https://master.openshift.example.com:8443 (openshift)
          Username: admin
          Password: Redhat
          Login successful.

You have access to the following projects and can switch between them with 'oc project <projectname>':

            * default
              kube-public
              kube-system
              logging
              management-infra1
              openshift
              openshift-infra1
              openshift-node
              openshift-web-console

Using project "default".


        # oc whoami
            admin


Step 15: Use below command to list the projects, pods, nodes, Replication Controllers, Services and Deployment Config. 

      # oc get projects
      
        NAME                    DISPLAY NAME   STATUS
        default                                Active
        kube-public                            Active
        kube-system                            Active
        logging                                Active
        management-infra1                       Active
        openshift                              Active
        openshift-infra1                        Active
        openshift-node                         Active
        openshift-web-console                  Active


    # oc get nodes 

        NAME                           STATUS    ROLES     AGE       VERSION
        master.openshift.example.com   Ready     master    29m       v1.9.1+a0ce1bc657
        node1.openshift.example.com    Ready     compute   22m       v1.9.1+a0ce1bc657
        node2.openshift.example.com    Ready     compute   22m       v1.9.1+a0ce1bc657
        infra.openshift.example.com    Ready     <none>    22m       v1.9.1+a0ce1bc657


     # oc get pod --all-namespaces

          NAMESPACE               NAME                          READY     STATUS    RESTARTS   AGE
          default                 docker-registry-1-krmpx       1/1       Running   0          15m
          default                 registry-console-1-z9sbd      1/1       Running   0          14m
          default                 router-1-q7g5v                1/1       Running   0          15m
          openshift-web-console   webconsole-5f649b49b5-hr9dq   1/1       Running   0          14m


      # oc get rc 

          NAME                 DESIRED   CURRENT   READY     AGE
          docker-registry-1    1         1         1         15m
          registry-console-1   1         1         1         14m
          router-1             1         1         1         16m


       # oc get svc 

          NAME               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                   AGE
          docker-registry    ClusterIP   172.30.34.127    <none>        5000/TCP                  16m
          kubernetes         ClusterIP   172.30.0.1       <none>        443/TCP,53/UDP,53/TCP     35m
          registry-console   ClusterIP   172.30.62.197    <none>        9000/TCP                  15m
          router             ClusterIP   172.30.141.143   <none>        80/TCP,443/TCP,1936/TCP   16m


       # oc get dc

           NAME               REVISION   DESIRED   CURRENT   TRIGGERED BY
          docker-registry    1          1         1         config
          registry-console   1          1         1         config
          router             1          1         1         config




Now you can access OpenShift Cluster using Web Browser as following: 


https://master.openshift.example.com:8443 

		Username: admin
		Password: Redhat


#################################################################################
