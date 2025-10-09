---
layout: post
title: "Everything You Need to Know About Sentinel Data Lake"
date: 2025-10-09
categories: [security, sentinel, azure]
tags: [microsoft-sentinel, data-lake, security-operations, azure-security, siem]
excerpt: "A comprehensive guide to Microsoft Sentinel Data Lake as at October 2025"
---

As with everything Microsoft Cloud, this product is going to be in a state of constant flux. Below are my notes on Data Lake, enhanced with information from official sources and organised by AI.

## 1. Overview

* **Announced:** Public Preview on **22 July 2024**, General Availability on **30 September 2024**.

* **Purpose:** Unifies security data across an organisation in a cost-effective way, enabling long-term retention and multi-modal analysis (KQL, Jupyter, Spark, and future graph capabilities).

* **Management Interface:** Entirely managed through the **Microsoft Defender portal** (not Azure Portal).

* **Setup Requirements:**
  
  * **Subscription Owner** or **User Access Administrator** permissions for billing/resource provisioning
  * **Security Administrator** or **Global Administrator** (Entra ID) for configuration
  * For new customers (on/after July 1, 2025), automatic onboarding to Defender portal may occur
  * Setup and onboarding guidance available via Microsoft Learn

* **Important Note:** Microsoft Sentinel in the Azure portal will be retired by **July 2026**.

---

## 2. What You Get

### 2.1 Graph-powered Interactive Visualisation *(Public Preview)*

* Enables **graph-based investigation and threat hunting** directly in the Defender portal.
* Graphs draw data from multiple sources (e.g. Sentinel, Entra ID, M365, Defender suite).
* Designed to assist with exploring **incident relationships** and **latent threat paths**.
* Serves as foundation for agentic defence and deeper security insights.

### 2.2 Query Over Historical Data

* Use **KQL** to query all retained data for threat hunting, anomaly detection, and behavioural or predictive analytics.
* **Spark** is available for advanced processing and ML workloads via Microsoft-managed Spark clusters provisioned automatically.
* Accessed through the **Visual Studio Code Sentinel extension** for notebook management.
* Supports AI agents via the **MCP server**, allowing tools like GitHub Copilot to query and automate security tasks.

### 2.3 Built-in, Flexible Data Tiers

* Offers **purpose-built tiers** for SOC operations:
  
  * **Analytics tier:** For hot data powering ongoing detections, automation, and SOAR.
  * **Data Lake tier:** For long-term storage of raw or normalised data at lower cost.

* Data can flow seamlessly between tiers, with mirroring for unified access.

### 2.4 Data from Everywhere

* Integrates data from **350+ native connectors**, spanning cloud, on-premises, and third-party sources.
* Supports **custom connector creation**.
* Automatically includes **asset data** from Microsoft 365, Entra ID, and Azure without additional setup.

### 2.5 Centralised Cost Management

* Central dashboard for managing and optimising data costs:
  
  * Track usage and estimate charges
  * Onboard data sources
  * Adjust retention and tiering policies
  * Manage storage costs and analytics vs lake retention

* **10 GB/day trial** available for 31 days

* **Commitment tiers** start at **100 GB/day** with predictable pricing

### 2.6 Platform Architecture

* **Storage architecture components:** 
  * **Analytics tier** - Hot data for real-time detection and SOAR
  * **Data Lake tier** - Long-term, cost-effective storage
  * **Asset store** - Asset metadata and entity information (platform component)
  * **Activity store** - Activity logs and security events (platform component)
  * **TI Store** - Dedicated Threat Intelligence storage (platform component)
* **AI/ML layer:** Includes models, embeddings, and entity analyser for advanced analytics
* **SIEM storage:** Bridges traditional SIEM capabilities with modern data lake architecture
* **Semantic capabilities:** Graph database and embeddings enable semantic search across all security data
* **Deep Security Copilot integration:** Bidirectional integration for AI-powered security operations

---

## 3. Data Ingestion and Mirroring

### 3.1 Ingestion Methods

* Uses **existing Sentinel connectors and DCRs** — no reconfiguration required.

* Data can be:
  
  * Sent **directly** to the Data Lake, or
  * **Mirrored** from the Analytics tier automatically.

* Data ingested into analytics tier is automatically mirrored to the lake, preserving a single copy.

* **Unified data connectors** feed into the entire platform, not just the data lake.

### 3.2 Mirroring

