# rhocp4_aws

### params.json
This is the parameters file that passes customizable params to the parent stack infra.yml

### infra.yml
This is in root directory and is the parent stack, it holds some parameters that can be modified that aren't in the parameters file.

### cf-modules
This directory holds the Cloudformation templates that create the different nested stacks that are called by infra.yml


## GitHub Actions
### validate.aml
Takes each module and validates