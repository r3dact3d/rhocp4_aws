# rhocp4_aws

### Assumptions
* Registered Domain in AWS with a Public Hosted Zone
* AWSCLI is installed and in PATH
* AWS User has Access Key credentials configured
* openshift-installer bindary in PATH from [cloud.redhat.com](https://cloud.redhat.com/openshift/install/aws/user-provisioned)
* Downloaded PullSecret from [cloud.redhat.com](https://cloud.redhat.com/openshift/install/aws/user-provisioned) for customer
* OC and Kubectl binaries are in PATH
* Using AWS Region us-east-1 - if you use another region, make sure it has 3 AZ.  Set as your default in \~/.aws/config and update the AWS_DEFAULT_REGION in the .github/workflows/validate.yaml file

### Installation
1. Create working directory
```bash
cd; mkdir ocp4Install; cd ocp4Install
```
2. Get the HostedZoneId for your domain and save this output to use later in parameters file. Replace `<domain name<` in the command below with your own domain name.
```bash
aws route53 list-hosted-zones-by-name --dns-name <domain name> --query 'HostedZones[0].Id' | cut -d/ -f3 |cut -d\" -f1
```
3. Create SSH Keypair, start ssh-agent, and add keypair to agent. if you do not already have one for use.
```bash
ssh-keygen -t rsa -b 4096 -N '' -f ~/.ssh/id_rsa
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_rsa
``` 
4. Create custom install file
```bash
openshift-install create install-config --dir=.
```
5. Edit the install-config.yaml and lower worker replicas from 3 to 0
6. Generate manifest files
```bash
openshift-install create manifests --dir=.
```
7. Remove files that define control plane and worker nodes
```bash
rm -rf openshift/99_openshift-cluster-api_{master,worker}-machines*.yaml
```
8. Create ignition files
```bash
openshift-install create ignition-configs --dir=.
```
9. Extract infraID for use later with parameters file
```bash
grep infraID metadata.json |cut -d: -f4 |cut -d, -f1
```
10. Create s3 DevopsBucket and Upload Bootstrap ignition. Make your own bucket name.
```bash
aws s3 mb s3://<devopsbucket>
aws s3 cp bootstrap.ign s3://<devopsbucket>/bootstrap.ign
```
11. Clone this repo 
```bash
git clone https://github.com/r3dact3d/rhocp4_aws.git
cd rhocp4_aws
```
12. Upload Cloudformation cf-modules to DevopsBucket 
```bash
aws s3 sync cf-modules s3://<devopsbucket>/cf-modules/
```
13. If you don't already have a RHCOS AMI to use, list and pick one
```bash
aws ec2 describe-images --query 'sort_by(Images, &CreationDate)[*].[CreationDate,Name,ImageId]' --filters "Name=name,Values=rhcos*"  --output table
```
14. Create a new git branch and update params.json with parameters for your cluster \*See Cloudformation Files section below\* \*NOTE: don't update certificate authorities here, that is updated next\*
```bash
git checkout -b deploy/<new branch name>
```
15. Update GitHub Secrets with your values \*See GitHub Secrets section below\*
16. Create PR for approval and merge to master branch.  This will kick off the GitHub Actions CI/CD pipeline. Watch progress of CloudFormation deployment either from AWS console or using AWS CLI\*See GitHub Actions section below\*
```bash
aws cloudformation describe-stacks --stack-name sandboxStack --query 'Stacks[*].StackStatus'
```
17. Watch for successful bootstrapping of OCP Cluster 
```bash
openshift-install wait-for bootstrap-complete --dir=. --log-level debug
```
18. Get kube credentials
```bash
export KUBECONFIG=auth/kubeconfig 
```
19. Approve [CSR](https://docs.openshift.com/container-platform/4.2/installing/installing_aws_user_infra/installing-aws-user-infra.html#installation-approve-csrs_installing-aws-user-infra)

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