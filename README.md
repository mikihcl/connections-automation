# HCL Connections and Component Pack Automation Scripts

This set of scripts is able to spin up end-to-end HCL Connections 7 with Component Pack and all the dependencies. They can be used as whole and set up end to end environment, including the set of fake users for a sake of quickly being able to log in and see how the application works, or they can can be used autonomously from each other.  

For HCL Connections 7 dependencies this means that:

* IBM DB2 will be installed, configured as per Performance tunning guide for HCL Connections, and license applied.
* HCL Connections Wizard will populate the database with needed schemas and grants.
* If needed for demo or even production purposes, OpenLDAP will be spun up and seeded with some demo users. OpenLDAP will be spun up with SSL enabled, as needed later for setting up IBM WebSphere Application Server properly.
* IBM TDI will be installed, configured, and run to populate profiles database in IBM DB2 with users from OpenLDAP
* IBM Installation Manager will be set up on the nodes where IBM WebSphere Application Server Network Deployment needs to be installed.
* IBM WebSphere Application Server Network Deployment will be set up where needed. Currently we tested it with Fixpack 16 and 18. By default, FP18 is going to be installed. Deployment manager and nodeagents profiles are going to be created, application security enabled, TLS certificated imported from LDAP, LDAP configured up to the point where it is ready to install HCL Connections 7.
* IBM HTTP Server is going to be installed, patched with the same fixpack as IBM WebSphere Application Server, and added to the deployment manager.
* NFS server will be installed, including master and clients configurations and proper folders set.

For HCL Connections 7 itself it means:

* The script will mount NFS shares needed for the shared data and message stores where needed.
* IBM DB2
* HCL Connections 7 will be downloaded and installed. Any type of layout is supported and customizable.
* In LotusConnections-config.xml dynamicHost will be updated.
* Optionaly, Prometheus JMX exported will be enabled for all HCL Connections clusters.
* Optionally, Moderation can be enabled as well.
* In case of upgrades, it will clean up temp folders to prevent possible issues with UI post upgrade.
* All or some (or none) clusters will be started automatically.
* IBM HTTP Server plugins will get generated and propagated
* IBM HTTP Server's httpd.conf will be generated ready to support Component Pack integrations.
* Needed post installation tasks on IBM WebSphere Application Server, like setting up properly session management and single sign on will be also handled.

For Component Pack for HCL Connections 7 it means:

* Nginx will be set up and configured to support Customizer.
* Haproxy will be set up configured to be the control plane for Kubernetes cluster and Component Pack.
* NFS will be set up for Component Pack.
* docker-ce 19.08.13 will be set up and configured for overlay2 and systemd support with the optimisations required by the version of Kubernetes.
* Docker Registry will be set up on all future Kubernetes nodes, including enabling and handling TLS.
* Kubernetes 1.18.10 will be set up.
* Component Pack will be set up by default using latest community Kubernetes Ingress, Grafana and Prometheus for monitoring out of the box.
* Post installation tasks needed for configuring Component Pack and the WebSphere-side of Connections to work together are also going to be executed, including enabling searches and Metrics using ElasticSearch 7.

## Requirements:

* Ansible 2.9 installed on your machine
* SSH access (passwordless makes stuff easier, but not mandatory) to the machine(s) you want to provision. Be sure that key forwarding is properly set on your controller and on all machines that you plan to manage with.
* Your user (or any user used for the purposes of provisioning) having passwordless sudo acces on the nodes being provisioned.

To learn more about how to install Ansible on your local machine or Ansible controller, please check out [official Ansible documentation](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html).

Supported OSs:

* CentOS 7+
* RHEL 7+

NOTE: The supported operating systems listed above have been tested by HCL. These scripts may run on other operating systems as well.

### Have files ready for download

To be able to use this automation you will need to be able to download the packages.

The suggestion is to have them all downloaded in a single location, and for this you would need at least 50G of disk space. Run a small HTTP server just to be able to serve them, it can be as simple as a single Ruby one liner to open web server on specific port so that automation can connect and download it.

