# Contents
- [Lab 1](#lab-1) - Deploy Environment
  - Install HDP & HDF
  - Bring up the Environment
- [Lab Start](#lab-start) - Refresh NiFi concepts
- [Lab 2](#lab-2) - Getting started with NiFi
  - Consuming the Meetup RSVP stream
  - Extracting JSON elements we are interested in
  - Splitting JSON into smaller fragments
  - Writing JSON to File System
- [Lab 3](#lab-3) - MiNiFi
  - Enable Site2Site in NiFi
  - Designing the MiNiFi Flow
  - Preparing the flow
  - Running MiNiFi
- [Lab 4](#lab-4) - Kafka Basics
  - Creating a topic
  - Producing data
  - Consuming data
- [Lab 5](#lab-5) - Integrating Kafka with NiFi
  - Creating the Kafka topic
  - Adding the Kafka producer processor
  - Verifying the data is flowing
- [Lab 6](#lab-6) - Integrating the Schema Registry
  - Creating the Kafka topic
  - Adding the Meetup Avro Schema
  - Sending Avro data to Kafka
- [Lab 7](#lab-7) - Integrating the NiFi Registry
  - Working with NiFi Variables
  - Creating NiFi Registry
  - Perform flow changes and commit to Registry  
- [Lab 8](#lab-8) - Real-time Analytics with SAM
  - Preparing the SAM Environment
  - Developing the SAM Application
  - Deploying the SAM Application
- [Lab 9](#lab-9) - Real-Time Data Visualization with Superset
  - Locating the Druid Data source
  - Create a Visualization
  - Create a Superset Dashboard


---------------

# Lab 1

This lab is to deploy HDF.

You should have a virtual machine allocated for a Linux Centos 7 VM to deploy HDF. Credentials to the VM will be provided by the instructor. Open an ssh session on your allocated VM and switch to root as a starting point for this lab (using ```sudo sh``` or ```sudo -i```).

For complete instructions, you can follow the [Official HDF 3.3 Documentation](https://docs.hortonworks.com/HDPDocuments/HDF3/HDF-3.3.0/installing-hdf/content/install-ambari.html) to deploy HDF 3.3. In the following instructions, we are applying these steps to deploy and install an HDF 3.3 environment.

## Apply prerequisites and prepare the environment

For this environment, we will use MySQL Community Edition as the database required for Streaming Analytics Manager, and the Schema Registry.

### Install required packages

Install MySQL and other prerequisites packages

```
yum localinstall -y https://dev.mysql.com/get/mysql57-community-release-el7-8.noarch.rpm
yum install -y git python-argparse epel-release mysql-connector-java* mysql-community-server nc curl ntp openssl python zlib wget unzip openssh-clients
```

### Apply Prerequisites for Ambari Server deployment

1. Disable ipv6: 
Create the file /etc/sysctl.d/99-hadoop-ipv6.conf containing the following configuration settings:
```
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
```
Apply the configuration changes using the following command:
```
sysctl -e -p /etc/sysctl.d/99-hadoop-ipv6.conf
```

2. Disable Transparent Huge Pages:
Check the existing setting. The value inside the brackets is the existing setting:
```
cat /sys/kernel/mm/transparent_hugepage/enabled
cat /sys/kernel/mm/transparent_hugepage/defrag
```
If the value is \[always\], you can change it as follows:
```
echo 'never' > /sys/kernel/mm/transparent_hugepage/enabled
echo 'never' > /sys/kernel/mm/transparent_hugepage/defrag
```

3. Disable selinux:
Check the current value:
```sestatus```
If enabled, you can disable it as follows:
```setenforce 0```
Edit ```/etc/selinux/config``` and set the SELINUX mode to ```disabled```:
```
# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#     enforcing - SELinux security policy is enforced.
#     permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
SELINUX=disabled
# SELINUXTYPE= can take one of three two values:
#     targeted - Targeted processes are protected,
#     minimum - Modification of targeted policy. Only selected processes are protected.
#     mls - Multi Level Security protection.
SELINUXTYPE=targeted
```

4. Disable firewalld/iptables:
Check if firewalld is installed:
```
yum list installed | grep firewalld
```
If this doesn't return anything, firewalld is not installed. You can skip the next step.
If firewalld is found, run the following commands:
```
systemctl disable firewalld
systemctl stop firewalld
```
Check if iptables/ip6tables is running:
```
systemctl status iptables
systemctl status ip6tables
```
If the service is installed, disable it using the commands below:
```
chkconfig iptables off
service iptables stop
chkconfig ip6tables off
service ip6tables stop
```

5. Enable ntpd:
```
ntpd -qg
chkconfig ntpd on 
service ntpd restart
```

6. Install Java:
Install OpenJDK Java version 1.8:
```
yum install -y java-1.8.0-openjdk-devel
mkdir -p /usr/java
ln -sf /etc/alternatives/java_sdk /usr/java/default
```
7. This lab requires at least 40 GB of RAM on your VM. If you have 40 GB of RAM or more, you can skip this step. If you have 32 GB of RAM, please add 8GB of swap space as 32 GB of memory will not be sufficient for this lab:
```
dd if=/dev/zero of=/swapfile bs=4K count=2M
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
free -m
```
You should see 8GB of swap space added in the free -m output above.

### Setup MySQL Databases for HDP & HDF

1. Enable and start MySQL service:
```
sudo systemctl enable mysqld.service
sudo systemctl start mysqld.service
```
2. Create the following mysql-setup.sql script:
```
ALTER USER 'root'@'localhost' IDENTIFIED BY 'Secur1ty!'; 
uninstall plugin validate_password;
CREATE DATABASE registry DEFAULT CHARACTER SET utf8; 
CREATE DATABASE streamline DEFAULT CHARACTER SET utf8; 
CREATE DATABASE druid DEFAULT CHARACTER SET utf8;
CREATE DATABASE superset DEFAULT CHARACTER SET utf8;
CREATE USER 'registry'@'%' IDENTIFIED BY 'StrongPassword'; 
CREATE USER 'streamline'@'%' IDENTIFIED BY 'StrongPassword';
CREATE USER 'druid'@'%' IDENTIFIED BY 'StrongPassword';
CREATE USER 'superset'@'%' IDENTIFIED BY 'StrongPassword';
GRANT ALL PRIVILEGES ON *.* TO 'registry'@'%' WITH GRANT OPTION ; 
GRANT ALL PRIVILEGES ON *.* TO 'streamline'@'%' WITH GRANT OPTION ;
GRANT ALL PRIVILEGES ON *.* TO 'druid'@'%' WITH GRANT OPTION;
GRANT ALL PRIVILEGES ON *.* TO 'superset'@'%' WITH GRANT OPTION;
commit;
```
3. Identify the password created by default and setup a new password. You can choose a password of your own and set it up in the following script. By default, it is StrongPassword:
```
#extract system generated Mysql password
oldpass=$( grep 'temporary.*root@localhost' /var/log/mysqld.log | tail -n 1 | sed 's/.*root@localhost: //' )
echo $oldpass
export db_password=${db_password:-StrongPassword}
```
4. Run the script mysql-setup created previously:
```
mysql -h localhost -u root -p"$oldpass" --connect-expired-password < mysql-setup.sql
```
5. Change root user password for mysql:
```
mysqladmin -u root -p'Secur1ty!' password ${db_password}
```
6. Test if the password changes were taken into effect:
```
mysql -u root -p${db_password} -e 'show databases;'
```
You should see a list of databases being returned:
```
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| registry           |
| streamline         |
| sys                |
+--------------------+
```

## Deploy Ambari

1. Download the Ambari repository
```
wget -nv http://public-repo-1.hortonworks.com/ambari/centos7/2.x/updates/2.7.1.0/ambari.repo -O /etc/yum.repos.d/ambari.repo
```
Verify that the repository has been added:
```
yum repolist
```

2. Install Ambari agent:
```
yum install -y ambari-agent
```

Edit ```/etc/ambari-agent/conf/ambari-agent.ini``` and add the parameter ```force_https_protocol=PROTOCOL_TLSv1_2``` at the \[security\] section of the file:
```
[security]
force_https_protocol=PROTOCOL_TLSv1_2
keysdir=/var/lib/ambari-agent/keys
server_crt=ca.crt
passphrase_env_var_name=AMBARI_PASSPHRASE
ssl_verify_cert=0
credential_lib_dir=/var/lib/ambari-agent/cred/lib
credential_conf_dir=/var/lib/ambari-agent/cred/conf
credential_shell_cmd=org.apache.hadoop.security.alias.CredentialShell
```
This is to workaround some jdk changes disabling [TLSv1](https://bugzilla.redhat.com/show_bug.cgi?id=1577372)

3. Start the Ambari agent:
```
chkconfig ambari-agent on
ambari-agent start
```

4. Install Ambari server:
```
yum install -y ambari-server
ambari-server setup -j /usr/java/default -s
```
5. Start Ambari server
```
ambari-server start
```
Make sure Ambari starts successfully.

## Ambari Server post-install steps

1. Setup MySQL JDBC Driver with Ambari:
```
ambari-server setup --jdbc-db=mysql --jdbc-driver=/usr/share/java/mysql-connector-java.jar
```
2. Install HDF MPack:
```
export mpack_url="http://public-repo-1.hortonworks.com/HDF/amazonlinux2/3.x/updates/3.3.0.0/tars/hdf_ambari_mp/hdf-ambari-mpack-3.3.0.0-165.tar.gz"
ambari-server install-mpack --verbose --mpack=${mpack_url}
```
3. Restart Ambari
```
ambari-server restart
```

## Deploy HDP and HDF

In this section, please proceed with an HDP + HDF installation using the Ambari wizard. Login to Ambari web UI by opening http://{YOUR_IP}:8080 and log in with **admin/admin**

1. For the version, select HDP-3.0.1
2. Choose RedHat7 for the Operating System
3. For install options, enter the FQDN of your host (output of ```hostname -f``` on your VM) and for Host Registration, select ```Perform manual registration on hosts and do not use SSH```.
4. Perform Host Registration. If it doesn't work, check the log.
5. **For the Services to be installed, please only choose the following ones:**
	- HDFS
	- YARN + MapReduce2
	- Tez
	- Pig
	- Zookeeper
	- Storm
	- Infra Solr
	- Ambari Metrics
	- Kafka
	- Log Search
	- Druid
	- NiFi
	- NiFi Registry
	- Schema Registry
	- Streaming Analytics Manager
	- Superset
	
	You will get some warnings about Limited Functionality for not selecting Apache Atlas, and Apache Ranger. Just click ```Proceed 	Anyway```.
6. For credentials, please use ```StrongPassword``` as the password for all components.
7. For databases, select ```MYSQL``` for Druid Metadata Storage type and for Superset.
8. In the superset configuration, in the ```ADVANCED``` tab, section ```Advanced superset```, please set SUPERSET_WEBSERVER_PORT to ```19088```. Leave all other options by default, and keep clicking ```Next```.
9. On the ```All Configurations``` tab, there is a bell in red. Click on the bell.
10. For the required configuration, edit the parameters for the passwords and use ```StrongPassword``` as the password for all parameters.
11. Click on ```Next``` and ```Deploy```.
12. Wait for installation to complete. This should take anywhere from 30 to 60 minutes, depending on your VM performance.
13. After installation will complete, Ambari will start all services. 
If some of the services start-up fails, it will abort starting-up all remaining services, and your installation will complete but with all the services down. Do not panic. Start all services individually, starting up with core Hadoop services first (HDFS, Zookeeper, YARN, MapReduce) followed by all other services with the exception of Smartsense (You can put Smartsense in Maintenance Mode). 
For any service failing to start-up, please check the log.

## Access your cluster

After installation:

- Login to Ambari web UI by opening http://{YOUR_IP}:8080 and log in with **admin/admin**

- You will see a list of Hadoop components running on your node on the left side of the page
  - They should all show green (ie started) status, except for SmartSense. If not, start them by Ambari via 'Service Actions' menu for that service.
  - If there are any errors starting any of the components, please start the service, check for any errors during startup and 		troubleshoot.
  - Once all services are started, please click on the NiFi service. On the right hand side, you will see a section called ```Quick Links```. Click on ```NiFi UI```. 
  - You are now ready to start the Lab 2.

-----------------------------

# Lab Start

Refreshing a few basic concepts around NiFi
- Put simply NiFi was built to automate the flow of data between systems. While the term 'dataflow' is used in a variety of contexts, we use it here to mean the automated and managed flow of information between systems. This problem space has been around ever since enterprises had more than one system, where some of the systems created data and some of the systems consumed data.
- Key terms
  - Flowfile : A FlowFile represents each object moving through the system and for each one, NiFi keeps track of a map of key/value pair attribute strings and its associated content of zero or more bytes.
  - Flowfile Processor : 	
Processors actually perform the work. A processor is doing some combination of data routing, transformation, or mediation between systems. Processors have access to attributes of a given FlowFile and its content stream. Processors can operate on zero or more FlowFiles in a given unit of work and either commit that work or rollback.
  - Connection : Connections provide the actual linkage between processors. These act as queues and allow various processes to interact at differing rates. These queues can be prioritized dynamically and can have upper bounds on load, which enable back pressure.
  - Flow Controller : The Flow Controller maintains the knowledge of how processes connect and manages the threads and allocations thereof which all processes use. The Flow Controller acts as the broker facilitating the exchange of FlowFiles between processors.
  - Process Group : 	
A Process Group is a specific set of processes and their connections, which can receive data via input ports and send data out via output ports. In this manner, process groups allow creation of entirely new components simply by composition of other components.


Keep the [NiFi Docs](http://nifi.apache.org/docs.html) specifically the User Guide handy.

-----------------------------

# Lab 2

In this lab, we will learn how to consume data from an external system, extract content that we are interested in and then store the data in our system.

Specifically,

  - Consume a JSON Stream (The Meetup RSVP Stream)
  - Extract the JSON elements we are interested in
  - Split the JSON into smaller fragments for analysis
  - Write the JSON to the file system

 Our final flow for this lab will look like the following:

  ![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/lab1.png)

   
  The final template for this flow can be found [here](https://github.com/dhananjaymehta/HDF-Workshop/blob/master/templates/HDF-Workshop_Lab1-Flow.xml)

  A good plan would be to refer to the template at the end after going through individual steps.

### Steps

1. With a blank canvas, click on the Configuration gear icon in the Operate box on the left side of the UI:

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab2_step1.png)

2. Under the CONTROLLER SERVICES tab, Add a JettyWebSocketClient service and click on the gear icon to edit the configure the controller service.

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab2_step2.png)

3. Under the PROPERTIES tab add the value for URI for the last property WebSocket URI, and paste it for the empty value for WebSocket URI in bold. The value pasted is ```ws://stream.meetup.com/2/rsvps```Lab2_step3. Your configuration should look like this:

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab2_step3.png)

4. Notice that the state for the Controller Service is Disabled. Click on the lightning icon on the right to enable it:

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab2_step4.png)

5. Add a ConnectWebSocket processor to the canvas by dragging the icon on the page:

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab2_step5.png)
  
![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab2_step5b.png)

6. Configure the ConnectWebSocket Processor so it looks like below:
  - Under the properties tab set the WebSocket Client Controller Service
  - Set the WebSocket Client ID to AGP-HDF-WS-TEST
  (Note this can be anything)
  ![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab2_step6.png)
  - Set the automatic termination as below
  ![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab2_step6b.png)

7. Add an UpdateAttribute Processor to the canvas using the same method as previously:

  - UpdateAttribute Processor
  ![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab2_step7.png)
  
  - Configure it to have a custom property called mime.type with the value of application/json:
  ![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab2_step7b.png)

8. Join ConnectWebSocket Processor and the UpdateAttribute Processor using a text message for relationships.

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab2_step8.png)
![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab2_step8b.png)

9. Add an EvaluateJsonPath processor and configure it as shown below:

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/jsonpath.png)

    
  Note the change in the value of Destination property to ```flowfile-attribute```

  The properties to add are:

    ```
    event.name      $.event.event_name
    event.url       $.event.event_url
    group.city      $.group.group_city
    group.state     $.group.group_state
    group.country   $.group.group_country
    group.name      $.group.group_name
    venue.lat       $.venue.lat
    venue.lon       $.venue.lon
    venue.name      $.venue.venue_name
    ```

10. Join the UpdateAttribute processor and EvaluateJsonPath processor by dragging the arrow from UpdateAttribute to EvaluateJsonPath. This is for matched relationship.

Also add a failure relationship (Note: recursive join)

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab2_step10b.png)

Auto-terminate unmatched relationships in the settings tab:

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab2_step10c.png)

11. Add a SplitJson processor and configure the JsonPath Expression to be ```$.group.group_topics ```. 

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab2_step11.png)

Also the Original relationship needs to be automatically terminated.  Your configuration should look like below:

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab2_step11b.png)

12. Join the EvaluateJsonPath processor and the SplitJson processor.

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab2_step12c.png)

In addition, create a failure recursive join on the SplitJson Processor. Should look like the below.

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab2_step12b.png)

13. Add a ReplaceText processor and configure the Search Value to be ```([{])([\S\s]+)([}])``` and the Replacement Value to be
    ```
         {
        "event_name": "${event.name}",
        "event_url": "${event.url}",
        "venue" : {
        	"lat": "${venue.lat}",
        	"lon": "${venue.lon}",
        	"name": "${venue.name}"
        },
        "meetupgroup" : {
          "group_city" : "${group.city}",
          "group_country" : "${group.country}",
          "group_name" : "${group.name}",
          "group_state" : "${group.state}",
          $2
         }
      }
      ```
  The processor should look like the below
  
  ![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab2_step13.png)

14. Join the SplitJson processor and the ReplaceText processor.

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab2_step14.png)

