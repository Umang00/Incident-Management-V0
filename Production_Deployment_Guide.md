# 🏭 Production Deployment Guide

> **Audience:** Engineers who have completed the [Quickstart](../Incident%20Management%20V0/README.md) and want to take this system beyond the sample data into a real production environment.

This guide covers what changes to make, what to swap, and what to add when deploying this workflow with your organisation's actual incidents, systems, and threat intel sources.

---

## 📋 Table of Contents

- [Customising the Data Schema](#1-customising-the-data-schema)
- [Changing the Embedding Model](#2-changing-the-embedding-model)
- [Moving to Hybrid Search](#3-moving-to-hybrid-search)
- [Adding Targeted Threat Intel APIs](#4-adding-targeted-threat-intel-apis)
- [Integrating Your Ticketing System](#5-integrating-your-ticketing-system)
- [MITRE ATT&CK Ticket Format Requirements](#6-mitre-attck-ticket-format-requirements)
- [Customising with AI Assistance](#7-customising-any-part-with-ai-assistance)

---

## 1. Customising the Data Schema

### What lives where

The v0 schema uses three tables in Supabase:

| Table                    | Purpose                             | Contains AI Embeddings?                   |
| ------------------------ | ----------------------------------- | ----------------------------------------- |
| `resolved_incidents_v0`  | Past incident vector store          | ✅ Yes — full markdown content + metadata |
| `reference_playbooks_v0` | Playbook vector store               | ✅ Yes — LLM-summarised trigger text      |
| `test_incidents_v0`      | Incoming alerts + generated reports | ❌ No — relational storage only           |

### The incident JSON schema

All resolved incidents must match this JSON structure (see `Resolved Incidents/` for examples):

```json
{
  "@timestamp": "2025-10-04T00:00:00Z",
  "incident_id": "INC-2025-1004-014",
  "severity": "medium",
  "status": "resolved",
  "category": "infrastructure",
  "type": "configuration_drift",
  "title": "Production server configuration drift detected",
  "description": "Production server configuration drift detected - incident detected and resolved in Q4 2025",
  "affected_systems": ["web-servers-prod", "app-servers"],
  "assigned_to": "infrastructure-team",
  "detection_method": "Automated monitoring",
  "tags": ["configuration-drift", "infrastructure", "medium"],
  "timeline": [
    {
      "timestamp": "2025-10-04T00:00:00Z",
      "event": "Incident detected by monitoring system",
      "actor": "monitoring-system"
    },
    {
      "timestamp": "2025-10-04T00:02:00Z",
      "event": "Alert acknowledged by on-call engineer",
      "actor": "oncall@company.com"
    },
    {
      "timestamp": "2025-10-04T00:15:00Z",
      "event": "Root cause identified",
      "actor": "engineering-team"
    },
    {
      "timestamp": "2025-10-04T01:38:00Z",
      "event": "Fix applied and testing",
      "actor": "engineering-team"
    },
    {
      "timestamp": "2025-10-04T02:21:00Z",
      "event": "Incident resolved and services restored",
      "actor": "engineering-team"
    }
  ],
  "resolved_timestamp": "2025-10-04T02:21:00Z",
  "resolved_by": "engineering@company.com",
  "resolution_time_minutes": 141,
  "resolution_summary": "Production server configuration drift detected resolved successfully. Systems restored to normal operation",
  "root_cause_analysis": "Root cause analysis completed for configuration_drift incident",
  "mitre": {
    "tactic": ["Defense Evasion"],
    "technique": ["T1562 - Impair Defenses"],
    "prevention": [
      "M1047 - Audit",
      "M1022 - Restrict File and Directory Permissions"
    ]
  },
  "mitre_tactic_ids": ["TA0005"],
  "mitre_technique_ids": ["T1562"],
  "mitre_prevention_ids": ["M1047", "M1022"],
  "affected_users_count": 350,
  "business_impact": "Infrastructure component degradation, backup systems engaged, limited impact",
  "estimated_cost": 11750,
  "sla_met": true,
  "mttr": 141,
  "mtta": 2,
  "remediation_actions": [
    "Isolated and secured affected systems: web-servers-prod, app-servers",
    "Applied immediate patches and configuration updates to affected components",
    "Verified system integrity and restored normal operations",
    "Conducted security validation and compliance checks"
  ],
  "systems_remediated": ["web-servers-prod", "app-servers"],
  "preventive_measures": [
    "Enhanced monitoring and alerting for similar attack patterns and anomalies",
    "Updated security policies and access control mechanisms",
    "Implemented additional validation and testing procedures",
    "Scheduled regular security audits and compliance reviews"
  ],
  "security_controls_updated": [],
  "documentation_updated": [
    "Incident response runbook updated with lessons learned",
    "Security procedures documentation enhanced",
    "System architecture diagrams updated",
    "Recovery and rollback procedures documented"
  ],
  "lessons_learned": "Root cause analysis completed for configuration_drift incident",
  "follow_up_tasks": [
    "Conduct post-incident review with all stakeholders",
    "Update disaster recovery and business continuity plans",
    "Schedule training sessions for affected teams",
    "Implement long-term preventive measures by Q1 2026"
  ],
  "resolution": "Production server configuration drift detected resolved successfully. Systems restored to normal operation",
  "action_taken": "Resolved by infrastructure-team within 141 minutes of detection"
}
```

### Adding or removing fields

The metadata fields extracted at ingestion time are defined in the **Code Node** called `Prepare Markdown Document & Metadata` in your n8n workflow.

**Current metadata fields extracted:**

- `@timestamp`
- `incident_id`
- `severity`
- `category`
- `type`
- `affected_systems`
- `mitre_tactic_ids`
- `mitre_technique_ids`
- `mitre_prevention_ids`

To **add a new field** (e.g. `environment: "production"` or `team: "platform"`), you need to:

1. Add the field to your incident JSON files
2. Modify the Code Node to extract it
3. Re-run the ingestion pipeline

> **Tip: Use AI to make these changes easily.** See [Section 7](#7-customising-any-part-with-ai-assistance) for how to prompt an AI model with the Code Node content to make precise modifications without hassle.

### Changing the database provider

The vector database is accessed via Supabase's `pgvector` extension. If you want to use a different vector database (e.g. Pinecone, Qdrant, Weaviate), the n8n node type changes but the architecture stays the same:

| Component                           | v0 (Supabase)                                   | Alternative                    |
| ----------------------------------- | ----------------------------------------------- | ------------------------------ |
| Vector insert at ingestion          | `Supabase Vector Store (Insert)` node           | Pinecone/Qdrant insert nodes   |
| Vector retrieval at query time      | `Supabase Vector Store (Retrieve-as-Tool)` node | Pinecone/Qdrant retrieve nodes |
| Relational storage (test incidents) | Supabase `test_incidents_v0` table              | Any DB (Postgres, MySQL)       |

The `supabase_schema_v0.sql` file contains the DDL for the Supabase-specific `pgvector` setup. If migrating, replicate the column structure in your target DB.

---

## 2. Changing the Embedding Model

### Current model: `gemini-embedding-001`

The workflow uses Google's `gemini-embedding-001` which produces **3072-dimensional** vectors — one of the highest dimensions available for a hosted model. This is configured in three Embeddings nodes in the workflow.

### When to consider a change

| Use Case                                         | Recommendation                                                                                              |
| ------------------------------------------------ | ----------------------------------------------------------------------------------------------------------- |
| Stay on Google ecosystem, simplest path          | Keep `gemini-embedding-001` — it actually outperforms `text-embedding-004` on retrieval-specific benchmarks |
| Best retrieval accuracy for security/log content | Switch to **Voyage AI `voyage-code-2`** — trained on code/technical data, best for log payloads and JSON    |
| Absolute maximum retrieval performance           | **Octen-Embedding-8B** via Fireworks AI or DeepInfra — currently #1 on RTEB leaderboard                     |
| Self-hosted / no external API calls              | **BGE-M3 (BAAI)** or **E5-Mistral** — requires GPU                                                          |

### How to change

1. Update all three `Embeddings Google Gemini` nodes in the workflow to use your preferred embedding provider node
2. Update the `vector(3072)` column dimension in `supabase_schema_v0.sql` to match your new model's output dimension (e.g. Voyage AI uses 1536)
3. **Critically:** Drop and recreate the vector tables and re-run all ingestion pipelines — you cannot mix embedding spaces

### Embedding model comparison

| Model                            | Dimensions    | RTEB Score | Best For                      | Cost                 |
| -------------------------------- | ------------- | ---------- | ----------------------------- | -------------------- |
| `gemini-embedding-001` (current) | 3072          | High       | General retrieval             | Free (rate limited)  |
| `text-embedding-004`             | 768 (elastic) | Medium     | Clustering + classification   | Free                 |
| `voyage-code-2`                  | 1536          | Very High  | Technical logs, code payloads | ~$0.10/1M tokens     |
| `Octen-Embedding-8B`             | 4096          | #1         | Max accuracy                  | Fireworks AI pricing |

---

## 3. Moving to Hybrid Search

### Why hybrid search matters

The v0 system uses **pure semantic search** (vector similarity only). This works well for conceptual matching ("this looks like a brute force attack") but can miss exact keyword matches for:

- Specific error codes: `RDS-EVENT-0056`, `OOM killer`, `SIGKILL`
- Exact CVE IDs: `CVE-2025-24813`
- Short technical strings: `pg_replication_slot`, `fs.closeSync`

**Hybrid search** combines vector similarity (70%) with keyword relevance via PostgreSQL Full Text Search (30%) to capture both.

### Architecture

The native n8n `Supabase Vector Store` node **cannot** do hybrid search — it only sends the embedding, not the raw query text. To enable hybrid search:

**Step 1: Add the RPC function to Supabase**

Run the following SQL in your Supabase SQL editor (Dashboard → SQL Editor → New Query):

```sql
CREATE OR REPLACE FUNCTION match_documents_hybrid(
  query_embedding vector(3072),
  query_text      text,
  match_count     int  DEFAULT 10,
  filter          jsonb DEFAULT '{}'
)
RETURNS TABLE (
  id         bigint,
  content    text,
  metadata   jsonb,
  similarity float
)
LANGUAGE plpgsql
AS $$
DECLARE
  vector_weight float := 0.7;
  keyword_weight float := 0.3;
BEGIN
  RETURN QUERY
  WITH vector_scores AS (
    SELECT
      d.id,
      d.content,
      d.metadata,
      1 - (d.embedding <=> query_embedding) AS score
    FROM documents d
    WHERE
      (filter = '{}' OR d.metadata @> filter)
  ),
  keyword_scores AS (
    SELECT
      d.id,
      ts_rank_cd(
        to_tsvector('english', d.content),
        plainto_tsquery('english', query_text)
      ) AS score
    FROM documents d
    WHERE
      (filter = '{}' OR d.metadata @> filter)
  )
  SELECT
    v.id,
    v.content,
    v.metadata,
    (vector_weight * v.score + keyword_weight * COALESCE(k.score, 0))::float AS similarity
  FROM vector_scores v
  LEFT JOIN keyword_scores k ON v.id = k.id
  ORDER BY similarity DESC
  LIMIT match_count;
END;
$$;
```

> **Note on dimensions:** If you use an embedding model other than `gemini-embedding-001`, update `vector(3072)` to match your model's output dimension (e.g. `vector(1536)` for Voyage AI). See [Section 2](#2-changing-the-embedding-model) for the full embedding model comparison.

> **Note on table name:** The function above references a `documents` table — this is the default table name created by the n8n Supabase Vector Store node. If your table name differs, update `FROM documents` accordingly. You can verify the exact table name in your Supabase Dashboard → Table Editor.

**Step 2: Replace the vector store nodes with HTTP Request Tool nodes**

Call the new RPC directly from n8n using an HTTP Request node:

```json
POST https://[your-project].supabase.co/rest/v1/rpc/match_documents_hybrid
Authorization: Bearer [your-service-role-key]
Content-Type: application/json

{
  "query_embedding": [...],
  "query_text": "the raw incident description text",
  "match_count": 20,
  "filter": {}
}
```

**Step 3:** Wire this HTTP Request as a tool to the AI Agent nodes that previously used the native Supabase Vector Store nodes.

### Hybrid search maturity ladder

| Stage              | What You Have                   | When to Upgrade                                                     |
| ------------------ | ------------------------------- | ------------------------------------------------------------------- |
| v0 (current)       | Pure semantic vector search     | Good starting point for most use cases                              |
| Hybrid (v1)        | Vector + PostgreSQL FTS (70/30) | When exact error codes / CVE IDs are being missed in retrieval      |
| Reranker (v2)      | Hybrid + Cohere/Jina reranker   | When retrieval quality still falls short at 50+ incident scale      |
| External BM25 (v3) | Elasticsearch/Typesense         | Only needed at 5M+ documents or when search UX is a product feature |

> **Recommendation:** Start with pure semantic (current setup). Move to hybrid only once you have 100+ real incidents and observe specific misses. The Postgres-native hybrid approach covers 80% of production needs without additional infrastructure.

---

## 4. Adding Targeted Threat Intel APIs

The v0 External Threat Intel branch uses only **Tavily** for web search. In production, you should progressively add direct API integrations for structured threat data. These are far more reliable and cost-effective than web search for known CVEs, IPs, and hashes.

### The Layered Search Architecture

```
Incident Alert
      │
      ▼
Layer 1: Internal RAG (always)     ← Supabase vector store
      │ No strong match
      ▼
Layer 2: Structured Threat Intel   ← Direct APIs (CVE, IP, hash)
      │ Still insufficient
      ▼
Layer 3: Web Search (conditional)  ← Tavily (current), SerpAPI, Exa AI
```

### API Integrations to Add (Priority Order)

#### 🟢 Immediate: Free, no auth required

| API              | What It Provides                                     | Endpoint                                                                              | Trigger                        |
| ---------------- | ---------------------------------------------------- | ------------------------------------------------------------------------------------- | ------------------------------ |
| **CISA KEV**     | Known exploited vulnerabilities list                 | `https://www.cisa.gov/sites/default/files/feeds/known_exploited_vulnerabilities.json` | Any CVE found in incident      |
| **NVD (NIST)**   | Official CVSS score, affected products, patch status | `https://services.nvd.nist.gov/rest/json/cves/2.0?cveId=CVE-...`                      | CVE ID in alert                |
| **MITRE ATT&CK** | Technique details, mitigations                       | STIX bundle or `https://attack.mitre.org/api/`                                        | MITRE technique ID in incident |

#### 🟡 Soon: Free tier sufficient for most use cases

| API                | What It Provides                                     | Free Tier     | Auth    |
| ------------------ | ---------------------------------------------------- | ------------- | ------- |
| **AbuseIPDB**      | IP reputation, abuse reports, malware classification | 1,000 req/day | API key |
| **VirusTotal**     | File hash malware classification, 70+ AV engines     | 500 req/day   | API key |
| **AlienVault OTX** | Threat intel pulses, IoCs, attacker campaigns        | Free          | API key |

#### 🔴 Later: Enterprise SOC tooling

| API                 | What It Provides                               | Notes              |
| ------------------- | ---------------------------------------------- | ------------------ |
| **Shodan**          | Internet-facing exposure of affected systems   | Paid               |
| **Recorded Future** | Commercial threat intel, threat actor profiles | Enterprise pricing |
| **ExploitDB**       | Verified exploit code for CVEs                 | Free, scraping     |

### n8n Implementation Pattern

All these APIs use standard HTTP requests — no special n8n nodes needed:

```
CVE Pattern Detected in Alert?
         │
         ▼
HTTP Request → NVD API → Extract CVSS, patch version, affected range
         │
         ▼
HTTP Request → CISA KEV → Is this actively exploited in the wild?
         │
         ▼
Pass structured intel to Synthesizer (alongside Tavily results)
```

Add these as IF node branches in the External Threat Intel section of the workflow, triggered by pattern matching on the incident data (CVE pattern, IP address, file hash).

### Web Search Tool Progression

When direct APIs don't have the answer, web search fills the gap:

| Tool                 | Best For                                 | Use Case                                                                            |
| -------------------- | ---------------------------------------- | ----------------------------------------------------------------------------------- |
| **Tavily** (current) | CVE lookups, threat intel synthesis      | Set `includeDomains: ["nvd.nist.gov", "cisa.gov"]` for pinned authoritative sources |
| **SerpAPI**          | Site-specific GitHub issues, vendor KB   | Use `site:github.com "node.js v22.5.0"` style targeted queries                      |
| **Exa AI**           | Novel attack patterns, semantic research | Neural search that understands concepts, not just keywords                          |
| **Jina AI**          | Reading full advisory pages              | Extracts clean markdown from any URL — use after Tavily finds the advisory link     |

---

## 5. Integrating Your Ticketing System

Out of the box, the workflow stores generated reports in Supabase (`test_incidents_v0.output`). In production, you'll want reports to flow into your existing incident management tooling.

### Common integrations (all via n8n HTTP Request or native nodes)

| System              | How to Connect                      | What to Do                                                                            |
| ------------------- | ----------------------------------- | ------------------------------------------------------------------------------------- |
| **PagerDuty**       | PagerDuty n8n node or REST API      | Add a note to an existing incident, or create a new one                               |
| **Jira/JSM**        | Jira n8n node                       | Create an Issue with the triage report as the description, set priority from severity |
| **ServiceNow**      | HTTP Request to ServiceNow REST API | Create or update an incident record                                                   |
| **Slack**           | Slack n8n node                      | Post the report summary to an on-call channel                                         |
| **Microsoft Teams** | Teams webhook or HTTP Request       | Post adaptive card with key findings                                                  |
| **OpsGenie**        | HTTP Request to OpsGenie API        | Create alert with structured fields                                                   |

### Integration architecture pattern

Add a node at the end of the retrieval workflow (after the Synthesizer writes to Supabase):

```
Synthesizer → Supabase Write → [Route by severity]
                                     │
                          ┌──────────┴──────────┐
                          ▼                     ▼
                    Critical/High          Medium/Low
                          │                     │
                    PagerDuty alert       Slack message
                    + Jira ticket         only
```

### SIEM / EDR Webhook as Trigger

To replace the manual trigger with real-time incident response:

1. Configure your SIEM (Splunk, Chronicle, Elastic SIEM) or EDR (CrowdStrike, SentinelOne) to send webhook alerts
2. Add a **Webhook** trigger node at the start of the retrieval workflow
3. Map your SIEM alert fields to the expected incident JSON schema (or add a transformation step)

> **Important:** The richer your alert payload, the better the RAG retrieval. See [Section 6](#6-mitre-attck-ticket-format-requirements) for what fields matter most.

---

## 6. MITRE ATT&CK Ticket Format Requirements

The effectiveness of the RAG-based retrieval depends directly on the quality of the metadata stored alongside each past incident. The most critical fields are the MITRE ATT&CK identifiers.

### Why MITRE metadata matters

The metadata filtering layer uses these fields to narrow retrieval to incidents with the same attack pattern:

```
Test incident: { mitre_technique_ids: ["T1110.001"] }

Vector search finds 50 semantically similar incidents
Metadata filter narrows to: incidents with T1110.001 in mitre_technique_ids
Result: 3 highly-targeted past brute force incidents with proven mitigations
```

Without MITRE metadata, the filter has nothing to narrow on and falls back to pure semantic similarity — which is less precise.

### Required fields for production effectiveness

| Field                  | Format          | Example                  | Why It Matters                                                   |
| ---------------------- | --------------- | ------------------------ | ---------------------------------------------------------------- |
| `mitre_tactic_ids`     | `["TA0006"]`    | `["TA0001", "TA0006"]`   | Tactic-level filtering (Initial Access, Credential Access, etc.) |
| `mitre_technique_ids`  | `["T1110.001"]` | `["T1190", "T1059.001"]` | Technique-level filtering — most specific, most valuable         |
| `mitre_prevention_ids` | `["M1032"]`     | `["M1032", "M1036"]`     | Maps to proven mitigations for this specific technique           |

### MITRE ID format rules

- **Tactics** use `TA` prefix: `TA0001` through `TA0043`
- **Techniques** use `T` prefix, sub-techniques use dot notation: `T1110`, `T1110.001`
- **Mitigations** use `M` prefix: `M1032` (MFA), `M1036` (Account Use Policies), etc.
- All IDs must be exact — the metadata alignment agent does fuzzy-matching, but exact IDs are far more reliable

### Where to get MITRE IDs

| Source                                                                        | How to Use                                        |
| ----------------------------------------------------------------------------- | ------------------------------------------------- |
| [MITRE ATT&CK Navigator](https://mitre-attack.github.io/attack-navigator/)    | Browse and identify relevant techniques visually  |
| [ATT&CK Matrix for Enterprise](https://attack.mitre.org/matrices/enterprise/) | Full matrix with technique IDs                    |
| SIEM tools (Splunk, Elastic)                                                  | Many now auto-tag alerts with MITRE technique IDs |
| EDR platforms (CrowdStrike, SentinelOne)                                      | Alert schemas often include MITRE mappings        |

### Practical guidance for real incident tickets

When ingesting real incidents from your SIEM or ticketing system:

1. **Add a MITRE tagging step** — use an AI node or lookup table to map your SIEM alert categories to MITRE technique IDs before storing in `resolved_incidents_v0`
2. **Minimum viable tagging:** At least one `mitre_technique_id` per incident is sufficient to unlock metadata filtering
3. **Missing MITRE data:** If your historical incidents don't have MITRE tags, use an LLM pre-processing step to infer them from the incident description before ingestion

---

## 7. Customising Any Part with AI Assistance

The code nodes, system prompts, and data transformation logic in this workflow are all **plain text** — designed to be read and modified by anyone, including with AI assistance.

### Modifying the Code Node (Data Preparation)

The `Prepare Markdown Document & Metadata` Code Node is where incident JSON is transformed into embeddable markdown. If you want to:

- Add a new metadata field
- Change how the markdown is structured
- Add or remove incident fields from the embedded content
- Change how MITRE IDs are extracted

**Do this:** Copy the full Code Node JavaScript from your n8n workflow. Paste it into any AI model (Claude, ChatGPT, Gemini) with a prompt like:

> _"Here is a JavaScript n8n Code Node that transforms incident JSON into markdown for embedding. [paste code]. I want to add a new metadata field called `environment` (values: production, staging, dev) and include it both in the metadata object and in the markdown header section. Give me the modified code."_

The AI will return a precise, drop-in replacement. No deep JavaScript knowledge required.

### Modifying System Prompts

The workflow has multiple agents with system prompts, and they fall into two categories:

**The Final Synthesizer** has no structured output parser — it produces a free-form markdown triage report. You can safely edit its system prompt to add, remove, or reformat report sections without touching any other node.

**The intermediate agents** (Threat Intel Agent, Historical Incidents retrieval agent, Playbook Routing agent) each have a **Structured Output Parser** node attached. These parsers enforce a specific JSON schema — `threat_intel_cards`, `mitigation_steps`, metadata filter fields, etc. If you modify what any of these agents output (add a new field, remove a field, rename a field), you must also update the JSON schema in the corresponding Structured Output Parser node.

**Practical guidance for editing intermediate agent prompts:**

1. Open the agent's system prompt in n8n
2. Also open the Structured Output Parser node that is connected to that agent
3. Paste both (the system prompt AND the parser JSON schema) into an AI model with a prompt like:

> _"Here is a system prompt for a Threat Intel Agent [paste prompt] and its JSON output schema [paste schema]. I want to add a new field `cve_score` (number, nullable) to the schema and update the system prompt to instruct the agent to populate it. Show me the updated schema and prompt."_

The AI will update both consistently. Change one without the other and the structured parser will reject or silently ignore the agent's output.

**Which nodes have structured output parsers?**

| Node                        | Has Structured Output Parser? | When to Update Parser                                       |
| --------------------------- | ----------------------------- | ----------------------------------------------------------- |
| Threat Intel Agent (Tavily) | ✅ Yes                        | If adding/removing fields from threat intel output          |
| Historical Incidents Agent  | ✅ Yes                        | If changing what metadata filter fields are extracted       |
| Playbook Routing Agent      | ✅ Yes                        | If changing routing logic or adding new playbook categories |
| Final Synthesizer           | ❌ No                         | Always safe to edit system prompt directly                  |

### Changing the Supabase Schema

> **Key clarification:** Adding a new metadata field to your incidents does **not** require a Supabase schema change in most cases.

Here is why: the `resolved_incidents_v0` table stores all metadata in a single **JSONB column** called `metadata`. JSONB is flexible — it accepts any JSON key-value pairs without a schema migration. When you add a new field in the Code Node (e.g., `environment: "production"`), it flows into this JSONB column automatically with no SQL changes.

**A Supabase schema change IS required when:**

| Change                                                      | Schema Update Needed? | What to Change                                                                              |
| ----------------------------------------------------------- | --------------------- | ------------------------------------------------------------------------------------------- |
| Adding a new metadata field (e.g., `environment`, `team`)   | ❌ No                 | Only the Code Node needs updating                                                           |
| Changing the embedding model (different dimensions)         | ✅ Yes                | Update `vector(3072)` to the new dimension in `supabase_schema_v0.sql`                      |
| Adding a new **separate SQL column** (not a metadata field) | ✅ Yes                | ALTER TABLE + update match functions                                                        |
| Upgrading from pure vector to hybrid search                 | ✅ Yes                | Add the `match_documents_hybrid` RPC function (see [Section 3](#3-moving-to-hybrid-search)) |

If you do need a schema change, paste `supabase_schema_v0.sql` into any AI model with a description of the change:

> _"Here is a Supabase SQL schema file. [paste SQL]. I want to update the `vector(3072)` column to `vector(1536)` to use a different embedding model. Show me the updated SQL and any dependent functions I need to update."_

Run the output in your Supabase SQL editor (Dashboard → SQL Editor → New Query).