This is the example data folder structure we are following at HCL

```
[root@c7lb1 packages]# ls -la *
Connections6.5:
total 3185512
drwxr-xr-x.  2 root orion         88 Sep 11 12:34 .
drwxr-xr-x. 13 root orion        192 Nov 18 08:33 ..
-rw-r--r--.  1 root orion 2444339200 Apr 23  2020 HCL_Connections_6.5_lin.tar
-r-xr-xr-x.  1 root orion  817623040 Apr 28  2020 HCL_Connections_6.5_wizards_lin_aix.tar

Connections7:
total 3551476
drwxr-xr-x.  3 dmenges orion        173 Nov 19 10:30 .
drwxr-xr-x. 13 root    orion        192 Nov 18 08:33 ..
-r-xr-xr-x.  1 dmenges orion  817623040 Aug  4 06:03 HCL_Connections_6.5_wizards_lin_aix.tar
-r-xr-xr-x.  1 root    root  2001274880 Nov 19 10:30 HCL_Connections_7.0_lin.tar
-r-xr-xr-x.  1 root    root   817807360 Oct 29 10:00 HCL_Connections_7.0_wizards_lin_aix.tar
drwxr-xr-x. 13 root    root         246 Oct  1 18:58 Wizards
-rw-r--r--.  1 dmenges orion        133 Nov 19 10:30 current.version

DB2:
total 2067052
drwxr-xr-x.  2 dmenges orion           96 Nov 19 11:01 .
drwxr-xr-x. 13 root    orion          192 Nov 18 08:33 ..
-rw-r--r--.  1 dmenges dmenges    3993254 Oct 16 13:13 CNB23ML.zip
-rw-r--r--.  1 dmenges orion    250880000 Jun  3 10:48 db2jdbc.tar
-rw-r--r--.  1 dmenges orion   1861783964 Apr 23  2020 v11.1.4fp5_linuxx64_universal_fixpack.tar.gz

Docs:
total 1397484
drwxr-xr-x.  2 root orion        78 Sep  7 11:31 .
drwxr-xr-x. 13 root orion       192 Nov 18 08:33 ..
-r-xr-xr-x.  1 root orion 737753769 Sep  7 11:02 IBMConnectionsDocs_2.0.1.zip

TDI:
total 704248
drwxr-xr-x.  2 root orion        70 May  6  2020 .
drwxr-xr-x. 13 root orion       192 Nov 18 08:33 ..
-r-xr-xr-x.  1 root orion  76251327 May  6  2020 7.2.0-ISS-SDI-FP0006.zip
-r-xr-xr-x.  1 root orion 644894720 Apr 30  2020 SDI_7.2_XLIN86_64_ML.tar

cp:
total 22500184
drwxr-xr-x.  2 dmenges orion         255 Nov 19 10:10 .
drwxr-xr-x. 13 root    orion         192 Nov 18 08:33 ..
lrwxrwxrwx.  1 root    root           31 Nov 17 21:01 component_pack_65.zip -> hybridcloud_20191122-120706.zip
lrwxrwxrwx.  1 root    root           31 Nov  6 17:21 component_pack_6501.zip -> hybridcloud_20200420-105909.zip
lrwxrwxrwx.  1 root    root           49 Nov 19 10:10 component_pack_latest.zip -> /data/packages/cp/hybridcloud_20201118-181644.zip
-r-xr-xr-x.  1 root    root   5339289797 Nov 17 21:01 hybridcloud_20191122-120706.zip
-rw-rw-r--.  1 dmenges orion  5559028115 Apr 20  2020 hybridcloud_20200420-105909.zip
-rw-rw-r--.  1 dmenges orion  6667502885 Aug 25 17:30 hybridcloud_20200825-172059.zip
-rw-r--r--.  1 lcuser  lcuser 5474360862 Nov 19 10:10 hybridcloud_20201118-181644.zip

was855:
total 8280848
drwxr-xr-x.  2 dmenges orion       4096 Oct 21 09:18 .
drwxr-xr-x. 13 root    orion        192 Nov 18 08:33 ..
-rw-r--r--.  1 dmenges orion 1025869744 Apr 23  2020 8.0.5.17-WS-IBMWASJAVA-Linux.zip
-rw-r--r--.  1 root    root  1022054019 Oct 21 09:15 8.0.6.15-WS-IBMWASJAVA-Linux.zip
-rw-r--r--.  1 dmenges orion  135872014 Apr 23  2020 InstalMgr1.6.2_LNX_X86_64_WAS_8.5.5.zip
-rw-r--r--.  1 dmenges orion 1054717615 Apr 23  2020 WAS_ND_V8.5.5_1_OF_3.zip
-rw-r--r--.  1 dmenges orion 1022550691 Apr 23  2020 WAS_ND_V8.5.5_2_OF_3.zip
-rw-r--r--.  1 dmenges orion  902443241 Apr 23  2020 WAS_ND_V8.5.5_3_OF_3.zip
-rw-r--r--.  1 dmenges orion  112238284 Apr 23  2020 WAS_ND_v8.5.5_Liberty.zip
-rw-r--r--.  1 dmenges orion  976299561 Apr 23  2020 WAS_V8.5.5_SUPPL_1_OF_3.zip
-rw-r--r--.  1 dmenges orion 1056708869 Apr 23  2020 WAS_V8.5.5_SUPPL_2_OF_3.zip
-rw-r--r--.  1 dmenges orion  998887246 Apr 23  2020 WAS_V8.5.5_SUPPL_3_OF_3.zip
-rw-r--r--.  1 dmenges orion  171921530 Apr 23  2020 agent.installer.linux.gtk.x86_64_1.8.9006.20190918_1303.zip
-rw-r--r--.  1 root    root   215292676 Aug 12 22:52 agent.installer.linux.gtk.x86_64_1.9.1003.20200730_2125.zip

was855FP16:
total 7391380
drwxr-xr-x.  2 dmenges orion       4096 Oct 21 09:08 .
drwxr-xr-x. 13 root    orion        192 Nov 18 08:33 ..
-rw-r--r--.  1 dmenges orion 1037725904 Apr 23  2020 8.0.5.37-WS-IBMWASJAVA-Linux.zip
-rw-r--r--.  1 dmenges orion  359841878 Apr 23  2020 8.0.5.37-WS-IBMWASJAVA-Win.zip
-rw-r--r--.  1 dmenges orion  967852764 Apr 23  2020 8.5.5-WS-WAS-FP016-part1.zip
-rw-r--r--.  1 dmenges orion  216147845 Apr 23  2020 8.5.5-WS-WAS-FP016-part2.zip
-rw-r--r--.  1 dmenges orion 1899893435 Apr 23  2020 8.5.5-WS-WAS-FP016-part3.zip
-rw-r--r--.  1 dmenges orion  471099562 Apr 23  2020 8.5.5-WS-WASSupplements-FP016-part1.zip
-rw-r--r--.  1 dmenges orion  716291953 Apr 23  2020 8.5.5-WS-WASSupplements-FP016-part2.zip
-rw-r--r--.  1 dmenges orion 1899893435 Apr 23  2020 8.5.5-WS-WASSupplements-FP016-part3.zip

was855FP18:
total 7240080
drwxr-xr-x.  2 root    orion       4096 Oct 21 09:12 .
drwxr-xr-x. 13 root    orion        192 Nov 18 08:33 ..
-rw-r--r--.  1 root    orion 1022054019 Sep 24 15:33 8.0.6.15-WS-IBMWASJAVA-Linux.zip
-rw-rw-r--.  1 dmenges orion 1024726232 Sep 24 16:17 8.5.5-WS-WAS-FP018-part1.zip
-rw-rw-r--.  1 dmenges orion  216415911 Sep 24 16:20 8.5.5-WS-WAS-FP018-part2.zip
-rw-rw-r--.  1 dmenges orion 1955876681 Sep 24 16:23 8.5.5-WS-WAS-FP018-part3.zip
-rw-r--r--.  1 root    orion  472424472 Sep 24 16:40 8.5.5-WS-WASSupplements-FP018-part1.zip
-rw-r--r--.  1 root    orion  766454469 Sep 24 16:44 8.5.5-WS-WASSupplements-FP018-part2.zip
-rw-r--r--.  1 root    orion 1955876681 Sep 24 16:47 8.5.5-WS-WASSupplements-FP018-part3.zip
```