In addition add a Failure recursive join.

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab2_step14c.png)
![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab2_step14b.png)

15. Add a PutFile processor to the canvas and configure it to write the data out to ```/tmp/rsvp-data```. Automatically terminate both on Success and Failure. The configuration should look like below.

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab2_step15.png)
![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab2_step15b.png)

16. Join the ReplaceText processor and the PutFile processor for successful relationships.

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab2_step16.png)
![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab2_step16b.png)

17. Click on an empty part of the canvas. You have now selected the entire NiFi Flow.Start the flow by clicking on the ```Play``` triangle icon in the Operate window:

  ![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab2_step17.png)

You should see data flowing after a couple of minutes.

##### Mandatory: Questions to Answer
1. Why did you assign a mime.type to UpdateAttribute Processor?
2. What does terminate relationship in a Processor mean? How is this different from auto-terminate?
3. What does a full RSVP Json object look like? Where would you find this?
4. In Putfile processor, what happens when more than 1 file with the same name is received? How do you resolve this?
5. How do you re-start a Processor?
6. How do you stop all the processors currently running in the Flow?
7. Login to your cluster and check if the files have been stored in the correct directory?
8. Open a file and check its contents. How does it look compared to the full RSVP JSON Object in the Meetup Stream?



##### Optional: Questions to Answer
1. How many output files do you end up with?
2. How can you change the file name that Json is saved as from PutFile?
3. Why do you think we are splitting out the RSVP's by group?
4. How can you change the flow to get the member photo from the Json and download it.