* Automatically synchronises data between **Analytics (hot)** and **Data Lake (cold)** tiers.
* **Free of charge** when Data Lake retention matches Analytics retention (both at 90 days).
* **Analytics tier** has 90 days free retention; extending beyond 90 days incurs costs for both tiers.
* **Additional cost** applies only for extended Data Lake retention beyond the Analytics tier retention period.
* **Existing data** in the Analytics tier is *not retroactively moved* into the Lake — mirroring is forward only.
* Creates a single copy of data accessible across both tiers without duplication charges.

---

## 4. Permissions and Onboarding

* **Pre-requisite:** Sentinel workspace must be **connected to the Defender portal** before onboarding.

* **Roles and Access:**
  
  * Setup requires **Subscription Owner** or **User Access Administrator** for billing and resource provisioning
  * **Security Administrator** or **Global Administrator** (Entra ID) for data ingestion authorisation
  * **Log Analytics Contributor** permission required to modify table settings in Azure portal/Log Analytics
  * In Defender portal, table management requires **Security Administrator/Operator** (Entra ID) or **Data (Manage)** Unified RBAC permission

* All management occurs through **Defender Portal → Microsoft Sentinel → Configuration → Tables / Data Connectors**.

* **Unified RBAC:** Starting July 2025, data lake permissions provided through Microsoft Defender XDR unified RBAC.

### 4.1 Critical Prerequisites for Data Lake Onboarding

#### Defender Portal Connection (Mandatory First Step)

* **CRITICAL:** Sentinel workspace MUST be connected to Defender portal BEFORE data lake onboarding
* Navigate to: Defender Portal → System → Settings → Microsoft Sentinel → Connect a workspace
* Select and designate a primary workspace during connection
* This is a separate prerequisite step, NOT part of the data lake setup itself

#### Subscription Ownership Requirements

* Must be **direct subscription owner** or **User Access Administrator** - management-group-level ownership is insufficient
* Required for billing setup and resource provisioning
* For new customers (on/after July 1, 2025), these permissions may result in automatic onboarding to Defender portal

#### Regional Limitations & Consent

* Data lake provisions in same region as primary Sentinel workspace
* Only workspaces in same region as primary can attach to data lake
* **Consent Required:** If Microsoft 365 data resides in different region, you consent to data ingestion into data lake region
* **BCDR Not Supported in:**
  * EU customers (EUDB compliance limitations)
  * Israel
  * Azure operated by 21Vianet
* **Data Lake BCDR Specifically Not Supported in:** Australia East, UK South, Switzerland North, Canada Central, Japan East, Central India, Southeast Asia, France Central, Israel Central

#### Policy Exemptions

* Azure Policy definitions may block deployment of required resources
* Configure policy exemption scoped to the resource group during onboarding
* Specifically exempt resource type: `Microsoft.SentinelPlatformServices/sentinelplatformservices`
* DL103 error indicates policies preventing creation of Azure managed resources

#### Important Limitations

* **CMK Not Supported:** Customer-Managed Keys not supported at GA - data lake uses Microsoft-managed keys only
* **No Retroactive Migration:** Existing analytics tier data not moved to lake - forward mirroring only
* **Data Availability Delays:** 
  * **Configuration changes**: Take effect within 1-2 minutes
  * **Data visibility after tier switch or first-time enablement**: 90-120 minutes for data to appear in the new tier
  * **Asset Data population**: Can take up to 24 hours to fully populate
  * **Regular data ingestion**: No additional delays once configured
* **Managed Identity Created:** Onboarding creates managed identity with prefix 'msg-resources-' followed by GUID
  * Requires Azure Reader role over subscriptions
  * For custom table creation, needs Log Analytics Contributor role
* **Auxiliary Logs:** Once integrated into data lake, accessible through Data Lake exploration but no longer available in Advanced Hunting
* **Basic Logs:** NOT supported in Data Lake tier - must convert to Analytics tier first

---

## 5. Data Lake Permissions and Access Control

### 5.1 Dual Permission Model Overview

The Microsoft Sentinel Data Lake uses a **dual-model permission structure** that combines:

1. **Microsoft Entra ID Roles** - Provide tenant-wide, broad access across ALL workspaces in the data lake
2. **Azure RBAC / Defender XDR Unified RBAC** - Provide granular, workspace-specific permissions

This represents a significant shift from traditional Sentinel RBAC, particularly for job management and cross-workspace operations.

### 5.2 Permission Requirements by Role

#### Read/Query Permissions

| Scope                  | Permission Type           | Required Roles/Permissions                                                                                                             |
| ---------------------- | ------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| **All workspaces**     | Microsoft Entra ID        | • Global Reader<br/>• Security Reader<br/>• Security Operator<br/>• Security Administrator<br/>• Global Administrator                  |
| **System tables**      | Defender XDR Unified RBAC | Custom role with "Security data basics (read)" permission                                                                              |
| **Specific workspace** | Azure RBAC                | • Log Analytics Reader<br/>• Microsoft Sentinel Reader<br/>• Microsoft Sentinel Contributor<br/>• Reader<br/>• Contributor<br/>• Owner |