Of course, you can drop it all to a single folder, or restructure it whatever way you prefer.

### Inventory files

Inventory files live in environments folder. In general all the variables are separated in two files - connections and component_pack. It could be merged of course, as lot of stuff is overlapping, but be careful if merging not to overlap stuff that eventually shouldn't be (like if you are using different NFSs for the WebSphere-side of Connections and Component Pack).

### Supported layouts

Those scripts should be able to support both single node installations and HA environments, specifically:

* We tested with a single WAS node (DMGR, IHS, and Node agent co-located)
* We tested with a proper distributed WAS (two separate WAS nodes, two separate IHSs behind internal load balancer, DMGR on a dedicated machine)
* Kubernetes can be single node installation or any type of HA. Scaling is also supported by those scripts (you can add new masters or workers if you created your Kubernetes environment using those scripts in the first place).

### Cache folders

Running those scripts specifically for Component Pack will create two cache folders:

* /tmp/k8s_cache and
* /home/${YOURUSER}/generated_charts

The second one will have all the values generated when running Helm installs. Those value files are used by Helm to install Component Pack. Values you see inside are coming from a variables inside your automation.

## Setting up HCL Connections with its dependencies

This scenario is useful in multiple cases:
* If you want to spin up a demo environment where you want to be able to quickly log in and start using it
* If you want to spin up a production ready environment that you know will work, see it work, and then start replacing component by component (like switching/adding LDAP servers, switching databases, etc).