### Post Lab 2
1. After completing this lab, create a new Process Group by dragging onto the canvas the Process Group icon and call it Lab2:

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab2_step18.png)

2. Select all the components of your flow including processors, and links between processors by pressing shift and selecting an area of the canvas which contains all the flow. You will see a rectangle being drawn in the canvas corresponding to the area selected. You may need to zoom out. Click on copy in the Operate area.

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab2_step19.png)

3. Double-click on the Lab2 process group. When a new canvass opens for the Lab2 process group, right-click and select ``` Paste ```. You should now have the entire flow pasted into this process group.

4. Click on the main NiFi flow to go back on the main canvas. Select the flow as per step 19, right-click and select ``` Delete ```.

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab2_step21.png)
![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab2_step21b.png)

You should now have an empty canvas to start for Lab 3.

------------------

# Lab 3

  ![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/lab3.png)
  A template for this flow can be found [here](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/templates/MiNiFi_Flow.xml)

## Download and Install MiNiFi

Check if MiNiFi has been installed under /usr/hdf/current or /usr/hdf in your cluster.

If it has been installed then ignore the below and jump to "Getting Started With MiNiFi"

If MiNiFi has not been installed then, run the below commands from a terminal as root:
```
mkdir -p /tmp/minifi

wget -P /tmp/minifi http://public-repo-1.hortonworks.com/HDF/3.4.0.0/minifi-0.6.0.3.4.0.0-155-bin.tar.gz

wget -P /tmp/minifi http://public-repo-1.hortonworks.com/HDF/3.4.0.0/minifi-toolkit-0.6.0.3.4.0.0-155-bin.tar.gz

cd /usr/hdf/3.4.0.0-155

tar zxvf /tmp/minifi/minifi-0.*-bin.tar.gz
tar zxvf /tmp/minifi/minifi-toolkit-0.*-bin.tar.gz

cd /usr/hdf/current
ln -s /usr/hdf/3.4.0.0-155/minifi-0.* minifi
ln -s /usr/hdf/3.4.0.0-155/minifi-toolkit-0.* minifi-toolkit
```

