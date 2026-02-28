# Autonomous Research Task — Secure AI Document Workflow & Copilot Studio Integration

You are an expert technical researcher. Your task is to produce two comprehensive, 
well-structured research documents in Markdown. Work autonomously. Do not ask for 
clarification — make reasonable assumptions and document them. Save outputs to the 
paths specified at the end of this prompt.

---

## RESEARCH PIECE 1: Secure AI-Powered Office Document Workflows

### Background & Problem Statement

The target workflow is:
  1. A user pastes one or more office documents (DOCX, XLSX, PPTX, PDF) plus a 
     natural-language prompt into an AI agent.
  2. The agent analyses the documents and generates a new office document as output 
     (e.g. a report, a summary deck, a populated spreadsheet, a contract draft).
  3. The agent may invoke code — Python, JavaScript, shell — to parse documents, 
     call APIs, or format outputs.

The candidate platforms to evaluate are:
  - Microsoft Copilot Studio (with Power Platform connectors)
  - Anthropic Claude (claude.ai Projects / Claude for Work / API with tool use)
  - Perplexity Computer (Perplexity's agentic/computer-use surface)
  - Self-hosted OpenClaw (or equivalent open-source agent framework such as 
    OpenHands / SWE-agent / AutoGen) running on Cloudflare Workers AI or a 
    Cloudflare Workers + Durable Objects architecture
  - Microsoft Copilot Tasks (the emerging scheduled/autonomous Copilot surface — 
    research current public roadmap as of early 2026)

---

### Section 1 — Threat Model

Research and document a complete threat model for this class of workflow. Cover:

1.1  PROMPT INJECTION via document content
     - Direct injection (attacker controls the document body — e.g. white-on-white 
       text, hidden XML nodes, metadata fields, footer text instructing the agent 
       to ignore its system prompt)
     - Indirect injection (document references an external URL; agent fetches it 
       and the fetched content contains injections)
     - Multi-document injection (malicious instructions span multiple documents so 
       no single document triggers filters)
     - Exfiltration via injection (injected instruction tells agent to summarise 
       all documents and POST the summary to an attacker-controlled endpoint)

1.2  Malicious code execution surface
     - npm install / pip install of attacker-named packages (typosquatting: 
       e.g. "docx-parser-pro", "xlsx-ai", "pdf2text-fast")
     - git clone / git pull of a repo whose URL was injected into the context
     - eval() / exec() of LLM-generated code that was itself influenced by 
       document content
     - Curl/wget/fetch to exfiltrate data or download second-stage payloads
     - Environment variable leakage through LLM-generated print/console.log/logging

1.3  Supply chain and dependency risks
     - LLM-suggested libraries that are abandoned, hijacked, or contain 
       known CVEs (research methodology: how to audit LLM-recommended packages)
     - Lockfile poisoning if the agent commits back to a repository

1.4  Data residency and confidentiality risks
     - Sensitive document content sent to third-party LLM APIs (vendor data 
       processing agreements, zero-retention policies, training opt-outs)
     - Intermediate artefacts stored in the vendor's infrastructure 
       (chat history, file uploads, tool call logs)
     - Differences between Copilot Studio (MSFT data boundary), Claude API 
       (Anthropic BAA for enterprise), Perplexity (privacy policy gaps), 
       self-hosted (full control but operational burden)

1.5  Identity and privilege escalation
     - Agent acting with the authenticated user's delegated token — what can it 
       access beyond the intended scope?
     - OAuth consent phishing through injected "please authorise this connection" 
       instructions
     - Connector credential exposure in Copilot Studio flows

---

### Section 2 — Platform Evaluation Matrix

Evaluate each of the five platforms against the following criteria. Use a scoring 
rubric (1–5) and provide written justification for each score.

Criteria:
  a. Sandboxing of code execution (is generated code run in an isolated environment 
     with no network egress, no filesystem access beyond a temp dir, no access to 
     secrets/env vars?)
  b. Prompt injection mitigations built into the platform (input sanitisation, 
     system prompt hardening, document parsing that strips executable content)
  c. Outbound network controls (can the agent make arbitrary HTTP calls? Can 
     egress be whitelisted/blocked?)
  d. Audit logging and observability (can you replay exactly what the agent did, 
     what code it ran, what it sent where?)
  e. Data residency and compliance posture (GDPR, SOC 2, ISO 27001, HIPAA BAA 
     availability, data processing location)
  f. Document format support (native DOCX/XLSX/PPTX parsing without shelling out 
     to arbitrary libraries)
  g. Operator control surface (can you restrict which tools/connectors/capabilities 
     are available to the agent? Can you enforce an allowlist of APIs it may call?)
  h. Cost and operational complexity
  i. Time-to-production for this specific workflow

For the self-hosted option, also cover:
  - Recommended Cloudflare architecture (Workers AI gateway, Durable Objects for 
    session state, R2 for document storage, wrangler secrets for credentials)
  - How to implement egress filtering within the Cloudflare network (Cloudflare 
    Gateway / Zero Trust policies to block unexpected outbound calls)
  - Container isolation options (Cloudflare Workers sandbox vs Docker + gVisor vs 
    Firecracker microVMs)

For Copilot Tasks, document:
  - Current public availability status and roadmap as of early 2026
  - Whether it supports file/document inputs
  - Security model compared to Copilot Studio

---

### Section 3 — Recommended Secure Architecture for Copilot Studio

Produce a detailed implementation guide for building this workflow in Copilot Studio 
that maximises security. Cover:

3.1  Copilot Studio agent design
     - System prompt hardening: how to write a meta-prompt that resists injection 
       (instruction hierarchy, refusal rules, scope boundaries)
     - How to use "Generative Actions" vs "Classic Actions" and the security 
       implications of each
     - How to restrict which connectors the agent can invoke (connector allowlisting 
       in the Power Platform environment admin settings)

3.2  Document ingestion pipeline
     - Recommended approach for receiving office documents (SharePoint upload trigger 
       → Power Automate flow → extract text via Graph API / Azure Document Intelligence 
       → pass clean text to agent — NOT raw binary to the LLM)
     - Why you should never pass raw DOCX/XLSX binary to the LLM as a file upload 
       without first parsing it in a controlled pipeline (hidden XML, embedded macros, 
       OLE objects)
     - How to sanitise extracted text before injecting it into the agent context 
       (strip comment nodes, hidden rows/slides, metadata author fields, revision 
       history that could carry injected instructions)

3.3  Output document generation
     - Recommended pattern: LLM generates structured JSON → a deterministic Power 
       Automate action renders the JSON into a DOCX/XLSX using a template (not 
       LLM-generated code) → no code execution in the hot path
     - Template options: Word Online (Business) connector, Excel Online (Business) 
       connector, or Azure Functions with a pinned version of python-docx/openpyxl 
       with a locked requirements.txt
     - Why deterministic rendering is safer than asking the LLM to write and run 
       Python to build the document

3.4  Data loss prevention
     - Microsoft Purview DLP policies for Power Platform: how to configure them to 
       block connectors that can exfiltrate data (HTTP with custom URL, SFTP, etc.)
     - How to scope the agent to only use approved connectors (SharePoint, Word 
       Online, Outlook — nothing else)
     - Tenant isolation: separate Power Platform environment for this agent with 
       its own DLP policies rather than embedding in the production environment

3.5  Monitoring and incident response
     - Power Platform admin analytics: what to log and alert on
     - How to use Microsoft Sentinel to ingest Copilot Studio / Power Platform 
       audit logs and write detection rules for anomalous agent behaviour
     - How to use Copilot Studio's "conversation transcripts" table in Dataverse 
       for forensic replay

3.6  Security checklist (produce a numbered checklist that can be used as a 
     pre-deployment sign-off)

---

### Section 4 — Comparative Recommendations

Synthesise the research into a recommendation section:

  4.1  "If you need enterprise compliance and are already in the Microsoft 365 
       ecosystem" — Copilot Studio recommendation with specific caveats
  4.2  "If you need maximum control and are comfortable with infrastructure" — 
       self-hosted recommendation
  4.3  "If you need fastest time-to-value for a non-regulated use case" — 
       Claude API recommendation
  4.4  A decision tree diagram (Mermaid syntax) to help readers choose a platform
  4.5  Red lines: things you should NEVER do in any platform (e.g. let the agent 
       install arbitrary npm/pip packages, pass raw document binaries to the LLM, 
       give the agent a service account with write access to production systems)

---

## RESEARCH PIECE 2: Copilot Studio Agent with Salesforce Integration

### Background

The goal is a Copilot Studio agent that:
  - Connects to Salesforce CRM data via the native Salesforce connector in 
    Power Platform (not a custom HTTP connector)
  - Has the Salesforce schema (objects, fields, relationships) embedded as 
    grounded knowledge so it can dynamically craft SOQL queries based on a 
    user's natural-language intent
  - Returns accurate, scoped results without hallucinating field names or 
    object relationships
  - Handles auth securely (no hardcoded credentials, proper OAuth 2.0 
    connected app pattern)

---

### Section 5 — Salesforce Connector Architecture in Power Platform

5.1  Authentication setup
     - Salesforce Connected App: required OAuth scopes (api, refresh_token, 
       offline_access — explain why each is needed and which to avoid)
     - Difference between "connection reference" (shareable, managed) and 
       per-user connections in Power Platform — recommend connection references 
       for agents
     - How to store and rotate the Connected App client secret in Azure Key Vault 
       and reference it from Power Platform without embedding it in the flow

5.2  Connector capabilities and limits
     - What the native Power Platform Salesforce connector supports: query, 
       get record, create/update/delete, get object metadata
     - SOQL character limits and how they interact with LLM-generated queries
     - Rate limiting: Salesforce API call limits (daily, per-user) and how to 
       handle 429s in Power Automate
     - Which Salesforce API version the connector uses and how to check for 
       deprecation risk

5.3  Dynamic SOQL generation risks
     - SOQL injection: can a user manipulate the agent into leaking records 
       they should not see?
     - How to implement field-level and record-level security enforcement 
       (rely on Salesforce FLS/sharing rules as the authoritative guard — NOT 
       the agent's judgement)
     - How to scope the Connected App's profile/permission set to the minimum 
       required objects (principle of least privilege)

---

### Section 6 — Schema as Grounded Knowledge

6.1  Schema extraction
     - How to use the Salesforce Metadata API / Tooling API to export a 
       machine-readable schema (object definitions, field names, data types, 
       picklist values, lookup relationships, required fields, formula fields)
     - Recommended format for the exported schema document (JSON vs Markdown 
       table vs CSV — weigh LLM token consumption against readability)
     - How frequently to refresh the schema export and how to automate that 
       refresh (scheduled Power Automate flow → Salesforce Metadata API → 
       update the knowledge source in Copilot Studio)

6.2  Knowledge source configuration in Copilot Studio
     - How to add a SharePoint document (the schema export) as a knowledge 
       source in Copilot Studio
     - How to write a system prompt instruction that tells the agent: 
       "Before crafting any SOQL query, consult the schema knowledge source 
       to verify the exact API name of the object and field you intend to query. 
       Never guess field names."
     - Chunking strategy: Salesforce schemas for large orgs can be very large — 
       how to chunk the schema document so relevant object definitions are 
       retrieved without overflowing the context window
     - How to use Copilot Studio's "Generative Answers" grounding to force 
       citations back to the schema document (so hallucinated field names are 
       caught)

6.3  SOQL query construction pattern
     - Recommended prompt engineering pattern for SOQL generation:
         Step 1: Intent classification (what object, what filter, what output)
         Step 2: Schema lookup (retrieve the relevant object definition from 
                 knowledge)
         Step 3: Draft SOQL (LLM drafts query using verified field API names)
         Step 4: Validation step (a deterministic regex/parser checks the SOQL 
                 for obviously dangerous patterns before execution — e.g. no 
                 SELECT * equivalent, no more than N records, no DELETE/UPDATE)
         Step 5: Execute via connector
         Step 6: Format results for user
     - Example system prompt snippets for each step
     - How to handle aggregate queries (COUNT, SUM, GROUP BY) vs record-fetch 
       queries differently

6.4  Handling missing or ambiguous schema references
     - What the agent should do when the user references an object or field 
       that does not exist in the schema (ask for clarification, suggest closest 
       match — do NOT hallucinate)
     - How to build a disambiguation flow for common Salesforce object aliases 
       (e.g. user says "deals" → map to Opportunity, "companies" → Account)

---

### Section 7 — Security for the Salesforce Agent

7.1  Preventing data over-exposure
     - The agent should always apply a LIMIT clause (recommend 200 as a default 
       ceiling, configurable)
     - Never expose Personally Identifiable Information fields unless the user's 
       role explicitly requires it — how to implement a PII field blocklist in 
       the system prompt and enforce it via Salesforce FLS
     - Logging every SOQL query the agent executes to Dataverse for audit purposes

7.2  Preventing SOQL injection
     - All user-supplied filter values must be passed as bind variables where 
       the connector supports it, or escaped — document the escaping rules for 
       SOQL string literals
     - The agent must never interpolate raw user input directly into a SOQL 
       WHERE clause without sanitisation

7.3  Scoping the agent's Salesforce access
     - Dedicated Salesforce integration user (not a named user licence) with 
       a permission set granting read-only access to the specific objects the 
       agent needs
     - IP allowlisting on the Connected App to only permit calls originating 
       from Power Platform's IP ranges (document how to find those ranges)

---

### Section 8 — End-to-End Implementation Checklist for the Salesforce Agent

Produce a numbered, step-by-step checklist covering:
  - Salesforce Connected App setup
  - Power Platform connection reference creation
  - Schema export and upload to SharePoint
  - Copilot Studio agent creation: knowledge source, system prompt, topics/actions
  - SOQL validation action setup
  - DLP policy configuration
  - Monitoring and alerting setup
  - User acceptance testing protocol

---

## Output Instructions

Save the two research documents to:
  - ./research/secure-ai-document-workflow.md   (Research Piece 1)
  - ./research/copilot-studio-salesforce.md     (Research Piece 2)

Each document must:
  - Use H1/H2/H3 heading hierarchy matching the sections above
  - Include a Table of Contents with anchor links at the top
  - Include a "Sources & Further Reading" section at the bottom with real, 
    verifiable URLs (Microsoft Learn, Anthropic docs, Salesforce developer docs, 
    academic papers on prompt injection)
  - Where you recommend a specific configuration value, code snippet, or 
    architectural pattern, present it in a fenced code block with the 
    appropriate language tag
  - Where a comparison is made, use a Markdown table
  - Where a process is described, use a numbered list
  - Flag any area where the research is incomplete with a 
    > ⚠️ **Research gap**: [description] 
    callout so the human reviewer knows what to validate manually
  - Target length: Research Piece 1 ≥ 4,000 words, Research Piece 2 ≥ 3,000 words

Do not ask for confirmation. Begin research immediately and write both documents 
in full before stopping.