```
ansible-playbook -i environments/examples/cnx7/connections_7_with_component_pack/connections playbooks/setup-connections-complete.yml
```

Running this playbook will:
* Setup DB2 with HCL Connections Wizards on top
* Install OpenLDAP and populate it with a number of fake user, one of them being the HCL Connections admin user
* Set and execute IBM TDI to popule PeopleDB with LDAP users
* Install IBM WebSphere Application Server Network Deployment and IBM HTTP Server and configure it properly
* Install HCL Connections on top of that

You can, of course, run those steps separately.

### Setting up IBM DB2

To install IBM DB2 only, execute:

```
ansible-playbook -i environments/examples/cnx7/connections_7_with_component_pack/connections playbooks/third_party/setup-db2.yml
```

This will install IBM DB2 on a Linux server, tune the server and IBM DB2 as per Performance tunning guide for HCL Connections, and apply the licence.

In case IBM DB2 was already installed nothing will happen, the scripts will just ensure that everything is as expected.

### Installing HCL Connections Wizards

This requires IBM DB2 already being set. To create the databases and apply the grants, execute:

```
ansible-playbook -i environments/examples/cnx7/connections_7_with_component_pack/connections playbooks/hcl/setup-connections-wizards.yml
```

If databases already exist, this script will execute runstats and rerogs on all the databases by default on each consecutive run.

If you want to recreate the databases, uncomment:

```
#cnx_force_repopulation=True
```

If you are upgrading from Connections 6.5.0.1 to 7 and want to only create IC360 database and run needed migrations, uncomment:

```
#db_enable_upgrades=True
```

in your inventory file. This will then drop all the databases and recreate them again. Don't forget to run TDI afterwards. Be sure to comment it again once you do it. 

### Setting up OpenLDAP with SSL and amount of fake users

To install OpenLDAP with SSL enabled and generate fake users, execute:

```
ansible-playbook -i environments/examples/cnx7/connections_7_with_component_pack/connections playbooks/third_party/setup-ldap.yml
```

You can turn on or off creating any of fake users by manipulating:

```
setup_fake_ldap_users=True
```

If you are creating them, you can manipulate the details using next set of variables:

```
ldap_nr_of_users=2500
ldap_userid="fakeuser"
ldap_user_password="password"
ldap_user_admin_password="password"
```

This will create 2500 fake accounts, starting with user id 'fakeuser1' and going to 'fakeuser2499'. First of them in this case (fakeuser1) will get 'ldap_user_admin_password' set, and all others are going to get 'ldap_user_password'. On top of that, it you will get automatically 10 more users created being set as external. Be sure to set in this case one of those users as HCL Connections admin user before the Connections installation like here:

```
connections_admin=fakeuser1
``` 

This comes in handy if you don't have any other LDAP server ready and you want to quickly get to the point where you can test HCL Connections. You can later replace this LDAP with any other one.

### Setting up IBM TDI and populating PeopleDB

To install IBM TDI, set up tdisol folder as per documentation, and migrate the users from LDAP to IBM DB2, run:

```
ansible-playbook -i environments/examples/cnx7/connections_7_with_component_pack/connections playbooks/third_party/setup-tdi.yml
```

### Installing and configuring IBM WebSphere Application Server only

To install IBM WebSphere Application Server, you should alraedy have either LDAP installed all LDAP data properly configured in your inventory file, in the section like this:

```
# This is just an example!
ldap_server=ldap1.internal.example.com
ldap_alias=ldap1
ldap_repo=LDAP_PRODUCTION1
ldap_bind_user=cn=Admin,dc=cxn,dc=example,dc=com
ldap_bind_pass=password
ldap_realm=dc=cxn,dc=example,dc=com
ldap_login_properties=uid;mail
```

LDAP should also have SSL enabled as IBM WebSphere Application Server is going to try to import its TLS certificate and fail if there is none.

Be sure that you also have proper values here:
```
dmgr_hostname=dmgr1.internal.example.com
was_domainname=.internal.example.com
domain_name=.internal.example.com
cnx_domainname=.internal.example.com
smtp_hostname=localhost
```

If you have more then one domain (for example, you dynamicHost is connections.example.com but all your servers live in DMZ on *.internal.example.com) be sure to also set:

```
sso_domainname=.internal.example.com;.example.com
```

And in the end, you need to create WebSphere user account by setting this:

```
was_username=wasadmin
was_password=password
```

To install IBM WebSphere Application Server, IBM HTTP Server and configure it, execute:

```
ansible-playbook -i environments/examples/cnx7/connections_7_with_component_pack/connections playbooks/third_party/setup-webspherend.yml
```

### Installing HCL Connections

To install the WebSphere-side of Connections only, on an already prepared environment (all previous steps are already done and the environment is ready for HCL Connections to be installed) execute:

```
ansible-playbook -i environments/examples/cnx7/connections_7_with_component_pack/connections playbooks/hcl/setup-connections-only.yml
```

To enable Moderation and Invites, set:

```
cnx_enable_moderation=true
cnx_enable_invite=true
```

To ensure that you don't have to pin specific offering version yourself, make sure that the next variable is set:

```
cnx_updates_enabled=True
```

You can also rewrite locations of message and shared data stores by manipulating next two variables before you hit install:

```
cnx_shared_area="/nfs/data/shared"
cnx_message_store="/nfs/data/messageStores"
```

### Running post installation tasks

Once your Connections installation is done, run this playbook to set up some post installation bits and pieces needed for Connections 7 on WebSphere:

```
ansible-playbook -i environments/examples/cnx7/connections_7_with_component_pack/connections playbooks/hcl/connections-post-install.yml
```

## Setting up Component Pack for HCL Connections 7 with its dependencies

To set up Component Pack, you should have the WebSphere-side of Connections already up and running and be able to log in successfully.

