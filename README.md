# Contents
- [Lab 1](#lab-1)
  - Install HDP & HDF
  - Bring up the Environment
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

<!-- - [Lab 7](#lab-7) - Tying it all together with SAM
  - Creating the Streaming Application
  - Watching the dashboard -->


---------------

# Lab 1

This lab is to deploy HDF.

You should have a virtual machine allocated for a Linux Centos 7 VM to deploy HDF. Credentials to the VM will be provided by the instructor. Open an ssh session on your allocated VM and switch to root as a starting point for this lab (using ```sudo sh``` or ```sudo -i```).

For complete instructions, you can follow the [Official HDF 3.2 Documentation](https://docs.hortonworks.com/HDPDocuments/HDF3/HDF-3.2.0/installing-hdf/content/install-ambari.html) to deploy HDF 3.2. In the following instructions, we are applying these steps to deploy and install an HDF 3.2 environment.

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
export mpack_url="http://public-repo-1.hortonworks.com/HDF/centos7/3.x/updates/3.2.0.0/tars/hdf_ambari_mp/hdf-ambari-mpack-3.2.0.0-520.tar.gz"  
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

# Lab 2

In this lab, we will learn how to:
  - Consume the Meetup RSVP stream
  - Extract the JSON elements we are interested in
  - Split the JSON into smaller fragments
  - Write the JSON to the file system


### Consuming RSVP Data

To get started we need to consume the data from the Meetup RSVP stream, extract what we need, splt the content and save it to a file:

#### Goals:
   - Consume Meetup RSVP stream
   - Extract the JSON elements we are interested in
   - Split the JSON into smaller fragments
   - Write the JSON files to disk

 Our final flow for this lab will look like the following:

  ![Image](https://github.com/apsaltis/HDF-Workshop/raw/master/lab1.png)
  
  A template for this flow can be found [here](https://raw.githubusercontent.com/apsaltis/HDF-Workshop/master/templates/HDF-Workshop_Lab1-Flow.xml)

1. With a blank canvas, click on the Configuration gear icon in the Operate box on the left side of the UI:

![Image](https://github.com/zoharsan/HDF-Workshop/blob/master/Lab2_step1.png)

2. Under the CONTROLLER SERVICES tab, Add a JettyWebSocketClient service and click on the gear icon to edit the configure the controller service.

![Image](https://github.com/zoharsan/HDF-Workshop/blob/master/Lab2_step2.png)

3. Under the PROPERTIES tab add the value for URI for the last property WebSocket URI, and paste it for the empty value for WebSocket URI in bold. The value pasted is ```ws://stream.meetup.com/2/rsvps```Lab2_step3. Your configuration should look like this:

![Image](https://github.com/zoharsan/HDF-Workshop/blob/master/Lab2_step3.png)

4. Notice that the state for the Controller Service is Disabled. Click on the lightning icon on the right to enable it:

![Image](https://github.com/zoharsan/HDF-Workshop/blob/master/Lab2_step4.png)

5. Add a ConnectWebSocket processor to the canvas by dragging the icon on the page:

![Image](https://github.com/zoharsan/HDF-Workshop/blob/master/Lab2_step5.png)
  
![Image](https://github.com/zoharsan/HDF-Workshop/blob/master/Lab2_step5b.png)

6. Configure the ConnectWebSocket Processor so it looks like below:
  1. Under the properties tab set the WebSocket Client Controller Service
  2. Set the WebSocket Client ID to AGP-HDF-WS-TEST
  ![Image](https://github.com/zoharsan/HDF-Workshop/blob/master/Lab2_step6.png)
  3. Set the automatic termination
  ![Image](https://github.com/zoharsan/HDF-Workshop/blob/master/Lab2_step6b.png)

7. Add an UpdateAttribute Processor to the canvas using the same method as previously:

![Image](https://github.com/zoharsan/HDF-Workshop/blob/master/Lab2_step7.png)
  
  - Configure it to have a custom property called mime.type with the value of application/json:
  
  ![Image](https://github.com/zoharsan/HDF-Workshop/blob/master/Lab2_step7b.png)

8. Join ConnectWebSocket Processor and the UpdateAttribute Processor using a text message for relationships.

![Image](https://github.com/zoharsan/HDF-Workshop/blob/master/Lab2_step8.png)
![Image](https://github.com/zoharsan/HDF-Workshop/blob/master/Lab2_step8b.png)

9. Add an EvaluateJsonPath processor and configure it as shown below:

![Image](https://github.com/apsaltis/HDF-Workshop/raw/master/jsonpath.png)

    The properties to add are:
    ```
    event.name    	$.event.event_name
    event.url     	$.event.event_url
    group.city    	$.group.group_city
    group.state   	$.group.group_state
    group.country	$.group.group_country
    group.name		$.group.group_name
    venue.lat		$.venue.lat
    venue.lon     	$.venue.lon
    venue.name		$.venue.venue_name
    ```

10. Join the UpdateAttribute processor and EvaluateJsonPath processor.  Also add a failure relationship (Note: recursive join)

![Image](https://github.com/zoharsan/HDF-Workshop/blob/master/Lab2_step10.png)
![Image](https://github.com/zoharsan/HDF-Workshop/blob/master/Lab2_step10b.png)

11. Add a SplitJson processor and configure the JsonPath Expression to be ```$.group.group_topics ```. Also the Original  relationship needs to be automatically terminated.  Your configuration should look like below:

![Image](https://github.com/zoharsan/HDF-Workshop/blob/master/Lab2_step11.png)
![Image](https://github.com/zoharsan/HDF-Workshop/blob/master/Lab2_step11b.png)

12. Join the EvaluateJsonPath processor and the SplitJson processor.  In addition, create a failure recursive join on the SplitJsaon Processor. Should look like the below.

![Image](https://github.com/zoharsan/HDF-Workshop/blob/master/Lab2_step12.png)
![Image](https://github.com/zoharsan/HDF-Workshop/blob/master/Lab2_step12b.png)
![Image](https://github.com/zoharsan/HDF-Workshop/blob/master/Lab2_step12c.png)

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
        "group" : {
          "group_city" : "${group.city}",
          "group_country" : "${group.country}",
          "group_name" : "${group.name}",
          "group_state" : "${group.state}",
          $2
         }
      }
      ```
  The processor should look like the below
  
  ![Image](https://github.com/zoharsan/HDF-Workshop/blob/master/Lab2_step13.png)

14. Join the SplitJson processor and the ReplaceText processor. In addition add an on Failure recursive join.

![Image](https://github.com/zoharsan/HDF-Workshop/blob/master/Lab2_step14.png)
![Image](https://github.com/zoharsan/HDF-Workshop/blob/master/Lab2_step14b.png)
![Image](https://github.com/zoharsan/HDF-Workshop/blob/master/Lab2_step14c.png)

15. Add a PutFile processor to the canvas and configure it to write the data out to ```/tmp/rsvp-data```. Automatically terminate both on Success and Failure. The configuration should look like below.

![Image](https://github.com/zoharsan/HDF-Workshop/blob/master/Lab2_step15.png)
![Image](https://github.com/zoharsan/HDF-Workshop/blob/master/Lab2_step15b.png)

16. Join the ReplaceText processor and the PutFile processor for successful relationships.

![Image](https://github.com/zoharsan/HDF-Workshop/blob/master/Lab2_step16.png)
![Image](https://github.com/zoharsan/HDF-Workshop/blob/master/Lab2_step16b.png)

17. Start the flow by clicking on the ```Play``` triangle icon in the Operate window:

  ![Image](https://github.com/zoharsan/HDF-Workshop/blob/master/Lab2_step17.png)

You should see tuples flowing after a couple of minutes.

##### Questions to Answer
1. What does a full RSVP Json object look like?
2. How many output files do you end up with?
3. How can you change the file name that Json is saved as from PutFile?
3. Why do you think we are splitting out the RSVP's by group?
4. Why are we using the Update Attribute processor to add a mime.type?
4. How can you cange the flow to get the member photo from the Json and download it.

18. After completing this lab, create a new Process Group by dragging onto the canvas the Process Group icon and call it Lab2:

![Image](https://github.com/zoharsan/HDF-Workshop/blob/master/Lab2_step18.png)

19. Select all the components of your flow including processors, and links between processors by pressing shift and selecting an area of the canvas which contains all the flow. You will see a rectangle being drawn in the canvas corresponding to the area selected. You may need to zoom out. 

![Image](https://github.com/zoharsan/HDF-Workshop/blob/master/Lab2_step19.png)

20. Double-click on the Lab2 process group. When a new canvass opens for the Lab2 process group, right-click and select ``` Paste ```. You should now have the entire flow pasted into this process group.

21. Click on the main NiFi flow to go back on the main canvas. Select the flow as per step 19, right-click and select ``` Delete ```.

![Image](https://github.com/zoharsan/HDF-Workshop/blob/master/Lab2_step21.png)
![Image](https://github.com/zoharsan/HDF-Workshop/blob/master/Lab2_step21b.png)

You should now have an empty canvas to start on Lab 3.

------------------

# Lab 3

  ![Image](https://github.com/apsaltis/HDF-Workshop/raw/master/lab3.png)
  A template for this flow can be found [here](https://raw.githubusercontent.com/apsaltis/HDF-Workshop/master/templates/MiNiFi_Flow.xml)

## Download and Install MiniFi

Run all the below commands as root:
```
wget http://apache.claz.org/nifi/minifi/0.5.0/minifi-0.5.0-bin.tar.gz
wget http://apache.claz.org/nifi/minifi/0.5.0/minifi-toolkit-0.5.0-bin.tar.gz
cp minifi-0.5.0-bin.tar.gz /usr/hdf
cd /usr/hdf
tar zxvf minifi-0.5.0-bin.tar.gz
tar xzvf minifi-toolkit-0.5.0-bin.tar.gz
```

## Getting started with MiNiFi ##

In this lab, we will learn how to configure MiNiFi to send data to NiFi:

* Setting up the Flow for NiFi
* Setting up the Flow for MiNiFi
* Preparing the flow for MiNiFi
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
  
  ![Image](https://github.com/apsaltis/HDF-Workshop/raw/master/Lab3_step1.png)

2. Restart NiFi via Ambari

3. Access the NiFi UI from Ambari or ```http://<public_IP_addr>:9090/nifi/```

4. Now we should be ready to create our flow. The first thing we are going to do is setup an Input Port. This is the port that MiNiFi will be sending data to. To do this drag the Input Port icon to the canvas and call it "From MiNiFi": ![Image](https://github.com/apsaltis/HDF-Workshop/raw/master/Lab3_step4.png)

![Image](https://github.com/apsaltis/HDF-Workshop/raw/master/Lab3_step4b.png)

5. Now that the Input Port is configured we need to have somewhere for the data to go once we receive it. In this case we will keep it very simple and just log the attributes. To do this drag the Processor icon to the canvas and choose the LogAttribute processor.

![Image](https://github.com/apsaltis/HDF-Workshop/raw/master/Lab3_step5.png)

6. On the Settings tab, under Auto-terminate relationships, select the checkbox next to Success. This will terminate FlowFiles after this processor has successfully processed them.

![Image](https://github.com/apsaltis/HDF-Workshop/raw/master/Lab3_step5b.png)

7. Also on the Settings tab, set the Bulletin level to Info. This way, when the dataflow is running, this processor will display the bulletin icon, and the user may hover over it with the mouse to see the attributes that the processor is logging.

![Image](https://github.com/apsaltis/HDF-Workshop/raw/master/Lab3_step7.png)

8. Now that we have the input port and the processor to handle our data, we need to connect them. After creating the connection your data flow should look like this:

![Image](https://github.com/apsaltis/HDF-Workshop/raw/master/Lab3_step8.png)

9. We are now ready to build the MiNiFi side of the flow. To do this do the following:
  - Add a GenerateFlowFile processor to the canvas. On the Scheduling tab, set Run schedule to: 5 sec. Note that the GenerateFlowFile processor can create many FlowFiles very quickly; that's why setting the Run schedule is important so that this flow does not overwhelm the system NiFi is running on.
  
  ![Image](https://github.com/apsaltis/HDF-Workshop/raw/master/Lab3_step9.png)
  
  - On the Properties tab, set File Size to: 10 kb	
  
  ![Image](https://github.com/apsaltis/HDF-Workshop/raw/master/Lab3_step9b.png)
  
  - Add a Remote Processor Group to the canvas. For the URL copy and paste the URL for the NiFi UI from your browser:
  
  ![Image](https://github.com/apsaltis/HDF-Workshop/raw/master/Lab3_step9c.png)
  
  - Connect the GenerateFlowFile to the Remote Process Group
  
  ![Image](https://github.com/apsaltis/HDF-Workshop/raw/master/Lab3_step9d.png)
    

5. The next step is to generate the flow we need for MiNiFi. To do this do the following steps:

   * Create a template for MiNiFi
   * Select the GenerateFlowFile, the NiFi Flow Remote Processor Group, and the Connection between them (these are the only things needed for MiNiFi)
   * Select the "Create Template" button from the toolbar
   * Choose a name for your template


7. Now we need to download the template
8. Now SCP the template you downloaded to the ````/tmp```` directory on your EC2 instance. If you are using Windows you will need to download WinSCP (https://winscp.net/eng/download.php)
9.  We are now ready to setup MiNiFi. However before doing that we need to convert the template to YAML format which MiNiFi uses. To do this we need to do the following:

    * Navigate to the minifi-toolkit directory (/usr/hdf/current/minifi-toolkit-0.4.0)
    * Transform the template that we downloaded using the following command:

      ````sudo bin/config.sh transform <INPUT_TEMPLATE> <OUTPUT_FILE>````

      For example:

      ````sudo bin/config.sh transform /temp/MiNiFi_Flow.xml config.yml````

10. Next copy the ````config.yml```` to the ````minifi-0.4.0/conf```` directory. That is the file that MiNiFi uses to generate the nifi.properties file and the flow.xml.gz for MiNiFi.

11. That is it, we are now ready to start MiNiFi. To start MiNiFi from a command prompt execute the following:

  ```
  cd /usr/hdf/current/minifi-0.4.0
  sudo bin/minifi.sh start
  tail -f logs/minifi-app.log
  ```

You should be able to now go to your NiFi flow and see data coming in from MiNiFi.

You may tail the log of the MiNiFi application by
   ```
   tail -f /usr/hdf/current/minifi/minifi-0.2.0/logs/minifi-app.log
   ```
If you see error logs such as "the remote instance indicates that the port is not in a valid state",
it is because the Input Port has not been started.
Start the port and you will see messages being accumulated in its downstream queue.

------------------

# Lab 4

## Kafka Basics
In this lab we are going to explore creating, writing to and consuming Kafka topics. This will come in handy when we later integrate Kafka with NiFi and Streaming Analytics Manager.

1. Creating a topic
  - Step 1: Open an SSH connection to your EC2 Node.
  - Step 2: Naviagte to the Kafka directory (````/usr/hdf/current/kafka-broker````), this is where Kafka is installed, we will use the utilities located in the bin directory.

    ````
    #cd /usr/hdf/current/kafka-broker/
    ````

  - Step 3: Create a topic using the kafka-topics.sh script
    ````
    bin/kafka-topics.sh --zookeeper localhost:2181 --create --partitions 1 --replication-factor 1 --topic first-topic

    ````

    NOTE: Based on how Kafka reports metrics topics with a period ('.') or underscore ('_') may collide with metric names and should be avoided. If they cannot be avoided, then you should only use one of them.

  - Step 4:	Ensure the topic was created
    ````
    bin/kafka-topics.sh --list --zookeeper localhost:2181
    ````

2. Testing Producers and Consumers
  - Step 1: Open a second terminal to your EC2 node and navigate to the Kafka directory
  - In one shell window connect a consumer:
  ````
 bin/kafka-console-consumer.sh --zookeeper localhost:2181 --from-beginning --topic first-topic
````

    Note: using –from-beginning will tell the broker we want to consume from the first message in the topic. Otherwise it will be from the latest offset.

  - In the second shell window connect a producer:
````
bin/kafka-console-producer.sh --broker-list demo.hortonworks.com:6667 --topic first-topic
````


- Sending messages. Now that the producer is  connected  we can type messages.
  - Type a message in the producer window

- Messages should appear in the consumer window.

- Close the consumer (ctrl-c) and reconnect using the default offset, of latest. You will now see only new messages typed in the producer window.

- As you type messages in the producer window they should appear in the consumer window.

------------------

# Lab 5

## Integrating Kafka with NiFi
1. Creating the topic
  - Step 1: Open an SSH connection to your EC2 Node.
  - Step 2: Naviagte to the Kafka directory (````/usr/hdf/current/kafka-broker````), this is where Kafka is installed, we will use the utilities located in the bin directory.

    ````
    #cd /usr/hdf/current/kafka-broker/
    ````

  - Step 3: Create a topic using the kafka-topics.sh script
    ````
    bin/kafka-topics.sh --zookeeper localhost:2181 --create --partitions 1 --replication-factor 1 --topic meetup_rsvp_raw

    ````

    NOTE: Based on how Kafka reports metrics topics with a period ('.') or underscore ('_') may collide with metric names and should be avoided. If they cannot be avoided, then you should only use one of them.

  - Step 4:	Ensure the topic was created
    ````
    bin/kafka-topics.sh --list --zookeeper localhost:2181
    ````

2. Integrating NiFi
  - Step 1: Add a PublishKafka_1_0 processor to the canvas.
  - Step 2: Add a routing for the success relationship of the ReplaceText processor to the PublishKafka_1_0 processor added in Step 1 as shown below:

    ![Image](https://github.com/apsaltis/HDF-Workshop/raw/master/publishkafka.png)
  - Step 3: Configure the topic and broker for the PublishKafka_1_0 processor,
  where topic is meetup_rsvp_raw and broker is demo.hortonworks.com:6667.


3. Start the NiFi flow
4. In a terminal window to your EC2 node and navigate to the Kafka directory and connect a consumer to the ````meetup_rsvp_raw```` topic:

    ````
    bin/kafka-console-consumer.sh --zookeeper localhost:2181 --from-beginning --topic meetup_rsvp_raw
    ````


5. Messages should now appear in the consumer window.


------------------

# Lab 6

## Integrating the Schema Registry
1. Creating the topic
  - Step 1: Open an SSH connection to your EC2 Node.
  - Step 2: Naviagte to the Kafka directory (````/usr/hdf/current/kafka-broker````), this is where Kafka is installed, we will use the utilities located in the bin directory.

    ````
    #cd /usr/hdf/current/kafka-broker/
    ````

  - Step 3: Create a topic using the kafka-topics.sh script
    ````
    bin/kafka-topics.sh --zookeeper localhost:2181 --create --partitions 1 --replication-factor 1 --topic meetup_rsvp_avro

    ````

    NOTE: Based on how Kafka reports metrics topics with a period ('.') or underscore ('_') may collide with metric names and should be avoided. If they cannot be avoided, then you should only use one of them.

  - Step 4:	Ensure the topic was created
    ````
    bin/kafka-topics.sh --list --zookeeper localhost:2181
    ````

2. Adding the Schema to the Schema Registry
  - Step 1: Open a browser and navigate to the Schema Registry UI. You can get to this from the either the ```Quick Links``` drop down in Ambari, as shown below:

    ![Image](https://github.com/apsaltis/HDF-Workshop/raw/master/registry_quick_link.png)

    or by going to ````http://<EC2_NODE>:17788````
  - Step 2: Create Meetup RSVP Schema in the Schema Registry
    1. Click on “+” button to add new schemas. A window called “Add New Schema” will appear.
    2. Fill in the fields of the ````Add Schema Dialog```` as follows:

        ![Image](https://github.com/apsaltis/HDF-Workshop/raw/master/add_schema_dialog.png)

        For the Schema Text you can download it [here](https://raw.githubusercontent.com/apsaltis/HDF-Workshop/master/meetup_rsvp.asvc) and either copy and paste it or upload the file.

        Once the schema information fields have been filled and schema uploaded, click **Save**.

3. We are now ready to integrate the schema with NiFi
  - Step 0: Remove the PutFile and PublishKafka_1_0 processors from the canvas, we will not need them for this section.
  - Step 1: Add a UpdateAttribute processor to the canvas.
  - Step 2: Add a routing for the success relationship of the ReplaceText processor to the UpdateAttrbute processor added in Step 1.
  - Step 3: Configure the UpdateAttribute processor as shown below:

    ![Image](https://github.com/apsaltis/HDF-Workshop/raw/master/update_attribute_schema_name.png)

  - Step 4: Add a JoltTransformJSON processor to the canvas.
  - Step 5: Add a routing for the success relationship of the UpdateAttribute processor to the JoltTransformJSON processor added in Step 5.
  - Step 6: Configure the JoltTransformJSON processor as shown below:

    ![Image](https://github.com/apsaltis/HDF-Workshop/raw/master/jolt_transform_config.png)

    The JSON used in the 'Jolt Specification' property is as follows:

    ``
    {
      "venue": {
        "lat": ["=toDouble", 0.0],
        "lon": ["=toDouble", 0.0]
      }
    }
  ``
  - Step 7: Add a LogAttribute processor to the canvas.
  - Step 8: Add a routing for the failure relationship of the JoltTransformJSON processor to the LogAttribute processor added in Step 7.
  - Step 9: Add a PublishKafkaRecord_1_0 to the canvas.
  - Step 10: Add a routing for the success relationship of the JoltTransformJSON processor to the PublishKafkaRecord_1_0 processor added in Step 9.
  - Step 11: Configure the PublishKafkaRecord_1_0 processor to look like the following:

    ![Image](https://github.com/apsaltis/HDF-Workshop/raw/master/publishkafka_record_configuration.png)


  - Step 12: When you configure the JsonTreeReader and AvroRecordSetWriter, you will first need to configure a schema registry controller service. The schema registry controller service we are going to use is the 'HWX Schema Registry', it should be configured as shown below:

    ![Image](https://github.com/apsaltis/HDF-Workshop/raw/master/hwx_schema_registry_config.png)

  - Step 13: Configure the JsonTreeReader as shown below:

    ![Image](https://github.com/apsaltis/HDF-Workshop/raw/master/json_tree_reader_config.png)

  - Step 14: Configure the AvroRecordSetWriter as shown below:

      ![Image](https://github.com/apsaltis/HDF-Workshop/raw/master/avro_recordset_writer.png)

    After following the above steps this section of your flow should look like the following:

    ![Image](https://github.com/apsaltis/HDF-Workshop/raw/master/update_jolt_kafka_section.png)


4. Start the NiFi flow
5. In a terminal window to your EC2 node and navigate to the Kafka directory and connect a consumer to the ````meetup_rsvp_avro```` topic:

    ````
    bin/kafka-console-consumer.sh --zookeeper localhost:2181 --from-beginning --topic meetup_rsvp_avro
    ````


5. Messages should now appear in the consumer window.


<!--

------------------


# Lab 7

## Tying it all together
For this lab we are going to break from the Meetup RSVP data and use a fictious IoT Trucking application.

  - Step 1: SSH to your EC2 instance
  - Step 2: We are now going to get a data loader running:

    ````
    cd /root/Data-Loader
    nohup java -cp /root/Data-Loader/stream-simulator-jar-with-dependencies.jar  hortonworks.hdp.refapp.trucking.simulator.SimulationRegistrySerializerRunnerApp 20000 hortonworks.hdp.refapp.trucking.simulator.impl.domain.transport.Truck  hortonworks.hdp.refapp.trucking.simulator.impl.collectors.KafkaEventSerializedWithRegistryCollector 1 /root/Data-Loader/routes/midwest/ 10000 demo.hortonworks.com:6667 http://demo.hortonworks.com:7788/api/v1 ALL_STREAMS NONSECURE &
    ````
  - Step 3: Now that the data is flowing, instantiate the 'IoT Trucking' NiFi template.
  - Step 4: Inspect the flow that is created and ensure there are no errors, if there are go ahead and correct those.
  - Step 5: Start this section of the NiFi flow.
  - Step 6: Go to your Ambari dashboard and navigate to the SAM UI, by selecting the following URL:

    ![Image](https://github.com/apsaltis/HDF-Workshop/raw/master/SAM_URL_Link.png)

  - Step 7: You should now see the SAM UI that looks like the following:

    ![Image](https://github.com/apsaltis/HDF-Workshop/raw/master/sam_default.png)

  - Step 8: To inspect the application, click the icon in the top right hadn corner of the applicaiton and chose 'Edit', as shown below:

    ![Image](https://github.com/apsaltis/HDF-Workshop/raw/master/sam_app_edit.png)

  - Step 9: You shoudl now see a UI that looks like the following:

    ![Image](https://github.com/apsaltis/HDF-Workshop/raw/master/sam_edit.png)

    Spend a moment to explore and dig into any of the components. Notice at the bottom right hand corner is states "Status Active", this indicates that the application is running.

  - Step 10: Verify that Storm the application is running using Storm Mon. To do this go back to your Ambari Dashboard and chose the "Storm Mon link" as shown below:

    ![Image](https://github.com/apsaltis/HDF-Workshop/raw/master/storm_mon_link.png)

    That should bring up a UI that looks like the following:

    ![Image](https://github.com/apsaltis/HDF-Workshop/raw/master/storm_mon_ui.png)

  - Step 11: We are now ready to explore Superset, to do this go back to the Ambari dashboard and from the Drui service chose the "Quick Link" to "Superset" as shown below:

    ![Image](https://github.com/apsaltis/HDF-Workshop/raw/master/superset_link.png)

  - Step 12: Exploring Superset -- following the link in Step 11 should take you to a UI that looks like the following:

    ![Image](https://github.com/apsaltis/HDF-Workshop/raw/master/superset_welcome.png)

  **NOTE: If you are prompted for a password use admin/admin**

    Spend some time exploring the dashboard and the datasources.
-->
