---
title: AWS/CrowdStrike Integrations
layout: page
permalink: /aws/

sidenav: aws-integrations-menu
subnav:
  - text: Prerequisites
    href: '#prerequisites'
  - text: Demo Description
    href: '#demo-description'
  - text: Deployment Steps
    href: '#deployment-steps'
  - text: Verify Deployment
    href: '#verify-deployment'
---

<!-- REMEMBER TO KEEP THESE IN ALPHABETICAL ORDER!! -->
<table class="usa-table usa-table--borderless usa-table--striped">
  <thead>
    <tr>
      <th scope="col">Integration</th>
      <th scope="col">Description</th>
      <th scope="col">Links</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th scope="row"><a href="aws-systems-manager-distributor/">AWS Systems Manager Distributor</a></th>
      <td>
        Automatically install and uninstall the CrowdStrike Falcon sensor into AWS EC2 instances.
        This integration utilizes CloudFormation scripts executed via AWS Lambda functions,
        which are executed whenever AWS Systems Manager Distributor detects EC2 instance
        creation or termination.
      </td>
      <td><a href="https://github.com/CrowdStrike/Cloud-AWS/tree/master/systems-manager">Code on GitHub</a></td>
    </tr>
    <tr>
      <th scope="row"><a href="aws-security-hub/">AWS Security Hub</a></th>
      <td>
        Publishes CrowdStrike detections to AWS Security Hub.
      </td>
      <td><a href="https://github.com/CrowdStrike/Cloud-AWS/tree/master/Security-Hub">Code on GitHub</a></td>
    </tr>
    <tr>
      <th scope="row"><a href="aws-network-firewall-with-crowdstrike-threat-intelligence/">AWS Network Firewall with CrowdStrike Threat Intelligence</a></th>
      <td>
        Add FQDN's from CrowdStrike detections to a domain block list in AWS Network Firewall.
      </td>
      <td><a href="https://crowdstrike.github.io/aws-network-firewall">Code on GitHub</a></td>
    </tr>
    <tr>
      <th scope="row"><a href="aws-private-link/">AWS PrivateLink</a></th>
      <td>
        Utilize AWS PrivateLink to provide secure connectivity between your CrowdStrike
        protected workloads/endpoints and the CrowdStrike Cloud.
      </td>
      <td><a href="https://github.com/CrowdStrike/Cloud-AWS/tree/master/aws-privatelink">Code on GitHub</a></td>
    </tr>
    <tr>
      <th scope="row"><a href="aws-private-link/">AWS S3 as Recipient of Falcon Data Replicator</a></th>
      <td>
        Replicate event data from your CrowdStrike environment to an AWS S3 bucket.
      </td>
      <td><a href="https://github.com/CrowdStrike/FDR">Code on GitHub</a></td>
    </tr>
  </tbody>
</table>