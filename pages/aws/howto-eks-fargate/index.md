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

  <li><b>Prefered AWS Region</b><br/>
  Where should the cluster be deployed? We'll need the AWS Region ID, such as <tt>us-west-1</tt> or <tt>us-east-2</tt>.<br/></li>
</ul>

MISSING
* CrowdStrike Customer ID / CID

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

*If you have previously used the AWS CLI tool on your system*, you may already have your AWS Credentials stored in ``~/.aws`` directory. You can optionally pass your AWS credentials into the Cloud Tools Image without having to re-enter them. This is done by sharing your ``~/.aws/`` directory as a read-only filesystem:

`````shell
sudo docker run --privileged=true \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -v ~/.aws:/root/.aws:ro -it --rm \
    quay.io/crowdstrike/cloud-tools-image
`````

*If you have not previously used the AWS CLI tool*, or if you would *not* like to pass your AWS credentials into the container, use the following command:

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

1. **Set your preferred cloud region as an environmental variable.**<br/>
This will help ensure you do not need to repetitively type it. Replace ``us-west-1`` with your desired region.
`````shell
CLOUD_REGION=us-west-1
`````
2. **Create a new EKS Fargate cluster.**<br/>
This may take a few minutes while AWS spins up your cluster.
`````shell
eksctl create cluster \
  --name eks-fargate-cluster --region $CLOUD_REGION \
  --fargate
`````
Example Output:
`````shell
[ℹ]  eksctl version 0.37.0
[ℹ]  using region eu-west-1
[ℹ]  setting availability zones to [eu-west-1b eu-west-1c eu-west-1a]
[ℹ]  subnets for eu-west-1b - public:192.168.0.0/19 private:192.168.96.0/19
[ℹ]  subnets for eu-west-1c - public:192.168.32.0/19 private:192.168.128.0/19
[ℹ]  subnets for eu-west-1a - public:192.168.64.0/19 private:192.168.160.0/19
[ℹ]  using Kubernetes version 1.18
[ℹ]  creating EKS cluster "eks-fargate-cluster" in "eu-west-1" region with Fargate profile
[ℹ]  if you encounter any issues, check CloudFormation console or try 'eksctl utils describe-stacks --region=eu-west-1 --cluster=eks-fargate-cluster'
[ℹ]  CloudWatch logging will not be enabled for cluster "eks-fargate-cluster" in "eu-west-1"
[ℹ]  you can enable it with 'eksctl utils update-cluster-logging --enable-types={SPECIFY-YOUR-LOG-TYPES-HERE (e.g. all)} --region=eu-west-1 --cluster=eks-fargate-cluster'
[ℹ]  Kubernetes API endpoint access will use default of {publicAccess=true, privateAccess=false} for cluster "eks-fargate-cluster" in "eu-west-1"
[ℹ]  2 sequential tasks: { create cluster control plane "eks-fargate-cluster", 2 sequential sub-tasks: { 2 sequential sub-tasks: { wait for control plane to become ready, create fargate profiles }, create addons } }
[ℹ]  building cluster stack "eksctl-eks-fargate-cluster-cluster"
[ℹ]  deploying stack "eksctl-eks-fargate-cluster-cluster"
[ℹ]  waiting for CloudFormation stack "eksctl-eks-fargate-cluster-cluster"
[ℹ]  waiting for CloudFormation stack "eksctl-eks-fargate-cluster-cluster"
[ℹ]  waiting for CloudFormation stack "eksctl-eks-fargate-cluster-cluster"
[ℹ]  creating Fargate profile "fp-default" on EKS cluster "eks-fargate-cluster"
[ℹ]  created Fargate profile "fp-default" on EKS cluster "eks-fargate-cluster"
[ℹ]  "coredns" is now schedulable onto Fargate
[ℹ]  "coredns" is now scheduled onto Fargate
[ℹ]  "coredns" pods are now scheduled onto Fargate
[ℹ]  waiting for the control plane availability...
[✔]  saved kubeconfig as "/root/.kube/config"
[ℹ]  no tasks
[✔]  all EKS cluster resources for "eks-fargate-cluster" have been created
[ℹ]  kubectl command should work with "/root/.kube/config", try 'kubectl get nodes'
[✔]  EKS cluster "eks-fargate-cluster" in "eu-west-1" region is ready
`````
3. **Configure your cluster to instantiate Falcon Admission Controller to Fargate.**<br/>
EKS clusters automatically decide which workloads shall be instantiated on Fargate vs EKS nodes. This decision process is configured by an AWS entity called *Fargate Profile*. Fargate profiles assign workloads based on Kubernetes namespaces. By default, ``kube-system`` and ``default`` namespaces are present on the cluster.<br/><br/>
CrowdStrike's Falcon Admission Controller will be deployed later in this guide to a namespace called ``falcon-system``. The following command configures a cluster to instantiate the Admission Controller on Fargate:
`````shell
eksctl create fargateprofile \
    --region $CLOUD_REGION \
    --cluster eks-fargate-cluster \
    --name fp-falcon-sytem \
    --namespace falcon-system
`````
Example Output:
`````shell
[ℹ]  creating Fargate profile "fp-falcon-sytem" on EKS cluster "eks-fargate-cluster"
[ℹ]  created Fargate profile "fp-falcon-sytem" on EKS cluster "eks-fargate-cluster"
`````
4. <b>Verify your ``kubectl`` utility has been configured to connect to the cluster.</b><br/>
``kubectl`` is a command line tool that lets you control Kubernetes clusters. For configuration, ``kubectl`` looks for a file named ``config`` in the ``$HOME/.kube`` directory. This configuration file was created previously by the ``eksctl create cluster`` command and contains login information for your newly created cluster.
`````shell
kubectl cluster-info
`````