## Getting started with MiNiFi ##

In this lab, we will learn how to configure MiNiFi to send data to NiFi:

* Setting up the Flow for NiFi
* Setting up the Flow for MiNiFi
* Preparing the Flow for MiNiFi
* Configuring and starting MiNiFi
* Enjoying the data flow!

## Setting up the Flow for NiFi

1. Before starting NiFi we need to enable Site-to-Site communication. To do that we will use Ambari to update the required configuration. In Ambari the below property values can be found at ````http://<ambari_url>:8080/#/main/services/NIFI/configs```` .

* Change:
  ````
    nifi.remote.input.socket.port=
  ````
  To

  ```
    nifi.remote.input.socket.port=10000
  ```
  Save the configuration changes. Click on ```PROCEED ANYWAY``` if receiving warnings.
  
  ![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab3_step1.png)

2. Restart NiFi via Ambari

3. Access the NiFi UI from Ambari or ```http://<public_IP_addr>:9090/nifi/```

4. Now we should be ready to create our flow. The first thing we are going to do is setup an Input Port. This is the port that MiNiFi will be sending data to. To do this drag the Input Port icon to the canvas and call it "From MiNiFi":

````
NOTE: The Input port must be configured on the Main Flow canvas and not inside a Process Group. 
````

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab3_step4.png)

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab3_step4b.png)

5. Now that the Input Port is configured we need to have a place for the data to go once we receive it. In this case we will keep it very simple and just log the attributes. To do this drag the Processor icon to the canvas and choose the LogAttribute processor.

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab3_step5.png)

6. On the Settings tab, under Auto-terminate relationships, select the checkbox next to Success. This will terminate FlowFiles after this processor has successfully processed them.

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab3_step5b.png)

7. Also on the Settings tab, set the Bulletin level to Info. This way, when the dataflow is running, this processor will display the bulletin icon, and the user may hover over it with the mouse to see the attributes that the processor is logging.

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab3_step7.png)

8. Now that we have the input port and the processor to handle our data, we need to connect them. After creating the connection your data flow should look like this:

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab3_step8.png)

9. We are now ready to build the MiNiFi side of the flow. To do this do the following:

  - Add a GenerateFlowFile processor to the canvas. On the Scheduling tab, set Run schedule to: 5 sec. Note that the GenerateFlowFile processor can create many FlowFiles very quickly; that's why setting the Run schedule is important so that this flow does not overwhelm the system NiFi is running on.
  
  ![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab3_step9.png)
  
  - On the Properties tab, set File Size to: 10 kb	
  
  ![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab3_step9b.png)
  
  - Add a Remote Processor Group to the canvas. Drag and drop a remote processor group to the canvas:
  
  ![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab3_step9b2.png)
    
  For the URL copy and paste the URL for the NiFi UI from your browser:
  
  ![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab3_step9c.png)

  - Note that the URL above is specific to that instance and your URL may be different.
  
  - Connect the GenerateFlowFile to the Remote Process Group. Your canvas should look like the following:
  
  ![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab3_step9d.png)
    

10. The next step is to generate the flow we need for MiNiFi. To do this we will do the following :

  - Create a template for MiNiFi
    * Select the GenerateFlowFile, the NiFi Flow Remote Processor Group, and the Connection between them (these are the only things needed for MiNiFi).
    * Select the "Create Template" button from the toolbar
    * Name your template: MiNiFi Flow
   
   ![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab3_step10.png)
  
11. Download the created template. Select ```Templates```

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab3_step11.png)

12. Copy the template you downloaded to the ````/tmp```` directory on your cluster. If you are using Windows on your workstation, you will need to download [WinSCP](https://winscp.net/eng/download.php)

On OSX and Linux:
```
scp -i <key.pem> MiNiFi_Flow.xml centos@<IPADDRESS>:/tmp/.
```

13.  We are now ready to setup MiNiFi. However before doing that we need to convert the template to YAML format which MiNiFi uses. To do this we need to do the following:

- Navigate to the minifi-toolkit directory

```
cd /usr/hdf/current/minifi-toolkit
```

- Transform the template that we downloaded using the following command: ````bin/config.sh transform <INPUT_TEMPLATE> <OUTPUT_FILE>````

  Example:
  ```
  sudo bin/config.sh transform /tmp/MiniFi_Flow.xml config.yml
  ```

14. Next copy the ````config.yml```` to the ````/usr/hdf/current/minifi/conf```` directory. This is the file that MiNiFi uses to generate the nifi.properties file and the flow.xml.gz for MiNiFi.

```
cd /usr/hdf/current/minifi/conf
sudo cp /usr/hdf/current/minifi-toolkit/config.yml .
```
15. That is it, we are now ready to start MiNiFi. To start MiNiFi from a command prompt execute the following:

  ```
  cd /usr/hdf/current/minifi
  sudo bin/minifi.sh start
  sudo tail -100f logs/minifi-app.log
  ```
16. Start your flows on NiFi by clicking on the ```Play``` triangle icon in the Operate window.

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab3_step12.png)

