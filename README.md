# MATACS-TH-2026-02: AI-Assisted Proposal Triage System

**Candidate:** Muzaffar Hassan  
**Assessment Reference:** `MATACS-TH-2026-02`  
**Time Spent:**  ~180 minutes

## Overview

This repository contains my system design for an AI-assisted contractor proposal triage and ranking system, submitted for the MATACS Associate Software Developer technical assessment.

The task is to design, not build, a system that evaluates contractor proposals for a 150 kW commercial rooftop solar installation in Windsor, Ontario.

The design separates AI-based fact extraction from deterministic rule validation and scoring. It supports configurable evaluation criteria, explainable rankings, human review, and final contract award decisions.

Administrators manage criteria, weights, thresholds, and scoring parameters through versioned configuration rather than proposal-specific hardcoded rules. When existing extracted facts cover updated criteria, the system can recalculate scores without running AI extraction again.

## Deliverables

- [Process Flow Diagram](./diagram/flow.png)
- [Editable Mermaid Diagram Source](./diagram/flow.md)
- [Design Write-up](./WRITEUP.md)
- [AI Extraction Prompts](./PROMPTS.md)
- [Sample Ranking Output](./ranking.json)

## Architecture at a Glance

```text
Admin Configuration
        ↓
PDF Upload and Text Extraction
        ↓
AI Fact Extraction
        ↓
Evidence Validation
        ↓
Mandatory Eligibility Checks
        ↓
Deterministic Weighted Scoring
        ↓
Ranking and Exception Handling
        ↓
Human Review and Final Decision
```

One reusable pipeline handles different project categories. AI extracts proposal facts into structured JSON. Application code validates evidence, applies mandatory rules, calculates weighted scores, and determines rankings. Flagged proposals remain unranked until human review resolves the outstanding issues.

## AI Usage Disclosure

I used Gemini 4 Argon and ChatGPT to understand the assessment requirements, then Gemini 3.8 Flash to break down each section and identify system design requirements. I drafted rough notes in Notepad to organize mandatory criteria, weights, and rules, then used ChatGPT and Gemini 4 Argon for brainstorming, refining the workflow, reviewing the diagram, and exploring edge cases. Finally, I used ChatGPT to refine the overall design and improve its clarity, reusability, and scalability.