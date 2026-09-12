# Enterprise System Design: Autonomous Banking Multi-Agent Engine

[![Domain: System Design](https://img.shields.io/badge/Domain-System%20Design-blue.svg)](#)
[![Focus: AI Architecture](https://img.shields.io/badge/Focus-Multi--Agent%20%26%20MCP-purple.svg)](#)
[![Compliance: Banking Grade](https://img.shields.io/badge/Security-PCI--DSS%20%2F%20PII%20Redacted-green.svg)](#)

A , production-ready system architecture designed to solve high customer support call volumes in retail banking through a multi-agent AI system, Model Context Protocol (MCP) integrations, zero-trust security controls, and fine-grained observability.

---

##  1. Business Problem & Scale Analysis

### Case Context
* **Monthly Inbound Calls:** 420,000 calls
* **Average Handling Time (AHT):** 4.0 minutes / 240 seconds
* **Target Query Profile:** 65% (~273,000 calls) consist of three repetitive queries:
  1. *Balance Inquiry* ("What's my balance?")
  2. *Ledger Clarification* ("What was this debit from my account?")
  3. *Fulfillment Request* ("Send me a cheque book.")

### Root Cause & Engineering Objective
Customers default to phone calls due to mobile app navigation friction (14+ screens). The objective is to design a secure, low-latency conversational architecture capable of safely answering these three query types while strictly adhering to banking regulatory constraints.

### System Non-Functional Requirements (NFRs)
* **Latency:** End-to-end response time $\le 2.5$ seconds (p95).
* **Availability:** $99.99\%$ uptime for state and routing services.
* **Security:** Zero PII leakage to public LLM endpoints; full auditability under PCI-DSS standards.
* **Concurrency:** Support peak load of $500$ concurrent user sessions.

---

##  2. Multi-Agent Delegation Strategy

The architecture avoids monolithic LLM prompts by delegating execution to domain-specific agents via an Orchestrator-Worker pattern.

| Agent | Responsibility | Backing System Target | Execution Guarantee |
| :--- | :--- | :--- | :--- |
| **Router Agent** | Intent classification, session context assembly, tool routing. | Redis Session Store | Deterministic fallback to human operator on uncertainty. |
| **Balance Agent** | Queries real-time ledger balances across liquid accounts. | Core Banking Ledger API | Read-Only data access; zero write permissions. |
| **Ledger Agent** | Analyzes recent debit transactions, merchant identifiers, and fraud flags. | Transaction History DB | Dynamic retrieval with pagination; PII masking enforced. |
| **Service Agent** | Executes multi-step workflows (e.g., cheque book dispatch). | Order Fulfillment Bus | Requires explicit step-by-step confirmation state. |

---

## 🔌 4. Model Context Protocol (MCP) Design

To ensure loose coupling and safety, agents interact with core systems solely through strongly-typed MCP interfaces. Raw SQL or direct API execution by the LLM is prohibited.

```json
{
  "mcp_version": "1.0",
  "tool_name": "execute_cheque_book_order",
  "description": "Dispatches a new cheque book to the user's primary registered address.",
  "parameters": {
    "account_id": { "type": "string", "required": true },
    "leaf_count": { "type": "integer", "enum": [25, 50, 100], "default": 25 },
    "confirmation_token": { "type": "string", "required": true }
  },
  "authorization_scope": "write:services:cheque_book"
}