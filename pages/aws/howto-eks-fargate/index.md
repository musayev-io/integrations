---
title: How to Deploy CrowdStrike to AWS EKS with AWS Fargate, <i>The Hard Way</i>
layout: page
permalink: /aws/how-to-deploy-crowdstrike-to-aws-eks-with-aws-fargate-the-hard-way/

sidenav: aws-integrations-menu
subnav:
  - text: About the Integration
    href: '#about-the-integration'
  - text: How to Deploy
    href: '#how-to-deploy'
  - text: Running a Demo
    href: '#running-a-demo'
---
<div
  class="usa-summary-box"
  role="region"
  aria-labelledby="summary-box-key-information">
  <div class="usa-summary-box__body">
    <h3 class="usa-summary-box__heading" id="summary-box-key-information">
      Goals of this How To
    </h3>
    <div class="usa-summary-box__text">
      <ol class="usa-list">
        <li>Using the AWS CLI, instantiate <a class="usa-summary-box__link" href="https://aws.amazon.com/eks/">Amazon Elastic Kubernetes Service (Amazon EKS)</a> to manage a kubernetes cluster (your control plane).</li><br/>
        <li>Using the AWS CLI, instantiate <a class="usa-summary-box__link" href="https://aws.amazon.com/fargate/">AWS Fargate</a> to provision and run kubernetes pod infrastructure (your data plane).</li><br/>
        <li>Integrate CrowdStrike with the <a class="usa-summary-box__link" href="https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/">Admission Controller of your Kubernetes cluster</a>. This integration injects the CrowdStrike Container Sensor into any new pod deployment on the cluster (your security plane).</li><br/>
        <li>Deploy CrowdStrike's Detection Container (<a class="usa-summary-box__link" href="https://github.com/CrowdStrike/detection-container">GitHub</a>, <a class="usa-summary-box__link" href="https://quay.io/repository/crowdstrike/detection-container">Quay.io</a>), a container image that will generate sample detections to ensure everything is wired correctly.</li>
      </ol>
      <p><b>Estimated time to complete:</b> 60 to 90 minutes.</p>
    </div>
  </div>
</div>

# Overview

## About Amazon EKS and AWS Fargate
Amazon Elastic Kubernetes Service (Amazon EKS) is Amazon managed Kubernetes service with an Amazon provided Kubernetes Control Plane (API Server) and Amazon Web Console. EKS clusters can be configured to run workloads either on EKS nodes or on AWS Fargate.

With Fargate the worker nodes of the Kubernetes cluster are managed by Amazon. No operational focus needs to be devoted to the infrastructure below the Kubernetes interface when Amazon EKS is deployed with AWS Fargate.

In this deployment pattern the operations boundaries are clearly drawn at the Kubernetes interface. There is no ability to install software on the kubernetes worker nodes, so therefor, CrowdStrike must be integrated into the native Admission Controller of the Kubernetes cluster.

## About CrowdStrike's Falcon Container Sensor
The Falcon Container Sensor extends runtime security to container workloads. The Falcon Container Sensor runs as an unprivileged container in user space with no code running in the kernel of the worker node operating system. This allows CrowdStrike to secure Kubernetes pods in clusters where access to the underlying operating system is not possible (e.g. Amazon EKS) and where privileged containers are often disallowed.

<div class="usa-alert usa-alert--info usa-alert--slim">
  <div class="usa-alert__body">
    <p class="usa-alert__text">
      In Kubernetes clusters where kernel module loading is supported by the worker node operating system, we recommend using CrowdStrike's Falcon Sensor for Linux to secure the underlying worker nodes, the kubernetes cluster, and containers, with a 
      single sensor.
    </p>
  </div>
</div>

