# Composition pattern — environment = set of modules
module "networking" {
  source      = "registry/platform/vpc/aws"
  version     = "~> 2.0"
  environment = var.environment
  cidr_block  = var.vpc_cidr
}

module "eks_cluster" {
  source      = "registry/platform/eks/aws"
  version     = "~> 4.1"
  environment = var.environment
  vpc_id      = module.networking.vpc_id
  node_count  = local.env_config[var.environment].node_count
}

locals {
  env_config = {
    dev     = { node_count = 2, instance = "t3.medium" }
    staging = { node_count = 3, instance = "m5.large"  }
    prod    = { node_count = 6, instance = "m5.xlarge" }
  }
}
```

**Self-service provisioning flow:**
```
Developer opens PR to environments repo
       ↓
Atlantis/Terraform Cloud runs plan
       ↓
Platform team auto-approves if policy passes
       ↓
Merge → apply → environment ready
       ↓
Outputs written to SSM / Service Catalog
```

---

### Interview Questions

**Q: How do you enable self-service environment provisioning without losing control?**
> Define approved patterns in templates. Use policy-as-code (OPA/Sentinel) to enforce guardrails at plan time. Require PRs — giving audit trail. Auto-approve if all policies pass; escalate to platform team only for exceptions. Use resource quotas and budget alerts to prevent runaway costs.

**Q: How do you ensure dev/staging/prod parity?**
> Same Terraform modules across all environments, config-driven differences only (instance size, replica count). Avoid "it works in staging" problems by keeping the topology identical — just scaled down. Use feature flags for behavioral differences, not separate code paths.

**Q: How do you handle environment sprawl — teams spinning up envs and abandoning them?**
> TTL on non-prod environments. Scheduled `terraform destroy` on envs older than X days with no activity. Ownership tags on every resource. Monthly cost reports sent to team leads. Require re-approval for environments to live beyond their TTL.

**Q: How do you onboard a new team onto the platform?**
> Self-service portal or IDP (Internal Developer Portal like Backstage). Team fills in a form: team name, service name, tech stack. Platform automation creates: Git repo, CI/CD pipeline from template, base infrastructure, IAM roles, observability setup. Target: < 1 day from request to first deployment.

---

## 4. Policy-as-Code

### What to Know
- Enforce standards programmatically, not by review
- Shift left — catch violations before they reach production
- Tools: OPA (Open Policy Agent), Conftest, Sentinel (Terraform Cloud), Checkov, Kyverno (Kubernetes)

### Implementation Layers
```
Developer Workstation     →  pre-commit hooks (tflint, checkov)
Pull Request              →  CI policy checks (conftest + OPA)
Terraform Plan            →  Sentinel / OPA policies on plan output
Kubernetes Admission      →  Kyverno / OPA Gatekeeper (runtime)
Drift Detection           →  Scheduled scans (AWS Config, Prisma)
