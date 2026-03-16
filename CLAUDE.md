# Salesforce Agentforce Agent Migration

This project migrates Agentforce agents and all their dependencies between Salesforce orgs using the Salesforce CLI (`sf`).

## Prerequisites

### Salesforce CLI
The migration requires the Salesforce CLI (`sf`). Check if it's installed:
```bash
sf --version
```

If not installed, install it with one of these methods:

**macOS (Homebrew):**
```bash
brew install sf
```

**npm (any platform):**
```bash
npm install -g @salesforce/cli
```

**Windows (installer):**
Download from https://developer.salesforce.com/tools/salesforcecli

After installing, verify it works:
```bash
sf --version
```

### GitHub CLI (optional, for repo operations)
```bash
brew install gh   # macOS
```

## What Gets Migrated

- **Agents**: Hybrid_Reasoning_Agent, Basic_Agent (EinsteinServiceAgent type, Atlas ConcurrentMultiAgentOrchestration)
- **Topics**: Topic Selector, Order_Management, Escalation, Off Topic, Ambiguous Question
- **GenAi Functions (18)**: Get_Loyalty_Tier, Lookup_Order, Lookup_Current_Order, Create_Return, Create_Case, Inform_Merchant, Get_Customer_Profile, Get_Storefronts_By_Name, Get_Storefronts_By_Account, Get_Active_Menus, Get_Menu_Items, Update_Menu_Item, Update_Menu_Item_Price, Update_Storefront_Hours, Update_Storefront_Details, Summarize_Reviews, Create_Promotion, Update_Promotion_Status
- **GenAi Plugins (6)**: Customer_Identification, Storefront_Management, Menu_Management, Pricing_Management, Promotions_Management, Reviews_Management
- **Flows (14)**: lookup_order, lookup_current_order, create_return, create_case, get_loyalty_tier, inform_merchant, Issue_Refund, Add_Delivery_Instructions, Update_Contact_Record, Route_Merchant_to_Queue, Personalized_Recommendations, Send_Email_with_Verification_Code, Verify_Code, Route_to_Merchant_Support_Agent
- **Apex Classes (12)**: AgentCustomerActions, AgentStorefrontActions, AgentGetStorefrontsByAccountActions, AgentGetActiveMenusActions, AgentGetMenuItemsActions, AgentUpdateMenuItemActions, AgentUpdateMenuItemPriceActions, AgentUpdateStorefrontHoursActions, AgentUpdateStorefrontDetailsActions, AgentSummarizeReviewsActions, AgentCreatePromotionActions, AgentUpdatePromotionStatusActions
- **Custom Objects (7)**: Storefront__c, Menu__c, Menu_Item__c, Promotion__c, Refund__c, Review__c, Storefront_Hours_of_Operation__c
- **Custom Fields**: Contact.Loyalty_Tier__c (Picklist: Basic, Pronto One)
- **Page Layout**: Contact Layout with Loyalty_Tier__c field
- **Permission Set**: Loyalty_Tier_Access (FLS for Loyalty_Tier__c)
- **Data**: Alex Morgan Contact (alex.morgan@example.com, Loyalty_Tier__c = "Basic", Account = "Morgan Household - 10039847")

## Migration Steps

### Step 0: Authenticate Both Orgs
```bash
sf org login web --alias source-org   # Org migrating FROM
sf org login web --alias target-org   # Org migrating TO
```

### Step 1: Retrieve from Source Org

#### Bots
```bash
sf project retrieve start --metadata "Bot:Hybrid_Reasoning_Agent" "Bot:Basic_Agent" --target-org source-org
```

#### GenAi Functions, Plugins, Flows (use manifest/package.xml with API v62.0)
```xml
<?xml version="1.0" encoding="UTF-8"?>
<Package xmlns="http://soap.sforce.com/2006/04/metadata">
    <types><members>*</members><name>GenAiFunction</name></types>
    <types><members>*</members><name>GenAiPlugin</name></types>
    <types><members>*</members><name>Flow</name></types>
    <version>62.0</version>
</Package>
```
```bash
sf project retrieve start --manifest manifest/package.xml --target-org source-org
```