To set up Component Pack, execute:

```
ansible-playbook -i environments/examples/cnx7/connections_7_with_component_pack/component_pack playbooks/setup-component-pack-complete.yml
```

This playbook will:

* Set up Nginx
* Set up Haproxy
* Set up NFS
* Set up Docker and Docker Registry
* Set up Kubernetes
* Install and configure Component Pack to work with Connections

By default, Component Pack will consider first node in [nfs_servers] to be your NFS master also, and by default it will consider that on NFS master, all needed folders for Component Pack live in /pv-connections. You can rewrite it with:

```
nfsMasterAddress="172.29.31.1"
persistentVolumePath="nfs"
```

This translates to //172.29.31.1:/nfs

Note: The Component Pack installation and configuration contains tasks with an impact on the WebSphere-side of Connections side which require restarts of WebSphere and Connections itself, and which will be executed.

### Setting up Nginx

Nginx is used only as an example of a reverse proxy for the WebSphere-side of Connections and Component Pack.

To install Nginx only, execute:

```
ansible-playbook -i environments/examples/cnx7/connections_7_with_component_pack/component_pack playbooks/third_party/setup-nginx.yml
```

### Setting up Haproxy

Haproxy is used as an example load balancer. To ensure it is not trying to bind to the same ports as Nginx, if you are co-locating it together, be sure to set those two variables in your inventory file:

```
[k8s_load_balancers:vars]
main_port='81'
main_ssl_port='444'
```

To install Haproxy only, execute:

```
ansible-playbook -i environments/examples/cnx7/connections_7_with_component_pack/component_pack playbooks/third_party/setup-haproxy.yml
```

### Setting up NFS

If you are setting up your own NFS, execute:

```
ansible-playbook -i environments/examples/cnx7/connections_7_with_component_pack/component_pack playbooks/third_party/setup-nfs.yml
```

This will set up the NFS master, create and export needed folders for Component Pack components, and set up the clients so they can connect to it.

By default, NFS master will export the folders for its own network. If you want to change the netmask, be sure to set:

```
[nfs_servers:vars]
nfs_export_netmask=255.255.0.0
```

### Setting up Docker and Docker Registry

This will install docker-ce and configure Docker Registry with SSL enabled, and also ensure that all future workers can authenticate to that Docker Registry. Default usernames/passwords can be overridden.

To set it up, execute:

```
ansible-playbook -i environments/examples/cnx7/connections_7_with_component_pack/component_pack playbooks/third_party/setup-docker-registry.yml
```

### Setting up Kubernetes

This will install Kubernetes, set up kubectl and Helm and make the Kubernetes cluster ready for Component Pack to be installed. However, this is really a minimum way of installing a stable Kubernetes using kubeadm. We do advice using more battle-proven solutions like kubespray or kops (or anything else) for production-ready Kubernetes clusters.

This set of automation will install by default 1.18.10 and should be always able to install the Kubernetes versions supported by Component Pack.

To install Kubernetes, execute:

```
ansible-playbook -i environments/examples/cnx7/connections_7_with_component_pack/component_pack playbooks/third_party/kubernetes/setup-kubernetes.yml
```

#### Setting up kubectl for your user

If the cluster is already created, and you only want to set up kubectl for your own user, just run:

```
ansible-playbook -i environments/examples/cnx7/connections_7_with_component_pack/component_pack playbooks/third_party/kubernetes/setup-kubectl.yml
```

## Troubleshooting

For each machine that is going to be used in any capacity with those scripts, ensure that:

