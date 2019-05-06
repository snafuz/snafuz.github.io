---
title: This is my title
layout: post
---


# OCI-C to OCI Migration

## Discovery phase

> In this phase we will deploy and configure the Migration Controller Instance (Control-S) in the source environment. 

### Deploy Control-S

In your OCI-C account, create the source controller (Control-S) instance with the following configuration.

* _Image: OL_7.5_UEKR4_x86_64_MIGRATION._ This image is available under Oracle Images on the console.
* _Shape: General Purpose oc7 (16 OCPUs, 120-GB RAM)_ or any other shape with a sufficient number of OCPUs
* _SSH Key: Associate an SSH public key with the Control-S instance_ You'll use the corresponding private key to connect to the Control-S instance. This key isn't the same as the SSH key pair used to access Linux source instances from Control-S

### Configure Control-S to run Discovery Tool

Once the Control-S instance has started, connect to the instance using SSH.  All of the tools required for the migration are already on the machine, but additional configuration is required to provide details of the source and target environments.

1. Download the latest code and configuration files. (execute `./mig install` on the server)
2. Setup your OCI-C profile.  
    The network and resource discovery tool connects to your source environment using connection information that you provide in a profile file. The tool use the _default_ profile in `~/.opc/profiles/default`  
    Below a profile template:

``` json
{
    "global": {
        "format": "text",
        "debug-request": false
    },
    "compute": {
        "user": "/Compute-example/user@example.com",
        "endpoint": "compute.uscom-central-1.oraclecloud.com"
    },
    "lbaas": {
        "user": "user@example.com",
        "endpoint": "lbaas-00000000000000000000000000000000.balancer.oraclecloud.com",
        "region": "uscom-central-1"
    },
    "paas": {
        "user": "user@example.com",
        "identity_id": "idcs-00000000000000000000000000000000",
        "endpoint": "psm.us.oraclecloud.com",
        "region": "uscom-central-1"
    },
    "object_storage": {
        "auth-endpoint": "uscom-central-1.storage.oraclecloud.com/auth/v1.0",
        "user": "Storage-example:user@example.com",
        "endpoint": "uscom-central-1.storage.oraclecloud.com/v1/Storage-example"
    }
}

```

### Discovery

Now it's possible to use `opcmigrate` tool to discover the source environment on OCI-C.
To generate a JSON formatted file that contains information about all the networking objects, instances, storage volumes, and other resources in the site, run

`opcmigrate discover`

The above command will use the default profile and run the discovery process and the whole OCI-C environment.
It's possible to specify the profile file and limit the discovery to specific OCI-C containers

`opcmigrate -p am-profile discover -c /Compute-0000000/andrea.marchesini@oracle.com`

This will generate a _resource-am-profile.json_

The _resource.json_ file serves as source to to explore all the resources, to generate reports and to plan the migration.

```Bash
# show OCI-C network resources
opcmigrate network

# generate a migration plan
opcmigrate plan create --include instance ip_network vnic interface security_rule vnic_set acl security_protocol ip_address_prefix_set --output migration-plan.json

# generate terraform script
opcmigrate generate --with-security-rule-union --plan migration-plan.json --output main.tf


```
## Migration phase 

Use the outcome of the Discovery process to plan the migration. 
Setup Control-S  and Control-T to run the migration process and start the migration.

### Prepare VM Migration  

