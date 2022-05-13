---
title: Integrating with CrowdStrike Threat Intelligence
---
Joint customers of AWS and CrowdStrike can gain the benefits of CrowdStrike's use
  of sophisticated signatureless artificial intelligence/machine learning and indicators
  of attack (IOA) to alert on connections to and from suspicious domains.

By integrating CrowdStrike Threat Intelligence with AWS Network Firewall, joint
  customers can enhance their cloud network security capabilities using native services.

## About the Integration
For security and compliance purposes, customers often have to control ingress and egress traffic related to Amazon EC2 instances and containers.  Previously, in order to achieve domain filtering, customers would have used a combination of NAT gateways and Squid or third party firewalls.  Stateful TCP/IP and UDP inspection was performed using Security Groups.   AWS Network Firewall extends the ability to monitor and control ingress and egress network traffic with its integration with AWS Firewall Manager and its ability to scale automatically.   

The CrowdStrike threat intelligence feed is already seamlessly integrated with Amazon GuardDuty. Clients of Amazon GuardDuty already gain the benefits of CrowdStrike's use of sophisticated signatureless artificial intelligence/machine learning and indicators of attack (IOA) to alert on connections to and from suspicious domains.  The AWS Network Firewall provides exciting opportunities for its customers to enhance their cloud network security capabilities using its native services. 

## How to Deploy
Please note the following when running the demo.

* _*This Template will take 15-20mins to fully deploy in your VPC*_

* _*It can take 15 minutes for CrowdStrike detections to appear in Security Hub*_

## Prerequisites