You should be able to now go to your NiFi flow and see data coming in from MiNiFi.

You may tail the log of the MiNiFi application by
   ```
   tail -100f /usr/hdf/current/minifi/logs/minifi-app.log
   ```
If you see error logs such as "the remote instance indicates that the port is not in a valid state",
it is because the Input Port has not been started.
Start the port and you will see messages being accumulated in its downstream queue.

17. You should see messages coming in through LogAttribute. Your canvas should look like the following:

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab3_step13.png)

18. Shut down the NiFi flow in the canvas and stop minifi via the command line.

```
cd /usr/hdf/current/minifi
sudo bin/minifi.sh stop
```

##### Mandatory: Questions to Answer
1. Look at the config.yml, whats the main property here that needs to be changed if you want to re-use the config?
2. What happens when "From MiNiFI" processor is stopped? Why?
3. What are the other command line parameters of minifi?

------------------

# Lab 4

## Kafka Basics
In this lab we are going to explore creating, writing to and consuming Kafka topics. This will come in handy when we later integrate Kafka with NiFi and Streaming Analytics Manager.

1. Creating a Topic
- Step 1: Open an SSH connection to your cluster. NOTE:You might have to run some of the commands as root.

- Step 2: Navigate to the Kafka directory 
````/usr/hdf/current/kafka-broker````, this is where Kafka is installed, we will use the utilities located in the bin directory.

  ````
  cd /usr/hdf/current/kafka-broker
  ````

- Step 3: Create a topic using the ````kafka-topics.sh```` script

  ````
  bin/kafka-topics.sh --zookeeper localhost:2181 --create --partitions 1 --replication-factor 1 --topic first-topic
  ````
  NOTE: Based on how Kafka reports metrics topics with a period ('.') or underscore ('_') may collide with metric names and should be avoided. If they cannot be avoided, then you should only use one of them.

- Step 4:	Ensure the topic was created

  ````
  bin/kafka-topics.sh --list --zookeeper localhost:2181
  ````

2. Testing Producers and Consumers
- Step 1: Open a second terminal to your cluster and navigate to the Kafka directory

- Step 2: In one shell window connect a consumer, customize the FQDN with the hostname of your cluster:

  ````
  cd /usr/hdf/current/kafka-broker
  bin/kafka-console-consumer.sh --bootstrap-server demo.hortonworks.com:6667 --from-beginning --topic first-topic
  ````

  Note 1: using –from-beginning will tell the broker we want to consume from the first message in the topic. Otherwise it will be from the latest offset.

- Step 3: In the second shell window connect a producer. Customize the FQDN in the broker-list with the hostname of your cluster:

  ````
  bin/kafka-console-producer.sh --broker-list bamako.field.hortonworks.com:6667 --topic first-topic
  ````
- Step 4: Sending messages. Now that the producer is connected we can type messages:
  - Type a random message in the producer window
  - Messages should appear in the consumer window.

- Step 5: Close the consumer (ctrl-c) and reconnect using the default offset, of latest. You will now see only new messages typed in the producer window.

````
bin/kafka-console-consumer.sh --bootstrap-server demo.hortonworks.com:6667 --topic first-topic
````
  As you type messages in the producer window they should appear in the consumer window.

- Step 6: Close both consumers and producers by using Ctrl+C on each session.

##### Optional: Questions to Answer
1. Check the options for Kafka Producer and Consumer. 
2. List the topics given , what is __consumer_offsets.

------------------

# Lab 5

## Integrating Kafka with NiFi
1. Creating the topic
  - Step 1: Open a SSH connection to your cluster.
  - Step 2: Navigate to the Kafka directory (````/usr/hdf/current/kafka-broker````), this is where Kafka is installed, we will use the utilities located in the bin directory.

    ````
    cd /usr/hdf/current/kafka-broker/
    ````

  - Step 3: Create a topic using the ````kafka-topics.sh```` script
    ````
    bin/kafka-topics.sh --zookeeper localhost:2181 --create --partitions 1 --replication-factor 1 --topic meetup_rsvp_raw

    ````

    NOTE: Based on how Kafka reports metrics topics with a period ('.') or underscore ('_') may collide with metric names and should be avoided. If they cannot be avoided, then you should only use one of them.

  - Step 4:	Ensure the topic was created
    ````
    bin/kafka-topics.sh --list --zookeeper localhost:2181
    ````

