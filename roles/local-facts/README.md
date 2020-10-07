local-facts
===========

This role will gather and set the following pre-requisite local-facts to ensure the playbook can run.

Fact Name | Description
--- | ---
FULL_CLUSTER_NAME | The full cluster name
CLOUD | The cloud type (eg: AWS, Azure, GCP)
REGION | The cloud region (eg: eu-east-2, uk-south)
CLUSTER_VERSION | Cluster version to one decimal place (eg: 4.4)
storage_class | Storage class type that's being used for PVCs
URL | The cluster URL 
AMI_ID | The AWS specific Amazon Machine Image ID which is used to create a virtual machine within the Amazon Elastic Compute Cloud