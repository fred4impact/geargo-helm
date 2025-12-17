# Migrate existing AWS resources into Terraform state

This guide shows safe, repeatable commands to import existing AWS resources into an existing Terraform state file. It's intended for situations where resources were created outside Terraform (manually or by other tooling) and you want to bring them under Terraform management by matching them with an existing Terraform configuration.

Important: terraform import only updates the state. You must have Terraform configuration (HCL) that matches the real resource attributes (or adjust after import). Always backup state before modifying.

---

## Prerequisites

- Terraform 1.0+ installed and on your PATH
- AWS CLI configured with credentials that have permission to list and describe resources and to manage the backend
- The Terraform configuration (HCL) for the resources you will import present locally
- Access to the Terraform backend or state file (S3 backend, local state, etc.)

## Safety first — backup the state

If you use a remote backend (S3 + DynamoDB locking) get a copy of current state first:

```bash
# For S3 backend: pull a local copy
terraform init
terraform state pull > tfstate-backup-$(date +%F-%H%M%S).backup

# Or download the state file from your backend as configured
aws s3 cp s3://<your-tfstate-bucket>/<path>/terraform.tfstate ./terraform.tfstate.backup
```

If you use local state simply copy it:

```bash
cp terraform.tfstate terraform.tfstate.backup
```

## General workflow

1. Ensure your Terraform configuration (resource blocks, module addresses) exists and corresponds to the AWS resource you want to import.
2. Run `terraform init` to initialize the backend and provider plugins.
3. Use `terraform import <ADDRESS> <ID>` to import the resource into the state.
4. Run `terraform plan` and reconcile configuration as needed (add missing attributes, computed fields, or ignore_changes).

### Example: import a VPC

Assume your Terraform configuration defines a VPC like:

```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  # ...
}
```

Import command (replace with your VPC id):

```bash
terraform init
terraform import aws_vpc.main vpc-0abcd1234
```

After importing, run:

```bash
terraform plan
```

If `plan` shows differences, update your HCL or add `lifecycle { ignore_changes = [...] }` where appropriate.

### Example: import a subnet

If you have a subnet resource `aws_subnet.private[0]` (a list), import with an index address:

```bash
terraform import 'aws_subnet.private[0]' subnet-0123456789abcdef0
```

### Importing resources inside modules

If the resource exists in a module, address it like:

```bash
terraform import module.<MODULE_NAME>.aws_eks_cluster.cluster my-eks-cluster-name
```

Or if module is called from a module block with an index:

```bash
terraform import module.eks_cluster[0].aws_eks_cluster.cluster my-eks-cluster-name
```

### Importing IAM roles, policies, and attachments

IAM resources commonly have ARNs that are used as IDs:

```bash
terraform import aws_iam_role.my_role arn:aws:iam::123456789012:role/my-role-name
terraform import aws_iam_policy_attachment.attach_name attachment-name
```

### Importing an EKS cluster and node group

EKS cluster:

```bash
terraform import aws_eks_cluster.cluster my-eks-cluster-name
```

Managed nodegroup (example):

```bash
terraform import aws_eks_node_group.nodegroup my-cluster-name:my-nodegroup-name
```

### Importing RDS

```bash
terraform import aws_db_instance.geargo_db db-ABCDEFGHIJKLMNOP
```

### Importing ECR repository

```bash
terraform import aws_ecr_repository.geargo_repo geargo
```

## Bulk imports — scripts

If you have many resources to import you can script imports. Example: import a list of subnets from CSV.

`imports/subnets-import.sh` (example):

```bash
#!/usr/bin/env bash
set -euo pipefail

# CSV file format: INDEX,SUBNET_ID
# e.g. 0,subnet-0123abcd
CSV=./imports/subnets.csv
TF_ADDR_BASE="aws_subnet.private"

while IFS=, read -r idx id; do
  addr="${TF_ADDR_BASE}[${idx}]"
  echo "Importing $id as $addr"
  terraform import "$addr" "$id"
done < "$CSV"
```

Tips for scripting:
- Use `set -euo pipefail` and logs
- Do imports in small batches and run `terraform plan` after each batch
- If import fails, inspect `terraform state list` and `terraform state show <address>`

## After import — reconcile configuration

- Run `terraform plan` and inspect differences.
- Update HCL to match remote attributes where possible.
- For attributes that must remain out-of-band, add `lifecycle { ignore_changes = [...] }` to resource blocks.
- If Terraform reports resources it wants to destroy, double-check the resource's configuration and state address mapping — don't apply unless you intend to change/destroy.

## Advanced state operations

- Rename addresses: use `terraform state mv <old> <new>` to move/imported resources to a different address if needed.

```bash
# Move an imported resource into a module address if you originally imported it to a flat address
terraform state mv aws_vpc.main module.network.aws_vpc.main
```

- Remove a resource from state (does not delete real resource):

```bash
terraform state rm aws_s3_bucket.oldbucket
```

- Show state contents:

```bash
terraform state list
terraform state show <ADDRESS>
```

- Push/pull state when using local copies (be careful):

```bash
terraform state pull > /tmp/state.json  # read
# modify if needed (careful)
terraform state push /tmp/state.json    # write (legacy; use remote backends instead)
```

## Remote backend considerations (S3 + DynamoDB lock)

- Ensure `terraform init` is run and can access the backend.
- Keep state locking in place while importing to avoid concurrent edits.
- If you need to import using a CI runner, make sure the runner has access to the backend and DynamoDB lock permissions.

## Common gotchas

- Resource address mismatch: your HCL name must match how you reference the resource in `terraform import` (including indexes and module paths).
- Computed-only attributes: some attributes are only set after creation; your HCL must not try to set them if they are computed.
- Provider versions: differences between provider versions can change attribute names/behaviors. Use the same provider version as in production when possible.
- Importing resources with generated names/tags: if Terraform config expects to create a resource but the existing resource has different attributes, plan may attempt to replace it.

## Quick checklist

1. git checkout -b tf-import-<component>
2. terraform init
3. terraform state pull > backup
4. Run `terraform import` for one resource
5. terraform plan
6. Update HCL or `ignore_changes` as needed
7. Repeat in small batches
8. Commit changes to HCL and push

---

If you'd like, I can:
- Add a sample import script tailored to specific AWS resources you have (EKS, RDS, VPC). Provide the resource IDs and desired Terraform addresses and I'll create the script.
- Generate a `imports/` folder with example CSVs and bash helpers.

Which resources would you like me to generate import scripts for? (e.g. VPC + subnets, EKS cluster + nodegroups, RDS instance, ECR repos)