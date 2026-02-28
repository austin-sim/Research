# Copilot Studio Agent with Salesforce Integration

> **Research Piece 2 of 2** — Schema-grounded SOQL generation, secure connector architecture, and end-to-end implementation guide for a Copilot Studio agent connected to Salesforce CRM.

---

## Table of Contents

1. [Background](#background)
2. [Section 5 — Salesforce Connector Architecture in Power Platform](#section-5--salesforce-connector-architecture-in-power-platform)
   - [5.1 Authentication Setup](#51-authentication-setup)
   - [5.2 Connector Capabilities and Limits](#52-connector-capabilities-and-limits)
   - [5.3 Dynamic SOQL Generation Risks](#53-dynamic-soql-generation-risks)
3. [Section 6 — Schema as Grounded Knowledge](#section-6--schema-as-grounded-knowledge)
   - [6.1 Schema Extraction](#61-schema-extraction)
   - [6.2 Knowledge Source Configuration in Copilot Studio](#62-knowledge-source-configuration-in-copilot-studio)
   - [6.3 SOQL Query Construction Pattern](#63-soql-query-construction-pattern)
   - [6.4 Handling Missing or Ambiguous Schema References](#64-handling-missing-or-ambiguous-schema-references)
4. [Section 7 — Security for the Salesforce Agent](#section-7--security-for-the-salesforce-agent)
   - [7.1 Preventing Data Over-Exposure](#71-preventing-data-over-exposure)
   - [7.2 Preventing SOQL Injection](#72-preventing-soql-injection)
   - [7.3 Scoping the Agent's Salesforce Access](#73-scoping-the-agents-salesforce-access)
5. [Section 8 — End-to-End Implementation Checklist](#section-8--end-to-end-implementation-checklist)
6. [Sources & Further Reading](#sources--further-reading)

---

## Background

The core problem: a Copilot Studio agent connected to Salesforce via the native Power Platform connector can query data, but without grounded knowledge of the Salesforce schema it will hallucinate field API names, guess object relationships incorrectly, and fail for non-technical users who do not know to phrase queries in Salesforce terminology.

The root cause is that **an LLM's pre-training knowledge of Salesforce object schemas is generic** — it knows the standard objects (Account, Contact, Opportunity) and common standard fields, but it does not know your org's custom objects (e.g. `Project__c`), custom fields (e.g. `Renewal_Date__c`), your specific picklist values, or the relationships between your custom objects. When the user asks "show me all open deals for ACME Corp", the LLM may attempt `SELECT Id, Name FROM Opportunity WHERE AccountName = 'ACME Corp'` — which will fail because `AccountName` is not a valid field (the correct field is `Account.Name` via a relationship query, or `AccountId` with a sub-query).

The solution is to embed the **schema as grounded knowledge** and force the agent to consult it before constructing any SOQL query. This document details exactly how to do that.

---

## Section 5 — Salesforce Connector Architecture in Power Platform

### 5.1 Authentication Setup

#### Salesforce Connected App — Required OAuth Scopes

Create a dedicated Connected App in Salesforce for the Power Platform integration. Navigate to **Setup → App Manager → New Connected App**.

| OAuth Scope | API Name | Required | Reason |
|---|---|---|---|
| Access and manage your data (api) | `api` | ✅ Yes | Required for all SOQL queries and record operations |
| Perform requests at any time (refresh_token, offline_access) | `refresh_token` | ✅ Yes | Required for long-lived Power Platform connections |
| Perform ANSI SQL queries on Salesforce data (wave_api) | `wave_api` | ❌ No | Only if using Salesforce Analytics/Tableau |
| Access all data accessible by the integration user | `full` | ❌ No — avoid | Too broad; use `api` scope only |
| Access custom permissions | `custom_permissions` | ❌ No | Not needed for read-only data access |
| Manage user data via APIs | `id` | ⚠️ Optional | Needed only if the agent needs to resolve user identities |

**Minimum scope recommendation**: `api refresh_token offline_access`

Avoid the `full` scope — it is broader than necessary and violates the principle of least privilege. The `api` scope is sufficient for all SOQL queries, record reads, and metadata API calls.

#### Connected App Security Settings

```text
Permitted Users: Admin approved users are pre-authorized
IP Relaxation: Relax IP restrictions (OR: Enforce IP restrictions and add Power Platform IP ranges)
Refresh Token Policy: Refresh token is valid until revoked
Require Secret for Refresh Token Flow: ✅ Enabled
Enable OAuth Settings: ✅ Enabled
Callback URL: https://global.consent.azure-apim.net/redirect
              https://login.microsoftonline.com/common/oauth2/nativeclient
```

#### Connection References (Shareable, Managed) vs. Per-User Connections

| Type | Description | Recommended for agents |
|---|---|---|
| Per-user connection | Each user authenticates with their own Salesforce credentials. The agent runs queries as the individual user — respecting that user's Salesforce permissions. | ✅ Yes, for agents where per-user data isolation is required |
| Connection reference (shared) | A single service account credential is shared across all users of the agent. All queries run as the integration user. | ✅ Yes, for read-only reporting agents where all authorised users should see the same data scope |

**Recommendation for this use case**: Use a **connection reference** backed by a dedicated Salesforce integration user. This gives you:
- Consistent, auditable query behaviour (all queries from one identity)
- No risk of the agent inheriting a named user's temporary elevated permissions
- Ability to rotate credentials without impacting individual users
- A single point of audit in Salesforce (all API calls attributed to the integration user)

#### Storing the Client Secret Securely

The Connected App's client secret should never be embedded in a Power Automate flow or Copilot Studio configuration. Use Azure Key Vault:

```text
1. Create an Azure Key Vault in your subscription (region: same as Power Platform environment)
2. Add the Salesforce Connected App client secret as a Key Vault Secret
3. Grant the Power Platform Managed Identity "Key Vault Secrets User" role on the vault
4. In Power Automate: use the Azure Key Vault connector's "Get secret" action to retrieve 
   the secret at runtime — do NOT store it in a flow variable that persists in run history
5. Rotate the secret every 90 days (set a Key Vault expiry alert)
```

> ⚠️ **Research gap**: As of early 2026, the Power Platform Salesforce connector stores OAuth tokens in the connection object within Power Platform infrastructure. Verify with your Microsoft account team whether these tokens are stored in Microsoft-managed encryption (they are, per the Power Platform security documentation) and whether the storage location meets your data residency requirements.

---

### 5.2 Connector Capabilities and Limits

#### What the Native Power Platform Salesforce Connector Supports

| Operation | Connector action | Notes |
|---|---|---|
| Query records | "Get records" / "Execute SOQL query" | Supports full SOQL including WHERE, ORDER BY, LIMIT, relationship queries |
| Get single record | "Get record" | Requires record ID |
| Create record | "Create record" | Not needed for read-only agent — disable via permission set |
| Update record | "Update record" | Not needed — disable |
| Delete record | "Delete record" | Not needed — disable |
| Get object metadata | "Get object metadata" | Returns field definitions for a given object — useful for schema validation |
| Describe global | Not directly available | Use Salesforce REST API via custom connector if needed |

#### SOQL Character Limits

Salesforce imposes limits on SOQL query length that interact with LLM-generated queries:

| Limit | Value |
|---|---|
| Maximum SOQL query length | 20,000 characters |
| Maximum number of characters in a field value in WHERE clause | 4,000 characters |
| Maximum fields in SELECT | 100 (for most objects) |
| Maximum records returned per query (without pagination) | 2,000 |
| Maximum records returned with LIMIT | Up to 50,000 (with Bulk API) |

LLM-generated SOQL for complex requests (multiple relationship traversals, many selected fields) can approach the 20,000 character limit. Implement a pre-execution check:

```python
def validate_soql_length(soql: str) -> bool:
    if len(soql) > 20000:
        raise ValueError(f"SOQL query exceeds maximum length: {len(soql)} characters")
    return True
```

#### Rate Limiting

Salesforce API call limits (Salesforce Enterprise Edition, per 24-hour rolling window):

| Edition | API calls per 24 hours |
|---|---|
| Enterprise | 1,000 per named user licence (minimum 500,000 for org) |
| Unlimited | Unlimited (with fair use) |
| Developer | 15,000 |

For agents used by many concurrent users, monitor API usage in **Setup → System Overview → API Usage**. Implement exponential backoff in Power Automate for 429 responses:

```text
Power Automate: Configure action retry policy
  Type: Exponential
  Count: 4
  Interval: PT5S (5 seconds initial)
  Maximum interval: PT1H
  Minimum interval: PT5S
```

> ⚠️ **Research gap**: The native Power Platform Salesforce connector API version should be verified. As of 2024, the connector used Salesforce API v55.0. Salesforce deprecates older API versions annually — check the Salesforce API lifecycle documentation and the connector's version at runtime using the "Get API version" capability or by inspecting connector metadata.

---

### 5.3 Dynamic SOQL Generation Risks

#### SOQL Injection

SOQL injection occurs when user-supplied input is interpolated directly into a SOQL string without escaping:

```
User input: "ACME Corp' OR Name != '"
Naive SOQL: SELECT Id, Name FROM Account WHERE Name = 'ACME Corp' OR Name != ''
Result: Returns ALL accounts — data exposure
```

A more dangerous example:
```
User input: "x' LIMIT 200 UNION SELECT Id, Password__c FROM User WHERE Id != '"
(Note: SOQL does not support UNION, but variations exist for nested queries)
```

**Mitigation**: Treat all user-supplied values as literals. Apply SOQL string escaping rules before interpolation (see Section 7.2).

#### Field-Level and Record-Level Security

The agent's LLM layer cannot be trusted as the authoritative access control enforcement point. Rely on Salesforce's native security model:

- **Field-Level Security (FLS)**: Salesforce will not return fields that the integration user does not have FLS read access to. The SOQL query will either omit the field silently or throw an error. Configure the integration user's permission set to exclude sensitive fields at the FLS level.
- **Sharing Rules and Record Visibility**: Salesforce's sharing model (OWD + sharing rules + role hierarchy) determines which records the integration user can see. The agent inherits these restrictions.
- **The agent is NOT the access control layer**: Even if the system prompt says "never return fields containing SSN", the LLM might ignore this instruction under a sufficiently crafted injection. FLS is the authoritative control.

---

## Section 6 — Schema as Grounded Knowledge

### 6.1 Schema Extraction

#### Extracting the Schema via Salesforce Metadata API

The Salesforce Metadata API allows retrieval of complete object definitions. For a Copilot Studio knowledge source, the most useful export is a combination of:
- Object API names and labels
- Field API names, labels, data types, and picklist values
- Lookup and master-detail relationship field targets
- Required fields (nillable = false)

**Method 1: Salesforce CLI (sfdx / sf)**

```bash
# Install Salesforce CLI
npm install -g @salesforce/cli

# Authenticate to your org
sf org login web --alias myorg

# Export all custom and standard object descriptions to JSON
sf sobject describe --sobject Account --target-org myorg --json > schema/Account.json
sf sobject describe --sobject Opportunity --target-org myorg --json > schema/Opportunity.json
sf sobject describe --sobject Contact --target-org myorg --json > schema/Contact.json
# Repeat for all objects the agent will query

# Or export all objects at once (large orgs may take several minutes):
sf sobject list --target-org myorg --json | \
  jq -r '.result[]' | \
  xargs -I{} sf sobject describe --sobject {} --target-org myorg --json > schema/all-objects.json
```

**Method 2: Salesforce REST API (for scheduled automation)**

```http
GET /services/data/v59.0/sobjects/{ObjectName}/describe
Authorization: Bearer {access_token}
```

```python
# Python script for scheduled schema export
import requests
import json

def export_object_schema(access_token: str, instance_url: str, object_name: str) -> dict:
    url = f"{instance_url}/services/data/v59.0/sobjects/{object_name}/describe"
    headers = {"Authorization": f"Bearer {access_token}"}
    response = requests.get(url, headers=headers)
    response.raise_for_status()
    return response.json()

def extract_relevant_fields(describe_result: dict) -> dict:
    """Extract only the fields relevant for LLM consumption."""
    return {
        "name": describe_result["name"],
        "label": describe_result["label"],
        "fields": [
            {
                "name": f["name"],
                "label": f["label"],
                "type": f["type"],
                "required": not f["nillable"] and not f["defaultedOnCreate"],
                "picklist_values": [v["value"] for v in f.get("picklistValues", []) if v["active"]],
                "reference_to": f.get("referenceTo", []),
                "relationship_name": f.get("relationshipName"),
            }
            for f in describe_result["fields"]
            if not f["deprecatedAndHidden"]
        ]
    }
```

#### Recommended Schema Document Format

For LLM consumption, Markdown tables strike the best balance between token efficiency and readability. Avoid raw JSON (high token consumption) and CSV (loses hierarchy for relationship fields).

**Recommended format** — one section per object:

```markdown
## Object: Opportunity (API: Opportunity)

**Description**: Represents an in-progress or closed sale.

### Fields

| Label | API Name | Type | Required | Notes |
|---|---|---|---|---|
| Opportunity Name | Name | Text(120) | ✅ | |
| Account | AccountId | Lookup(Account) | ✅ | Related: Account.Name |
| Close Date | CloseDate | Date | ✅ | |
| Stage | StageName | Picklist | ✅ | Values: Prospecting, Qualification, Proposal/Price Quote, Negotiation/Review, Closed Won, Closed Lost |
| Amount | Amount | Currency | ❌ | |
| Owner | OwnerId | Lookup(User) | ✅ | Related: Owner.Name |
| Renewal Date | Renewal_Date__c | Date | ❌ | Custom field |
| Deal Type | Deal_Type__c | Picklist | ❌ | Custom. Values: New Business, Renewal, Expansion |

### Relationship Queries

To get Account name: `SELECT Id, Name, Account.Name FROM Opportunity`
To get Owner name: `SELECT Id, Name, Owner.Name FROM Opportunity`
```

**Token consumption comparison** (for a 50-field object):

| Format | Approximate tokens | LLM readability | Machine parseability |
|---|:---:|:---:|:---:|
| Full JSON (Salesforce API response) | ~8,000 | Low | High |
| CSV | ~2,000 | Medium | Medium |
| Markdown table (recommended) | ~2,500 | High | Medium |
| Compressed JSON (API names only) | ~1,500 | Low | High |

#### Schema Refresh Automation

Automate schema updates using a scheduled Power Automate flow:

```text
Trigger: Recurrence — every 7 days (or after every Salesforce deployment)

Step 1: HTTP action → Salesforce REST API /services/data/v59.0/sobjects/{Object}/describe
        (Repeat for each object the agent queries)
Step 2: Compose action → Format as Markdown schema document (using the template above)
Step 3: SharePoint "Update file" action → Overwrite the schema document in the 
        knowledge source SharePoint library
Step 4: Copilot Studio trigger (if API available) → Invalidate the knowledge source 
        cache to force re-indexing of the updated document

Note: As of early 2026, triggering Copilot Studio knowledge re-indexing 
programmatically requires the Copilot Studio management API (preview).
```

> ⚠️ **Research gap**: Copilot Studio's knowledge source re-indexing latency after a SharePoint document update is not well-documented. In practice, re-indexing may take 15–60 minutes. For time-sensitive schema updates (e.g., immediately after a Salesforce deployment that adds new objects), manually trigger re-indexing via the Copilot Studio UI: **Knowledge → [source] → Sync now**.

---

### 6.2 Knowledge Source Configuration in Copilot Studio

#### Adding the Schema Document as a Knowledge Source

1. In Copilot Studio, navigate to your agent → **Knowledge** → **Add knowledge**
2. Select **SharePoint** (recommended) or **Files** (for initial testing)
3. For SharePoint: enter the URL of the SharePoint document library containing your schema Markdown file
4. Configure the knowledge source:
   - **Name**: `Salesforce Schema Reference`
   - **Description**: `Contains the API names, field definitions, picklist values, and relationship definitions for all Salesforce objects this agent can query. Always consult this source before constructing a SOQL query.`
5. Enable **Generative Answers** for this knowledge source (this forces the agent to cite the schema document when generating query content)

#### System Prompt Instruction for Schema Consultation

Add the following to the agent's **Instructions** (system prompt) in Copilot Studio:

```markdown
## SALESFORCE QUERY RULES

You have access to a knowledge source called "Salesforce Schema Reference". 
This document contains the exact API names of all Salesforce objects and fields 
in our organisation.

MANDATORY PROCESS for every Salesforce data request:

1. BEFORE writing any SOQL query, search the "Salesforce Schema Reference" knowledge 
   source to find:
   a. The exact API name of the Salesforce object (e.g. Opportunity, not "deals")
   b. The exact API names of all fields you intend to include in SELECT or WHERE
   c. The correct relationship traversal syntax for any related fields 
      (e.g. Account.Name, Owner.Name — NOT AccountName, OwnerName)

2. NEVER guess field API names. If you cannot find a field in the schema reference, 
   DO NOT include it in the query. Tell the user you could not find that field and 
   ask for clarification.

3. ALWAYS include a LIMIT clause. Default: LIMIT 200. Never exceed LIMIT 500 without 
   explicit user request and manager approval.

4. NEVER generate DELETE, UPDATE, INSERT, or MERGE SOQL statements.

5. NEVER include fields containing personal data (email addresses, phone numbers, 
   social security numbers) unless the user explicitly asks for them AND the 
   field is listed in the approved PII fields list in the schema reference.
```

#### Chunking Strategy for Large Schemas

Salesforce orgs with many custom objects can have schemas exceeding 100,000 tokens — far beyond what can fit in a single context window. The recommended chunking approach:

1. **Split by object**: Each Salesforce object (Account, Opportunity, Contact, etc.) gets its own section in the schema document, or its own separate document.
2. **Separate documents by domain**: Group related objects (e.g., all Sales objects together, all Service objects together). Create separate SharePoint documents per group.
3. **Add a "Schema Index" document**: A short (< 2,000 token) index that lists all objects and their 1-line descriptions. The agent retrieves the index first to determine which object-specific schema document to consult.

```markdown
# Salesforce Schema Index

## Sales Objects
- **Account** (Account): Company or organisation records. Fields: Name, Industry, Type, AnnualRevenue, etc.
- **Opportunity** (Opportunity): Sales deals, pipeline. Fields: Name, StageName, Amount, CloseDate, etc.
- **Contact** (Contact): Individual people at companies. Fields: FirstName, LastName, Email, Phone, etc.
- **Lead** (Lead): Unconverted prospects. Fields: FirstName, LastName, Company, Status, etc.

## Service Objects
- **Case** (Case): Support tickets. Fields: CaseNumber, Subject, Status, Priority, etc.
- **Solution** (Solution): Known issue resolutions.

## Custom Objects
- **Project** (Project__c): Internal projects linked to Accounts.
- **Renewal** (Renewal__c): Subscription renewal tracking for Opportunities.
```

The agent retrieves the index, identifies the relevant object(s), then retrieves only the object-specific schema sections needed for the query — dramatically reducing token consumption.

---

### 6.3 SOQL Query Construction Pattern

#### Six-Step Query Construction Process

This is the mandatory process the agent should follow for every Salesforce data request. Implement each step as a structured reasoning step in the agent's system prompt or as explicit Copilot Studio topics.

**Step 1: Intent Classification**

Before writing any SOQL, the agent must classify:
- What Salesforce object is the user asking about?
- What filter conditions apply? (e.g., "open deals" → StageName NOT IN ('Closed Won', 'Closed Lost'))
- What output format does the user need? (list, count, aggregate, specific fields)
- Is this a record-fetch query or an aggregate query?

```markdown
## SYSTEM PROMPT — STEP 1: INTENT CLASSIFICATION

When a user asks a Salesforce data question, first identify:
1. Object: Which Salesforce object? (consult the Schema Index if unsure)
2. Filters: What conditions narrow the results? 
3. Output: What fields does the user need to see?
4. Query type: Record retrieval (SELECT fields FROM...) or Aggregate (COUNT, SUM, etc.)?

Do not proceed to query construction until you have classified all four elements. 
If any element is ambiguous, ask a single clarifying question.
```

**Step 2: Schema Lookup**

```markdown
## SYSTEM PROMPT — STEP 2: SCHEMA LOOKUP

After classifying intent, retrieve the relevant object schema from the 
"Salesforce Schema Reference" knowledge source.

Specifically retrieve:
- The exact Object API name
- The exact API names for all fields you will use in SELECT and WHERE
- The relationship syntax for any related object fields (e.g. Account.Name)
- Valid picklist values for any picklist field you will use in WHERE

STOP HERE if the schema does not contain the object or field the user 
is asking about. Tell the user and ask for clarification. Never guess.
```

**Step 3: Draft SOQL**

```markdown
## SYSTEM PROMPT — STEP 3: DRAFT SOQL

Using ONLY the API names you verified in Step 2, draft the SOQL query.

Rules:
- SELECT must list explicit field names (never SELECT *)
- WHERE conditions must use verified API names and valid picklist values from schema
- Always include ORDER BY for consistency (typically ORDER BY CreatedDate DESC)
- Always include LIMIT (default: 200)
- Relationship traversal: use dot notation (Account.Name) not join syntax

Example:
User intent: "Show me open deals for ACME Corp over $50k"

Verified from schema:
- Object: Opportunity
- Name field: Name
- Account relationship: AccountId → Account.Name
- Stage field: StageName (picklist: Prospecting, Qualification, ..., Closed Won, Closed Lost)
- Amount field: Amount

Draft SOQL:
SELECT Id, Name, Account.Name, StageName, Amount, CloseDate
FROM Opportunity
WHERE Account.Name = 'ACME Corp'
  AND Amount > 50000
  AND StageName NOT IN ('Closed Won', 'Closed Lost')
ORDER BY CloseDate ASC
LIMIT 200
```

**Step 4: Validation**

Before executing, run the SOQL through a deterministic validation check:

```python
import re

BLOCKED_SOQL_PATTERNS = [
    r'\bDELETE\b',
    r'\bUPDATE\b',
    r'\bINSERT\b',
    r'\bMERGE\b',
    r'\bUPSERT\b',
    r'SELECT\s+\*',           # no wildcard select
    r'LIMIT\s+([0-9]{4,})',   # no limit > 9999
]

def validate_soql(soql: str) -> tuple[bool, str]:
    soql_upper = soql.upper().strip()
    
    # Must start with SELECT
    if not soql_upper.startswith('SELECT'):
        return False, "Query must start with SELECT"
    
    for pattern in BLOCKED_SOQL_PATTERNS:
        match = re.search(pattern, soql_upper)
        if match:
            return False, f"Blocked pattern detected: {match.group()}"
    
    # Must contain LIMIT
    if 'LIMIT' not in soql_upper:
        return False, "Query must include a LIMIT clause"
    
    # Extract LIMIT value and check it
    limit_match = re.search(r'LIMIT\s+(\d+)', soql_upper)
    if limit_match:
        limit_val = int(limit_match.group(1))
        if limit_val > 500:
            return False, f"LIMIT {limit_val} exceeds maximum allowed value of 500"
    
    return True, "OK"
```

**Step 5: Execute via Connector**

In Copilot Studio, implement execution as a Power Automate flow action:

```text
Copilot Studio Action: "Run Salesforce Query"
  Input: soql_query (string) — the validated SOQL string
  
Power Automate flow:
  Trigger: Copilot Studio action
  Step 1: Validate SOQL (call validation Azure Function)
  Step 2: If validation fails → return error message to agent → agent tells user
  Step 3: If validation passes → Salesforce connector "Get records" action (SOQL query)
  Step 4: Log SOQL query to Dataverse audit table (see Section 7.1)
  Step 5: Return results JSON to Copilot Studio agent
```

**Step 6: Format Results**

```markdown
## SYSTEM PROMPT — STEP 6: FORMAT RESULTS

When returning Salesforce query results to the user:

1. Never dump raw JSON — format as a clear, human-readable response
2. For record lists: use a Markdown table with human-readable column headers 
   (use the field Labels from the schema, not the API names)
3. For counts: state the count clearly in a sentence
4. If the result is empty: tell the user no records matched and suggest they 
   check their filter criteria
5. For large result sets (>20 records): show the first 10, summarise the rest,
   and offer to export the full results
6. Never include the SOQL query itself in the user-facing response unless the 
   user explicitly asks to see it
```

#### Aggregate vs. Record-Fetch Queries

| Query type | User intent examples | SOQL pattern | Notes |
|---|---|---|---|
| Record fetch | "Show me open deals", "List contacts at ACME" | `SELECT fields FROM Object WHERE ... LIMIT n` | Standard pattern |
| Count | "How many open cases?", "How many deals closed this month?" | `SELECT COUNT(Id) FROM Object WHERE ...` | Returns single row |
| Sum/Average | "What's the total pipeline value?", "Average deal size?" | `SELECT SUM(Amount), AVG(Amount) FROM Opportunity WHERE ...` | Returns single row |
| Group By | "Pipeline by stage", "Cases by priority this week" | `SELECT StageName, COUNT(Id), SUM(Amount) FROM Opportunity GROUP BY StageName` | Returns one row per group |

For aggregate queries, the LIMIT clause behaves differently — `LIMIT` on a GROUP BY query limits the number of groups returned, not records. Document this in the system prompt.

---

### 6.4 Handling Missing or Ambiguous Schema References

#### When the Object or Field Does Not Exist

```markdown
## SYSTEM PROMPT — HANDLING MISSING SCHEMA REFERENCES

If you search the "Salesforce Schema Reference" and cannot find the object or 
field the user is asking about:

1. DO NOT guess the API name
2. DO NOT proceed with query construction
3. Tell the user: "I couldn't find [field/object name] in the Salesforce schema. 
   Could you clarify what you're looking for? Here are the objects I have schema 
   information for: [list from index]"
4. Offer the closest match if obvious (e.g. "Did you mean the 'Amount' field on 
   Opportunity?")
```

#### Common Object Alias Disambiguation

Build a disambiguation mapping into the system prompt:

```markdown
## COMMON BUSINESS TERM → SALESFORCE OBJECT MAPPING

When users use business language, map it to the correct Salesforce object:

| User says | Salesforce object | Notes |
|---|---|---|
| "deals", "opportunities", "pipeline" | Opportunity | |
| "companies", "customers", "clients", "accounts" | Account | |
| "people", "contacts", "contacts at..." | Contact | |
| "leads", "prospects", "inbound leads" | Lead | |
| "cases", "tickets", "support requests" | Case | |
| "tasks", "to-dos", "follow-ups" | Task | |
| "meetings", "events", "calls" | Event | |
| "users", "reps", "sales reps", "account executives" | User | Use Owner.Name in queries, not direct User query |
| "products", "items", "SKUs" | Product2 | |
| "quotes" | Quote | If CPQ is installed, may be SBQQ__Quote__c |
| "invoices" | (check schema) | Not a standard object — likely custom |

If the user's term does not appear in this mapping, search the Schema Index 
before asking for clarification.
```

---

## Section 7 — Security for the Salesforce Agent

### 7.1 Preventing Data Over-Exposure

#### LIMIT Clause Enforcement

The agent must always apply a LIMIT clause. Implement this at two layers:

1. **System prompt**: "Always include LIMIT 200 unless the user explicitly requests more."
2. **Validation function** (Step 4 above): Reject any SOQL that does not include LIMIT, and cap at 500.

Salesforce can return up to 2,000 records in a single synchronous API response. Without a LIMIT, the agent could expose thousands of records in a single response — both a data exposure risk and a token cost risk.

#### PII Field Blocklist

Implement a PII field blocklist in the system prompt AND in Salesforce FLS:

```markdown
## SYSTEM PROMPT — PII FIELD RESTRICTIONS

The following fields contain personally identifiable information (PII). 
NEVER include these fields in a SOQL SELECT clause unless explicitly 
required and the user has a documented business need:

Contact: Email, Phone, MobilePhone, HomePhone, OtherPhone, MailingAddress, 
         OtherAddress, Birthdate
Lead: Email, Phone, MobilePhone, Street, PostalCode
User: Email, MobilePhone, Street, PostalCode
Account: BillingAddress, ShippingAddress, Phone (treat as confidential)

If a user asks for PII fields, ask: "This field contains personal information. 
Can you confirm why you need this data and that you are authorised to access it?"
```

In Salesforce, configure FLS on the integration user's permission set to **remove read access** to PII fields that the agent should never see. The system prompt is a soft control; FLS is a hard control.

#### Audit Logging SOQL Queries to Dataverse

Every SOQL query executed by the agent should be logged:

```text
Dataverse table: agent_soql_audit_log (custom table)
Columns:
  - timestamp (DateTime, required)
  - user_id (Lookup to User, required) — the Copilot Studio user
  - soql_query (Multi-line Text, required)
  - execution_status (Choice: Success, Failed, Blocked, required)
  - record_count_returned (Whole Number)
  - error_message (Text, optional)
  - conversation_id (Text — links to msdyn_conversationtranscript)
```

In Power Automate, add a Dataverse "Create a new row" action before and after the Salesforce query step to capture both the attempted query and the result.

---

### 7.2 Preventing SOQL Injection

#### SOQL String Literal Escaping Rules

SOQL string literals are delimited by single quotes. The following characters must be escaped in user-supplied values before interpolation:

| Character | Escaped form | Notes |
|---|---|---|
| Single quote `'` | `\'` | Primary injection vector |
| Backslash `\` | `\\` | Escape character itself |
| Null character | Not allowed | Strip from input |
| Line feed `\n` | `\n` | Allowed but should be stripped for WHERE clauses |
| Carriage return `\r` | `\r` | Same |

```python
def escape_soql_string(value: str) -> str:
    """
    Escape a user-supplied string value for safe interpolation into a SOQL 
    string literal. Enclose the result in single quotes.
    """
    if not isinstance(value, str):
        raise TypeError(f"Expected string, got {type(value)}")
    # Escape backslashes first (must be before quote escaping)
    value = value.replace('\\', '\\\\')
    # Escape single quotes
    value = value.replace("'", "\\'")
    # Strip null bytes
    value = value.replace('\x00', '')
    return f"'{value}'"

# Usage:
user_input = "ACME Corp' OR Name != '"
safe_value = escape_soql_string(user_input)
# Result: "'ACME Corp\\' OR Name != \\''" — safe for interpolation
soql = f"SELECT Id, Name FROM Account WHERE Name = {safe_value}"
```

#### The Agent Must Never Interpolate Raw User Input

The system prompt must reinforce this:

```markdown
## SYSTEM PROMPT — SOQL INJECTION PREVENTION

When constructing a WHERE clause that includes a value the user provided:

CORRECT:
  WHERE Name = '{escaped_user_value}'
  (Pass the user's value through the escaping function, then interpolate)

INCORRECT:
  WHERE Name = '{raw_user_input}'
  (Never interpolate raw user input directly)

For ID values: always validate that the value matches Salesforce ID format 
(15 or 18 alphanumeric characters) before using in a WHERE clause.
```

```python
import re

def validate_salesforce_id(value: str) -> bool:
    """Validate Salesforce record ID format (15 or 18 char alphanumeric)."""
    return bool(re.match(r'^[a-zA-Z0-9]{15}$|^[a-zA-Z0-9]{18}$', value))
```

---

### 7.3 Scoping the Agent's Salesforce Access

#### Dedicated Integration User

Create a dedicated Salesforce user for the Power Platform integration:

```text
User Type: Integration User (not a named user licence — use an Integration licence 
           which does not consume a named user seat in Salesforce)
Profile: Minimum profile (e.g. "Salesforce API Only User" — no UI access)
Permission Set: Create a custom permission set with ONLY:
  - Object permissions: Read on the specific objects needed (Account, Opportunity, 
    Contact, Case, etc.) — no Create, Edit, Delete
  - Field permissions: Read on only the fields the agent needs — explicitly remove 
    read from PII fields
  - No System Permissions (no "Modify All Data", no "View All Data" — use 
    sharing rules instead)
```

#### IP Allowlisting on the Connected App

To restrict the integration user to calls originating from Power Platform only:

1. In Salesforce, navigate to **Setup → Network Access**
2. Find Power Platform's outbound IP ranges:
   - Microsoft publishes these at: <https://www.microsoft.com/en-us/download/details.aspx?id=56519>
   - Filter for `PowerPlatform` service tag in your region
3. Add each IP range to Salesforce's **Network Access** trusted IP list
4. On the Connected App, set **IP Relaxation** to `Enforce IP restrictions`

> ⚠️ **Research gap**: Power Platform's outbound IP ranges are not static — they change when Microsoft scales infrastructure. Subscribe to the Microsoft IP range change notifications (available via the download page above) and update the Salesforce Network Access list accordingly. Consider automating this via a scheduled Power Automate flow that compares current Power Platform IP ranges against the Salesforce Network Access list and alerts on discrepancies.

#### Power Platform DLP Policy for the Salesforce Agent Environment

```text
Business data group (allowed):
  - Salesforce (native connector)
  - SharePoint (knowledge source)
  - Microsoft Dataverse (audit logging)

Blocked:
  - HTTP with custom URL   ← must be blocked
  - All other connectors
```

---

## Section 8 — End-to-End Implementation Checklist

### Phase 1: Salesforce Setup

1. **[ ] Create Connected App** in Salesforce Setup → App Manager
   - OAuth scopes: `api refresh_token offline_access`
   - Callback URLs: Power Platform OAuth redirect URLs
   - IP Relaxation: Set to "Enforce IP restrictions" (configure trusted IPs in Phase 4)
   - Note the Consumer Key and Consumer Secret

2. **[ ] Create Integration User**
   - User type: Integration User licence
   - Profile: Minimum-access profile (no UI access)
   - Assign to a new custom Permission Set (created in next step)

3. **[ ] Create and configure Permission Set**
   - Object permissions: Read-only on target objects (Account, Opportunity, Contact, Case, etc.)
   - Field permissions: Read on required fields only; explicitly deny PII fields
   - Assign permission set to the integration user

4. **[ ] Pre-authorise Connected App for integration user**
   - In Setup → Connected Apps → Manage → Edit Policies
   - Set "Permitted Users" to "Admin approved users are pre-authorized"
   - Pre-authorize the integration user's profile/permission set

5. **[ ] Store client secret in Azure Key Vault**
   - Create Key Vault in Azure
   - Add Connected App Consumer Secret as a Key Vault Secret
   - Note the Key Vault URL and Secret name
   - Set 90-day expiry with alert

### Phase 2: Power Platform Connection Setup

6. **[ ] Create Power Platform environment** (dedicated, isolated from production)
   - Name: e.g. "SalesforceAgentEnv"
   - Type: Production (not sandbox — sandboxes have connectivity restrictions)
   - Region: Match your data residency requirements

7. **[ ] Create Salesforce Connection Reference**
   - In Power Apps → Data → Connections → New connection → Salesforce
   - Authenticate as the integration user using the Connected App
   - Save as a Connection Reference (not a personal connection)
   - Name: e.g. "Salesforce-CRM-ReadOnly"

8. **[ ] Configure DLP policy** for the dedicated environment
   - Business group: Salesforce, SharePoint, Dataverse
   - Blocked: HTTP with custom URL, all others

### Phase 3: Schema Export and Knowledge Source

9. **[ ] Export Salesforce schema** using Salesforce CLI or REST API script
   - Export all objects the agent will query
   - Format as Markdown (using the template in Section 6.1)
   - Create a "Schema Index" document (Section 6.2)

10. **[ ] Upload schema documents to SharePoint**
    - Create a dedicated SharePoint document library: "CopilotStudio-Knowledge"
    - Upload the Schema Index document and all object-specific schema documents

11. **[ ] Set up schema refresh automation**
    - Create Power Automate flow: Recurrence (weekly) → Salesforce API → update SharePoint documents
    - Test the flow end-to-end
    - Configure alerting for flow failures

### Phase 4: Copilot Studio Agent Configuration

12. **[ ] Create Copilot Studio agent**
    - Navigate to Copilot Studio → Create → New agent
    - Name: e.g. "Salesforce Data Assistant"
    - Select the dedicated Power Platform environment

13. **[ ] Configure system prompt (Instructions)**
    - Add the mandatory process instructions from Section 6.2
    - Add object alias disambiguation table from Section 6.4
    - Add PII field restrictions from Section 7.1
    - Add SOQL validation rules from Section 7.2

14. **[ ] Add knowledge source**
    - Add the SharePoint document library as a knowledge source
    - Name it "Salesforce Schema Reference"
    - Write the description that tells the agent when to consult it
    - Enable Generative Answers grounding

15. **[ ] Create the "Run Salesforce Query" action**
    - New action in Copilot Studio → Power Automate flow
    - Input: soql_query (string)
    - Flow steps: Validate SOQL → Execute Salesforce connector → Log to Dataverse → Return results
    - Test with a known-good SOQL query

16. **[ ] Configure Classic Actions** for sensitive operations
    - Salesforce data retrieval should use Classic Actions (deterministic trigger)
    - Avoid Generative Actions for connector invocations

### Phase 5: SOQL Validation Action

17. **[ ] Deploy SOQL validation function**
    - Options: Azure Function, Power Automate Compose with expression, or inline in the flow
    - Implement the validation rules from Section 6.3 Step 4
    - Test with injection payloads:
      ```
      Test: "SELECT Id, Name FROM Account WHERE Name = 'x' OR Id != '"
      Expected: Blocked (injection detected via escaping)
      
      Test: "DELETE FROM Account"  
      Expected: Blocked (DELETE pattern)
      
      Test: "SELECT Id, Name FROM Account"
      Expected: Blocked (no LIMIT clause)
      
      Test: "SELECT Id, Name FROM Account LIMIT 200"
      Expected: Pass
      ```

18. **[ ] Create Dataverse audit table** `agent_soql_audit_log`
    - Columns as specified in Section 7.1
    - Configure table permissions: agent's Managed Identity can create rows, no user read access except admins

### Phase 6: DLP and Monitoring

19. **[ ] Verify DLP policy is active**
    - In Power Platform admin center, confirm the DLP policy shows as Active for the agent environment
    - Run a test that attempts to use the HTTP connector — confirm it is blocked

20. **[ ] Configure Power Platform admin analytics alerts**
    - DLP violation alerts → immediate notification
    - Flow failure alerts for the Salesforce query flow
    - Token consumption anomaly alerts

21. **[ ] Deploy Microsoft Sentinel detection rules** (if Sentinel is in scope)
    - DLP violation detection rule
    - Unusual session length rule
    - High-volume query detection rule (e.g. >100 Salesforce API calls from agent in 1 hour)

22. **[ ] Update Salesforce Network Access** with Power Platform IP ranges
    - Download current ranges from Microsoft IP list
    - Add to Salesforce Setup → Network Access
    - Enable IP enforcement on Connected App

### Phase 7: User Acceptance Testing

23. **[ ] Test natural-language query coverage**
    - Standard objects: Account, Opportunity, Contact, Case
    - Alias terms: "deals", "companies", "tickets", "prospects"
    - Aggregate queries: counts, sums, group by
    - Ambiguous queries: confirm agent asks clarifying questions rather than guessing

24. **[ ] Test schema grounding**
    - Query a custom field: confirm agent finds the correct API name from schema
    - Query a non-existent field: confirm agent declines and asks for clarification
    - Query with picklist filter: confirm agent uses valid picklist values from schema

25. **[ ] Test security controls**
    - Attempt SOQL injection via query: confirm blocked and logged
    - Attempt to request PII fields: confirm agent declines
    - Attempt query with no LIMIT: confirm validation blocks it
    - Confirm all executed queries appear in Dataverse audit log

26. **[ ] Test with non-technical users**
    - Recruit 3–5 representative end users (sales reps, managers)
    - Give them realistic scenarios with no technical guidance
    - Document: (a) queries that succeed on first try, (b) queries requiring clarification, (c) failures
    - Iterate on alias disambiguation table and system prompt based on results

27. **[ ] Document known limitations and communicate to users**
    - Maximum record limit per query
    - Objects and fields that are not in scope
    - How to request additions to the schema
    - How to report incorrect or unexpected results

---

## Sources & Further Reading

### Salesforce Developer Documentation

- Salesforce. "SOQL and SOSL Reference." <https://developer.salesforce.com/docs/atlas.en-us.soql_sosl.meta/soql_sosl/>
- Salesforce. "Salesforce APIs." <https://developer.salesforce.com/docs/apis>
- Salesforce. "Metadata API Developer Guide." <https://developer.salesforce.com/docs/atlas.en-us.api_meta.meta/api_meta/>
- Salesforce. "Connected Apps." <https://help.salesforce.com/s/articleView?id=sf.connected_app_overview.htm>
- Salesforce. "Field-Level Security Overview." <https://help.salesforce.com/s/articleView?id=sf.field_permissions.htm>
- Salesforce. "API Call Limits." <https://developer.salesforce.com/docs/atlas.en-us.salesforce_app_limits_cheatsheet.meta/salesforce_app_limits_cheatsheet/salesforce_app_limits_platform_api.htm>
- Salesforce CLI. <https://developer.salesforce.com/docs/atlas.en-us.sfdx_cli_reference.meta/sfdx_cli_reference/>

### Microsoft Copilot Studio and Power Platform

- Microsoft Learn. "Salesforce connector for Power Platform." <https://learn.microsoft.com/en-us/connectors/salesforce/>
- Microsoft Learn. "Add knowledge to your Copilot Studio agent." <https://learn.microsoft.com/en-us/microsoft-copilot-studio/knowledge-add>
- Microsoft Learn. "Configure generative answers." <https://learn.microsoft.com/en-us/microsoft-copilot-studio/nlu-generative-answers-azure-openai>
- Microsoft Learn. "Create and use connection references." <https://learn.microsoft.com/en-us/power-apps/maker/data-platform/create-connection-reference>
- Microsoft Learn. "Data loss prevention policies." <https://learn.microsoft.com/en-us/power-platform/admin/wp-data-loss-prevention>
- Microsoft Learn. "Use Azure Key Vault secrets in Power Automate." <https://learn.microsoft.com/en-us/power-automate/desktop-flows/use-sensitive-information-ui-flows>
- Microsoft. "Power Platform IP address ranges." <https://www.microsoft.com/en-us/download/details.aspx?id=56519>

### Security and Prompt Injection

- OWASP. "OWASP Top 10 for LLM Applications — LLM01: Prompt Injection." <https://genai.owasp.org/llmrisk/llm01-prompt-injection/>
- Greshake, K. et al. (2023). "Not What You've Signed Up For: Compromising Real-World LLM-Integrated Applications with Indirect Prompt Injection." arXiv:2302.12173. <https://arxiv.org/abs/2302.12173>
- Salesforce. "SOQL Injection." <https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/pages_security_tips_soql_injection.htm>
- Microsoft Learn. "Responsible AI FAQ for Copilot Studio." <https://learn.microsoft.com/en-us/microsoft-copilot-studio/responsible-ai-overview>

### Schema and Prompt Engineering

- Salesforce. "sObject Describe." <https://developer.salesforce.com/docs/atlas.en-us.api_rest.meta/api_rest/resources_sobject_describe.htm>
- Microsoft Learn. "Prompt engineering best practices for Copilot Studio." <https://learn.microsoft.com/en-us/microsoft-copilot-studio/nlu-prompt-node>
- Anthropic. "Prompt engineering overview." <https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/overview>
