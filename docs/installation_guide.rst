################
Installing EDCOP
################

Prerequisites
=============
In order to begin to install EDCOP the following will be needed:

#. Two or more servers that support the following specs:

- A processor and chipset supporting VT-X and VT-D.  These must be enabled in the BIOS.  Refer to your system manufacturer for this.  Dual processor systems are ideal due to to NUMA enhancements.  The more cores the better.
- Two network interfaces, one for the PXE network that will be used to build minions and another interface for the internal and external network traffic using Calico.  PXE network should be completely isolated and non-routable.  The Internal network must be connected to the Internet at this time.  Isolated environments will be possible with some work to bring externally hosted resources into the network.
- If you are planning on implementing network sensors such as Suricata or Bro, at least 2x nic ports supporting SR-IOV if using Passive only configuration and 4x nic ports if using inline and passive.  Currently only the Intel XL710 has been tested but others should work.  For an Intel list of supported cards see here: https://www.intel.com/content/www/us/en/support/articles/000005722/network-and-i-o/ethernet-products.html.  
- At least two disk volumes, one will install the OS and the other will install additional storage for log analysis and other applications
- Minimum two cores per host.
- Minimum of 16GB of Ram per physical host.  Depending on applications deployed this number will need to be a lot higher.  16 GB will get you the Kubernetes cluster, elasticsearch and one or two tools.  More memory is always better.  Warning:  Kubernetes does not allow for Swap memory, this means that if you start to run out of memory, pods will be killed and weird things can begin to happen.  Monitor memory to ensure you aren't using more than 70% of memory.  The included Cockpit tool is useful for this.
- Minimum two cores per host, recommend at least four.
- A USB thumb drive of 8 GB or more
- Servers should be identical configurations if possible, the Intel network card should be plugged into the exact same PCI-E slot.
- Servers must be set to UEFI in the BIOS

A master server will be chosen and be given a slightly different network configuration than all minions.  The master will be installed using a USB thumb drive.  The minons will be installed using PXE off of the master server.

Network configuration
=====================

EDCOP requires a specific network configuration to work properly.  A switch will need to be configured with the following VLANS:

- One vlan to support PXE booting.  This will be tied to all systems and
- One vlan to support the Calico Overlay network which will allow the containers in the Kubernetes cluster to communicate.  The master will optionally be connected to this VLAN over a LAGG connection and the minions will have a single port connected to this this.  This VLAN must have a route to your users as well as the Internet.

For supporting network sensors (Bro, Suricata, Moloch, etc), additional network interfaces using SR-IOV will be required.  Two NICs for inline and one additional for passive.  Passive and inline can be used together and will require three total interfaces.

For network deployment options see here https://github.com/sealingtech/EDCOP/blob/master/docs/network_design_and_deployment.rst

DNS Requirements
================
EDCOP requires two DNS entries. When you setup the master you will be asked for the fully qualified domain name (FQDN).  The same domain name must be entered into the DNS server pointing to the master system.  In addition, there must be a wild card subdomain under that same FQDN.  The wild card DNS entry can be pointed either at the master as well or it can be load balanced by an external load balancer to the master and all minions on 80 and 443.  Load balancing is optional

Example:

You name the master, master.edcop.io and give it the IP of 192.168.1.10.  You create a wild card subdomain of *.master.edcop.io of 192.168.1.10.

It is necessarry to be able to resolve these entries from both inside the cluster as well as any clients accessing the cluster.  You must access the cluster by using the domain name and not by IP.


