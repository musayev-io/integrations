---
title: AWS Integrations
---

# AWS Integrations

| Integration | Description                          | Links   |
| ----------- | ------------------------------------ | ------- |
| AWS Systems Manager Distributor | Automatically install and uninstall the CrowdStrike Falcon sensor into AWS EC2 instances. This integration utilizes CloudFormation scripts executed via AWS Lambda functions, which are executed whenever AWS Systems Manager Distributor detects EC2 instance creation or termination. | Code on GitHub |
| AWS Security Hub | Publishes CrowdStrike detections to AWS Security Hub. | Code on GitHub |
| [AWS Network Firewall with CrowdStrike Threat Intelligence](aws-network-firewall/cs-threat-intel.md) | Add FQDN's from CrowdStrike detections to a domain block list in AWS Network Firewall. | Code on GitHub |
| AWS PrivateLink | Utilize AWS PrivateLink to provide secure connectivity between your CrowdStrike protected workloads/endpoints and the CrowdStrike Cloud. | Code on GitHub |
| AWS S3 as Recipient of Falcon Data Replicator | Replicate event data from your CrowdStrike environment to an AWS S3 bucket. | Code on GitHub |