## Prerequisites / Required Software and Credentials
<ul>
  <li><b>Docker installed and running locally.</b><br/>
  To ensure required tooling is consistently installed, this guide utilizes CrowdStrike's Cloud Tools Image (<a href="https://github.com/CrowdStrike/cloud-tools-image">GitHub</a>, <a href="https://quay.io/repository/crowdstrike/cloud-tools-image">Quay.io</a>).
  This image includes tools for communication with public and private cloud providers, such as <a href="https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html">eksctl</a> and <a href="https://kubernetes.io/docs/tasks/tools/">kubectl</a>.</li><br/>

  <li><b>CrowdStrike API Credentials with Download Permission</b><br/>
  These credentials can be created in the CrowdStrike platform under "Support" --> "API Clients and Keys," or
  directly via <a href="https://falcon.crowdstrike.com/support/api-clients-and-keys">https://falcon.crowdstrike.com/support/api-clients-and-keys</a>.<br/></li>
</ul>

# Deployment and Configuration

### Step 1: Download the CrowdStrike Cloud Tools Image
<div class="usa-step-indicator usa-step-indicator--counters-sm" aria-label="progress">
  <ol class="usa-step-indicator__segments">
    <li class="usa-step-indicator__segment usa-step-indicator__segment--current" aria-current="true"><span class="usa-step-indicator__segment-label">CrowdStrike Cloud Tools Image <span class="usa-sr-only">not completed</span></span></li>
    <li class="usa-step-indicator__segment"><span class="usa-step-indicator__segment-label">Create Cluster <span class="usa-sr-only">not completed</span></span></li>
    <li class="usa-step-indicator__segment"><span class="usa-step-indicator__segment-label">Deploy Falcon Sensor <span class="usa-sr-only">not completed</span></span></li>
    <li class="usa-step-indicator__segment"><span class="usa-step-indicator__segment-label">Kubernetes Admission Controller <span class="usa-sr-only">not completed</span></span></li>
    <li class="usa-step-indicator__segment"><span class="usa-step-indicator__segment-label">Deploy Detection Container <span class="usa-sr-only">not completed</span></span></li>
    <li class="usa-step-indicator__segment"><span class="usa-step-indicator__segment-label">Experiment <span class="usa-sr-only">not completed</span></span></li>
    <li class="usa-step-indicator__segment"><span class="usa-step-indicator__segment-label">Deprovision <span class="usa-sr-only">not completed</span></span></li>    
  </ol>
</div>

If you have previously used the AWS CLI tool on your system, you may already have your AWS Credentials stored in ``~/.aws`` directory. If that is the case, you can pass your credentials into the Cloud Tools Image without having to re-enter them:

`````shell
sudo docker run --privileged=true \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -v ~/.aws:/root/.aws:ro -it --rm \
    quay.io/crowdstrike/cloud-tools-image
`````

If you have not previously used the AWS CLI tool, or if you would not like to pass your AWS credentials into the container, use the following command:

`````shell
sudo docker run --privileged=true \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -v ~/.aws:/root/.aws -it --rm \
    quay.io/crowdstrike/cloud-tools-image
`````


### Step 2: Create Amazon EKS with AWS Fargate Cluster
<div class="usa-step-indicator usa-step-indicator--counters-sm" aria-label="progress">
  <ol class="usa-step-indicator__segments">
    <li class="usa-step-indicator__segment usa-step-indicator__segment--complete"><span class="usa-step-indicator__segment-label">CrowdStrike Cloud Tools Image <span class="usa-sr-only">completed</span></span></li>
    <li class="usa-step-indicator__segment usa-step-indicator__segment--current"><span class="usa-step-indicator__segment-label" aria-current="true">Create Cluster <span class="usa-sr-only">not completed</span></span></li>
    <li class="usa-step-indicator__segment"><span class="usa-step-indicator__segment-label">Deploy Falcon Sensor <span class="usa-sr-only">not completed</span></span></li>
    <li class="usa-step-indicator__segment"><span class="usa-step-indicator__segment-label">Kubernetes Admission Controller <span class="usa-sr-only">not completed</span></span></li>
    <li class="usa-step-indicator__segment"><span class="usa-step-indicator__segment-label">Deploy Detection Container <span class="usa-sr-only">not completed</span></span></li>
    <li class="usa-step-indicator__segment"><span class="usa-step-indicator__segment-label">Experiment <span class="usa-sr-only">not completed</span></span></li>
    <li class="usa-step-indicator__segment"><span class="usa-step-indicator__segment-label">Deprovision <span class="usa-sr-only">not completed</span></span></li>    
  </ol>
