# Architecture and design notes

## Goal
Deliver a production-shaped, AI-assisted support case-management system on the
Microsoft Power Platform that takes an enquiry from natural language to a
structured, routed, reported case, entirely on free tiers.

## Design principles
- **AI where it adds judgement, deterministic logic everywhere else.** The
  Copilot Studio agent handles language understanding, classification and
  drafting. Everything that must be reliable and auditable (record creation,
  routing, escalation, status) is deterministic Power Automate. This keeps the
  automation predictable and free of paid AI consumption.
- **Separation of concerns.** Intake (agent) is decoupled from persistence
  (Create case flow) which is decoupled from side effects (routing flow). Each
  can be changed without touching the others.
- **Cost as a constraint, not an afterthought.** Every component was chosen to
  stay at £0: Copilot Studio free trial for AI, free standard connectors for
  automation, Gmail for email, Power BI Desktop for analytics.

## Component decisions
| Decision | Rationale |
|---|---|
| Copilot Studio agent for intake | Natural-language understanding, severity inference and drafting are exactly where an LLM adds value; the trial is free for build and test. |
| Deterministic Create case flow as the agent's tool | Keeps data writes reliable and keeps AI consumption out of Power Automate; the agent fills typed inputs, the flow does the write. |
| "Added" trigger only on the routing flow | The flow updates the case (status), so triggering on modify as well would create a self-trigger loop. |
| Gmail connector for email | The Mail connector is restricted on new tenants and Office 365 Outlook needs a licensed mailbox the free admin does not have. |
| Choice columns mapped in the flow | Dataverse stores integers behind choice labels; mapping label to value in the deterministic flow keeps the agent contract simple (it passes labels). |
| Solution-based ALM | A managed solution export makes the whole system deployable to another environment in one import. |

## Severity model
Priority is inferred from business impact, not from the customer using the word
"urgent":
- **High**: blocking the customer or causing loss or risk now (no access, outage,
  wrong or double charge, suspected fraud or compromised account, lost data,
  failed transaction, legal threat, hard deadline).
- **Medium**: a real fault with a workaround or not blocking now.
- **Low**: questions, minor or cosmetic requests, how-to asks that still need a
  human.
A calm tone does not downgrade a serious issue, and strong language does not
upgrade a minor one.

## Data model
- Customer 1..* Case (Customer lookup on Case).
- Case 1..* Case Note (Case lookup on Case Note).
- Case *..1 User (Assigned Agent lookup).
Choice columns: Category (Billing/Technical/Account/Complaint/Other, values 1-5),
Priority (Low/Medium/High, values 1-3), Status (New/In Progress/Resolved,
values 1-3).

## Security model
Four personas with least-privilege Dataverse roles (Customer, Agent, Team Lead,
Admin), each mapped to an Entra security group.

## Known trade-offs and planned work
- The Create case flow currently always creates a customer; find-or-create by
  email is the next enhancement.
- The routing flow assumes a customer is present; a null-customer guard is
  planned.
