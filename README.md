# rhocp4_aws

### Assumptions
* Registered Domain in AWS with a Public Hosted Zone
* Have Openshift-installer bindary in PATH from [cloud.redhat.com](https://cloud.redhat.com/openshift/install/aws/user-provisioned)
* Downloaded PullSecret from [cloud.redhat.com](https://cloud.redhat.com/openshift/install/aws/user-provisioned) for customer
* Have SSH Key and Agent running
* Using AWS Region with 3 AZs
* OC and Kubectl binaries are in PATH


### Installation
1. Get the HostedZoneId for your domain and save this output to use later in parameters file. Replace `<domain name<` in the command below with your own domain name.
```bash
aws route53 list-hosted-zones-by-name --dns-name <domain name> --query 'HostedZones[0].Id' | cut -d/ -f3 |cut -d\" -f1
```
2. Create SSH Keypair, start ssh-agent, and add keypair to agent. if you do not already have one for use.
```bash
ssh-keygen -t rsa -b 4096 -N '' -f ~/.ssh/id_rsa
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_rsa
``` 
3. Create customer install file
```bash
openshift-install create install-config --dir=.
```
4. Edit the install-config.yaml and lower worker reaplicas from 3 to 0
5. Generate manifest files
```bash
openshift-install create manifests --dir=.
```
6. Remove files that define control plane and worker nodes
```bash
rm -rf openshift/99_openshift-cluster-api_{master,worker}-machines*.yaml
```
7. Create ignition files
```bash
openshift-install create ignition-configs --dir=.
```
8. Extract infraID for use later with parameters file
```bash
grep infraID metadata.json |cut -d: -f4 |cut -d, -f1
```
9. Create s3 DevopsBucket and Upload Bootstrap ignition
```bash
aws s3 mb s3://ocp4-devops-infra
aws s3 cp bootstrap.ign s3://ocp4-devops-infra/bootstrap.ign
```
10. Clone this repo and update params.json with parameters for your cluster \*See Cloudformation Files section below\*
```bash
git clone https://github.com/r3dact3d/rhocp4_aws.git
cd rhocp4_aws
```
11. Update GitHub Secrets with your values \*See GitHub Secrets section below\*
12. Create PR for approval and merge to master branch.  This will kick off the GitHub Actions CI/CD pipeline. \*See GitHub Actions section below\*


### GitHub Secrets
* CERTIFICATE_AUTHORITIES - This is found in __master.ign__
* AWS_ACCESS_KEY_ID - This is your key
* AWS_SECRET_ACCESS_KEY - This is your secret

### Cloudformation Files
* params.json - This is the parameters file that passes customizable params to the parent stack infra.yml
* infra.yml - This is in root directory and is the parent stack, it holds some parameters that can be modified that aren't in the parameters file.
* cf-modules - This directory holds the Cloudformation templates that create the different nested stacks that are called by infra.yml

### GitHub Actions
* validate.aml - After updating pushing any changes to Master this will automatically start a pipeline to validate the template and create a parent stack with nested resources.  Will need to edit this, to make specific stack name or change the AWS region.

#### TODO
* Add step in pipeline to put cf-modules in s3 bucket - needs to read DevopsBucket from params
* Add some kind of way to either __create-stack__ or __update-stack__ via GitHub Action
* Add creation of Public Hosted Zone -> HostedZoneId 

#### Log
Deploy 0.0.2