</div>

### Step 3: Create Container Registry and Push Falcon Sensor Image
<div class="usa-step-indicator usa-step-indicator--counters-sm" aria-label="progress">
  <ol class="usa-step-indicator__segments">
    <li class="usa-step-indicator__segment usa-step-indicator__segment--complete"><span class="usa-step-indicator__segment-label">CrowdStrike Cloud Tools Image <span class="usa-sr-only">completed</span></span></li>
    <li class="usa-step-indicator__segment usa-step-indicator__segment--complete"><span class="usa-step-indicator__segment-label" aria-current="true">Create Cluster <span class="usa-sr-only">not completed</span></span></li>
    <li class="usa-step-indicator__segment usa-step-indicator__segment--current"><span class="usa-step-indicator__segment-label">Deploy Falcon Sensor <span class="usa-sr-only">not completed</span></span></li>
    <li class="usa-step-indicator__segment"><span class="usa-step-indicator__segment-label">Kubernetes Admission Controller <span class="usa-sr-only">not completed</span></span></li>
    <li class="usa-step-indicator__segment"><span class="usa-step-indicator__segment-label">Deploy Detection Container <span class="usa-sr-only">not completed</span></span></li>
    <li class="usa-step-indicator__segment"><span class="usa-step-indicator__segment-label">Experiment <span class="usa-sr-only">not completed</span></span></li>
    <li class="usa-step-indicator__segment"><span class="usa-step-indicator__segment-label">Deprovision <span class="usa-sr-only">not completed</span></span></li>    
  </ol>
</div>

### Step 4: Install the Kubernetes Admission Controller
<div class="usa-step-indicator usa-step-indicator--counters-sm" aria-label="progress">
  <ol class="usa-step-indicator__segments">
    <li class="usa-step-indicator__segment usa-step-indicator__segment--complete"><span class="usa-step-indicator__segment-label">CrowdStrike Cloud Tools Image <span class="usa-sr-only">completed</span></span></li>
    <li class="usa-step-indicator__segment usa-step-indicator__segment--complete"><span class="usa-step-indicator__segment-label" aria-current="true">Create Cluster <span class="usa-sr-only">not completed</span></span></li>
    <li class="usa-step-indicator__segment usa-step-indicator__segment--complete"><span class="usa-step-indicator__segment-label">Deploy Falcon Sensor <span class="usa-sr-only">not completed</span></span></li>
    <li class="usa-step-indicator__segment usa-step-indicator__segment--current"><span class="usa-step-indicator__segment-label">Kubernetes Admission Controller <span class="usa-sr-only">not completed</span></span></li>
    <li class="usa-step-indicator__segment"><span class="usa-step-indicator__segment-label">Deploy Detection Container <span class="usa-sr-only">not completed</span></span></li>
    <li class="usa-step-indicator__segment"><span class="usa-step-indicator__segment-label">Experiment <span class="usa-sr-only">not completed</span></span></li>
    <li class="usa-step-indicator__segment"><span class="usa-step-indicator__segment-label">Deprovision <span class="usa-sr-only">not completed</span></span></li>    
  </ol>
</div>

### Step 5: Deploy the CrowdStrike Detection Container
<div class="usa-step-indicator usa-step-indicator--counters-sm" aria-label="progress">
  <ol class="usa-step-indicator__segments">
    <li class="usa-step-indicator__segment usa-step-indicator__segment--complete"><span class="usa-step-indicator__segment-label">CrowdStrike Cloud Tools Image <span class="usa-sr-only">completed</span></span></li>
    <li class="usa-step-indicator__segment usa-step-indicator__segment--complete"><span class="usa-step-indicator__segment-label" aria-current="true">Create Cluster <span class="usa-sr-only">not completed</span></span></li>
    <li class="usa-step-indicator__segment usa-step-indicator__segment--complete"><span class="usa-step-indicator__segment-label">Deploy Falcon Sensor <span class="usa-sr-only">not completed</span></span></li>
    <li class="usa-step-indicator__segment usa-step-indicator__segment--complete"><span class="usa-step-indicator__segment-label">Kubernetes Admission Controller <span class="usa-sr-only">not completed</span></span></li>
    <li class="usa-step-indicator__segment usa-step-indicator__segment--current"><span class="usa-step-indicator__segment-label">Deploy Detection Container <span class="usa-sr-only">not completed</span></span></li>
    <li class="usa-step-indicator__segment"><span class="usa-step-indicator__segment-label">Experiment <span class="usa-sr-only">not completed</span></span></li>
    <li class="usa-step-indicator__segment"><span class="usa-step-indicator__segment-label">Deprovision <span class="usa-sr-only">not completed</span></span></li>    
  </ol>