1. You must install the template in a region that currently supports the AWS Network Firewall. You can check the latest [here](https://aws.amazon.com/about-aws/global-infrastructure/regional-product-services/?p=ngi&loc=4).

    _(Currently US East (N. Virginia), US West (Oregon), and Europe (Ireland) Regions)_

2. You must have security hub enabled in the region where you are testing.

3. The demo will require the following CrowdStrike subscriptions 
   * Falcon Prevent
   * Falcon X

4. You must have the ability to create OAuth2 API keys in the falcon console

5. The demo will use the following AWS services.  You must have the ability to run these services in your account. 
   * Security Hub
   * EC2 Amazon Linux 2
   * Lambda
   * SQS
   * Systems Manager
   
## Demo Description
The diagram below shows the infrastructure deployed by the cloudformation template and the flows involved.  The demo will allow you to evaluate the two integration scenarios discussed in the [overview](overview.md) 

![](images/demo-infra.png)

The Cloudformation template will setup the following in a new VPC 

* Windows EC2 Instance in a private subnet

    The windows instance is used to generate a detection related to a suspicious domain. 
 
* Linux EC2 Instance in a public subnet
    
    The EC2 linux instance runs the security hub integration process that pulls detections from the CrowdStrike API and sends them them as "findings" to AWS security hub.  Note an SQS queue and lambda function are also deployed to assist with the process. See the Security Hub Integration (FIG) documentation for more information. 

* Internet GW

* VPC 
    - 2 x Subnets (Public + Private)
    - 3 x Route tables (Public + Private + IGW)
    
* 1 x SQS Queue
    Security Hub Integration 

* 4 x Lambda Functions - 
    
    - Function to deploy the Network Firewall
    
    - Security Hub (FIG) function required for security hub integration 
    
    - Function that is triggered by a security hub custom action to extract the domain information from the finding and 
    push it to the Network Firewall rule. 
    
    - Function that is triggered by a cloudwatch event to update a domain list with current IOC's from Falcon X

### Routing Setup
The diagram below shows the setup of the VPC

![](images/demo-routing.png)

When the Network Firewall is created a VPC endpoint is created in each AZ.  The vpc endpoint is then used as the next 
hop in the routing tables of subnets that are to be protected by the network firewall.

#### Private Subnet
The private route table is associated with the private subnet (protected subnet) and has one additional route table entry
 - default route 0.0.0.0/0 with next hop as the firewall vpc endpoint (vpce)


#### Firewall Subnet
The Firewall route table is associated with the firewall subnet and has one additional route table entry
 - default route 0.0.0.0/0 with next hop as IGW


#### IGW Subnet
The Gateway route table has an edge association with the Internet Gateway and has one additional route table entry
 - 10.0.1.32/28(the protected subnet) with next hop as the firewall vpc endpoint (vpce)


## Deployment Steps
1. Create CrowdStrike API keys
Create an OAuth2 key pair with permissions for the Streaming API and Hosts API
   
   | Service                           | Read | Write |
   | -------                           |----- | ----- |
   | Detections                        | x    |       |
   | Hosts                             | x    |       |
   | Detections                        | x    |       |
   | Actors (Falcon X)                 | x    |       |
   | Indicators (Falcon X)             | x    |       |  
   | Host groups                       | x    |       |  
   | Incidents                         | x    |       |  
   | IOCs (Indicators of Compromise)   | x    |       |
   | Sensor Download                   | x    |       |
   | Event Streams                     | x    |       | 

   Screenshot from key creation. Copy the CLIENT ID and SECRET values for use later as input parameters to the cloudformation template.
    
   ![](images/api-key.png)
    
   Make a note of your customer ID (CCID)
    
2. Download the following files
   * network-firewall-demo.yaml file from the cloudformation folder
   * All files in the s3-bucket folder

3. Create an S3 bucket in the region where you will be deploying the template.
   The bucket files will be accessed by a lambda function that is created by the template. No other access is required.
    
   ![](images/create-bucket.png)

4. Upload the files from the s3-bucket folder to the new bucket you created in the previous step.

   _The contents of this folder may change over time.  The screenshot is not a definitive list of files_
    
   ![](images/bucket-upload.png)

5. Load the CloudFormation Template 
   
   Add the required Parameters

   | Parameters | Description | Default | User Input Required |
   | :--- | :--- | :---- | :----: | 
   | CCID    | CrowdStrike Customer ID    |  | *Yes* |
   | FalconClientId    | Falcon OAuth2 Client ID.    |    | *Yes*   |  
   | FalconSecret    | Falcon Oath2 API secret.    |     | *Yes*    | 
   | FWConfigBucket    |  S3 Bucket containing firewall policy config files    |     | *Yes* |
   | Owner    | Owner/Creator of resource    |     |  *Yes* |
   | Reason    | Reason for Deployment eg Testing    |     |  *Yes* |
   | AvailabilityZones    | Availability Zone to use for the subnets in the VPC |     | *Yes* | 
   | KeyPairName    | Public/private key pairs allow you to securely connect to your instance after it launches | |*Yes* |
   | DomainRGName | Domain Rule Group Name | Default: CRWD-Demo-Domain-RG | No |
   | StatefulRGName | Stateful Rule Group Name | CRWD-Demo-Stateful-RG | No |
   | PolicyName | Firewall Policy Name | CRWD-Demo-Firewall-Policy | No |
   | FirewallName | Firewall Name | CRWD-Demo-Firewall| No |
   | StatelessRGName    | Stateless Rule Group Name    | CRWD-Demo-Domain-RG    | No |
   | DomainRGName    | Domain Rule Group Name    | CRWD-Demo-Domain-RG    | No    | 
   | StatefulRGName    | Stateful Rule Group Name    | CRWD-Demo-Stateful-RG    | No    | 
   | PolicyName    | Firewall Policy Name    | CRWD-Demo-Firewall-Policy    | No    | 
   | FirewallName    | Firewall Name.    | CRWD-Demo-Firewall    | No    | 
   | FirewallDescription    | Crowdstrike demo firewall name.    | Crowdstrike demo firewall    | No  | 
   | PrivateSubnetCIDR    | CIDR block parameter must be in the form x.x.x.x/16-28    | 10.0.1.32/28     | No |
   | PublicSubnetCIDR    | CIDR Block for the public DMZ subnet for secure administrative entry       | 10.0.1.0/28     |No|
   | trustedSource    | CIDR from which access to bastion is to be permitted    | 0.0.0.0/0    | No |
   | VPCCIDR    | CIDR Block for the VPC    | 10.0.1.0/24 | No|
   | LatestLinuxAMI    | /aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2 | | No |
   | LatestWindowsAMI    | /aws/service/ami-windows-latest/Windows_Server-2016-English-Full-Base | | No |
   | WindowsInstanceType    | AllowedValues: t2.small t2.micro | t2.small    | No |
   | LinuxInstanceType    | AllowedValues: t2.small t2.micro | t2.small    | No |
   | FigFileName  | The FIG is the process that integrates that Falcon Console with Security Hub |  fig-2.0.8-install.run | No |
    
    
6. The template will take 15-20 minutes to fully deploy (Please be patient)  

## Verify Deployment  

After 15-20 mins the stack deployment should be complete
 
![](images/cft-status.png)

1. Verify that you have a Firewall in your newly created VPC.  Verify the firewall rule groups associated with the firewall policy.
![](images/fw-status.png)

2. Verify that you have a route entry in the route table associated with the private subnet that has a next hop of the firewall vpce.
![](images/private-rt.png)

3. Verify that you have a route entry in the route table with and edge association of the internet gateway. 
![](images/igw-rt.png)

2. Verify that you have a route entry in the route table associated with the public subnet that has a next hop of the firewall vpce.
![](images/public-rt.png)



## Running a Demo

1. Find the Windows instance that has been deployed

   ![](images/connect.png)

   Download the remote desktop file.  The RDP file contains the connection details for the host.  You will need to 
   decrypt the password using the "Get password" link. 

2. Decrypt the password 
    
   ![](images/decrypt-pwd.png)

3. Connect to the windows instance and verify that the CrowdStrike agent is installed.
    
   Run the command *'sc query csagent'*
   
   ![](images/cs-status.png)
    
4. Verify from the Windows hostname that the agent is connected to the console and that a policy is applied.

   ![](images/hostname.png)
    
   Check the falcon console  Got the the Hosts console and search for the hostname shown in the console output.
   
   ![](images/falcon-host.png)
   
5. Open a browser and try the connect to http://adobeincorp.com

   The connection should fail but it will be sufficient to generate a detection in the console.
    
   ![](images/generate-detection.png)
    
    

6. Verify the detection in the CrowdStrike console
     
   Observe the "Triggering Indicator" and "command line" fields in the detection providing information about how the 
   detection was triggered. 
     
   ![](images/detection.png)
    
    
    
7. Check the Security Hub console

   ![](images/security-hub.png)
    
   Search the security hub console for a finding related to the detection.  *(It may take up to 10 minutes for the 
   detection to appear in security hub as a finding")*
    
   Search by _**"Company name: is CrowdStrike"**_
    
   ![](images/sec-hub-search.png)
    
   Select the finding of interest
    
   ![](images/sec-hub-findings.png)
    
    
    
8. Select the finding 
    
   ![](images/sec-hub-actions.png)
    
   Select the action _*"CRWD-Domain-To-FW"*_
    
   This action will trigger a lambda function which will add the domain to the firewall domain rule group.

9. Goto the Network Firewall Rule Group settings in the AWS console

   ![](images/net-fw-rg.png)
    
   Verify that the domain has been added to the rule group
    
   ![](images/domain-rg-policy.png) 