#### Custom Objects, Fields, Apex Classes
```bash
sf project retrieve start --metadata \
  "CustomObject:Storefront__c" "CustomObject:Menu__c" "CustomObject:Menu_Item__c" \
  "CustomObject:Promotion__c" "CustomObject:Refund__c" "CustomObject:Review__c" \
  "CustomObject:Storefront_Hours_of_Operation__c" \
  "CustomField:Contact.Loyalty_Tier__c" \
  "ApexClass:AgentCustomerActions" "ApexClass:AgentStorefrontActions" \
  "ApexClass:AgentGetStorefrontsByAccountActions" "ApexClass:AgentGetActiveMenusActions" \
  "ApexClass:AgentGetMenuItemsActions" "ApexClass:AgentUpdateMenuItemActions" \
  "ApexClass:AgentUpdateMenuItemPriceActions" "ApexClass:AgentUpdateStorefrontHoursActions" \
  "ApexClass:AgentUpdateStorefrontDetailsActions" "ApexClass:AgentSummarizeReviewsActions" \
  "ApexClass:AgentCreatePromotionActions" "ApexClass:AgentUpdatePromotionStatusActions" \
  --target-org source-org
```

#### Authoring Bundles (Agent Scripts)
List available versions first, then retrieve the latest:
```bash
sf org list metadata --metadata-type AiAuthoringBundle --target-org source-org
sf project retrieve start --metadata "AiAuthoringBundle:Hybrid_Reasoning_Agent_3" "AiAuthoringBundle:Basic_Agent_4" --target-org source-org
```

#### Page Layout
```bash
sf project retrieve start --metadata "Layout:Contact-Contact Layout" --target-org source-org
```

### Step 2: Discover Dependencies
Scan for custom object/field and Apex class references to make sure nothing is missing:
```bash
grep -roh '[A-Za-z_]*__c' force-app/main/default/flows/ | sort -u
grep -roh '[A-Za-z_]*__c' force-app/main/default/classes/ | sort -u
grep -rh "invocationTarget" force-app/main/default/genAiFunctions/ | sort -u
```

### Step 3: Prepare Metadata for Target Org

#### Remove org-specific botUser from Bot XML
In both `bots/*/Agent.bot-meta.xml`, delete the `<botUser>...</botUser>` line entirely.

#### Fix empty BotVersion XML files
EinsteinServiceAgent BotVersion files retrieve as nearly empty. Replace contents with:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<BotVersion xmlns="http://soap.sforce.com/2006/04/metadata">
    <fullName>vN</fullName>
    <entryDialog>Agent_Name_Main</entryDialog>
    <botDialogs>
        <botSteps>
            <stepIdentifier>step_1</stepIdentifier>
            <type>Wait</type>
        </botSteps>
        <developerName>Agent_Name_Main</developerName>
        <label>Main</label>
        <showInFooterMenu>false</showInFooterMenu>
    </botDialogs>