2. Integrating NiFi
  - Step 0: Re-use the Process Group from Lab2.

  - Step 1: Add a PublishKafka_2_0 processor to the canvas. Note that there are processors for other versions of Kafka as well. Choose the correct one.

  - Step 2: Add a routing for the success relationship of the ReplaceText processor to the PublishKafka_2_0 processor added in Step 1 as shown below:
  
  - Step 3: Configure the topic and broker for the PublishKafka_2_0 processor,
  where:
     - Topic is ```meetup_rsvp_raw```
     - Broker is ```<host-name-fqdn>:6667```
     - Use Transactions is set to ```false```
     
  ![Image](https://raw.githubusercontent.com/vsellappa/HDF-Workshop/master/img/Lab5_2_step3.png)
  
  - Step 4: In the Settings tab of the processor, select ```success``` for the ```Automatically Terminate Relationships```.
  
  - Step 5: Create a failure recursive join on the processor itself. Your flow should look like the following:
  
  ![Image](https://raw.githubusercontent.com/vsellappa/HDF-Workshop/master/img/Lab5_2_step3c.png)


3. Start the NiFi flow

4. Login to your cluster node and navigate to the Kafka directory and connect a consumer to the ````meetup_rsvp_raw```` topic:

    ````
    bin/kafka-console-consumer.sh --bootstrap-server demo.hortonworks.com:6667 --from-beginning --topic meetup_rsvp_raw
    ````

5. Messages should now appear in the consumer window.

6. Stop the NiFi flow and the consumer.

##### Mandatory: Questions to Answer
1a. What happens when the Kafka consumer is stopped and the NiFi producer continues to run?

1b. What happens when the Kafka consumer is running and the NiFi producer is stopped?

------------------

# Lab 6

## Integrating the Schema Registry

1. Creating the topic
  - Step 1: Open a SSH connection to your cluster.

  - Step 2: Navigate to the Kafka directory (````/usr/hdf/current/kafka-broker````), this is where Kafka is installed, we will use the utilities located in the bin directory.

    ````
    cd /usr/hdf/current/kafka-broker/
    ````

  - Step 3: Create a topic using the ````kafka-topics.sh```` script

    ````
    bin/kafka-topics.sh --zookeeper localhost:2181 --create --partitions 1 --replication-factor 1 --topic meetup_rsvp_avro
    ````

    NOTE: Based on how Kafka reports metrics topics with a period ('.') or underscore ('_') may collide with metric names and should be avoided. If they cannot be avoided, then you should only use one of them.

  - Step 4:	Ensure the topic was created

    ````
    bin/kafka-topics.sh --list --zookeeper localhost:2181
    ````

2. Adding the Schema to the Schema Registry
  - Step 1: Open a browser and navigate to the Schema Registry UI. You can get to this either from a drop down in Ambari, as shown below:

    ![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab6_2_step1.png)

    or by going to ````http://<host-FQDN>:7788````
    
  - Step 2: Create Meetup RSVP Schema in the Schema Registry

    1. Click on “+” button to add new schemas. A window called “Add New Schema” will appear.

    2. Fill in the fields of the ````Add Schema Dialog```` as follows:

        ![Image](https://github.com/dhananjaymehta/HDF-Workshop/raw/master/img/add_schema_dialog.png)

        For the Schema Text you can download it [here](https://raw.githubusercontent.com/zoharsan/HDF-Workshop/master/meetup_rsvp.asvc) and either copy and paste it or upload the file.

        Once the schema information fields have been filled and schema uploaded, click **Save**. You should now see the following:
	
	![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab6_2_step2.png)
	
   3. We are now ready to integrate the schema with NiFi
      - Step 1: Remove the PutFile and PublishKafka_2_0 processors from the canvas, we will not need them for this section. Before removing the processors, (ensure the NiFi flow is stopped), you will need to remove the links between ReplaceText and these processors. Select the links/processors on the canvas, and press delete.

      - Step 2: Add a UpdateAttribute processor to the canvas.

      - Step 3: Add a routing for the success relationship of the ReplaceText processor to the UpdateAttrbute processor added in Step 1.

      - Step 4: Configure the UpdateAttribute processor as shown below:

   ![Image](https://raw.githubusercontent.com/vsellappa/HDF-Workshop/master/img/update_attribute_schema_name.png)

  - Step 5: Add a JoltTransformJSON processor to the canvas.

  - Step 6: Add a routing for the success relationship of the UpdateAttribute processor to the JoltTransformJSON processor added in Step 5.

  - Step 7: Configure the JoltTransformJSON processor as shown below:

    ![Image](https://raw.githubusercontent.com/vsellappa/HDF-Workshop/master/img/jolt_transform_config.png)

    The JSON used in the 'Jolt Specification' property is as follows:

    ``
    {
      "venue": {
        "lat": ["=toDouble", 0.0],
        "lon": ["=toDouble", 0.0]
      }
    }
  ``
  - Step 8: Add a LogAttribute processor to the canvas. In the settings tab of the processor, select ```success``` in the ```Automatically Terminate Relationships```.

  - Step 9: Add a routing for the failure relationship of the JoltTransformJSON processor to the LogAttribute processor added in Step 8.

  - Step 10: Add a PublishKafkaRecord_2_0 to the canvas.

  - Step 11: Add a routing for the success relationship of the JoltTransformJSON processor to the PublishKafkaRecord_2_0 processor added in Step 10.

  - Step 12: Configure the PublishKafkaRecord_2_0 processor to look like the following:

	- Set Kafka Brokers to: ```<host-FQDN>:6667```
	- Set Topic Name to: ```meetup_rsvp_avro```
	- Set Use Transactions to: ```false```
	- Set Record Reader to: ```JsonTreeReader```. Note that you will have to first Select ```Create new Service...``` from the drop down.
	- Set Record Writer to: ```AvroRecordSetWriter```. Note that you will have to first Select ```Create new Service...``` from the drop down.
	
    ![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab6_step12.png)
       
       - In the Settings tab of the processor, select ```success``` for the ```Automatically Terminate Relationships``` like you did in the previous lab.
       - Create a failure recursive join on the processor itself like you did in the previous lab.

  - Step 13: When you configure the JsonTreeReader and AvroRecordSetWriter, you will first need to configure a schema registry controller service. The schema registry controller service we are going to use is the 'HWX Schema Registry'.
  
     - Click on the Configuration gear icon in the Operate box on the left side of the UI:

     ![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab2_step1.png)

     - Click on the '+' sign on the right hand side of the Controller Services window and select ```HortonworksSchemaRegistry```.
     
     ![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab6_step13_1.png)
     
     - Click on the settings (gear icon) for HortonworksSchemaRegistry:
     
     ![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab6_step13_2.png)
      
     - It should be configured as shown below. Customize the URL with the actual FQDN of your cluster:

    ![Image](https://github.com/dhananjaymehta/HDF-Workshop/raw/master/img/hwx_schema_registry_config.png)
     
     - Enable the HortonworksSchemaRegistry service controller by clicking on the lightning icon, next to the setting/gear icon:
     
     ![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab6_step13_3.png)
     
   - Step 14: Configure the JsonTreeReader. 
   
     - Click on the setting/gear icon next to JsonTreeReader controller service:
     
     ![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab6_step14.png)
     
     - Configure JsonTreeReader as shown below:

     ![Image](https://raw.githubusercontent.com/vsellappa/HDF-Workshop/master/img/Lab6_step14_2.png)
    
     - Enable the JsonTreeReader service controller by clicking on the lightning icon, next to the setting/gear icon, as you did for the HortonworksSchemaRegistry service.

  - Step 15: Configure the AvroRecordSetWriter:
  
     - Click on the setting/gear icon next to AvroRecordSetWriter controller service:	
     - Configure AvroRecordSetWriter as shown below:

      ![Image](https://raw.githubusercontent.com/vsellappa/HDF-Workshop/master/img/Lab6_step15.png)
      
     - Enable the AvroRecordSetWriter service controller by clicking on the lightning icon, next to the setting/gear icon, as you did for the HortonworksSchemaRegistry service.

    After following the above steps this section of your flow should look like the following:

    ![Image](https://raw.githubusercontent.com/vsellappa/HDF-Workshop/master/img/update_jolt_kafka_section.png)


4. Start the NiFi flow.

5. In a terminal window to your cluster, navigate to the Kafka directory and connect a consumer to the ````meetup_rsvp_avro```` topic, remembering to change the FQDN of your bootstrap-server:

    ````
    cd /usr/hdf/current/kafka-broker

    bin/kafka-console-consumer.sh --bootstrap-server demo.hortonworks.com:6667 --from-beginning --topic meetup_rsvp_avro
    ````

6. Messages should now appear in the consumer window.

7. Shutdown the NiFi flow and the consumer.

------------------

# Lab 7

## NiFi Registry
NiFi registry provides a central location for storage and management of shared resources. It allows storing and managing versioned flows. 

For this lab we are going to set up NiFi registry and use variables. 

1. Open Nifi Registry from Ambari UI - 
Ambari UI -> NiFi Registry -> NiFi Registry UI or visit ```<FQDN>:61080```

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab6_NR_01.png)

2. Create a Bucket to do version control on NiFi Registry UI. 

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab6_NR_02.png)

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab6_NR_021.png)
 

3. Right click on processor group ```Lab2``` to select Version Control.

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab6_NR_3.png)

4. The bucket created in NiFi Registry should automatically appear if it’s in the same cluster

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab6_NR_4.png)


## Variable Registry
```Nifi >1.4``` provides UI based variable registry to help simplify the configuration management of flows across different environments.  

Step 1: On the processor group Lab2 select "variables"

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab6_VR_01.png)

Step 2: Using + symbol add new variable and add corresponding value to the variable. In this example we are adding a variable for Kafka topic.

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab6_VR_02.png)
![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab6_VR_03.png)

Step 3: Apply the variable and close. 

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab6_VR_04.png)

Step 4: Go to PublishKafkaRecord_2_0 processor and update the ```Topic Name```

![Image](https://raw.githubusercontent.com/zoharsan/HDF-Workshop/master/img/Lab6_VR_05b.png)


Step 5: Now we can commit these changes,  Make changes to canvas such as position or time (even position change of processor is tracked). 

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab6_NR_5.png)


Step 6: You can see the changes that were made by selecting “Show Local Changes”

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab6_VR_051.png)


Step 7: Decide to Commit or Revert any changes that are made. If you decide to save the changes, comment before you commit about the changes. Note that you can configure to commit these changes to Github as well.

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab6_NR_7.png)

