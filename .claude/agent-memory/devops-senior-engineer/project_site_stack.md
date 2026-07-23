---
name: project-site-stack
description: ADR-0005 and ADR-0006 implemented together in the site/ Terraform stack — CloudFront, S3, WAF, Route53, GitHub OIDC roles
metadata:
  type: project
---

The `site/` stack at `devopsproject-ecommerce-iac/terraform/site/` covers two ADRs in one state file:

- **ADR-0005**: GitHub Actions OIDC — `iam.oidc-provider.tf`, `iam.github-backend-role.tf`, `iam.github-frontend-role.tf`
- **ADR-0006**: CloudFront + S3 static site with continuous deployment — everything else

**Why:** Both ADRs share the same resources (S3 buckets, CloudFront distributions) so splitting them would require cross-stack output wiring for no benefit.

**How to apply:** Two-phase apply is mandatory.

### Phase 1 (current code)
`cloudfront.production.tf` has `lifecycle { ignore_changes = [continuous_deployment_policy_id] }`.
The `continuous_deployment_policy_id` attribute IS declared but the lifecycle block prevents AWS from rejecting the first apply before the staging distribution exists.

### Phase 2 (operator action required)
After Phase 1 succeeds: remove the `lifecycle` block from `cloudfront.production.tf` and run `terraform apply` again. This wires `aws_cloudfront_continuous_deployment_policy.this.id` into the production distribution.

### VPC Origin note
`aws_cloudfront_vpc_origin` for ALB requires the ALB to already exist (created by the server/EKS stack). The `data.aws_lb.cluster` lookup will fail if the ALB is not yet provisioned. Ensure the server stack has been applied first.

### Backend state keys
- networking: `networking/terraform.tfstate`
- server: `server/terraform.tfstate`
- site (this stack): `site/terraform.tfstate`

All in bucket `devopsproject-terraform-state-files`, DynamoDB lock `devopsproject-terraform-state-locking`.