</BotVersion>
```
Replace `vN` with the version number (e.g. v2, v3) and `Agent_Name` with the bot developer name.

#### Update Agent Script (.agent) files
In each `aiAuthoringBundles/*/Agent.agent`:
1. **Update `default_agent_user`** to a valid EinsteinServiceAgent user in the target org:
   ```bash
   sf data query --query "SELECT Username FROM User WHERE Profile.Name LIKE '%Einstein%'" --target-org target-org
   ```
2. **Remove `knowledge:` section** and any FAQ/Knowledge topic — the `rag_feature_config_id` is org-specific
3. **Remove variables** that reference knowledge (e.g. knowledgeSummary, citations) if FAQ topic was removed

#### Add Loyalty_Tier__c to Contact Page Layout
In the Contact Layout XML, add inside the Contact Information `<layoutColumns>`:
```xml
<layoutItems>
    <behavior>Edit</behavior>
    <field>Loyalty_Tier__c</field>
</layoutItems>
```

#### Create Permission Set for FLS
Create `permissionsets/Loyalty_Tier_Access.permissionset-meta.xml`:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<PermissionSet xmlns="http://soap.sforce.com/2006/04/metadata">
    <fieldPermissions>
        <editable>true</editable>
        <field>Contact.Loyalty_Tier__c</field>
        <readable>true</readable>
    </fieldPermissions>
    <hasActivationRequired>false</hasActivationRequired>
    <label>Loyalty Tier Access</label>
</PermissionSet>
```

### Step 4: Deploy to Target Org (in dependency order)

#### Phase 1: Custom Objects + Apex Classes
```bash
sf project deploy start --source-dir force-app/main/default/objects --source-dir force-app/main/default/classes --target-org target-org
```

#### Phase 2: Flows
```bash
sf project deploy start --source-dir force-app/main/default/flows --target-org target-org
```

#### Phase 3: GenAi Functions + Plugins
```bash
sf project deploy start --source-dir force-app/main/default/genAiFunctions --source-dir force-app/main/default/genAiPlugins --target-org target-org
```

#### Phase 4: Bots
```bash
sf project deploy start --source-dir force-app/main/default/bots --target-org target-org
```

#### Phase 5: Authoring Bundles (deploy then publish)
```bash
sf project deploy start --source-dir force-app/main/default/aiAuthoringBundles --target-org target-org

sf agent publish authoring-bundle --api-name Hybrid_Reasoning_Agent --target-org target-org --skip-retrieve
sf agent publish authoring-bundle --api-name Basic_Agent --target-org target-org --skip-retrieve
```

#### Phase 6: Layout + Permission Set
```bash
sf project deploy start --source-dir force-app/main/default/layouts --source-dir force-app/main/default/permissionsets --target-org target-org
```

#### Phase 7: Assign Permission Set
```bash
sf org assign permset --name Loyalty_Tier_Access --target-org target-org
```

### Step 5: Migrate Data

#### Check/Create Alex Morgan Contact
```bash
# Check if Account exists
sf data query --query "SELECT Id FROM Account WHERE Name = 'Morgan Household - 10039847'" --target-org target-org

# Check if Contact exists
sf data query --query "SELECT Id, Loyalty_Tier__c FROM Contact WHERE FirstName = 'Alex' AND LastName = 'Morgan'" --target-org target-org

# Create if missing (replace ACCOUNT_ID)
sf data create record --sobject Contact --values "FirstName=Alex LastName=Morgan Email=alex.morgan@example.com Loyalty_Tier__c=Basic AccountId=ACCOUNT_ID" --target-org target-org

# Or update if exists but Loyalty_Tier__c is null (replace CONTACT_ID)
sf data update record --sobject Contact --record-id CONTACT_ID --values "Loyalty_Tier__c=Basic" --target-org target-org
```

### Step 6: Verify Migration
```bash
# Bots
sf data query --query "SELECT DeveloperName, MasterLabel FROM BotDefinition WHERE DeveloperName IN ('Hybrid_Reasoning_Agent','Basic_Agent')" --target-org target-org

# Planners with topics
sf data query --query "SELECT DeveloperName, MasterLabel, PlannerType FROM GenAiPlannerDefinition WHERE DeveloperName LIKE '%Hybrid_Reasoning%' OR DeveloperName LIKE '%Basic_Agent%'" --target-org target-org

# Topic associations
sf data query --query "SELECT PlannerId, Plugin FROM GenAiPlannerFunctionDef WHERE PlannerId IN (SELECT Id FROM GenAiPlannerDefinition WHERE DeveloperName LIKE '%Hybrid_Reasoning%' OR DeveloperName LIKE '%Basic_Agent%')" --target-org target-org

# Contact data + field visibility
sf data query --query "SELECT FirstName, LastName, Email, Loyalty_Tier__c FROM Contact WHERE FirstName = 'Alex' AND LastName = 'Morgan'" --target-org target-org
```

## Key Gotchas

1. **GenAiPlannerBundle retrieval times out** — Don't use GenAiPlannerBundle metadata type. Use AiAuthoringBundle (Agent Script) instead. `sf agent publish authoring-bundle` creates all planner/topic/action config from the .agent file.
2. **BotVersion XML is empty for EinsteinServiceAgent** — Must manually add entryDialog + botDialogs with a stepIdentifier or deploy will fail with "Required field is missing: entryDialog".
3. **botUser is org-specific** — Remove from Bot XML. Update in Agent Script to a valid EinsteinServiceAgent user in target org.
4. **Knowledge/RAG config is org-specific** — `rag_feature_config_id` won't exist in target. Remove knowledge section + FAQ topic from Agent Script, reconfigure in target org Setup.
5. **Deploy order matters** — Custom Objects → Apex → Flows → GenAi Functions/Plugins → Bots → Authoring Bundles (publish) → Layouts/PermSets.
6. **Field-Level Security** — Custom fields deploy but aren't visible without FLS. Create a permission set and assign it, or update the Admin profile.
7. **Page Layout** — Custom fields aren't automatically added to layouts. Update the Contact Layout XML to include the field.
8. **Authoring bundle versions auto-increment** — List versions with `sf org list metadata --metadata-type AiAuthoringBundle` and pick the latest before retrieving.
9. **API version for metadata** — GenAiFunction/GenAiPlugin/Flow work at v62.0. Don't waste time trying GenAiPlannerBundle at any version — it always times out.
