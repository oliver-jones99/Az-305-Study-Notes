# Design identity, governance, and monitoring solutions (25–30%)

## Resources:

-

## Design solutions for logging and monitoring

#### -Recommend a logging solution

##### Log types – what they show

| Log | Scope | Shows | On by default? | Default retention | How to collect |
|---|---|---|---|---|---|
| **Activity Log** | Subscription (control plane / ARM) | Who created, changed or deleted a *resource* – RBAC, policy, deletes, Service Health | Yes | 90 days | Subscription diagnostic setting (Export Activity Logs) → `AzureActivity` |
| **Resource logs** | Per resource (data plane) | What happened *inside* a resource – blob deleted, secret read, SQL queries | **No** | Not kept | Diagnostic setting on **each** resource → e.g. `StorageBlobLogs`, `AzureDiagnostics` |
| **Entra ID logs** | Tenant (identity) | Sign-ins (who, where, MFA result) and directory changes (users, groups, roles) | Yes, in Entra | 7–30 days (by licence) | Entra ID → Diagnostic settings → `SigninLogs`, `AuditLogs` |
| **Guest OS logs** | Inside a VM | Windows events, Syslog, performance counters | No | – | **Azure Monitor Agent + data collection rule** (MMA retired) |

- A **diagnostic setting** = "send these logs there". No setting means no resource logs.
- Logging is **forward-only** – nothing older is backfilled.
- Control-plane actions show *who* in `AzureActivity` (`Caller`), not in the resource log.
- Resource logs only show *who* with **Entra auth** (UPN). Account key or trusted access = no identity → disable shared key.
- Ingestion delay is roughly 5–15 minutes.
- At scale: **DeployIfNotExists** policy at management-group level to enforce diagnostic settings (needs a managed identity, plus a remediation task for existing resources).

##### Architecture – one central workspace

**Default:** one Log Analytics workspace in the **management subscription** (CAF platform). Every subscription sends to it – a workspace can collect from any subscription in the tenant.

```
mg-platform
 └─ sub-management (lab: sub-platform)
     └─ rg-logging → law-central
            ▲              ▲                ▲
   Activity Log    Activity Log      Resource logs
   (sub-platform)  (sub-landingzone) (KV, storage, SQL…)
```

**Why one:** a single KQL view across everything · one place for alerts, workbooks and Sentinel · pooled free allowance and commitment tiers · less admin.

**Only split for:** data residency · billing separation · hard access boundaries (another tenant, MSP customers).

**Not reasons to split:**
- Teams only seeing their own logs → **resource-context access** ("Use resource or workspace permissions")
- Different retention → set **per table**

**Retention:** interactive (~30 days included; `AzureActivity` 90 days free) → long-term (cheap, up to 12 years, via search job or restore) → export to storage for immutable/WORM compliance.


#### -Recommend a solution for routing logs

**Destinations:**
 
| Destination | What it's for | Think of it as |
|---|---|---|
| **Log Analytics workspace** | KQL queries, alerts, workbooks, Sentinel | The **investigation room** |
| **Storage account** | Cheap long-term archive; can be made immutable (WORM) | The **filing cabinet with a lock** |
| **Event Hubs** | Streams logs out in near real time to something else (a third-party SIEM, Splunk, a custom app) | The **conveyor belt** out of Azure |
| **Partner solution** | Native integrations (e.g. Datadog, Elastic) | A **direct line** to a vendor |
 
**Rules worth knowing:**
- One diagnostic setting can send to **several destinations at once**, but only **one of each type**.
- A resource can have **up to 5 diagnostic settings** – use another one for, e.g., a second workspace, or different log categories going to different places.
- **Storage and Event Hubs must be in the same region as the resource.** Log Analytics doesn't have this restriction.
**Exam gotchas:**
- "Stream", "third-party SIEM", "Splunk/QRadar", "near real time" → **Event Hubs**. "**Microsoft Sentinel**" → **Log Analytics**, because Sentinel sits on top of a workspace.
- Event Hub or storage in a different region from the resource → it won't appear in the diagnostic setting. Fix: create one in the resource's region (the SIEM can still read it cross-region).

#### -Recommend a monitoring solution

## Design authentication and authorization solutions

#### -Recommend an authentication solution