#### Write Permissions

| Scope                    | Permission Type           | Required Roles/Permissions                                                                                                                                                                                                                                                          |
| ------------------------ | ------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **All data lake tables** | Microsoft Entra ID        | • Security Operator<br>• Security Administrator<br>• Global Administrator                                                                                                                                                                                                           |
| **System tables**        | Defender XDR Unified RBAC | Custom role with "Data (Manage)" permission                                                                                                                                                                                                                                         |
| **Specific workspace**   | Azure RBAC                | Roles with these permissions:<br>• `microsoft.operationalinsights/workspaces/write`<br>• `microsoft.operationalinsights/workspaces/tables/write`<br>• `microsoft.operationalinsights/workspaces/tables/delete`<br><br>Built-in roles: Log Analytics Contributor, Owner, Contributor |

#### Job Management Permissions

| Task                        | Required Permission                                                                              | Important Note                                 |
| --------------------------- | ------------------------------------------------------------------------------------------------ | ---------------------------------------------- |
| **Create/manage KQL jobs**  | Microsoft Entra ID:<br>• Security Operator<br>• Security Administrator<br>• Global Administrator | ⚠️ **No workspace-level delegation available** |
| **Schedule analytics jobs** | Defender XDR Unified RBAC:<br>"Analytics Jobs Schedule" (Read/Manage)                            | Requires Unified RBAC activation               |

### 5.3 Defender XDR Unified RBAC Permissions (Preview)

Starting July 2025, new granular permissions are available through Microsoft Defender XDR Unified RBAC:

| Permission                  | Level  | Capabilities                                                                                                                                                                                       |
| --------------------------- | ------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Data**                    | Manage | • Manage data retention policies<br>• Move data between tiers (Analytics ↔ Data Lake)<br>• Create and delete data lake tables<br>• Configure and manage data connectors<br>• Modify table settings |
| **Analytics Jobs Schedule** | Read   | • View scheduled jobs and their configurations<br>• Access Lake Exploration (read-only)<br>• View notebook outputs                                                                                 |
| **Analytics Jobs Schedule** | Manage | • Create and modify scheduled KQL jobs<br>• Use Lake Exploration with write capabilities<br>• Execute notebooks in Azure Data Explorer<br>• Enable/disable/delete existing jobs                    |
| **Security data basics**    | Read   | • Query all data lake tables<br>• Access system tables<br>• View data lake experiences in Defender portal                                                                                          |

### 5.4 Permission Requirements by Common Tasks

| Task                                     | Minimum Permission Required                                                        | Notes                             |
| ---------------------------------------- | ---------------------------------------------------------------------------------- | --------------------------------- |
| **Run KQL queries (all workspaces)**     | Security Reader (Entra ID)                                                         | Broadest access                   |
| **Run KQL queries (specific workspace)** | Log Analytics Reader (Azure RBAC)                                                  | Workspace-scoped                  |
| **Create scheduled KQL jobs**            | Security Operator (Entra ID)                                                       | Cannot be delegated per workspace |
| **Modify table retention/tiers**         | Security Operator (Entra ID) OR<br>Custom role with "Data (Manage)"                | Affects billing                   |
| **Write query results to tables**        | Security Operator (Entra ID)                                                       | Via KQL jobs                      |
| **Configure data connectors**            | Custom role with "Data (Manage)" OR<br>Microsoft Sentinel Contributor (Azure RBAC) | Per workspace                     |
| **Use Jupyter notebooks**                | Security Operator (Entra ID) OR<br>"Analytics Jobs Schedule (Manage)"              | Requires VS Code                  |
| **View job status/history**              | "Analytics Jobs Schedule (Read)"                                                   | Read-only access                  |

### 5.5 Key Limitations and Considerations

#### Critical Limitations

1. **Job Management is All-or-Nothing**
   
   - No workspace-level job permissions available
   - Requires high-privilege Entra ID roles (Security Operator minimum)
   - Cannot delegate job creation to workspace-specific teams

2. **System Tables Special Handling**
   
   - Require Defender XDR Unified RBAC custom roles
   - Cannot be managed through traditional Azure RBAC

3. **Cross-Workspace Operations**
   
   - Require Entra ID roles for any operation spanning multiple workspaces
   - No way to grant cross-workspace access without tenant-wide permissions

#### Migration Considerations

