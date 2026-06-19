# Lessons Learned

## What I set out to do

I had already built two separate labs — a Unified Endpoint Management lab and an Email Authentication lab — and I wanted to connect them into a single, coherent enterprise environment rather than leave them as unrelated projects. The new piece was Infrastructure as Code: using Terraform to provision the governance and identity foundation that the rest of the environment runs on.
---

## What worked well

- **Treating Terraform as the foundation, not the whole project.** Provisioning governance, identity groups, RBAC, and monitoring as code — and deliberately leaving Conditional Access, PIM, and Intune in the portal — kept the scope realistic and the story clear.
- **Tagging as code.** Seeing `managed_by: Terraform` show up as a tag in the portal made the connection between code and live infrastructure obvious and verifiable.
- **A dynamic device group.** Expressing membership logic (`device.deviceOSType -eq "Windows"`) in code, then targeting it with an Intune compliance policy, was the cleanest link between the foundation and the existing endpoint work.

---

## What I'd do differently

- **Disable bulk provider registration from the start.** My first `terraform plan` appeared to hang because the provider was registering every resource provider on the subscription. Setting `resource_provider_registrations = "none"` up front would have avoided that.
- **Plan the screenshot naming convention before starting**, not midway. Consistent, descriptive filenames (`phaseN-descriptor.png`) make the final repo look far more professional, and renaming after the fact is where naming drifts.

---

## The hardest concept to grasp

**The Infrastructure-as-Code boundary** — deciding what belongs in Terraform and what should stay in the portal. It's tempting to put everything in code, but Conditional Access and PIM carry real lockout and approval-workflow concerns, and the right move is to operate them by hand and document why. Understanding *why* the line is drawn there, rather than just where, was the most valuable thing I took from this project.

---

## What I'd build next

- Move Terraform state to a remote backend (Azure Storage) with state locking.
- Add a Microsoft Sentinel layer on top of the existing Log Analytics workspace for alerting on suspicious sign-ins.
- Explore managing Intune and Conditional Access policy as code via the Microsoft Graph provider as it matures.
