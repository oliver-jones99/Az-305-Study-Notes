# Design identity, governance, and monitoring solutions (25–30%)

## Resources:

-

## Design solutions for logging and monitoring

#### -Recommend a logging solution

#### -Recommend a solution for routing logs

#### -Recommend a monitoring solution

## Design authentication and authorization solutions

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

#### -Recommend an authentication solution

#### -Recommend an identity management solution

#### -Recommend a solution for authorizing access to Azure resources

#### -Recommend a solution for authorizing access to on-premises resources

#### -Recommend a solution to manage secrets, certificates, and keys

## Design governance

#### -Recommend a structure for management groups, subscriptions, and resource groups, and a strategy for resource tagging

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

#### -Recommend a solution for identity governance