What if I don't have a DNS server?
==================================
It is still possible to make it work, just not ideal.  It is possible to add entries to your /etc/hosts or on Windows (https://support.rackspace.com/how-to/modify-your-hosts-file).  Unfortunately the hosts file doesn't work with wild cards, so it will be necessary to add multiple entries.

Currently there are five deployed with EDCOP:
- https://admin.<fqdn>/
- https://kubernetes.<fqdn>/
- https://loadbalancer.<fqdn>/
- https://apps.<fqdn>/
- https://ceph.<fqdn>/

These will need to be pointed at one of the servers (usually the master).  When adding new capabilities that have an Ingress (for example Kibana) you will need to add a new entry to your hosts file in order to deploy this capability.


Building ISO image
==================
Note- This step can be skipped if you don't want to build by hand.  We reccomend you simply download the latest ISO from the releases section https://github.com/sealingtech/EDCOP/releases.

You will need a workstation with Git, Docker and Make installed in order to build the image.  So far it has been tested to build with both Mac and Linux.  Docker can be installed here https://docs.docker.com/install/

First step will be to build an ISO to install the Master server.  To do this a Docker container has been created that will pull the latest CENTOS and EDCOP packages and build the ISO file necessary to install the master.  

From the workstation run the following commands:

.. code-block:: bash

  git clone https://github.com/sealingtech/EDCOP
  cd EDCOP/
  make iso

Once this process is done you will have an ISO image created in the EDCOP/ directory.  

NOTE: This process will erase EVERYTHING ON THE THUMB DISK!  Make sure there is nothing on the disk.  Make sure you select the correct disk!

To write the ISO to a thumb drive:

On Mac replace the # with the proper number displayed in diskutil list:

.. code-block:: bash

  diskutil list #This will show the list of disks on your systems, find the correct thumb drive disk path (will look something like /dev/disk<#>)
  sudo diskutil unmountDisk /dev/disk<#>
  sudo dd if=EDCOP-dev.iso of=/dev/disk<#>

On Linux

.. code-block:: bash

  fdisk -l #find the name of the device corresponding to your thumb drive
  umount <path to thumb drive>
  sudo dd if=EDCOP-dev.iso of=<path to thumb drive> bs=1m

On Windows (probably works, use Rufus?)

Installing EDCOP
================

When the Master is coming up, ensure it boots to the Thumb drive.  Select "Install the Expandable DCO Platform".

NOTE: This procedure will ERASE everything on the master node and the minion nodes once they are PXE booted.  YOU HAVE BEEN WARNED!

You will be asked if you want to select the default network configuration, generally you will need to select "N" at this point.

#. Enter the hostname, this must be an FQDN and match the DNS record entered earlier.
#. You will be printed an interface list, select Y to team the interfaces if you plan on implemeting LAGG or N if you only are going to use a single interface for the host.  (Note at this time you must use a CAPITAL LETTER)
#. Enter in the name of the interface you want to use for the main network if you selected N on the teaming question.  If you answered yes enter in the name of the interfaces seperated by commas for all interfaces included in the LAGG
#. Select if your network uses DHCP
#. Enter in the IP address to assign the master (Note, this must match the IP given to the DNS entry)
#. Enter the netmask
#. Enter the gateway
#. Enter the DNS server
#. Enter the interface name of the PXE boot interface
#. Enter in the IP address of the PXE interface.  This will be a non-routed network
#. Enter in the netmask of the PXE interface
#. Enter in the last octet of the starting IP (For example, if your IP address was 10.50.50.10 and you enter in 100 here then your starting IP will be 10.50.50.100)
#. Enter in the last octet of the ending IP
#. Enter Y to accept defaults for network_configuration
#. You will be presented with the disks available on your system.  There will be a number by each of these, Enter in the number of the disk corresponding to the disk you would like to install the OS on as well as assign storage.  These drives can be the same for each option or they can be different.
#. Enter in the number you would like to install the rest of the data to

After this process is completed, the master will reboot. You can logon with root and the password open.local.box

There is a systemctl process that runs on first boot, to see the status of this run the command:

.. code-block:: bash

  systemctl status EDCOP-firstboot

Wait until this process is over, the Active setting will go to "inactive (dead)" once this process is completed

To change the root password run the command:

.. code-block:: bash

  passwd

Enter the password twice.


Accessing Cockpit
=================

If you have configured the DNS entry correctly, then Cockpit should be available at this point.  Open a web browser and go to:

https://admin.<fqdn>/


Logon with root as the user and the password you set earlier

Building the Minions
====================

Once the master is successfully running, minions can be PXE booted off of the main system.  This is not needed on single node deployments.

Boot off of the PXE Interface in startup (see system manual for this process)

If the PXE is configured correctly, an Install the Expandable DCO Platform (EDCOP) option will be displayed, select Enter

After the installation process is completed and the system reboots.  Access cockpit and select Cluster -> Nodes and your new node should appear here after a bit and the status should be set to ready.

From the command line, it is also possible to do this from the command line on the master using:

.. code-block:: bash

  kubectl get nodes


Labeling nodes
==============

Nodes must be given roles in order to take certain tasks.  Each of these labels must be applied somwhere throughout the cluster.  For small deployments, simply label the master as all of them.  For larger deployments it is possible to selectively apply the labels to specific nodes throughout the cluster.


.. code-block:: bash

  node=<name of node>
  kubectl label node $node nodetype=worker
  kubectl label node $node sensor=true
  kubectl label node $node data=true
  kubectl label node $node infrastructure=true
  kubectl label node $node ingest=true


Please see the node labelling guide  https://github.com/sealingtech/EDCOP/blob/master/docs/node_labels.rst


Verifying installation
======================

After a few minutes all the pods should be either in a "running" or "completed" state.  To verify these have come up, run the command.  

.. code-block:: bash
 
  kubectl get pods --all-namespaces



Accessing other Services
========================

EDCOP has deployed a number of internal web inferfaces automatically for you.  To view these:

- https://admin.<fqdn>/
- https://kubernetes.<fqdn>/
- https://loadbalancer.<fqdn>/
- https://apps.<fqdn>/
- https://ceph.<fqdn>/

Please view the ingress guide https://github.com/sealingtech/EDCOP/blob/master/docs/ingress_design.rst for more details.


SSL Certificate Management
==========================

By default EDCOP will create a wild card certificate that is used by all domains.  This certificate has been signed by an auto-generated Certificate Authority (CA) that is used for internal CA operations.  This CA is generally not trusted by your browser.  To make SSL error messages go away a user can trust the internal kubernetes certificate authority.  

The certificate is stored in /root/ca.cer and can be added to user's internal Root CA store.

For windows follow this guide:
https://blogs.technet.microsoft.com/sbs/2008/05/08/installing-a-self-signed-certificate-as-a-trusted-root-ca-in-windows-vista/


Deploying Capabilities
======================

To deploy additional tools users can go to apps.<fqdn> and select the applications to they want to deploy.  Selecting "Available Capabilities" will bring up a number of charts that can then be deployed.  Each chart will have built in instructions.  Many of these charts values are set to defaults that will work with smaller deployments but more planning is required for larger deployments to get more performance out of the tools.  When you deploy capabilities with a web front end, make sure that you change the ingress host option.  

For example with kibana:

.. code-block:: bash

  ingress:
    #Enter the subdomain and the FQDN that will be used to access Kibana
    host: kibana.edcop.io  
    
If you don't set these, the chart will deploy but won't be reachable by you.  
    
Change the host option to be a subdomain under your DNS domain you created earlier.  To get better performance out of many of these tools it will be necessarry to carefully plan out resources.  Please view the optimization guide for more details.

https://github.com/sealingtech/EDCOP/blob/master/docs/optimization_guide.rst


Proper shutdown procedure
=========================

EDCOP utiilizes the shared storage solution Rook (https://rook.io).  Because of this, special care should be taken when powering down nodes.  To drain nodes, the proper procedure is as follows:

From the master server, get a list of all minions and then run the kubectl drain command below for the minion you want to shut down.  If this procedure is not followed hosts will not power down properly and data loss may occur.

.. code-block:: bash

  kubectl get minions
  kubectl drain <each minion, one at a time if shutting down more than one> --ignore-daemonsets --delete-local-data
  kubectl drain <the master goes last if you are shutting down the master as well> --ignore-daemonsets --delete-local-data

Once this procedure is done it is safe to run shutdown now to power down each host.  

When powering up the hosts, services will not start until the host is "uncordoned".  To do this, uncordon the master first, then each of the minions.

.. code-block:: bash

  kubectl uncordon <name of master>
  kubectl uncordon <name of each minion>
  
After this process it may take services a few minutes to start up normally.


If Elasticsearch is deployed, the proper method for shutting down nodes is to first disable shard allocation to ensure that the cluster doesn't attempt to recover.

To perform the procedure from the master node, get the IP address of the data-service:

.. code-block:: bash

  [root@virtual ~]# kubectl get service
  NAME                               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                                        AGE
  data-service                       ClusterIP   10.111.51.141    <none>        9200/TCP,9300/TCP                              49m


Run the fllowing curl command against the data-service IP from any node in the cluster.

.. code-block:: bash

  curl -X PUT "<ip of the data-service>:9200/_cluster/settings" -H 'Content-Type: application/json' -d'
  {
    "persistent": {
      "cluster.routing.allocation.enable": "none"
    }
  }
  '

Once maintenece is complete re-enable the allocation of shards:

.. code-block:: bash

  curl -X PUT "<ip of the data-service>:9200/_cluster/settings" -H 'Content-Type: application/json' -d'
  {
    "persistent": {
      "cluster.routing.allocation.enable": null
    }
  }
  '

Monitor Kibana to ensure the cluster goes from yellow to green after a few minutes.