#### -Recommend an identity management solution

#### -Recommend a solution for authorizing access to Azure resources

#### -Recommend a solution for authorizing access to on-premises resources

#### -Recommend a solution to manage secrets, certificates, and keys

## Design governance

#### -Recommend a structure for management groups, subscriptions, and resource groups, and a strategy for resource tagging

## What each layer is for
| Layer | Use it for | Group by |
|---|---|---|
| Management group | Applying the same policy and RBAC to many subscriptions | **Policy archetype** (platform, Corp, Online, sandbox), not the org chart or projects |
| Subscription | Unit of **billing, quota, RBAC and blast radius**. In CAF, each one is a landing zone | One per workload (and environment) |
| Resource group | Unit of **lifecycle**: things deployed and deleted together | App tier or lifecycle |
 
## Key rules
- **Assign as high as sensible.** Inheritance does the work, and new subscriptions get it automatically.
- Child scopes **can't remove** inherited policy. They can only be **exempted**.
- **Exemptions** apply to a whole assignment, at a scope you choose (management group, subscription, resource group or resource):
  - Categories: *Waiver* or *Mitigated*. Set an expiry date.
  - Exempting a single "Not allowed resource types" assignment allows **every** listed type.
- **Moving a subscription** between management groups instantly changes its inherited policy and RBAC.
- **New subscriptions land under the Tenant Root Group** by default. Fix this in *Hierarchy settings → Default management group*.
- **Separate subscriptions vs resource groups and tags:** tags give **cost visibility**; subscriptions give **isolation** (security, quota, blast radius).

## Exam traps
- "Create a subscription to group subscriptions" → never valid. Use a **management group**.
- "Smallest change to allow X for one app" → a **policy exemption scoped to the resource group**, not moving or removing the assignment.
- Owner **can't override** a Deny policy.
- Administrative units scope **Entra ID** roles, not Azure resources.
- A management group per environment is wrong if the environments need the same policies.


## Design principle
Enforce tags where there's nothing to inherit from (resource groups), and let everything below inherit. You get consistent data without blocking delivery teams.
 
## Designing a taxonomy
Start from the requirements: who pays, who owns it, what automation needs. Keep the set small, and define the naming convention first; tags hold what the name can't.
 
| Category | Example tags | Drives |
|---|---|---|
| Functional | `workload`, `env` | Cost per project and environment |
| Billing | `costcentre`, `owner` | Chargeback, accountability |
| Classification | `dataclass` | Security and compliance |
| Operational | `criticality`, `backup`, `shutdown` | Automation |

## Gotchas
- **Policy comparisons are case-insensitive.** `in` and `equals` let `Dev` through; use `match`/`notMatch` for exact case.
- **Deny doesn't fix existing resources;** they just show as non-compliant. Use Modify plus remediation.
- **Remediation fails while new resources still get tagged** = the identity lacks a role on the targets.
- **Tags aren't a security boundary.** Reader can see them, and they appear in cost exports. Never put secrets in tags.
- **Azure-created resource groups** (e.g. `NetworkWatcherRG`) won't have your tags, so a resource group Deny blocks them. Plan an exemption.
- **Tags don't do anything by themselves.** Automation has to read them (runbook, Function).
- **Limits:** 50 tags per resource; tag names are case-insensitive, values case-sensitive.

#### -Recommend a solution for managing compliance

 
## Policy building blocks
 
**Policy definition vs initiative**
- A policy definition is one rule (e.g. "Require a tag on resource groups").
- An initiative (policy set) bundles several definitions into one unit: one assignment, one managed identity, one compliance score.
- Regulatory standards (CIS, ISO 27001, MCSB) are just large built-in initiatives.
- Exam: "apply many policies together and report as one" → initiative.
**Definition vs assignment**
- The definition/initiative is *what* the rules are. It's saved at a definition location (e.g. mg-sandbox) and does nothing on its own.
- The assignment is *where and how* the rules apply: scope, enforcement mode, managed identity, exclusions, parameter values, non-compliance message.
- One initiative can have many assignments (e.g. Audit in sandbox, Deny in landing zone).
- Exemptions and remediation tasks attach to the assignment, not the definition.
- The definition location limits where it can be assigned: saved at mg-sandbox means it can only be assigned at mg-sandbox and below. Save at mg-lab to reuse it everywhere.
**Policy parameters vs initiative parameters**
- Policy parameters are inputs to each individual policy (tag name, effect). "Set value" fixes them inside the initiative.
- Initiative parameters are filled in at assignment time. Use "Use initiative parameter" to defer a value, so one initiative behaves differently per scope, and one parameter can feed several policies.
- Exam: "reuse one initiative across environments with different values" → initiative parameters.
## Assignment settings
 