Example Output:
`````shell
Kubernetes control plane is running at https://EEAB38XXXXXXXXXXXXXXXXXXXXXXXXXX.sk1.eu-west-1.eks.amazonaws.com
CoreDNS is running at https://EEAB38XXXXXXXXXXXXXXXXXXXXXXXXXX.sk1.eu-west-1.eks.amazonaws.com/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
`````


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

1. <b>Create Container Repository in AWS Elastic Container Registry (AWS ECR)</b><br/>
AWS Elastic Container Registry is a cloud serice providing a container registry. The command
below creates a new repository and this repository will be subsequently used to store the Falcon
Container Sensor image.
`````shell
aws ecr create-repository --region $CLOUD_REGION --repository-name falcon-sensor
`````
<br/>
Example Output:
`````shell
{
    "repository": {
        "repositoryArn": "arn:aws:ecr:eu-west-1:123456789123:repository/falcon-sensor",
        "registryId": "12345678912345",
        "repositoryName": "falcon-sensor",
        "repositoryUri": "123456789123.dkr.ecr.eu-west-1.amazonaws.com/falcon-sensor",
        "createdAt": "2021-02-04T10:30:30+00:00",
        "imageTagMutability": "MUTABLE",
        "imageScanningConfiguration": {
            "scanOnPush": false
        },
        "encryptionConfiguration": {
            "encryptionType": "AES256"
        }
    }
}
`````
Note the ``repositoryUri`` of the newly created repository. This will be used in the next step!<br/><br/>
2. <b>Set the FALCON_IMAGE_URI Variable</b><br/>
Set the ``FALCON_IMAGE_URL`` variable to your managed ECR based on the ECR ``repositoryUri`` from above.
`````shell
FALCON_IMAGE_URI=$(aws ecr describe-repositories --region $CLOUD_REGION | jq -r '.repositories[] | select(.repositoryName=="falcon-sensor") | .repositoryUri')
`````
3. <b>Push the Falcon Container Image</b>
`````shell
falcon-container-sensor-push $FALCON_IMAGE_URI
`````

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
Kubernetes Admission Controllers are a service that intercepts requests to the Kubernetes API Server. CrowdStrike's
Container Sensor hooks into this service and injects the Falcon Container Sensor to any new pod deployment on the cluster. In this step we will configure and deploy the admission hook and the admission application.

1. <b>Provide CrowdStrike Falcon Customer ID (CID) as Environment Variable</b><br/>
The CID will later be used to register newly deployed pods to CrowdStrike.
`````shell
CID=1234567890ABCDEFG1234567890ABCDEF-12
`````
2. <b>Install the Admission Controller</b><br/>
`````shell
docker run --rm --entrypoint installer $FALCON_IMAGE_URI:latest \
    -cid $CID -image $FALCON_IMAGE_URI:latest \
    | kubectl apply -f -
`````
Example Output:
`````shell
namespace/falcon-system created
configmap/injector-config created
secret/injector-tls created
deployment.apps/injector created
service/injector created
mutatingwebhookconfiguration.admissionregistration.k8s.io/injector.falcon-system.svc created
`````
3. <b>(OPTIONAL) Watch Deployment Progress</b><br/>
`````shell
watch 'kubectl get pods -n falcon-system'
`````
Example Output:
`````shell
NAME                        READY   STATUS    RESTARTS   AGE
injector-6499dbd4b5-v5gqr   1/1     Running   0          2d3h
`````
4. <b>(OPTIONAL) Run Installer with --help</b><br/>
The ``--help`` command-line argument displays available configuration options for the deployment:
`````shell
docker run --rm --entrypoint installer $FALCON_IMAGE_URI:latest --help
`````
Example Output:
`````shell
usage:
  -cid string
    	Customer id to use
  -days int
    	Validity of certificate in days. (default 3650)
  -falconctl-env value
    	FALCONCTL options in key=value format.
  -image string
    	Image URI to load (default "crowdstrike/falcon")
  -mount-docker-socket
    	A boolean flag to mount docker socket of worker node with sensor.
  -namespaces string
    	Comma separated namespaces with which image pull secret need to be created, applicable only with -pullsecret (default "default")
  -pullpolicy string
    	Pull policy to be defined for sensor image pulls (default "IfNotPresent")
  -pullsecret string
    	Secret name that is used to pull image (default "crowdstrike-falcon-pull-secret")
  -pulltoken string
    	Secret token, stringified dockerconfig json or base64 encoded dockerconfig json, that is used with pulling image
  -sensor-resources string
    	A valid json string or base64 encoded string of the same, which is used as k8s resources specification.
`````
Full explanation of various configuration options and deployment scenarios is available through the
<a href="https://falcon.crowdstrike.com/support/documentation/146/falcon-container-sensor-for-linux#additional-installation-options">Falcon Console</a>.

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
