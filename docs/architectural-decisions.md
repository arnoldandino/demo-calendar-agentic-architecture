# Architectural Decision Records (ADRs) & Consultative Boundary Analysis

This document details the architectural edge cases, design decisions, and system trade-offs identified during the decomposition of the calendar governance scenario.

---

## ADR 001: Separation of Intent from Mutation (The Gatekeeper Pattern)
* **Status:** Accepted
* **Context:** Generative agents are probabilistic by design. In enterprise systems of record (calendars, claims, financial ledgers), unauthorized mutations lead to regulatory, operational, and executive disruption.
* **Decision:** The Copilot Studio agent is strictly provisioned with read-only tools (`SearchCalendarEvents`, `FetchDataverseRules`). It is intentionally withheld any direct Graph API `PATCH`, `POST`, or `DELETE` capabilities. All state mutations must be delegated as an intentional request to a secondary workflow (`ModifycalendarWithRuleCheck`).
* **Consequence:** Eliminates prompt injection attack vectors against calendar data. All modifications are subject to hard-coded deterministic policy checks.

---

## ADR 002: Externalized Rule Engine via Microsoft Dataverse
* **Status:** Accepted
* **Context:** Embedding business rules directly into agent system instructions causes model drift, requires prompt engineering redeployments for policy changes, and limits cross-agent rule reuse.
* **Decision:** Store business governance rules as natural language strings in a Dataverse table (`cr_calendar_governance_rule`). Both the Agent (at runtime) and the Gatekeeper Flow (at validation time) independently read from this table.
* **Consequence:** Business administrators can update scheduling constraints dynamically. Both the reasoning layer and the execution layer remain synchronized against a single source of truth.

---

## ADR 003: Defensive Redundant Rule Validation ("Dual-Check")
* **Status:** Accepted
* **Context:** Even if an agent is grounded with Dataverse rules, an LLM could hallucinate or fail to classify an event correctly, passing an invalid update payload to the backend.
* **Decision:** The `ModifycalendarWithRuleCheck` flow does not trust the agent's evaluation. It performs an independent fetch of the Dataverse rule and inspects the raw event metadata retrieved directly from Microsoft Graph before committing any change.
* **Consequence:** Provides true defense-in-depth. If an agent attempts to reschedule a protected meeting, the execution flow catches the violation deterministically and aborts the operation.

---

## 5 Consultative Boundary Questions & Implemented POC Behaviors

### 1. What triggers a proposed calendar change?
* **Design Decision:** In Flow 1, the trigger is an incoming event creation (`When an event is added, updated or deleted (V3)`). The agent's evaluation role is conflict detection: it uses the Graph API search tool to verify if the newly requested time overlaps an existing commitment. If a conflict occurs, it calculates the resolution path.

### 2. Which meeting is moved during a conflict?
* **Design Decision:** The incoming event retains precedence unless the existing conflicting meeting is tagged as protected (e.g., Board Meeting) or higher tier. In this POC, the agent identifies the non-protected conflicting event and submits a reschedule request for that event.

### 3. How does the agent choose a new time slot?
* **Design Decision:** The agent executes `SearchCalendarEvents` with a parameter window of the next 5 business days (09:00–17:00 EST). It locates the earliest available open block that matches the original meeting's duration, preserving all attendee lists.

### 4. What happens if the newly created event itself is a Board Meeting?
* **Design Decision:** The agent matches the incoming subject against the Dataverse rule. Recognizing it as a protected entity, the agent will never generate a rescheduling request targeting this event. Any overlapping lower-priority meetings are proposed to be shifted instead.

### 5. What is the exact payload and validation contract for Flow 2?
* **Design Decision:** Flow 2 accepts:
  ```json
  {
    "eventId": "AAMkADk2...",
    "meetingSubject": "Q3 Board Meeting",
    "requestedStart": "2026-10-15T14:00:00Z",
    "requestedEnd": "2026-10-15T15:00:00Z",
    "agentReason": "Resolving scheduling overlap with client review"
  }
