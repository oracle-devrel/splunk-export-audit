# splunk-export-audit

[![License: UPL](https://img.shields.io/badge/license-UPL-green)](https://img.shields.io/badge/license-UPL-green) [![Quality gate](https://sonarcloud.io/api/project_badges/quality_gate?project=oracle-devrel_splunk-export-audit)](https://sonarcloud.io/dashboard?id=oracle-devrel_splunk-export-audit)

# Splunk-Export-Audit

## Table Of Contents
- [Splunk-Export-Audit](#splunk-export-audit)
  - [Table Of Contents](#table-of-contents)
  - [Introduction](#introduction)
  - [Components](#components)
  - [Setup a Splunk Trial for Testing](#setup-a-splunk-trial-for-testing)
  - [Design Goals](#design-goals)
  - [Quickstart For Setup On OCI Side](#quickstart-for-setup-on-oci-side)
    - [Create Compartments and Groups](#create-compartments-and-groups)
    - [Create a Dynamic Group](#create-a-dynamic-group)
    - [Create Tenancy IAM policy - Splunk Export Group](#create-tenancy-iam-policy---splunk-export-group)
    - [Create Tenancy IAM policy - Splunk Export Dynamic Group](#create-tenancy-iam-policy---splunk-export-dynamic-group)
    - [Create Compartment Level IAM Policy](#create-compartment-level-iam-policy)
    - [Create a VCN, Subnet & Security Lists](#create-a-vcn-subnet--security-lists)
    - [Configure Cloud Shell](#configure-cloud-shell)
    - [The Cloud Shell](#the-cloud-shell)
    - [Create a Function Application](#create-a-function-application)
    - [Getting Started with Fn Deployment](#getting-started-with-fn-deployment)
    - [Step-8](#step-8)
    - [Deploy the Functions](#deploy-the-functions)
    - [Create  an API Gateway](#create-an-api-gateway)
    - [Create API Gateway Deployment Endpoints](#create-api-gateway-deployment-endpoints)
    - [Create Notification Channels](#create-notification-channels)
    - [Create Streaming](#create-streaming)
    - [Set the Environment Variables for Each Function](#set-the-environment-variables-for-each-function)
  - [Invoke and Test !](#invoke-and-test)
  - [Health Checks for Scheduled Trigger](#health-checks-for-scheduled-trigger)
    - [Target Settings](#target-settings)


## Introduction 
For Integrated SecOps, the SIEM plays an important component and Splunk is a very popular SIEM solution. The setup described below helps in near-realtime, zero-touch audit-log export to the Splunk Http event Collector for Indexing and Analysis.The same architecture can be used to export to almost any SIEM as it leverages the HTTP Event Collector.

![](media/TheSimpleArchitecture.png)
A Scalable and Low Cost Splunk event exporter to publish OCI Audit Logs to Splunk.

## Components
-   The `OCI Audit API` is  queried for audit events every 2 minutes for all regions and all compartments relevant to the tenancy. 
-   The `OCI Functions` trigger a series of queries and publish events to the Splunk HTTP Event Collector End point.
-  `OCI Health Checks` are used to trigger a GET Request on to a public HTTPS Endpoint every `5-min` / `1-min` /  `30s`
-   The `OCI API Gateway` is used to Front-End the Functions. to allow for the fn invokation process through HTTP(S) Requests rather than using the `OCI Fn Invocation mechanism` or the `oci-curl`
- `OCI Streaming` as a scalable source of persistence that stores `state`, `audit events`, `most up-to-date compartment & region list` in the tenancy. 
- `OCI Notifications` are used to notify downstream functions for triggering along with information on  `stream-cursor, offset, partition` information to the next function. 
- The `Splunk HTTP event Collector` is a simplified mechanism that splunk provides to publish events 

![](/media/SimpleRepresentation.png)

## Setup a Splunk Trial for Testing
* Splunk provides a 15 day Cloud trial and the capability to store about 5GB worth of event data that you forward/export to Splunk. Here's the link to sign-up [Splunk Sign Up](https://www.splunk.com/en_us/download.html)
* Select Splunk-Cloud, provide your data and Login to Splunk. 
* To setup the HTTP Event Collector which we leverage, the solution refer to link [Setup HTTP event collector](https://docs.splunk.com/Documentation/Splunk/8.0.2/Data/UsetheHTTPEventCollector)
* Points to note 

## Design Goals 
``` 
Self Perpetuating
Event driven 
Scalable 
Low-Cost
Zero maintenance
```

## Quickstart For Setup On OCI Side
This quickstart assumes you have working understanding of basic principles of OCI around IAM | Networking and you know how to get around the OCI Console. 

### Create Compartments and Groups
- `Burger-Menu` --> `Identity` --> `Compartments | Users | Groups`
1. Create a Compartment `splunk-export-compartment`
2. Create a Group  `splunk-export-users`
3. Add `Required User` to group `splunk-export-users`
4. Create a Dynamic Group `splunk-export-dg`
5. Write appropriate IAM Policies at the tenancy level and compartment level.

### Create a Dynamic Group
- `Burger-Menu` --> `Identity` --> `Dynamic Groups`
 - Create a Dynamic Group `splunk-export-dg` Instances that meet the criteria defined by any of these rules will be included in the group.
```
ANY {instance.compartment.id = [splunk-export-compartment OCID]}
ANY {resource.type = 'ApiGateway', resource.compartment.id =[splunk-export-compartment OCID]}
ANY {resource.type = 'fnfunc', resource.compartment.id = [splunk-export-compartment OCID]}
```

### Create Tenancy IAM policy - Splunk Export Group 
- `Burger-Menu` --> `Identity` --> `Policies`
 - Create an IAM Policy `splunk-export-tenancy-policy` with the following policy statements in the `root` compartment 
```
Allow group splunk-export-users to manage repos in tenancy
Allow group splunk-export-users to read audit-events in tenancy
Allow group splunk-export-users to read tenancies in tenancy
Allow group splunk-export-users to read compartments in tenancy
Allow service FaaS to read repos in tenancy
```

###  Create Tenancy IAM policy - Splunk Export Dynamic Group 
- `Burger-Menu` --> `Identity` --> `Policies`
 - Create an IAM Policy `splunk-export-dg-tenancy-policy` with the following policy statements in the `root` compartment 
```
Allow dynamic-group splunk-export-dg to read audit-events in tenancy
Allow dynamic-group splunk-export-dg to read tenancies in tenancy
Allow dynamic-group splunk-export-dg to read compartments in tenancy
```

### Create Compartment Level IAM Policy
- `Burger-Menu` --> `Identity` --> `Policies`
Create an IAM Policy `splunk-export-compartment-dg-policy` inside the compartment `splunk-export-compartment`
```
Allow service FaaS to use all-resources in compartment splunk-export-compartment
Allow dynamic-group splunk-export-dg to use ons-topics in compartment splunk-export-compartment
Allow dynamic-group splunk-export-dg to use stream-pull in compartment splunk-export-compartment
Allow dynamic-group splunk-export-dg to use stream-push in compartment splunk-export-compartment
Allow dynamic-group splunk-export-dg to use virtual-network-family in compartment splunk-export-compartment
```

### Create a VCN, Subnet & Security Lists
- `Burger-Menu` --> `Networking` --> `Virtual Cloud Networks`
 - Use VCN Quick Start to Create a VCN `splunk-export-vcn` with Internet.
 - Connectivity Go to Security List and Create a `Stateful Ingress Rule` in the `Default Security list` to allow Ingress Traffic in  `TCP 443`
 - Go to Default Security List and verify if a `Stateful Egress Rule` is available in the `Default Security List` to allow egress traffic in  `all ports and all protocols`


### Configure Cloud Shell

![Cloud Shell](media/cloudShell.png)

Setup Cloud Shell in your tenancy - [Link](https://docs.cloud.oracle.com/en-us/iaas/Content/API/Concepts/cloudshellgettingstarted.htm?TocPath=Developer%20Tools%20%7C%7CUsing%20Cloud%20Shell%7C_____0)

### The Cloud Shell

* Oracle Cloud Infrastructure Cloud (OCI) Shell is a web browser-based terminal accessible from the Oracle Cloud Console. 
* Cloud Shell provides access to a Linux shell, with a pre-authenticated Oracle Cloud Infrastructure CLI, a pre-authenticated Functions, Ansible installation, and other useful tools for following Oracle Cloud Infrastructure service tutorials and labs. 


### Create a Function Application
- `Burger-Menu` --> `Developer Services` --> `Functions`
 - Create a Function Application `splunk-export-app` in the compartment `splunk-export-compartment` while selecting `splunk-export-vcn` and the `Public Subnet`

![FnApplication](media/createFnApplication.png)


If you are not an IAM Policy Expert, just create these policies as shown in the Function Pre-requisites


![Pre-Requisites](media/fn-pre-reqs.png)

 - Setup Papertrail / OCI Logging Service to debug Function executions if required.  [Setup PaperTrail](https://papertrailapp.com/) , check them out. 



### Getting Started with Fn Deployment
Click on the Getting started Icon after Function Application Creation is done. 

![Getting Started](media/fn-getting-Started.png)

Follow the steps on the screen for simplified Fn Deployment. While you have the option of local Fn Development environment, I'd recommend using the cloud shell if you simply want to deploy Functions. 

Follow the steps until **Step-7**

### Step-8
Instead of creating a new fn we are deploying an existing function. So clone

Clone the Repo in the cloud shell 
`git clone https://github.com/vamsiramakrishnan/splunk-export-audit.git`


### Deploy the Functions
Each folder within the repo represents a function , go to each folder and deploy the function using the the `fn --verbose deploy `
```
cd splunk-export-audit
cd list-regions
fn --verbose deploy splunk-export-app list-regions

cd fetch-audit-events
fn --verbose deploy splunk-export-app fetch-audit-events

cd publish-to-splunk
fn --verbose deploy splunk-export-app publish-to-splunk
```
The Deploy Automatically Triggers an Fn Build and Fn Push to the Container registry repo setup for the functions.

### Create  an API Gateway 
* `Burger-Menu` --> `Developer Services` --> `API Gateway`
 * Create an API Gateway `splunk-export-apigw` in the compartment `splunk-export-compartment`while selecting `splunk-export-vcn` and the `Public Subnet`

### Create API Gateway Deployment Endpoints
Map the endpoint as follows 
 | Deployment Name | Prefix  | Method | Endpoint     | Fn-Name      |
 | --------------- | ------- | ------ | ------------ | ------------ |
 | list-regions    | regions | GET    | /listregions | list-regions |

```
Note:The API Gateway is setup in this example with HTTPS without an Auth Mechanism , but this can be setup with an authorizer Function , that works with a simple Token mechanism
```
### Create Notification Channels 
- `Burger-Menu` --> `Application Integration` --> `Notifications`
- Create two notification channels `splunk-fetch-audit-event`  `splunk-publish-to-splunk`. Create subscriptions to Trigger the Functions

### Create Streaming
- `Burger-Menu` --> `Analytics` --> `Streaming`
```
Note: Using a Single Partition and Single Streaming endpoint for Audit, Can scale based on requirements
```
| Stream Attribute | value                |
| ---------------- | -------------------- |
| stream-name      | splunk-export-stream |
| retention-period | 24 Hours             |
| partitions       | 1                    |
| stream-pools     | default-stream-pool  |



### Set the Environment Variables for Each Function
These environment variables help call other functions. One after the other. 


| Fn-Name            | Parameter Name     | Description                                                                                                     | Example                                   |
| ------------------ | ------------------ | --------------------------------------------------------------------------------------------------------------- | ----------------------------------------- |
| list-regions       | records_per_fn     | Batch Size of Number of Events to be processed in one go by the Next Fn                                         | 50                                        |
| list-regions       | audit_topic        | OCID of Notifications topic used to trigger and notify the Fn fetch-audit-events                                | ocid1.onstopic.oc1.phx.aaaaaaaa           |
| list-regions       | stream_ocid        | OCID of the Stream used to Publish the list of compartments and regions from where Audit events will be fetched | ocid1.stream.oc1.phx.amaaaaaa             |
| list-regions       | streaming_endpoint | Endpoint of Streaming, depends on which region you provision Streaming                                          | ocid1.stream.oc1.phx.amaaaaaa             |
| fetch-audit-events | records_per_fn     | Batch Size of Number of Events to be processed in one go by the Next Fn                                         | 30                                        |
| fetch-audit-events | splunk_topic       | OCID of the Topic used to Notify& Trigger the publish-to-splunk Function                                        | ocid1.onstopic.oc1.phx.aaaaaaaa           |
| fetch-audit-events | stream_ocid        | OCID of the Stream used to Publish the actual Audit Event payload                                               | ocid1.stream.oc1.phx.amaaaaaa             |
| fetch-audit-events | streaming_endpoint | Endpoint of Streaming, depends on which region you provision Streaming                                          | ocid1.stream.oc1.phx.amaaaaaa             |
| publish-to-splunk  | source_source_name | The Source Name that you would like Splunk to see                                                               | oci-hec-event-collector                   |
| publish-to-splunk  | source_host_name   | The Source Hostname that you would like Splunk to see                                                           | oci-audit-logs                            |
| publish-to-splunk  | splunk_url         | The Splunk Cloud URL ( Append input to the beginning of your splunk cloud url, do not add any http/https etc.   | input-prd-p-hh6835czm4rp.cloud.splunk.com |
| publish-to-splunk  | splunk_hec_token   | The Token that is unqiue to that HEC                                                                            | TOKEN                                     |
| publish-to-splunk  | splunk_index_name  | The index into which you'd like these logs to get aggregated                                                    | main                                      |
| publish-to-splunk  | stream_ocid        | OCID of the Stream used to Publish the actual Audit Event payload                                               | ocid1.stream.oc1.phx.amaaaaaa             |
| publish-to-splunk  | streaming_endpoint | Endpoint of Streaming, depends on which region you provision Streaming                                          | ocid1.stream.oc1.phx.amaaaaaa             |
| publish-to-splunk  | splunk_hec_port    | The listener port of the HEC Endpoint of Splunk                                                                 | 8088                                      |

## Invoke and Test !
Invoke Once and the loop will stay active as long as the tenancy does continuously pushing events to Splunk . 
```
curl --location --request GET '[apigateway-url].us-phoenix-1.oci.customer-oci.com/regions/listregions'
```
If all is well in papertrail/ oci-function-logs/metrics, proceed.
## Health Checks for Scheduled Trigger
Create a `Health Check` named `splunk-export-health-check` with the following settings 
### Target Settings
Get the API-Gateway URL where the list-regions Fn is deployed. 
**Example**
``` https://<Random-Alphanumeric-String>.apigateway.us-phoenix-1.oci.customer-oci.com/regions/listregions```
Copy the String  `<Random-Alphanumeric-String>.apigateway.us-phoenix-1.oci.customer-oci.com` and ignore the rest
### Other Health Check Settings
| Field         | Setting              |
| ------------- | -------------------- |
| Vantage Point | *Select any One*     |
| Request Type  | HTTP                 |
| Protocols     | HTTPS                |
| Port          | 443                  |
| Path          | /regions/listregions |
| Timeout       | 30s                  |
| Interval      | 5 Min                |
| Method        | GET                  |

```
Note:The API Gateway is setup in this example with HTTPS without an Auth Mechanism , but this can be setup with an authorizer Function , that works with a simple Token mechanism.If Auth is setup , the token can be specified in the Header of the Health Check.

```

## Contributing
This project is open source.  Please submit your contributions by forking this repository and submitting a pull request!  Oracle appreciates any contributions that are made by the open source community.

## License
Copyright (c) 2022 Oracle and/or its affiliates.

Licensed under the Universal Permissive License (UPL), Version 1.0.

See [LICENSE](LICENSE) for more details.

ORACLE AND ITS AFFILIATES DO NOT PROVIDE ANY WARRANTY WHATSOEVER, EXPRESS OR IMPLIED, FOR ANY SOFTWARE, MATERIAL OR CONTENT OF ANY KIND CONTAINED OR PRODUCED WITHIN THIS REPOSITORY, AND IN PARTICULAR SPECIFICALLY DISCLAIM ANY AND ALL IMPLIED WARRANTIES OF TITLE, NON-INFRINGEMENT, MERCHANTABILITY, AND FITNESS FOR A PARTICULAR PURPOSE.  FURTHERMORE, ORACLE AND ITS AFFILIATES DO NOT REPRESENT THAT ANY CUSTOMARY SECURITY REVIEW HAS BEEN PERFORMED WITH RESPECT TO ANY SOFTWARE, MATERIAL OR CONTENT CONTAINED OR PRODUCED WITHIN THIS REPOSITORY. IN ADDITION, AND WITHOUT LIMITING THE FOREGOING, THIRD PARTIES MAY HAVE POSTED SOFTWARE, MATERIAL OR CONTENT TO THIS REPOSITORY WITHOUT ANY REVIEW. USE AT YOUR OWN RISK. 