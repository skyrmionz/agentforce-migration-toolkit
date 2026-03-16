# Agentforce Migration Toolkit

Migrate Agentforce agents and all their dependencies from one Salesforce org to another — including topics, actions, flows, Apex classes, custom objects, custom fields, page layouts, permissions, and data records.

Built for use with [Claude Code](https://claude.ai/claude-code), which reads the included `CLAUDE.md` playbook automatically and walks you through each step.

---

## What's Included

### Agents
| Agent | Type | Planner | Description |
|---|---|---|---|
| **Hybrid Reasoning Agent** | EinsteinServiceAgent | Atlas ConcurrentMultiAgentOrchestration | Customer support agent with order management, escalation, loyalty-gated returns, and conditional greeting logic |
| **Basic Agent** | EinsteinServiceAgent | Atlas ConcurrentMultiAgentOrchestration | Streamlined customer support agent with order management, escalation, and merchant notification |

### Topics (per Agent)
| Topic | Description |
|---|---|
| **Topic Selector** | Entry point — checks loyalty tier, routes user to the right topic |
| **Order Management** | Order lookup (current + by number), returns (loyalty-gated), merchant notification |
| **Escalation** | Transfer to live agent or create a support case |
| **Off Topic** | Redirects off-topic requests back to supported topics |
| **Ambiguous Question** | Asks user to clarify vague requests |

### Agent Actions (18 GenAi Functions)
| Action | Type | Description |
|---|---|---|
| Get_Loyalty_Tier | Flow | Get customer's loyalty tier |
| Lookup_Order | Flow | Look up order by order number |
| Lookup_Current_Order | Flow | Look up most recent order |
| Create_Return | Flow | Initiate a return (Pronto One members only) |
| Create_Case | Flow | Create a support case |
| Inform_Merchant | Flow | Notify merchant about a return |
| Get_Customer_Profile | Apex | Look up customer by email |
| Get_Storefronts_By_Name | Apex | Search storefronts by name |
| Get_Storefronts_By_Account | Apex | List storefronts for an account |
| Get_Active_Menus | Apex | Get active menus for a storefront |
| Get_Menu_Items | Apex | Get items from a menu |
| Update_Menu_Item | Apex | Update a menu item |
| Update_Menu_Item_Price | Apex | Update menu item pricing |
| Update_Storefront_Hours | Apex | Update storefront hours |
| Update_Storefront_Details | Apex | Update storefront details |
| Summarize_Reviews | Apex | Summarize reviews for a storefront |
| Create_Promotion | Apex | Create a new promotion |
| Update_Promotion_Status | Apex | Update promotion status |

### Action Groups (6 GenAi Plugins)
Customer Identification, Storefront Management, Menu Management, Pricing Management, Promotions Management, Reviews Management

### Flows (14)
`lookup_order`, `lookup_current_order`, `create_return`, `create_case`, `get_loyalty_tier`, `inform_merchant`, `Issue_Refund`, `Add_Delivery_Instructions`, `Update_Contact_Record`, `Route_Merchant_to_Queue`, `Personalized_Recommendations`, `Send_Email_with_Verification_Code`, `Verify_Code`, `Route_to_Merchant_Support_Agent`

### Apex Classes (12)
`AgentCustomerActions`, `AgentStorefrontActions`, `AgentGetStorefrontsByAccountActions`, `AgentGetActiveMenusActions`, `AgentGetMenuItemsActions`, `AgentUpdateMenuItemActions`, `AgentUpdateMenuItemPriceActions`, `AgentUpdateStorefrontHoursActions`, `AgentUpdateStorefrontDetailsActions`, `AgentSummarizeReviewsActions`, `AgentCreatePromotionActions`, `AgentUpdatePromotionStatusActions`

### Custom Objects (7)
`Storefront__c`, `Menu__c`, `Menu_Item__c`, `Promotion__c`, `Refund__c`, `Review__c`, `Storefront_Hours_of_Operation__c`

### Custom Fields
- `Contact.Loyalty_Tier__c` — Picklist (Basic, Pronto One)

### Page Layout & Permissions
- Contact Layout updated with Loyalty Tier field
- `Loyalty_Tier_Access` permission set for field-level security

### Data Record
- **Alex Morgan** — Contact (alex.morgan@example.com, Loyalty Tier: Basic, Account: Morgan Household - 10039847)

---

## Quick Start

### 1. Install Prerequisites

**Salesforce CLI:**
```bash
# macOS
brew install sf

# npm (any platform)
npm install -g @salesforce/cli
```

**Claude Code:**
Follow the install instructions at [claude.ai/claude-code](https://claude.ai/claude-code)

### 2. Clone This Repo
```bash
git clone https://github.com/skyrmionz/agentforce-migration-toolkit.git
cd agentforce-migration-toolkit
```

### 3. Open Claude Code
```bash
claude
```
Claude will automatically read the `CLAUDE.md` playbook in this directory.

### 4. Tell Claude to Migrate
Say something like:
> Migrate these agents to my org

Claude will guide you through:
1. Authenticating your target org
2. Preparing the metadata (updating agent user, removing org-specific references)
3. Deploying in the correct dependency order
4. Publishing the agent authoring bundles (creates topics and action assignments)
5. Migrating the Alex Morgan contact record
6. Setting up field-level security and page layouts
7. Verifying everything works

---

## Manual Migration (Without Claude Code)

If you prefer to run the steps yourself, see the full playbook in [`CLAUDE.md`](CLAUDE.md). The key deployment order is:

1. Custom Objects + Apex Classes
2. Flows
3. GenAi Functions + GenAi Plugins
4. Bot Definitions
5. Authoring Bundles (deploy + `sf agent publish authoring-bundle`)
6. Page Layout + Permission Set (deploy + `sf org assign permset`)
7. Contact Data (`sf data create record` / `sf data update record`)

---

## Source Org

These agents were originally built in an OrgFarm EPIC org. The metadata in this repo is retrieved and ready to deploy — no access to the source org is needed.

## Key Things to Know

- **Agent user is org-specific** — The `.agent` script files reference a `default_agent_user` that must be updated to a valid EinsteinServiceAgent user in your target org before publishing.
- **Knowledge/FAQ is org-specific** — The Hybrid Reasoning Agent originally had a FAQ topic with RAG-based knowledge search. This was removed because the `rag_feature_config_id` is org-specific. Reconfigure knowledge in your target org's Setup if needed.
- **Deploy order matters** — Custom Objects must go before Flows (which reference them), which must go before GenAi Functions (which invoke them), etc.
- **Authoring bundle publish is critical** — Simply deploying the Bot metadata doesn't create the planner/topic configuration. You must run `sf agent publish authoring-bundle` to create the full agent with topics, instructions, and action assignments.
- **Field-level security** — Custom fields deploy but aren't visible without FLS. The included `Loyalty_Tier_Access` permission set handles this.