**Enforcement mode**
| Portal | API value | Behaviour |
|---|---|---|
| Default (enforcement) | Default | Effects applied – Deny blocks, Modify changes |
| Do not enforce | DoNotEnforce | Evaluates and reports only – use to test before enforcing |
| Enroll (preview) | Enroll | Enforces only on scopes added via Policy Enrollments (CLI/ARM/Bicep/AzAPI). Not exam material |
 
**Exclusions vs exemptions**
| | Exclusion | Exemption |
|---|---|---|
| Set on | The assignment (notScopes) | Separate object linked to the assignment |
| Expiry | None – permanent | Optional expiry date |
| Reason recorded | No | Yes – Waiver or Mitigated, plus description |
| Compliance view | Not evaluated at all | Shows as Exempt |
- Time-limited, auditable break for one team → exemption (Waiver + expiry).
**Effect values: `deny` vs `Deny`**
- Effects aren't case-sensitive. Some built-ins list both spellings for backwards compatibility. Use `Deny`.
## How effects behave
 
**Deny**
- Blocks create and update requests only. It does not change or remove existing non-compliant resources – they just show as non-compliant.
- Proved: with enforcement on, creating a storage account with anonymous blob access failed at validation.
**Modify / "Inherit a tag from the resource group if missing"**
- Only applies on create/update. Existing resources need a **remediation task**.
- Needs a **managed identity** with a role that can write tags (e.g. Tag Contributor). The portal creates it when assigning – but creating the role assignment needs Owner or User Access Administrator, not Resource Policy Contributor.
- Only flags a resource when the RG **has** the tag and the resource **doesn't**. If the RG has no tag, there's nothing to inherit, so the resource shows compliant.
- Design pattern: pair *Require a tag on resource groups* (Deny) with *Inherit a tag from the RG* (Modify). Deny guarantees the parent has the tag; Inherit copies it down. Neither works well alone. (My existing `env` tag setup on mg-sandbox already works this way.)
- "Create remediation task" is greyed out when no resources are non-compliant with a Modify/DINE policy.

## Compliance scans
 
- New or updated resources: evaluated within about 15 minutes.
- Full standard scan: every 24 hours.
- On demand: `az policy state trigger-scan --resource-group <rg>` (or `--subscription`). Queued on Microsoft's side, takes 5–15+ minutes. It re-evaluates **every** assignment in scope (CIS, MCSB etc.), not just mine.
- **"Compliant" with 0 total resources = not evaluated yet**, not passed. Always check the Total resources column.
## Defender for Cloud
 
**Resource provider registration**
- Services need their resource provider (here `Microsoft.Security`) registered per subscription. Check under Subscriptions → Resource providers. Registering is free.
- "Unregistered" in Environment settings can lag even after the provider is registered – click into the subscription to check.
**Adding a regulatory standard (CIS)**
- Path: Defender for Cloud → Environment settings → subscription → Security policies → Standards → toggle the standard On (pick the highest version).
- This creates an Azure Policy assignment of the CIS initiative (mostly Audit effects), which then shows under Policy → Compliance.
- Worked on the **free Foundational CSPM** tier – no paid plan needed to add it. Paid Defender CSPM adds extras (attack paths, cloud security explorer).
- The Regulatory compliance dashboard can take up to 24 hours to fill in.
- Policy *enforces* guardrails; Defender for Cloud *measures* posture against frameworks. Same engine underneath.
## Proving Microsoft's compliance
 
- **Service Trust Portal** (servicetrust.microsoft.com) → Certifications, Reports and Resources → ISO/IEC.
- Exam trap: Policy and Defender check *your* resources. Proof of *Microsoft's* certifications (ISO 27001, SOC reports) → Service Trust Portal.


#### -Recommend a solution for identity governance
