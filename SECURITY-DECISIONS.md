# Security Decisions

This document explains *why* each control in the Northgate Solutions environment lives where it does — specifically, what is managed as code with Terraform versus operated in the portal, and the reasoning behind each choice. The boundary itself is the point: knowing what belongs in Infrastructure as Code, and what deliberately does not, is a core platform-engineering judgment.

---

## The Infrastructure-as-Code boundary

| Control | Managed by | Why |
|---------|-----------|-----|
| Resource groups, tags | **Terraform** | Repeatable, low-risk, the foundation everything else sits in |
| Azure Policy guardrail | **Terraform** | Governance expressed as reviewable code |
| Entra ID groups & dynamic groups | **Terraform** | Reproducible identity; membership logic versioned |
| RBAC role assignments | **Terraform** | Least privilege as code, scoped to resource groups |
| Log Analytics workspace | **Terraform** | Centralized monitoring infrastructure |
| Entra diagnostic routing | **Portal** | Tenant-level setting, outside the Azure resource model |
| Conditional Access | **Portal** | Lockout risk; sensitive blast radius |
| Privileged Identity Management | **Portal** | Approval workflows and just-in-time elevation |
| Intune compliance & config | **Portal** | Graph-based IaC for Intune is immature at this scope |
| SPF / DKIM / DMARC | **DNS / Microsoft 365** | DNS records and mail flow, outside Azure entirely |

---

## Key decisions

Each decision is framed as **Decision → Rationale → Trade-off**.

### 1. Conditional Access and PIM are operated in the portal, not Terraform
- **Decision:** Access policies and privileged-role elevation are configured and changed in the portal, not declared in Terraform.
- **Rationale:** The `azuread` provider *can* manage Conditional Access, but a single faulty apply can lock every administrator out of the tenant. PIM centers on approval workflows and just-in-time elevation, which are a poor fit for declarative, all-at-once infrastructure code.
- **Trade-off:** These controls aren't version-controlled the way the foundation is. Mitigated by documenting them here and keeping their target *groups* in code, so the objects they apply to are still reproducible.

### 2. RBAC is scoped least-privilege to resource groups
- **Decision:** Role assignments are scoped to individual resource groups rather than the whole subscription.
- **Rationale:** Limits blast radius — a group with Reader on `rg-northgate-identity` cannot see or touch anything else.
- **Trade-off:** More granular assignments to manage as the environment grows; acceptable for the security gain.

### 3. A dynamic device group targets Windows endpoints
- **Decision:** Windows devices are grouped by a dynamic membership rule (`device.deviceOSType -eq "Windows"`) rather than manual membership.
- **Rationale:** New Windows 11 devices are covered by Intune compliance automatically as they enroll; the membership logic lives in code.
- **Trade-off:** Requires a Microsoft Entra ID P1 license, and the rule must be maintained as device strategy evolves.

### 4. Terraform state is local for this lab
- **Decision:** State is stored locally rather than in a remote backend.
- **Rationale:** Simplicity for a single-operator lab; no collaboration or locking needs.
- **Trade-off:** No remote state locking or team workflow. A production deployment would use a remote backend (Azure Storage) with state locking — noted as the explicit next step.

### 5. Entra diagnostic routing is configured in the portal
- **Decision:** Terraform provisions the Log Analytics workspace; the routing of Entra sign-in and audit logs into it is configured in the portal.
- **Rationale:** Tenant-level Entra diagnostic settings sit outside the Azure resource model that the `azurerm` provider manages cleanly.
- **Trade-off:** The routing configuration isn't captured in code; documented here instead, and the workspace it targets *is* code.

### 6. Email authentication is managed at DNS and Microsoft 365
- **Decision:** SPF, DKIM, and DMARC for the Northgate domain live at the DNS provider and in Exchange Online, not in Azure or Terraform.
- **Rationale:** These are DNS records and mail-flow configuration — a different management plane from Azure resources.
- **Trade-off:** A separate management surface, tied into the overall architecture through documentation and the shared `northgate` domain rather than through code.

### 7. No secrets in state or in the repository
- **Decision:** No plaintext secrets are committed; state files are gitignored; only a non-secret `terraform.tfvars.example` is published.
- **Rationale:** Protects credentials and tenant data.
- **Trade-off:** The repo isn't directly runnable by a third party without supplying their own values — an acceptable and expected trade for a public portfolio.

---

## How the layers connect

Terraform provisions the **foundation** — governance, identity groups, least-privilege RBAC, and the monitoring workspace. On top of that foundation:

- **Conditional Access** enforces MFA against the Terraform-provisioned `Northgate-Helpdesk` group.
- **Intune** applies compliance policy to the Terraform-provisioned `Northgate-Windows11-Devices` dynamic group, so every enrolled Windows 11 device is governed automatically.
- **PIM** provides just-in-time elevation for privileged roles rather than standing access.
- **SPF / DKIM / DMARC** protect the same `northgate` domain from spoofing — closing the gap that identity controls alone can't, since access policies stop unauthorized *sign-ins* but not attackers *impersonating the domain* to phish users.
- **Log Analytics** centralizes Entra sign-in and audit activity, giving the foundation a single monitoring surface.

The result is one connected environment in which Infrastructure as Code, identity, endpoints, email, and monitoring reinforce each other — and in which every manually operated control is a deliberate, documented decision rather than a gap.
