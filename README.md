# Fuse Standalone Playbook

## About
This [Ansible Playbook](http://docs.ansible.com/ansible/playbooks.html) includes
a set of different roles:

* **fuse-standalone**: Deploys a set of Red Hat JBoss Fuse Standalone instances on several hosts.
* **fuse-deploy-bundle**: Deploys a set of Application Bundles on several hosts.
* **fuse-undeploy-bundle**: Undeploys a set of Application Bundles on several hosts.

These roles with the right configuration will allow you to deploy a full complex
Red Hat JBoss Fuse environment and automate the most common tasks.

## Ansible Requirements

This playbook has tested with the next Ansible versions:

		ansible 2.2.1.0

The playbook has to be executed with **root permission**, using the **root user** or
via **sudo** because it will install packages and configure the hosts.

## RHEL Subscription Required
This role is prepared to be executed in a RHEL7 Server. Subscription is required to
execute the **Grant RHEL and Fuse repos enabled** task, which enables the required
repositories to install Fuse dependencies

## Ansible Roles

### Host Variables

Each role will use a set of Host Variables defined in the playbook for each
host defined in the inventory.

These variables are:

* **esb_name**: Logical name to identify each Fuse Instance. Mandatory.
* **port_offset**: Port offset to be added to default Fuse ports. Mandatory.
* **nob**: Sets that this instance should be a member of a Network of Brokers. Values: (true|false). Optional.
* **esb_type**: Identify each Fuse type. This variable will define the final
	features needed in this instance. The current values permited:
	* **full**: Full Fuse instance. Default value.
	* **noamq**: Disable A-MQ features.

These variables should be defined in the playbook as:

		roles:
			# Two Fuse Standalone with a Network of Brokers
			- { role: fuse-standalone, esb_name: 'esb01',  port_offset: '0', nob: 'true' }
			- { role: fuse-standalone, esb_name: 'esb02',  port_offset: '100', nob: 'true' }

### fuse-standalone role

This role deploys several Fuse Standalone instances in a set of hosts.

The main tasks done are:

* Install Fuse prerequisites and set up host
* Install Fuse binaries
* Create OS users and OS services
* Set up different Fuse features: Karaf, Security, A-MQ, Jetty, ...

Red Hat JBoss Fuse binaries should be downloaded from [Red Hat Customer Portal](https://access.redhat.com/jbossnetwork/restricted/listSoftware.html?downloadType=distributions&product=jboss.fuse&version=6.3.0)
(subscription needed). The binaries should be copied into **/tmp** folder from
the host where the playbook will be executed.

#### Configuration parameters
Role's execution could be configured with the following variables.

* **fuse**: Define the Red Hat JBoss Fuse version and patch to install. This values will
	form the path the the binaries: */tmp/jboss-fuse-karaf-{{ fuse['version'] }}.redhat-{{ fuse['patch'] }}.zip*

		# Fuse Version and Patch
		fuse:
			version: '6.3.0'
			patch: '254'

* **user**: OS user to execute the Fuse process.

		# OS User to install/execute Fuse
		user:
			name: 'fuse'
			shell: '/bin/bash'
			homedir: 'True'

* **JAVA_HOME**: Path to Java Virtual Machine to execute the process.

		# Java Home
		JAVA_HOME: /usr/lib/jvm/jre-1.8.0-openjdk

* **FUSE_HOME**: Fuse Home path where it will installed each instance. This
	variable will be defined for each host with a name. This allow installs more
	instances in the same hosts.

		# Fuse Home
		FUSE_HOME: '/opt/fuse/latest-{{ esb_name }}'

* **fuse_users**: Users map to be created in each Fuse instance. These users will
	be stored in **{{ FUSE_HOME }}/etc/users.properties** file.

		# Fuse Administrative Users
		fuse_users:
			admin:
				username: admin
				password: karaf
				roles:
					- admin
					- SuperUser

* **nob_multicast_name**: Multicast cluster name to create the Network of Brokers.

		# Network of Brokers Multicast
		nob_multicast_name: default

Variables are defined in *group_vars/all.yaml* file.

##### Example playbook

To execute this playbook:

    ansible-playbook -i hosts fuse-install.yaml

Inventory (*host* file):

		[fuse-lab-environment]
		rhel7jboss01
		rhel7jboss02
		rhel7jboss03

Playbook (*fuse-install.yaml* file):

		---
		- name: Fuse Standalone Playbook
			hosts: fuse-lab-environment
			serial: 1
			remote_user: root
			gather_facts: true
			become: yes
			become_user: root
			become_method: sudo
			roles:
				# Two Fuse Standalone with a Network of Brokers
				- { role: fuse-standalone, esb_name: 'esb01',  port_offset: '0', nob: 'true' }
				- { role: fuse-standalone, esb_name: 'esb02',  port_offset: '100', nob: 'true' }

Other alternatives:
* Two Fuse Standalone without a Network of Brokers:

		roles:
			- { role: fuse-standalone, esb_name: 'esb01',  port_offset: '0', nob: 'false' }
			- { role: fuse-standalone, esb_name: 'esb02',  port_offset: '100', nob: 'false' }

* One Fuse Standalone with Full profiles and One Fuse Standalone without A-MQ profile

		roles:
			- { role: fuse-standalone, esb_name: 'esb01',  port_offset: '0', nob: 'true' }
			- { role: fuse-standalone, esb_name: 'esb02',  port_offset: '100', esb_type: 'noamq' }

### fuse-deploy-bundle role

This role deploys several Application Bundles into a set of Fuse Standalone instances.

The main tasks done are:

* Download artifacts from a Maven Repository Manager
* Copy artifacts into **{{ FUSE_HOME }}/deploy** folder
* Install Features repositories (URL)
* Install Features from URL list of repositories

#### Configuration parameters

Role's execution could be configured with the following variables.

* **app_home**: Location to store the applications to be deployed before to do it.

		# Applications Home
		app_home: '/opt/fuse/applications'

* **maven_repository_manager**: Maven Repository location to resolve the artifacts
	to be deployed.

		# Maven Repository
		maven_repository_manager: http://rhel7jboss01:8081/nexus/content/groups/public

* **applications**: List of Maven dependencies to be deployed. The artifacts should
	be located using their GAV coordinates.

		# Application List to deploy
		applications:
			-
				groupId: com.redhat.camel
				artifactId: camel-amq-consumer
				version: 1.1.0-SNAPSHOT
			-
				groupId: com.redhat.camel
				artifactId: camel-amq-producer
				version: 1.1.0-SNAPSHOT
			-
				groupId: com.redhat.camel
				artifactId: camel-amq-forwarder
				version: 1.1.0-SNAPSHOT

* **features**: List of Features to be deployed. The artifacts should
	be located using their GAV coordinates. The *name* attribute defines the
	name of the feature to install from the list url defined by the GAV coordinates

		features:
			-
				groupId: com.redhat.fuse.demo
				artifactId: fuse-demo-features
				version: 1.1.0-SNAPSHOT
				name: fuse-demo-features

##### Example playbook

To execute this playbook:

    ansible-playbook -i hosts fuse-deploy-bundle.yaml

Inventory (*host* file):

		[fuse-lab-environment]
		rhel7jboss01
		rhel7jboss02
		rhel7jboss03

Playbook (*fuse-deploy-bundle.yaml* file):

		---
		- name: Fuse Deploy Bundles Playbook
			hosts: fuse-lab-environment
			serial: 1
			remote_user: root
			gather_facts: true
			become: yes
			become_user: root
			become_method: sudo
			roles:
				# Deploying Bundles in Two Fuse Standalone
				- { role: fuse-deploy-bundle, esb_name: 'esb01' }
				- { role: fuse-deploy-bundle, esb_name: 'esb02' }

### fuse-undeploy-bundle role

This role undeploys several Application Bundles from a set of Fuse Standalone instances.

The main tasks done are:

* Remove artifacts from **{{ FUSE_HOME }}/deploy** folder
* Uninstall Features from URL list of repositories
* Uninstall Features repositories (URL)

#### Configuration parameters

Role's execution could be configured with the following variables.

* **maven_repository_manager**: Maven Repository location to resolve the artifacts
	to be deployed.

		# Maven Repository
		maven_repository_manager: http://rhel7jboss01:8081/nexus/content/groups/public

* **applications_undeploy**: List of Maven dependencies to be undeployed. The artifacts should
	be located using their GAV coordinates.

		# Application List to undeploy
		applications_undeploy:
			-
				groupId: com.redhat.camel
				artifactId: camel-amq-consumer
				version: 1.1.0-SNAPSHOT
			-
				groupId: com.redhat.camel
				artifactId: camel-amq-producer
				version: 1.1.0-SNAPSHOT
			-
				groupId: com.redhat.camel
				artifactId: camel-amq-forwarder
				version: 1.1.0-SNAPSHOT

* **features_undeploy**: List of Maven dependencies to be undeployed.

		features_undeploy:
			-
				groupId: com.redhat.fuse.demo
				artifactId: fuse-demo-features
				version: 1.1.0-SNAPSHOT
				name: fuse-demo-features

##### Example playbook

To execute this playbook:

    ansible-playbook -i hosts fuse-undeploy-bundle.yaml

Inventory (*host* file):

		[fuse-lab-environment]
		rhel7jboss01
		rhel7jboss02
		rhel7jboss03

Playbook (*fuse-undeploy-bundle.yaml* file):

		---
		- name: Fuse Undeploy Bundles Playbook
			hosts: fuse-lab-environment
			serial: 1
			remote_user: root
			gather_facts: true
			become: yes
			become_user: root
			become_method: sudo
			roles:
				# Undeploying Bundles in Two Fuse Standalone
				- { role: fuse-undeploy-bundle, esb_name: 'esb01' }
				- { role: fuse-undeploy-bundle, esb_name: 'esb02' }

# Main References

* [Product Documentation for Red Hat JBoss Fuse](https://access.redhat.com/documentation/en/red-hat-jboss-fuse/)
* [Ansible Documentation](http://docs.ansible.com/ansible/)
* [How to remove the embedded ActiveMQ Broker from being deployed in JBoss Fuse?](https://access.redhat.com/solutions/1229533)
* [ActiveMQ - Network of Brokers](http://activemq.apache.org/networks-of-brokers.html)
