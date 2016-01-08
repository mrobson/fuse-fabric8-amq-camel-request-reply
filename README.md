Fabric8 ActiveMQ Camel Request and Reply
========================================
Author: Matt Robson

Technologies: Fuse, Fabric8, ActiveMQ, Camel (Request/Reply), fabric8-maven-plugin, Profiles

Product: Fuse 6.2.1, ActiveMQ 6.2.1

Breakdown                                                                                                                     
---------                                                                                                                     
This is a code and profile based example to demonstrate how to implement and deploy a basic Camel Request Reply route in Fabric.

For more information see:

* <https://access.redhat.com/site/documentation/JBoss_Fuse/> for more information about Red Hat JBoss Fuse
* <http://www.jboss.org/products/fuse/overview/> for more information about the upstream community
* <http://fabric8.io/> for more information about fabric8

System Requirements
-------------------
Before building out your Fabric, you will need:
* Java 1.7 or 1.8
* JBoss Fuse 6.2.1
* JBoss ActiveMQ 6.2.1

Prerequisites
-------------
* A working Fabric and AMQ Broker Network as outlined in part 1, 'fuse-fabric8-getting-started' <https://github.com/mrobson/fuse-fabric8-getting-started> and part 2, 'fuse-fabric8-ssh-containers' <https://github.com/mrobson/fuse-fabric8-ssh-containers>

Build and Deploy
----------------

1) clone the project

        git clone https://github.com/mrobson/fuse-fabric8-amq-camel-request-reply.git

2) change to project directory

        cd fuse-fabric8-amq-camel-request-reply/

3) build the project

	mvn clean install

4) deploy the project to the fabric8 maven repo
	
	mvn bundle:deploy

4) create and deploy the profile

	mvn fabric8:deploy -Dfabric8.jolokiaUrl=http://fusefabric1.lab.com:8181/jolokia

This will deploy the configured profile to your fabric8 install.  The feature and bundle is read from the configured localRepository

The profile, 'example-camel-requestreply', deploys a basic camel route that uses a timer to generate an event every second and inserts it onto an incoming queue, 'com.redhat.incoming1'.  A second route reads from 'com.redhat.incoming1' and sends the message to a service queue, 'com.redhat.service1' where request/reply semantics are applied.  A reply queue, 'com.redhat.service1.fuse-services1.reply' is created to automatically handle the response. A third route reads from 'com.redhat.service1', processes the message and the reply is automatically put back onto the reply queue. 

The profiles are built and deployed with the necessary dependancies using the fabric8 maven plugin.

	JBossFuse:karaf@root> profile-display example-camel-requestreply 
	Profile id: example-camel-requestreply
	Version   : 1.0
	Attributes: 
		abstract: false
		parents: feature-camel
	Containers: 

	Container settings
	----------------------------
	Repositories : 
		mvn:org.mrobson.example.fabric8-amq/features/1.0-SNAPSHOT/xml/features

	Features : 
		fuse-fabric8-amq-camel-rr

	Agent Properties : 
		  lastRefresh.example-camel-requestreply = 1452281908404


	Configuration details
	----------------------------
	PID: org.mrobson.example.amq.camel1
	  name Matt
	  hostname ${karaf.name}
	  question What is your name?

5) Now that the profile has been deployed into the fabric, you can assign them to the services container.

	JBossFuse:admin@root> container-add-profile fuse-services1 example-camel-requestreply

Once the profile loads, you will see it start to produce a message every second along with the reply.

	2016-01-08 14:45:16,064 | INFO  | /productionTimer | produce-incoming1                | 259 - org.apache.camel.camel-core - 2.15.1.redhat-621084 | What is your name? Asked at: Fri Jan 08 14:45:16 EST 2016
	2016-01-08 14:45:16,071 | INFO  | redhat.service1] | consume-process-reply1           | 259 - org.apache.camel.camel-core - 2.15.1.redhat-621084 | My name is Matt!

6) Verify everyone was successfully provisioned

	JBossFuse:karaf@root> container-list 
	[id]            [version]  [type]  [connected]  [profiles]                                                  [provision status]
	fuse-services1  1.0        karaf   yes          default, mq-client-default, example-camel-requestreply      success           
	root*           1.0        karaf   yes          fabric, fabric-ensemble-0000-1, jboss-fuse-full             success           

7) You can check the ActiveMQ statistics to see the brokers in action.

	Fabric8:admin@amq-broker1> activemq:dstat 
	Name                                                Queue Size  Producer #  Consumer #   Enqueue #   Dequeue #    Memory %
	ActiveMQ.Advisory.Connection                                 0           0           0          11           0           0
	ActiveMQ.Advisory.Consumer.Queue.redhat.queue                0           0           1           5           0           0
	ActiveMQ.Advisory.MasterBroker                               0           0           0           1           0           0
	ActiveMQ.Advisory.NetworkBridge                              0           0           0           3           0           0
	ActiveMQ.Advisory.Queue                                      0           0           0           1           0           0
	redhat.queue                                                 0           0           1        4728        4728           0

Done!

Notes: For the fabric8-maven-plugin plus localRepository approach to work correctly, there is a bug fix in 6.2.1 P1 required.

	profile-edit --pid io.fabric8.maven/io.fabric8.maven.localRepository=/mnt/maven default

	Set '-Daether.updateCheckManager.sessionState=bypass' for startup

	profile-edit --pid io.fabric8.agent/org.ops4j.pax.url.mvn.repositories='file:${runtime.home}/${karaf.default.repository}@snapshots@id=karaf-default,file:${runtime.data}/maven/upload@snapshots@id=fabric-upload-remote@releasesUpdate=always@snapshotsUpdate=always,file:/mnt/maven@snapshots@id=nfs-maven@releasesUpdate=always@snapshotsUpdate=always' default

	Bug for for fabric-maven-proxy @ ENTESB-4012
