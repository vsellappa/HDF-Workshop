# Contents
- [Lab 1](#lab-1)
  - Deploy and Install HDF
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

You should have a virtual machine allocated for a Linux Centos 7 VM to deploy HDF. Credentials to the VM will be provided by the instructor. Login to the VM as root as a starting point for this lab.

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

### Setup MySQL Databases for HDF

- Enable and start MySQL service:
```
sudo systemctl enable mysqld.service
sudo systemctl start mysqld.service
```
- Create the following mysql-setup.sql script:
```
ALTER USER 'root'@'localhost' IDENTIFIED BY 'Secur1ty!'; 
uninstall plugin validate_password;
CREATE DATABASE registry DEFAULT CHARACTER SET utf8; CREATE DATABASE streamline DEFAULT CHARACTER SET utf8; 
CREATE USER 'registry'@'%' IDENTIFIED BY '${db_password}'; CREATE USER 'streamline'@'%' IDENTIFIED BY '${db_password}'; 
GRANT ALL PRIVILEGES ON registry.* TO 'registry'@'%' WITH GRANT OPTION ; GRANT ALL PRIVILEGES ON streamline.* TO 'streamline'@'%' WITH GRANT OPTION ; 
commit;
```
- Identify the password created by default and setup a new password. You can choose a password of your own and set it up in the following script. By default, it is StrongPassword:
```
#extract system generated Mysql password
oldpass=$( grep 'temporary.*root@localhost' /var/log/mysqld.log | tail -n 1 | sed 's/.*root@localhost: //' )
echo $oldpass
export db_password=${db_password:-StrongPassword}
```
- Run the script mysql-setup created previously:
```
mysql -h localhost -u root -p"$oldpass" --connect-expired-password < mysql-setup.sql
```
- Change root user password for mysql:
```
mysqladmin -u root -p'Secur1ty!' password ${db_password}
```
- Test if the password changes were taken into effect:
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
Start the Ambari agent:
```
chkconfig ambari-agent on
ambari-agent start
```

3. Install Ambari server:
```
yum install -y ambari-server
ambari-server setup -j /usr/java/default -s
```
4. Start Ambari
```
ambari-server start
```
Make sure Ambari starts successfully.

5. Ambari Server post-install steps
Setup MySQL JDBC Driver with Ambari:
```
ambari-server setup --jdbc-db=mysql --jdbc-driver=/usr/share/java/mysql-connector-java.jar
```
Install HDF MPack:
```
export mpack_url="http://public-repo-1.hortonworks.com/HDF/centos7/3.x/updates/3.2.0.0/tars/hdf_ambari_mp/hdf-ambari-mpack-3.2.0.0-520.tar.gz"  
```
Restart Ambari
```
ambari-server restart
```

## Deploy HDP and HDF

In this section, please proceed with an HDP + HDF installation using the Ambari wizard. Login to Ambari web UI by opening http://{YOUR_IP}:8080 and log in with **admin/admin**



## Use your Cluster

### To connect using Putty from Windows laptop

NOTE: The following instructions are for using Putty. You can also use other popular SSH tools such as [MobaXterm](https://mobaxterm.mobatek.net/) or [SmarTTY](http://smartty.sysprogs.com/)

- Right click to download [this ppk key](https://raw.githubusercontent.com/apsaltis/HDF-Workshop/master/hdf-workshop.ppk) > Save link as > save to Downloads folder
- Use putty to connect to your node using the ppk key:
  - Connection > SSH > Auth > Private key for authentication > Browse... > Select hdf-workshop.ppk
![Image](https://raw.githubusercontent.com/apsaltis/HDF-Workshop/master/putty.png)

- Create a new seession called `hdf-workshop`
   - For the Host Name use: centos@IP_ADDRESS_OF_EC2_NODE
   - Click "Save" on the session page before logging in

![Image](https://github.com/apsaltis/HDF-Workshop/raw/master/putty-session.png)


### To connect from Linux/MacOSX laptop

- SSH into your EC2 node using below steps:
- Right click to download [this pem key](https://raw.githubusercontent.com/apsaltis/HDF-Workshop/master/hdf-workshop.pem)  > Save link as > save to Downloads folder
- Copy pem key to ~/.ssh dir and correct permissions
    ```
    cp ~/Downloads/hdf-workshop.pem ~/.ssh/
    chmod 400 ~/.ssh/hdf-workshop.pem
    ```
 - Login to the ec2 node of the you have been assigned by replacing IP_ADDRESS_OF_EC2_NODE below with EC2 node IP Address (your instructor will provide this)
    ```
     ssh -i  ~/.ssh/hdf-workshop.pem centos@IP_ADDRESS_OF_EC2_NODE

    ```

  - To change user to root you can:
    ```
    sudo su -
    ```


#### Login to Ambari

- Login to Ambari web UI by opening http://{YOUR_IP}:8080 and log in with **admin/admin**

- You will see a list of Hadoop components running on your node on the left side of the page
  - They should all show green (ie started) status. If not, start them by Ambari via 'Service Actions' menu for that service

#### NiFi Install

- NiFi is installed at: /usr/hdf/current/nifi



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


  - Step 1: Add a ConnectWebSocket processor to the cavas
      - In case you are using a downloaded template, the ControllerService will be prepopulated. You will need to enable the ControllerService. Double-click the processor and follow the arrow next to the JettyWebSocketClient
      - Set WebSocket Client ID to your favorite number.
  - Step 2: Add an UpdateAttribute procesor
    - Configure it to have a custom property called ``` mime.type ``` with the value of ``` application/json ```
  - Step 3. Add an EvaluateJsonPath processor and configure it as shown below:
  ![Image](https://github.com/apsaltis/HDF-Workshop/raw/master/jsonpath.png)

    The properties to add are:
    ```
    event.name    $.event.event_name

    event.url     $.event.event_url

    group.city    $.group.group_city

    group.state   $.group.group_state

    group.country	$.group.group_country

    group.name		$.group.group_name

    venue.lat		  $.venue.lat

    venue.lon     $.venue.lon

    venue.name		$.venue.venue_name
    ```
  - Step 4: Add a SplitJson processor and configure the JsonPath Expression to be ```$.group.group_topics ```
  - Step 5: Add a ReplaceText processor and configure the Search Value to be ```([{])([\S\s]+)([}])``` and the Replacement Value to be
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
  - Step 6: Add a PutFile processor to the canvas and configure it to write the data out to ```/tmp/rsvp-data```

##### Questions to Answer
1. What does a full RSVP Json object look like?
2. How many output files do you end up with?
3. How can you change the file name that Json is saved as from PutFile?
3. Why do you think we are splitting out the RSVP's by group?
4. Why are we using the Update Attribute processor to add a mime.type?
4. How can you cange the flow to get the member photo from the Json and download it.


------------------

# Lab 3

  ![Image](https://github.com/apsaltis/HDF-Workshop/raw/master/lab3.png)
  A template for this flow can be found [here](https://raw.githubusercontent.com/apsaltis/HDF-Workshop/master/templates/MiNiFi_Flow.xml)


## Getting started with MiNiFi ##

In this lab, we will learn how to configure MiNiFi to send data to NiFi:

* Setting up the Flow for NiFi
* Setting up the Flow for MiNiFi
* Preparing the flow for MiNiFi
* Configuring and starting MiNiFi
* Enjoying the data flow!


## Setting up the Flow for NiFi
**NOTE:** Before starting NiFi we need to enable Site-to-Site communication. To do that we will use Ambari to update the required configuration. In Ambari the below property values can be found at ````http://<EC2_NODE>:8080/#/main/services/NIFI/configs```` .

* Change:
  ````
			nifi.remote.input.socket.port=

  ````
  To
  ```
   		nifi.remote.input.socket.port=10000

  ```
* Restart NiFi via Ambari


Now we should be ready to create our flow. To do this do the following:

1.	The first thing we are going to do is setup an Input Port. This is the port that MiNiFi will be sending data to. To do this drag the Input Port icon to the canvas and call it "From MiNiFi".

2. Now that the Input Port is configured we need to have somewhere for the data to go once we receive it. In this case we will keep it very simple and just log the attributes. To do this drag the Processor icon to the canvas and choose the LogAttribute processor.

3.	Now that we have the input port and the processor to handle our data, we need to connect them.

4.  We are now ready to build the MiNiFi side of the flow. To do this do the following:
	* Add a GenerateFlowFile processor to the canvas (don't forget to configure the properties on it)
	* Add a Remote Processor Group to the canvas

           For the URL copy and paste the URL for the NiFi UI from your browser
   * Connect the GenerateFlowFile to the Remote Process Group

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