Step 8: The pushed changes will appear in NiFi Registry.

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab6_NR_8.png)

------------------

> # Sections below are outdated and have been kept here for legacy purposes and will be removed going forward.

------------------

# Lab 8

## Real-time Analytics with SAM

For this lab we are going to consume data from the previous NiFi application and develop a simple application with SAM, which does some basic analytics on the data.

### Prepare the SAM Environment.

1. Open the SAM UI from Ambari. From Ambari, click on Streaming Analytics Manager Service, then click on ```SAM UI``` from Quick Links on the right hand side of the Ambari console:

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab7_step1.png)

The following screen will appear:

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab7_step1b.png)

2. Define the Service Pool. As described in the welcome screen, we first need to define a Service Pool. Click on the tool icon on the left-hand side tool bar and select Service Pool in the menu:

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab7_step2.png)

3. Update the Ambari URL corresponding to your cluster by replacing the placeholders in the URL with the following values:
  - Ambari_host: Public IP of your VM.
  - Port: 8080
  - CLUSTER_NAME: Your cluster name
  Then, click on the AUTO-ADD button
  
  ![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab7_step3.png)
  
  You will be prompted for your Ambari credentials. Enter admin/admin.
  
  Once the cluster has been added successfully, you will see it appear as a service pool:
  
  ![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab7_step3b.png)
  
4. Define a Development Environment. Click on the tool icon on the left-hand side toolbar and select Environments:  
  
  ![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab7_step4.png)
  
  - In the new screen, please click Add (Green hexagon with the ‘+’ sign on the top right):  
  
  ![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab7_step4b.png)
  
  - Create a new Environment called Development, and select all services (They should be highlighted in blue):
  
  ![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab7_step4c.png)
  
  - Then, click OK. The new environment will appear as a new tile:
  
  ![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab7_step4d.png)
  
  At this point, we are ready to develop a new SAM (Streaming Analytics Manager) Application.
  
5. Click on the application icon on the left hand side toolbar, and select ‘My Application’. Click on Add (Green hexagon on the top righ with the ‘+’ sign), and select ```New Application```:

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab7_step5.png)

- Enter the following Application NAME: ‘MeetupSamApp’. It’s important that there are no spaces in your application name, as this could potentially cause some issues with Storm. For the environment, please select ‘Development’ that we just created previously.

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab7_step5b.png)

You will now have an empty canvas. We are ready to develop the SAM Application.

### Develop the SAM Application

1. Read data from Kafka. We first want to read data from source. As a source, we are going to use the Kafka topic on which we wrote with Apache NiFi on the previous section. 

- From the various operators available on the processor menu, please select Kafka from SOURCE, then drag and drop it onto the canvas:

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab72_step1.png)

- Double-click on the Kafka operator on the canvas, and enter the following values:

	- CLUSTER NAME: Your cluster name
	- SECURITY PROTOCOL: PLAINTEXT
	- BOOTSTRAP SERVERS: Leave default value
	- KAFKA_TOPIC: meetup_rsvp_avro
	- READER MASTER BRANCH: MASTER
	- READER SCHEMA VERSION: 1
	- CONSUMER GROUP ID: kafka_gid_0
	
![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab72_step1b.png)

You will notice that the schema will appear on the output. The schema is retrieved from the schema registry.

2. Ingest data into Druid. Druid is a column-oriented, open-source, distributed data store written in Java. Druid is designed to quickly ingest massive quantities of event data, and provide low-latency queries on top of the data. For real-time dashboards, we want to write data to Apache Druid and visualize the data later with a real-time dashboard implemented with Superset.

- Select the Druid processor from SINK, then drag and drop it on the canvas:

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab72_step2.png)

- At this point, link the ‘KAFKA’ and ‘DRUID’ operators by clicking on the green circle on the right hand side of the ‘KAFKA’ operator and bringing the arrow to the grey circle on the ‘DRUID’ operator:

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab72_step2b.png)

- Double-click on the ```DRUID``` operator and enter the following values:

	- DATASOURCE NAME: meetup-dsn
	- DIMENSIONS: Add all the available dimensions from the drop down list.
	- TIMESTAMP FIELD NAME: processingTime.
	- WINDOW PERIOD: PT5M
	- INDEX RETRY PERIOD: PT5M
	- SEGMENT GRANULARITY: FIVE_MINUTE
	- QUERY GRANULARITY: MINUTE

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab72_step2c.png)

3. As this is a JSON record with nested structures, we need to project all fields in order to do a SQL operation like an aggregate.

- Select the ```PROJECTION``` processor and drag and drop it to the canvas:

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab72_step3.png)

- Link the ‘KAFKA’ and ‘PROJECTION’ operators by clicking on the green circle on the right hand side of the ‘KAFKA’ operator and bringing the arrow to the grey circle on the ‘PROJECTION’ operator like you did in the previous step:

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab72_step3b.png)

