# Standing up a Distributed Splunk Environment

## Table of Contents:
- [Important Links](#important-links)
- [Splunk Environment](#splunk-environment)
- [Standing up a distributed Environment](#standing-up-a-distributed-environment)
    - [Installing Splunk](#installing-splunk)
    - [Fixing Ulimits and THP](#fixing-ulimit-and-thp)
- [Connecting Splunk Instances](#connecting-splunk-instances)
    - [Instance Explanations and Breakdowns](#instance-explanations-and-breakdowns)
    - [License Manager](#setting-up-the-license-manager)
    - [Indexer Cluster](#setting-up-the-indexer-cluster)
    - [Heavy Forwarder](#setting-up-the-heavy-forwarder)
    - [Deployment Server](#setting-up-the-deployment-server)
    - [Deploy Apps to the Index and Search Head Clusters](#deploy-apps-to-the-indexers-and-search-head-clusters)
    - [Install apps on the CM, DPMCLM, and DS](#install-apps-on-the-cm-dpmclm-and-ds)
    - [Monitoring Console](#setting-up-the-monitoring-console)

---
---
<br/>

---
---
## Important Links
- Splunk Base App Templates:
    - Base Apps (Non-clustered): https://drive.google.com/drive/folders/107qWrfsv17j5bLxc21ymTagjtHG0AobF
    - Clustered Environment Apps: https://drive.google.com/drive/folders/10aVQXjbgQC99b9InTvncrLFWUrXci3gz

---
---
<br/>

---
---

## Important Considerations
- After installing any app that modifies splunk configuration files (.conf), restart the splunk instance.
    - When apps are distributed to the Indexer Cluster via the Cluster Manager, restartes should be handled as indicated by the serverclass.conf file.  Veriy that restarts occur via the "splunk show cluster-bundle-status" CLI command.
    - When apps are distributed to the SEarch Head Cluser bia the Deployer, manually initiate a rolling restart by running the "splunk rolling-restart shcluster-members" command on the Search Head Cluster Captain.
        - Determine the captain by running "splunk show shcluster-status" command on any Search Head that is part of the SHC.
---
---
<br/>

---
---


## Splunk Environment

The splunk environment consits of the following server, all of which are RHEL 8.

Instance Type|Abbreviation|Number|IPs
-|-|-|-|
Deployer, Monitoring Console, License Manager|DPMCLM|1|192.168.10.27
Search Head|SH|2|192.168.10.20<br>192.168.10.21
Cluster Manager|CM|1|192.168.10.22
Indexer|IDX|3|192.168.10.23<br>192.168.10.24<br>192.168.10.25
Deployment Server|DS|1|192.168.10.28
Heavy Forwarder|HF|1|192.168.10.26


Non-Splunk Servers|Abbreviation|Number|IPs
-|-|-|-|
Syslog||1|192.168.10.29
---
---
<br/>

---
---

## Standing up a distributed Environment

Once the RHEL 8 servers have been stood up, log in as root or another user with sudo privliges and perform the following on each of them (**excluding the Syslog server**):
> **NOTE** With the exception of setting the servername and default-hostname, all of the following steps are identical for each instance of Splunk.  I recommend using a program like [Remote Desktop Manager](https://devolutions.net/remote-desktop-manager/) which inludes a *broadcast* function to perform these steps on all instance simultaneously.

### Installing Splunk
---
#### 1. Verify that the system is up-to-date:
```
dnf update install
```

#### 2. Verify that wget and tar are installed
```
dnf install wget
dnf install tar
```

#### 3. Open Necessary ports in the firewall, then reload the firewall

Port|Use|Type|Direction|Instances
-|-|-|-|-
8000|SplunkWeb UI|TCP|Inbound/Outbound|DPMCLM<br>SH<br>DS<br>CM<br>HF
8089|Management Port|TCP|Inbound/Outbound|All Instances
9100|Replication Port|TCP|Inbound/Outbound|SH<br>DPMCLM<br>IDX
9997|Data Ingestion Port|TCP|Inbound/Outbound|All Instances
514|Standard Syslog Port|UDP|Inbound|Syslog Server<br>HF

```
firewall-cmd --permanent --zone=public --add-port=8000/tcp
firewall-cmd --permanent --zone=public --add-port=8089/tcp
firewall-cmd --permanent --zone=public --add-port=9100/tcp
firewall-cmd --permanent --zone=public --add-port=9997/tcp
firewall-cmd --permanent --zone=public --add-port=514/udp

# Reload the firewalls for changes to take effect.
firewall-cmd --reload

# Verify that all ports have been opened:
firewall-cmd --list-all
```
> **NOTE**: Verify the zone parameter with your network admins.

#### 4. Add the Splunk User
```
adduser splunk
```

#### 5. Download the target Splunk Enterprise installer.
```
wget -O /tmp/splunk-9.2.0.1-d8ae995bf219-Linux-x86_64.tgz "https://download.splunk.com/products/splunk/releases/9.2.0.1/linux/splunk-9.2.0.1-d8ae995bf219-Linux-x86_64.tgz"
```

> **NOTE**: Splunk version-tracking is very important, particually in clustered environment.  Do not use another version of Splunk in this environment without ensuring that all instances go through the upgrade process in the proper order per the [documentation](https://docs.splunk.com/Documentation/Splunk/latest/Installation/UpgradeyourdistributedSplunkEnterpriseenvironment).

#### 6. Extract Splunk to the /opt directory
> template:
```
tar -xzvf /tmp/<SPLUNK INSTALL> -C /opt/
```
> For the downlaoded Splunk version:
```
tar -xzvf /tmp/splunk-9.2.0.1-d8ae995bf219-Linux-x86_64.tgz -C /opt/
```

#### 7. Change ownership of the splunk directory to the splunk user
```
chown -R splunk:splunk /opt/splunk
```

#### 8. Switch to the Splunk user and go to the splunnk bin directoy
```
su splunk

cd /opt/splunk/bin
```

#### 9. Start Splunk
```
./splunk start --accept-license --answer-yes
```

#### 10. When prompted, set the admin username and password.
An admin username of 'admin' is recommended.  Avoid admin usernames with special characters such as '.' to avoid issues with cluster setup.

#### 11. Set the servername and default-hostname
> **NOTE:** !!! ***IF YOU ARE USING BROADCAST*** - Disable broadcast for this step.  Each Splunk instance should have a unique servername and hostname
```
./splunk set servername <SERVERNAME>
./splunk set default-hostname <HOSTNAME>
```
Generally, these two settings are set to the same value.  **Note** that if these values different from your machine name, certain data sources may show two different hosts for events coming from the same machine.  This is not a functional problem, but may make it more difficult to isolate results from a specific machine.

#### 2. Verify that you can access the SplunkWeb UI:
```
http://<IP>:8000
```
Got to the IP of the Splunk instances, port 8000, and verify that you can access the Splunk UI.  If you are unable to connect to the splunk UI, verify that splunk is running, that you are connecting over http:, and that the ports have been opened on the local machine.  If none of these is the issue, then it's likely that an interstitial firewall is blocking communication.

#### 13. Set Splunk to run at boot
```
# Stop Splunk before proceeding
./splunk stop

# This must be done as root or a sudo-user.
exit

/opt/splunk/bin/splunk enable boot-start -systemd-managed 1 -user splunk
```
> ***Before hitting enter, verify that you have added the '-user splunk' flag.  Do not run Splunk as a root/sudo user unless absolutly necessary.***

This command will mark splunk as being run through systemd.  This is considered best practice in most modern Linux distros.  However, if it is necessary to run splunk through init.d, leave out the '-systemd-managed 1' flag from the command above.  If you do this, please note that the [ulimits section](#fixing-ulimit-and-thp) below will not work.

#### 14. Start Splunk and verify its status.
```
systemctl start Splunkd

systemctl status Splunkd
```
> Other commands needed during Splunk configuration:
> ```
> # Stop the Splunk service:
> systemctl stop Splunkd
> 
> # Restart the Splunk service:
> systemctl restart Splunkd
> ```

This concludes the segment about install Splunk.  Before beginning ingst, go through the process to [fix ulimits and transparent-huge pages](#fixing-ulimit-and-thp) on all instances.

---
---


### Fixing Ulimit and THP
---
The following steps are for instances running Splunk through SystemD, and should be done as the root user.  These should be performed on all Splunk Enterprise instances.  Failure to modify these settings may result in performance degregation.  This should also be done on all UFs collecting syslog data.  

#### 1. Edit the Splunkd.service configuration file
```
vi /etc/systemd/system/Splunkd.service
```
#### 2. Remove the original LimitNOFILE and LIMITPRIO settings.  
Paste the following where they were removed:
```
LimitCORE=0
LimitDATA=infinity
LimitNICE=0
LimitFSIZE=infinity
LimitSIGPENDING=385952
LimitMEMLOCK=65536
LimitRSS=infinity
LimitMSGQUEUE=819200
LimitRTPRIO=0
LimitSTACK=infinity
LimitCPU=infinity
LimitAS=infinity
LimitLOCKS=infinity
LimitNOFILE=1024000
LimitNPROC=512000
TasksMax=infinity
```

The file breakdown should looks soomething like:
```
[Service]
Type=simple
Restart=always
ExecStart=/opt/splunk/bin/splunk _internal_launch_under_systemd
KillMode=mixed
KillSignal=SIGINT
TimeoutStopSec=600
```
```
LimitCORE=0
LimitDATA=infinity
LimitNICE=0
LimitFSIZE=infinity
LimitSIGPENDING=385952
LimitMEMLOCK=65536
LimitRSS=infinity
LimitMSGQUEUE=819200
LimitRTPRIO=0
LimitSTACK=infinity
LimitCPU=infinity
LimitAS=infinity
LimitLOCKS=infinity
LimitNOFILE=1024000
LimitNPROC=512000
TasksMax=infinity
```
```
SuccessExitStatus=51 52
...
# There is more to the file after this.
```

#### 3. Create a THP service file
```
vi /etc/systemd/system/disable-thp.service
```

#### 4. Paste the following into the file and save it.
```
[Unit]
Description=Disable Transparent Huge Pages (THP)

[Service]
Type=simple
ExecStart=/bin/sh -c "echo 'never' > /sys/kernel/mm/transparent_hugepage/enabled && echo 'never' > /sys/kernel/mm/transparent_hugepage/defrag"

[Install]
WantedBy=multi-user.target
```

#### 5. Mark disable-thp.service as executable:
```
chmod 755 /etc/systemd/system/disable-thp.service
```

#### 6. Enable and start the service:
```
systemctl daemon-reload
```
```
systemctl start disable-thp
```
```
systemctl enable disable-thp
```

#### 7. Reboot the system
```
reboot
```

#### 8. As the Splunk, verify that the ulimts have been set properly after the reboot.
```
su splunk

tail -500 /opt/splunk/var/log/splunk/splunkd.log | grep ulimit
```

Look for output like this: (**Note**: The actual log events will include timestamps)
```
INFO  ulimit [2923 MainThread] - Limit: virtual address space size: unlimited
INFO  ulimit [2923 MainThread] - Limit: data segment size: unlimited
INFO  ulimit [2923 MainThread] - Limit: resident memory size: unlimited
INFO  ulimit [2923 MainThread] - Limit: stack size: unlimited
INFO  ulimit [2923 MainThread] - Limit: core file size: 0 bytes
WARN  ulimit [2923 MainThread] - Core file generation disabled.
INFO  ulimit [2923 MainThread] - Limit: data file size: unlimited
INFO  ulimit [2923 MainThread] - Limit: open files: 1024000 files
INFO  ulimit [2923 MainThread] - Limit: user processes: 512000 processes
INFO  ulimit [2923 MainThread] - Limit: cpu time: unlimited
INFO  ulimit [2923 MainThread] - Linux transparent hugepage support, enabled="never" defrag="never""
```

---
---
<br/>

---
---

## Connecting Splunk Instances

### Instance Explanations and Breakdowns

Once all instances have been set up, the next step is to connect them.  The following is a breakdown of hte purpose of each instance, and the instances to which it need to be connected.

> **NOTE**: I am linking to the interor of the associate app directories.  Download the full app archive.  
> 
> **NOTE 2**: The org_deployer_deploymentclient is a custom modification of the org_manager_deploymentclient app.

Most apps will be destributed to Splunk instances through the Deployment Server. The exceptions are the DP/MC/LM and CM.  Apps will need to copied to those instances manually.  This is an affect of using the Deployment Server as the master record for all apps, including those that are being sent to the Indexers and Search Heads.

Instance Type|Purpose|Connect to:
-|-|-
DP/MC/LM | The Deployer portion of the instance acts as the master reference for the Search Head Cluster.  The Monitorin Console tracks the health of your overall environment.  All other instances subscribe to the License Manager portion to track licensing| Deployment Server: [org_deployer_deploymentclient](https://drive.google.com/drive/folders/12AEJfMk0IRftW1WfQsBf07QG5_P0OhOE)<br>Indexer Cluster: [org_cluster_search_base](https://drive.google.com/drive/folders/12YGuQZTANzrm3lfxaUgIH9JcXb3cQhXy)
SH | The Search Heads serach data located on the indexers.  They are a main install point for apps and certain add-ons.| Deployment Server: [org_all_deploymentclient](https://drive.google.com/drive/folders/11oJsdNJgX8JjMvf3xSGoXZxXsTGZKsmD)<br>License Manager: [org_full_license_server](https://drive.google.com/drive/folders/11AgVMe46mdDyrYU5oxVE2xNifPK1m5Dk)<br>Indexers: [org_cluster_forwarder_outputs](https://drive.google.com/drive/folders/12nUzjTgHJj-pcddTAlkRSQd0-P1k3-sp)<br>Indexer Cluster: [org_cluster_search_base](https://drive.google.com/drive/folders/12YGuQZTANzrm3lfxaUgIH9JcXb3cQhXy)
IDX|Indexers receive events from all instances and UFs.|Cluster Manager: [org_cluster_indexer_base](https://drive.google.com/drive/folders/12qPz104uq7IBaPyA4N60odisku7r0EHJ)<br>License Manager: [org_full_license_server](https://drive.google.com/drive/folders/11AgVMe46mdDyrYU5oxVE2xNifPK1m5Dk)
CM|The Cluster Manager manages and distributes apps to the Indexers in the indexer cluster.|Deployment Server: [org_manager_deploymentclient](https://drive.google.com/drive/folders/12AEJfMk0IRftW1WfQsBf07QG5_P0OhOE)
HF|The Heavy Forwarder will act as the colleciton point for apps that pull events via APIs or other methods that can't be covered by a Universal Forwarder.|Deployment Server: [org_all_deploymentclient](https://drive.google.com/drive/folders/11oJsdNJgX8JjMvf3xSGoXZxXsTGZKsmD)<br>License Manager: [org_full_license_server](https://drive.google.com/drive/folders/11AgVMe46mdDyrYU5oxVE2xNifPK1m5Dk)<br>Indexers: [org_cluster_forwarder_outputs](https://drive.google.com/drive/folders/12nUzjTgHJj-pcddTAlkRSQd0-P1k3-sp)

---

> **NOTE**: Be sure to modift the 'org' portion of app namesto reflect your organization.
---

### Setting up the License Manager
> **NOTE:** Configuration of other aspects of the DP/MC/LM instance will be handled later.  This portion is only about installing your Splunk license.

1. On the DP/MC/LM UI: Go to **Settings > Licensing**.
2. Click *Add License*
3. Click *Choose File*.  Select your license file.
4. Click *Install*

---
---

### Setting up the Indexer Cluster

An Indexer Cluster consists of the following instances:

- Cluster Manager
- 2 or more Indexers

Any apps that need to be installed on the indexers are distributed to them from the Cluster Manager.  In turn, the Cluster Manager receives its apps from the Deployment Server.

#### 1. Mark the Cluster Manager as a Cluster Manager
> Intance: CM
>
> App: [org_cluster_manager_base](https://drive.google.com/drive/folders/12nEeT0uRV6xG1U7KC0dSNRzxXckwR-Gc)
>
> Install Location: */opt/splunk/etc/apps*

The CM will receive all apps from the Deployment Server.  Rather than being placed in the /etc/apps directory of the splunk install like normal, they will be redirected to the /etc/manager-apps folder to be distributed.

> server.conf
```
[clustering]
mode = manager
replication_factor = 3
search_factor = 2

pass4SymmKey = <KEY>
```
The pass4SymmKey is used by Indexers to authenticate with the Cluster Manager.  Replace '<KEY>' with a secure password.


#### 2. Connect the Cluster Manager To the Deployment Server.
> Intance: CM
>
> App: [org_manager_deploymentclient](https://drive.google.com/drive/folders/12AEJfMk0IRftW1WfQsBf07QG5_P0OhOE)
>
> Install Location: */opt/splunk/etc/apps*

> deploymentclient.conf
```
[deployment-client]
phoneHomeIntervalInSecs = 60

repositoryLocation = $SPLUNK_HOME/etc/manager-apps
serverRepositoryLocationPolicy = rejectAlways

[target-broker:deploymentServer]
targetUri= <DS IP>:8089
```

> **NOTE**: I recommend reducing the phoneHomeIntervalInSecs to 60 seconds during initial configuration.  Be sure to adjust it before finalizing the environment.


#### 3. Connect the Cluster Manager to the License Manager
> Instance: CM
>
> App: [org_full_license_server](https://drive.google.com/drive/folders/11AgVMe46mdDyrYU5oxVE2xNifPK1m5Dk)
>
> Install Location: */opt/splunk/etc/apps*

> server.conf
```
[license]
manager_uri = https://<DPMCLM IP>:8089
```

Restart the Cluster Manager.  (As root/sudoer)
```
systemctl restart Splunkd
```

#### 4. Connect the Indexers to the Cluster Manager
> Instance: IDX 1-3
>
> App: [org_cluster_indexer_base](https://drive.google.com/drive/folders/12qPz104uq7IBaPyA4N60odisku7r0EHJ)
>
> Install Location: */opt/splunk/etc/apps*

> server.conf
```
[clustering]
mode = peer
manager_uri = https://<CLUSTER MANAGER IP>:8089
pass4SymmKey = <KEY>

[replication_port://9100]
disabled = false
```
The <KEY\> value here is the same as the on in the *org_cluster_manager_base* app.

Restart the Indexers.  (As root/sudoer)
```
systemctl restart Splunkd
```

Verify that the Indexers are taling to the Cluster Manager:
> Instance: CM
```
/opt/splunk/bin/splunk show cluster-status --verbose
```
All three indexers should show as connected to the CM.

#### 5. Deploy the License Manager app to the Indexers

> **NOTE**: This can wait to be handled until you are [setting up the Deployment Server](#setting-up-the-deployment-server). It is not strictly necessary that it connect immediately.

The indexers need to be connected to the license manager.  This is done by distributing the License Manager app created above from the CM to the IDXs.

> Instance: CM

> **NOTE**: Be sure to adjust the app names in these commands to reflect the modified app name.

1. Copy the org_full_license_manager app from the /etc/apps firectory to the /etc/manager-apps directory:
```
cp -r /opt/splunk/etc/apps/org_full_license_server /opt/splunk/etc/manager-apps
```

2. Validate and distribute the app
```
/opt/splunk/bin/splunk validate cluster-bundle
```
Verify that there are no errors in the bundle
```
/opt/splunk/bin/splunk show cluster-bundle-status
```
If there are any errors they will be displayed above the indexer list.  If there are, correct them.  If not, then distribute the bundle.
```
/opt/splunk/bin/splunk apply cluster-bundle
```

Use the show cluster-bundle-status command to track that all the bundles have been applied and the indexers have restarted their splunk instance.  

If you check the peer-apps directory on the indexers, you should find the *ord_full_license_sever* app.  If you check the licensing menu on the DP/MC/LM, under *all license details*, you should find the three indexers.

This concludes the setup of the Indexer Cluster.  As data ingest begins it will be necessary to add more apps to the Indexers.  This will be accomplished by putting them in the /etc/deployment-apps directory on the Deployment Server, adding them to the server class that handles Indexer apps, and then manuall distribute them from the CM as shown in [step 4](#4-deploy-the-license-manager-app-to-the-indexers) above.

---
---

### Setting up the Search Head Cluster

A Search Head Cluster consists of the following instances:
- Deployer
- 2 or more Search Heads. 

Any apps that need to be installed on the Search Heads are distributed from the Deployer.  In turn, the deployer recieves its apps from the Deployment Server.

#### 1. Mark the Deployer as as Deployer for the SHC.

There is no base app for this.  Use the [org_APP_TEMPLATE](https://drive.google.com/drive/folders/10tTKGjJO5EeACNYejEIgsh2CassW0KQg) as a base and add a *server.conf* file to the local directory.

Be sure to rename the app to <ORG\>_deployer_base.

> Instance: DP/MC/LM
>
> App: [org_APP_TEMPLATE](https://drive.google.com/drive/folders/10tTKGjJO5EeACNYejEIgsh2CassW0KQg)
>
> Install Location: */opt/splunk/etc/apps*

> server.conf
```
[shclustering]
pass4SymmKey = <KEY FOR SHCLUSTER>
shcluster_label = SROC_shcluster
```

This pass4SymmKey is a different key than the one used for the Indexer Cluster but serves the same purpose for the Search Head Cluster.

Restart splunk
```
# As the root user:
systemctl restart Splunkd
```

#### 2. Connect the Deployer To the Deployment Server.
This uses a modified version of [org_manager_deploymentclient](https://drive.google.com/drive/folders/12AEJfMk0IRftW1WfQsBf07QG5_P0OhOE) that redirects apps to the */etc/shcluser/apps* directory so they can be deployed to the Search HEads in the Search Head Cluster.

> Instance: DP/MC/LM
>
> App: [org_deployer_deploymentclient](https://drive.google.com/drive/folders/12AEJfMk0IRftW1WfQsBf07QG5_P0OhOE)
>
> Install Location: */opt/splunk/etc/apps*

> deploymentclient.conf
```
[deployment-client]
phoneHomeIntervalInSecs = 60

repositoryLocation = $SPLUNK_HOME/etc/shcluster/apps
serverRepositoryLocationPolicy = rejectAlways

[target-broker:deploymentServer]
targetUri= <DS IP>:8089
```

#### 3. Connect the Search Heads to the Deployer:
On each of the Search Heads, run the following command:
> template:
```
splunk init shcluster-config -mgmt_uri <URI>:<management_port> -replication_port <replication_port> -replication_factor <n> -conf_deploy_fetch_url <URL>:<management_port> -secret <security_key> -shcluster_label <label>
```

Flag|Purpose|Example
-|-|-
-mgmt_uri|full (https://...) ip for this Search Head with the management port (8089)|-mgmt_uri https://192.168.10.20:8089
-replication_port|The port over which Knowledge Objects will be replicated between the Search Heads.|-replication_port 9100
-replication_factor| The number of copies of each Knowledge Object that are maintained by the SHC.  Cannot exceed the number of Search Heads in the cluster|-replication_factor 2
-conf_deploy_fetch_url|Deployer instance URL and port|-conf_deploy_fetch_url https://192.168.10.27:8089
-secret|The pass4SymmKey for the Search Head Cluster|-secret <KEY\>
-shcluster_label|The label for the search head cluster.  This should match the label defined int he app from [step 1](#1-mark-the-deployer-as-as-deployer-for-the-shc).|-shcluster_label SROC_shcluster

I recommend writing these out completely in an editor before running it on the SHs.  There should be two distinct commands, thought he only difference between them should be the URI of the search head upon which the command is being run.

Restart the Search Heads (As root/sudo)
```
systemctl restart Splunkd
```

#### 4. Connect the Search Heads to the Indexers
> Instance: DS
>
> App: [org_cluster_search_base](https://drive.google.com/drive/folders/12YGuQZTANzrm3lfxaUgIH9JcXb3cQhXy)
>
> Install Location: */opt/splunk/etc/deployment-apps/*

> server.conf

```
[clustering]
mode = searchhead
manager_uri = clustermanager:one

[clustermanager:one]
manager_uri=https://<CM IP>:8089
pass4SymmKey = <KEY>
multisite = false
```

> **NOTE**: In this app, the pass4SymmKey value is the Key for the Indexer Cluster, not the Search Head Cluster.  This key authenticates the search heads to the indexers and allows them to search the data on in the indexers.

#### 5. Set up output for the Search Heads
All instances of Splunk (aside from teh indexers) need to output their internal logs to the Indexers.

> Instance: DS
>
> App: [org_cluster_forwarder_outputs](https://drive.google.com/drive/folders/12nUzjTgHJj-pcddTAlkRSQd0-P1k3-sp)
>
> Install Location: */opt/splunk/etc/deployment-apps/*

> outputs.conf
```
[tcpout]
defaultGroup = primary_indexers 
forceTimebasedAutoLB = true
forwardedindex.2.whitelist = (_audit|_introspection|_internal)

[tcpout:primary_indexers]
server = <IDX1 IP>:9997, <IDX2 IP>:9997, <IDX3 IP>:9997
```

> **NOTE:** The IPs listed after 'server =' do not need 'https://'.  Including it will cause an error and prevent logs from being sent out.
>
> **NOTE 2**: This app will need to be copied to the DPMCLM, CM, and DS.  This will be covered later, but be sure to set it aside for future use.

#### 6. Set up a serverclass for this app
Before the forwader_outputs app above can be sent to the Search Head Cluster, it has to be deployed to the DP/MC/LM instance

This serverclass should send out to all Splunk instances except the indexers, but due to how the DP and CM are configured, they need to be handled seperately to prevent them trying to be enabled from the wrong directory.  Those instances, as well as the DS itself, will need to have the app manually copied to the */opt/splunk/etc/apps* directory and be restarted.

> Instance: DS
>
> App: None
>
> Location: */opt/splunk/etc/apps/system/local/*

> serverclass.conf
```
[serverClass:all_splunk_instances]
whitelist.0 = *
blacklist.0 = <DPMCLM IP>
blacklist.1 = <CM IP>

[serverClass:all_splunk_instances:app:org_all_forwarder_ouputs]
restartSplunkWeb = 0
restartSplunkd = 1
stateOnClient = enabled

[serverClass:dp_appstodistribute]
whitelist.0 = <DPMCLM IP>

[serverClass:dp_appstodistribute:app:org_all_forwarder_outputs]
restartSplunkWeb = 0
restartSplunkd = 0
stateOnClient = noop
```

Reload the deployment server
```
/opt/splunk/bin/splunk reload 
```

Once this is reloaded, the clients should show up in **Settings > Forwarder Management**, as should the three server classes and org_all_forwarder_outputs app.  If they do, then the app should be distributed immediately.

> **NOTE**: Be sure to rename the app in the serverclass.conf file to match the name of your version of the app.

#### 7. Verify that the DP has received the org_all_forwarder_outputs app

> Instance: DP/MC/LM

```
ls -l /opt/splunk/etc/shcluster/apps
```

If the app is present, move on.  If not, begin diagnosing what's wrong with the DS/serverclass.  The Forwarder MAnagement menu should give oyu some indication about errors.

#### 8. Distribute apps to the Search Heads
Once these apps are complete, they needs to be distributed to the search heads manually.  As the Splunk user:

```
# If you're not already the splunk user, switch to them:
su splunk

cd /opt/splunk/bin

./splunk apply shcluster-bundle
```

There is no validation process like with the Indexer Cluster.  If the app fails to distribute and the message something along the lines of "pre-check failed," or "pre-deploy check failed," there is a good chance that there is something wrong with one of your apps.


#### 9. Verify that the Search Heads are connected to the Indexers
Once the search heads have had a change to receive the new bundle and restart, check to make sure they are able to search the Indexers.

From one of Search Heads' UI, go to **Apps > Search & Reporting** and run the following search over the last 4 hours of data:
```
index=_internal | stats count by host
```

You should see the two search heads and three indexers listed as hosts.  If you don't, verify that the apps have been distributed to the Search Heads.  If they have, restart the Search Heads.  If there is still no connection, search the _internal logs on the search heads for any errors:
```
index=_internal log_level!="info"
```
Also, be sure to verify that port 9997 is open on the indexers.  

---
---

### Setting up the Heavy Forwarder

For now, the Heavy Forwarder only needs to be connected to the Deployment Server.  Additional apps will be distributed from the Deployment Server at a later time.

#### 1. Connect the Heavy Forwarder to the Deployment Server
> Instance: HF
>
> App: [org_all_deploymentclient](https://drive.google.com/drive/folders/11oJsdNJgX8JjMvf3xSGoXZxXsTGZKsmD)
>
> Install Location: */opt/splunk/etc/apps/*

> deploymentclient.conf
```
[deployment-client]
phoneHomeIntervalInSecs = 60

[target-broker:deploymentServer]
targetUri = <DS IP>:8089
```
> **NOTE**: As with the other deployment client apps, set the phone home interval to 60 seconds during development, but set it to something more reasonable once your environment is set up.

Because the Deployment Server is set up to send the *org_all_forwarder_outputs* app to each splunk client that connectes to it, the HF should receive the app, reboot, and being forwarding internal logs to the indexers.

---
---

### Setting up the Deployment Server

As of Splunk version 9.2, there is a problem with Deployment Servers keeping track of their subscribed clients.

#### 1. Correct issue with Deployment Server not tracking clients
> Instance: DS
>
> Apps: None
>
> Location: */opt/splunk/etc/system/local/outputs.conf*

> outputs.conf
```
[indexAndForward]
index = true
selectiveIndexing = true
```

#### 2. Create server class for org_all_deploymentclient
Once an instance has subscribed as a deployment client, the deploymentclient app should be switched to managed by the Deployment Server.  This is done in the event that you ever need to change to a new Deployment Server.  You can make the change on the old DS, point the configuration at the new DS, and it should switch over automatically.

> Instance: DS
>
> App: [org_all_deploymentclient](https://drive.google.com/drive/folders/11oJsdNJgX8JjMvf3xSGoXZxXsTGZKsmD)

Copy the org_all_deploymentclient app to the */opt/splunk/etc/deployment-apps* directory.  It should be the same app you installed on the HF.

> **NOTE**: This app will not be distributed to the CM, but it will be distributed to the DP.  It will need to be copied manually to the */opt/splunk/etc/apps* directory on the:
> - DP/MC/LM
> - CM
> - DS
>
> It will also need to be manually deployed from the DP to the SHC. 

> Instance: DS
>
> Apps: None
>
> Location: */opt/splunk/etc/system/local/serverclass.conf*

> serverclass.conf
```
[serverClass:all_splunk_instances:app:org_all_deploymentclient]
restartSplunkWeb = 0
restartSplunkd = 1
stateOnClient = enabled

[serverClass:dp_appstodistribute:app:org_all_deploymentclient]
restartSplunkWeb = 0
restartSplunkd = 0
stateOnClient = noop
```

> **NOTE**: I recommend keeping all the apps that belong to a given serverclass together in the serverclass.conf file.  This makes editing easier.

#### 3. Create servercalss for org_full_license_server

As mentioned above, the *org_full_license_server* app on the Cluster Manager needs to be distributed to all Splunk instances.

It will be sent to the following instances:
- DP/MC/LM
- CM
- HF

From the DP and IDX it will be distributed to the indexers and search heads.

> Instance: DS
>
> App: [org_full_license_server](https://drive.google.com/drive/folders/11AgVMe46mdDyrYU5oxVE2xNifPK1m5Dk)

Copy the *org_full_license_server* app to the */opt/splunk/etc/deployment-apps* directory on the Deployment Server.

This app will be distributed in the same way as the deploymenyclient app above, with the addition of needing to be distributed to the CM and Indexers.

> Instance: DS
>
> Apps: None
>
> Location: */opt/splunk/etc/system/local/serverclass.conf*

> serverclass.conf
```
[serverClass:all_splunk_instances:app:org_full_license_server]
restartSplunkWeb = 0
restartSplunkd = 1
stateOnClient = enabled

[serverClass:dp_appstodistribute:app:org_full_license_server]
restartSplunkWeb = 0
restartSplunkd = 0
stateOnClient = noop

[serverClass:cm_appstodistribute]
whitelist.0 = <CM IP>

[serverClass:cm_appstodistribute:app:org_full_license_server]
restartSplunkWeb = 0
restartSplunkd = 0
stateOnClient = noop
```

Reload the deployment server:
```
/opt/splunk/bin/splunk reload deploy-server
```

---
---

### Deploy Apps to the Indexers and Search Head Clusters

#### 1. Deploy apps to the Indexer Cluster
> Instance: CM
```
su splunk

cd /opt/splunk/bin
```
Validate a new bundle:
```
./splunk validate cluster-bundle
```
Verify that there are no errors
```
./splunk show cluster-bundle-status
```
If there are errors or warnings, address them.  If not, apply the new bundle:
```
./splunk apply cluster-bundle
```
Verify that the new bundle is being/has been applied.  Check the active bundle's hash and compare it against the hash genereated dirong the validation step.  Also, check if the instances are in need of being restarted.
```
./splunk show cluster-bundle-status
```


#### 2. Deploy apps to the Search Head Cluster
> Instance: DP/MC/LM
```
su splunk

cd /opt/splunk/bin
```
Verify that all the apps needed for your bundle are in the /opt/splunk/etc/shcluster/apps directory
```
ls -l /opt/splunk/etc/shcluster/apps
```
If all your apps are present, apply the new app bundle.

Unlike the Indexer Cluster, there is no validation process for the search head cluster bundles.  When you run the following command, if there are errors related to pre-deployment, its likely that there are errors in one of your apps.
```
./splunk apply shcluster-bundle
```

After a few minutes, verify that the apps have been distributed.
> Instance: SH1/2
```
ls -l /opt/splunk/etc/apps
```

---
---

### Install apps on the CM, DP/MC/LM, and DS
Due to how distribution works in this environment, certain apps have to be installed manually.  Any apps that need to run on the:
- Deployment Server
- Deployer/Monitoring Console/License Manager
- Cluster Manager

have to be installed manually.  Most of these apps can be copied directly from the instance's specific distribution folder:

Instace|Distribution Folder
-|-
DS|/opt/splunk/etc/deployment-apps
DP/MC/LM|/opt/splunk/etc/shcluster/apps
CM|/opt/splunk/etc/manager-apps

The following apps need to copied to the */opt/splunk/etc/apps* directory on each machine:
App|Purpose|Notes
-|-|-
org_clustser_forwarder_outputs|Forward the instance's internal logs to the indexers|
org_full_license_server|Connect the instances to the License Manager|This is not needed on the DP/MC/LM

Copy these apps to the /opt/splunk/etc/apps directory on each instance and restart Splunk

```
# as the root user
systemctl restart Splunkd
```
---
---

### Setting up the Monitoring Console

The Monitoring Console acts a single point of reference for the overall health of your environment.  It is a place to quickly check things like ingest queues, available indexers, indexes, Search head health, etc.

To get the Monitoring Console working properly, it has to be able to talk to each of the instances in your splunk environment.  To do this, most instances need to be connected as Search Peers.  (The exception is Indexers.)

#### 1. Connect necessary instances as Search Peers
> Instance: DP/MC/LM

In the Splunk UI:
1. Go to: **Settings > Distributed Search**.
2. Click *Add new* in the "Search peers" row.
3. Fillin the following information:
> Item|Purpose|Example
> -|-|-
> Peer URI|The full URI (including https://) of the target instance|https://192.168.10.20:8089
> Remote username|The username of the main admin account on the target instance|admin
> Remote password|The password of the admin account on the target instance|abc123
> Confirm password|same as above|abc123

> **Note**: Despite not being marked as reuqired, failure to confirm the password will result in a failed connection.

Add the following instances as Search Peers:
- SH1
- SH2
- CM
- HF
- DS

#### 2. Connect the DP/MC/LM instance to the Indexer Cluster
In order to collect needed data from the Indexers, the Monitoring Console instance needs to be subscribed to the Cluster as a Search Head.

> **NOTE**: The following app is identical to the one on the DS.  You can copy that same app to the DP/MC/LM with the scp command:
> 
>> Instance: DS
> ```
> scp -r /opt/splunk/etc/deployment-apps <root/admin account>@<DPMCLM IP>:/tmp
> ```
> Then copy to the app to the /etc/apps directory on the DP/MC/LM
>> Intsance: DP/MC/LM
> ```
> # Switch to the splunk user
> su splunk
>
> cp -r /tmp/org_cluster_search_base /opt/splunk/etc/apps
> ```
> 
> Once copied, restart Splunk
> ```
> # As the root user
> systemctl restart Splunkd
> ```

> Instance: DP/MC/LM
>
> App: [org_cluster_search_base](https://drive.google.com/drive/folders/12YGuQZTANzrm3lfxaUgIH9JcXb3cQhXy)
>
> Location: /opt/splunk/etc/apps

> server.conf
```
[clustering]
mode = searchhead
manager_uri = clustermanager:one

[clustermanager:one]
manager_uri=https://<CM IP>:8089
pass4SymmKey = <KEY>
multisite = false
```
> **NOTE**: The <KEY> value is the pass4SymmKey for the Indexer Cluster.

Restart Splunk
```
# As the root user
systemctl restart Splunkd
```

#### 3. Configure Distributed Monitoring
> Instance: DP/MC/LM - UI
1. Go to **Settings > Monitoring Console > Settings > General Setup**
2. Next to **Mode**, select *Distributed*
3. After the menu reloads, you should see all the Splunk Enterprise (non-UF) instances in your environment listed.
4. Edit each of the instances to adjust their Server Roles:
> Instance Type|Server Roles
> -|-
> CM|Cluster Manager
> DP/MC/LM|Search Head<br>License Manager<br>SHC Deployer
> DS|Deployment Server
> HF|Indexer
> IDX|Indexer
> SH|Search Head<br>KV Store
5. Hit *Apply Changes*
    - Note: You may see warnings related to cluster labeling.  These will not present problems to your environment functioning.
6. Once the changes have been applied, go to the Overview screen and you should see all instances in your environment represented.  This screen gives you a quick overview of your environment's health.

---
---