* You run yum/dnf update before you start doing anything (there are multiple components that can fail installing if you don't do this)
* You have user accounts created
* DNS is properly set and reverse DNS works fine in any direction
* You have proper network and internet access where needed (no firewall issues etc)
* Machine hostnames are set properly (there are multiple components that can fail installing if this is not the case)

As a rule of thumb, be sure that:

* User that is running Ansible can SSH into all the servers mentioned inside your inventory files.
* User that is running Ansible have passwordless sudo access on all the servers mentioned inside your inventory files.
* User that is running Ansible has key forwarding enabled. Automation will try to SSH from one machine to another to execute specific tasks, run rsync etc. which can fail if key forwarding is not properly configured.
* Ansible is using Python. It *can* happen that something doesn't work as expected in your own scenario, depending on OS level, changes in distribution, etc.


# HCL Connections Docs Automation Scripts
This set of scripts is able to spin up end-to-end HCL Connections Docs on top of an existing HCL Connections deployed by the ansible scripts described above.

For HCL Connections Docs dependencies this means that:
* HCL Docs database will be created with needed user, schemas and grants.
* WebSphere application clusters will be created for each Docs components.
* WebSphere Job Manager targets will be created for Docs and Conversion nodes, Deployment Manager and Web Server.

For HCL Connections Docs itself it means:
* The script will mount NFS shares needed for the HCL Connections shared data, Docs data and Viewer data.
* HCL Docs installation kit will be downloaded.  Document Format Conversion, Editor and Viewer applications, Editor and Viewer Extensions, Editor Proxy Filter will be installed.
* IBM HTTP Server plugins will be re-generated and propagated to the web servers.
* All clusters will be started automatically at the end of the install.

## Requirements:
* HCL Connections has been installed using the Ansible script describe above.
* If HCL Connections Docs will be installed on a domain different from the external URL, setup an external route to connect to the load balances for the HCL Connections Docs.

### Have files ready for download
* See [Have files ready for download](https://github.com/HCL-TECH-SOFTWARE/connections-automation#have-files-ready-for-download) above.

### Installing HCL Connections Docs

To install HCL Connections Docs, adjust the inventory file in `environments/examples/connections_docs`, execute:
```
ansible-playbook -i environments/examples/connections_docs playbooks/hcl/setup-connections-docs.yml
```

### HCL Connections Docs Troubleshooting
* If HCL Connections Docs configuration adjustment is needed after the install, it can be done in `/opt/IBM/WebSphere/AppServer/profiles/Dmgr01/config/cells/${CELLNAME}/IBMDocs-config` on the WebSphere Deployment Manager.  Make sure to do a full re-synchronize to propagate any changes to all Docs nodes.
* If options are not available in HCL Connections Files to create Document/Spreadsheet/Presentation, follow Step#3 in [here](https://help.hcltechsw.com/docs/v2.0.1/onprem/install_guide/guide/text/functional_verification_of_installation.html) to check if the HCL Docs and File Viewer extensions are installed and active within HCL Connections. If the modules are not listed, on the HCL Connections server, use the `findmnt` command to locate the mount target of `{{nfsMasterAddress}}:{{cnx_data_nfs}}` based on the inventory file, then go to `/provision/webresources` in that location to make sure the jars are there. If not present, try to locate them in `{{cnx_data_nfs}}` as local path and move them to the correct location (i.e. `<Connections shared data>/provision/webresources`). You'll need to restart Connections to see the modules listed.
* If you wish to re-install a HCL Connections Docs components, cd to `/opt/HCL/<component>/installer` then run `sudo ./uninstall.sh`.  Delete the .success file in `/opt/HCL/<component>/` then run the playbook again.  It is recommended to comment out the prior steps in the playbook to save time.

## Acknowledgments

This project was inspired by the [Ansible WebSphere/Connections 6.0 automation done by Enio Basso](https://github.com/ebasso/ansible-ibm-websphere). It is done in a way that it can interoperate with the mentioned project or parts of it. 

## License

This project is licensed under Apache 2.0 license. 

## Disclaimer

This product is not officially supported, and can be used as is. This product is only proof of concept, and HCL Technologies Limited would welcome any feedback. HCL Technologies Limited does not make any warranty about the completeness, reliability and accuracy of this code. Any action you take by using this code is strictly at your own risk, and HCL Technologies Limited will not be liable for any losses and damages in connection with the use of this code.