1. Configure the file  `secret.yml`in the directory _/home/opc/ansible_. You can use the sample file available at _/home/opc/ansible/secret.yml.sample_ to create your _secret.yml_ file. 
Enter the details of your OCI Compute Classic account and your OCI OCIDs. 
Also use the output of the command `opcmigrate instances-export --plan migration-plan.json --format yaml` to provide a list of instances to be migrated.

	``` bash
	# OCI info
	# Destination environment
	compartment_id: ocid1.compartment.oc1..aaaaaaaa...
	user_id: ocid1.user.oc1..aaaaaaaa...
	fingerprint: a0:a0:a0:a0:a0...
	tenancy_id: ocid1.tenancy.oc1..aaaaaaaa...
	region: us-ashburn-1
	availability_domain: kWVD:US-ASHBURN-AD-3
	# version and shape used to the Control-T instance
	# 'Oracle Linux' is the only supported operating_system

	oracle_linux_version: '7.6'
	shape: 'VM.Standard2.1' 

	# subnet must be from the availability_domain you specified
	subnet_id: ocid1.subnet.oc1.phx.aaaaaaaa...

	# optional pass_phrase if it's used for OCI PEM file
	pass_phrase:

	# PAR is used to upload info to ocic-oci-sig bucket when the instance comes up on OCI target side
	# to facilitate attachments for migrated volumes

	ocic_oci_sig_par: https://objectstorage.us-phoenix-1.oraclecloud.com/p/aaaa...

	# OCI-C info
	# specify your endpoint here
	opc_profile_endpoint: compute.uscom-central-1.oraclecloud.com
	opc_password: 
	container: /Compute-000000/user

	# number of attachments slots on ctls instance to be used for volume migration (maximum is 8)
	workerThreadCount: 10  # The number of worker threads working on volume migrations
	targetControllerAvailableStorageInGB: 2048
	# instance name is composed of the label and UUID like this my_instance/fd2cd6d5-4b53-4275-a18f-c245b3e002c7

	instances:
	- {attached_only: 'false', name: /Compute-000000/user/instance01...,os: windows, osSku: Server 2016 Standard,specified_volumes_only: [], "shutdown_policy": "wait", "specified_launch_mode": "PARAVIRTUALIZED"}

	- {attached_only: 'false', name: /Compute-000000/user/instance02...,os: linux, osSku: '',specified_volumes_only: [], "shutdown_policy": "wait", "specified_launch_mode": "PARAVIRTUALIZED"}
	```

2.  run  the service setup command  to apply the above configuration  
	`opcmigrate migrate instance service setup`  

3.	configure the file `hosts.yml` in _/home/opc/ansible/_. You can use the provided _/home/opc/ansible/hosts.yml.sample_ as a template.
	```yaml
	source:
	  hosts:
	    1.1.1.1: #instance public IP
	      label: your_label_here #Label must be unique, you can find in the OCI-C web console
	      remote_user: user_name_here
	      ansible_ssh_private_key_file: ~/.ssh/private_key_here
	      # If migrating Linux with Python 3 only uncomment the line below
	      # ansible_python_interpreter: /usr/bin/python3
	    2.2.2.2:
	      label: your_label_here
	      remote_user: user_name_here
	      ansible_ssh_private_key_file: ~/.ssh/private_key_here
	```
	> The label  is the unique identifier provided in the OCI-C console with the format: _/Compute-0000000/user/instance_name/__ID___. Be aware of the __ID__ may change after instance stop/start. 

4.  copy PEM key required for the OCI API connection in _home/opc/.oci/oci_api_key.pem_

##### Prepare Source VMs - Linux

1. copy into Control-S the SSH private key(s) to access the source linux instance(s) 
2. run  
`opcmigrate migrate instance source setup`  
Review the output from this command and follow the instructions provided by the script, if any.

##### Prepare Source VMs - Windows
1.  Use RDP to log in to the instance as the Administrator.
2.  Copy the file  _/home/opc/src/windows_migrate.ps1_  from Control-S to each source instance.
3.  On each source instance, navigate to the folder where you've saved the file and run it  
	`windows_migrate.ps1`

### Migration

Create Control-T instance in OCI by running the bleow commando on Control-S

`opcmigrate migrate instance ctlt setup`

Start migration service.

`opcmigrate migrate instance service start`

To start the migration run 
`opcmigrate migrate instance job run`

By default, the migration job uses the list of instances provided in the  `secret.yml`  file to identify resources to be migrated. If you want to run multiple jobs in parallel or migrate just one VM at a time, specify a job file for each job.  

`opcmigrate migrate instance job run --job_file <full_path/job_file_name>`

You can use the instances.json file as starting point to create your job file(s).  

The migration begins and the display shows the ongoing status of the process. Depending on the size and number of storage volumes being migrated, the process could take a few hours to complete.


> _written by Andrea Marchesini - last update: May 6th, 2019_
<!--stackedit_data:
eyJwcm9wZXJ0aWVzIjoiYXV0aG9yOiBBbmRyZWEgTWFyY2hlc2
luaVxuIiwiaGlzdG9yeSI6WzY5MjExMDkyNl19
-->