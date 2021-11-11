# AWS OpenShift 4.x UPI Install via Cloudformation - DRAFT

## 1. Overview 
The steps in this guide for preforming a UPI-based installation are outlined here: [Official Installing of OpenShift4 on AWS](https://docs.openshift.com/container-platform/4.9/installing/installing_aws/installing-aws-user-infra.html).
The work here is based on GitHub user Roberto Carratalá's project: https://github.com/rcarrata/ocp4-aws-upi-cf  
Several Cloudformation templates and parameters files are provided to assist in completing these steps. The CloudFormation templates are just an example to help model your own.

I am working with OpenShift version 4.9 in AWS. The CloudFormation template YAMLs are based on that version.  To deploy a different version of OpenShift copy the template from the CloudFormation template from the desired version and save it as a YAML file on your computer. 

* [OpenShift 4.8 for AWS](https://docs.openshift.com/container-platform/4.8/installing/installing_aws/installing-aws-user-infra.html)
* [OpenShift 4.7 for AWS](https://docs.openshift.com/container-platform/4.7/installing/installing_aws/installing-aws-user-infra.html)
* [OpenShift 4.6 for AWS](https://docs.openshift.com/container-platform/4.6/installing/installing_aws/installing-aws-user-infra.html)

**Assumptions**  
These steps were tested on both WSL Ubuntu 20.04 and a Ubuntu 18.04 VM.  
You will need to have the AWS cli tool installed and the AWS credentials updated.  
There is a base domain created on AWS Route53 to use: example.com  
I am using <FirstInitialLastname>-<ObjectName> as the naming convention for this project:   
    example: chernandez-vpc 

  ---
  **TIP**
  You can download the README.md and do a 'find and replace' to update the commands:
  * Replace cherandez with your name 
  * Replace c.hernandez@example.com with your email
  ---



## 2. Retrieve the OpenShift Install and Generate the Install files

### 2.1 Download openshift-install binary
```
openshift_version=$(curl -s https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/ | grep -E "openshift-install-linux-.*.tar.gz" | sed -r 's/.*href="([^"]+).*/\1/g')
wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/$openshift_version 
or curl  https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/$openshift_version --output $openshift_version
sudo tar -xvzf $openshift_version -C /usr/local/bin/
```

### 2.2 Download oc binary
```
openshift_cli=$(curl -s https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/ | grep -E "openshift-client-linux-.*.tar.gz" | sed -r 's/.*href="([^"]+).*/\1/g')
wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/$openshift_cli
or 
curl  https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/$openshift_cli --output $openshift_cli
sudo tar -xvzf $openshift_cli -C /usr/local/bin/
```



## 3. Creating the installation files for AWS

```
# mkdir upi_ocp4_aws
# cd upi_ocp4_aws/
```
Download the Cloudformation templates and parameters files from the Git repository to this directory.

**Generate the install-config.yaml file:**
  1. Select your RSA public key
  2. Select AWS as the platform
  3. Select your AWS region
  4. Enter the base domain: **example.com**
  5. Enter a name for your cluster
  6. Paste in the pull secret obtained from Red Hat

```
# openshift-install create install-config --dir=.
? SSH Public Key ~/.ssh/id_rsa.pub
? Platform aws
? Region us-west-2
? Base Domain example.com
? Cluster Name chernandez-ocp4
? Pull Secret [? for help] *************************************************************************************************************************************************************************
```

**Set Worker Replicas to 0**

We'll be providing the compute machines ourselves, so edit the resulting install-config.yaml to set replicas to 0 for the compute pool:

```
# sed -i '3,/replicas: / s/replicas: .*/replicas: 0/' install-config.yaml
```

**Add CredentialsMode and Owner Tags**

You can modify the to add tags to identify you as the Contact and Owner. 
The install-config.yml will have these properties:

```
# cat install-config.yaml
apiVersion: v1
baseDomain: example.com
compute:
- hyperthreading: Enabled
  name: worker
  platform: {}
  replicas: 0
controlPlane:
  hyperthreading: Enabled
  name: master
  platform: {}
  replicas: 3
metadata:
  creationTimestamp: null
  name: chernandez-ocp4
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  machineCIDR: 10.0.0.0/16
  networkType: OpenShiftSDN
  serviceNetwork:
  - 172.30.0.0/16
platform:
  aws:
    region: us-west-2
    userTags:
      Contact: Carlos Hernandez
      Owner: c.hernandez@example.com    
pullSecret: <<Pull-Secret>>
sshKey: |
  ssh-rsa <<RSA>>
```

**Backup the install-config.yaml for future purposes:**

```
# cp -pr install-config.yaml install-config.yaml.bk
```

**Generate the Kubernetes manifests for the cluster:**

```
# openshift-install create manifests --dir=.
INFO Credentials loaded from the "default" profile in file "~/.aws/credentials"
INFO Consuming Install Config from target directory
WARNING Making control-plane schedulable by setting MastersSchedulable to true for Scheduler cluster settings
INFO Manifests created in: manifests and openshift
```

**Remove the files that define the control plane machines and remove the Kubernetes manifest files that define the worker machines:**

```
# rm -f openshift/99_openshift-cluster-api_master-machines-*.yaml
# rm -f openshift/99_openshift-cluster-api_worker-machineset-*
# ll openshift/
total 32
drwxr-x--- 2 root root 4096 Nov  8 21:39 ./
drwxr-xr-x 5 root root 4096 Nov  8 21:33 ../
-rw-r----- 1 root root  181 Nov  8 21:33 99_kubeadmin-password-secret.yaml
-rw-r----- 1 root root 2470 Nov  8 21:33 99_openshift-cluster-api_master-user-data-secret.yaml
-rw-r----- 1 root root 2470 Nov  8 21:33 99_openshift-cluster-api_worker-user-data-secret.yaml
-rw-r----- 1 root root  529 Nov  8 21:33 99_openshift-machineconfig_99-master-ssh.yaml
-rw-r----- 1 root root  529 Nov  8 21:33 99_openshift-machineconfig_99-worker-ssh.yaml
-rw-r----- 1 root root  217 Nov  8 21:33 99_role-cloud-creds-secret-reader.yaml
-rw-r----- 1 root root  173 Nov  8 21:33 openshift-install-manifests.yaml
```

**Unschedule Control Plane**

Check that the **mastersSchedulable** parameter in the <installation_directory>/manifests/cluster-scheduler-02-config.yml Kubernetes manifest file is set to **false**. This setting prevents pods from being scheduled on the control plane machines:

  1. Open the <installation_directory>/manifests/cluster-scheduler-02-config.yml file.
  2. Locate the **mastersSchedulable** parameter and ensure that it is set to **false**.
  3. Save and exit the file.

**Obtain the Ignition config files:**

```
# openshift-install create ignition-configs --dir=.
INFO Consuming Common Manifests from target directory
INFO Consuming Openshift Manifests from target directory
INFO Consuming OpenShift Install (Manifests) from target directory
INFO Consuming Worker Machines from target directory
INFO Consuming Master Machines from target directory
INFO Ignition-Configs created in: . and auth
```

## 4. Upload the ignition files to the s3 bucket

**Create the s3 bucket for the ignition files:**

```
# aws s3 mb s3://chernandez-ocp4
make_bucket: chernandez-ocp4

# aws s3 ls
2020-07-01 01:26:01 cf-templates-xxxxx-us-east-1
2020-06-15 14:12:20 cf-templates-xxxxx-us-west-1
2020-06-25 23:36:05 cf-templates-xxxxx-us-west-2
2021-11-09 09:48:38 chernandez-ocp4
2019-10-05 02:01:46 elasticbeanstalk-us-east-1-xxxxx
```

**Upload the aws s3 bootstrap.ign to the s3 bucket:**

```
# aws s3 cp bootstrap.ign s3://chernandez-ocp4
upload: ./bootstrap.ign to s3://chernandez-ocp4/bootstrap.ign

# aws s3 ls s3://chernandez-ocp4
2021-11-09 09:50:41     279214 bootstrap.ign
```

## 5.Generating Cloudformation Templates
Prepare copy of the CloudFormation JSON template files. 
```
# for i in $(ls *.json.orig); do cp -p $i $(echo $i | sed -e "s/\.json\.orig/\.json/g"); done
```

## 6. Creating the VPC

This step is not necessary to create the VPC and their resources, because the you can install OCP4 into an existing AWS VPC network infrastructure.

```
#  aws cloudformation create-stack --stack-name chernandez-clustervpc --template-body file://01_cluster_vpc.yaml --parameters file://01_cluster_vpc.json --tags Key="Owner",Value="c.hernandez@example.com" Key="Name",Value="chernandez-vpc"

{
    ""StackId": "arn:aws:cloudformation:us-west-2:xxxxx:stack/chernandez-clustervpc/cfe025a0-4290-11ec-a532-xxxxx"
}
```
```
# aws cloudformation wait stack-create-complete --stack-name chernandez-clustervpc
```

## 7. Creating Networking and Load Balancing Components in AWS

### 7.1 Inputs for CF Networking and Load Balancing

* ClusterName

```
# ClusterName="chernandez-ocp4" ; sed -i -e "s/clustername/$ClusterName/g" *.json
```

* InfrastructureName

```
# infrastructurename=$(jq -r .infraID ./metadata.json)
# echo $infrastructurename
chernandez-ocp4-pgmmr
```

```
# sed -i -e "s/infrastructurename/$infrastructurename/g" *.json
```

* HostedZoneName

```
# hostedzonename="example.com"
```

NOTE: Remember to do not include the absolute domain name "with the dot(.) at the end", use the
relative domain name (without the dot(.) at the end"

```
# sed -i -e "s/hostedzonename/$hostedzonename/g" *.json
```

* HostedZoneId

```
# hostedzoneid=$(aws route53 list-hosted-zones | jq -r --arg hostedzonename "$hostedzonename." '.HostedZones[] | select(.Config.PrivateZone==false and .Name==$hostedzonename) | .Id' | cut -d"/" -f3)
```

```
# sed -i -e "s/hostedzoneid/$hostedzoneid/g" *.json
```

* VpcId

```
# vpcid=$(aws ec2 describe-vpcs | jq -r '.Vpcs[] | select(.Tags[].Value=="chernandez-clustervpc")? | .VpcId')

# echo $vpcid
vpc-068d6d598exxxxx
```

```
# sed -i -e "s/vpcid/$vpcid/g" *.json
```

* PrivateSubnets

* NOTE: The filter for identifying the Private and the Public subnets in this case is
  Tags.Key=="kubernetes.io/role/internal-elb because the Subnets are deployed with this tags

```
# privatesubnets=$( aws ec2 describe-subnets --filter="Name=vpc-id,Values=$vpcid" | jq -r '.Subnets[] | select(.Tags[].Value | contains("PrivateSubnet"))  | .SubnetId' | paste -s -d",")

# echo $privatesubnets
subnet-064445b01e6xxxxx,subnet-0ba6da724f2xxxxx,subnet-016378620axxxxx
```

```
# sed -i -e "s/privatesubnets/$privatesubnets/g" *.json
```

* PublicSubnets

```
# publicsubnets=$( aws ec2 describe-subnets --filter="Name=vpc-id,Values=$vpcid" | jq -r '.Subnets[] | select(.Tags[].Value | contains("PublicSubnet"))  | .SubnetId' | paste -s -d",")

# echo $publicsubnets
subnet-0152e8b6467xxxxx,subnet-057a7683cd0xxxxx,subnet-0ad38bf2893xxxxx
```

```
# sed -i -e "s/publicsubnets/$publicsubnets/g" *.json
```

### 7.2 Creating Networking and Load Balancing Components in AWS

```
#  aws cloudformation create-stack --stack-name chernandez-clusterinfra --template-body file://02_cluster_infra.yaml --parameters file://02_cluster_infra.json --capabilities CAPABILITY_NAMED_IAM --tags Key="Owner",Value="c.hernandez@example.com" Key="Name",Value="chernandez-clusterinfra"

{
    "StackId": "arn:aws:cloudformation:us-west-2:xxxxx:stack/chernandez-clusterinfra/2847b811-4292-11ec-b811-xxxxx"
}
```

```
# aws cloudformation wait stack-create-complete --stack-name chernandez-clusterinfra
```

```
# aws cloudformation describe-stacks --stack-name chernandez-clusterinfra | jq -r '.Stacks[].Outputs[]'
```

## 8. Creating security group and roles in AWS

### 8.1 Input Parameters Json Cloudformation Template

* InfrastructureName (already filled)

* VpcCidr

```
# vpccidr=$( aws ec2 describe-vpcs | jq -r --arg vpcid "$vpcid" '.Vpcs[] | select(.VpcId==$vpcid) | .CidrBlock' | sed 's/^\([0-9]*\.[0-9]*.[0-9]*\.[0-9]*\)/\1\\/')
```
```
# sed -i -e "s/vpccidr/$vpccidr/g" *.json
```

* PrivateSubnets (already filled)

* VpcId (already filled)

### 8.2 Executing Cloudformation Template for Networking and Load Balancing

```
# aws cloudformation create-stack --stack-name chernandez-clustersecurity --template-body file://03_cluster_security.yaml --parameters file://03_cluster_security.json --capabilities CAPABILITY_NAMED_IAM --tags Key="Owner",Value="c.hernandez@example.com"

{
    "StackId": "arn:aws:cloudformation:us-west-2:xxxxx:stack/chernandez-clustersecurity/12f5d810-4293-11ec-b1cf-xxxxx"
}
```

```
# aws cloudformation wait stack-create-complete --stack-name chernandez-clustersecurity
```

```
# aws cloudformation describe-stacks --stack-name chernandez-clustersecurity | jq -r '.Stacks[].Outputs[] | .OutputKey +": "+ .OutputValue'
MasterSecurityGroupId: sg-0b4d04bbd05xxxxx
MasterInstanceProfile: chernandez-clustersecurity-MasterInstanceProfile-XXXXX
WorkerSecurityGroupId: sg-0e1c51e1eaxxxxx
WorkerInstanceProfile: chernandez-clustersecurity-WorkerInstanceProfile-XXXXX
```

## 9. Creating the bootstrap node in AWS

### 9.1 Input Parameters for Bootstrap Node
You must use a valid Red Hat Enterprise Linux CoreOS (RHCOS) AMI for your Amazon Web Services (AWS) zone for your OpenShift Container Platform nodes.

```
# openshift-install coreos print-stream-json | jq -r '.architectures.x86_64.images.aws.regions["us-west-2"].image'
ami-09794d8cbc9a5ea5f
```

```
# rhcosami="ami-09794d8cbc9a5ea5f"
```

```
# sed -i -e "s@rhcosami@$rhcosami@g" *.json
```

* AllowedBootstrapSshCidr: by default to "0.0.0.0/0"
* PublicSubnet:

```
# publicsubnet=$( aws ec2 describe-subnets --filter="Name=vpc-id,Values=$vpcid" | jq -r '.Subnets[] | select(.Tags[].Value | contains("PublicSubnet"))?  | .SubnetId' | head -1 )
```

```
# sed -i -e "s@publicsubnet@$publicsubnet@g" *.json
```
* MasterSecurityGroupID:

```
# mastersecuritygroupid=$(aws cloudformation describe-stacks --stack-name chernandez-clustersecurity | jq -r '.Stacks[].Outputs[] | select(.OutputKey=="MasterSecurityGroupId") | .OutputValue')

```

```
# sed -i -e "s/mastersecuritygroupid/$mastersecuritygroupid/g" *.json
```

* VpcId (already filled)

* BootstrapIgnitionLocation

```
# aws s3 ls s3://chernandez-ocp4/bootstrap.ign
2021-11-09 09:50:41     279214 bootstrap.ign
```

```
# bootstrapignitionlocation="s3://chernandez-ocp4/bootstrap.ign"
```

```
# sed -i -e "s@bootstrapignitionlocation@$bootstrapignitionlocation@g" *.json
```

* AutoRegisterELB: Whether or not to register a network load balancer (NLB). By default yes

* RegisterNlbIpTargetsLambdaArn:

```
# registernlbiptargetslambdaarn=$(aws cloudformation describe-stacks --stack-name chernandez-clusterinfra | jq -r '.Stacks[].Outputs[] | select(.OutputKey=="RegisterNlbIpTargetsLambda") | .OutputValue')

# echo $registernlbiptargetslambdaarn
arn:aws:lambda:us-west-2:xxxxx:function:clusterinfra-RegisterNlbIpTargets-XXXXX
```

```
# sed -i -e "s@registernlbiptargetslambdaarn@$registernlbiptargetslambdaarn@g" *.json
```

* ExternalApiTargetGroupArn

```
# externalapitargetgrouparn=$(aws cloudformation describe-stacks --stack-name chernandez-clusterinfra | jq -r '.Stacks[].Outputs[] | select(.OutputKey=="ExternalApiTargetGroupArn") | .OutputValue')

# echo $externalapitargetgrouparn
arn:aws:elasticloadbalancing:us-west-2:xxxxx:targetgroup/clust-Exter-XXXXX/XXXXX
```

```
# sed -i -e "s@externalapitargetgrouparn@$externalapitargetgrouparn@g" *.json
```

* InternalApiTargetGroupArn

```
# internalapitargetgrouparn=$( aws cloudformation describe-stacks --stack-name chernandez-clusterinfra | jq -r '.Stacks[].Outputs[] | select(.OutputKey=="InternalApiTargetGroupArn") | .OutputValue')
```

```
# sed -i -e "s@internalapitargetgrouparn@$internalapitargetgrouparn@g" *.json
```

* InternalServiceTargetGroupArn

```
# internalservicetargetgrouparn=$(aws cloudformation describe-stacks --stack-name chernandez-clusterinfra | jq -r '.Stacks[].Outputs[] | select(.OutputKey=="InternalServiceTargetGroupArn") | .OutputValue')
```

```
# sed -i -e "s@internalservicetargetgrouparn@$internalservicetargetgrouparn@g" *.json
```

### 9.2 Executing Bootstrap Cloudformation Template

```
# aws cloudformation create-stack --stack-name chernandez-clusterbootstrap --template-body file://04_cluster_bootstrap.yaml --parameters file://04_cluster_bootstrap.json --capabilities CAPABILITY_NAMED_IAM --tags Key="Owner",Value="c.hernandez@example.com"

{
    "StackId": "arn:aws:cloudformation:us-west-2:xxxxx:stack/chernandez-clusterbootstrap/6ba53df0-9293-11e9-965a-xxxxx"
}
```

```
# aws cloudformation wait stack-create-complete --stack-name chernandez-clusterbootstrap
```

## 10. Creating the control plane machines in AWS

### 10.1 Input Variables for Cloudformation Template

* InfrastructureName: (already filled)
* RhcosAmi: (already filled)
* AutoRegisterDNS: (By default yes)

* PrivateHostedZoneId

```
#  privatehostedzoneid=$( aws cloudformation describe-stacks --stack-name chernandez-clusterinfra | jq -r '.Stacks[].Outputs[] | select(.OutputKey=="PrivateHostedZoneId") | .OutputValue')

```

```
# sed -i -e "s@privatezoneid@$privatehostedzoneid@g" *.json
```

* PrivateHostedZoneName

```
# InfrastructureShortName=$(jq -r .clusterName ./metadata.json | cut -d"-" -f1-3)

# privatehostedzonename=$(aws route53 list-hosted-zones | jq -r '.HostedZones[] | select(.Config.PrivateZone==true) | .Name' | grep $InfrastructureShortName)
```

```
# sed -i -e "s@privatezonename@$privatehostedzonename@g" *.json
```

* Master0Subnet

```
# master0subnet=$(aws ec2 describe-subnets --filter="Name=vpc-id,Values=$vpcid" | jq -r '.Subnets[] | select(.Tags[].Value=="PrivateSubnet") | .SubnetId' | paste -s -d"," - | cut -d"," -f1)

# echo $master0subnet
subnet-064445b01e6xxxxx
```

```
# sed -i -e "s@master0subnet@$master0subnet@g" *.json
```

* Master1Subnet

```
# master1subnet=$(aws ec2 describe-subnets --filter="Name=vpc-id,Values=$vpcid" | jq -r '.Subnets[] | select(.Tags[].Value=="PrivateSubnet2") | .SubnetId' | paste -s -d"," - | cut -d"," -f2)
# echo $master1subnet
subnet-0ba6da724f2xxxxx
```

```
# sed -i -e "s@master1subnet@$master1subnet@g" *.json
```

* Master2Subnet

```
# master2subnet=$(aws ec2 describe-subnets --filter="Name=vpc-id,Values=$vpcid" | jq -r '.Subnets[] | select(.Tags[].Value=="PrivateSubnet3") | .SubnetId' | paste -s -d"," - | cut -d"," -f3)
# echo $master2subnet
subnet-016378620axxxxx
```

```
# sed -i -e "s@master2subnet@$master2subnet@g" *.json
```

* MasterSecurityGroupId

```
# mastersecuritygroupid=$(aws cloudformation describe-stacks --stack-name chernandez-clustersecurity | jq -r '.Stacks[].Outputs[] | select(.OutputKey=="MasterSecurityGroupId") | .OutputValue')
# echo $mastersecuritygroupid
sg-054ce5b3ab7xxxxx
```

```
# sed -i -e "s@mastersecuritygroupid@$mastersecuritygroupid@g" *.json
```

* IgnitionLocation:

```
# apiserverdnsname=$(aws cloudformation describe-stacks --stack-name chernandez-clusterinfra | jq -r '.Stacks[].Outputs[] | select(.OutputKey=="ApiServerDnsName") | .OutputValue')
# echo $apiserverdnsname
api-int.chernandez-ocp4.aa37.sandbox675.opentlc.com

# masterignitionlocation="https://$apiserverdnsname:22623/config/master"
```

```
# sed -i -e "s@masterignitionlocation@$masterignitionlocation@g" *.json
```

* CertificateAuthorities

```
#  mastercert=$(cat ../upi_ocp4_aws/master.ign | jq -r .ignition.security.tls.certificateAuthorities[].source)
```

```
# sed -i -e "s@mastercert@$mastercert@g" *.json
```

* MasterInstanceProfile

```
# masterinstanceprofile=$(aws cloudformation describe-stacks --stack-name chernandez-clustersecurity | jq -r '.Stacks[].Outputs[] | select(.OutputKey=="MasterInstanceProfile") | .OutputValue')

# echo $masterinstanceprofile
clustersecurity-MasterInstanceProfile-XXXXX
```

```
# sed -i -e "s@masterinstanceprofile@$masterinstanceprofile@g" *.json
```

* MasterInstanceType: [MasterInstancesTypes](https://docs.openshift.com/container-platform/4.1/installing/installing_aws_user_infra/installing-aws-user-infra.html#installation-creating-aws-control-plane_installing-aws-user-infra)

```
# masterinstancetype="m4.xlarge"
```

```
# sed -i -e "s@masterinstancetype@$masterinstancetype@g" *.json
```
* AutoRegisterELB: (By default yes)
* RegisterNlbIpTargetsLambdaArn: (already filled)
* ExternalApiTargetGroupArn: (already filled)
* InternalApiTargetGroupArn: (already filled)
* InternalServiceTargetGroupArn: (already filled)

### 10.2 Executing the control plane machines in AWS

```
# aws cloudformation create-stack --stack-name chernandez-clustermaster --template-body file://05_cluster_master_nodes.yaml --parameters file://05_cluster_master_nodes.json --capabilities CAPABILITY_NAMED_IAM --tags Key="Owner",Value="c.hernandez@example.com"

{
    "StackId": "arn:aws:cloudformation:us-west-2:xxxxx:stack/chernandez-clustermaster/79636a60-8e7f-11e9-b025-xxxxx"
}

# aws cloudformation wait stack-create-complete --stack-name chernandez-clustermaster
```

## 11. Creating the worker nodes in AWS

* InfrastructureName: (already filled)
* RhcosAmi: (already filled)
* Subnet: (already filled)

* WorkerInstanceProfile

```
# workerinstanceprofile=$( aws cloudformation describe-stacks --stack-name chernandez-clustersecurity     | jq -r '.Stacks[].Outputs[] | select(.OutputKey=="WorkerInstanceProfile") | .OutputValue')

# echo $workerinstanceprofile
clustersecurity-WorkerInstanceProfile-XXXXX
```

```
# sed -i -e "s/workerinstanceprofile/$workerinstanceprofile/g" *.json
```

* WorkerSecurityGroupId

```
# workersecuritygroupid=$( aws cloudformation describe-stacks --stack-name chernandez-clustersecurity  | jq -r '.Stacks[].Outputs[] | select(.OutputKey=="WorkerSecurityGroupId") | .OutputValue')
```

```
# sed -i -e "s/workersecuritygroupid/$workersecuritygroupid/g" *.json
```

* WorkerInstanceType

```
# workerinstancetype="m4.xlarge"
```

```
# sed -i -e "s/workerinstancetype/$workerinstancetype/g" *.json
```

```
# workerignitionlocation="https://$apiserverdnsname:22623/config/worker"
```

```
# sed -i -e "s@workerignitionlocation@$workerignitionlocation@g" *.json
```

* CertificateAuthorities

```
# workercert=$(cat ../upi_ocp4_aws/worker.ign | jq -r .ignition.security.tls.certificateAuthorities[].source)
```

```
# sed -i -e "s@workercert@$workercert@g" *.json
```

```
# aws cloudformation create-stack --stack-name chernandez-clusterworker1 --template-body file://06_cluster_worker_node.yaml --parameters file://06_cluster_worker_node.json --capabilities CAPABILITY_NAMED_IAM --tags Key="Owner",Value="c.hernandez@example.com"

{
    "StackId": "arn:aws:cloudformation:us-west-2:xxxxx:stack/clustermaster/79636a60-8e7f-11e9-b025-xxxxx"
}

# aws cloudformation create-stack --stack-name chernandez-clusterworker2 --template-body file://07_secondary_cluster_worker_node.yaml --parameters file://07_secondary_cluster_worker_node.json --capabilities CAPABILITY_NAMED_IAM --tags Key="Owner",Value="c.hernandez@example.com"
# aws cloudformation create-stack --stack-name chernandez-clusterworker3 --template-body file://08_tertiary_cluster_worker_node.yaml --parameters file://08_tertiary_cluster_worker_node.json --capabilities CAPABILITY_NAMED_IAM --tags Key="Owner",Value="c.hernandez@example.com"
# aws cloudformation wait stack-create-complete --stack-name chernandez-clusterworker1
```
```
# openshift-install wait-for bootstrap-complete --dir=.
```

**IMPORTANT STEP:**

You must watch if the csrs are aprroved, if not you must approve them with the next command:
```
# export KUBECONFIG="~/upi_ocp4_aws/auth/kubeconfig"
# csr=$(oc get csr | grep "Pending")
# if $csr oc get csr -ojson | jq -r '.items[] | select(.status == {} ) | .metadata.name' | xargs oc adm certificate approve
or
# oc get csr -o name | xargs oc adm certificate approve
```
Wait for the installation complete.
```
openshift-install wait-for install-complete --dir=.
```
And finally, delete the bootstrap resources.
```
 aws cloudformation delete-stack --stack-name clusterbootstrap
```
**You must leave your cluster running for the first 24 hours**, because the first certificate rotation happens within that period. Once that completes you can shutdown the cluster and bring it up whenever you need it. 