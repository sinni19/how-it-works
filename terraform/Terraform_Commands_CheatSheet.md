# Terraform Command Cheat Sheet

![Terraform_Command_CheatSheet.](terraform_command_cheatsheet.jpg.jpg "terraform commandcheatsheet")

## Initialize
```bash
terraform init
```
Initialize a Terraform working directory.

## Format
```bash
terraform fmt
terraform fmt -recursive
```
Format Terraform configuration files.

## Validate
```bash
terraform validate
```
Validate configuration files.

## Plan
```bash
terraform plan
terraform plan -out=tfplan
terraform plan -var="region=us-east-1"
terraform plan -var-file="dev.tfvars"
```
Preview infrastructure changes.

## Apply
```bash
terraform apply
terraform apply tfplan
terraform apply -auto-approve
```
Apply infrastructure changes.

## Destroy
```bash
terraform destroy
terraform destroy -auto-approve
```
Destroy managed infrastructure.

## Show
```bash
terraform show
terraform show tfplan
```
Show the current state or execution plan.

## Output
```bash
terraform output
terraform output instance_ip
terraform output -json
```
Display output variables.

## State
```bash
terraform state list
terraform state show aws_instance.example
terraform state mv SOURCE DESTINATION
terraform state rm RESOURCE
```
Inspect or modify Terraform state.

## Import
```bash
terraform import aws_instance.example i-1234567890abcdef0
```
Import existing infrastructure into Terraform state.

## Refresh
```bash
terraform refresh
```
Refresh state to match real infrastructure.

## Workspace
```bash
terraform workspace list
terraform workspace new dev
terraform workspace select dev
terraform workspace delete dev
```
Manage workspaces.

## Providers
```bash
terraform providers
terraform providers lock
```
Inspect provider requirements.

## Common Global Options
```bash
terraform plan -chdir=env/dev
terraform apply -lock=false
terraform plan -parallelism=20
```

## Useful Environment Variables
```bash
export TF_VAR_region=us-east-1
export TF_LOG=INFO
export TF_LOG_PATH=./terraform.log
export TF_INPUT=0
export TF_IN_AUTOMATION=1
```

## Typical Workflow
```bash
terraform init
terraform fmt -recursive
terraform validate
terraform plan -out=tfplan
terraform apply tfplan
```
