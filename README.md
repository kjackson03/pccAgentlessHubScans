# Prisma Cloud Agentless Hub Org Permissions Templates

The templates in this folder are to allow segregation of permissions between hub and target accounts, when onboarding an organization into Prisma Cloud.
Both were validated by QA at some point of time, but were frozen since then - meaning that it is required to test them out on the current version before sending them to customers.

##Table of Contents

* [Introduction](#introduction)
* [Getting Started](#getting-started)
  * [Azure](#azure)
  * [AWS](#aws)
  * [GCP](#gcp)
* [Limitations](#limitations)

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




