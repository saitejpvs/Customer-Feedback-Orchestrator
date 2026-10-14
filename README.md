# Customer Feedback Orchestrator & MCP Server Automation

An end-to-end multi-agent AI system built on **n8n** that automates the ingestion, analysis, and dispatching of customer feedback. The system combines an orchestrator-agent pattern with Model Context Protocol (MCP) tool invocation to process customer reviews, draft product insights, enforce human-in-the-loop (HITL) approval, and automatically update Google Drive and Gmail.

## Architecture Overview

The automation consists of two main n8n workflow components:

```
[Chat Trigger / CSV File] 
          │
          ▼
┌────────────────────────────────────────────────────────┐
│              Feedback Orchestrator Agent               │
└─────────────────────────┬──────────────────────────────┘
                          │
         ┌────────────────┴────────────────┐
         ▼                                 ▼
┌──────────────────────────┐    ┌─────────────────────────┐
│ Review Intelligence Agent│    │   Ops Dispatch Agent    │
│  - Theme Clustering      │    │  - Human Approval Gate  │
│  - Quote Extraction      │    │  - Calls MCP Client     │
│  - Fee Explainer         │    └────────────┬────────────┘
│  - SerpAPI Validation    │                 │
└──────────────────────────┘                 │ (MCP Protocol)
                                             ▼
                                ┌─────────────────────────┐
                                │       MCP Server        │
                                │  - Draft Gmail          │
                                │  - Create Drive Doc     │
                                └─────────────────────────┘

```

1. **Customer Feedback Orchestrator (Main Agent Workflow)**

   * Ingests feedback via chat interface or uploaded CSV file.
   * Delegates feedback analysis to a sub-agent (`Review Intelligence Agent`).
   * Presents analysis to the user and enforces a **Human-in-the-Loop approval gate**.
   * On affirmative approval, delegates publication to a sub-agent (`Ops Dispatch Agent`).

2. **MCP Server (Tool Execution Workflow)**

   * Operates as an MCP server trigger listening for tools requested by the `Ops Dispatch Agent`.
   * Integrates directly with **Gmail** and **Google Drive**.

## Components & Sub-Agents

### 1. Feedback Orchestrator (Primary Agent)

* **Model:** GPT-4o (`gpt-4o`)
* **Role:** Acts as the Principal Product & Support Operations Orchestrator. Coordinates sub-agents, manages memory context, and holds the approval gate before external publishing.

### 2. Review Intelligence Agent (Agent Tool)

* **Model:** GPT-4o (`gpt-4o`)
* **Tools:** `SerpAPI Tool`
* **Responsibilities:**
  * Analyzes customer reviews and clusters feedback into up to 5 key themes (ranks top 3).
  * Extracts verbatim quotes highlighting friction points.
  * Identifies fee-related confusion (e.g., auto-renewal charges).
  * Drafts a concise **Weekly Product Pulse** ($\le 250$ words).
  * Drafts a **Standardized Fee Explainer** ($\le 6$ bullet points) and verifies official links using `SerpAPI`.

### 3. Ops Dispatch Agent (Agent Tool)

* **Model:** GPT-4o (`gpt-4o`)
* **Tools:** `MCP Client`
* **Responsibilities:**
  * Invoked **only** after receiving human approval (`Yes`/`Approve`).
  * Triggers the MCP Server to log internal documentation and draft operational emails.

### 4. MCP Server Workflow

* **Trigger:** `@n8n/n8n-nodes-langchain.mcpTrigger`
* **Integrations:**
  * **Gmail Node:** Drafts/sends emails based on AI-generated parameters.
  * **Google Drive Node:** Generates text documents under target folders (e.g., `Mile5`).

## Setup & Prerequisites

### Prerequisites

* **n8n** instance (v1.5+ with LangChain support and MCP nodes enabled).
* **OpenAI API Key** (configured for GPT-4o).
* **SerpAPI Key** (for real-time web verification).
* **OAuth Credentials** configured in n8n for:
  * Google Drive API (`googleDriveOAuth2Api`)
  * Gmail API (`gmailOAuth2`)

## Installation & Configuration

1. **Import Workflows:**
   * Import `Customer Feedback Orchestrator - Agent as Tool.json` into n8n.
   * Import `MCP server.json` into n8n.

2. **Configure MCP Endpoint:**
   * Activate the `MCP server` workflow to generate its webhook URL.
   * Copy the webhook URL from the `MCP Server Trigger` node.
   * Paste the endpoint URL into the `MCP Client` node in the `Customer Feedback Orchestrator` workflow:
     ```
     https://<your-n8n-instance>/mcp/<webhook-id>
     ```

3. **Set Up Credentials:**
   * Attach your **OpenAI API** credential to the chat model nodes.
   * Attach your **SerpAPI** credential to the `SerpAPI Tool` node.
   * Attach your **Gmail** and **Google Drive** credentials to their respective nodes in the MCP Server workflow.

## Workflow Execution Flow

1. **Submit Input:** Send a chat message containing raw customer feedback or upload a CSV file with reviews.
2. **Review Synthesis:** The Orchestrator calls the `Review Intelligence Agent`, which formats a Product Pulse summary and Fee Explainer.
3. **User Approval Gate:** The Orchestrator outputs the generated summary and asks:
   > *"Do you approve appending this entry to internal documentation and generating the Gmail draft? (Yes/No)"*
4. **Dispatch Execution:**
   * Response = **"Yes"**: Orchestrator invokes `Ops Dispatch Agent` $\rightarrow$ triggers `MCP Client` $\rightarrow$ Creates Google Drive file & drafts Gmail message.
   * Response = **"No"**: Workflow halts; no external modifications occur.

## Key Features & Safety Rules

* **Human-in-the-Loop (HITL):** Strict condition prevents autonomous dispatch prior to human verification.
* **Single Execution Guard:** System prompts restrict sub-agents from being called multiple times per turn.
* **Separation of Concerns:** Intelligence extraction is isolated from dispatch operations via distinct memory buffer contexts.