- Double-click on the PROJECTION operator:

	- PROJECTION FIELDS: event_name, event_url
	- Add the following PROJECTION EXPRESSION:
		```
		- venue.name: venue_name
		- meetupgroup.group_city: group_city
		- meetupgroup.group_country: group_country
		- meetupgroup.group_name: group_name
		- meetupgroup.group_state: group_state
		- meetupgroup.urlkey: group_urlkey
		- meetupgroup.topic_name: group_topic_name
		```
![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab72_step3d.png)

4. For our real-time analytics component, we want to aggregate in real-time the number of RSVPs per Country, and City across all Meetup Groups to have a real-time indication on the vitality of Meetups community in various geographies.

- Select the AGGREGATE operator and drag and drop it to the canvas:

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab72_step4.png)

- Link the PROJECTION and AGGREGATE  operators by clicking on the green circle on the right hand side of the PROJECTION operator and bringing the arrow to the grey circle on the AGGREGATE operator:

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab72_step4b.png)

- Double-click on the AGGREGATE operator:

	- KEYS: group_country, group_city
	- WINDOW INTERVAL TYPE: Time
	- WINDOW INTERVAL: 5 Minutes
	- SLIDING INTERVAL: 5 Minutes
	- TIMESTAMP FIELD: processingTime
	- AGGREGATE EXPRESSION: COUNT(event_url)
	- FIELDS NAME: rsvp_count
	
![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab72_step4b.png)
		
Click OK.

5. Write the aggregates on HDFS.

- Select the HDFS Operator from the **SINK Operators** and drag and drop it to the canvas:

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab72_step5.png)

- Link the AGGREGATE and HDFS  operators by clicking on the green circle on the right hand side of the AGGREGATE operator and bringing the arrow to the grey circle on the HDFS operator:

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab72_step5b.png)

- Double-click on the HDFS Operator:

	- CLUSTER: Your Cluster name
	- HDFS URL: Default value associated with your cluster HDFS URI (Filled automatically)
	- PATH: /tmp/rsvp-agg
	- FLUSH COUNT: 10
	- ROTATION POLICY: Time Based Rotation
	- ROTATION INTERVAL Multiplier: 5
	- ROTATION INTERVAL UNIT: MINUTES
	- OUTPUT FIELDS: Select All

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab72_step5c.png)

6. We also want to persist on HDFS all the data received from Kafka.

- Select the HDFS Operator from the **SINK Operators** and drag and drop it to the canvas:

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab72_step5.png)

- Link the PROJECTION and HDFS  operators by clicking on the green circle on the right hand side of the PROJECTION operator and bringing the arrow to the grey circle on the HDFS operator:

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab72_step6.png)

- Double-click on the HDFS Operator:

	- CLUSTER: Your Cluster name
	- HDFS URL: Default value associated with your cluster HDFS URI (Filled automatically)
	- PATH: /tmp/meetup-rsvps
	- FLUSH COUNT: 100
	- ROTATION POLICY: Time Based Rotation
	- ROTATION INTERVAL Multiplier: 3
	- ROTATION INTERVAL UNIT: MINUTES
	- OUTPUT FIELDS: Select All

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab72_step6b.png)

7. Your application is now ready. We are now ready to deploy. The SAM Application flow should look like:

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab72_step7.png)

- Click on Configure on the top right hand side of the screen:

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab72_step7b.png)

- Fill the values as below:

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab72_step7c.png)

- Run the SAM Application. On the bottom right of the canvas, click on Run icon:

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab72_step7d.png)

- Click OK on the window asking you to confirm the configuration. Give a few minutes for the application to deploy. You should get a notification that the application has been deployed successfully. The icon will now change to the following state. Do NOT click on Kill:

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab72_step7e.png)

- On the top left of your web browser window, click on ‘My Applications’ to get back to the main Application screen. You will be asked if you want to navigate away from the page. Just click OK:

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab72_step7f.png)


8. On the main window, click on the 3 dots on the top right of the tile corresponding to your application, and click on 'Refresh' from time to time after waiting for a couple of minutes. You should see some tuples emitted and transferred in your application.

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab72_step8.png)

9. Go back to Ambari and Click on File View on the drop down from the Views Menu icon

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab72_step9.png)

10. Navigate to /tmp/meetups-rsvp and /tmp/rsvp-agg and preview the files. Note that you will need to wait at least 5 minutes of processing before seeing any file in /tmp/rsvp-agg, as these files are generated every 5 minutes.

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab72_step9b.png)

Let this application run at least for 30-45 minutes to populate the Druid cube data source used for Lab 8.

------------------

# Lab 9

Superset is a Business Intelligence tool packaged with many features for designing, maintaining and enabling the storytelling of data through meaningful data visualizations for real-time data.

In this section, we will explore data visualization with Superset for the Meetup dataset, that has been ingested into Druid, through the SAM application created in Lab 7. You should let your SAM application run at least for 30 minutes before starting the Lab 8. Otherwise, the Druid data source will not appear.

1. Go to Ambari, and click on Druid. Make sure all Druid components are up.  Click on the Druid Coordinator console hyperlink from the Quick Links section:

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab8_step1.png)

2. On the Druid console, you should see the meetup-dsn appear. If not, make sure it shows that indexing tasks are running, and that you have let the SAM application run for around 30 minutes.

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab8_step2.png)

3. Once you see the data source name, go back to Ambari and click on the Superset service. Click on Superset hyperlink from the Quick Links section:

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab8_step3.png)

On the Superset window, login as admin/StrongPassword

4. On Superset top menu, click on Sources and select 'Scan New Datasources' from the drop down:

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab8_step4.png)

You should see the meetup-dsn source appearing on the list of Druid data sources:

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab8_step4b.png)

5. Click on the actual data source name meetup-dsn. This should bring you to a new window to build a visualization. Click on the Visualization Type:

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab8_step5.png)

- Choose Sunburst:

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab8_step5b.png)

- In the new window, for Time Granularity choose '5 minutes' and for Hierarchy, choose these fields in the same order: group_country, group_city, group_name:

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab8_step5c.png)

- Click on Run Query at the top right. You will see a sunburst diagram appear. Click on its title to rename it and call it "Meetup RSVP per Geo".

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab8_step5d.png)

- Click on 'Save' right next to 'Run Query' and save the visualization as follows, then click 'Save and Go to Dashboard':

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab8_step5e.png)

- You will now go to the dashboard. Click on Actions on the right hand-side, and set Auto-refresh to every 5 minutes. From the same menu, click on 'Save the dashboard':

![Image](https://raw.githubusercontent.com/dhananjaymehta/HDF-Workshop/master/img/Lab8_step5f.png)

You should see this dashboard refresh every 5 minutes.\

At this point, you can explore adding more visualizations, such as a timeline or a sankey. 

------------------