- **Existing Azure RBAC roles continue to work** for workspace-specific operations
- **Microsoft Sentinel Reader/Contributor** roles automatically work with data lake for their assigned workspaces
- **New capabilities** (job management, cross-workspace queries) require Entra ID roles
- **Unified RBAC activation** required for granular permissions (available July 2025+)

### 5.6 Activating Defender XDR Unified RBAC

To use the new granular permissions:

1. Navigate to Defender Portal → **Permissions** → **Microsoft Defender XDR** → **Roles**
2. Activate Unified RBAC for your tenant
3. Create custom roles with specific Data Lake permissions
4. Assign roles to users/groups with appropriate data source scopes

**Important:** Once activated, some traditional RBAC behaviours may change. Test in a non-production environment first.

---

## 6. Table Management

### 6.1 Configuration Options

* Manage table-level storage and retention directly in the portal.

* Choose:
  
  * **Analytics tier** or **Data Lake tier** or **Both**
  * **Analytics retention:** up to 2 years
  * **Total retention:** up to 12 years

* Changes apply **only to new data** — existing data remains in its current tier.

### 6.2 Tier Guidance

* **Analytics tier:** Best for high-fidelity data (EDR, email, identity, SaaS, cloud logs) requiring real-time detection.
* **Data Lake tier:** Ideal for high-volume, low-fidelity logs (NetFlow, firewall, proxy logs).

### 6.3 Important Notes

* Moving tables to **Data Lake ONLY** disables ingestion to Analytics tier, which impacts:
  * **Analytics rules** that rely on hot data
  * **Hunting queries** expecting Analytics tier data
  * **SOAR playbooks** that depend on current data
  * These features won't get new data but may still function on historical Analytics tier data
* **Auxiliary logs** are automatically integrated into the Data Lake.
* **Basic Logs** are NOT supported in Data Lake tier - must be converted to Analytics tier first.
* **Basic and Auxiliary Logs** users should plan migration as no further enhancements are planned for these legacy structures.

---

## 7. Querying Data in the Data Lake

### 6.1 KQL Query Interface

* Found under **Defender Portal → Data Lake Exploration → KQL Queries**.

* UI mirrors *Advanced Hunting*:
  
  * Tables list (left)
  * KQL editor (centre)
  * Query results (bottom)

* Supports full KQL capabilities including machine learning functions and advanced analytics.

### 6.2 Asset Data ("Default Scope")

* Automatically includes rich **asset metadata** from the default workspace:
  
  * **Azure Resource Graph tables:**
    * `ARGAuthorizationResources`
    * `ARGResourceContainers`
    * `ARGResources`
  * **Entra ID tables:**
    * `EntraApplications`
    * `EntraGroupMemberships`
    * `EntraGroups`
    * `EntraMembers`
    * `EntraOrganizations`
    * `EntraServicePrincipals`
    * `EntraUsers`

* Provides contextual enrichment for investigations (e.g. group memberships, app access, device ownership).

* Appears under the **"default" workspace scope**, alongside Sentinel workspace data.

* **No setup required** — automatically populated on Data Lake onboarding.

* These tables store the **current state** of your environment, they are not chronological logs.

---

## 7. Data Lake Jobs

* **Data Lake jobs** = scheduled **KQL queries** that write results to:
  
  * A **new custom table** (suffix `_KQL_CL`), or
  * An **existing table** (schema of the KQL output must match the destination table).

* Configurable options include:
  
  * Job name, description, schedule, target workspace, and output table.
  * Supports both **ad hoc** and **recurring** runs.

* **Monitoring:** Job runs and status visible under **Defender Portal → Jobs**.

### 7.1 Comparison: KQL Jobs vs Summary Rules

| Feature           | KQL Jobs                                     | Summary Rules                          |
| ----------------- | -------------------------------------------- | -------------------------------------- |
| Query language    | Full KQL (JOINs, UNIONs, advanced operators) | Limited subset                         |
| Lookback period   | Up to 12 years                               | Up to 24 hours                         |
| Scheduling        | Customisable                                 | Fixed intervals                        |
| Output target     | Any custom/existing table                    | Analytics tier only                    |
| Data Lake support | ✅ Supported                                  | ✅ Supported (can query a single table) |

---

## 8. Jupyter Notebooks and Spark Integration

### 8.1 Overview

* Supports **Spark-backed Jupyter notebooks** for advanced analytics and ML workloads.
* Accessed through **Visual Studio Code** with the **Microsoft Sentinel extension**.
* Enables deep analysis for forensics, incident response, and anomaly detection.

### 8.2 Sentinel VS Code Extension

