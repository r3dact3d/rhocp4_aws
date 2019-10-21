# rhocp4_aws
## Assumptions
* Public Hosted Zone is created
* Downloaded PullSecret from https://cloud.redhat.com/openshift/install for customer
* Have SSH Key and Agent running
* Have your own install-config, manifests, and ignition files

### GitHub Secrets
* **CONTAINER_AUTHORITIES** - This is found in __worker.ign__
* **AWS_ACCESS_KEY_ID**
* **AWS_SECRET_ACCESS_KEY**

**params.json**
This is the parameters file that passes customizable params to the parent stack infra.yml

**infra.yml**
This is in root directory and is the parent stack, it holds some parameters that can be modified that aren't in the parameters file.

**cf-modules**
This directory holds the Cloudformation templates that create the different nested stacks that are called by infra.yml


### GitHub Actions
**validate.aml**


#### TODO
* Add step in pipeline to put cf-modules in s3 bucket - needs to read DevopsBucket from params
* Add some kind of way to either __create-stack__ or __update-stack__ via GitHub Action
* Add creation of Public Hosted Zone -> HostedZoneId 