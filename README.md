# Enterprise Agentic Calendar Governance & Orchestration Architecture

An enterprise-grade, decoupled Agentic AI architecture demonstrating the **Gatekeeper Pattern**, externalized business rule governance in **Microsoft Dataverse**, and deterministic execution guardrails using **Microsoft Copilot Studio**, **Power Automate Agent Flows**, and the **Microsoft Graph API**.

---

## 1. Executive Summary & Design Rationale

In enterprise environments, generative models and LLM agents operate probabilistically. While valuable for intent classification, entity extraction, and conversational synthesis, **unconstrained agents must never possess direct mutation rights (write/update/delete) on core enterprise systems of record**. 

If business rules are embedded directly into prompt instructions, systems become vulnerable to:
1. **Prompt injection & jailbreaking** (malicious or unintended user prompts overriding rules).
2. **Instruction drift & hallucination** (model probabilistic variance causing rule bypass).
3. **Rigid deployment cycles** (requiring AI engineers to alter prompts whenever business policies change).

### The Solution: The Decoupled "Dual-Check" Gatekeeper Architecture
This POC decouples policy, intelligence, and execution:
* **Decoupled Policy (Dataverse):** Business rules live dynamically in a secure Dataverse table, manageable by executive admins without code deployment.
* **Bounded Intelligence (Copilot Studio Agent):** The agent receives read-only context to inspect schedules and evaluate rules, but is **intentionally denied direct calendar modification tools**.
* **Deterministic Gatekeeper Flow (`ModifycalendarWithRuleCheck`):** When the agent proposes a modification, it must invoke a secondary, deterministic workflow. This flow independently queries the Dataverse rule engine, validates the target event against enterprise protection criteria, and executes the Graph API mutation only upon verification.

---
How it works
Think of it as an executive assistant working with a policy handbook and a separate approver:

| Component | Beginner-friendly role |
|---|---|
| Outlook | Holds the meetings and signals when a new one appears. |
| Copilot Studio agent | Investigates conflicts and proposes a replacement time. |
| Dataverse | Holds editable business rules, such as “board meetings cannot move.” |
| Microsoft Graph | Provides the interface for reading and updating calendar events. |
| Gatekeeper flow | Independently checks the proposal before updating the calendar. |

A typical request follows this path:

flowchart LR
    A[New Outlook event] --> B[Agent reads rules and calendar]
    B --> C{Real overlap?}
    C -->|No| D[Leave calendar unchanged]
    C -->|Yes| E[Choose movable meeting and new time]
    E --> F[Gatekeeper checks proposal]
    F -->|Allowed| G[Update calendar]
    F -->|Rejected| H[Leave event unchanged]

For example, if a normal meeting overlaps a protected board meeting, the agent should preserve the board meeting and find another slot for the normal meeting within two business days. It preserves the meeting’s duration and treats back-to-back meetings as nonconflicting. When the newly created event is protected and conflicts, it also sends the configured assistant the verified outcome.

The Gatekeeper reads the event and rules again, requires an ALLOW classification, checks duration and other conditions, then submits the update. Its rule interpretation still uses AI, so this is an independent check with deterministic safeguards, rather than fully deterministic policy enforcement.

---

## 2. End-to-End Sequence Diagram

```mermaid
sequenceDiagram
    autonumber
    actor Organizer as Meeting Organizer
    participant Cal as Exchange / M365 Calendar
    participant Flow1 as Flow 1: Ingestion Orchestrator
    participant Agent as Copilot Studio Agent
    participant DV as Dataverse (cr_calendar_governance_rule)
    participant Graph as Microsoft Graph API
    participant Flow2 as Flow 2: ModifycalendarWithRuleCheck

    Organizer->>Cal: Creates new calendar event
    Cal->>Flow1: Trigger: When an upcoming event is created (Webhook/Graph)
    Flow1->>Agent: Invoke Agent with Event Context (Subject, Start, End, Organizer)
    
    Agent->>DV: Tool Call: Fetch active governance rules (cr_natural_language_rule)
    DV-->>Agent: Return: "Board meetings are critical and cannot be moved..."
    
    Agent->>Graph: Tool Call: Search Calendar (HTTP w/ Entra ID OAuth)
    Graph-->>Agent: Return existing calendar schedule & potential conflicts
    
    Note over Agent: Agent reasons on schedule.<br/>Detects need to reschedule conflicting meeting.<br/>Agent CANNOT modify calendar directly.
    
    Agent->>Flow2: Tool Invocation: Request Event Modification<br/>(EventID, TargetSubject, ProposedStart, ProposedEnd)
    
    rect rgb(240, 248, 255)
    Note over Flow2: GATEKEEPER DEFENSE-IN-DEPTH
    Flow2->>DV: Independent Query: Fetch active rule "RULE-CAL-001"
    DV-->>Flow2: Return rule criteria
    Flow2->>Graph: Verify actual target meeting details from Exchange
    Graph-->>Flow2: Return Target Meeting Metadata
    
    alt Target Event Subject contains "Board Meeting"
        Note over Flow2: DETERMINISTIC BLOCK TRIGGERED
        Flow2-->>Agent: HTTP 200: "Board meetings are protected and cannot be moved."
        Agent-->>Flow1: Report: Action rejected per governance policy
    else Target Event is Standard Meeting
        Flow2->>Graph: PATCH /v1.0/me/events/{id} (Execute Update)
        Graph-->>Flow2: 200 OK (Event Updated)
        Flow2-->>Agent: HTTP 200: "Meeting successfully rescheduled."
        Agent-->>Flow1: Report: Action completed successfully
    end
    end