* View and manage:
  
  * Data Lake tables and schemas
  * Job definitions and run status

* Create, run, and schedule notebooks directly in VS Code.

* Integrates with **GitHub Copilot** for AI-assisted security operations.

### 8.3 Execution Environment

* Microsoft provisions and manages a **dedicated Spark cluster** for each Data Lake tenant.

* Select **kernel** and **compute pool size** when executing notebooks:
  
  * *Small:* Single-table queries
  * *Medium:* Joins and data aggregation
  * *Large:* Multi-table joins or ML workloads

* **Charged based on compute time** (per active session).

* Spark pools run **only during query execution**, reducing idle cost.

---

## 9. Microsoft Sentinel MCP Server *(Preview)*

### 9.1 Overview

* **Model Context Protocol (MCP)** server provides unified, hosted interface for AI-driven security operations.
* Enables natural language queries against security data without requiring schema knowledge.
* Integrates with **GitHub Copilot** and **VS Code** for building intelligent security agents.
* Uses **semantic search** and **embeddings** for intelligent data discovery.

### 9.2 Key Capabilities

* **Semantic Search:** Query tools find relevant tables based on natural language prompts.
* **Query Tools:** Execute KQL queries and retrieve data using conversational language.
* **Custom Analysis:** Build security agents that automate enrichment, anomaly detection, and forensics.
* **Entity Analysis:** Leverages entity analyser for advanced correlation.

### 9.3 Tool Collections

Available collections include:

| Collection           | Purpose                                             | URL                                                                  |
| -------------------- | --------------------------------------------------- | -------------------------------------------------------------------- |
| **Data Exploration** | Search tables and query data in natural language    | `https://sentinel.microsoft.com/mcp/data-exploration`                |
| **Agent Creation**   | Build Security Copilot agents for complex workflows | `https://sentinel.microsoft.com/mcp/security-copilot-agent-creation` |

### 9.4 Agent Creation Features

* Generate agent YAML files from natural language descriptions (e.g., "Build an agent that triages compromised accounts").
* Discover relevant Security Copilot tools and skills automatically.
* Deploy agents at user or workspace scope.
* Iterative development through conversational refinement.
* Integrates with **1p tools** (first-party Microsoft tools).

### 9.5 Integration Requirements

* Requires onboarding to Sentinel Data Lake.
* **Security Reader** role minimum for access.
* Works with Visual Studio Code and Security Copilot platforms.

---

## 10. Pricing and Billing

### 10.1 Consumption Model

* **Ingestion:** Data processing charge of $0.10/GB for data lake-only tables (not for mirrored data).
* **Storage:** Compressed at 6:1 ratio across all data sources, billed per GB/month.
* **Compute:** Charged for KQL queries and Spark notebook execution time.
* **Mirroring:** Free when retention matches analytics tier (90 days).

### 10.2 Cost Optimisation

* Use **commitment tiers** for predictable pricing.
* **Data Lake tier** costs less than 15% of traditional analytics logs.
* New **cost management experience** (preview) provides usage insights and alerts.
* Set usage-based alerts on specific metres (query usage, notebook time).

### 10.3 Important Notes

* Archive logs automatically transition to Data Lake billing upon enablement.
* Auxiliary logs switch to Data Lake metres after onboarding.
* XDR data requires retention >30 days to ingest beyond default tier.

---

## 13. Summary

| Area                 | Key Points                                                                                  |
| -------------------- | ------------------------------------------------------------------------------------------- |
| **Integration**      | Uses existing Sentinel connectors; no pipeline changes                                      |
| **Cost**             | Mirroring free within retention; <15% cost vs traditional logs                              |
| **Setup**            | Global/Security Admins with subscription ownership OR Security Admin + Sentinel Contributor |
| **Analysis Methods** | KQL, scheduled KQL jobs, Spark, Jupyter, graph visualisations, MCP/AI agents                |
| **Management**       | Centralised in Defender portal (Azure portal retiring July 2026)                            |
| **AI Capabilities**  | MCP server enables natural language queries and automated agent creation                    |
| **Storage Model**    | Three-tier architecture: Asset, Activity, and TI stores on raw storage                      |
| **Advantages**       | Unified storage, flexible analysis, rich asset context, cross-source visibility             |

---

## 14. Future Roadmap

* **SQL query support** planned for Data Lake exploration.
* **Consolidated KQL interface** merging Advanced Hunting and Data Lake exploration.
* **Enhanced MCP capabilities** for more sophisticated AI-driven security operations.
* **Customer-Managed Keys (CMK)** support for enhanced data sovereignty.
* **Government cloud availability** anticipated soon.