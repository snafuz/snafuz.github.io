---
title: OCI-C to OCI Migration demo runbook
description: Andrea Marchesini's guides
---
> This document is intended to support training delivery session and it's based on a pre-existent environment. It doesn't mean to describe the full migration process.
For further details about the OCI-C to OCI migration process please see:
[Discovery](https://docs.oracle.com/en/cloud/iaas/compute-iaas-cloud/migrate-oci/identify-and-translate-resources-your-source-environment.html#GUID-75664E09-FAD9-4CA0-A6D5-86F23DDE9B49)
and 
[Migration](https://docs.oracle.com/en/cloud/iaas/compute-iaas-cloud/migrate-oci/migrate-vms-and-block-storage.html#GUID-7099C387-9826-4D22-A448-C3FC4ABBD094)

### OCI-C to OCI migration demo runbook 

### Discovery

1. update opcmigrate tool
`./mig install`

2. setup _~/.opc/profiles/default_
3. run discovery tool, to catalog all the resources in _resource-default.json_
`opcmigrate discover -c /Compute-610669640/andrea.marchesini@oracle.com` 
4. useful command to help planning phase
	```Bash
	# view a list summarizig the resources
	opcmigrate summary

	# show the network details
	opcmigrate network

	# generate Excel spreadsheet with all the resources
	opcmigrate report
	# generate a resource relashionsips graph
	opcmigrate graph

	```
5. generate migration plan
	```Bash
	opcmigrate plan create --include instance ip_network vnic interface security_rule vnic_set acl security_protocol ip_address_prefix_set --output migration-plan.json
	```
6. generate terraform script
	```Bash
	opcmigrate generate --with-security-rule-union --plan migration-plan.json --output main.tf
	```
7. generate instance export files
	```Bash
	opcmigrate instances-export --plan migration-plan.json --format json > instances.json
	opcmigrate instances-export --plan migration-plan.json --format yaml > instances.yaml
	```
   

### Migration

1.  setup `secret.yml`
2. copy PEM key required for the OCI API connection in _/home/opc/.oci/oci_api_key.pem_
3.  setup the migration service
	`opcmigrate migrate instance service setup`
4. setup  source instances
	+  __Linux__ 
		+ setup `hosts.yml` including linux instances
		+ copy into Control-S private SSH keys to access the source instances
		+ run `opcmigrate migrate instance source setup`
	+ __Windows__ 
		+ connect via RDP as administrator and run `windows_migrate.ps1`
5.  on OCI 
	+ 	create `ocic-oci-sig` bucket and PAR to access it
	+ create a VCN with at least one subnet, igw,  and relative route rule 
6. create Control-T instance
	`opcmigrate migrate instance ctlt setup`
7. on Control-S, start migration service
	`opcmigrate migrate instance service start``
8. setup job descriptor if you want to run multiple jobs in parallel or migrate just one VM at a time
9.  start migration job
	`opcmigrate migrate instance job run --job_file workdir/job_1.json`
10. to check the job status
	`tail -n 30 -f /images/migration_logs/run_migration.log`



> _written by Andrea Marchesini - last update: May 6th, 2019_
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTcwNDEzMDY3MywtMjkwOTAwOTM2XX0=
-->