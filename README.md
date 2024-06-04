# Prisma Cloud Agentless Hub Org Permissions Templates

The templates in this folder are to allow segregation of permissions between hub and target accounts, when onboarding an organization into Prisma Cloud.
Both were validated by QA at some point of time, but were frozen since then - meaning that it is required to test them out on the current version before sending them to customers.

##Table of Contents

* [Introduction](#introduction)
* [Getting Started](#getting-started)
  * [Azure](#azure)
  * [AWS](#aws)
  * [GCP](#gcp)

## Introduction

When it comes to some customers, the standard onboarding template that is downloaded from the console will not be least permissive enough for agentless hub scanning, The templates here will help to overcome that by segregating the permissions needed to the hub subscription (specifically to the **resource group** created in the Hub subscription for the write permissions needed to create network infrastructure and the scanner VMs). The variable name where these permissions are assigned is called **"custom_agentless_resource_group_actions"**.

* **For any questions regarding these templates please ask in the #pcs-agentless slack channel**.

## Getting-Started

### Azure

**File name:** Azure-tenant-with-hub.tf.json

This terraform template is used for Azure Org onboarding. It receives the agentless hub subscription ID, tenant_id and initial location for the resource group as env variables: 

You can enter the values of the three variables at the time you run the TF template, or you can export them prior to running it.

* export TF_VAR_hub_subscription_id=”Subscription_id”
* export TF_VAR_tenant_id=”tenant_id”
* export TF_VAR_hub_resource_group_location="location" (i.e. eastus)

The hub subscription ID is needed to assign only the relevant network infrastructure related write permissions to the subscription that will be used as the “hub”, leaving all other subscriptions (aka target subscriptions) with only read permissions.

In this template, there is also a custom role for VM image scans that contains various write permissions that overlap with what the hub subscription uses. The role is called, **“custom_prisma_VM_Image_Scanning_role”**, and will assign the write permissions to every subscription in the tenant. The VM Image scan permission block is called, **“custom_VM_image_scanning_write_actions”**.

**You will need to remove the following references to this role and the permission block if VM image scans are out of scope and/or to ensure least privilege for the hub scan. 

#### "custom_prisma_VM_Image_Scanning_role"

* Line 90
* Lines 146-155
* Line 187-195

#### "custom_VM_image_scanning_write_actions"

* Lines 192-195
* Lines 584-599

### How it Works

On line 80, the template checks to see if a resource group named "PCCAgentlessScanResourceGroup" exists in the hub subscription. If no, a resource group with that name is created in the hub subscription in the region specified in the env variable at the beginning of this document. If it already exists, it moves on to assign the write permissions to a role specific to that resource group.

### Limitations

If the "Permission Check" will be enabled in the agentless config, the **"Microsoft.Resources/subscriptions/resourceGroups/write"** permission (Line 503) is required for all subscriptions in the org. Even though the write permissions are isolated to the resource group in the hub subscription, each target subscription wants to create a resource group when scanned (although it will remain empty). 

The only workaround for now would to be remove this permission on Line 503 and **disable** the permission check in the agentless config for all subscriptions.


### AWS

**File name:** AWS-org-with-hub-and-networking-deny.json

This CFT allows for onboarding of an AWS org to isolate the write permissions needed for an agentless scan to a single “hub” account, while denying those same permissions to all other “target” accounts.

This CFT has three variables that you need to add values for:

* Hub Account number (Line 32)
* AWS Org ID (Line 27)
* Replace the Prisma ID with the correct one for the tenant you are working in (~36 locations)
  * When the template is downloaded from the console, the Prisma ID get appended to several values in the CFT, including the Prisma Cloud Role Name (i.e. PrismaCloudRole-1182536372507056128).
  * To clean this up for your CFT, copy the customer's Prisma ID (i.e. from the licensing page of the customer's tenant or the Support App).
  * Do a find on the current value (i.e. 1182536372507056128) and replace all with the new Prisma ID.
  * This is not required, but will be cleaner and specific to the tenant you are working in.
* External ID ("sts:ExternalId": "xxxx" has 3 locations in the CFT)
  * To get the correct externalID value for the tenant you are working in, you can either copy it from the AWS onboarding page in the console (if onboarding has been done before), or you can download the CFT template from Prisma Cloud and copy from it there.
  * Similar to above, find the current external ID value (search for sts:ExternalId and copy the value) in the template and do a find/replace all with correct value for the tenant you are working in.

### How it Works

When this CFT is deployed as a Stack in AWS, it does the following:
* Creates 3 stacksets
  * A stackset to create the Prisma Cloud role for all of the target accounts (including the various permission policies), and adds an additional policy called, "Agentless Targets" that denies access to the write permissions needed for the agentless scan.
  * A stackset to create the Prisma Cloud role for the hub account (including the various permission policies), and adds an additional policy called, "Agentless Hub" that allows access to the write permissions needed for the agentless scan.
  * A stackset that creates the networking infrastructure in the hub account needed by the agentless scans (i.e. VPC, subnet, IGW, etc). This is dones in case the customer does not want Prisma Cloud creating and destroying these resources each time a scan kicks off.
    * By default, this stackset is deployed to the region selected when the onboarding template is deployed, but could easily be modified to deploy this networking infrastructure to every region being used by the customer.
    * To use the networking stack created by the stackset, you must add all of the regions where VMs/containers will be scanned.
      * On Line 181, delete the snippet below
        > {  "Ref": "AWS::Region"      }
      * Add the region name values into the Regions array
        > "Regions": [ "us-east-1", "us-east-2" ]