</div>

### Step 6: Experiment!
<div class="usa-step-indicator usa-step-indicator--counters-sm" aria-label="progress">
  <ol class="usa-step-indicator__segments">
    <li class="usa-step-indicator__segment usa-step-indicator__segment--complete"><span class="usa-step-indicator__segment-label">CrowdStrike Cloud Tools Image <span class="usa-sr-only">completed</span></span></li>
    <li class="usa-step-indicator__segment usa-step-indicator__segment--complete"><span class="usa-step-indicator__segment-label" aria-current="true">Create Cluster <span class="usa-sr-only">not completed</span></span></li>
    <li class="usa-step-indicator__segment usa-step-indicator__segment--complete"><span class="usa-step-indicator__segment-label">Deploy Falcon Sensor <span class="usa-sr-only">not completed</span></span></li>
    <li class="usa-step-indicator__segment usa-step-indicator__segment--complete"><span class="usa-step-indicator__segment-label">Kubernetes Admission Controller <span class="usa-sr-only">not completed</span></span></li>
    <li class="usa-step-indicator__segment usa-step-indicator__segment--complete"><span class="usa-step-indicator__segment-label">Deploy Detection Container <span class="usa-sr-only">not completed</span></span></li>
    <li class="usa-step-indicator__segment usa-step-indicator__segment--current"><span class="usa-step-indicator__segment-label">Experiment <span class="usa-sr-only">not completed</span></span></li>
    <li class="usa-step-indicator__segment"><span class="usa-step-indicator__segment-label">Deprovision <span class="usa-sr-only">not completed</span></span></li>    
  </ol>
</div>

### Step 7: Deprovision your Testing Environment 
<div class="usa-step-indicator usa-step-indicator--counters-sm" aria-label="progress">
  <ol class="usa-step-indicator__segments">
    <li class="usa-step-indicator__segment usa-step-indicator__segment--complete"><span class="usa-step-indicator__segment-label">CrowdStrike Cloud Tools Image <span class="usa-sr-only">completed</span></span></li>
    <li class="usa-step-indicator__segment usa-step-indicator__segment--complete"><span class="usa-step-indicator__segment-label" aria-current="true">Create Cluster <span class="usa-sr-only">not completed</span></span></li>
    <li class="usa-step-indicator__segment usa-step-indicator__segment--complete"><span class="usa-step-indicator__segment-label">Deploy Falcon Sensor <span class="usa-sr-only">not completed</span></span></li>
    <li class="usa-step-indicator__segment usa-step-indicator__segment--complete"><span class="usa-step-indicator__segment-label">Kubernetes Admission Controller <span class="usa-sr-only">not completed</span></span></li>
    <li class="usa-step-indicator__segment usa-step-indicator__segment--complete"><span class="usa-step-indicator__segment-label">Deploy Detection Container <span class="usa-sr-only">not completed</span></span></li>
    <li class="usa-step-indicator__segment usa-step-indicator__segment--complete"><span class="usa-step-indicator__segment-label">Experiment <span class="usa-sr-only">not completed</span></span></li>
    <li class="usa-step-indicator__segment usa-step-indicator__segment--current"><span class="usa-step-indicator__segment-label">Deprovision <span class="usa-sr-only">not completed</span></span></li>    
  </ol>
</div>
